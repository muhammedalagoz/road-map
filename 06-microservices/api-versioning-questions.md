# 06b — API Versioning: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Breaking Change & Backward Compatibility

### Gerçek Hayat Sorunları

---

**Sorun 1: Alan adı değiştirildi — mobil client'lar çöktü**

```
Senaryo:
  Backend ekibi: "customerId düz alan yerine nested customer objesi olsun."
  
  v1 response (önceki):
    { "orderId": "123", "customerId": "456", "total": 99.99 }

  Yeni response (versioning YOK, direkt değiştirildi):
    { "orderId": "123", "customer": { "id": "456", "name": "Ali" }, "total": 99.99 }

  Sonuç:
    Android app: order.customerId → NullPointerException (alan yok artık).
    iOS app: JSON parse hatası → siparişler ekranı boş.
    3rd party partner: entegrasyon bozuldu.
    
    Mobil app düzeltmesi: App Store review 3-5 gün.
    Bu 3-5 gün boyunca tüm mobil kullanıcılar etkilendi.

Kök neden:
  "customerId → customer.id" → breaking change.
  Versioning olmadan direkt değişiklik → client'lar hazırlıksız.

Doğru yaklaşım:
  Seçenek A — Additive (geriye uyumlu):
    Eski alan KOR, yeni alanı da EKLe:
    {
      "orderId": "123",
      "customerId": "456",       // ESKİ — backward compat için tut
      "customer": {              // YENİ
        "id": "456",
        "name": "Ali"
      },
      "total": 99.99
    }
    Eski client: customerId alanını okur, "customer" alanını ignore eder.
    Yeni client: customer objesini kullanır.
    Deprecation: customerId alanını "deprecated" olarak belgele, 6 ay sonra kaldır.

  Seçenek B — URL versioning:
    /api/v1/orders → eski format (customerId flat)
    /api/v2/orders → yeni format (customer nested)
    Her iki endpoint paralel çalışır.
    Mobil app store onayı beklendikten sonra v1 sunset.
```

---

**Sorun 2: Response'tan alan silindi — downstream servis NullPointerException**

```
Senaryo:
  OrderService: v1 response'ta "estimatedDelivery" alanı vardı.
  Backend: "bu hesaplama yanlıştı, kaldıralım" → versioning olmadan sildi.

  ShippingService (consumer):
    String deliveryDate = orderResponse.getEstimatedDelivery();
    // null geldi → sonraki satır NullPointerException
    LocalDate date = LocalDate.parse(deliveryDate); // PATLADI

  Sonuç:
    ShippingService: runtime exception → sipariş gönderimleri durdu.
    Incident: 2 saat production outage.

Non-breaking vs breaking karar rehberi:

  NON-BREAKING (versiyonlama gerekmez):
    ✓ Yeni alan EKLE (client bilmediği alanı ignore eder)
    ✓ Yeni endpoint ekle
    ✓ Opsiyonel request parametresi ekle (default değer ile)
    ✓ Response'a yeni opsiyonel alan ekle
    ✓ Hata mesajı değiştir (yalnızca string, parse edilmiyorsa)

  BREAKING (yeni versiyon şart):
    ✗ Alan adı değiştir
    ✗ Alan tipini değiştir (string → int, int → string[])
    ✗ Zorunlu request alanı ekle (eski client göndermiyor)
    ✗ Response'tan alan SİL
    ✗ HTTP status code değiştir (200 → 201, 404 → 400)
    ✗ URL path değiştir
    ✗ Davranış değiştir (sıralama FIFO → LIFO)
    ✗ Enum'a yeni değer ekle (consumer pattern match yapıyorsa)

Düzeltme:
  "estimatedDelivery" kaldırmak yerine → null veya boş döndür.
  Deprecation header ekle: response'ta uyar.
  ShippingService: null check ekle (defensive coding).
  API contract test (consumer-driven): Pact ile → consumer beklentisi kontrol.
```

---

**Sorun 3: Zorunlu alan eklendi — eski client 400 Bad Request aldı**

```
Senaryo:
  PaymentService v1: POST /payments
  Request body: { "amount": 100, "currency": "TRY" }

  Backend: "artık paymentMethodId zorunlu" → validasyona eklendi.
  Versioning yapılmadı, aynı endpoint güncellendi.

  Eski Android app (v1): paymentMethodId alanını bilmiyor.
  POST /payments → { "amount": 100, "currency": "TRY" }
  Response: 400 Bad Request → "paymentMethodId is required"

  Sonuç:
    App Store'daki eski app'ler ödeme yapamıyor.
    Kullanıcı şikayeti: "neden ödeme yapamıyorum?"
    Güncelleme zorunluluğu → force update mekanizması devreye alındı.
    Force update: kötü kullanıcı deneyimi.

Zorunlu alan ekleme = her zaman breaking change.
Çözüm seçenekleri:

  1. Opsiyonel yap + server-side default:
     paymentMethodId opsiyonel → backend varsayılan ödeme yöntemini kullansın.
     Yeni client: açıkça gönderir. Eski client: default kullanır.

  2. Yeni endpoint (versioning):
     POST /api/v1/payments → eski format (paymentMethodId opsiyonel)
     POST /api/v2/payments → yeni format (paymentMethodId zorunlu)
     Eski client: v1 kullanmaya devam.
     Yeni client: v2.

  3. Graceful degradation:
     paymentMethodId gelmezse → kullanıcının default ödeme yöntemi.
     DB'den çek → "en son kullanılan kart."
     Backward compat korundu, yeni field opsiyonel.

API tasarım ilkesi:
  "Yeni zorunlu alan" → mutlaka yeni versiyon.
  "Yeni opsiyonel alan" → backward compat (versiyon gerekmez).
```

---

**Sorun 4: gRPC field number değiştirildi — serileştirme bozuldu**

```
Senaryo:
  Proto v1:
    message OrderRequest {
      string order_id = 1;
      string customer_id = 2;
      double amount = 3;
    }

  Backend: "customer_id yerine customer_uuid kullanalım" → field sildiler, yenisi 2. field number'a koyuldu:
    message OrderRequest {
      string order_id = 1;
      string customer_uuid = 2;   // YANLIŞ — field number 2 eskiden customer_id'ydi
      double amount = 3;
    }

  Sonuç:
    Protobuf serileştirme field NUMBER ile çalışır (isim değil).
    Eski client: customer_id → field 2 → binary gönderdi.
    Yeni server: field 2 = customer_uuid olarak parse etti.
    customer_uuid = "456" (eski customer_id değeri) → yanlış eşleşme.
    Sessiz veri bozulması — exception yok, yanlış veri geçti.

Protobuf breaking change kuralları:
  ✗ Field number değiştirme → binary uyumsuzluk (sessiz bozulma!)
  ✗ Field type değiştirme (string → int)
  ✗ Required alan ekleme
  ✓ Yeni field ekle (yeni field number ile)
  ✓ Alanı "reserved" yap (sil, ama number'ı koruma altına al)
  ✓ Yeni RPC ekle

  Doğru silme:
    message OrderRequest {
      string order_id = 1;
      reserved 2;                  // customer_id kaldırıldı, number rezervde
      reserved "customer_id";      // isim de rezervde (yanlışlıkla tekrar kullanma)
      string customer_uuid = 4;    // YENİ field → yeni number
      double amount = 3;
    }

  gRPC versioning:
    package com.example.orders.v1;
    package com.example.orders.v2;
    → Proto package ile major versiyon yönetimi.
    Schema Registry (Confluent veya Buf): proto değişikliklerini denetler.
```

---

**Sorun 5: Deprecation duyurusu yapılmadı — v1 ani sunset, partner entegrasyonları çöktü**

```
Senaryo:
  Backend: "v1 çok eski, artık desteklemiyoruz" → v1 endpoint'i kaldırdı.
  Duyuru: sadece Slack'te iç mesaj, partner'lara bildirim YOK.

  Partner A: hâlâ /api/v1/orders kullanıyor.
  T+0: v1 kaldırıldı → 404 Not Found.
  Partner A: siparişleri alamıyor → müşteri şikayeti → iş durdu.

  SLA ihlali: partner sözleşmesinde "API değişikliklerinde 3 ay önceden bildirim" yazıyor.
  Hukuki sorun: sözleşme ihlali → tazminat talebi.

Deprecation lifecycle standartı:

  1. Deprecation bildir (en az 6 ay, partner API'lar için 12 ay):
     Response header ile her istekte bildir:
     
     Deprecation: true
     Sunset: Sat, 31 Dec 2025 23:59:59 GMT
     Link: <https://api.example.com/v2/orders>; rel="successor-version"

  2. Email/dokümantasyon bildirimi:
     "v1 API 31 Aralık 2025'te kapatılacaktır. Migration guide: ..."
     Partner portalında duyuru.

  3. Monitoring: v1 trafiğini izle.
     Sunset tarihine yakın: hâlâ v1 kullanan client var mı?
     Varsa: direkt iletişim.

  4. Soft sunset:
     İlk birkaç gün: 410 Gone + yönlendirme mesajı.
     "Bu endpoint kaldırıldı. Lütfen v2 kullanın: https://..."
     Hard sunset: tamamen kaldır.

  Spring interceptor ile otomatik deprecation header:
  @Component
  class DeprecationHeaderInterceptor implements HandlerInterceptor {
      @Override
      public boolean preHandle(HttpServletRequest req,
                               HttpServletResponse res, Object handler) {
          if (req.getRequestURI().contains("/v1/")) {
              res.addHeader("Deprecation", "true");
              res.addHeader("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT");
          }
          return true;
      }
  }
```

---

## Bölüm 2: Versioning Stratejileri

### Gerçek Hayat Sorunları

---

**Sorun 6: Header versioning + CDN cache — tüm kullanıcılar aynı yanıtı aldı**

```
Senaryo:
  Header versioning:
    GET /api/orders/123
    Accept-Version: v1  (eski client)
    Accept-Version: v2  (yeni client)

  CDN (CloudFront/Nginx cache): URL bazlı cache.
  URL aynı → aynı cache key → aynı response!

  Timeline:
    v2 client → GET /api/orders/123 (Accept-Version: v2) → v2 response → CDN cache'e girdi.
    v1 client → GET /api/orders/123 (Accept-Version: v1) → CDN cache hit → v2 response döndü!
    v1 client: v2 formatı beklemiyordu → parse hatası.

Kök neden:
  Header versioning: CDN URL'yi cache key olarak kullanır.
  Farklı header → farklı response ama CDN bunları ayırt edemez (default config).

Çözüm 1: Vary header (CDN'ye "bu header farklıysa farklı cache" söyle):
  Response: Vary: Accept-Version
  CDN: cache key = URL + Accept-Version değeri.
  v1 ve v2 ayrı cache entry'leri.
  Dezavantaj: cache hit rate düşer (her kombinasyon ayrı entry).

Çözüm 2: URL path versioning (cache-friendly):
  /api/v1/orders/123 → tamamen farklı URL → CDN doğal ayırt eder.
  En basit çözüm. Cache sorunsuz.

Çözüm 3: API Gateway cache key özelleştirme:
  Kong/AWS API Gateway: custom cache key (URL + header).
  Header bazlı versioning + gateway seviyesinde doğru cache.

Sonuç: Public API + CDN + caching kritikse → URL path versioning.
Header versioning: internal API, CDN olmayan senaryolar için uygun.
```

---

**Sorun 7: API Gateway versioning — service içinde if/else versiyon karmaşası**

```
Senaryo:
  API Gateway: /v1/ ve /v2/ requestlerini aynı servis'e yönlendiriyor.
  Servis: X-API-Version header'ına göre if/else:

  @GetMapping("/orders/{id}")
  ResponseEntity<?> getOrder(@PathVariable String id,
                              @RequestHeader("X-API-Version") String ver) {
      Order order = orderService.findById(id);
      if ("1".equals(ver)) {
          return ResponseEntity.ok(OrderResponseV1.from(order));
      } else if ("2".equals(ver)) {
          return ResponseEntity.ok(OrderResponseV2.from(order));
      } else if ("3".equals(ver)) {
          return ResponseEntity.ok(OrderResponseV3.from(order));
      }
      // v4 geldi → buraya if ekle, test et, deploy et...
  }

  1 yıl sonra: v5, 6 versiyon → 6 if/else → metod 200 satır.
  Yeni geliştirici: "hangi versiyon ne döndürüyor?" → anlamak zor.
  Test: her versiyon için ayrı test case'ler → test süresi uzadı.

Daha iyi yaklaşım: Strateji pattern + ayrı controller:

  // Her versiyon kendi sınıfı:
  interface OrderResponseMapper {
      Object map(Order order);
  }

  @Component("v1")
  class OrderResponseMapperV1 implements OrderResponseMapper {
      public OrderResponseV1 map(Order order) { ... }
  }

  @Component("v2")
  class OrderResponseMapperV2 implements OrderResponseMapper {
      public OrderResponseV2 map(Order order) { ... }
  }

  // Controller:
  @Autowired Map<String, OrderResponseMapper> mappers;

  ResponseEntity<?> getOrder(String id, String version) {
      OrderResponseMapper mapper = mappers.getOrDefault(version, mappers.get("v2"));
      return ResponseEntity.ok(mapper.map(orderService.findById(id)));
  }

  // Veya: ayrı @RequestMapping controller sınıfları (en temiz)
  @RestController @RequestMapping("/api/v1/orders") class OrderControllerV1 { ... }
  @RestController @RequestMapping("/api/v2/orders") class OrderControllerV2 { ... }

  Eski versiyonu silmek: sadece ilgili controller ve DTO sınıfını sil.
  Etki alanı net → bağımlılık yok.
```

---

### Mülakat Soruları — Versioning Stratejileri

**Junior / Mid:**

1. API versioning nedir ve neden gereklidir?

   > **Beklened:** API versioning: mevcut client'ları bozmadan API'de değişiklik yapabilme stratejisi. Neden: backend her geliştiğinde tüm client'lar (mobil, partner, 3rd party) aynı anda güncellenemez. Mobil: App Store review 2-7 gün. Partner entegrasyon: kendi release döngüsü. Eski sürüm cihazlardaki app: güncellenemiyor. Versioning olmadan: API değişti → eski client bozuldu → incident. Versioning ile: v1 + v2 aynı anda çalışır → eski client v1, yeni client v2. Deprecation süreci: client'lara migration zamanı tanı → v1 sunset. Temel prensip: backend'in değişme özgürlüğü + client'ın bağımsızlığı.

2. URL path, header ve query param versioning stratejilerini karşılaştır.

   > **Beklened:** URL path (/api/v1/orders): en yaygın, URL'de görünür, cache-friendly (CDN URL bazlı), tarayıcıda test kolay. REST puristleri: "URL resource tanımlar, versiyon değil" der. Dezavantaj: URL kalabalıklaşır. Header (Accept-Version: v2): URL temiz, REST'e uygun. Dezavantaj: tarayıcıda test zor (header eklemek için araç gerekir), CDN cache sorunları (Vary header gerekir). Query param (/orders?version=2): test kolay, URL görece temiz. Dezavantaj: REST standartlarına en az uyan, cache bozabilir (?version query param cache key'e girmeyebilir). API Gateway versioning: versiyonu gateway'de yönet → servis saf kalır. Canary: v1:%80, v2:%20 traffic split. Seçim: public API + CDN → URL path. Internal API, clean URL → header. Test/prototip → query param.

3. Breaking change ile non-breaking change farkı nedir? Örneklerle açıkla.

   > **Beklened:** Non-breaking: yeni alan ekle (client bilmediği alanı ignore eder), yeni endpoint ekle, opsiyonel request parametresi ekle, hata mesajı değiştir (client parse etmiyorsa). Örnek: { "id": 1, "name": "Ali" } → { "id": 1, "name": "Ali", "email": "..." } — eski client email'ı ignore eder. Breaking: alan adı değiştir (customerId → customer.id), alan tipini değiştir (string → int), zorunlu request alanı ekle, response'tan alan sil, HTTP status code değiştir, URL path değiştir, davranış değiştir (sıralama değişimi). Sinir bozucu kural: enum'a yeni değer ekle → consumer switch/pattern-match yapıyorsa hata verir (unhandled case). Pratik test: "eski client bu değişiklikle bozulur mu?" → evet → breaking.

---

**Senior / Architect:**

4. API Gateway ile versioning yapmanın avantajları nelerdir? Ne zaman tercih edilir?

   > **Beklened:** Avantajlar: (1) Servis saf kalır — versiyon mantığı gateway'de, servis iş mantığına odaklanır. (2) Traffic shifting: canary deployment — v1:%80, v2:%20; yavaşça v2'ye geç. (3) Merkezi yönetim: birden fazla servisin versioning'i tek noktada. (4) Monitoring: hangi versiyon ne kadar kullanılıyor? (5) Backward compat: servis yeni versiyonu deploy etti, eski client'lar gateway üzerinden v1 almaya devam. Tercih edildiği yer: çok servis, çok versiyon; canary/blue-green deployment; versiyonu servisten izole etmek. Dikkat: gateway SPOF riski → HA konfigürasyonu. Servis içinde if/else versiyonlama: gateway bunu temizler ama servis tamamen tek formatta response dönmeli. Spring Cloud Gateway YAML: predicates + StripPrefix + AddRequestHeader ile konfigürasyon.

5. Consumer-driven contract testing API versioning sürecinde nasıl kullanılır?

   > **Beklened:** Problem: backend değişiklik yaptı, "breaking" olmadığını düşündü — ama bir consumer servis bozuldu. Consumer-driven contract: her consumer "ben bu API'den şunu bekliyorum" → Pact/Spring Cloud Contract ile kontrat yazar. Backend: kontratları çalıştırır → "eski consumer bozuldu mu?" → anında feedback. API versioning flow: (1) Consumer yeni field bekliyor: kontrat güncelle → backend yeni field ekle (non-breaking). (2) Backend alan silmek istiyor: kontrat çalıştır → "ShippingService bu alanı kullanıyor" → breaking! → versioning gerekli. (3) V1 sunset öncesi: v1 kontratları sil → hangi consumer hâlâ v1 kullanıyor? → bildir. Fayda: breaking change'i production'a gitmeden yakala. CI/CD: her deploy → kontrat testleri → bozulursa deploy durdur.

---

## Karma — Architect Seviyesi

6. **"Public REST API tasarlıyorsun. Versioning stratejisi ve deprecation lifecycle'ı nasıl yönetirsin?"**

   > **Beklened:** Strateji seçimi: public API + 3rd party → URL path versioning (/api/v1/, /api/v2/). CDN + cache: URL path → doğal. Tarayıcıda test: kolay. Deprecation lifecycle: GA: v2 çıktı → v1 tam destek sürüyor. Deprecation duyurusu: e-posta, developer portal, response header (Deprecation: true, Sunset tarih, Link: v2). Minimum süre: 6 ay consumer'lar için, partner/3rd party için 12 ay. Monitoring: v1 trafiği izle — hâlâ kullananlar var mı? Sunset yakın: aktif kullanıcılara direkt ulaş. Soft sunset: 410 Gone + migration message. Hard sunset: endpoint kaldır. Breaking change policy: breaking → mutlaka yeni major versiyon. Non-breaking → mevcut versiyona ekle. API changelog: her değişikliği belgele. Kod organizasyonu: ayrı controller sınıfları, Strateji pattern. Sunset'te: controller + DTO sil → temiz.

7. **"v1'i hâlâ kullanan partner var, ama v1 kodu teknik borç biriktiriyor. Ne yaparsın?"**

   > **Beklened:** Hemen kapatma: sözleşme ihlali + iş riski → hayır. Adımlar: (1) Analiz: partner ne kullanıyor? Tüm v1 endpoint'leri mi, birkaç tanesi mi? v2'ye geçiş engeli nedir? (2) Partner ile görüşme: migration maliyeti nedir? Bizden teknik destek lazım mı? Deadline üzerinde anlaş. (3) Adapter katmanı: v1 endpoint → v2 business logic'e yönlendir (v1 controller sadece dönüşüm yapar). Teknik borç azalır: v1 controller ince wrapper, iş mantığı v2'de. (4) Sunset tarihi: partner ile yazılı anlaşma → "X tarihinde v1 kapanıyor." (5) Response header: her v1 isteğinde Deprecation + Sunset. Monitoring alert: partner geçtikten sonra v1 trafiği sıfıra indi mi? (6) Geçtikten sonra: v1 kodunu sil. Temiz teknik borç kapatma. Ana prensip: partner ilişkisi + teknik sağlık dengesi. Hız yerine güven.

8. **"gRPC API'de bir alanı kaldırmak istiyorsun. Nasıl yaparsın?"**

   > **Beklened:** Yanlış: field'ı proto'dan sil veya field number'ı yeniden kullan → binary uyumsuzluk, sessiz veri bozulması. Doğru adımlar: (1) Alanı önce deprecated yap: comment veya özel attribute ile belgele. (2) Consumer'ları migrate et: hangi servis bu alanı kullanıyor? → bulup güncelle. (3) Tüm consumer geçti → proto'dan sil ama reserved koru: reserved 5; reserved "old_field_name"; Bu sayede field number ve isim yanlışlıkla yeniden kullanılmaz. (4) Yeni alan gerekiyorsa: her zaman yeni field number → eski number asla recycle etme. (5) Proto versioning (major değişim): package com.example.orders.v1 → package com.example.orders.v2. (6) Schema Registry (Buf/Confluent): CI/CD'de proto değişiklik denetimi → breaking change'i otomatik tespit. gRPC wire format: alan ismi değil, field number kritik → bu hatayı yapmak kolay, etkisi sessiz ve tehlikeli.
