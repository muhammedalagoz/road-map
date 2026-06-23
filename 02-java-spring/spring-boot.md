# 02c — Spring Boot: Tüm Modüller

Her modül şu 5 soruya göre yapılandırılmıştır:
**Ne?** · **Neden?** · **Nasıl?** · **Ne zaman?** · **Trade-off?**

---

## 1. Spring Boot Auto-Configuration

### Ne?
Classpath'teki kütüphanelere ve environment'a bakarak bean'leri otomatik oluşturan mekanizma. Sen `pom.xml`'e bağımlılığı eklersin, Spring Boot geri kalanını halleder.

### Neden?
Spring Framework güçlü ama her proje için `DataSource`, `EntityManagerFactory`, `TransactionManager` bean'lerini manuel tanımlamak tekrarlı ve hata eğilimlidir. Auto-configuration bu standart kurulumları ortadan kaldırır, "convention over configuration" prensibini hayata geçirir.

### Nasıl?
```
@SpringBootApplication
    ├── @SpringBootConfiguration   → @Configuration alias
    ├── @ComponentScan             → bu paketin altını tara
    └── @EnableAutoConfiguration   → sihir burada

AutoConfigurationImportSelector:
  1. Classpath'teki tüm jar'larda ara:
     META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  2. Listelenen her AutoConfiguration sınıfını değerlendir
  3. @Conditional koşulları sağlanıyorsa → bean'leri oluştur

Örnek — DataSourceAutoConfiguration:
@AutoConfiguration
@ConditionalOnClass({ DataSource.class })             // classpath'te JDBC var mı?
@ConditionalOnMissingBean(DataSource.class)           // sen tanımlamadın mı?
@EnableConfigurationProperties(DataSourceProperties.class)
class DataSourceAutoConfiguration {
    @Bean
    DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
            .url(props.getUrl())
            .username(props.getUsername())
            .build();
    }
}
```

**@Conditional türleri:**
```java
@ConditionalOnClass(DataSource.class)      // classpath'te bu sınıf varsa
@ConditionalOnMissingClass("org.X")        // classpath'te bu sınıf yoksa
@ConditionalOnBean(CacheManager.class)     // bu bean zaten yaratıldıysa
@ConditionalOnMissingBean(DataSource.class)// bu bean henüz yaratılmadıysa
@ConditionalOnProperty(                    // property varsa ve değeri uyuyorsa
    name = "feature.cache.enabled",
    havingValue = "true",
    matchIfMissing = false)
@ConditionalOnWebApplication               // web uygulamasıysa
@ConditionalOnResource("classpath:banner.txt") // dosya varsa
```

**Öncelik sırası (yüksekten düşüğe):**
```
1. Komut satırı argümanları  (--server.port=9090)
2. SPRING_APPLICATION_JSON  (env var olarak JSON)
3. OS environment variables (SPRING_DATASOURCE_URL)
4. application-{profile}.yml
5. application.yml
6. @PropertySource
7. Default değerler
```

**Debug:**
```bash
java -jar app.jar --debug
# veya
SPRING_OUTPUT_ANSI_ENABLED=always java -jar app.jar --debug

# Çıktı:
# CONDITIONS EVALUATION REPORT
# Positive matches:
#   DataSourceAutoConfiguration matched
#     - @ConditionalOnClass found 'javax.sql.DataSource'
# Negative matches:
#   MongoAutoConfiguration did not match
#     - @ConditionalOnClass 'com.mongodb.MongoClient' not found
```

### Ne zaman?
Her Spring Boot uygulamasında temeldir. Kendi auto-configuration yazmak istiyorsan starter kütüphane geliştirirken kullanılır.

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Hızlı başlangıç, az boilerplate | "Sihir" — ne neden oluştu anlamak zor |
| Classpath'e göre akıllı karar | Yanlış konfigürasyon sessizce kabul edilebilir |
| Override kolay (@Bean ile ezersin) | Büyük classpath → uzun startup (lazy init açılabilir) |
| Test edilebilir (`--debug` ile görünür) | Çakışan auto-config'ler sorun yaratabilir |

---

## 2. Spring Boot Actuator

### Ne?
Çalışan uygulamanın sağlık durumunu, metriklerini, environment değişkenlerini ve iç durumunu HTTP endpoint'leri üzerinden gösteren production monitoring aracı.

### Neden?
Uygulama production'da çalışırken "DB bağlantısı açık mı?", "kaç thread aktif?", "bellek ne durumda?", "son 100 istek ne kadarda cevaplandı?" gibi sorulara cevap gerekir. Log'lardan bunları çıkarmak yerine Actuator hazır endpoint'ler sunar.

### Nasıl?
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,beans,loggers,threaddump
      base-path: /actuator
  endpoint:
    health:
      show-details: always   # DB, Redis, Kafka detayları göster
      show-components: always
  server:
    port: 8081               # Management portunu ayır (güvenlik için)
```

**Kritik endpoint'ler:**
```
GET /actuator/health
  → { "status": "UP", "components": { "db": { "status": "UP" }, "redis": {...} }}
  → Kubernetes liveness/readiness probe burayı kullanır

GET /actuator/metrics
  → Tüm metric isimlerini listeler

GET /actuator/metrics/http.server.requests
  → { "statistic": "COUNT", "value": 1234 }
  → tags: [{"tag":"uri","values":["/api/orders"]}, {"tag":"status","values":["200"]}]

GET /actuator/prometheus
  → Prometheus text formatında tüm metrikler
  → Grafana dashboard bu endpoint'i kullanır

GET /actuator/env
  → Tüm property kaynakları ve değerleri (şifreler *** ile maskelenir)

GET /actuator/beans
  → Container'daki tüm bean'ler, scope'ları, bağımlılıkları

POST /actuator/loggers/com.example.service
  → Body: {"configuredLevel": "DEBUG"}
  → Runtime'da log level değiştir (restart gerekmez!)

GET /actuator/threaddump
  → Tüm thread'lerin stack trace'i (deadlock debug için)

GET /actuator/heapdump
  → heap.hprof indir → JVisualVM/MAT ile analiz et
```

**Custom HealthIndicator:**
```java
@Component
class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Autowired PaymentGatewayClient client;

    @Override
    public Health health() {
        try {
            PingResponse response = client.ping();
            return Health.up()
                .withDetail("responseTime", response.getLatencyMs() + "ms")
                .withDetail("endpoint", response.getEndpoint())
                .build();
        } catch (Exception ex) {
            return Health.down()
                .withDetail("error", ex.getMessage())
                .withDetail("endpoint", client.getEndpoint())
                .build();
        }
    }
}
// Sonuç: /actuator/health → { "paymentGateway": { "status": "DOWN", "details": {...} }}
```

**Custom Metric:**
```java
@Service
class OrderService {
    private final Counter createdCounter;
    private final Timer processingTimer;
    private final Gauge queueSizeGauge;

    OrderService(MeterRegistry registry, OrderQueue queue) {
        createdCounter = Counter.builder("orders.created")
            .tag("region", "eu-west-1")
            .description("Toplam oluşturulan sipariş sayısı")
            .register(registry);

        processingTimer = Timer.builder("orders.processing.duration")
            .description("Sipariş işleme süresi")
            .register(registry);

        queueSizeGauge = Gauge.builder("orders.queue.size", queue, OrderQueue::size)
            .description("Kuyruk boyutu")
            .register(registry);
    }

    void createOrder(Order order) {
        processingTimer.record(() -> {
            process(order);
            createdCounter.increment();
        });
    }
}
```

**Kubernetes entegrasyonu:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 10
  periodSeconds: 5
```

### Ne zaman?
Her production uygulamasında. CI/CD pipeline'ında health check için, Prometheus/Grafana ile monitoring kurulumunda zorunlu.

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Production monitoring hazır | `/actuator/env` hassiz bilgi açığa çıkarabilir — güvenli porta al |
| Kubernetes probe entegrasyonu kolay | Tüm endpoint'ler açıksa güvenlik riski |
| Runtime log level değiştirme | Heapdump gibi endpoint'ler yük oluşturabilir |
| Custom metric/health kolay | Management portu açık unutulursa risk |

---

## 3. Spring Boot DevTools

### Ne?
Geliştirme sürecini hızlandıran araç: kod değişikliğinde otomatik restart, browser'da otomatik yenileme ve geliştirme ortamına özgü property override.

### Neden?
Her kod değişikliğinde uygulamayı manuel durdurup başlatmak geliştirme hızını yavaşlatır. Tam restart 10-30 saniye sürerken DevTools bunu 1-3 saniyeye düşürür.

### Nasıl?
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>  <!-- production jar'a dahil edilmez -->
</dependency>
```

**İki ClassLoader stratejisi:**
```
Base ClassLoader:      Bağımlılıklar (jar'lar) — değişmez, yeniden yüklenmez
Restart ClassLoader:   Uygulama sınıfları — değişince yenilenir

Dosya değişikliği tespit edilince:
  1. Restart ClassLoader yeniden oluşturulur
  2. Uygulama sınıfları yeniden yüklenir
  3. Spring context refresh edilir
  → Cold start: 15-30s vs DevTools restart: 2-4s
```

**Otomatik property override (geliştirme için):**
```
DevTools, geliştirme ortamında şunları otomatik devre dışı bırakır:
  - Template caching (Thymeleaf, Freemarker)
  - Static resource caching
  - H2 console aktif edilir

~/.spring-boot-devtools.properties  → tüm projeler için global config
```

**LiveReload:**
```
DevTools → LiveReload server (35729 portu) başlatır
Browser extension → sunucuya bağlanır
Sayfa değiştiğinde → sunucu push → browser otomatik yenilenir
```

### Ne zaman?
Sadece geliştirme ortamında. `<optional>true</optional>` ile production'a gitmemesi sağlanır; üretim JAR'ında bulunmaz.

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Restart süresi dramatik düşer | Zaman zaman tam restart gerekir (context tutarsız olabilir) |
| Template değişikliklerinde anında görme | Remote sunucuda çalıştırmak önerilmez |
| Otomatik geliştirme property'leri | İki ClassLoader → nadiren ClassCastException |

---

## 4. Spring Boot Test

### Ne?
Spring uygulamalarını çeşitli izolasyon seviyelerinde test etmek için annotation seti ve test altyapısı.

### Neden?
Tüm Spring context'i ayağa kaldırmadan sadece controller'ı veya sadece repository'yi test etmek hem hızlı hem güvenilir test suitleri sağlar. Doğru test katmanını seçmek test süresini 10x kısaltabilir.

### Nasıl?

**Test piramidi ve Spring Boot karşılıkları:**
```
    /\          E2E / Integration Test
   /  \         @SpringBootTest(RANDOM_PORT)
  /----\
 /      \       Slice Tests
/--------\      @WebMvcTest / @DataJpaTest / @DataRedisTest
/          \
/────────────\  Unit Tests
              Mockito (Spring yok, hızlı)
```

**@SpringBootTest — tam integration test:**
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {

    @Autowired TestRestTemplate restTemplate;
    @LocalServerPort int port;

    @Test
    void createOrderReturns201() {
        var request = new CreateOrderRequest("customer-1", List.of("item-1"));
        var response = restTemplate.postForEntity("/api/orders", request, Order.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getId()).isNotNull();
    }
}
// Tüm context yüklenir → gerçekçi ama yavaş (10-30s)
```

**@WebMvcTest — sadece web katmanı:**
```java
@WebMvcTest(OrderController.class)
// Yüklenler: Controller, ControllerAdvice, Filter, WebMvcConfigurer
// Yüklenmeyenler: Service, Repository, Component
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;  // Service mock'lanır

    @Test
    void getOrderReturns200() throws Exception {
        given(orderService.findById(1L)).willReturn(new Order(1L, "ACTIVE", BigDecimal.TEN));

        mockMvc.perform(get("/api/orders/1").accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("ACTIVE"))
            .andExpect(jsonPath("$.total").value(10));
    }

    @Test
    void createOrderWithInvalidBodyReturns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))           // boş body → validation fail
            .andExpect(status().isBadRequest());
    }
}
// Sadece web katmanı → hızlı (1-3s)
```

**@DataJpaTest — sadece JPA katmanı:**
```java
@DataJpaTest
// Yüklenler: Repository, Entity, JPA config
// Yüklenmeyenler: Controller, Service, diğer bean'ler
// H2 in-memory DB otomatik — her test sonrası rollback
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepo;
    @Autowired TestEntityManager em;

    @Test
    void findByCustomerIdReturnsOrders() {
        em.persist(new Order(null, 123L, "ACTIVE", BigDecimal.TEN));
        em.flush();

        List<Order> found = orderRepo.findByCustomerId(123L);

        assertThat(found).hasSize(1);
        assertThat(found.get(0).getStatus()).isEqualTo("ACTIVE");
    }
}
```

**Testcontainers — gerçek bağımlılıklar:**
```java
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15-alpine");

    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired OrderService orderService;

    @Test
    void placeOrderPublishesEvent() {
        // Gerçek PostgreSQL + Gerçek Kafka → integration test
        Order order = orderService.placeOrder(new CreateOrderRequest(...));
        assertThat(order.getId()).isNotNull();
        // Kafka consumer'ı verify et...
    }
}
```

**Test slice'larının karşılaştırması:**
```
Annotation           Yüklenen context              Süre
@SpringBootTest      Tüm context                   Yavaş (10-30s)
@WebMvcTest          Sadece web katmanı             Hızlı (1-3s)
@DataJpaTest         JPA + H2                      Hızlı (2-5s)
@DataRedisTest       Redis (embedded)              Hızlı (1-2s)
@DataMongoTest       MongoDB (embedded)            Hızlı (1-3s)
@RestClientTest      RestTemplate/WebClient mock   Çok hızlı
Unit test (Mockito)  Spring yok                    En hızlı (<1s)
```

### Ne zaman?
- Unit test: Spring'e bağımlılık yoksa her zaman tercih et
- @WebMvcTest: Controller mantığı, validation, serialization test
- @DataJpaTest: Query doğruluğu, custom query test
- @SpringBootTest: Servisler arası entegrasyon, E2E akış
- Testcontainers: Gerçek DB/broker davranışı kritikse

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Katmanlı test → hızlı feedback | @SpringBootTest yavaş — CI pipeline'ı uzatır |
| @MockBean ile bağımlılık izolasyonu | @MockBean her kullanımda yeni context yükler |
| Testcontainers → gerçekçi test | Docker gerektiriyor (CI ortamı desteği gerekir) |
| H2 ile hızlı repository testi | H2 davranışı gerçek DB'den farklı olabilir |

---

## 5. Spring Batch

### Ne?
Büyük veri setlerini toplu işlemek için tasarlanmış çerçeve. Okuma → işleme → yazma adımlarını, hata toleransını ve paralel çalışmayı yönetir.

### Neden?
"Ayın sonunda 10 milyon siparişi işle ve fatura oluştur" gibi toplu iş senaryolarında bireysel request/response modeli çalışmaz. Spring Batch yeniden başlatma, hata atlatma, paralel işleme ve durum takibi sağlar.

### Nasıl?
```
Job
└── Step 1: Dosyadan müşterileri oku
│   ├── ItemReader    → CSV/DB/XML'den chunk chunk oku
│   ├── ItemProcessor → Dönüştür, filtrele, doğrula
│   └── ItemWriter    → DB'ye, dosyaya veya API'ye yaz
└── Step 2: Özet rapor oluştur
    └── Tasklet (tek işlem)
```

```java
@Configuration
class ImportJobConfig {

    @Bean
    Job importOrdersJob(JobRepository repo, Step readStep, Step reportStep) {
        return new JobBuilder("importOrders", repo)
            .start(readStep)
            .next(reportStep)
            .listener(new JobExecutionListener() {
                @Override public void beforeJob(JobExecution je) {
                    log.info("Job başlıyor: {}", je.getJobParameters());
                }
                @Override public void afterJob(JobExecution je) {
                    log.info("Job bitti: {}", je.getStatus());
                }
            })
            .build();
    }

    @Bean
    Step readStep(JobRepository repo, PlatformTransactionManager tm) {
        return new StepBuilder("readStep", repo)
            .<CsvOrder, Order>chunk(500, tm)  // 500'lük batch — tek transaction
            .reader(csvReader())
            .processor(orderProcessor())
            .writer(dbWriter())
            .faultTolerant()
                .skip(ValidationException.class)  // bu hataları atla
                .skipLimit(1000)                  // en fazla 1000 atla
                .retry(DeadlockLoserDataAccessException.class)
                .retryLimit(3)                    // 3 kez dene
            .build();
    }

    @Bean
    FlatFileItemReader<CsvOrder> csvReader() {
        return new FlatFileItemReaderBuilder<CsvOrder>()
            .name("csvReader")
            .resource(new FileSystemResource("orders.csv"))
            .linesToSkip(1)  // header'ı atla
            .delimited()
                .names("orderId", "customerId", "amount", "date")
            .targetType(CsvOrder.class)
            .build();
    }

    @Bean
    ItemProcessor<CsvOrder, Order> orderProcessor() {
        return csvOrder -> {
            if (csvOrder.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
                throw new ValidationException("Negatif tutar: " + csvOrder.getOrderId());
            }
            return new Order(csvOrder.getOrderId(), csvOrder.getCustomerId(),
                           csvOrder.getAmount(), LocalDate.parse(csvOrder.getDate()));
        };
    }

    @Bean
    JpaItemWriter<Order> dbWriter() {
        return new JpaItemWriterBuilder<Order>()
            .entityManagerFactory(entityManagerFactory)
            .build();
    }
}

// Paralel işleme — partition ile
@Bean
Step parallelStep(JobRepository repo, PlatformTransactionManager tm, Step workerStep) {
    return new StepBuilder("parallelStep", repo)
        .partitioner("worker", new RangePartitioner(totalRecords, 10))  // 10 partition
        .step(workerStep)
        .taskExecutor(new SimpleAsyncTaskExecutor())
        .gridSize(10)
        .build();
}
```

**Job yeniden başlatma:**
```
JobRepository: Job state'ini DB'de saklar (BATCH_JOB_INSTANCE, BATCH_STEP_EXECUTION)
Job başarısız olursa:
  - Tamamlanan step'ler atlanır
  - Başarısız step'ten devam edilir
  - Offset'ten okuma devam eder (ItemReader checkpoint)
```

### Ne zaman?
- Büyük veri toplu işleme (ETL, raporlama, faturalama)
- Dönemsel işler (gece çalışan batch)
- Hata toleransı ve yeniden başlatma kritikse
- Paralel işleme gerekiyorsa

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Hata toleransı (skip, retry) yerleşik | Öğrenme eğrisi dik |
| Yeniden başlatma (checkpoint) | Karmaşık konfigürasyon |
| Paralel/remote partition | Küçük işler için over-engineering |
| Job monitoring (JobRepository) | DB tabloları gerekir (BATCH_JOB_INSTANCE vs.) |

---

## 6. Spring Cache

### Ne?
Method sonuçlarını cache'leyen annotation tabanlı soyutlama katmanı. Altta Redis, Caffeine, EhCache gibi farklı provider'lar çalışabilir.

### Neden?
Aynı parametrelerle sık çağrılan pahalı methodlar (DB sorgusu, dış API, hesaplama) için her seferinde işlem yapmak gereksizdir. Cache ile ilk sonuç saklanır, sonrakiler cache'ten döner.

### Nasıl?
```java
@Configuration
@EnableCaching
class CacheConfig {

    @Bean
    CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withCacheConfiguration("products",              // cache-specific TTL
                defaultConfig.entryTtl(Duration.ofHours(2)))
            .withCacheConfiguration("userSessions",
                defaultConfig.entryTtl(Duration.ofMinutes(5)))
            .build();
    }
}

@Service
class ProductService {

    // İlk çağrıda DB'den oku → Redis'e yaz → sonrakiler Redis'ten döner
    @Cacheable(
        value = "products",
        key = "#id",
        condition = "#id > 0",          // sadece geçerli ID'ler için cache
        unless = "#result == null"       // null sonucu cache'leme
    )
    Product findById(Long id) {
        return productRepo.findById(id).orElse(null);
    }

    // Her zaman DB'ye yaz VE cache'i güncelle (değer değişti)
    @CachePut(value = "products", key = "#product.id")
    Product update(Product product) {
        return productRepo.save(product);
    }

    // Cache'ten sil (veri silindi veya stale)
    @CacheEvict(value = "products", key = "#id")
    void delete(Long id) {
        productRepo.deleteById(id);
    }

    // Tüm cache'i temizle
    @CacheEvict(value = "products", allEntries = true)
    void clearProductCache() { }

    // Birden fazla cache operasyonu
    @Caching(
        put  = { @CachePut("products") },
        evict = { @CacheEvict(value = "categories", key = "#product.categoryId") }
    )
    Product updateWithCategoryInvalidation(Product product) {
        return productRepo.save(product);
    }
}
```

**Cache key stratejisi:**
```java
// Varsayılan key: tüm parametreler
@Cacheable("orders")
List<Order> findByStatus(OrderStatus status, Long customerId)
// key = [status, customerId]

// SpEL ile özel key
@Cacheable(value = "orders", key = "#status.name() + ':' + #customerId")
// key = "ACTIVE:123"

// Custom KeyGenerator
@Cacheable(value = "orders", keyGenerator = "hashKeyGenerator")
```

**Thundering herd problemi:**
```
Cache boşaldı → 1000 thread aynı anda DB'ye → DB çöktü

Çözüm 1: Caffeine + lock (tek thread load eder, diğerleri bekler)
Çözüm 2: Asenkron cache refresh (expire olmadan önce arka planda güncelle)
Çözüm 3: Jitter (TTL'e rastgele offset ekle → hepsi aynı anda expire olmasın)
```

### Ne zaman?
- Sık okunan, nadiren değişen veriler (ürün kataloğu, konfigürasyon, referans data)
- Pahalı hesaplamalar
- Dış API sonuçları

### Ne zaman kullanma?
- Sık değişen veriler (stok miktarı, anlık fiyat)
- Kullanıcıya özel, kişiselleştirilmiş veri (cache key dikkatli tasarlanmazsa)
- ACID gerektiren işlemler

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Dramatik performans artışı | Cache invalidation karmaşık |
| DB yükünü azaltır | Stale veri riski |
| Provider bağımsız (Redis↔Caffeine değiştir) | Bellek tüketimi |
| Annotation ile minimal kod | Dağıtık ortamda tutarsızlık olabilir |

---

## 7. Spring Retry

### Ne?
Geçici hatalar nedeniyle başarısız olan method çağrılarını otomatik tekrar deneyen, geri çekilme (backoff) stratejisi ve fallback mekanizması içeren modül.

### Neden?
Ağ kesintisi, anlık DB yükü, dış servis geçici hatası gibi durumlarda ilk deneme başarısız olabilir ama hemen tekrar denense başarılı olurdu. Bunu her yere try/catch/retry bloğu yazmak yerine `@Retryable` annotation ile deklaratif hale getirirsin.

### Nasıl?
```java
@Configuration
@EnableRetry
class RetryConfig {}

@Service
class PaymentService {

    @Retryable(
        retryFor = {
            HttpServerErrorException.class,      // 5xx hatalar
            ResourceAccessException.class        // network timeout
        },
        noRetryFor = {
            IllegalArgumentException.class,      // bu hataları tekrar deneme
            ValidationException.class
        },
        maxAttempts = 3,
        backoff = @Backoff(
            delay = 1000,        // ilk bekleme: 1sn
            multiplier = 2.0,    // exponential: 1s → 2s → 4s
            maxDelay = 10000,    // maksimum bekleme: 10sn
            random = true        // jitter ekle (thundering herd önlemi)
        )
    )
    PaymentResult charge(PaymentRequest request) {
        return externalGateway.process(request);
    }

    @Recover  // tüm retry başarısız olunca çağrılır
    PaymentResult recover(HttpServerErrorException ex, PaymentRequest request) {
        log.error("Ödeme gateway'i 3 denemede başarısız: {}", request.getOrderId(), ex);
        alertService.notifyOps("Payment gateway down", request);
        return PaymentResult.failed("GATEWAY_UNAVAILABLE");
    }
}

// Programatik kullanım (annotation yerine)
RetryTemplate retryTemplate = RetryTemplate.builder()
    .maxAttempts(3)
    .exponentialBackoff(1000, 2, 10000)
    .retryOn(HttpServerErrorException.class)
    .build();

PaymentResult result = retryTemplate.execute(
    ctx -> externalGateway.process(request),
    ctx -> PaymentResult.failed("GATEWAY_UNAVAILABLE")  // recovery callback
);
```

**Backoff stratejileri:**
```
Fixed:       1s → 1s → 1s      (sabit bekleme)
Exponential: 1s → 2s → 4s → 8s (katlanarak artan)
Random:      0.8s → 2.3s → 3.7s (jitter — dağıtık sistemde çakışmayı önler)
```

### Ne zaman?
- Geçici ağ hataları (dış API, DB bağlantısı)
- Rate limit aşımı (429 → bekle → tekrar dene)
- Optimistic locking çakışmaları

### Ne zaman kullanma?
- İdempotent olmayan işlemler (ödeme, email) — duplicate riski!
- Kalıcı hatalar (validasyon, yetkilendirme) — retry mantıksız

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Geçici hataları şeffaf yönetir | İdempotency sağlanmazsa duplicate işlem |
| Backoff ile sistemi kurtarır | Yanlış yapılandırma → gereksiz gecikme |
| @Recover ile fallback temiz | Uzun retry → kullanıcı bekler |

---

## 8. Spring Scheduling

### Ne?
Metodları zamanlanmış şekilde veya cron ifadesine göre otomatik çalıştıran modül.

### Neden?
"Her gece 02:00'de expired token'ları sil", "her 5 dakikada bir queue'u tara", "her ayın 1'inde fatura oluştur" gibi ihtiyaçlar için harici cron job veya timer yerine Spring içinde tanım yapmak deployment'ı basitleştirir.

### Nasıl?
```java
@Configuration
@EnableScheduling
@EnableAsync  // @Async için
class SchedulingConfig {

    // Scheduled görevlerin çalışacağı thread pool
    @Bean
    TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("scheduled-");
        return scheduler;
    }
}

@Component
class ScheduledTasks {

    // Bir önceki çalışmanın BİTİŞİNDEN itibaren 60sn sonra tekrar
    @Scheduled(fixedDelay = 60_000)
    void cleanupExpiredSessions() {
        sessionRepo.deleteExpiredBefore(Instant.now());
    }

    // Bir önceki çalışmanın BAŞLANGICINDAN itibaren 60sn sonra tekrar (sabit rate)
    @Scheduled(fixedRate = 60_000)
    void healthCheck() {
        externalServiceMonitor.check();
    }

    // Cron ifadesi
    @Scheduled(cron = "0 0 2 * * *")     // Her gece 02:00
    void generateDailyReport() {
        reportService.generateAndSendDaily();
    }

    @Scheduled(cron = "0 0 9 * * MON-FRI")  // Hafta içi sabah 09:00
    void sendBusinessSummary() {
        emailService.sendDailySummary();
    }

    // Asenkron — başka scheduled task'ı bloklamaz
    @Async
    @Scheduled(fixedRate = 5_000)
    void processQueue() {
        queueProcessor.process();
    }
}
```

**Cron ifadesi anatomisi:**
```
"0  0  2  *  *  MON-FRI"
 │  │  │  │  │  └─ Haftanın günü (0=Pazar, 1=Pazartesi..., MON-FRI)
 │  │  │  │  └──── Ay (1-12 veya JAN-DEC, * = hepsi)
 │  │  │  └─────── Ayın günü (1-31, * = hepsi)
 │  │  └────────── Saat (0-23)
 │  └───────────── Dakika (0-59)
 └──────────────── Saniye (0-59)

Örnekler:
"0 */15 * * * *"       → Her 15 dakikada bir (00:00, 00:15, 00:30...)
"0 0 0 1 * *"          → Her ayın 1'i gece yarısı
"0 0 8-18 * * MON-FRI" → Hafta içi 08:00-18:00 arası her saat başı
"0 0 9 * * ?"          → Her gün 09:00 (? = herhangi gün)
```

**Birden fazla instance sorunu:**
```
Problem: 3 pod çalışıyor → aynı cron her 3 pod'da tetiklenir → 3x çalışır!

Çözüm 1: ShedLock (distributed lock ile tek instance çalışır)
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "dailyReport", lockAtMostFor = "PT30M")
void generateDailyReport() { ... }

Çözüm 2: Kubernetes CronJob (uygulama dışında)
Çözüm 3: Spring Cloud Quartz (cluster mode)
```

### Ne zaman?
- Periyodik temizlik (expired records, temp files)
- Düzenli raporlama
- Health check, monitoring
- Cache pre-warming

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Deployment basit (ayrı süreç yok) | Birden fazla instance → duplicate çalışma riski |
| Spring context'e tam erişim | Başarısız görev monitoring zor |
| Cron + fixedRate + fixedDelay esnekliği | Karmaşık scheduling → Quartz veya K8s CronJob daha uygun |

---

## 9. Spring AMQP (RabbitMQ)

### Ne?
Spring uygulamalarını RabbitMQ ile entegre eden modül. Mesaj gönderme, alma, exchange/queue tanımlama ve listener yönetimini Spring idiom'larıyla sağlar.

### Neden?
Ham RabbitMQ Java client ile çalışmak connection, channel, serialization, acknowledgement ve exception handling için çok tekrarlı kod gerektirir. Spring AMQP bunları `RabbitTemplate` ve `@RabbitListener` ile soyutlar.

### Nasıl?
```java
// Altyapı tanımı
@Configuration
class RabbitConfig {

    @Bean
    DirectExchange ordersExchange() {
        return new DirectExchange("orders.exchange", true, false);
    }

    @Bean
    Queue ordersQueue() {
        return QueueBuilder.durable("orders.queue")
            .withArgument("x-dead-letter-exchange", "orders.dlx")  // başarısız → DLX
            .withArgument("x-message-ttl", 300_000)               // 5dk TTL
            .withArgument("x-max-length", 10_000)                 // max 10K mesaj
            .build();
    }

    @Bean
    Binding ordersBinding() {
        return BindingBuilder
            .bind(ordersQueue())
            .to(ordersExchange())
            .with("order.created");  // routing key
    }

    // Dead Letter Queue — başarısız mesajlar buraya
    @Bean
    DirectExchange ordersDlx() { return new DirectExchange("orders.dlx"); }
    @Bean
    Queue ordersDlq() { return QueueBuilder.durable("orders.dlq").build(); }
    @Bean
    Binding dlqBinding() {
        return BindingBuilder.bind(ordersDlq()).to(ordersDlx()).with("order.created");
    }
}

// Mesaj gönder
@Service
class OrderService {
    @Autowired RabbitTemplate rabbitTemplate;

    void publishOrder(Order order) {
        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "order.created",
            order,
            message -> {
                message.getMessageProperties().setContentType("application/json");
                message.getMessageProperties().setCorrelationId(order.getId().toString());
                message.getMessageProperties().setExpiration("60000"); // 60sn TTL
                return message;
            }
        );
    }

    // Publisher confirms — mesaj broker'a ulaştı mı?
    void publishWithConfirm(Order order) {
        rabbitTemplate.invoke(ops -> {
            ops.convertAndSend("orders.exchange", "order.created", order);
            if (!ops.waitForConfirms(5000)) {
                throw new MessageDeliveryException("Broker confirm gelmedi");
            }
            return null;
        });
    }
}

// Mesaj al
@Component
class OrderConsumer {

    @RabbitListener(
        queues = "orders.queue",
        concurrency = "3-10",         // min 3, max 10 concurrent consumer
        ackMode = "MANUAL"            // manuel acknowledge
    )
    void consume(Order order,
                 Channel channel,
                 @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
        try {
            processOrder(order);
            channel.basicAck(deliveryTag, false);   // başarılı
        } catch (BusinessException ex) {
            channel.basicNack(deliveryTag, false, false); // DLX'e gönder, requeue etme
        } catch (TransientException ex) {
            channel.basicNack(deliveryTag, false, true);  // queue'ya geri koy
        }
    }
}
```

### Ne zaman?
- Task queue (işleri worker'lara dağıt)
- Karmaşık routing (topic/headers exchange)
- Delayed/scheduled mesajlar (TTL + DLX trick)
- RPC pattern (request-reply)
- Kafka'dan daha düşük throughput ama daha esnek routing

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Esnek routing (exchange type'ları) | Kafka'ya göre daha düşük throughput |
| Push model → düşük latency | Mesaj replay yok (consume edilince gider) |
| DLX ile hata yönetimi yerleşik | Yönetim karmaşıklığı (exchange, queue, binding) |
| Priority queue, TTL, delayed message | Cluster kurulumu zor |

---

## 10. Spring Kafka

### Ne?
Apache Kafka ile Spring uygulamalarını entegre eden modül. `KafkaTemplate` ile üretim, `@KafkaListener` ile tüketim ve transaction desteği sağlar.

### Neden?
Ham Kafka client konfigürasyonu ve error handling karmaşıktır. Spring Kafka bunu Spring idiom'larına uygun şekilde sarmalar, `@KafkaListener` ile annotation-driven tüketim, Micrometer entegrasyonu ve transaction desteği sunar.

### Nasıl?
```java
// Producer
@Service
class OrderEventPublisher {
    @Autowired KafkaTemplate<String, OrderEvent> kafkaTemplate;

    void publish(OrderCreatedEvent event) {
        kafkaTemplate.send("orders", event.getOrderId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Gönderim başarısız", ex);
                    return;
                }
                RecordMetadata meta = result.getRecordMetadata();
                log.debug("Partition={} Offset={}", meta.partition(), meta.offset());
            });
    }

    // Transactional: ya hepsi ya hiçbiri
    @Transactional("kafkaTransactionManager")
    void publishBatch(List<OrderEvent> events) {
        events.forEach(e -> kafkaTemplate.send("orders", e.getOrderId().toString(), e));
    }
}

// Consumer
@Component
class InventoryConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service",
        concurrency = "6",            // 6 consumer thread (partition sayısı kadar)
        containerFactory = "kafkaListenerContainerFactory"
    )
    void consume(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack
    ) {
        try {
            inventoryService.reserve(event);
            ack.acknowledge();       // offset commit
        } catch (StockUnavailableException ex) {
            // ack ETME → yeniden denemek için
            // ama dikkat: rebalance olunca başa dönebilir
        }
    }

    // Batch consume — throughput için
    @KafkaListener(topics = "orders", batch = "true", concurrency = "3")
    void consumeBatch(List<OrderCreatedEvent> events, Acknowledgment ack) {
        inventoryService.reserveBatch(events);
        ack.acknowledge();  // tüm batch'i commit
    }

    // Dead Letter Topic — başarısız mesajlar
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 1000, multiplier = 2),
        dltTopicSuffix = ".DLT"
    )
    @KafkaListener(topics = "payments")
    void consumePayment(PaymentEvent event) {
        paymentProcessor.process(event);
        // Başarısız → payments-retry-0 → retry → payments-retry-1 → ... → payments.DLT
    }
}
```

**Konfigürasyon:**
```yaml
spring:
  kafka:
    bootstrap-servers: kafka-1:9092,kafka-2:9092,kafka-3:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all           # tüm ISR'lar yazsın
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
    consumer:
      group-id: inventory-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false  # manuel commit
      properties:
        spring.json.trusted.packages: "com.example.events"
```

### Ne zaman?
- Yüksek throughput (100K+ mesaj/sn)
- Event sourcing, audit log, CDC
- Mesaj replay gerekiyorsa
- Çok sayıda consumer aynı mesajı okumalıysa
- Stream processing (Kafka Streams)

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Çok yüksek throughput | Kurulum/yönetim karmaşık |
| Mesaj saklama + replay | Düşük latency senaryolarında RabbitMQ daha iyi |
| Partition ile paralel işlem | Kesin sıralama sadece partition içinde |
| Büyük ekosistem (Kafka Streams, Connect) | Consumer offset yönetimi dikkat ister |

---

## 11. Spring Session

### Ne?
HTTP session'ı sunucu belleği yerine dışarıda (Redis, JDBC, MongoDB) saklayan ve birden fazla uygulama instance'ı arasında session paylaşımını sağlayan modül.

### Neden?
Birden fazla sunucu instance'ı çalışırken kullanıcı session'ı sadece bir instance'ın belleğinde tutulursa sticky session gerekir. Instance çöküşe geçerse session kaybolur. Spring Session bunu Redis gibi merkezi bir store'a taşır — her instance aynı session'ı görür.

### Nasıl?
```yaml
spring:
  session:
    store-type: redis        # redis, jdbc, mongodb, hazelcast
    timeout: 30m
  data:
    redis:
      host: redis.internal
      port: 6379
```

```java
@Configuration
@EnableRedisIndexedHttpSession(maxInactiveIntervalInSeconds = 1800)
class SessionConfig {}

// Uygulama kodu değişmez — HttpSession API aynı
@RestController
class AuthController {

    @PostMapping("/login")
    ResponseEntity<?> login(@RequestBody LoginRequest req, HttpSession session) {
        User user = authService.authenticate(req.getUsername(), req.getPassword());
        session.setAttribute("userId", user.getId());
        session.setAttribute("roles", user.getRoles());
        return ResponseEntity.ok(new LoginResponse(user));
    }

    @GetMapping("/profile")
    ResponseEntity<?> profile(HttpSession session) {
        Long userId = (Long) session.getAttribute("userId");
        if (userId == null) return ResponseEntity.status(401).build();
        return ResponseEntity.ok(userService.findById(userId));
    }
}

// Redis'te session:
// KEY: spring:session:sessions:{sessionId}
// TYPE: Hash
// FIELDS: sessionAttr:userId, sessionAttr:roles, lastAccessedTime, maxInactiveInterval
```

**Spring Security ile:**
```java
http.sessionManagement(sm -> sm
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
    .maximumSessions(1)               // eş zamanlı tek oturum
    .maxSessionsPreventsLogin(false)  // false: yeni login → eski session sonlanır
                                      // true: ikinci login engellenir
);
```

### Ne zaman?
- Birden fazla instance (yatay ölçekleme) + session tabanlı auth
- Instance yeniden başlatmalar arasında session koruma
- Zero-downtime deployment (session kaybolmasın)

### Ne zaman kullanma?
- JWT (stateless) kullanıyorsan — Spring Session gereksiz
- Tek instance uygulamalar

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Stateless instance → yatay ölçekleme kolay | Redis bağımlılığı (SPOF riski) |
| HttpSession API değişmez | Her request'te Redis round-trip |
| Sticky session gerekmez | Session data büyürse Redis bellek tüketimi |
| Instance yeniden başlatmada session korunur | Distributed session invalidation kompleks |
