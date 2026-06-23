# 02d — Spring Cloud: Tüm Modüller

Her modül şu 5 soruya göre yapılandırılmıştır:
**Ne?** · **Neden?** · **Nasıl?** · **Ne zaman?** · **Trade-off?**

---

## 1. Spring Cloud Gateway

### Ne?
Microservice mimarisinde tüm gelen trafiğin geçtiği API Gateway. Routing, rate limiting, authentication, circuit breaking ve request/response dönüşümü gibi çapraz kesme (cross-cutting) görevleri tek noktada yönetir. Spring WebFlux üzerine inşa edilmiştir — non-blocking.

### Neden?
Her microservice'e ayrı ayrı rate limiting, JWT doğrulama, CORS, SSL termination eklemek hem tekrarlı hem tutarsızdır. API Gateway bu ortak görevi merkezi olarak üstlenir; servisler sadece iş mantığına odaklanır.

### Nasıl?
```
Client → API Gateway
            ├── Route eşleştir (path, method, header, host)
            ├── Filter chain uygula (pre-filters)
            │       ├── JWT doğrula
            │       ├── Rate limit kontrol et
            │       └── Request header ekle/değiştir
            ├── Upstream servise forward et (lb://order-service)
            ├── Filter chain (post-filters)
            │       └── Response header ekle/değiştir
            └── Client'a cevap döndür
```

**Route konfigürasyonu:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service          # Eureka/K8s service discovery
          predicates:
            - Path=/api/orders/**          # bu path ile eşleşen
            - Method=GET,POST,PUT,DELETE   # bu HTTP metodları
            - Header=X-API-Version, v2     # bu header varsa
          filters:
            - StripPrefix=1                # /api/orders → /orders
            - AddRequestHeader=X-Gateway-Source, true
            - AddResponseHeader=X-Response-Time, now
            - RequestRateLimiter:          # Redis token bucket
                redis-rate-limiter.replenishRate: 100   # token/sn
                redis-rate-limiter.burstCapacity: 200   # max burst
                key-resolver: "#{@userKeyResolver}"      # kim için sınır?
            - CircuitBreaker:
                name: order-service-cb
                fallbackUri: forward:/fallback/orders    # CB açıksa buraya
            - Retry:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE,GATEWAY_TIMEOUT
                methods: GET               # sadece idempotent metodlarda retry

        - id: static-assets
          uri: https://cdn.example.com
          predicates:
            - Path=/static/**
          filters:
            - RewritePath=/static/(?<segment>.*), /${segment}
```

**Custom Global Filter (tüm route'lara uygulanır):**
```java
@Component
@Order(-1)  // diğer filter'lardan önce çalış
class JwtAuthenticationFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // Public endpoint'leri atla
        if (path.startsWith("/api/auth/") || path.startsWith("/actuator/")) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return jwtValidator.validate(authHeader.substring(7))
            .flatMap(claims -> {
                // Downstream servise kullanıcı bilgilerini header ile ilet
                ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Roles", String.join(",", claims.getRoles()))
                    .build();
                return chain.filter(exchange.mutate().request(mutatedRequest).build());
            })
            .onErrorResume(ex -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }
}
```

**Rate Limiter Key Resolver:**
```java
@Bean
KeyResolver userKeyResolver() {
    // JWT'den user ID al → her kullanıcı için ayrı rate limit
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    ).defaultIfEmpty("anonymous");
}

@Bean
KeyResolver ipKeyResolver() {
    // IP bazlı rate limit (DDoS koruması)
    return exchange -> Mono.just(
        Objects.requireNonNull(exchange.getRequest().getRemoteAddress())
               .getAddress().getHostAddress()
    );
}
```

### Ne zaman?
- Birden fazla microservice'e tek giriş noktası gerekiyorsa
- Cross-cutting concern'ler merkezi yönetilmeli (auth, rate limit, logging)
- SSL termination, CORS tek noktada yapılacaksa
- API versioning (path prefix veya header ile routing)

### Ne zaman kullanma?
- Tek servis varsa — gereksiz karmaşıklık
- Servisler direkt birbirini çağırıyorsa (internal communication) — Feign/gRPC kullan

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Cross-cutting concern merkezi | Ek hop → latency (~1ms) |
| Servisler temiz kalır | SPOF olabilir (HA kurulumu şart) |
| WebFlux → yüksek throughput | Config karmaşıklaşabilir |
| Rate limiting, CB yerleşik | Debugging: gateway mi yoksa servis mi hata verdi? |

---

## 2. Spring Cloud Config

### Ne?
Tüm microservice'lerin konfigürasyon dosyalarını merkezi bir yerde (Git repo, Vault, veritabanı) tutan ve her servise HTTP üzerinden sunan merkezi konfigürasyon servisi.

### Neden?
10 microservice × 3 environment (dev/staging/prod) = 30 ayrı konfigürasyon dosyası. Bir DB şifresi değiştiğinde 10 servisi yeniden deploy etmek zorunda kalmak yerine Config Server'da tek bir değişiklik yap → tüm servisler güncellenir. Ayrıca konfigürasyonlar kod repository'sinden ayrılır, auditable olur.

### Nasıl?
```
Git Repository:
├── application.yml              → tüm servisler için ortak default
├── application-prod.yml         → prod environment için ortak override
├── order-service.yml            → sadece order-service
├── order-service-staging.yml    → order-service + staging
└── inventory-service.yml        → sadece inventory-service

Config Server:
  Request: GET /{app-name}/{profile}/{label}
  Örnek:   GET /order-service/prod/main

Öncelik (yüksekten düşüğe):
  order-service-prod.yml
  order-service.yml
  application-prod.yml
  application.yml
```

**Config Server kurulumu:**
```java
@SpringBootApplication
@EnableConfigServer
class ConfigServerApplication { }
```

```yaml
# Config Server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'  # her servis kendi klasöründe
          username: ${GIT_USERNAME}      # private repo için
          password: ${GIT_PASSWORD}
          clone-on-start: true           # startup'ta clone et
          timeout: 10
```

**Client konfigürasyonu:**
```yaml
# order-service bootstrap.yml (veya application.yml + spring.config.import)
spring:
  config:
    import: "configserver:http://config-server:8888"
  application:
    name: order-service    # hangi config dosyası okunacak
  profiles:
    active: prod           # hangi profile
  cloud:
    config:
      fail-fast: true      # Config Server'a ulaşamazsa başlama
      retry:
        max-attempts: 6
        initial-interval: 1000
```

**Hot Reload — servisi yeniden başlatmadan config güncelle:**
```java
// 1. @RefreshScope — bu bean config değiştiğinde yeniden oluşturulur
@RefreshScope
@Service
class FeatureFlagService {
    @Value("${feature.new-algorithm.enabled:false}")
    private boolean newAlgorithmEnabled;

    @Value("${pricing.discount.rate:0.0}")
    private double discountRate;
}

// 2. Güncelleme nasıl tetiklenir?
// Manuel: POST /actuator/refresh (tek servis)
// Otomatik: Spring Cloud Bus ile broadcast

// Spring Cloud Bus (tüm servislere yay):
// Config değişikliği → Git webhook → Config Server → Kafka/RabbitMQ → Bus event
// → Tüm servisler /actuator/busrefresh alır → @RefreshScope bean'ler yenilenir
```

**Environment-specific secret yönetimi:**
```yaml
# Config Server → Vault backend ile secret'ları sakla
spring:
  cloud:
    config:
      server:
        vault:
          host: vault.internal
          port: 8200
          kvVersion: 2
# Artık application.yml'deki şifreler Vault'tan gelir
```

### Ne zaman?
- Birden fazla microservice, birden fazla environment
- Konfigürasyonu kod repo'sundan ayırmak istiyorsun
- Runtime konfigürasyon güncellemesi gerekiyor
- Audit trail (kim hangi config'i ne zaman değiştirdi — Git history)

### Ne zaman kullanma?
- Tek servis, tek environment → application.yml yeterli
- Kubernetes kullanıyorsan → ConfigMap + Secret daha native

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Merkezi yönetim | Config Server'a bağımlılık (SPOF) |
| Git history = audit trail | Startup sırasında ulaşılamazsa sorun |
| Hot reload (@RefreshScope) | @RefreshScope tüm bean'lere uygulanmaz |
| Environment isolation | Ekstra altyapı bileşeni |

---

## 3. Spring Cloud OpenFeign

### Ne?
Interface tanımlayarak HTTP client oluşturmayı sağlayan declarative REST client. Annotation'larla uzak servis metodunu sanki yerel bir Java metodu gibi çağırırsın.

### Neden?
`RestTemplate` veya `WebClient` ile HTTP çağrısı yazmak tekrarlıdır: URL oluştur, body serialize et, header ekle, hata yakala, response deserialize et. Feign bunu bir interface'e indirgeer; implementasyonu otomatik üretir.

### Nasıl?
```java
// 1. Interface tanımla — Spring MVC annotation'ları kullanılır
@FeignClient(
    name = "inventory-service",               // service discovery adı
    url = "${inventory.service.url:}",        // discovery yoksa direct URL
    fallbackFactory = InventoryFallbackFactory.class,
    configuration = InventoryFeignConfig.class
)
interface InventoryClient {

    @GetMapping("/inventory/{productId}")
    InventoryDTO getInventory(@PathVariable String productId);

    @PostMapping("/inventory/reserve")
    ReservationResponse reserve(@RequestBody ReservationRequest request);

    @DeleteMapping("/inventory/reservation/{reservationId}")
    void cancel(@PathVariable String reservationId);

    @GetMapping("/inventory")
    Page<InventoryDTO> listInventory(
        @RequestParam(required = false) String category,
        @RequestParam(defaultValue = "0") int page,
        Pageable pageable  // Spring Data Pageable desteği
    );
}

// 2. Kullan — yerel bean gibi inject et
@Service
class OrderService {
    @Autowired InventoryClient inventoryClient;

    void placeOrder(Order order) {
        InventoryDTO inventory = inventoryClient.getInventory(order.getProductId());
        if (inventory.getStock() < order.getQuantity()) {
            throw new InsufficientStockException();
        }
        inventoryClient.reserve(new ReservationRequest(order.getProductId(), order.getQuantity()));
    }
}

// 3. Fallback Factory — circuit breaker açıksa ne yapılsın?
@Component
class InventoryFallbackFactory implements FallbackFactory<InventoryClient> {
    @Override
    public InventoryClient create(Throwable cause) {
        log.warn("Inventory service unavailable: {}", cause.getMessage());
        return new InventoryClient() {
            @Override
            public InventoryDTO getInventory(String id) {
                return InventoryDTO.unavailable(id);  // fallback response
            }
            @Override
            public ReservationResponse reserve(ReservationRequest req) {
                throw new ServiceUnavailableException("Inventory service down");
            }
            @Override
            public void cancel(String id) { /* best effort */ }
            @Override
            public Page<InventoryDTO> listInventory(String cat, int page, Pageable p) {
                return Page.empty();
            }
        };
    }
}

// 4. Custom konfigürasyon
class InventoryFeignConfig {

    @Bean
    Logger.Level feignLogLevel() {
        return Logger.Level.FULL; // NONE, BASIC, HEADERS, FULL
    }

    @Bean
    RequestInterceptor authInterceptor() {
        return template -> {
            // Her isteğe servis token ekle (internal auth)
            template.header("X-Internal-Token", internalTokenProvider.get());
        };
    }

    @Bean
    Retryer retryer() {
        return new Retryer.Default(100, 1000, 3); // period, maxPeriod, maxAttempts
    }
}
```

**Konfigürasyon (application.yml):**
```yaml
feign:
  circuitbreaker:
    enabled: true
  client:
    config:
      default:                    # tüm Feign client'lar
        connectTimeout: 2000
        readTimeout: 5000
      inventory-service:          # bu client'a özel
        connectTimeout: 500
        readTimeout: 2000
        loggerLevel: BASIC

spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true           # request body gzip
        response:
          enabled: true           # response body gzip decompress
```

### Ne zaman?
- Servisler arası senkron HTTP iletişimi
- Service discovery ile dinamik URL çözümleme
- Circuit breaker entegrasyonu gerekiyorsa
- Mock edilmesi kolay client gerekiyorsa (interface → test'te Mockito ile mock)

### Ne zaman kullanma?
- Yüksek throughput → gRPC daha verimli
- Async iletişim → Kafka/RabbitMQ
- Blocking olmayan reactive stack → WebClient

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Çok az boilerplate | HTTP soyutlaması → debug karmaşıklaşabilir |
| Interface → test'te mock kolay | Blocking HTTP (WebFlux ile zor entegrasyon) |
| Spring MVC annotation'ları aynı | Karmaşık auth/signing senaryolarında custom interceptor şart |
| Load balancing + circuit breaker entegre | SOAP/binary protocol desteklemez |

---

## 4. Spring Cloud Stream

### Ne?
Kafka, RabbitMQ gibi mesajlaşma sistemlerine bağlanmak için binder soyutlaması. Broker bağımsız kod yazarsın; binder değiştiğinde kod değişmez.

### Neden?
Kafka ile konuşan kod doğrudan `KafkaTemplate` kullandığında sistemi RabbitMQ'ya geçirmek için kodu yeniden yazmak gerekir. Stream bir abstraction katmanı ekler: sen sadece `Supplier`/`Consumer`/`Function` bean tanımlarsın, altta ne çalıştığını bilmen gerekmez.

### Nasıl?
```
Fonksiyonel programlama modeli:

Supplier<T>           → kaynak: mesaj üretir (producer)
Consumer<T>           → sink: mesaj tüketir (consumer)
Function<T, R>        → işlemci: tüketir + üretir (processor)
```

```java
// Mesaj üret — Supplier
@Bean
Supplier<OrderCreatedEvent> orderCreatedProducer() {
    return () -> {
        // Bu metod periyodik olarak çağrılır (poller ile) veya StreamBridge ile tetiklenir
        return orderQueue.poll();
    };
}

// Mesaj tüket — Consumer
@Bean
Consumer<OrderCreatedEvent> inventoryReserver() {
    return event -> {
        log.info("Rezervasyon yapılıyor: {}", event.getOrderId());
        inventoryService.reserve(event.getProductId(), event.getQuantity());
    };
}

// Tüket + Üret — Function (stream processor)
@Bean
Function<OrderCreatedEvent, PaymentRequestEvent> orderToPayment() {
    return orderEvent -> {
        return new PaymentRequestEvent(
            orderEvent.getOrderId(),
            orderEvent.getCustomerId(),
            orderEvent.getTotalAmount()
        );
    };
}

// Programatik gönderim (REST Controller'dan tetikle)
@Service
class OrderService {
    @Autowired StreamBridge streamBridge;

    void placeOrder(Order order) {
        orderRepo.save(order);
        OrderCreatedEvent event = OrderCreatedEvent.from(order);
        streamBridge.send("orderCreatedProducer-out-0", event);
        // binding name: {functionName}-out-{index}
    }
}
```

**Konfigürasyon — Kafka binder:**
```yaml
spring:
  cloud:
    stream:
      bindings:
        # Supplier → Kafka topic
        orderCreatedProducer-out-0:
          destination: orders.created
          content-type: application/json

        # Consumer ← Kafka topic
        inventoryReserver-in-0:
          destination: orders.created
          group: inventory-service       # consumer group
          consumer:
            max-attempts: 3
            back-off-initial-interval: 1000

        # Function: input ← orders.created, output → payments.request
        orderToPayment-in-0:
          destination: orders.created
          group: payment-transformer
        orderToPayment-out-0:
          destination: payments.request

      kafka:
        binder:
          brokers: kafka:9092
          auto-create-topics: false      # production'da false
        bindings:
          inventoryReserver-in-0:
            consumer:
              startOffset: latest
              resetOffsets: false

# Binder değiştirmek: kafka → rabbit
      binders:
        rabbit:
          type: rabbit
      default-binder: rabbit
```

**Error Handling:**
```java
@Bean
Consumer<Message<OrderCreatedEvent>> inventoryReserver() {
    return message -> {
        try {
            inventoryService.reserve(message.getPayload());
        } catch (StockUnavailableException ex) {
            // Dead Letter Channel'a gönder
            throw new RuntimeException(ex);
        }
    };
}
```

```yaml
spring.cloud.stream.bindings.inventoryReserver-in-0:
  consumer:
    max-attempts: 3
    back-off-initial-interval: 1000
    back-off-multiplier: 2.0
    dead-letter-queue-name: orders.created.DLQ
```

### Ne zaman?
- Farklı projelerde farklı broker (bazıları Kafka, bazıları RabbitMQ)
- Broker bağımsız, taşınabilir mesajlaşma kodu
- Stream processing (Function composition)
- Hızlı prototyping

### Ne zaman kullanma?
- Kafka'ya özgü özellikler gerekiyorsa (compaction, Kafka Streams, transaction) → direkt `spring-kafka`
- Binder overhead istemiyorsan → direkt client daha şeffaf

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Broker bağımsız kod | Soyutlama → broker'a özgü özellikler kısıtlanır |
| Functional model temiz | Binder config karmaşıklaşabilir |
| Binder değiştirmek kolay | Debug: sorun Stream mi, binder mı? |
| Otomatik serialization | Binding isim convention'ı karıştırıcı olabilir |

---

## 5. Spring Cloud Vault

### Ne?
HashiCorp Vault ile Spring Boot entegrasyonu. Secret'ları Vault'tan çeker ve `@Value` ile inject eder; dynamic secret (geçici DB kullanıcı oluşturma) desteği sağlar.

### Neden?
DB şifresi, API key, sertifika gibi hassas verileri `application.yml` veya environment değişkenine koymak güvenlik riski oluşturur. Vault bu secret'ları şifreli, erişim kontrollü ve rotasyonlu şekilde saklar; Spring Boot startup'ta bunları çekip inject eder.

### Nasıl?
```yaml
spring:
  config:
    import: vault://
  cloud:
    vault:
      host: vault.internal
      port: 8200
      scheme: https

      # Authentication yöntemi — Kubernetes (production için önerilen)
      authentication: KUBERNETES
      kubernetes:
        role: order-service          # Vault'ta tanımlı role
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token

      # Alternatif: AWS IAM auth
      # authentication: AWS_IAM

      # KV secrets engine
      kv:
        enabled: true
        backend: secret
        application-name: order-service
        default-context: order-service
        # Vault path: secret/order-service
        #              secret/application  (shared)
```

```java
// Vault'tan otomatik inject — diğer property'lerden farkı yok
@Service
class DatabaseConfig {
    @Value("${database.password}")      // Vault path: secret/order-service → database.password
    private String dbPassword;

    @Value("${external.api.key}")       // Vault path: secret/order-service → external.api.key
    private String apiKey;
}

// Programatik erişim
@Service
class SecretService {
    @Autowired VaultTemplate vaultTemplate;

    Map<String, Object> readSecret(String path) {
        VaultResponse response = vaultTemplate.read("secret/data/" + path);
        return response.getData();
    }

    void writeSecret(String path, Map<String, Object> data) {
        vaultTemplate.write("secret/data/" + path, data);
    }
}
```

**Dynamic Secrets — en güçlü özellik:**
```yaml
spring:
  cloud:
    vault:
      # Vault PostgreSQL engine — her uygulama için geçici kullanıcı oluştur
      database:
        enabled: true
        role: order-service-role        # Vault'ta tanımlı DB role
        backend: database
        username-property: spring.datasource.username
        password-property: spring.datasource.password
```

```
Vault PostgreSQL engine ne yapar?
1. Spring Boot başlar
2. Vault'tan DB credentials ister (role: order-service-role)
3. Vault → PostgreSQL'de geçici kullanıcı oluşturur:
   CREATE USER vault_abc123 WITH PASSWORD 'xyz...' VALID UNTIL '2024-01-01 10:00';
   GRANT SELECT, INSERT, UPDATE ON orders TO vault_abc123;
4. Bu credentials Spring'e inject edilir
5. TTL dolunca Vault kullanıcıyı siler

Faydası:
- Her deployment farklı credential
- Vault log'unda tam erişim kaydı
- Sızdırılmış credential → kısa sürede expire
- Rotation otomatik (uygulama bilmez)
```

**Secret Renewal (Lease):**
```yaml
spring.cloud.vault:
  generic:
    enabled: true
  # Token renewal — Vault token'ı otomatik yenile
  token: ${VAULT_TOKEN}
  # Veya AppRole:
  authentication: APPROLE
  app-role:
    role-id: ${VAULT_ROLE_ID}
    secret-id: ${VAULT_SECRET_ID}
```

### Ne zaman?
- Production'da secret yönetimi gerekiyorsa (her zaman!)
- DB credential rotation
- Servis-servis TLS sertifika yönetimi
- Compliance gereksinimleri (secret audit trail)
- Dynamic secrets ile geçici credential

### Ne zaman kullanma?
- Geliştirme/test ortamı → `.env` dosyası yeterli
- Kubernetes kullanıyorsan → K8s Secret de bir seçenek (ama Vault daha güçlü)

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Secret'lar kod dışında | Vault altyapısı kurulumu/yönetimi kompleks |
| Dynamic secret → çok güvenli | Vault'a bağımlılık (startup'ta erişilmezse sorun) |
| Tam audit trail | Öğrenme eğrisi |
| Otomatik rotation | HA Vault kurulumu gerekir |

---

## 6. Spring Cloud Eureka (Service Discovery)

### Ne?
Microservice'lerin kendilerini kaydettikleri ve birbirlerini adresle bulmalarını sağlayan servis kayıt/keşif sunucusu.

### Neden?
Kubernetes olmayan ortamlarda microservice'ler dinamik IP ve port alır. `order-service`'in `inventory-service`'i çağırmak için sabit IP/port bilmesi gerekirse her deployment'ta konfigürasyon güncellenmeli. Eureka bu dinamik adresleri otomatik yönetir.

### Nasıl?
```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
class EurekaServerApplication {}
```

```yaml
# Eureka Server
eureka:
  client:
    register-with-eureka: false   # server kendini kaydetmesin
    fetch-registry: false
  server:
    eviction-interval-timer-in-ms: 10000   # 10sn'de bir ölü instance'ları temizle
    renewal-percent-threshold: 0.85        # %85 heartbeat gelmezse → self-preservation
```

```yaml
# Eureka Client (her microservice)
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
    registry-fetch-interval-seconds: 30   # registry'yi 30sn'de bir güncelle
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30  # heartbeat sıklığı
    lease-expiration-duration-in-seconds: 90 # bu süre heartbeat gelmezse çıkar
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
```

**Feign + Eureka (client-side load balancing):**
```java
@FeignClient(name = "inventory-service")  // URL değil, Eureka adı
interface InventoryClient {
    @GetMapping("/inventory/{id}")
    InventoryDTO get(@PathVariable String id);
}
// Spring Cloud LoadBalancer: Eureka'dan instance listesi çeker → round-robin
```

```
Request akışı:
1. order-service → inventory-service çağır
2. Spring Cloud LoadBalancer → Eureka'dan inventory-service instance'larını çek
   [{ip: 10.0.0.1, port: 8082}, {ip: 10.0.0.2, port: 8082}]
3. Round-robin seç → 10.0.0.1:8082
4. HTTP request gönder
```

### Ne zaman?
- Kubernetes kullanmayan VM/bare-metal ortamlar
- Docker Compose ile geliştirme (service discovery olmadan sabit port gerekir)
- Spring Cloud ekosisteminin diğer parçalarıyla (Feign, Gateway) entegrasyon

### Ne zaman kullanma?
- Kubernetes kullanıyorsan → K8s Service + DNS yeterli (Eureka redundant olur)
- HashiCorp Consul daha fazla özellik sunar (health check, K/V store, DNS)

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Spring Cloud ile sıkı entegrasyon | Kubernetes'te gereksiz |
| Client-side LB (Spring Cloud LoadBalancer) | Eureka server HA kurulumu gerekir |
| Peer-to-peer replika desteği | Self-preservation mode kafa karıştırıcı |
| Dashboard yerleşik | 30-90sn propagation gecikmesi |

---

## 7. Spring Cloud Circuit Breaker (Resilience4j)

### Ne?
Başarısız veya yavaş servislere yapılan çağrıları kesen, sistemi kaskad başarısızlıktan koruyan circuit breaker pattern'ının Spring Cloud abstraction'ı. Altta Resilience4j kullanır.

### Neden?
`inventory-service` çöktü. `order-service` her siparişte 30 saniye timeout bekliyor → thread'ler tükeniyor → `order-service` de yanıt veremez hale geliyor → kaskad çöküş. Circuit breaker bunu tespit eder ve hatalı servise hemen "hayır" der — timeout beklemez.

### Nasıl?
```
State Machine:
CLOSED → (hata oranı eşiği aşılır) → OPEN
OPEN → (wait duration geçer) → HALF_OPEN
HALF_OPEN → (test çağrı başarılı) → CLOSED
HALF_OPEN → (test çağrı başarısız) → OPEN
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        sliding-window-size: 10           # son 10 çağrıya bak
        failure-rate-threshold: 50        # %50 hata → OPEN
        slow-call-rate-threshold: 80      # %80 yavaş çağrı → OPEN
        slow-call-duration-threshold: 3s  # 3sn+ = yavaş
        wait-duration-in-open-state: 30s  # 30sn sonra HALF_OPEN dene
        permitted-calls-in-half-open-state: 3  # 3 test çağrısı
        minimum-number-of-calls: 5        # karar için minimum çağrı
  timelimiter:
    instances:
      inventory-service:
        timeout-duration: 2s              # 2sn'de cevap gelmezse hata say
  retry:
    instances:
      inventory-service:
        max-attempts: 3
        wait-duration: 500ms
```

```java
// Feign ile otomatik entegrasyon (sadece config yeterli)
@FeignClient(name = "inventory-service",
             fallbackFactory = InventoryFallbackFactory.class)
interface InventoryClient { ... }

// Programatik kullanım
@Service
class InventoryService {

    @CircuitBreaker(name = "inventory-service", fallbackMethod = "getInventoryFallback")
    @TimeLimiter(name = "inventory-service")
    @Retry(name = "inventory-service")
    CompletableFuture<InventoryDTO> getInventory(String productId) {
        return CompletableFuture.supplyAsync(
            () -> inventoryClient.get(productId)
        );
    }

    CompletableFuture<InventoryDTO> getInventoryFallback(String productId, Throwable ex) {
        log.warn("CB open for inventory-service: {}", ex.getMessage());
        return CompletableFuture.completedFuture(InventoryDTO.unavailable(productId));
    }
}
```

**Monitoring:**
```java
// Actuator endpoint'i ile durum izle
// GET /actuator/circuitbreakers
// {
//   "circuitBreakers": {
//     "inventory-service": {
//       "state": "OPEN",
//       "failureRate": 75.0,
//       "slowCallRate": 0.0,
//       "bufferedCalls": 10,
//       "failedCalls": 7
//     }
//   }
// }

// Micrometer metrik: Prometheus'a otomatik
// resilience4j_circuitbreaker_state{name="inventory-service"} → Grafana'da görselleştir
```

### Ne zaman?
- Servisler arası çağrıda timeout/hata riski varsa (her zaman)
- Kaskad başarısızlığı önlemek için
- Yavaş servislerden izolasyon

### Ne zaman kullanma?
- İdempotent olmayan çağrılar + agresif retry → duplicate işlem riski
- Basit uygulama, dış servis yoksa gereksiz

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Kaskad çöküşü önler | Konfigürasyon parametrelerini doğru ayarlamak zor |
| Fallback ile degraded mode | Half-open'da kararlar yanıltıcı olabilir |
| Hızlı fail → thread tüketimi önlenir | Actuator monitoring kurulumu gerekir |
| Retry + timeout + CB kombine | Async (CompletableFuture) zorunluluğu kafa karıştırır |
