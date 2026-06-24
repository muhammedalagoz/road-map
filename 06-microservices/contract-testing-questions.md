# 06d — Contract Testing: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Temel Hatalar & Contract Yönetimi

### Gerçek Hayat Sorunları

---

**Sorun 1: Contract test yok — provider değişikliği consumer'ı sessizce bozdu**

```
Senaryo:
  PaymentService ekibi: response'taki "paymentId" alanını "id" olarak rename etti.
  "Non-breaking gibi görünüyor, yeni alan ekliyoruz + eskisini bırakıyoruz" sanıldı.
  Ama eski alan silindi.

  OrderService: response.getPaymentId() → null döndü.
  OrderService: null kontrol yok → NullPointerException → sipariş tamamlanamadı.
  
  Tespit: production'da. 2 saat sonra alert geldi.
  Etki: 1200 sipariş başarısız oldu.

  E2E test neden tutmadı?
    E2E test ortamı: her zaman ayakta değil. Nightly build → sabah sonuç.
    Bu değişiklik nightly'den önce deploy edildi → ertesi sabah fark edildi.

Contract test ile nasıl önlenir:

  Consumer (OrderService) kontratı:
    .body(new PactDslJsonBody()
        .uuid("paymentId")     // "paymentId" alanı UUID formatında gelmeli
        .stringValue("status", "CHARGED")
    )

  Provider (PaymentService) contract doğrulaması:
    CI pipeline'da: provider testi çalışır.
    "paymentId" → "id" rename edildi → contract "paymentId" arıyor → bulamıyor.
    Test FAIL → CI pipeline durdu → deploy engellendi.

  Sonuç: breaking change production'a gitmeden CI'da yakalandı.
  Düzeltme: PaymentService ya eski alanı tuttu (backward compat)
             ya da OrderService contract güncelledi + beraber deploy.
```

---

**Sorun 2: Provider state kurulmadı — contract test geçti ama production'da hata**

```
Senaryo:
  OrderService contract:
    given("payment with id pay-999 exists")
    uponReceiving("a get payment request")
    path("/api/payments/pay-999")
    → willRespondWith 200 + payment detayı

  Provider test:
    @State("payment with id pay-999 exists")
    void setupPaymentExists() {
        // TODO: DB'ye ekle
        // Şimdilik boş bırakıldı — "sonra yazarız"
    }

  Sonuç:
    Provider test çalıştı → Pact framework HTTP isteği yaptı.
    PaymentService: pay-999 DB'de yok → 404 döndü.
    Contract: 200 bekliyordu → FAIL?
    
    Hayır — çünkü setupPaymentExists boş → provider her halükarda 404 döndü.
    Pact: "interaction doğrulandı" → PASS! (yanlış pozitif)
    
    Production: OrderService pay-999'u sorguladı → 404 → işlem bozuldu.
    Contract test geçti ama gerçek durum test edilmedi.

Doğru provider state implementasyonu:
  @State("payment with id pay-999 exists")
  void setupPaymentExists() {
      // Gerçekten DB'ye ekle (test veri)
      paymentRepository.save(Payment.builder()
          .id("pay-999")
          .orderId("order-123")
          .amount(new BigDecimal("99.99"))
          .status(PaymentStatus.CHARGED)
          .build());
  }

  @State("payment with id pay-999 exists")
  @Transactional
  void setupPaymentExistsAndCleanup() {
      paymentRepository.deleteAll();           // önce temizle
      paymentRepository.save(testPayment());   // sonra ekle
  }

Kural:
  Provider state = "consumer'ın 'given' koşulunu gerçekten kur."
  Boş bırakmak = contract testi kandırmak = güvenlik hissi yanlış.
  Her @State metodu review edilmeli: "bu koşul gerçekten kuruldu mu?"
```

---

**Sorun 3: Pact Broker kullanılmıyor — pact dosyaları elle kopyalandı, senkronizasyon bozuldu**

```
Senaryo:
  Küçük ekip: "Pact Broker kurmak zahmetli, pact dosyasını Git repo'ya commit edelim."

  OrderService: pact dosyası → Git'e commit.
  PaymentService: pact dosyasını Git'ten alıyor.

  Sorunlar 6 ay sonra:
    OrderService: pact dosyasını güncelledi → yeni branch'te.
    Branch merge gecikmeli → PaymentService eski pact ile çalışıyor.
    OrderService prod'a deploy oldu → yeni contract → PaymentService bozuldu.
    Hangi pact versiyonu hangi env'de geçerli? → belirsiz.

  Birden fazla consumer:
    InventoryService de PaymentService'i kullanıyor.
    Her biri ayrı pact dosyası → Git'te farklı dizinler.
    PaymentService: 4 farklı yerden pact dosyası toplamak zorunda.
    Biri eksik → test eksik → yanlış güven.

Pact Broker neden şart:

  Merkezi depo: tüm consumer'ların pact'ları tek yerde.
  Versioning: consumer v1.2.3 → pact X. consumer v1.3.0 → pact Y.
  "Can I Deploy?": "PaymentService v2.0 → tüm consumer'larla uyumlu mu?"
    → Pact Broker otomatik kontrol eder.
  Dashboard: hangi contract pass/fail → görsel.

  Ücretsiz: Pact Broker open-source (Docker ile kur).
  Managed: PactFlow (Pact'ın SaaS versiyonu).

  CI entegrasyonu:
    Consumer CI: test çalış → pact publish → Pact Broker.
    Provider CI: pact'ları Pact Broker'dan çek → doğrula → can-i-deploy.
    Doğru akış — versiyon karışıklığı yok.
```

---

**Sorun 4: Kafka event contract test edilmedi — schema değişikliği consumer'ı bozdu**

```
Senaryo:
  OrderService: "OrderCreated" event'i yayınlıyor.
  InventoryService: bu event'i consume ediyor.

  OrderService: event'e yeni zorunlu alan ekledi: "warehouseId".
  Schema Registry yok, Pact mesaj contract testi yok.

  InventoryService: OrderCreated event'ini deserialize etti.
    @KafkaListener
    void handleOrderCreated(OrderCreatedEvent event) {
        String warehouseId = event.getWarehouseId(); // null — alan yok!
        inventoryService.reserve(event.getItems(), warehouseId); // NPE
    }

  Sonuç: InventoryService consumer grubu hata → rebalance → lag birikimi.
  3 saat sonra: 50.000 işlenmemiş event → stok rezervasyonları gecikmeli.

Pact Message Contract Test çözümü:

  Consumer (InventoryService) event contract yazar:
    @Pact(consumer = "inventory-service", provider = "order-service")
    MessagePact orderCreatedPact(MessagePactBuilder builder) {
        return builder
            .expectsToReceive("an OrderCreated event")
            .withContent(new PactDslJsonBody()
                .stringType("orderId")
                .stringType("customerId")
                .stringType("warehouseId")   // consumer bu alanı bekliyor
                .array("items")
                    .object()
                        .stringType("productId")
                        .integerType("quantity")
                    .closeObject()
                .closeArray()
            )
            .toPact();
    }

  Provider (OrderService) doğrular:
    Event'te "warehouseId" var mı? → yeni alan eklenince → yeni contract yazılır.
    Consumer hazır değilse → can-i-deploy FAIL → deploy bloklanır.

  Schema Registry (Confluent/AWS Glue) ile birlikte:
    Avro/Protobuf schema → backward/forward compatibility kontrolü.
    Pact: "consumer bu alanları bekliyor."
    Schema Registry: "schema backward compat mi?"
    İkisi birlikte → event contract güvencesi.
```

---

**Sorun 5: "Can I Deploy?" kontrolü CI'da yok — uyumsuz versiyon prod'a gitti**

```
Senaryo:
  Ekip: Pact Broker var, consumer + provider testleri yazıldı.
  Ama CI pipeline'da "can-i-deploy" adımı YOK.

  OrderService v2.0: yeni payment contract beklentisi.
  PaymentService: henüz v2.0 contract'ını doğrulamamış (ekip meşguldü).

  OrderService v2.0 → production'a deploy edildi.
  PaymentService: prod'da hâlâ v1.8 (eski contract).
  OrderService: PaymentService'ten v2.0 response bekledi → eski format → hata.

Can I Deploy entegrasyonu (CI pipeline):

  # GitHub Actions / Jenkins / GitLab CI
  - name: Can I Deploy?
    run: |
      pact-broker can-i-deploy \
        --pacticipant order-service \
        --version ${{ github.sha }} \
        --to-environment production \
        --broker-base-url https://pact-broker.internal \
        --broker-token ${{ secrets.PACT_TOKEN }}
    # Exit code 0 → deploy güvenli
    # Exit code 1 → INCOMPATIBLE → pipeline durur, deploy olmaz

  Doğru CI akışı:
    Consumer test → pact publish → Can I Deploy? → deploy
    Provider test → pact verify → Can I Deploy? → deploy

  "Can I Deploy?" ne kontrol eder:
    "Bu versiyon, production'daki tüm karşı servislerle uyumlu mu?"
    order-service v2.0 ↔ payment-service v1.8 (prod'daki) → uyumlu mu?
    Değilse: deploy durdur.

  Ekstra güvence: pending pacts + WIP (Work in Progress) pacts.
  Yeni consumer contract → provider henüz doğrulamadı → "pending."
  Provider: pending pact yüzünden bloklanmaz ama görünür.
```

---

**Sorun 6: Contract test ile unit test karıştırıldı — provider tüm business logic'i test etti**

```
Senaryo:
  PaymentService provider testi:
    @State("payment service is available")
    void setupAvailable() { }

    @TestTemplate
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

  Ekip: "provider testi çalışıyor, başka test yazmayalım" dedi.
  Pact provider test: sadece HTTP request/response kontratını doğrular.
  Business logic: fraud kontrolü, para birimi dönüşümü, komisyon hesabı.
  Bu logic'ler Pact tarafından test edilmez.

  Sonuç:
    Fraud kontrolü bug'ı: Pact testi geçti (response format doğru).
    Production: sahte ödemeler onaylandı.
    Pact, fraud logic'i test etmedi — sadece "şu endpoint şu format döndürür" test etti.

Test piramidi ile doğru kullanım:

  Unit test:       Business logic → fraud kontrolü, hesaplama, validasyon.
  Contract test:   HTTP/event contract → "format doğru mu, alan var mı?"
  Smoke test:      Prod'da sanity → "endpoint erişilebilir mi?"
  E2E test:        Kritik happy path → minimum sayıda.

  Contract test NELER DEĞİLDİR:
    ✗ Business logic testi
    ✗ Performans testi
    ✗ Security testi (auth bypass, injection)
    ✗ Infrastructure testi (DB migration, network)
    ✗ Gerçek entegrasyon (DB'ye gerçekten yazıldı mı?)

  Ne ZAMaN YETERLİDİR:
    "Servisler birbiriyle konuşabiliyor mu?" → contract test.
    "İş mantığı doğru mu?" → unit test.
    "Sistem uçtan uca çalışıyor mu?" → E2E (az sayıda, kritik path).
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Contract testing nedir? E2E testten farkı nedir?

   > **Beklened:** Contract testing: consumer'ın provider'dan ne beklediğini "sözleşme" olarak yazar, provider bu sözleşmeyi karşılayıp karşılamadığını otomatik doğrular. E2E testten farkı: E2E → tüm servisler ayakta olmalı, gerçek ortamda test. Yavaş (30-60 dk), flaky (herhangi bir servis instabil → tüm test başarısız), debug zor. Contract test → consumer testi: consumer + mock provider (izole, hızlı). Provider testi: provider + contract dosyası (izole, hızlı). Tüm servisler ayakta olmak zorunda değil. Neyi çözer: microservis A değişince mikroservis B'yi bozuyor mu? Contract test: breaking change'i CI'da, production'a gitmeden yakalar. Pact: en yaygın framework. Consumer contract yazar → Pact Broker'a publish → Provider doğrular → "can-i-deploy" kararı.

2. Pact'ta "consumer-driven" ne anlama gelir? Provider-driven'dan farkı nedir?

   > **Beklened:** Consumer-driven: contract'ı consumer yazar. "Ben bu servisin şu endpointinden şunu bekliyorum" — consumer belirler. Provider: "consumer'ın beklentilerini karşılıyor muyum?" diye test eder. Fayda: provider API'yi consumer'ın gerçek ihtiyacına göre tasarlar. "Belki lazım olur" diye fazla alan koymaz. Provider-driven (alternatif): provider kendi API spec'ini yazar (OpenAPI/Swagger), consumer ona uyar. Sorun: provider "ben bu formatı sunuyorum, consumer kullansın" → ama consumer tam ihtiyacını alamayabilir. Consumer-driven öğrenme: provider, consumer'ın gerçekte ne kullandığını görür. Kullanılmayan alanlar → temizlenebilir. Kullanıcı (consumer) odaklı API tasarımı.

3. Pact "matching rules" nedir? Exact match ile type match farkı?

   > **Beklened:** Exact match: "status" alanı tam olarak "CHARGED" string'i olmalı. Değeri test eder. Type match: "orderId" alanı string tipinde olmalı — değeri önemli değil, tipi önemli. Neden type match: consumer "orderId string gelecek" bilgisini kullanır, "abc-123" mi "def-456" mi olduğunu umursamaz. Exact match: "status" için mantıklı (CHARGED, FAILED gibi sabit değerler). Gereksiz exact match: "paymentId" alanı "550e8400-..." sabit UUID → her testte aynı UUID → gerçekçi değil. Doğru: .uuid("paymentId") → UUID formatında olsun, değeri değişebilir. Diğer matchers: decimalType, integerType, datetime (format), regex. Kural: değer önemliyse exact, tip/format önemliyse type/format matcher kullan.

---

**Senior / Architect:**

4. Pact Broker "can-i-deploy" komutu nasıl çalışır? CI/CD'ye nasıl entegre edilir?

   > **Beklened:** Can-i-deploy: "bu servis versiyonu, hedef environment'taki karşı servislerle uyumlu mu?" sorusunu yanıtlar. Pact Broker: her servis versiyonu + doğrulanmış contract'ları bilir. "order-service v1.2.3 → production'a deploy edilsin mi?" → Broker: "prod'da payment-service v2.1.0 var. order-service v1.2.3'ün payment-service v2.1.0 ile kontratı doğrulandı mı?" → Evet → deploy güvenli. CI entegrasyonu: deploy adımından ÖNCE çalışır. Başarısız → pipeline durur, deploy olmaz. Başarılı → deploy devam eder. Environment kaydı: deploy başarılı olunca Pact Broker'a bildir: `pact-broker record-deployment --pacticipant order-service --version 1.2.3 --environment production`. Broker: "artık prod'da order-service v1.2.3 var" bilir. Sonraki can-i-deploy: bu bilgiyi kullanır.

5. Kafka event'leri için contract test nasıl yapılır? HTTP contract testinden farkı?

   > **Beklened:** HTTP contract: request/response senkron. Mock server kurulur, consumer HTTP çağrısı yapar, response format doğrulanır. Event/message contract: asenkron, request/response yok. Consumer "şu formatta event alacağım" yazar. Provider "şu formatta event yayınlıyorum" doğrular. Pact MessagePact: consumer → `expectsToReceive("OrderCreated event").withContent(...)`. Test: mock message ile consumer handler'ını test et — gerçek Kafka yok. Provider verification: OrderService event üretir → Pact formatla karşılaştırır. Fark: HTTP'de mock server HTTP isteği alır. Event'te mock message handler'a verilir — Kafka broker gereksiz. Kullanım: schema değişikliğini production'a gitmeden yakala. Schema Registry ile birlikte: Pact consumer beklentisi + Registry schema compat → çift güvence. Özellikle: yeni zorunlu alan, alan silme, tip değişimi → her ikisi de yakalar.

6. Contract testing hangi sorunları ÇÖZMEZ? Test stratejisinde nereye konumlanır?

   > **Beklened:** Contract testing çözmez: (1) Business logic doğrulama → "fraud kontrolü doğru mu?" → unit test. (2) Performans → "10.000 req/s'de ne olur?" → load test. (3) Infrastructure → "DB migration güvenli mi?" → integration test. (4) Güvenlik → "SQL injection önlendi mi?" → security test. (5) Gerçek entegrasyon → "DB'ye gerçekten yazıldı mı?" → integration/E2E. (6) 3rd party servisler → "Stripe API'si gerçekten çalışıyor mu?" → Wiremock stub + smoke test. Test piramidi konumu: unit (çok, hızlı) → contract (servisler arası "konuşabiliyorlar mı?") → smoke (prod sanity) → E2E (az, kritik path). Contract test: "format uyumluluğu" güvencesi. İş mantığı, performans, güvenlik → ayrı katmanlarda. Yanılgı: "contract test var, E2E gerekmez" → kritik path hâlâ E2E ile doğrulanmalı.

---

## Karma — Architect Seviyesi

7. **"10 mikroservisli sistemde contract testing'i sıfırdan kuruyorsun. Nereden başlarsın?"**

   > **Beklened:** (1) Pact Broker kur: Docker ile on-premise veya PactFlow (SaaS). CI erişimi olan URL + token. (2) En kritik servis çiftinden başla: ödeme akışı (OrderService → PaymentService) veya en sık bozulan entegrasyon. Tüm servisleri aynı anda yapmaya çalışma — başarısız olur. (3) Consumer testini yaz: OrderService → PaymentService için Pact consumer testi. Pact dosyası üret, Pact Broker'a publish. (4) Provider testini yaz: PaymentService → Pact Broker'dan consumer pact'larını çek, doğrula. Provider state handler'larını yaz (boş bırakma!). (5) CI entegrasyon: consumer CI: test → publish. Provider CI: verify → can-i-deploy. (6) Genişlet: ikinci servis çifti. Ekip eğitimi: "yeni endpoint → önce contract yaz." (7) Monitoring: Pact Broker dashboard → hangi contract fail, hangi servis kırıldı. (8) Kural koy: "can-i-deploy geçmeden deploy yok" → ekip disiplini. Pitfall: provider state'leri boş bırakmak, pact publish'i unutmak, can-i-deploy'u CI'a eklememeği ertelemek.

8. **"Provider ekibi 'breaking change yapmıyoruz, neden contract test yazalım?' diyor. Ne yanıt verirsin?"**

   > **Beklened:** Argüman 1 — "Breaking change" tanımı geniş: Alan adı rename, tip değişimi, zorunlu alan ekleme, enum değeri ekleme (consumer switch yaparsa), HTTP status kodu değişimi — hepsi breaking. "Breaking change yapmıyoruz" derken bunların hepsini kapsıyor mu? Genellikle hayır. Argüman 2 — İnsan hatası: "yapmıyoruz" ile "hiç olmadı" farklı. Contract test: insan hafızasına güvenmez, otomatik doğrular. Argüman 3 — Consumer'ın gerçekte ne kullandığını bilmiyorsunuz: "paymentId" alanını 3 consumer kullanıyor, 1'i kullanmıyor. Hangisi? Contract test olmadan bilmezsiniz. Silebilir misiniz? Bilemezsiniz. Argüman 4 — "Can I Deploy?" güvencesi: yeni provider versiyonu deploy edilmeden önce "tüm consumer'larla uyumlu mu?" → tek komutla. Olmadan: "uyumlu sanıyoruz" → production'da test. Argüman 5 — Maliyet: Pact Broker kurma: 1 gün. Consumer + provider test yazma: 2-3 gün. Production incident maliyeti: 1 engineer × 3 saat + müşteri güvensizliği. ROI nette pozitif.
