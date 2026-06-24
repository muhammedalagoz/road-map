# 02d — Spring Cloud: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Spring Cloud Gateway

### Gerçek Hayat Sorunları

---

**Sorun 1: Gateway SPOF — tek node çöküşü tüm sistemi durdurdu**

```
Senaryo:
  API Gateway tek instance olarak çalışıyor.
  Gateway pod crash etti → tüm client trafiği kesildi.
  Order, inventory, payment servisleri sağlıklı ama ulaşılamıyor.
  Uptime SLA ihlali.

Sorun: Gateway'i SPOF (Single Point of Failure) yapmak.

Düzeltme:
  1. Minimum 2 replica (farklı node'larda):
     Kubernetes: topologySpreadConstraints ile aynı node'a koyma.
     PodDisruptionBudget: minAvailable: 1 → rolling update sırasında
     en az 1 pod sağlıklı.

  2. HPA (Horizontal Pod Autoscaler):
     CPU/RPS bazlı otomatik scale-out.
     Trafik spike'ında yeni pod'lar devreye girer.

  3. Circuit breaker + fallback:
     Upstream servis down → gateway fallback response döndürür.
     Gateway tamamen down → DNS/CDN katmanında static fallback.

  4. Multi-region:
     Her region'da gateway → bölgesel failover.
     Global load balancer (Cloudflare, AWS Route53) → sağlıklı region'a yönlendir.
```

---

**Sorun 2: Rate limiter Redis bağımlılığı — Redis down, tüm istekler bloklandı**

```
Senaryo:
  Spring Cloud Gateway RequestRateLimiter → Redis token bucket kullanıyor.
  Redis connection timeout → gateway her isteği reject etti (429).
  Kullanıcılar limit aşmadıkları halde "Too Many Requests" aldı.

Sorun: Rate limiter'ın Redis'e olan bağımlılığı circuit açılmadı.

Konfigürasyon düzeltmesi:
  redis:
    timeout: 100ms            # hızlı fail
    connect-timeout: 50ms

  Gateway filter:
    - name: RequestRateLimiter
      args:
        deny-empty-key: false  # key resolve edilemezse blokla değil, geçir

Mimari düzeltme:
  Redis Sentinel veya Cluster → HA.
  Fallback: Redis down → rate limiting bypass (allow-all) → alert gönder.
  Local rate limiter (Caffeine) → Redis replica olarak kullan.
```

---

**Sorun 3: Gateway'de authentication bilgisi downstream'e iletilmedi**

```java
// Senaryo: JWT doğrulama gateway'de yapılıyor ama
// downstream servis "kim bu kullanıcı?" bilgisini görmüyor.

// YANLIŞ — validation yapıldı ama bilgi iletilmedi
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return jwtValidator.validate(token)
        .flatMap(claims -> chain.filter(exchange));  // orijinal request geçti, header yok
}

// DOĞRU — claims header'a ekleniyor
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return jwtValidator.validate(token)
        .flatMap(claims -> {
            ServerHttpRequest mutated = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", String.join(",", claims.getRoles()))
                .header("X-Tenant-Id", claims.get("tenantId", String.class))
                .build();
            return chain.filter(exchange.mutate().request(mutated).build());
        });
}

// Downstream servis:
@GetMapping("/orders")
List<Order> getOrders(@RequestHeader("X-User-Id") String userId) {
    return orderRepo.findByUserId(userId);
}
// JWT tekrar parse etmek gerekmez — gateway halletti.
```

---

### Mülakat Soruları — Gateway

**Junior / Mid:**

1. API Gateway ne işe yarar? Olmasa ne olur?

   > **Beklenen:** Merkezi cross-cutting concern yönetimi: auth, rate limiting, CORS, SSL termination, routing, logging. Olmasa: her servis kendi auth, rate limit vb. yazmak zorunda → tekrar, tutarsızlık. Client N farklı servis adresini bilmek zorunda → coupling. Versioning, A/B testing için ek IP. Gateway: tek giriş noktası, servisler temiz kalır. Dezavantaj: ek hop latency (~1ms), SPOF riski, karmaşıklık.

2. `PredicateFactory` ve `GatewayFilter` farkı nedir?

   > **Beklenen:** Predicate: routing kararı — "bu istek bu route'a gitsin mi?" (Path, Method, Header, Host, Query). Filter: request/response'u dönüştür — "geçmesine izin verildi, şimdi ne yapalım?" (AddHeader, StripPrefix, Retry, RateLimiter, CircuitBreaker). Predicate eşleşirse route seçilir, sonra filterlar çalışır.

3. `StripPrefix` ne zaman kullanılır?

   > **Beklenen:** Client: `GET /api/orders/1`. Gateway → upstream `order-service`'e gönderirken `/api` prefix'ini çıkar → `/orders/1` olarak ilet. Servis sadece kendi path'ini bilsin, `/api` prefix'ini değil. `StripPrefix=1`: ilk path segment'i sil. `/api/orders/1` → `/orders/1`. `StripPrefix=2`: `/v1/api/orders/1` → `/orders/1`.

---

**Senior / Architect:**

4. Gateway'de JWT doğrulama mı, her serviste mi? Trade-off nedir?

   > **Beklenen:** Gateway'de: tek noktada doğrulama — servisler kurtulur. Dezavantaj: gateway her isteği Vault/JWKS'e sorabilir → latency; gateway bypass edilirse güvensiz. Her serviste: defense in depth — gateway bypass edilse bile servis doğrular. Dezavantaj: tekrar kod, latency. Öneri: Gateway doğrular + claims header'a yazar → servis trust but verify (imzayı kontrol etmez, header'ı kabul eder, ama public endpoint gateway'den gelmek zorunda). İç ağda mTLS ile gateway'den geldiği garanti edilir.

5. Gateway'de rate limiter için Redis'in alternatifi var mı? Nasıl tasarlanır?

   > **Beklenen:** Alternatifler: (1) Caffeine (local, in-process): Redis yok, hızlı. Ama multi-instance'ta her gateway pod ayrı sayar → limit N pod × limit kadar istek geçer. (2) Hazelcast distributed cache: Redis olmadan cluster-wide rate limit. (3) Sliding window log algoritması: Redis'te timestamp listesi — bellek yoğun. (4) Token bucket (Redis): Spring Cloud default, en yaygın. Mimari seçim: distributed rate limit kritikse → Redis Cluster. Yaklaşık limit yeterliyse → Caffeine. External CDN (Cloudflare) → gateway'den önce.

---

## Bölüm 2: Spring Cloud Config

### Gerçek Hayat Sorunları

---

**Sorun 4: Config Server SPOF — servisler açılamıyor**

```
Senaryo:
  Config Server tek instance → bakım sırasında kapatıldı.
  Yeni pod başlatıldı → Config Server'a bağlanamadı → startup fail.
  fail-fast: true → "Config Server ulaşılamıyor, başlamıyorum."
  Bütün servisler scale-out yapamaz, crash-loop'a girdi.

Çözüm 1: Config Server HA
  Birden fazla instance + load balancer.
  Eureka'ya kayıt → Feign client ile bulunur.

Çözüm 2: Config Client cache
  spring.cloud.config.fail-fast: false
  Önceki başarılı config'i local cache'de sakla.
  Config Server down iken local cache kullan.

Çözüm 3: Kubernetes ConfigMap fallback
  Config Server down → K8s ConfigMap'ten oku (temel config).
  Config Server up → override et.

Çözüm 4: fail-fast + retry
  fail-fast: true
  retry.max-attempts: 6
  retry.initial-interval: 1000
  retry.multiplier: 1.5
  → 6 kez denedikten sonra başlamayı reddet (hızlı fail yerine sabırla bekle).
```

---

**Sorun 5: @RefreshScope çalışmıyor — bean yenilenmedi**

```java
// Senaryo: Config güncellendi, POST /actuator/refresh çağrıldı,
// ama servis hâlâ eski değeri kullanıyor.

// NEDEN 1: @RefreshScope eksik
@Service  // @RefreshScope YOK
class PricingService {
    @Value("${pricing.discount:0.0}")
    private double discount;    // refresh sonrası değişmez!
}

// DÜZELTME:
@RefreshScope  // şimdi refresh sonrası yeniden oluşturulur
@Service
class PricingService {
    @Value("${pricing.discount:0.0}")
    private double discount;
}

// NEDEN 2: @Configuration sınıfına @RefreshScope koymak
// @Configuration + @RefreshScope → tüm @Bean metodları yeniden çalışır
// Ama Spring bazı özel bean'ları (DataSource, Security config) yeniden
// oluşturmaz → tutarsız state. Sadece iş mantığı bean'lerine ekle.

// NEDEN 3: Singleton başka singleton'ı inject etti
@Service  // @RefreshScope yok
class OrderService {
    @Autowired PricingService pricing;  // PricingService @RefreshScope olsa bile
    // OrderService yeniden oluşturulmadığı için aynı pricing referansını tutar
}
// Çözüm: OrderService de @RefreshScope VEYA
// ApplicationContext.getBean() ile dinamik lookup VEYA
// Spring Cloud Bus ile tüm context refresh
```

---

**Sorun 6: Git repository'de plaintext şifre — güvenlik ihlali**

```
Senaryo:
  Config repo'ya bakılınca:
  # application-prod.yml
  spring.datasource.password: mySecretPassword123

  Git history'de sonsuza kadar kalır → "git log -p" ile görülebilir.
  Repo sızdıysa → tüm production credentials açıkta.

Çözümler (tercih sırasıyla):

1. Spring Cloud Config Encryption:
   Config Server'da jasypt veya Spring Cloud Crypto:
   spring.datasource.password: '{cipher}AQBd...'
   encrypt.key: ${ENCRYPT_KEY}  → env variable, git'e girmiyor.
   
2. Vault Backend:
   Config Server → Vault backend → şifreler Vault'ta, git'te değil.
   
3. Kubernetes Secret + volume mount:
   Config Server'dan çekme, K8s Secret mount et → dosyadan oku.
   
4. Environment variable override:
   git'te placeholder: spring.datasource.password: ${DB_PASSWORD}
   K8s secret → env variable → Spring override eder.
```

---

### Mülakat Soruları — Config

**Junior / Mid:**

6. Spring Cloud Config'in property öncelik sırası nasıldır?

   > **Beklenen:** En yüksekten en düşüğe: `order-service-prod.yml` → `order-service.yml` → `application-prod.yml` → `application.yml`. Spesifik (uygulama+profil) genel olanı (tüm uygulamalar) ezer. Config Server da application.yml'in bir kaynağı — ama OS env var ve command line argümanları Config Server'dan gelen değerleri ezer. Yani: komut satırı > env var > Config Server > local application.yml.

7. `@RefreshScope` nasıl çalışır? Her bean'e eklenmeli mi?

   > **Beklenen:** @RefreshScope: `/actuator/refresh` çağrılınca bu bean destroy edilir, sonraki kullanımda yeniden oluşturulur. Her bean'e eklenmemeli: DataSource, Security config, JPA config gibi kritik bean'ler yeniden oluşturulunca bağlantılar kopar, tutarsız state oluşabilir. Sadece config değeri okuyan servis/component bean'lerine ekle. Alternatif: `Environment` interface'ini inject et → her çağrıda güncel değer okunur (`@RefreshScope` gerekmez).

8. Hot reload ile restart arasındaki fark? Hangisi ne zaman?

   > **Beklenen:** Hot reload (@RefreshScope + actuator/refresh): servis çalışmaya devam eder, sadece değişen property'leri etkileyen bean'ler yenilenir. Downtime yok. Kısıt: @RefreshScope olan bean'ler yenilenir, yoksa değişmez. Restart: tam yeniden başlatma — tüm config tekrar okunur, garanti tam. Ne zaman: runtime değiştirilebilir config (feature flag, timeout, rate) → hot reload. Kritik altyapı config (DB URL, port) → restart gerekir.

---

**Senior / Architect:**

9. Config Server'ı Kubernetes ortamında kullanmak mantıklı mı? Alternatifler?

   > **Beklenen:** K8s ortamında: ConfigMap ve Secret native olarak bu işi görür. Config Server ek altyapı demek. Ancak Config Server avantajları: Git history (audit), @RefreshScope (hot reload), Vault entegrasyonu, environment inheritance (application.yml → servis.yml hiyerarşisi). Karar: Basit config + K8s → ConfigMap yeterli. Audit, hot reload, Vault entegrasyonu → Config Server değer katar. Hybrid: Config Server Vault'tan çeker, K8s'e expose eder.

10. Spring Cloud Bus nedir? Config değişikliğini nasıl tüm servislere yayar?

    > **Beklenen:** Spring Cloud Bus: mesaj broker (Kafka/RabbitMQ) üzerinden tüm servislere event yayma. Akış: Config repo değişti → Git webhook → Config Server `/actuator/busrefresh` → Kafka/RabbitMQ'ya `RefreshRemoteApplicationEvent` → tüm subscribe servisler event alır → kendi `/actuator/refresh` mantığını çalıştırır → @RefreshScope bean'ler yenilenir. Manuel `POST /actuator/refresh` yerine otomatik. Dezavantaj: Kafka/RabbitMQ bağımlılığı. Alternatif: ArgoCD/Flux GitOps → config değişince otomatik rolling update.

---

## Bölüm 3: Spring Cloud OpenFeign

### Gerçek Hayat Sorunları

---

**Sorun 7: Feign timeout yanlış konfigürasyon — cascade yavaşlama**

```
Senaryo:
  order-service → inventory-service Feign çağrısı.
  inventory-service yavaşladı: her istek 30 saniye sürüyor.
  Feign default timeout: connectTimeout=10sn, readTimeout=60sn.
  order-service thread havuzu: 200 thread.
  200 thread × 30sn bekleme → thread havuzu tükendi → order-service de yanıt veremez.

Sorun: Yüksek readTimeout → kaynak tüketimi.

Düzeltme:
feign:
  client:
    config:
      inventory-service:
        connectTimeout: 500    # bağlantı 500ms'de kurulmazsa fail
        readTimeout: 2000      # cevap 2sn'de gelmezse fail
      default:
        connectTimeout: 1000
        readTimeout: 5000

CircuitBreaker ile birlikte:
  - 2sn timeout → CB failure sayıldı
  - 5 çağrıda %50 fail → CB OPEN
  - inventory-service'e çağrı duruyor → thread'ler serbest
  - Fallback: stok bilgisi "bilinmiyor" → sipariş yine alındı

Kural: timeout < circuit breaker wait duration
  Timeout: 2sn → CB: 5 çağrıda karar → OPEN → 30sn bekle → HALF_OPEN
```

---

**Sorun 8: Feign client interface'inde Pageable serileştirme sorunu**

```java
// Sorun: Feign, Spring Data Pageable'ı query param'a doğru çeviremiyor
@FeignClient("product-service")
interface ProductClient {
    @GetMapping("/products")
    Page<Product> list(Pageable pageable);  // YANLIŞ çalışabilir
    // URL: /products?page=...&size=...&sort=... → ama format farklı olabilir
}

// Hata belirti:
// Upstream: sort=[name,asc] beklenti → Feign: sort=name%2Casc gönderiyor

// Düzeltme 1: Parametreleri ayrı geç
@GetMapping("/products")
Page<Product> list(
    @RequestParam int page,
    @RequestParam int size,
    @RequestParam(required = false) String sort
);

// Düzeltme 2: Feign PageableSpringEncoder ekle
@Bean
PageableSpringEncoder pageableSpringEncoder(ObjectFactory<HttpMessageConverters> converters) {
    return new PageableSpringEncoder(new SpringEncoder(converters));
}
// feign.autoconfiguration.jackson.enabled=true
```

---

**Sorun 9: Fallback factory exception bilgisi kayboluyor**

```java
// Sorun: Fallback çağrıldı ama neden bilinmiyor → debug edilemiyor

// YANLIŞ — FallbackFactory yerine Fallback class
@FeignClient(name = "payment-service", fallback = PaymentFallback.class)
interface PaymentClient { ... }

class PaymentFallback implements PaymentClient {
    PaymentResult charge(PaymentRequest req) {
        log.warn("Payment service unavailable");  // neden? bilinmiyor!
        return PaymentResult.failed();
    }
}

// DOĞRU — FallbackFactory: cause parametresi ile neden bilgisi
@FeignClient(name = "payment-service", fallbackFactory = PaymentFallbackFactory.class)
interface PaymentClient { ... }

@Component
class PaymentFallbackFactory implements FallbackFactory<PaymentClient> {
    public PaymentClient create(Throwable cause) {
        return req -> {
            if (cause instanceof FeignException.ServiceUnavailable) {
                log.error("Payment service down: {}", cause.getMessage());
                alertService.notify("PAYMENT_DOWN");
            } else if (cause instanceof FeignException.TooManyRequests) {
                log.warn("Payment service rate limited");
            }
            return PaymentResult.failed(cause.getMessage());
        };
    }
}
```

---

### Mülakat Soruları — OpenFeign

**Junior / Mid:**

11. `@FeignClient` ile `RestTemplate` farkı nedir? Feign'in avantajı?

    > **Beklenen:** RestTemplate: imperative HTTP client, URL string'leri, manual serialize/deserialize, verbose kod. Feign: interface + annotation → implementasyonu Spring üretir. Avantaj: (1) Minimal kod — servis endpoint'leri interface metodları olarak görünür. (2) Test: interface mock edilir. (3) Load balancer, circuit breaker otomatik entegrasyon. (4) Request interceptor (auth header tüm çağrılara ekle). Dezavantaj: soyutlama → debug için Feign log level FULL aç.

12. `fallback` ile `fallbackFactory` farkı nedir? Hangisi ne zaman kullanılır?

    > **Beklenen:** `fallback`: sabit class, exception bilgisi yok. `fallbackFactory`: exception nedeni ile çağrılır — farklı hata türleri için farklı davranış. Kural: neden fallback'e düştüğünü bilmek istiyorsan → FallbackFactory. Sadece "down olunca şunu dön" → fallback yeterli. Production: FallbackFactory tercih et → observability için exception log'u kritik.

13. Feign'de request interceptor ne zaman kullanılır?

    > **Beklenen:** Tüm Feign çağrılarına ortak bir şey eklenecekse: internal auth token (servisler arası), correlation ID (request tracing), tenant ID (multi-tenant). `RequestInterceptor` bean → `apply(RequestTemplate)` → header ekle. Global config veya client'a özel config'e tanımlanabilir. Örnek: `template.header("X-Internal-Token", tokenProvider.get())`.

---

**Senior / Architect:**

14. Feign + Circuit Breaker + Retry birlikte kullanılınca operasyon sırası nedir? Tehlikeli senaryo hangisi?

    > **Beklenen:** Sıra: Retry (en içte) → Circuit Breaker → TimeLimiter (en dışta). Tehlikeli senaryo: Retry × 3 deneme × her biri timeout → CB hata sayacına 3 hata ekler → CB daha çabuk OPEN olur. İdempotent olmayan POST'ta retry → duplicate işlem. Kural: retry sadece GET veya idempotency key olan işlemler. Timeout < retry wait < CB sliding window period → cascade önlenir.

15. Servisler arası senkron Feign çağrısı mı, asenkron Kafka mı? Nasıl karar verirsin?

    > **Beklenen:** Feign (senkron): kullanıcı cevap bekliyor, sonuç anında lazım (stok kontrolü, fiyat sorgusu). Dezavantaj: kaynak bağımlılığı, timeout, cascade failure. Kafka (asenkron): kullanıcı cevap beklemez, eventual consistency kabul edilir (sipariş oluştur → stok async azalt). Avantaj: decoupling, retry kolay, downstream down → mesaj bekler. Karar matrisi: "downstream down olursa ne olmalı?" → Feign'de fail, Kafka'da mesaj birikir. "Kullanıcıya anında cevap lazım mı?" → Feign. "Eventual consistency yeterli mi?" → Kafka.

---

## Bölüm 4: Spring Cloud Stream

### Gerçek Hayat Sorunları

---

**Sorun 10: Binding isim convention karışıklığı — mesaj gitmiyor**

```yaml
# Sorun: Binding ismi yanlış yazıldı → mesaj gönderilemiyor ama hata da yok

# YANLIŞ konfigürasyon
spring:
  cloud:
    stream:
      bindings:
        orderProducer-out-0:      # ← isim yanlış!
          destination: orders

# Kod:
@Bean
Supplier<OrderEvent> orderCreatedProducer() {   # ← metod adı: orderCreatedProducer
    return () -> queue.poll();
}

# Doğru binding ismi: {functionName}-{in|out}-{index}
# orderCreatedProducer → out (supplier) → index 0
# Doğru: orderCreatedProducer-out-0

# DOĞRU konfigürasyon:
bindings:
  orderCreatedProducer-out-0:     # ← metod adıyla eşleşmeli!
    destination: orders

# Debug:
spring.cloud.stream.bindings.* → actuator/bindings endpoint'i ile kontrol et
GET /actuator/bindings → mevcut binding'leri listeler
```

---

**Sorun 11: Binder değişikliği — Kafka → RabbitMQ geçişi beklentisi karşılanmadı**

```
Senaryo:
  "Stream soyutlaması sayesinde Kafka'dan RabbitMQ'ya geçeceğiz,
   sadece config değiştireceğiz" → gerçek olmadı.

Sorunlar:
  1. Kafka partition → RabbitMQ'da consumer group farklı semantik.
     Kafka: partition × consumer thread. RabbitMQ: queue × consumer.
     Concurrency konfigürasyonu tamamen farklı.

  2. Kafka consumer group offset management → RabbitMQ ack/nack.
     auto.offset.reset: earliest → RabbitMQ'da karşılığı yok.

  3. Kafka compaction, transaction, Kafka Streams → RabbitMQ'da yok.

  4. Error handling: DLQ vs DLX farklı yapılandırma.

Gerçek:
  Stream soyutlaması basit publish/consume için iyi.
  Broker'a özgü özellikler kullandıkça soyutlama sızıyor.

Öneri:
  Broker değişikliği düşünülmüyorsa → direkt spring-kafka veya spring-amqp.
  Stream: farklı projeler farklı broker kullanıyorsa veya hızlı prototip için.
```

---

### Mülakat Soruları — Stream

**Junior / Mid:**

16. `Supplier`, `Consumer`, `Function` bean'leri ne anlama geliyor?

    > **Beklenen:** `Supplier<T>`: mesaj üretici (source/producer). Periyodik çağrılır veya StreamBridge ile tetiklenir → output binding'e mesaj yollar. `Consumer<T>`: mesaj tüketici (sink). Input binding'den mesaj alır → işler. `Function<T, R>`: processor. Input alır, transform eder, output'a yollar (pipe). Önemli: metod adı binding isminin bir parçası olur: `myFunction-in-0`, `myFunction-out-0`.

17. StreamBridge ne zaman kullanılır?

    > **Beklenen:** Supplier bean'i periyodik polling ile çalışır — her N milisaniyede poll eder. Ama REST endpoint'ten "şu an" mesaj göndermek istiyorsan → StreamBridge. Örnek: sipariş oluşturuldu → `streamBridge.send("orderCreatedProducer-out-0", event)`. Supplier'dan farklı: event-driven, istek geldiğinde tetiklenir. En yaygın production pattern: REST + StreamBridge → Kafka.

---

**Senior / Architect:**

18. Spring Cloud Stream ile direkt Spring Kafka kullanımı arasında nasıl seçim yaparsın?

    > **Beklenen:** Stream: broker bağımsız kod, farklı projelerde farklı broker kullanılıyor, basit pub/sub yeterli, hızlı prototip. Direkt Spring Kafka: Kafka Streams (stateful processing, windowing), Kafka transaction (exactly-once), compacted topic, consumer group offset yönetimi, partition stratejisi önemli, debug kolaylığı. Tavsiye: yeni proje, Kafka spesifik özellik → direkt spring-kafka. Mevcut Stream kodu, binder değişikliği ihtimali → Stream.

---

## Bölüm 5: Spring Cloud Vault

### Gerçek Hayat Sorunları

---

**Sorun 12: Vault token expire — uygulama başlamıyor**

```
Senaryo:
  VAULT_TOKEN=s.abc123 env variable olarak set edildi.
  Token TTL: 24 saat.
  Servis haftalarca çalıştı → sorun yok (uygulama başlarken token kullandı).
  Yeni deployment: 25 saat sonra → token expired → Vault auth fail → startup fail.

Sorun: Long-lived static token kullanımı.

Düzeltme 1: AppRole Authentication
  role-id: stabil (uzun ömürlü)
  secret-id: kısa ömürlü (CI/CD'de üretilir, her deployment yeni)
  Uygulama: role-id + secret-id → Vault token üretir → kısa TTL ama auto-renew.

Düzeltme 2: Kubernetes Authentication (önerilen)
  Pod'un Kubernetes ServiceAccount token'ı → Vault'a doğrulama.
  ServiceAccount token: K8s tarafından otomatik yönetilir.
  Uygulama başlarken: /var/run/secrets/kubernetes.io/serviceaccount/token → Vault.
  Vault: "Bu token, bu namespace'te bu ServiceAccount'a ait → valid" → izin ver.
  
  Avantaj: Manuel token yönetimi yok, rotation otomatik, K8s native.

Vault Agent Sidecar:
  Vault Agent pod içinde sidecar olarak çalışır.
  Token renewal otomatik — uygulama token expire'ı bilmez bile.
```

---

**Sorun 13: Dynamic secret — DB connection pool sürpriz kapanması**

```
Senaryo:
  Vault dynamic secret: PostgreSQL credentials, TTL: 1 saat.
  HikariCP: connection pool oluşturuldu, credentials ile bağlantılar açık.
  1 saat sonra: Vault PostgreSQL kullanıcısını sildi.
  Mevcut bağlantılar: hâlâ açık (DB'deki session devam ediyor).
  Yeni bağlantı denemesi: "kullanıcı yok" → fail → HikariPool hatası.

Çözüm:
  Vault lease TTL > connection pool max-lifetime.
  HikariCP max-lifetime: 1800000ms (30dk) → Vault TTL: 2 saat.
  Bu süre farkı: Vault yenilemesi için window sağlar.

  Vault Agent: lease renewal otomatik → TTL bitmeden yeniler.
  Spring Cloud Vault: @RefreshScope + credential refresh → pool recreate.
  
  Production best practice:
    Vault TTL: 1 saat
    Vault Agent renewal threshold: %75 (45dk'da yenile)
    HikariCP max-lifetime: 30dk
    → Kullanıcı her zaman geçerli, bağlantılar yenilenmeden önce kapanır.
```

---

### Mülakat Soruları — Vault

**Junior / Mid:**

19. Vault dynamic secret nedir? Static secret'tan farkı?

    > **Beklenen:** Static: bir kez tanımlanır, değişmez (veya manuel rotate edilir). Dynamic: her uygulama başlatıldığında Vault geçici kullanıcı oluşturur (PostgreSQL, MySQL, AWS IAM). TTL dolunca Vault kullanıcıyı siler. Avantaj: sızdırılsa kısa süre geçerli, her deployment farklı credential, tam audit ("kim ne zaman hangi credential kullandı"), rotation otomatik. Dezavantaj: TTL yönetimi, Vault + DB koordinasyonu gerekir.

20. Kubernetes ortamında Vault authentication için hangi yöntem önerilir?

    > **Beklenen:** Kubernetes Auth Method: Pod'un ServiceAccount JWT'si Vault'a gönderilir, Vault Kubernetes API ile doğrular. Avantaj: manuel token yönetimi yok, K8s native, role tabanlı (namespace + serviceaccount = izin). AppRole alternatif: CI/CD'de secret-id üretilir, daha genel ortamlar için. Token auth: en basit ama süresi dolan token riski. Production öneri: Kubernetes Auth + Vault Agent Sidecar (token renewal otomatik).

---

**Senior / Architect:**

21. Vault'u Spring Cloud Config'le veya K8s Secret'la nasıl karşılaştırırsın?

    > **Beklenen:** K8s Secret: base64 encode (şifrelenmiş değil!), etcd'de etcd encryption gerekir, RBAC ile erişim. Basit ama Vault kadar güçlü değil. Spring Cloud Config + Vault backend: Config Server Vault'tan çeker, servislere Config Server protokolüyle sunar. Katman ekledi. Direkt Spring Cloud Vault: uygulama direkt Vault'a bağlanır, Config Server gereksiz. Vault güçlü yönleri: dynamic secret, fine-grained policy, full audit log, cross-platform. Seçim: compliance + audit + dynamic secret → Vault. Basit K8s native ortam → K8s Secret + encrypted etcd.

---

## Bölüm 6: Spring Cloud Eureka

### Gerçek Hayat Sorunları

---

**Sorun 14: Self-preservation mode — ölü instance'lar kayıttan silinmiyor**

```
Senaryo:
  Network problemi: tüm servisler Eureka'ya heartbeat gönderemedi.
  Eureka: "heartbeat oranı %15'in altına düştü → self-preservation modu aktif"
  Self-preservation: registry'yi temizleme, eski instance'ları silme.
  Network düzeldi ama Eureka hâlâ ölü instance'ları gösteriyor.
  order-service → inventory-service çağrısı → ölü instance'a gitti → bağlantı hatası.

Neden self-preservation var?
  Network split senaryosu: Eureka birçok heartbeat alamadı.
  "Bunlar gerçekten öldü mü, yoksa network partition mı?"
  Agresif silme → healthy instance'ları sil → kötü sonuç.
  Self-preservation: şüphe durumunda koru.

Sorun: Recovery yavaş — ölü instance'lar uzun süre kayıtta kalıyor.

Ayarlar:
  # Eureka Server
  eviction-interval-timer-in-ms: 5000    # default: 60sn → 5sn'e düşür
  renewal-percent-threshold: 0.49        # %49 → self-preservation daha geç tetiklenir

  # Eureka Client
  lease-renewal-interval-in-seconds: 10  # heartbeat sıklığı (default: 30)
  lease-expiration-duration-in-seconds: 30  # bu süre heartbeat gelmezse sil (default: 90)

Kubernetes'te:
  Eureka yerine K8s Service → readinessProbe başarısız pod → endpoint'ten çıkar.
  Çok daha hızlı (<10sn) ve güvenilir.
```

---

**Sorun 15: Eureka'da stale instance — yeni deployment eski instance'a yönlendiriyor**

```
Senaryo:
  inventory-service: 10.0.0.5:8080 (eski version) → 10.0.0.6:8080 (yeni version)
  Eski pod kapatıldı ama Eureka registry'si temizlenmeden önce
  order-service Eureka'dan eski adresi aldı → bağlantı hatası.

Timeline:
  T+0: Yeni pod başlatıldı, Eureka'ya kayıt oldu.
  T+30s: Eski pod kapatıldı.
  T+90s: Eski instance Eureka'dan silindi (lease-expiration).
  T+0 ile T+90s arası: Eureka her iki instance'ı da gösteriyor.

Sorun: Propagation delay (30-90 saniye).

Düzeltme:
  1. Graceful shutdown: Pod kapanmadan önce Eureka'ya de-register et.
     eureka.instance.hostname → önce DOWN status → sonra kapat.
     Spring Boot: /actuator/service-registry ile DOWN işaretle.
  
  2. Kubernetes: Feign + Kubernetes Service (Eureka değil).
     K8s endpoint controller: pod terminating → endpoint'ten hemen çıkar.
     ~5sn propagation vs Eureka'nın 30-90sn'si.
  
  3. Client-side retry: stale instance'a bağlantı hataları için retry + next instance.
     Spring Cloud LoadBalancer: başarısız instance'ı geçici olarak blacklist et.
```

---

### Mülakat Soruları — Eureka

**Junior / Mid:**

22. Eureka self-preservation modu nedir? Ne zaman açılır?

    > **Beklenen:** Eureka belirli sürede beklenen heartbeat'in %85'inden azını alırsa → "network partition olabilir, agresif silme yapma" → self-preservation. Ölü gibi görünen instance'ları kayıttan çıkarmaz. Sorun: gerçekten ölü instance'lar da korunur → client onlara istek gönderir → hata. Çözüm: threshold'u düşür veya lease-expiration süresini kısalt. Kubernetes'te nadiren gerekir.

23. Eureka ile Kubernetes Service Discovery farkı nedir?

    > **Beklenen:** Eureka: client-side discovery — client registry'den instance listesi çeker, kendi load balance eder. Kubernetes Service: server-side discovery — kube-proxy ve iptables/ipvs yönetir, client sadece service DNS adresine bağlanır. K8s avantajları: native, sidecar gerekmez, readinessProbe ile hızlı unhealthy pod'ları çıkarır, dil bağımsız. Eureka: Kubernetes dışı VM ortamlar, Spring Cloud ekosistemi ile derin entegrasyon. Kubernetes ortamında Eureka genellikle gereksizdir.

---

**Senior / Architect:**

24. Client-side load balancing ile server-side load balancing trade-off'ları nelerdir?

    > **Beklenen:** Client-side (Eureka + Spring Cloud LoadBalancer): client instance listesini bilir, kendi seçer. Avantaj: çok esneklik (sticky session, zone-aware routing, custom algorithm). Dezavantaj: her client registry bilgisini güncel tutmak zorunda, stale data riski. Server-side (Kubernetes Service, NGINX, AWS ALB): merkezi karar. Avantaj: client basit, tek adres. Dezavantaj: ek hop, merkezi yük. Hybrid: Kubernetes Service + Istio service mesh (L7 load balancing, retry, circuit break) — en güçlü kombinasyon.

---

## Bölüm 7: Spring Cloud Circuit Breaker (Resilience4j)

### Gerçek Hayat Sorunları

---

**Sorun 16: Yanlış CB parametresi — çok hassas, sürekli açılıyor**

```
Senaryo:
  sliding-window-size: 5
  failure-rate-threshold: 20   → 5 çağrıda 1 hata = %20 = CB OPEN

  Üretim: inventory-service bazen geçici 500ms timeout (normal dalgalanma).
  5 çağrıda 1 hata → CB anında OPEN.
  wait-duration-in-open-state: 60s → 60sn hiç istek gitmiyor.
  Kullanıcı 60sn boyunca stok bilgisi göremedi (fallback: "stok bilinmiyor").

Fazla hassas CB → false positive → gereksiz degraded mode.

Düzeltme:
  sliding-window-size: 20        # daha geniş pencere, geçici spike'ları absorbe eder
  failure-rate-threshold: 50     # %50 hata → gerçekten sorun var
  minimum-number-of-calls: 10   # karar için en az 10 çağrı (az örnekle karar verme)
  slow-call-duration-threshold: 3s  # 3sn'den yavaş = yavaş çağrı
  slow-call-rate-threshold: 70   # %70 yavaş → OPEN
  wait-duration-in-open-state: 15s  # 15sn sonra HALF_OPEN dene

# Kural: CB konfigürasyonu load test ile belirle, varsayıma göre değil.
```

---

**Sorun 17: CB fallback — stale data gösterilmesi fark edilmedi**

```
Senaryo:
  product-service CB OPEN → fallback → Redis cache'ten ürün bilgisi dönüyor.
  Cache TTL: 24 saat.
  Ürün fiyatı güncellendi (campaign ended) → product-service'de yeni fiyat.
  CB 30 dakika OPEN kaldı → müşteriler 30dk boyunca eski (düşük) fiyatı gördü.
  Sipariş verildi → gerçek fiyat farklı → müşteri şikayeti.

Sorun: Fallback veri kaynağının ne kadar "stale" olabileceği bilinmiyordu.

Çözüm:
  1. Fallback'te stale olduğunu göster: UI'da "Fiyat güncel olmayabilir" banner.
  
  2. Cache TTL'i kıs: 5dk → en fazla 5dk stale.
  
  3. Fiyat-kritik işlem → CB open'da fallback değil, hata döndür.
     Kullanıcı "şu an işlem yapılamıyor" görür ama yanlış fiyatla sipariş vermez.
  
  4. CB event listener: OPEN durumunda alert → ops ekibi devreye girer.
     circuitBreakerRegistry.circuitBreaker("product-service")
         .getEventPublisher()
         .onStateTransition(event -> alertService.notify(event));
```

---

**Sorun 18: Cascade Circuit Breaker açılması — tüm servisler birbirini etkiliyor**

```
Senaryo:
  Sistem: order → inventory → warehouse → shipping
  warehouse-service çöktü.
  inventory CB: warehouse çağrısı fail → inventory OPEN.
  order CB: inventory çağrısı fail (CB OPEN → CallNotPermittedException) → order OPEN.
  Kullanıcı: hiçbir işlem yapamıyor. Kök neden: warehouse, ama tüm chain etkilendi.

Gerçek kaskad başarısızlığı örneği.

Çözümler:
  1. Bulkhead pattern: Her servis için ayrı thread pool.
     warehouse-service yavaşlasa bile inventory-service'in thread pool'u dolmuyor.
     Sadece warehouse çağrısı için ayrılmış thread'ler tükenir.

  2. Kritik path vs opsiyonel: Stok kontrolü opsiyonelse → CB open'da "stok var say" fallback.
     Kargo bilgisi kritikse → CB open'da sipariş alma.

  3. Saga pattern: Uzun iş akışlarını atomik olmayan adımlara böl.
     Her adım bağımsız CB → tek başarısızlık tüm chain'i durdurmuyor.

  4. Observability: distributed tracing (Zipkin/Jaeger) → hangi servis kök neden?
     Tüm chain'i görmeden root cause bulmak zor.
```

---

### Mülakat Soruları — Circuit Breaker

**Junior / Mid:**

25. Circuit Breaker state machine'ini açıkla. HALF_OPEN ne işe yarar?

    > **Beklenen:** CLOSED: normal çalışma, hatalar sayılır. OPEN: hata eşiği aşıldı, hiçbir istek geçmiyor — hızlı fail, fallback. HALF_OPEN: wait_duration sonrası, "iyileşti mi?" test modeli. `permitted-calls-in-half-open-state` kadar istek geçer. Başarılıysa → CLOSED. Başarısızsa → tekrar OPEN. Neden HALF_OPEN lazım: doğrudan CLOSED'a geçsek → servis hâlâ hasta → yeniden OPEN → waste. HALF_OPEN: kontrollü test.

26. `slow-call-rate-threshold` neden önemlidir?

    > **Beklenen:** Servis yavaş ama hata fırlatmıyor: timeout olmayan 10sn response. Failure rate: 0% (hata yok). Ama thread'ler bloklanıyor. slow_call_rate: "X saniyeden uzun çağrı oranı %Y'yi geçerse → OPEN". Yavaş servis de başarısız servis kadar tehlikeli — thread havuzu tüketir. Bu threshold olmadan sadece exception'lara bakılır, yavaş servis fark edilmez.

27. `@CircuitBreaker` annotation'ı mı, `CircuitBreakerRegistry` programatik mı? Ne zaman hangisi?

    > **Beklenen:** Annotation: deklaratif, temiz kod, AOP proxy üzerinden — Spring bean metodlarına uygulanır. Dezavantaj: self-invocation sorun (proxy bypass), private metoda uygulanamaz. Programatik: tam kontrol, non-Spring nesnelerine uygulanır, conditional CB (bazı durumlarda CB kullan, bazında kullanma). Öneri: çoğu senaryo için annotation yeterli. Dinamik CB name veya non-Spring context → programatik.

---

**Senior / Architect:**

28. Bulkhead pattern Circuit Breaker'dan nasıl farklıdır? Ne zaman birlikte kullanılır?

    > **Beklenen:** Circuit Breaker: "Bu servis çalışıyor mu?" — hata oranına göre açılır/kapanır. Bulkhead: "Bu servis için kaç kaynak ayıracağım?" — thread pool veya semaphore ile. Birlikte: Bulkhead → warehouse için max 10 thread. CB → warehouse hata oranı %50 → OPEN. Bulkhead: kaynak izolasyonu (bir servisin yavaşlaması diğerini etkilemez). CB: başarısız servisi hızlı reddet. Her ikisi: hem izolasyon hem hızlı fail. Örnek: inventory 10 thread, warehouse 10 thread → warehouse dolar, inventory thread'leri serbest.

29. Bir mikrosервisle bağlantı kurulduktan sonra Circuit Breaker açıldığında kullanıcı deneyimi nasıl yönetilir?

    > **Beklenen:** Graceful degradation stratejileri: (1) Cache fallback: son bilinen değeri göster (stale label ile). (2) Default/empty response: "Şu an mevcut değil" — işlemi engelleme. (3) Partial response: diğer veriler göster, eksik kısım placeholder. (4) Queue: işlemi kuyruğa al, servis düzelince işle (async). (5) Feature flag: CB open → feature disabled. Senaryo bağımlı: fiyat → hata mesajı (yanlış fiyat riski). Ürün listesi → cache. Ödeme → kesin fail, belirsizlik risk. UX: kullanıcıya dürüst mesaj > sonsuz yükleme spinner.

---

## Karma — Architect Seviyesi

30. **"Microservice mimarisinde bir isteğin tüm servislerden geçmesini nasıl izlersin?"**

    > **Beklenen:** Distributed tracing: her request'te unique trace ID üret (gateway'de). Feign interceptor / Kafka header → trace ID'yi downstream'e ilet. Spring Cloud Sleuth (deprecated) veya Micrometer Tracing: otomatik trace ID propagation. Zipkin / Jaeger: trace collector. Grafana Tempo: Prometheus ile entegre. Log correlation: trace ID'yi MDC'ye koy → tüm log satırlarında trace ID → Kibana'da "bu trace'e ait tüm loglar". Örnek: `X-Trace-Id: abc123` header → gateway, order-service, inventory-service loglarında aynı ID.

31. **"Gateway → Feign → Circuit Breaker → Retry zincirine düşen bir istek 503 aldı. Nasıl debug edersin?"**

    > **Beklenen:** (1) Trace ID ile distributed trace'i aç → hangi servis 503 verdi? (2) CB durumu: `GET /actuator/circuitbreakers` → OPEN mu? Hangi servis? (3) Retry sayısı: `resilience4j_retry_calls_total` metric → kaç kez denendi? (4) Gateway log: hangi upstream'e yönlendirdi, hangi HTTP kodu aldı? (5) Feign log level FULL → request/response detayı. (6) Upstream servis log: gerçekten 503 üretiyor mu, yoksa timeout mu? (7) HikariCP metric: pool tükenmiş mi? (8) GC log: Full GC pause mı oldu? Sistematik: client → gateway → CB → upstream → DB sırasıyla eleme.

32. **"Bir startup şirketi için microservice altyapısı kuracaksın. Spring Cloud'un hangi bileşenlerini seçersin, hangilerini Kubernetes'e bırakırsın?"**

    > **Beklenen:** Kubernetes'e bırak: Service Discovery (Eureka yerine K8s Service + CoreDNS), Load Balancing (kube-proxy), Config basit ise (ConfigMap/Secret). Spring Cloud'dan seç: API Gateway (Spring Cloud Gateway → Nginx/Kong alternatif ama Spring entegrasyonu güçlü), Circuit Breaker (Resilience4j → Istio service mesh alternatif ama learning curve), Vault (K8s Secret + Vault Agent → production grade secret). İdeal stack: K8s + Spring Cloud Gateway + Resilience4j + Vault + Micrometer Tracing. Erken aşama: Spring Cloud Gateway + Resilience4j minimal. Ölçeklendikçe: Istio service mesh → CB, retry, mTLS K8s'e taşınır.

33. **"Feign timeout → Circuit Breaker → Retry kombinasyonunu idempotency gözetilerek nasıl tasarlarsın?"**

    > **Beklenen:** POST /payments: idempotent değil → retry tehlikeli. Çözüm: (1) İstemci UUID idempotency key üretir, header'a ekler. (2) Gateway request interceptor → `Idempotency-Key` header ekler. (3) Feign interceptor → header'ı taşır. (4) Payment service: aynı key görürse → DB'den önceki sonucu döndür (işlemi tekrar yapma). (5) CB + Retry aktif olabilir → her deneme aynı key → payment service duplicate'i engeller. GET için: retry güvenli (side effect yok). Kural: retry sadece idempotent veya idempotency mekanizması olan işlemler.
