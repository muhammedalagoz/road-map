# 06c — BFF Pattern: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Over-fetching & Under-fetching

### Gerçek Hayat Sorunları

---

**Sorun 1: Generic API + mobil — over-fetching, yüksek bant genişliği tüketimi**

```
Senaryo:
  E-ticaret mobil uygulaması. Generic API: sipariş listesi endpoint'i.
  Web dashboard için tasarlandı: her siparişte müşteri detayı, fatura,
  ürün görselleri (base64), 50 sipariş aynı anda.

  Mobil uygulama aynı endpoint'i kullandı:
    Response boyutu: 2.4MB (50 sipariş × müşteri + ürün görseli)
    3G bağlantı: 2.4MB → 8 saniye yükleme süresi.
    Kullanıcı: sipariş listesi 8 saniye sonra açıldı → uygulama "donmuş" hissi.
    Görüntülenen veri: sipariş no, tutar, durum → 3 alan.
    Kullanılan veri: response'ın %3'ü. Kalan %97 → çöpe gitti.

Kök neden:
  Web ihtiyacı ≠ Mobil ihtiyacı.
  Generic API: en büyük ortak paydayı karşılar → mobil "fazla" alır.

Mobile BFF çözümü:
  Sadece mobil'in ihtiyacı:
    - orderId, statusLabel, totalAmount, orderDate
    - Sayfalama: 10'ar 10'ar
    - Görseller yok (lazy load, ayrı endpoint)

  Mobile BFF response: ~8KB (10 sipariş × 800 byte)
  Yükleme süresi: 3G → 80ms.
  Performans: 100× iyileşme.

  @GetMapping("/mobile/api/orders")
  PagedResponse<MobileOrderSummary> getOrders(
          @RequestParam(defaultValue = "0") int page,
          @RequestParam(defaultValue = "10") int size) {
      return orderClient.getOrders(userId, PageRequest.of(page, size))
          .map(order -> MobileOrderSummary.builder()
              .orderId(order.getId())
              .statusLabel(localizeStatus(order.getStatus()))
              .totalAmount(formatCurrency(order.getTotal()))
              .orderDate(formatDate(order.getCreatedAt()))
              .build());
  }
```

---

**Sorun 2: Generic API + web — under-fetching, N+1 frontend çağrısı**

```
Senaryo:
  Web dashboard: sipariş listesi + her siparişin müşteri adı + fatura durumu.
  Generic OrderService API: sadece sipariş verisini döndürüyor.

  Frontend (React):
    1. GET /api/orders → 50 sipariş listesi
    2. Her sipariş için:
       GET /api/users/{customerId}     → 50 çağrı (müşteri)
       GET /api/payments/{orderId}     → 50 çağrı (fatura)
    Toplam: 1 + 50 + 50 = 101 HTTP çağrısı!

  Sonuç:
    Network waterfall: paralel bile olsa 50 çağrı overhead.
    Rate limit: UserService → 100 req/s limit → dashboard açılınca limit aşıldı.
    Web sayfası yükleme: 3.2 saniye (101 çağrı).
    Her UserService/PaymentService: 50 çağrı yükü → diğer kullanıcılar etkilendi.

Web BFF çözümü:
  Tek endpoint: tüm veriyi aggrege ederek döndür.
  BFF → paralel CompletableFuture ile servisleri çağırır.

  @GetMapping("/web/api/orders/dashboard")
  DashboardResponse getDashboard() {
      List<Order> orders = orderClient.getRecentOrders(userId, 50);

      // Batch fetch (N+1 önleme):
      Set<String> customerIds = orders.stream()
          .map(Order::getCustomerId).collect(toSet());
      Map<String, CustomerDto> customers = userClient.getBatch(customerIds);

      Set<String> orderIds = orders.stream()
          .map(Order::getId).collect(toSet());
      Map<String, InvoiceDto> invoices = paymentClient.getBatchInvoices(orderIds);

      return orders.stream()
          .map(order -> OrderDashboardItem.builder()
              .order(order)
              .customer(customers.get(order.getCustomerId()))
              .invoice(invoices.get(order.getId()))
              .build())
          .collect(toDashboardResponse());
  }
  // Toplam: 3 çağrı (sipariş + batch müşteri + batch fatura)
  // Web: 3.2s → 180ms.
```

---

**Sorun 3: Monolithic BFF — tek BFF her client'a hizmet, bottleneck**

```
Senaryo:
  Başlangıçta: tek BFF, web + mobil + partner hepsini karşılıyor.
  Gerekçe: "tek BFF, bakımı kolay."

  6 ay sonra:
    Web: dashboard, raporlama, admin panel → karmaşık aggregation.
    Mobil: push notification, device register, deep link.
    Partner: rate limiting, API key auth, webhook.

  BFF kodu:
    6000+ satır tek sınıf.
    Web özelliği değişti → tüm BFF deploy edildi → mobil client etkilendi.
    Partner rate limit yükseltildi → BFF restart → web dashboard 10s down.
    Test: web testi mobil kodunu etkiler mi? Birbirinden bağımsız test edilemiyor.

Monolithic BFF anti-pattern:
  "BFF büyüdükçe özgün amacını kaybeder."
  Web, mobil, partner → farklı deployment, farklı scale ihtiyacı.

Doğru: her client için ayrı BFF.
  Web BFF:     web dashboard + raporlama
  Mobile BFF:  iOS + Android (ortak mobil deneyim)
  Partner BFF: 3rd party entegrasyon

  Avantajlar:
    Bağımsız deploy: Web BFF güncellendi → mobil etkilenmez.
    Bağımsız scale: Black Friday → Mobile BFF 20 pod, Web BFF 5 pod.
    Bağımsız test: her BFF kendi test suite'i.
    Farklı teknoloji: Web BFF Java, Mobile BFF Node.js (frontend ekibi yazar).

  Dezavantaj: daha fazla servis → daha fazla operasyonel yük.
  Tradeoff: team ownership + independent deployment > operasyonel yük.
```

---

**Sorun 4: BFF'te iş mantığı birikti — "experience layer" iş kuralı katmanı oldu**

```
Senaryo:
  Mobile BFF başlangıçta: sadece veri şekillendirme.
  Zamanla eklenen "küçük" iş mantıkları:
    - Sipariş iptal edilebilir mi? (durum + süre kontrolü)
    - Kampanya indirimi hesaplama
    - Kullanıcı segmentasyonu (premium mi?)
    - Fraud kontrolü (ödeme öncesi)

  Mobile BFF: 4000+ satır iş mantığı.
  
  Sorunlar:
    Web BFF: aynı "iptal edilebilir mi?" mantığını tekrar yazdı.
    Partner BFF: kampanya hesaplamasını tekrar yazdı.
    3 BFF'te 3 farklı implementasyon → tutarsız davranış.
    OrderService'te ise bu mantık yok → kim doğru?

BFF'in doğru sorumluluğu:
  BFF = "experience layer" (deneyim katmanı):
    ✓ Veri şekillendirme (mobil için küçük, web için büyük)
    ✓ Aggregation (birden fazla servis → tek response)
    ✓ Protokol dönüşümü (REST ↔ gRPC, format dönüşümü)
    ✓ Client-specific auth/rate limiting
    ✓ Lokalizasyon, format (para birimi, tarih)

  BFF'te OLMAMALI:
    ✗ İş kuralı (iptal koşulları, kampanya mantığı)
    ✗ Veri doğrulama (domain mantığı)
    ✗ DB erişimi (BFF stateless olmalı)
    ✗ Fraud/risk hesaplama

  Düzeltme:
    "Sipariş iptal edilebilir mi?" → OrderService'e taşı.
    BFF: OrderService'e sorar → sonucu client'a şekillendirir.
    Tek kaynak of truth: iş mantığı domain servisinde.
```

---

**Sorun 5: BFF authentication — her BFF ayrı auth, tutarsızlık**

```
Senaryo:
  3 BFF: Web, Mobile, Partner.
  Her BFF kendi auth mantığını yazdı.
    Web BFF: session cookie tabanlı.
    Mobile BFF: JWT Bearer token.
    Partner BFF: API key.
  
  Sonuç:
    Token yenileme (refresh): Mobile BFF'te farklı, Web BFF'te farklı.
    Revocation: bir kullanıcı logout → Mobile BFF token'ı geçersiz kılıyor.
      Web BFF: session'ı biliyor mu? Hayır → kullanıcı web'de hâlâ giriş yapabildi.
    Audit log: kim hangi BFF'ten ne yaptı? → farklı format, birleştirilemiyor.
    Partner API key rotation: Partner BFF'te → User servisi bilmiyor.

BFF auth organizasyonu:
  Centralized auth: API Gateway veya Auth Servis → token doğrulama.
  BFF: token'ı alır, API Gateway'e veya Auth Servis'e doğrulatır.
    Authorization header → Introspect → kim bu kullanıcı?
  
  Her BFF'te TEKRAR auth yazmak yerine:
    1. API Gateway: tüm BFF'lerin önünde → token validate.
       Gateway: "token geçerli, user_id=123" → header'a koy → BFF'e ilet.
       BFF: sadece X-User-Id header'ını okur.
    
    2. Shared auth library (iç kütüphane):
       AuthFilter: tüm BFF'lerde aynı → merkezi yönetim.
    
    3. Service mesh (Istio/Linkerd):
       mTLS + policy → service identity doğrulama BFF dışında.

  Partner BFF API key:
    API key → Auth Servis → partner_id → BFF'e header ile ilet.
    Revocation: Auth Servis'te → tüm BFF'ler etkilenir.
```

---

**Sorun 6: GraphQL BFF N+1 sorunu — her field ayrı servis çağrısı**

```
Senaryo:
  GraphQL BFF: her Order için Customer ve Invoice lazy load.

  @SchemaMapping(typeName = "Order")
  Customer customer(Order order) {
      return userClient.getCustomer(order.getCustomerId()); // Her sipariş için!
  }

  @SchemaMapping(typeName = "Order")
  Invoice invoice(Order order) {
      return paymentClient.getInvoice(order.getId()); // Her sipariş için!
  }

  Query: 50 sipariş + her birinin customer + invoice.
  Çağrı sayısı: 1 (orders) + 50 (customers) + 50 (invoices) = 101 çağrı.
  Latency: 50 ayrı network round-trip.

DataLoader çözümü (GraphQL N+1 önleme):
  DataLoader: field çözümleyicilerini batching ile çalıştırır.
  "Bu 50 siparişin customer'larını topluca çek."

  @Bean
  DataLoaderRegistry dataLoaderRegistry() {
      DataLoader<String, CustomerDto> customerLoader = DataLoader.newDataLoader(
          customerIds -> {
              // Tüm customer ID'leri toplu çek
              Map<String, CustomerDto> customers = userClient.getBatch(customerIds);
              return CompletableFuture.completedFuture(
                  customerIds.stream().map(customers::get).toList()
              );
          }
      );
      DataLoaderRegistry registry = new DataLoaderRegistry();
      registry.register("customerLoader", customerLoader);
      return registry;
  }

  @SchemaMapping(typeName = "Order")
  CompletableFuture<CustomerDto> customer(Order order, DataFetchingEnvironment env) {
      DataLoader<String, CustomerDto> loader = env.getDataLoader("customerLoader");
      return loader.load(order.getCustomerId()); // Batch'e eklenir
  }
  // Çalışma: 50 sipariş → DataLoader → tek getBatch çağrısı.
  // 101 çağrı → 3 çağrı.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. BFF pattern nedir? Neden ihtiyaç duyulur?

   > **Beklened:** BFF (Backend for Frontend): her client tipi için ayrı API katmanı. Web, mobil, partner — her birinin veri ihtiyacı farklı. Generic API: herkese aynı response → mobil fazla alır (over-fetching), web az alır (under-fetching). Over-fetching: mobil 2.4MB response, sadece 3 alanı kullanır → bant genişliği israfı, yavaş yükleme. Under-fetching: web dashboard 50 sipariş için 101 ayrı çağrı yapmak zorunda → N+1 problemi. BFF çözümü: Web BFF → büyük aggregated response (dashboard için). Mobile BFF → küçük, sayfalı, optimize response. Partner BFF → rate limiting, API key auth, webhook. Her BFF o client'ın "dilini" konuşur.

2. "Monolithic BFF" anti-pattern nedir? Nasıl önlenir?

   > **Beklened:** Monolithic BFF: tek BFF tüm client tiplerini (web + mobil + partner) karşılar. Başlangıçta kolay, zamanla büyür. Sorunlar: web değişikliği → tüm BFF deploy → mobil etkilendi. Mobil için scale → web de scale edildi (gereksiz kaynak). Test: birbirinden bağımsız test edilemiyor. Bağımlılık: web özelliği mobil'i bozabilir. Önlem: her client için ayrı BFF servisi. Web BFF, Mobile BFF, Partner BFF → ayrı kod tabanı, ayrı deploy, ayrı scale. Tradeoff: daha fazla servis → daha fazla operasyonel yük. Ama: team ownership (frontend ekibi kendi BFF'ini yönetir) + bağımsız deployment → değer.

3. BFF'te hangi sorumluluklar olmalı, hangisi olmamalı?

   > **Beklened:** OLMALI: veri şekillendirme (web büyük, mobil küçük), aggregation (birden fazla servis → tek response), protokol dönüşümü (gRPC → REST, format), client-specific auth/rate limiting, lokalizasyon ve format (tarih, para birimi). OLMAMALI: iş mantığı (iptal koşulları, kampanya hesabı → domain servisinde olmalı), DB erişimi (BFF stateless), fraud/risk kontrolü, veri doğrulama (domain kuralı). Neden: iş mantığı BFF'te → 3 BFF'te 3 ayrı implementasyon → tutarsızlık. Bir BFF güncellendi, diğerleri unutuldu → bug. İş mantığı domain servisinde → tek kaynak, tüm BFF'ler oradan alır. BFF = "deneyim katmanı" — teknik şekillendirme, domain mantığı değil.

---

**Senior / Architect:**

4. BFF pattern ile GraphQL'i karşılaştır. Hangisi ne zaman tercih edilir?

   > **Beklened:** BFF (REST): ayrı servisler, her client'a özel endpoint. Avantaj: tam kontrol, cache-friendly (GET endpoint), basit client. Dezavantaj: yeni alan her BFF'te güncelleme, over/under-fetching riski. GraphQL BFF: tek endpoint, client istediği alanları seçer. Web: customer + invoice + items → 1 query. Mobil: sadece id + status → 1 query. Avantaj: over/under-fetching yok, self-documenting schema, hızlı iterasyon (backend değişmeden client yeni alan alabilir). Dezavantaj: N+1 sorunu (DataLoader gerekir), cache zor (POST query), öğrenme eğrisi, karmaşık query → güvenlik riski (depth limit). Seçim: frontend ekibi hızlı iterasyon istiyor, veri ihtiyacı çok değişken → GraphQL. Basit, stabil API, CDN cache kritik, REST tooling geniş → REST BFF. Pratik: GraphQL BFF = "tek BFF hem web hem mobil" — ayrı BFF gerekmeyebilir.

5. BFF'te CompletableFuture ile paralel servis çağrısı neden önemlidir? Dikkat edilmesi gerekenler?

   > **Beklened:** Neden: Web BFF dashboard → 1 sipariş için 3 servis (customer + invoice + products). Sıralı: 3 × 100ms = 300ms. Paralel: max(100ms, 100ms, 100ms) = 100ms. 50 sipariş → paralel: 100ms vs 15 saniye. CompletableFuture.supplyAsync ile her çağrı ayrı thread'de. Dikkat edilmesi gerekenler: (1) Thread pool boyutu: varsayılan ForkJoinPool → CPU-bound için. IO-bound servis çağrısı → custom Executor (IO thread pool) kullan. (2) Timeout: bir servis yavaşlarsa CompletableFuture.orTimeout() ile sınırla. (3) Hata yönetimi: bir servis hatası tüm response'ı bozmamalı. exceptionally() ile fallback: invoice gelmezse null dön, dashboard göster. (4) Backpressure: 50 sipariş × 3 çağrı = 150 çağrı aynı anda → downstream servis yük. Batch fetch (getBatch) ile N+1'i önle: 3 çağrı yeterli. (5) Bulkhead: bir servisteki yavaşlama BFF'in thread pool'unu tüketmesin → Resilience4j Bulkhead.

6. Partner BFF ile Web/Mobile BFF arasındaki temel tasarım farkları nelerdir?

   > **Beklened:** Partner BFF farklılıkları: (1) Auth mekanizması: API key (X-Partner-Key header), OAuth2 client credentials. Web/Mobile: user token (JWT, session). (2) Rate limiting: partner bazında (partner A: 1000 req/h, partner B: 5000 req/h). Web/Mobile: kullanıcı bazında. (3) API versioning: partner entegrasyonları değiştirilmesi zor → strict versioning (v1 yıllarca desteklenir). Deprecation: 12 ay minimum. (4) External ID mapping: partner kendi sipariş ID'sini kullanır → internal ID'ye map. (5) Webhook: partner'a event bildirimi (sipariş durumu değişti). (6) SLA: partner SLA sözleşmesi → uptime garantisi, latency SLA. (7) Data masking: partner'a gösterilmemesi gereken alanlar (iç müşteri ID, marj bilgisi). (8) Audit log: her partner çağrısı loglanmalı (sözleşme uyumu, debug). Web/Mobile BFF bunların çoğuna ihtiyaç duymaz — daha basit, daha hızlı iterasyon.

---

## Karma — Architect Seviyesi

7. **"Tek bir generic API'den BFF mimarisine geçiyorsun. Migration planın nedir?"**

   > **Beklened:** (1) Analiz: mevcut API'yi hangi client'lar nasıl kullanıyor? Traffic log'larından: web hangi endpoint'leri, mobil hangileri, over/under-fetching nerede? (2) Sınır çizimi: kaç BFF? Web + Mobile + Partner → 3 BFF. İOS ve Android aynı mı? → tek Mobile BFF (genellikle evet). (3) Öncelik: en çok pain olan client → önce ona BFF yaz. Mobil: over-fetching → Mobile BFF ilk. (4) Strangler Fig: generic API'yi hemen kapatma. BFF → generic API'yi çağırır (proxy). Zamanla: BFF → direkt microservice'leri çağırır. Generic API trafiği sıfıra iner → kapat. (5) Client migration: mobil app — App Store review süreci. Feature flag: yeni BFF endpoint'ini toggle ile aç/kapat. (6) Monitoring: BFF → latency, error rate, upstream servis hataları. Generic API → azalan trafik (migration ilerledikçe). (7) Ownership: her BFF'i kim yönetecek? Frontend ekibi mi, backend mi? Netleştir. (8) Rollback planı: BFF sorunluysa → gateway routing ile generic API'ye geri dön.

8. **"BFF mi, API Gateway mi, GraphQL mi? Hangi senaryoda hangisini seçersin?"**

   > **Beklened:** API Gateway: routing, auth, rate limiting, SSL termination → her mimaride şart. BFF değil, altyapı. BFF: client'a özel aggregation, şekillendirme. Gateway'in üzerinde veya yanında. Seçim: farklı client tipleri, çok farklı veri ihtiyacı → BFF. Basit routing + auth → sadece gateway yeterli. GraphQL: tek endpoint, client-driven query. BFF'e alternatif veya birlikte. Seçim kılavuzu: Web + Mobil + Partner, belirgin farklı ihtiyaç, büyük ekip → ayrı BFF. Küçük ekip, benzer ihtiyaç → GraphQL BFF (tek servis, her client kendi query'sini yazar). Stabil API, CDN cache kritik, REST olgunluğu → REST BFF. Hızlı iterasyon, frontend-driven → GraphQL. Kombinasyon: API Gateway + Mobile BFF (REST) + Web (GraphQL) mümkün. Tek doğru cevap yok — ekip yapısı, client sayısı, cache ihtiyacı belirler. Conway's Law: organizasyon yapısı mimariyi şekillendirir.
