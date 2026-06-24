# 05d — Bulkhead, Timeout & Retry: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Bulkhead Pattern

### Gerçek Hayat Sorunları

---

**Sorun 1: Bulkhead yok — ödeme servisi yavaşladı, tüm sistem çöktü**

```
Senaryo:
  E-ticaret platformu: OrderService tek bir thread pool (200 thread).
  Downstream: PaymentService, InventoryService, ShippingService.
  Black Friday: PaymentService 3rd party gateway yükünden dolayı yavaşladı.
  Her ödeme isteği 30sn thread tutuyor.

Cascade failure:
  T+0s:  200 thread var. PaymentService yavaşladı.
  T+30s: 200 thread tümü PaymentService'i bekliyor.
  T+30s: InventoryService ve ShippingService isteği geliyor → thread yok → kuyruk.
  T+60s: Kuyruk doldu → TimeoutException.
  T+60s: Kullanıcılar sipariş VEREMİYOR (ürün listeleme de çalışmıyor).
  
  Kök neden: Ödeme sorunu tüm thread pool'u tüketti.

Bulkhead ile izolasyon:
  PaymentService pool:   10 thread (max bekleme: 5)
  InventoryService pool: 30 thread
  ShippingService pool:  20 thread
  Web request pool:      100 thread (ana pool)

  Aynı senaryo:
  T+0s:  PaymentService 10 thread → doluyor.
  T+10s: 11. ödeme isteği → BulkheadFullException → hızlı fail → fallback.
  T+10s: InventoryService, Shipping, listeleme → kendi pool'larında çalışmaya devam.
  Kullanıcılar ürünü sepete ekleyebilir, siparişi görebilir.
  Sadece ödeme geçici olarak degraded → "Ödeme şu an kullanılamıyor" mesajı.
```

---

**Sorun 2: Bulkhead pool boyutu yanlış — çok küçük veya çok büyük**

```java
// Senaryo A: Pool çok küçük → meşru istekler reddediliyor
ThreadPoolBulkheadConfig tooSmallConfig = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(2)   // Sadece 2 thread payment için
    .queueCapacity(0)       // Kuyruk yok
    .build();
// Normal yük: 50 eş zamanlı ödeme isteği gelebilir
// 2 thread dolu → 48 istek BulkheadFullException → %96 istek reddedildi

// Senaryo B: Pool çok büyük → izolasyon yok
ThreadPoolBulkheadConfig tooBigConfig = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(200)  // Ana pool kadar büyük
    .build();
// Payment servis yavaşlayınca 200 thread yine tükeniyor → bulkhead anlamsız

// Doğru boyutlandırma:
// Formül: Little's Law = Throughput × Response Time
// PaymentService: ortalama 20 istek/sn, ortalama 200ms response
// Gerekli thread = 20 × 0.2 = 4 thread (normal yük)
// Peak × 2 + buffer = 4 × 2 + 2 = 10 thread

ThreadPoolBulkheadConfig correctConfig = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)       // Peak yük + buffer
    .coreThreadPoolSize(4)       // Normal yük
    .queueCapacity(5)            // Ani spike için küçük kuyruk
    .keepAliveDuration(Duration.ofSeconds(20))
    .build();

// Boyutu load test ile doğrula:
// Actuator metric: resilience4j.bulkhead.available.concurrent.calls
// Pool doluluk oranı > %80 → büyüt
// Pool doluluk oranı < %20 → küçült (kaynak israfı)
```

---

**Sorun 3: Thread pool bulkhead yerine semaphore seçildi — reactive stack'te deadlock**

```java
// Senaryo: WebFlux (non-blocking) ile semaphore bulkhead kullanıldı.
// Semaphore: event loop thread'ini bloke eder.

// YANLIŞ: WebFlux + Semaphore bulkhead
@Bulkhead(name = "payment", type = Bulkhead.Type.SEMAPHORE)  // default SEMAPHORE
Mono<PaymentResult> charge(Order order) {
    return paymentWebClient.post("/charge")
        .retrieve()
        .bodyToMono(PaymentResult.class);
    // Mono dönüyor ama semaphore acquires → event loop thread bloke!
    // Event loop: az thread (CPU sayısı). Bloke thread → sistem donuyor.
}

// DOĞRU A: WebFlux + Thread Pool Bulkhead (ayrı thread pool'da çalıştır)
@Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL)
CompletableFuture<PaymentResult> charge(Order order) {
    return CompletableFuture.supplyAsync(
        () -> paymentClient.charge(order)  // blocking call ayrı pool'da
    );
}

// DOĞRU B: WebFlux için semaphore + async (event loop bloke etmeden)
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(25)
    .maxWaitDuration(Duration.ofMillis(0))  // hemen fail (bekleme yok)
    .build();
// Non-blocking kullanım: Bulkhead.decorateSupplier + Schedulers.boundedElastic

// Kural:
// Blocking stack (Spring MVC) → ThreadPool veya Semaphore (her ikisi çalışır)
// Non-blocking stack (WebFlux) → ThreadPool Bulkhead (veya reactive operator)
```

---

### Mülakat Soruları — Bulkhead

**Junior / Mid:**

1. Bulkhead pattern nedir? Gemi perdesinden esinlenme ne anlama gelir?

   > **Beklened:** Gemide su geçirmez bölmeler: bir bölme su alsa diğerleri etkilenmez — gemi batar ama bölmeli yapı hayatta kalır. Yazılımda: her downstream servis veya işlem kategorisi için ayrı kaynak havuzu (thread pool, semaphore, connection pool). Bir servis yavaşlarsa sadece o havuz dolar — diğer servisler kendi havuzlarından çalışmaya devam eder. Olmadan: tek thread pool → bir servis yavaşlar → tüm thread'ler o servisi bekler → diğer servisler de cevap veremez → cascade failure.

2. Thread pool bulkhead ile semaphore bulkhead farkı nedir?

   > **Beklened:** Semaphore: eş zamanlı çağrı sayısını sınırlar. Çağrıyı yapan thread aynı thread (caller thread'i bloke eder). Lightweight — ek thread yok. Thread pool: çağrıyı ayrı thread pool'unda çalıştırır. Caller thread serbest kalır (non-blocking). Timeout desteği var (TimeLimiter ile). Seçim: Blocking stack (MVC, JDBC) + timeout lazım → thread pool. Non-blocking (WebFlux) veya lightweight → semaphore (ama caller thread'ini bloke etme dikkat). Kural: çağrı kısa ve CPU-bound → semaphore. Uzun, IO-bound → thread pool.

3. `BulkheadFullException` alındığında ne yapılmalıdır?

   > **Beklened:** Hızlı fail: kullanıcıya "sistem meşgul" mesajı ver — yavaş cevap bekletme. Fallback metod: degraded deneyim sun (cache'den oku, default değer). Kuyruk: isteği async işleme kuyruğuna at (iş kritikse). Metric kayıt: `bulkhead.call.rejected` sayacını artır — alert. Circuit breaker ile birlikte: yeterli rejection → CB açılır — servis stabilize olana kadar tampon. Kesinlikle yapma: exception yut ve boş yanıt dön (kullanıcı ne olduğunu bilmez).

---

**Senior / Architect:**

4. Bulkhead pool boyutunu nasıl belirlersin? Little's Law nedir?

   > **Beklened:** Little's Law: `L = λ × W`. L = sistemdeki ortalama öğe sayısı (thread), λ = throughput (istek/sn), W = ortalama işlem süresi. Örnek: payment servisi 50 istek/sn, ortalama 100ms → gerekli thread = 50 × 0.1 = 5. Peak × 2 buffer = 10 thread. Adımlar: (1) Prometheus/Actuator ile P95 response time ölç. (2) Peak throughput tahmin et. (3) Little's Law ile hesapla. (4) Load test ile doğrula. (5) Pool doluluk oranını izle: sürekli >%80 → büyüt, <%20 → küçült. Tuzak: tahminle belirle, load test etmeden production'a çıkma.

5. Bulkhead, Circuit Breaker ve Rate Limiter birbirinden nasıl farklıdır? Birlikte nasıl kullanılırlar?

   > **Beklened:** Bulkhead: kaynak izolasyonu — bir servisin kaç eş zamanlı kaynağa erişeceğini sınırla. Cascade failure önle. Circuit Breaker: health durumu — hata oranı yükselince kapat, hızlı fail ver. Servis sağlığına tepki. Rate Limiter: hız sınırı — saniyede kaç istek geçsin. Aşım → 429. Birlikte: Rate Limiter (giriş kapısı) → Bulkhead (kaynak havuzu) → Circuit Breaker (downstream sağlığı) → TimeLimiter (bitiş süresi). Rate Limiter olmadan: flood → bulkhead dolar. CB olmadan: downstream down olsa bile bulkhead thread'leri hep dolu bekler. Üçü tamamlayıcı, biri diğerinin yerini tutmaz.

---

## Bölüm 2: Timeout Stratejileri

### Gerçek Hayat Sorunları

---

**Sorun 4: Timeout yok — yavaş servis tüm thread'leri tüketti**

```java
// Senaryo: InventoryService bazen 120sn yanıt veriyor (eski legacy sistem).
// Timeout tanımlanmadı.

// YANLIŞ: Timeout yok
RestTemplate restTemplate = new RestTemplate();  // default: sonsuz bekleme
// veya
HttpClient httpClient = HttpClient.create();  // responseTimeout set edilmedi

// Akış:
// T+0s:  Request geldi, inventory check başladı
// T+120s: Yanıt geldi (geç de olsa)
// Bu sürede: thread bloke → 200 thread varsa 200 eş zamanlı bloke mümkün
// Thread pool tükendi → yeni istekler reject → sistem yanıt veremiyor

// DOĞRU: Her dış çağrıda timeout zorunlu
@Bean
RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(2000);  // 2sn bağlantı kur
    factory.setReadTimeout(3000);     // 3sn yanıt al
    return new RestTemplate(factory);
}

// WebClient (Reactive):
@Bean
WebClient webClient() {
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
                .responseTimeout(Duration.ofSeconds(3))
        ))
        .build();
}

// Feign Client:
feign:
  client:
    config:
      inventory-service:
        connect-timeout: 2000
        read-timeout: 3000

// HikariCP (DB):
spring.datasource.hikari.connection-timeout=5000
spring.datasource.hikari.validation-timeout=2000
```

---

**Sorun 5: Cascade timeout yanlış — iç servis timeout'u dış servisten uzun**

```
Senaryo: A → B → C → D zinciri
  A'nın timeout'u: 5sn
  B'nin timeout'u: 4sn
  C'nin timeout'u: 6sn  ← YANLIŞ!
  D'nin timeout'u: 8sn  ← YANLIŞ!

Sorun:
  A kullanıcı isteği 5sn'de timeout verir.
  B, A'nın isteğini karşılamak için C'yi çağırdı.
  C timeout'u 6sn → A zaten 5sn'de timeout verdi, kullanıcıya hata döndü.
  Ama B hâlâ C'yi bekliyor (4sn) → C hâlâ D'yi bekliyor (6sn).
  Sonuç: A timeout'unu verdi ama B, C, D thread'leri hâlâ çalışıyor → kaynak israfı.

Doğru cascade timeout tasarımı:
  D timeout: 1sn   (en içteki, en küçük)
  C timeout: 1.5sn (D + buffer)
  B timeout: 2.5sn (C + buffer)
  A timeout: 4sn   (B + buffer + kendi işleme süresi)

  Formül: timeout[n] = timeout[n+1] × retry_count + backoff + processing_buffer
  
  Kural: Dış servis timeout'u iç servisten HER ZAMAN büyük olmalı.
  Aksi halde: dış servis hata verir ama iç thread'ler çalışmaya devam eder → "orphan request".

gRPC deadline: otomatik propagate edilir.
  stub.withDeadlineAfter(4000, MILLISECONDS).getInventory(req)
  → gRPC bu deadline'ı iç çağrılara otomatik geçirir → orphan yok.

HTTP: X-Request-Deadline header ile manuel propagate et.
```

---

**Sorun 6: Timeout değeri çok kısa — P99 spike'ında gereksiz hata**

```
Senaryo:
  InventoryService P99 latency: 200ms (normalse).
  Timeout: 150ms (P99'un altında!).
  Normal günde: %5 timeout → false positive.
  Black Friday: GC pause, yüksek yük → P99 400ms → %60 timeout.

Timeout çok kısa sorunları:
  - Servis sağlıklıyken bile hata üretir (false positive).
  - Retry tetiklenir → servis üzerine daha fazla yük → snowball.
  - Circuit Breaker açılır → servis sağlıklıyken degraded mode.

Timeout değeri nasıl belirlenmeli:
  1. P99 latency ölç (Prometheus histogram_quantile):
     histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
     
  2. Timeout = P99 × 2-3:
     P99: 200ms → timeout: 400-600ms
     (Çok sıkı: false positive. Çok gevşek: yavaş servis uzun bekletir.)
     
  3. SLO'ya göre ayarla:
     "Kullanıcı 3sn'den fazla beklememeli" → tüm zincir budget 3sn.
     Her downstream çağrısına bu budget'tan pay ver.

  4. Monitoring: timeout rate'i izle.
     Timeout rate > %1 → ya timeout değeri çok kısa ya da servis gerçekten yavaş.

P99 spike güvencesi:
  Timeout = P99 + GC_pause_buffer
  Java GC pause: max 200ms (G1 ile) → 200ms ekle.
  timeout = 200ms (P99) + 200ms (GC buffer) = 400ms → güvenli.
```

---

### Mülakat Soruları — Timeout

**Junior / Mid:**

6. Connection timeout ile read timeout farkı nedir? Hangisi genellikle daha küçük olur?

   > **Beklened:** Connection timeout: TCP bağlantısı kurulana kadar bekleme — SYN→SYN-ACK→ACK handshake süresi. Sunucu yanıt vermiyorsa veya erişilemiyorsa bu timeout tetiklenir. Genellikle 1-5 sn. Read timeout (socket timeout): bağlantı kuruldu, sunucu isteği aldı, cevap üretene kadar bekleme. Sunucu processing time + network. İş mantığına bağlı (50ms - birkaç dakika). Connection timeout genellikle daha küçük: ya bağlanır ya bağlanamaz, 3sn içinde anlaşılır. Read timeout: servise göre — basit query 100ms, rapor üretimi 30sn.

7. Timeout değeri nasıl belirlenir? P99 nedir?

   > **Beklened:** P99 (99. yüzdelik): tüm isteklerin %99'u bu süreden kısa cevap verdi. Örnek: P99=200ms → 100 istekten 99'u 200ms'den kısa, 1'i daha uzun. Timeout belirleme: (1) Prometheus/Micrometer ile P99 latency ölç. (2) Timeout = P99 × 2-3 (buffer için). (3) SLO'ya göre kısıtla: kullanıcı max 3sn bekleyebilir → tüm chain bütçesi 3sn. Tuzaklar: P50 (medyan) ile belirle → %50 timeout! P99 altında belirle → normal günde false positive. Çok gevşek belirle → yavaş servis uzun bloke.

---

**Senior / Architect:**

8. Deadline propagation nedir? Olmadan ne olur?

   > **Beklened:** Deadline propagation: A→B→C zincirinde A'nın kalan süresini (deadline) B'ye ve C'ye iletmek. Olmadan: A 5sn'de timeout → kullanıcıya hata döndü. Ama B hâlâ C'yi bekliyor, C hâlâ D'yi bekliyor → orphan requests → kaynak israfı, downstream servis gereksiz yük. gRPC: deadline built-in, `withDeadlineAfter()` otomatik propagate. HTTP: `X-Request-Deadline` header — interceptor ile her çağrıya ekle. Kafka: mesaja timestamp koy, consumer işlemeden önce "hâlâ geçerli mi?" kontrol et. Önemi: 100 eş zamanlı isteğin %10'u timeout olsa → 10 orphan chain × N servis = N×10 gereksiz thread.

9. Timeout ile Circuit Breaker birlikte nasıl çalışır? Sıralama neden önemlidir?

   > **Beklened:** TimeLimiter (Resilience4j): tek çağrı için süre sınırı. Circuit Breaker: hata oranı izler → eşik aşılınca açılır. Sıralama: `TimeLimiter → CircuitBreaker → actual call`. TimeLimiter timeout → CB bunu hata sayar. Yeterli timeout → CB açılır → sonraki çağrılar TimeLimiter'a bile gitmez, anında fail. Ters sıra yanlış: CB içte, TimeLimiter dışta → CB açık olsa bile TimeLimiter bekleme süresi kadar geçer. Pratik: timeout CB'yi besler, CB timeout'ları engeller (OPEN durumda). Her ikisi gerekli: timeout olmadan CB çok yavaş tepki verir (thread'ler bloke).

---

## Bölüm 3: Retry & Exponential Backoff

### Gerçek Hayat Sorunları

---

**Sorun 7: Retry + idempotency yok — çift ödeme**

```java
// Senaryo: Ödeme servisi timeout verdi, retry yapıldı.
// İlk istek gateway'de işlendi ama timeout nedeniyle onay gelmedi.
// Retry: aynı istek tekrar → ikinci ödeme işlendi.
// Kullanıcı hesabından çift para çekildi.

// YANLIŞ: Idempotency key olmadan retry
@Retryable(retryFor = TimeoutException.class, maxAttempts = 3)
PaymentResult charge(Order order) {
    return paymentGateway.charge(order.getTotal(), order.getCardToken());
    // Her retry → yeni ödeme işlemi başlatılıyor!
}

// DOĞRU: Idempotency key ile retry
@Retryable(retryFor = TimeoutException.class, maxAttempts = 3)
PaymentResult charge(Order order) {
    String idempotencyKey = "payment-" + order.getId();  // sabit, retry'da değişmez
    return paymentGateway.charge(
        order.getTotal(),
        order.getCardToken(),
        idempotencyKey  // gateway: aynı key → aynı sonucu döndür, tekrar işleme
    );
}

// Gateway tarafı (idempotency storage):
PaymentResult charge(BigDecimal amount, String card, String idempotencyKey) {
    // Daha önce bu key ile işlem yapıldı mı?
    Optional<PaymentResult> existing = idempotencyStore.get(idempotencyKey);
    if (existing.isPresent()) {
        return existing.get();  // Önceki sonucu döndür, tekrar işleme
    }
    
    PaymentResult result = processCharge(amount, card);
    idempotencyStore.set(idempotencyKey, result, Duration.ofHours(24));
    return result;
}

// Kural: Retry yapacaksan → idempotent ol ya da idempotency key ekle.
// GET, DELETE → doğası gereği idempotent.
// POST /payments → idempotency key zorunlu.
// PUT /orders/{id}/status → idempotent (aynı state'e getiriyor).
```

---

**Sorun 8: Thundering herd — jitter yok, tüm client'lar aynı anda retry yaptı**

```
Senaryo:
  1000 client aynı anda servis çağrıyor.
  Servis geçici yük altında → 10:00:00'da tümü timeout.
  Retry konfigürasyonu: 1sn bekle, retry.
  
  10:00:01: 1000 client aynı anda retry → servis üzerine 3x yük (yeni + retry'lar)
  10:00:01: Servis yine timeout → 1000 client 1sn bekle
  10:00:02: 1000 client aynı anda retry → 3x yük
  Servis hiç toparlanamıyor → cascading failure derinleşiyor.

Jitter olmadan exponential backoff bile yetersiz:
  1. deneme: 1sn bekle → 1000 client aynı anda
  2. deneme: 2sn bekle → 1000 client aynı anda
  3. deneme: 4sn bekle → 1000 client aynı anda
  Hep synchronized spike!

Jitter ile:
  Client A: 1.3sn bekle
  Client B: 0.6sn bekle
  Client C: 1.8sn bekle
  → Yük zamana yayılır → servis her "wave"de daha az istek görür → toparlanabilir

Implementasyon:
  // Resilience4j: randomization factor
  .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
      500,   // base 500ms
      2.0,   // 2x çarpan
      0.5,   // ±%50 jitter
      30_000 // max 30sn
  ))
  
  // Manuel hesaplama:
  long delay = (long)(baseDelay * Math.pow(2, attempt));
  long maxDelay = 30_000;
  delay = Math.min(delay, maxDelay);
  long jitter = (long)(delay * 0.5 * Math.random());  // ±%50
  Thread.sleep(delay + jitter - (long)(delay * 0.25)); // equal jitter
```

---

**Sorun 9: 4xx hatalarda retry — validation hatası için boşuna deneme**

```java
// Senaryo: Kullanıcı bozuk sipariş verisi gönderdi → 400 Bad Request.
// Retry mekanizması 3 kez daha denedi → 3×400 → gereksiz yük.
// Her retry 500ms bekliyor → kullanıcı 1.5sn sonra hata alıyor.

// YANLIŞ: Tüm exception'larda retry
@Retryable(maxAttempts = 4)
OrderResponse createOrder(OrderRequest req) {
    return orderClient.post("/orders", req);
    // 400 gelirse: retry → 400 → retry → 400 → retry → 400
    // Veri değişmedi, sonuç değişmez
}

// DOĞRU: HTTP durum koduna göre retry kararı
@Retryable(
    retryFor  = {TimeoutException.class, ServiceUnavailableException.class},
    noRetryFor = {
        HttpClientErrorException.BadRequest.class,       // 400
        HttpClientErrorException.Unauthorized.class,     // 401
        HttpClientErrorException.Forbidden.class,        // 403
        HttpClientErrorException.NotFound.class,         // 404
        ValidationException.class
    },
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2, random = true)
)
OrderResponse createOrder(OrderRequest req) {
    return orderClient.post("/orders", req);
}

// Retry YAP (geçici hatalar):
// 500 Internal Server Error
// 502 Bad Gateway (upstream down)
// 503 Service Unavailable
// 504 Gateway Timeout
// 429 Too Many Requests (Retry-After header'ına bak)
// Connection timeout, read timeout

// Retry YAPMA (kalıcı hatalar):
// 400 Bad Request (veri bozuk, retry çözmez)
// 401 Unauthorized (token refresh gerekiyor, retry değil)
// 403 Forbidden (izin yok, retry çözmez)
// 404 Not Found (kaynak yok, retry çözmez)
// 409 Conflict (business logic çatışması)
// 422 Unprocessable Entity (validation hatası)
```

---

**Sorun 10: Retry sayısı çok yüksek — servis toparlanma şansı bulamadı**

```
Senaryo:
  payment-service.maxAttempts = 10
  backoff.delay = 100ms, multiplier = 1.5

  1 client:  10 deneme × ortalama 300ms = 3sn bekleme
  1000 client × 10 deneme = 10.000 istek tek servis üzerine
  Servis zaten aşırı yük altında → retry'lar yükü daha da artırıyor.
  "Retry storm": retry servisin toparlanmasını engelliyor.

Doğru retry stratejisi:
  maxAttempts = 3 (genellikle yeterli)
  - 1. deneme: anlık spike → çözülür.
  - 2. deneme: kısa dönem sorun → çözülür.
  - 3. deneme: hâlâ yok → Circuit Breaker açılsın.
  10 deneme: servise toparlanma fırsatı vermeden bombalıyorsun.

  Retry yerine Circuit Breaker:
  3 deneme → CB açılır → sonraki 30sn istek CB'den geçmez → servis nefes alır.
  HALF_OPEN: 1 test isteği → başarılıysa CB kapanır → normal operasyon.

  Toplam latency bütçesi:
  Kullanıcı SLA: 3sn
  Zincir: orderService → paymentService
  Payment servis timeout: 500ms, retry 3 kez, backoff 200-800ms
  Toplam: 3 × (500ms + backoff) ≈ 3sn → tam bütçe kullanıldı
  Başka downstream çağrı yoksa kabul edilebilir.
  Başka çağrılar varsa: retry sayısını veya timeout'u küçült.
```

---

### Mülakat Soruları — Retry & Backoff

**Junior / Mid:**

10. Exponential backoff nedir? Neden sabit aralıklı retry'dan üstündür?

    > **Beklened:** Sabit aralık: her 1sn'de retry. Servis 10sn sonra düzelecekse 10 gereksiz retry. Yük sürekli sabit. Exponential backoff: her denemede bekleme iki katına çıkar. 1sn → 2sn → 4sn → 8sn → 16sn. Servis düzelince daha az retry gördüğü için toparlanabilir. Matematiksel: `delay = base × 2^attempt`. Max delay cap ile sonsuz büyüme önlenir. Gerçek hayat: AWS, Google tüm SDK'larında exponential backoff önerir. Dezavantaj: gecikme artar — kullanıcı daha uzun bekler (latency budget etkisi).

11. Jitter neden eklenir? Jitter olmayan exponential backoff'un sorunu nedir?

    > **Beklened:** Jitter olmadan: 1000 client aynı anda fail → aynı anda exponential bekle → aynı anda retry → synchronized spike. Her retry dalgası servisi ezer, toparlanma imkânsız — "thundering herd". Jitter: bekleme süresine rastgelelik ekler. Her client farklı zamanda retry → yük zamana yayılır → servis her dalgada makul sayıda istek görür → toparlanabilir. Full jitter: `sleep = random(0, calculated_delay)`. Equal jitter: `sleep = calculated_delay/2 + random(0, calculated_delay/2)` — minimum garantili bekleme. AWS önerisi: full jitter. Sonuç: dağıtık sistemlerde retry daima jitter ile.

12. Hangi HTTP status code'larında retry yapılır, hangilerinde yapılmaz?

    > **Beklened:** Retry YAP: 500 (internal error — geçici olabilir), 502 (bad gateway — upstream restart), 503 (service unavailable), 504 (gateway timeout), 429 (rate limited — Retry-After header'ına göre bekle), connection timeout, read timeout. Retry YAPMA: 400 (bad request — veri bozuk, retry çözmez), 401 (auth gerekli — token refresh et, retry değil), 403 (izin yok, retry çözmez), 404 (kaynak yok), 409 (conflict — business logic), 422 (validation). Temel kural: "Aynı isteği tekrar göndersem farklı sonuç alır mıyım?" Hayırsa → retry etme.

---

**Senior / Architect:**

13. Non-idempotent işlemler için retry stratejisi nasıl tasarlanır?

    > **Beklened:** POST /payments gibi non-idempotent işlemler: retry = çift işlem riski. Çözüm 1: Idempotency key. Client: her istek için UUID üret, request body veya header'a ekle. Server: key'i Redis/DB'ye kaydet → aynı key tekrar gelince önceki sonucu döndür. Retry güvenli. Çözüm 2: Saga pattern + compensation. İşlem başarısız → compensation transaction. Retry yerine telafi. Çözüm 3: Outbox + mesaj queue. İşlemi DB'ye yaz (atomik), ayrı bir worker bir kez gönderir. Retry yerine at-least-once delivery + idempotent consumer. Altın kural: "Retry yapacaksan idempotent yap veya idempotency key ekle."

14. Latency budget nedir? Retry ve timeout'ların SLO etkisini nasıl hesaplarsın?

    > **Beklened:** Latency budget: kullanıcıya verilen SLO süresi (örn. 3sn P99). Tüm zincir bu budget içinde tamamlanmalı. Hesaplama: A → B → C zinciri, budget 3sn. C: timeout 500ms, 2 retry, backoff 200ms → max: 2×(500+200) + 500 = 1.9sn. B: C'yi çağırır + kendi işleme 200ms → max 2.1sn. A: B'yi çağırır + 300ms → 2.4sn → budget içinde. Tuzak: retry × timeout cascade → budget'ı aşar. Örnek: 3 retry × 1sn timeout = 3sn (tek başına budget tüketildi). Çözüm: retry sayısını veya timeout'u küçült. Circuit Breaker: retry yerine CB → budget korunur.

---

## Bölüm 4: Pattern Kombinasyonu

### Gerçek Hayat Sorunları

---

**Sorun 11: Yanlış sıralama — Retry CB dışında, boşuna tekrar denedi**

```java
// Senaryo: CB açıkken retry çalışıyordu — her retry anında CallNotPermittedException.
// Retry: 3 deneme × 500ms backoff = 1.5sn gecikme. Hepsi reddedildi.
// CB zaten açık, retry'lar anlamsız yük ve gecikme.

// YANLIŞ sıralama: Retry dışta, CB içte
// Dıştan içe execution: Retry → CB → actual call
// CB OPEN → CallNotPermittedException → Retry "hata var, tekrar dene" → CB yine OPEN
// 3 retry × hepsi anında fail → gecikme + anlamsız döngü

// DOĞRU sıralama (Resilience4j annotation stack):
// Annotation'lar: dıştan içe (en alttaki en dışta execute edilir)
@Retry(name = "payment")           // 4. dışta → başarısız olunca retry
@CircuitBreaker(name = "payment")  // 3. → CB state kontrol
@Bulkhead(name = "payment")        // 2. → kaynak sınırla
@TimeLimiter(name = "payment")     // 1. içte → timeout sınırla
CompletableFuture<PaymentResult> processPayment(Order order) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(order));
}

// Execution akışı (iç → dış):
// TimeLimiter (timeout sınırla)
// → Bulkhead (thread/semaphore kontrol)
// → CircuitBreaker (CB state kontrol, OPEN ise → hızlı fail)
// → Actual call
// Hata → CircuitBreaker hata sayar
// Hata → Bulkhead serbest bırak
// Hata → Retry (CB OPEN değilse tekrar dene, OPEN ise retry anlamsız)

// CB açıkken Retry ne yapmalı?
// CallNotPermittedException → noRetryFor listesine ekle:
RetryConfig.custom()
    .ignoreExceptions(CallNotPermittedException.class)  // CB açıksa retry etme
    .build();
```

---

### Mülakat Soruları — Kombinasyon

**Junior / Mid:**

15. Resilience4j'de Retry, CircuitBreaker, Bulkhead ve TimeLimiter'ın doğru sıralaması nedir? Neden bu sıra?

    > **Beklened:** Execution sırası (içten dışa): TimeLimiter → Bulkhead → CircuitBreaker → Retry. Annotation sırası (alttaki içte): @TimeLimiter en altta. Neden: TimeLimiter tek çağrının zamanını sınırlar — en içte. Bulkhead: eş zamanlı kaynak sınırı. CircuitBreaker: başarısızsa açılır, açıksa istek geçmez. Retry: hata alındıysa dışarıdan tekrar dener. CB OPEN → CallNotPermittedException → Retry bunu görür. Doğru Retry config: CB açıkken retry etme (`ignoreExceptions(CallNotPermittedException.class)`). Yanlış sıralama: Retry içte, CB dışta → CB açılmaz (retry başarısız olsa bile CB retry'ı görür, actual call'ı değil).

---

**Senior / Architect:**

16. Bir microservice sisteminde "ödeme yapılamıyor, stoğu rezerve edemiyorum" gibi çoklu downstream bağımlılık arızasında tüm pattern'ları nasıl konumlandırırsın?

    > **Beklened:** Mimari yaklaşım: (1) Bulkhead: payment pool (10 thread), inventory pool (20 thread) — biri dolu diğerini etkilemez. (2) Timeout: payment 500ms, inventory 200ms (P99 × 2). Cascade: order servisi 1sn timeout (payment + inventory + buffer). (3) Retry: inventory (GET, idempotent) → 3 retry + jitter. Payment (POST) → idempotency key + 2 retry max. (4) Circuit Breaker: payment CB (10 çağrıda %50 hata → OPEN, 30sn). inventory CB (20 çağrıda %60 hata → OPEN). (5) Fallback: payment CB OPEN → "ödeme şu an alınamıyor" mesajı, isteği async kuyruğa al. inventory CB OPEN → son bilinen stok cache'ten (stale label ile). (6) Deadline propagation: gelen HTTP isteğinin kalan süresini tüm downstream'e geç. (7) Monitoring: CB state, bulkhead rejection rate, retry exhaustion count → Grafana alert.

17. Hangi senaryoda retry hiç kullanmazsın? Alternatif nedir?

    > **Beklened:** Retry kullanma: (1) Non-idempotent işlem + idempotency key yok → çift işlem. (2) Business logic hatası (409 Conflict, 422) → data düzelmeden retry faydasız. (3) Downstream servis kalıcı down → retry latency tüketir, CB açılsın daha iyi. (4) Latency budget çok kısa → kullanıcı zaten timeout aldı, geç gelen retry anlamsız. Alternatifler: Circuit Breaker: hızlı fail, CB toparlanmayı yönetir. Async/queue: isteği kuyruğa at, servis düzelince işle — kullanıcıya "işlemin alındı" de. Saga + compensation: step başarısız → compensating transaction. Outbox pattern: at-least-once delivery, idempotent consumer.

---

## Karma — Architect Seviyesi

18. **"Yeni bir microservice mimarisi tasarlıyorsun. Her servis için resilience stack'ini nasıl belirlersin?"**

    > **Beklened:** Adım 1 — Bağımlılık haritası: Her servisin hangi downstream'lere bağlı olduğunu çiz. Kritiklik: "Bu servis down olursa kullanıcı ne görür?" Adım 2 — SLO belirle: P99 latency, error rate. Adım 3 — Timeout: Her downstream için P99 × 2-3. Cascade: dış > iç. Adım 4 — Bulkhead: Her downstream için ayrı pool. Pool size: Little's Law + load test. Adım 5 — Retry: Sadece idempotent veya idempotency key olan işlemler. Max 3 deneme. Jitter zorunlu. 4xx → retry yok. Adım 6 — Circuit Breaker: Sliding window (20 çağrı), %50 hata → OPEN. wait_duration: downstream'in tipik recovery süresi. Adım 7 — Fallback: CB OPEN veya timeout → degraded deneyim. Cache, default, kuyruk. Adım 8 — Monitoring: CB state, rejection rate, retry exhaustion, P99 latency per downstream.

19. **"Production'da bir servis %20 timeout oranı görüyor. Nasıl debug edersin?"**

    > **Beklened:** (1) Timeout nerede oluyor? Client-side mi, server-side mi? Client log: "Read timeout after 3000ms". Server log: istek geldi mi? Gelmediyse → ağ/load balancer sorunu. Geldiyse → server yavaş. (2) Hangi endpoint/işlem? Tüm endpoint'ler mi, belirli biri mi? Prometheus: `http_server_request_duration_seconds` histogram → P99 spike hangi endpoint'te. (3) Server tarafında ne yavaş? Thread dump: thread'ler nerede bekliyor? DB sorgusunda mı? Dış API'de mi? (4) DB: slow query log, pg_stat_activity. Connection pool doluluk oranı. (5) Downstream: o servisin P99 nedir? Timeout chain mi? (6) GC: GC log, pause süresi > timeout değeri mi? (7) Infrastructure: CPU throttling (K8s CPU limit), network latency. (8) Çözüm sıralı: önce kök neden, sonra timeout değeri artır (geçici). Sonra altta yatan sorunu çöz.
