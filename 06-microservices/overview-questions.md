# 06 — Microservices & DDD: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Monolith → Microservices & DDD

### Gerçek Hayat Sorunları

---

**Sorun 1: Erken microservices — domain netleşmeden ayrılma, "distributed monolith"**

```
Senaryo:
  Startup, 5 developer, 3 aylık ürün.
  "Microservices yapalım, modern mimari olsun."

  8 servis oluşturuldu: UserService, OrderService, ProductService,
  CartService, PricingService, DiscountService, NotificationService, SearchService.

  6 ay sonra:
    Senaryo A: "Sepete ürün ekle" işlemi:
      CartService → ProductService (stok var mı?)
      CartService → PricingService (fiyat ne?)
      CartService → DiscountService (indirim var mı?)
      CartService → UserService (kullanıcı aktif mi?)
      4 senkron çağrı → CartService down riski 4×.

    Senaryo B: "Yeni indirim ekle" değişikliği:
      DiscountService değişti → PricingService değişmeli → CartService değişmeli.
      3 servis aynı anda deploy edilmeli → "bağımsız deployment" yok.

    Senaryo C: Ekip 5 kişi → her servis için ayrı CI/CD, monitoring, logging.
      Operasyonel yük: 5 kişiyi ezdi.

  Teşhis: "Distributed Monolith."
    Servisler teknik sınırla ayrıldı, iş yeteneği bazlı değil.
    Sıkı coupling → bağımsız değiştirilemiyor.
    Küçük ekip → operasyonel yük karşılanamıyor.

Ne zaman microservices:
  ✓ Ekip büyüdü (10+ developer, birden fazla takım).
  ✓ Domain netleşti (servis sınırları değişmiyor).
  ✓ Belirli modüller farklı ölçekleme ihtiyacı (ör: arama çok yoğun).
  ✓ Bağımsız deployment kritik hale geldi.
  
  Kural: "Monolith ile başla, ihtiyaç doğunca ayır."
  Sam Newman: "Monolith first, microservices when you have good reason."
```

---

**Sorun 2: Yanlış bounded context — "Customer" kavramı her yerde farklı**

```
Senaryo:
  E-ticaret, tek "Customer" modeli her serviste:

  class Customer {
      Long id;
      String name, email, phone;        // Order'ın ihtiyacı
      String segment, loyaltyPoints;    // CRM'in ihtiyacı
      String billingAddress, taxNumber; // Payment'ın ihtiyacı
      String shippingAddresses[];       // Shipping'in ihtiyacı
      String supportTickets[];          // Support'un ihtiyacı
  }

  Sorunlar:
    OrderService: Customer sınıfına kendi ihtiyacı olmayan 15 alan yükledi.
    CRM güncelledi: loyaltyPoints → loyaltyScore rename.
    OrderService: bu alanı kullanmıyor ama şema değişti → compile error.
    PaymentService güncelledi: taxNumber zorunlu oldu.
    OrderService: taxNumber'ı bilmiyor → 3 serviste aynı anda değişiklik.

Bounded Context çözümü:
  Her context kendi Customer modelini tanımlar:

  // Order Context: sadece ihtiyacı olan
  class Customer {
      CustomerId id;
      String name;
      Address shippingAddress;
  }

  // Payment Context
  class Customer {
      CustomerId id;
      String taxNumber;
      BillingAddress billingAddress;
  }

  // CRM Context
  class Customer {
      CustomerId id;
      String email, phone;
      CustomerSegment segment;
      LoyaltyPoints loyaltyPoints;
      List<SupportTicket> tickets;
  }

  "Customer her context'te farklı anlam taşır — bu normal."
  DDD öğrenimi: aynı kavramın farklı context'lerdeki farklı modeli.
  Anti-Corruption Layer (ACL): dış sistemden gelen Customer → iç modele çevir.
  Dış API değişse → sadece ACL güncellenir, iç model korunur.
```

---

**Sorun 3: Outbox pattern yok — DB kaydedildi, Kafka mesajı gönderilmedi**

```
Senaryo:
  // YANLIŞ implementasyon:
  @Transactional
  void createOrder(Order order) {
      orderRepo.save(order);                            // DB: OK
      kafkaTemplate.send("order-created", event);      // Kafka: HATA!
      // Kafka broker geçici olarak ulaşılamıyor (30s)
  }

  T+0:  orderRepo.save(order) → DB'ye yazıldı.
  T+1:  kafkaTemplate.send() → Kafka timeout → KafkaException.
  T+1:  @Transactional: exception → DB rollback.
  
  Sorun 1: Kafka hata verdi → DB rollback → sipariş kaydedilmedi.
           Kullanıcı: "sipariş oluşturuldu" mu "oluşturulmadı" mı? Belirsiz.
  
  Ama bazı durumlarda tersine:
  T+0:  orderRepo.save(order) → commit (transaction dışında kafkaTemplate!)
  T+1:  kafkaTemplate.send() → HATA → ama DB zaten commit edildi!
  Sonuç: DB'de sipariş var, Kafka'da event yok → InventoryService bilmedi → stok düşmedi.

Outbox Pattern çözümü:
  @Transactional
  void createOrder(Order order) {
      orderRepo.save(order);
      // Kafka'ya değil, outbox tablosuna yaz — AYNI transaction!
      outboxRepo.save(OutboxMessage.builder()
          .aggregateType("Order")
          .aggregateId(order.getId())
          .eventType("OrderCreated")
          .payload(toJson(event))
          .build());
      // Transaction commit → hem order hem outbox atomik yazıldı.
  }

  // CDC (Debezium) outbox tablosunu izler:
  // PostgreSQL WAL → Debezium → Kafka topic
  // DB commit anında event akışa girer → polling yok, gerçek zamanlı.

  Garanti:
    DB commit başarılı → outbox kaydı var → CDC → Kafka → guaranteed delivery.
    DB rollback → outbox yok → Kafka'ya gitmez.
    Kafka geçici olarak down → outbox tablosunda bekler → Kafka gelince gönderilir.
    Atomiklik: DB transaction garantisi ile event publishing.
```

---

**Sorun 4: Servis sınırı yanlış çizildi — her özellik birden fazla servisi değiştiriyor**

```
Senaryo:
  "Teknik katman" bazlı servis ayrımı:
    DataService    → tüm veri işlemleri
    BusinessService → iş mantığı
    APIService     → dış API'ler

  Yeni özellik: "Sipariş iptal edilince kullanıcıya email gönder."
    BusinessService: iptal mantığı eklendi.
    DataService: iptal durumu schema güncellendi.
    APIService: email endpoint eklendi.
  3 servis → 3 ayrı PR → 3 ayrı deploy → koordinasyon.

  Heuristic ihlali: "Bir değişiklik birden fazla servisi etkiliyorsa → yanlış ayrım."

Doğru: İş yeteneği bazlı ayrım (business capability):
  OrderManagement  → sipariş al, güncelle, iptal et.
  Notification     → email, SMS, push notification.
  
  "Sipariş iptal email" değişikliği:
    OrderManagement: iptal mantığı → OrderCancelled event yayınla.
    Notification: OrderCancelled event'i dinle → email gönder.
    2 servis etkilendi ama bağımsız olarak değiştirilebilir.
    OrderManagement: Notification'ı bilmez → loose coupling.

Conway's Law:
  "Sistemin mimarisi, onu geliştiren organizasyonun iletişim yapısını yansıtır."
  Takım A → OrderManagement. Takım B → Notification.
  Takımlar bağımsız → servisler bağımsız → mimari sağlıklı.
  Ters Conway Manevrası: "Mimarimizi nasıl istiyorsak, ekibimizi öyle yapılandıralım."
```

---

## Bölüm 2: CQRS, Event Sourcing & Service Mesh

### Gerçek Hayat Sorunları

---

**Sorun 5: CQRS olmadan yüksek okuma yükü — write DB darboğazı**

```
Senaryo:
  E-ticaret dashboard: Order tablosundan karmaşık aggregation sorguları.
  Aynı Order tablosu yazma işlemleri için de kullanılıyor.

  Okuma: 100 req/s → her biri 200ms (GROUP BY, JOIN).
  Yazma: 10 req/s → her biri 5ms.

  Sorun:
    Okuma sorguları DB'yi meşgul ediyor → yazma yavaşladı.
    Sipariş oluşturma: normalde 5ms → 150ms'ye çıktı (read lock contention).
    Read replica ekledik → ama read replica da yavaşladı (aynı şema, aynı index).

CQRS çözümü:

  Write side (Command):
    Order tablosu → normalized, ACID, yazma optimize.
    OrderCreated, OrderUpdated event'leri yayınlar.

  Read side (Query):
    Event'leri dinle → denormalize read model oluştur:
    
    OrderSummaryView:   {orderId, customerName, total, status, createdAt}
    CustomerOrdersView: {customerId → sipariş listesi}
    DashboardStatsView: {günlük sipariş sayısı, toplam gelir, iade oranı}
    
    Her view kendi ihtiyacına göre optimize edilmiş index.
    DashboardStatsView: ClickHouse (OLAP) → aggregate sorgular µs.
    CustomerOrdersView: Redis → O(1) okuma.

  Sonuç:
    Yazma: Order tablosu rahatladı → 5ms.
    Okuma: optimize read model → 5ms (Redis) veya 20ms (ClickHouse).
    Total: 200ms → 20ms okuma, 5ms yazma, birbirini etkilemiyor.
    
  Ne zaman CQRS:
    Read/Write oranı 10:1'den fazla.
    Read model yazma modelinden farklı optimizasyon gerektiriyor.
    Event Sourcing kullanılıyorsa (doğal uyum).
```

---

**Sorun 6: Event Sourcing — schema değişikliği, eski event'ler uyumsuz**

```
Senaryo:
  OrderCreatedEvent v1:
    { "orderId": "123", "customerId": "456", "items": [...] }

  6 ay sonra: "currency" alanı zorunlu oldu.
  OrderCreatedEvent v2:
    { "orderId": "123", "customerId": "456", "items": [...], "currency": "TRY" }

  Event store'da 10 milyon eski event (v1 format, "currency" yok).
  Siparişi rebuild etmek için tüm event'leri replay et:
    apply(OrderCreatedEvent event) → event.getCurrency() → null!
    NullPointerException → sipariş state rebuild edilemiyor.

Upcasting çözümü:
  // Event'i okurken eski versiyonu yeni versiyona dönüştür:
  class OrderCreatedEventUpcaster implements EventUpcaster {
      public boolean canUpcast(EventData event) {
          return "OrderCreated".equals(event.getType())
              && event.getVersion() < 2;
      }

      public EventData upcast(EventData event) {
          JsonNode payload = parseJson(event.getPayload());
          // Eksik alanı ekle (default değer)
          ((ObjectNode) payload).put("currency", "TRY"); // v1 döneminde sadece TRY
          return event.withPayload(payload).withVersion(2);
      }
  }

  Event okunurken: v1 → Upcaster → v2 → aggregate.apply(v2).
  Event store: orijinal v1 kayıtları değişmez (immutable).

Event Sourcing kuralları:
  1. Event'ler immutable — hiçbir zaman mevcut event'i değiştirme.
  2. Yeni alan opsiyonel + default ile ekle → backward compat.
  3. Alan sil/rename → versiyonlu event + upcaster.
  4. Event schema: Avro/Protobuf → schema registry → compat kontrolü.
  5. Snapshot: 1000 event sonra anlık durum → replay süresi azalt.
```

---

**Sorun 7: Service Mesh olmadan — her serviste ayrı retry/circuit breaker kodu**

```
Senaryo:
  10 microservice. Her servis:
    Resilience4j circuit breaker konfigürasyonu (Java).
    Retry logic.
    Timeout ayarları.
    mTLS kendi implementasyonu.
    Tracing: her serviste manuel OpenTelemetry.

  Sorunlar:
    6 ay sonra: timeout değerleri tutarsız (A servisi 5s, B servisi 2s, C servisi 30s).
    Circuit breaker threshold'ları farklı → bir servis OPEN'da, diğeri CLOSED.
    mTLS: bir servis implement etmeyi unuttu → güvenlik açığı.
    Go ile yazılmış yeni servis: Java kütüphaneleri yok → sıfırdan yaz.
    Retry storm: A retry etti, B retry etti → cascading failure.

Service Mesh (Istio) çözümü:
  Uygulama kodu değişmeden → Envoy sidecar proxy:

  Tüm özellikler YAML ile merkezi yönetim:

  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: order-service
  spec:
    http:
    - timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,connect-failure

  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  spec:
    trafficPolicy:
      outlierDetection:          # Circuit Breaker
        consecutive5xxErrors: 5
        interval: 30s
        baseEjectionTime: 30s

  Avantajlar:
    mTLS: otomatik, tüm servis-servis trafiği şifreli (kod değişikliği yok).
    Merkezi policy: timeout, retry, CB → tek yerden tüm servislere.
    Dil bağımsız: Java, Go, Python → hepsi Envoy kullanır.
    Distributed tracing: Envoy otomatik trace header ekler.

  Dezavantaj:
    Kompleksite: Istio öğrenme eğrisi yüksek.
    Latency: +1-2ms (Envoy hop).
    Kaynak: her pod +50-100MB Envoy.
    Ne zaman: 10+ servis, compliance (mTLS zorunlu), merkezi observability.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Microservices ne zaman tercih edilmez? Hangi koşullarda monolith daha iyi?

   > **Beklened:** Monolith tercih: ekip < 10 developer (her servis için ayrı operasyonel yük taşınamaz). Domain henüz netleşmemiş (sık değişen sınırlar → sürekli refactor). İlk ürün, pivot olabilir (microservices: yüksek initial complexity). Tüm modüller aynı ölçekleme ihtiyacında. Microservices'in maliyetleri: distributed tracing, service discovery, network latency, Saga pattern, eventual consistency, çoklu DB yönetimi. Erken microservices: "distributed monolith" riski (servisler sıkı coupled → tüm fayda yok, tüm maliyet var). Martin Fowler: "Monolith first" — önce iyi monolith, sonra ihtiyaç doğunca ayır. Ne zaman ayır: ekipler bağımsız deploy edemiyor, belirli modüller farklı ölçekleme gerektiriyor, domain boundaries netleşti.

2. DDD'de Bounded Context nedir? Neden "Customer" her context'te farklı olabilir?

   > **Beklened:** Bounded Context: bir domain modelinin geçerli olduğu sınır. Model o sınır içinde tutarlı ve net anlam taşır. Neden farklı: "Customer" kavramı her iş alanında farklı anlam. Order Context: müşteri kim? id, ad, teslimat adresi — yeterli. CRM Context: müşteri kim? segmenti, puanları, destek geçmişi, email kampanyaları. Payment Context: müşteri kim? vergi numarası, fatura adresi, ödeme yöntemi. Hepsi "Customer" ama çok farklı. Yanlış: herkese uyacak tek büyük Customer modeli → her context gereksiz alanlar taşır → değişiklik coupling'i artar. Doğru: her context kendi modelini tanımlar. Anti-Corruption Layer: dış sistem farklı Customer modeli → ACL dönüştürür → iç model korunur. Ubiquitous Language: o context içinde herkes aynı dili konuşur (developer + business).

3. Outbox Pattern neden gereklidir? Hangi problemi çözer?

   > **Beklened:** Problem: DB'ye kaydet + Kafka'ya mesaj gönder — ikisi iki ayrı sistem → atomik değil. Senaryo 1: DB başarılı, Kafka hata → DB rollback → data kaybı, kullanıcı belirsizlikte. Senaryo 2: Kafka @Transactional dışında → DB commit, Kafka hata → DB'de kayıt var, Kafka'da event yok → InventoryService bilmedi. Outbox çözümü: DB'ye kaydet + outbox tablosuna event yaz → AYNI transaction. İkisi ya beraber commit ya beraber rollback → atomik. Sonra: CDC (Debezium) outbox tablosunu izler → PostgreSQL WAL → Kafka. Kafka geçici down: outbox tablosunda bekler → gelince gönderilir. Garanti: at-least-once delivery. Consumer idempotent olmalı (aynı event iki kez gelebilir).

4. gRPC'yi REST'e tercih ettiğin senaryolar nelerdir?

   > **Beklened:** gRPC tercih: (1) Internal microservice iletişimi: browser erişimi yok, binary protocol sorun değil. (2) Yüksek throughput: JSON parsing overhead → Protobuf 3-10× küçük payload, 5-10× hızlı parse. (3) Streaming: server-side streaming (gerçek zamanlı feed), bidirectional (chat, real-time). (4) Strong typing: proto şema → compile-time contract, code generation (client/server stub). (5) Çok dilli ekip: proto → Java, Go, Python stub otomatik üretilir. REST tercih: (1) Public API, browser erişimi. (2) Human-readable debug. (3) Mevcut HTTP infrastructure. (4) Basit CRUD, proto şema yönetimi gereksiz. Hibrit: dış API → REST. İç servisler arası → gRPC.

---

**Senior / Architect:**

5. CQRS nedir? Ne zaman kullanılır, ne zaman overkill?

   > **Beklened:** CQRS: read ve write modelini ayır. Write: normalize DB, ACID, command handler → domain event. Read: event'leri dinle → denormalize read model (view) oluştur, okuma için optimize. Ne zaman kullanılır: Read/Write oranı çok farklı (100:1 gibi). Read modeli farklı optimizasyon gerektiriyor (dashboard → aggregate sorgular, liste → hızlı lookup). Event Sourcing ile doğal uyum. Farklı ölçekleme: write DB ACID (az sayıda), read DB yüksek throughput. Ne zaman overkill: basit CRUD (todo app, admin panel). Küçük ekip (kompleksite kaldıramaz). Read/Write benzer yük. Eventual consistency kabul edilemiyorsa (read model biraz stale). Maliyet: iki model → iki codebase, eventual consistency, yeni view eklemek için event replay. Fayda: write ve read birbirini etkilemez, her biri bağımsız optimize.

6. Event Sourcing'in avantajları ve dezavantajları. Hangi senaryoda tercih edilir?

   > **Beklened:** Avantajlar: (1) Tam audit log: "ne zaman ne oldu?" tüm geçmiş. Finansal sistemler, compliance için. (2) Zaman yolculuğu: herhangi bir anki state'e gidebilirsin (event replay). (3) Yeni projeksiyon: geçmişteki tüm event'leri yeni view için replay et (yeni analitik rapor — veri zaten var). (4) Event-driven ile doğal uyum. Dezavantajlar: (1) Karmaşıklık: aggregate reconstitution, snapshot, upcaster. (2) Eventual consistency: projeksiyon biraz geride olabilir. (3) Event schema değişimi zor: upcaster yazmak zorunda kalırsın. (4) Sorgu zorluğu: "stoku 50'den az olan ürünler?" → projeksiyon olmadan doğrudan event'lerden cevaplanamaz. (5) Yüksek event sayısı → replay yavaş (snapshot ile azaltılır). Tercih: finansal sistemler (audit şart), karmaşık domain logic, event-driven mimari, yeni analitik ihtiyaçları sık çıkan sistemler. Kullanma: basit CRUD, küçük ekip, audit gereksinimi yok.

7. Service Mesh ne zaman gerekir? Istio'nun maliyeti nedir?

   > **Beklened:** Ne zaman gerekir: 10+ microservice (az serviste overhead değmez). mTLS zorunluysa (compliance, HIPAA, PCI-DSS). Fine-grained traffic control: canary (%5 yeni versiyon), A/B test, traffic mirroring. Merkezi observability: tüm servislerden tek yerden distributed trace. Dil bağımsız policy: Java + Go + Python → hepsi Envoy, hepsi aynı policy. Maliyetler: (1) Karmaşıklık: Istio öğrenme eğrisi yüksek. Operasyon: istiod yönetimi, VirtualService/DestinationRule YAML. (2) Latency: +1-2ms Envoy hop (her request iki kez Envoy'dan geçer). (3) Kaynak: her pod +50-100MB Envoy sidecar, CPU overhead. (4) Debugging: "neden bu request yavaş?" → Envoy mi, uygulama mı? Alternatifler: az servis → Resilience4j (kod içi). mTLS gerekmiyorsa → API Gateway yeterli. Linkerd: Istio'dan hafif, daha az özellik ama operasyonu kolay.

---

## Karma — Architect Seviyesi

8. **"Startup'ın monolith'ini microservicese geçirmek istiyor. Nereden başlarsın?"**

   > **Beklened:** Önce soru sor: neden geçmek istiyorlar? Deployment bağımsızlığı mı, ölçekleme mi, ekip büyümesi mi? Gerçek problem net olmalı. Strangler Fig planı: (1) API Gateway ekle: tüm trafik gateway'den geçsin. Monolith arkada duruyor. (2) En iyi ayrışan servisi seç: az bağımlılık, net domain sınırı, sık değişen veya farklı ölçekleme ihtiyacı. Örnek: NotificationService — email/SMS gönderme, monolith'e bağımlılık az. (3) Yeni servis yaz: NotificationService bağımsız deploy edilir. (4) Gateway: notification endpoint'leri → yeni servis. Monolith: notification kodu deaktive. (5) Tekrarla: bir sonraki servisi belirle. Domain boundaries netleştikçe devam. Kaçınılacaklar: hepsini aynı anda ayırma. Shared DB anti-pattern → database-per-service geçiş planı yap. Domain netleşmeden ayırma → sık sınır değişikliği. Observability: distributed tracing, centralized logging → ilk günden kur.

9. **"Bir engineer 'her şeyi event sourcing ile yapalım' diyor. Ne söylersin?"**

   > **Beklened:** Event sourcing her yerde: overkill ve tehlikeli. Doğru yerde güçlü, yanlış yerde maliyet. Soru sor: audit log gerçekten şart mı? Tüm sistemde mi, yoksa belirli domainlerde mi? "Herhangi bir anki state'e git" özelliği kullanılacak mı? Ekip event sourcing ve upcaster'ı yönetecek olgunlukta mı? Maliyetler hatırlat: event schema değişimi karmaşık (upcaster). Projection olmadan sorgulama zor. Eventual consistency her yerde. Event store büyüdükçe replay yavaş (snapshot). Karşı öneri: event sourcing, gerçekten audit + temporal query gereksinimi olan domainler için (FinancialTransaction, OrderLifecycle). Basit CRUD domainler (UserProfile, ProductCatalog) → klasik DB. Hibrit: bazı aggregateler event sourcing, bazıları değil. Martin Fowler: "Event Sourcing is a significant architectural decision, not a default." Önce ihtiyacı kanıtla, sonra implement et.
