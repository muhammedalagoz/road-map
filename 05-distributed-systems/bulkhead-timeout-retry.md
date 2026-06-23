# 05d — Bulkhead, Timeout & Retry Pattern'ları

## 1. Bulkhead Pattern

### Ne?

**Bulkhead (Perde):** Bir servisin sorunlarının diğerlerini etkilememesi için kaynakları (thread pool, connection pool, semaphore) izole etme pattern'ı. Gemi perdelerinden ilham alınır — bir bölme su alsa diğerleri batmaz.

```
Bulkhead olmadan:
  [OrderService Thread Pool: 200 thread]
  PaymentService → yavaş (300ms yerine 5s)
  200 thread de PaymentService bekler → Order API tamamen dondu
  Kullanıcılar sipariş VEREMEZ (Payment sorununu kendine bulaştırdı)

Bulkhead ile:
  [OrderService Thread Pool]
    PaymentService pool: 50 thread
    InventoryService pool: 50 thread
    ShippingService pool: 50 thread
    Web request pool: 50 thread
  PaymentService → 50 thread dolu → sadece payment işlemleri başarısız
  Diğer servisler çalışmaya devam eder
```

### Neden?

**Cascade failure önleme:** Bir bileşenin yavaşlaması tüm sistemi etkiler (bulkhead olmadan).

```
Senaryo — Cascading failure:
  3rd party ödeme sağlayıcısı → timeout 30s
  Her ödeme isteği → 30s thread tutar
  Trafik spike → 1000 eş zamanlı istek
  1000 thread → thread pool doldu → yeni istekler queue'da
  Queue doldu → timeout → tüm uygulama yanıt vermez
  
  Sorun: Ödeme servisi tüm uygulamayı çökertti
```

### Nasıl?

#### Thread Pool Bulkhead (Resilience4j)

```java
// Her downstream servis için ayrı thread pool
ThreadPoolBulkheadConfig paymentConfig = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)       // max 10 thread payment için
    .coreThreadPoolSize(5)       // normal yük: 5 thread
    .queueCapacity(20)           // bekleme kuyruğu max 20
    .keepAliveDuration(Duration.ofMillis(20))
    .build();

ThreadPoolBulkhead paymentBulkhead = ThreadPoolBulkhead.of(
    "payment-service", paymentConfig);

ThreadPoolBulkheadConfig inventoryConfig = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(20)
    .coreThreadPoolSize(10)
    .queueCapacity(50)
    .build();

ThreadPoolBulkhead inventoryBulkhead = ThreadPoolBulkhead.of(
    "inventory-service", inventoryConfig);

// Kullanım
@Service
class OrderService {

    CompletableFuture<PaymentResult> chargePayment(Order order) {
        Supplier<PaymentResult> paymentCall = () -> 
            paymentClient.charge(order.getTotal());

        return paymentBulkhead.executeSupplier(paymentCall)
            .exceptionally(ex -> {
                if (ex instanceof BulkheadFullException) {
                    // Thread pool dolu → hızlı fail
                    return PaymentResult.rejected("Sistem meşgul");
                }
                return PaymentResult.failed(ex.getMessage());
            });
    }
}
```

#### Semaphore Bulkhead (Lightweight)

```java
// Thread pool yerine sadece eş zamanlı çağrı sayısını sınırla
BulkheadConfig semaphoreConfig = BulkheadConfig.custom()
    .maxConcurrentCalls(25)      // max 25 eş zamanlı çağrı
    .maxWaitDuration(Duration.ofMillis(500)) // 500ms bekle, doluysa fail
    .build();

Bulkhead bulkhead = Bulkhead.of("external-api", semaphoreConfig);

// Spring AOP ile
@Bulkhead(name = "payment-service", fallbackMethod = "paymentFallback")
PaymentResult charge(Order order) {
    return paymentClient.charge(order);
}

PaymentResult paymentFallback(Order order, BulkheadFullException ex) {
    // Kuyrukta beklet veya hata döndür
    eventQueue.add(new PendingPayment(order));
    return PaymentResult.queued();
}
```

#### Spring Cloud Gateway ile Bulkhead

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: payment-route
          uri: lb://payment-service
          filters:
            - name: RequestRateLimiter
            - name: Bulkhead
              args:
                maxConcurrentCalls: 50
                maxWaitDuration: 500ms
                fallbackUri: forward:/fallback/payment
```

---

## 2. Timeout Stratejileri

### Ne?

**Timeout:** Bir işlemin belirli süreden uzun sürmesi durumunda iptal edilmesi. İki temel tip:

```
Connection Timeout: Bağlantı kurulana kadar bekleme süresi
  TCP handshake + sunucunun "kabul etmesi" süresi
  Tipik: 3-10 saniye

Read Timeout (Socket Timeout): Bağlantı kurulduktan sonra yanıt bekleme
  Sunucu yanıt üretene kadar bekleme
  Tipik: iş mantığına göre (30ms - 30s)
```

### Neden?

```
Timeout olmadan:
  Thread → downstream servisi bekler (sonsuza kadar)
  Downstream servis yavaş/çöktü → thread sonsuza kadar bloke
  1000 eş zamanlı istek → 1000 thread bloke
  Thread pool tükendi → sistem çöktü

Timeout ile:
  Thread 5 saniye bekler → timeout → exception fırlatır → thread serbest
  Hata kullanıcıya döner, sistem kurtarılır
```

### Nasıl?

#### Spring RestTemplate / WebClient Timeout

```java
// RestTemplate — connection ve read timeout
@Bean
RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory = 
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(3000);     // 3s bağlantı timeout
    factory.setReadTimeout(5000);        // 5s okuma timeout

    return new RestTemplate(factory);
}

// WebClient (Reactive) — timeout
@Bean
WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(5)));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}

// Per-request timeout (WebClient)
webClient.get()
    .uri("/api/inventory/{id}", productId)
    .retrieve()
    .bodyToMono(InventoryResponse.class)
    .timeout(Duration.ofSeconds(2))  // bu istek için override
    .onErrorReturn(TimeoutException.class, InventoryResponse.unknown());
```

#### Timeout Değerlerini Nasıl Seçmeli?

```
P99 latency'yi ölç:
  Normal koşulda servisin P99 yanıt süresi: 150ms
  Timeout = P99 * 2-3 = 300-450ms
  (Çok sıkı → gereksiz timeout, çok gevşek → yavaş servis tolere edilir)

Cascade timeout:
  A → B → C → D (zincir)
  D timeout: 1s
  C timeout: 1.5s (D + buffer)
  B timeout: 2.5s
  A timeout: 3.5s
  Her timeout bir öncekinden büyük olmalı

Timeout > Retry * (Upstream timeout + backoff):
  Upstream timeout: 1s
  Retry count: 3
  Backoff: 0.5s
  Min timeout: 3 * (1 + 0.5) = 4.5s
```

#### Deadline Propagation

```java
// gRPC — deadline otomatik propagate edilir
Channel channel = ManagedChannelBuilder
    .forAddress("inventory-service", 9090)
    .build();

OrderServiceGrpc.OrderServiceBlockingStub stub = 
    OrderServiceGrpc.newBlockingStub(channel);

// A → B çağrısında kalan süreyi geç
stub.withDeadlineAfter(remainingBudget, TimeUnit.MILLISECONDS)
    .getInventory(request);

// HTTP'de manual deadline (X-Request-Deadline header ile)
@Component
class DeadlineInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
        String deadline = RequestContext.getDeadline();
        if (deadline != null) {
            request.getHeaders().add("X-Request-Deadline", deadline);
        }
        return execution.execute(request, body);
    }
}
```

---

## 3. Retry with Exponential Backoff + Jitter

### Ne?

**Retry:** Başarısız işlemleri belirli koşullarda tekrar denemek.
**Exponential Backoff:** Her denemede bekleme süresini katlanarak artırmak.
**Jitter:** Bekleme süresine rastgelelik eklemek (thundering herd önleme).

### Neden?

```
Sorun 1: Naive retry (hemen tekrar)
  Servis geçici yük altında → timeout
  1000 client aynı anda retry → 3x yük → daha da kötü

Sorun 2: Fixed interval retry
  Her 1 saniyede retry → servis düzelmedi → hep aynı yük
  
Sorun 3: Exponential backoff olmadan (thundering herd)
  10:00:01 → 1000 client fail
  10:00:02 → 1000 client retry (aynı anda!)
  10:00:03 → 1000 client retry (aynı anda!)
  Servis toparlanamaz

Çözüm: Exponential Backoff + Jitter
  Client A: 1s bekle
  Client B: 0.7s bekle (jitter)
  Client C: 1.3s bekle (jitter)
  2. deneme: 2s +/- jitter
  3. deneme: 4s +/- jitter
  → Yük dağıtılır, servis toparlanabilir
```

### Nasıl?

```
Backoff formülü:
  delay = min(baseDelay * 2^attempt, maxDelay)
  
  attempt=0: delay = 1s
  attempt=1: delay = 2s
  attempt=2: delay = 4s
  attempt=3: delay = 8s
  attempt=4: delay = 16s
  attempt=5: delay = 32s → max=30s ile cap → 30s

Jitter türleri:
  Full jitter:     sleep = random(0, delay)
  Equal jitter:    sleep = delay/2 + random(0, delay/2)
  Decorrelated:    sleep = min(maxDelay, random(base, prevSleep * 3))
  
  AWS "Exponential Backoff and Jitter" blog → Full jitter önerilir
```

```java
// Resilience4j Retry
RetryConfig retryConfig = RetryConfig.custom()
    .maxAttempts(4)
    .waitDuration(Duration.ofMillis(500))               // base delay
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
        500,    // base delay ms
        2.0,    // multiplier
        0.5,    // randomization factor (jitter ±50%)
        30_000  // max delay ms
    ))
    .retryOnException(ex -> 
        ex instanceof TimeoutException ||
        ex instanceof ServiceUnavailableException
    )
    .ignoreExceptions(
        ValidationException.class,   // 4xx → retry etme
        AuthenticationException.class
    )
    .build();

Retry retry = Retry.of("inventory-service", retryConfig);

// Kullanım
@Retry(name = "inventory-service", fallbackMethod = "inventoryFallback")
@CircuitBreaker(name = "inventory-service")
InventoryResponse checkInventory(String productId) {
    return inventoryClient.check(productId);
}

InventoryResponse inventoryFallback(String productId, Exception ex) {
    return InventoryResponse.unknown(productId);
}
```

#### Spring @Retryable

```java
@Service
class ExternalApiService {

    @Retryable(
        retryFor = {TimeoutException.class, HttpServerErrorException.class},
        noRetryFor = {HttpClientErrorException.class},  // 4xx → retry etme
        maxAttempts = 4,
        backoff = @Backoff(
            delay = 500,        // 500ms başlangıç
            multiplier = 2.0,   // her seferinde 2x
            maxDelay = 30_000,  // max 30s
            random = true       // jitter ekle
        )
    )
    Response callExternalApi(Request request) {
        return httpClient.post("/api/endpoint", request);
    }

    @Recover
    Response recoverFromFail(Exception ex, Request request) {
        // tüm retry'lar başarısız → fallback
        metricsService.increment("external_api.exhausted_retries");
        return Response.serviceUnavailable();
    }
}
```

#### Ne Zaman Retry Yapmamalı?

```
Retry YAP:
✓ 5xx (500, 502, 503, 504) — geçici sunucu hatası
✓ Timeout — ağ gecikmesi
✓ Connection refused — servis restart oluyor olabilir
✓ Rate limit 429 — kısa süre sonra retry

Retry YAPMA:
✗ 4xx (400, 401, 403, 404) — client hatası, retry çözmez
✗ İdempotent olmayan işlemler (POST /payments — çift ödeme)
✗ Validation hatası — veri düzelmeden retry faydasız
✗ Business logic hatası — retry mantıksal sorunu çözmez

İdempotency + Retry birlikte:
  Retry yapacaksan → idempotency key ekle
  POST /payments + Idempotency-Key: uuid → çift retry = tek ödeme
```

---

## Resilience Pattern'ları Birlikte Kullanımı

```java
// Circuit Breaker + Bulkhead + Retry + Timeout — sıralama önemli!
@Retry(name = "payment")
@CircuitBreaker(name = "payment")
@Bulkhead(name = "payment")
@TimeLimiter(name = "payment")
CompletableFuture<PaymentResult> processPayment(Order order) {
    return CompletableFuture.supplyAsync(
        () -> paymentClient.charge(order)
    );
}

// Sıralama (dıştan içe):
// Retry → CircuitBreaker → Bulkhead → TimeLimiter → Actual Call
// 
// Neden bu sıra?
//   1. TimeLimiter: çağrıyı zaman sınırıyla wrap et
//   2. Bulkhead: eş zamanlı çağrı sınırla
//   3. CircuitBreaker: başarısız → aç
//   4. Retry: hata alıyorsa → dene (CB OPEN ise retry faydasız ama CB kontrol eder)
```

---

## Ne zaman?

```
Bulkhead:
✓ Birden fazla downstream bağımlılık varsa → her biri için ayrı pool
✓ Yavaş servis tüm sistemi etkiler riski varsa
✓ Kritik vs kritik olmayan işlem izolasyonu

Timeout:
✓ Her dış çağrıda (HTTP, DB, cache, mesajlaşma)
✓ Kullanıcı isteği zincirinde (deadline propagation)

Retry:
✓ Geçici hatalarda (network blip, restart)
✓ İdempotent işlemlerde
✗ Non-idempotent işlemlerde (idempotency key olmadan)
✗ İş mantığı hatasında
```

---

## Trade-off?

| Pattern | Avantaj | Dezavantaj |
|---------|---------|-----------|
| **Bulkhead** | Cascade failure önler | Kaynak israfı (her pool ayrı) |
| **Bulkhead** | Servis izolasyonu | Pool boyutu ayarı zor |
| **Timeout** | Thread'leri serbest bırakır | Çok kısa → gereksiz hata |
| **Timeout** | Cascade failure önler | Çok uzun → yavaş tepki |
| **Retry** | Geçici hata tolere | Yük artırabilir |
| **Retry + Jitter** | Thundering herd önler | Gecikme artar (latency budget) |
| **Exponential Backoff** | Servise toparlanma süresi | Max retry → kullanıcı uzun bekler |
