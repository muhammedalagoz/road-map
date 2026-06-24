# 02c — Spring Boot: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Auto-Configuration

### Gerçek Hayat Sorunları

---

**Sorun 1: İstenmeyen bean override — SessionFactory çakışması**

```
Senaryo:
  Hibernate SessionFactory bean'i manuel tanımlandı.
  Ayrıca spring-boot-starter-data-jpa pom'a eklendi.
  Spring Boot: "SessionFactory bean'i zaten var" → sessizce kendi bean'ini OLUŞTURDU.
  İki farklı SessionFactory → transaction manager karışıklığı → intermittent veri tutarsızlığı.

Neden?
  @ConditionalOnMissingBean: bean henüz yoksa oluştur.
  Manuel bean daha önce kayıtlıysa auto-config devreye girmez.
  Ama kayıt sırası değişirse → auto-config önce çalışır → manual sonra override eder.
  Bean sırası belirsizse ikisi de oluşabilir.

Debug:
  java -jar app.jar --debug → CONDITIONS EVALUATION REPORT
  "SessionFactory matched" veya "did not match" satırını bul.

Düzeltme:
  Manuel bean'e @Primary ekle → Spring hangisini kullanacağını bilir.
  veya auto-config'i exclude et:
  @SpringBootApplication(exclude = HibernateJpaAutoConfiguration.class)
```

---

**Sorun 2: Property öncelik sırası karışıklığı — production'da dev ayarı**

```
Senaryo:
  Geliştiriciler application-dev.properties'e logging.level.root=DEBUG ekledi.
  Kubernetes deployment'ta SPRING_PROFILES_ACTIVE env variable set edilmedi.
  Uygulama "default" profil ile ayağa kalktı → DEBUG log production'da aktif.
  Her request için binlerce satır log → Elasticsearch doldu → alert'ler bastırıldı.

Öncelik sırası hatırlatma:
  1. Komut satırı argümanları (en yüksek)
  2. OS env vars (SPRING_DATASOURCE_URL)
  3. application-{profile}.yml
  4. application.yml
  5. Default değerler (en düşük)

Düzeltme:
  application.properties: güvenli production default'lar koy.
    logging.level.root=WARN
    management.endpoints.web.exposure.include=health,info
  application-dev.properties: gevşek geliştirici ayarları.
  Kubernetes: SPRING_PROFILES_ACTIVE=production her zaman set et.
  Startup'ta kontrol: @PostConstruct'ta aktif profili logla.
```

---

### Mülakat Soruları — Auto-Configuration

**Junior / Mid:**

1. `@ConditionalOnMissingBean` ve `@ConditionalOnClass` farkı nedir?

   > **Beklenen:** `@ConditionalOnClass(Foo.class)`: classpath'te Foo.class dosyası varsa aktif (dependency yüklü mü kontrolü). `@ConditionalOnMissingBean(Foo.class)`: Spring context'te Foo bean'i yoksa aktif (kullanıcı kendi bean'ini tanımlamadıysa). Auto-config'in neden override edilebildiğinin mekanizması: `@ConditionalOnMissingBean` sayesinde kendi `@Bean`'ini tanımlarsan auto-config onu oluşturmaktan vazgeçer.

2. Spring Boot'un `application.properties` property öncelik sırasını açıkla.

   > **Beklenen:** Komut satırı argümanları → OS env vars → `application-{profile}.yml` → `application.yml` → `@PropertySource` → default. Pratikte: env var OS'tan gelir, deployment aracı (K8s) set eder, uygulama kodunu değiştirmeden override mümkün olur. Güvenlik: Production secret'ları env var olarak ver, `application.properties`'e koyma.

3. Auto-configuration debug nasıl yapılır?

   > **Beklenen:** `--debug` flag veya `logging.level.org.springframework.boot.autoconfigure=DEBUG`. Çıktı: CONDITIONS EVALUATION REPORT → "Positive matches" (devreye girenler) ve "Negative matches" (neden girmediği). Örnek: "DataSourceAutoConfiguration did not match → @ConditionalOnClass 'javax.sql.DataSource' not found" → JDBC dependency eksik.

---

**Senior / Architect:**

4. Kendi custom Spring Boot Starter'ını nasıl yazarsın?

   > **Beklenen:** (1) `autoconfigure` modülü: `@AutoConfiguration` sınıfı + `@Conditional` koşullar. (2) `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` dosyasına sınıf adını ekle. (3) `starter` modülü: sadece pom.xml, autoconfigure + kullanıcı bağımlılıklarını transitively çeker. (4) Test: `ApplicationContextRunner` ile farklı koşulları test et. Örnek kullanım: şirket içi monitoring, multi-tenant, audit starter.

5. `@SpringBootApplication`'ın içinde ne var? `@ComponentScan` ne zaman sorun yaratır?

   > **Beklenen:** `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`. `@ComponentScan` varsayılan olarak main sınıfın paketi ve alt paketleri tarar. Sorun: Ana paket çok geniş seçilirse (com.company) → yüzlerce sınıf taranır → startup yavaşlar, istenmeyen bean'ler yüklenir. Düzeltme: `scanBasePackages` ile daralt. Büyük projelerde modüler yapı + explicit bean tanımı tercih edilir.

---

## Bölüm 2: Actuator

### Gerçek Hayat Sorunları

---

**Sorun 3: `/actuator/env` production'da açık — credential sızıntısı**

```
Senaryo:
  Geliştirici kolaylık için tüm Actuator endpoint'lerini açık bıraktı:
  management.endpoints.web.exposure.include=*
  
  Saldırgan /actuator/env'e istek attı:
  → DB şifresi, API key'leri (Spring Boot bazılarını maskeler ama hepsi değil),
    internal hostname'ler görünüyor.

  /actuator/heapdump: JVM heap'ini indir → memory'de plaintext şifreler!
  /actuator/shutdown (POST): uygulamayı durdur!

Düzeltme:
  management.endpoints.web.exposure.include=health,info,metrics,prometheus
  management.server.port=8081  → ayrı port, sadece internal network
  Spring Security ile /actuator/** → IP whitelist veya basic auth.
  
  Güvenli minimum set:
    health: Kubernetes probe için
    info: build bilgisi
    prometheus: Prometheus scrape
```

---

**Sorun 4: Kubernetes liveness probe yanlış yapılandırma — pod sonsuz restart döngüsü**

```
Senaryo:
  livenessProbe: /actuator/health → tüm health bileşenlerini kontrol ediyor.
  DB geçici olarak yavaşladı → /actuator/health → DOWN.
  Kubernetes: "Uygulama ölü" → pod restart.
  Pod restart → DB bağlantısı yeniden kuruluyor → yeniden DOWN → tekrar restart.
  Tüm pod'lar aynı anda restart → sıfır available pod → site tamamen down.

Sorun:
  liveness: "JVM hayatta mı?" sorusunu sormalı.
  readiness: "Trafik alabilir mi?" (DB, Redis bağlantısı) sorusunu sormalı.

Düzeltme:
  livenessProbe: /actuator/health/liveness
    → sadece JVM'in canlı olup olmadığı (custom liveness state)
    → DB DOWN olsa bile pod restart olmaz
  
  readinessProbe: /actuator/health/readiness
    → DB, Redis, Kafka bağlantıları
    → DOWN: pod trafik almaz ama restart olmaz

Spring Boot 2.3+:
  management.health.livenessState.enabled=true
  management.health.readinessState.enabled=true
```

---

**Sorun 5: Custom metric boyutu patlaması (high cardinality)**

```java
// YANLIŞ — userId tag olarak kullanmak: milyonlarca farklı değer
@GetMapping("/products/{id}")
Product getProduct(@PathVariable Long id, @AuthenticationPrincipal User user) {
    Timer timer = Timer.builder("api.request")
        .tag("userId", user.getId().toString())  // YANLIŞ! Her kullanıcı ayrı time series
        .register(registry);
    // 1M kullanıcı → 1M farklı time series → Prometheus OOM
}

// DOĞRU — düşük cardinality tag'ler kullan
Timer timer = Timer.builder("api.request")
    .tag("endpoint", "/products/{id}")  // sabit değer
    .tag("method", "GET")
    .tag("region", "eu-west-1")
    .register(registry);
// Sonuç: sadece birkaç time series

// Kural: tag değerleri bounded (sınırlı, az sayıda farklı değer) olmalı.
// userId, orderId, sessionId → asla tag. Bunları log'a yaz.
```

---

### Mülakat Soruları — Actuator

**Junior / Mid:**

6. Liveness ve Readiness probe farkı nedir? Spring Boot'ta nasıl yapılandırılır?

   > **Beklenen:** Liveness: "Uygulama canlı mı, restart gerekiyor mu?" — JVM kilitlendiyse, deadlock'a düştüyse restart. Readiness: "Trafik alabilir mi?" — DB bağlantısı yoksa trafik gönderme ama restart etme. Spring Boot: `/actuator/health/liveness` ve `/actuator/health/readiness`. Kubernetes: `livenessProbe` → liveness, `readinessProbe` → readiness. Kritik: DB DOWN → sadece readiness DOWN olmalı, liveness DOWN olmamalı.

7. Custom `HealthIndicator` neden yazılır? Örnek ver.

   > **Beklenen:** Varsayılan Actuator DB, Redis, Kafka'yı check eder. Ama 3rd party servis (ödeme gateway, SMS provider) check edilmez. Custom HealthIndicator: `implements HealthIndicator` → `health()` metodu → `Health.up()` / `Health.down()`. Kubernetes: bu servis DOWN olunca pod trafik almaz. Dikkat: health check'in kendisi hafif olmalı — her 10 saniyede çağrılıyor, ağır sorgu çalıştırma.

8. Runtime'da log level nasıl değiştirilir?

   > **Beklenen:** `POST /actuator/loggers/com.example.service` → body: `{"configuredLevel": "DEBUG"}`. Uygulamayı restart etmeden o paketin log seviyesini değiştir. Production'da bir sorun araştırırken çok kullanışlı. Kalıcı değil: restart sonrası application.properties değerine döner. Spring Boot Admin ile UI üzerinden de yapılabilir.

---

**Senior / Architect:**

9. Prometheus scraping ile push-based monitoring farkı nedir? Hangisini ne zaman seçersin?

   > **Beklenen:** Pull (Prometheus): Prometheus scraper uygulamaya gelir, `/actuator/prometheus` çeker. Avantaj: merkezi kontrol, uygulamanın Prometheus'u bilmesi gerekmez. Dezavantaj: kısa ömürlü job'lar (batch) kaçırılabilir — PushGateway çözüm. Push: uygulama metric'i gönderir (StatsD, InfluxDB). Avantaj: batch job, Lambda için iyi. Dezavantaj: her uygulama hedef adresi bilmeli. Karar: uzun ömürlü servis → Prometheus pull. Batch job, Lambda → push/PushGateway.

---

## Bölüm 3: Test

### Gerçek Hayat Sorunları

---

**Sorun 6: `@MockBean` context cache'i kırıyor — CI yavaş**

```
Senaryo:
  Test suite'inde 50 @SpringBootTest sınıfı var.
  Her sınıfta farklı @MockBean kombinasyonları.
  Spring context'i bir kez ayağa kaldırıp paylaşması gerekirken
  her farklı @MockBean seti için context yeniden başlatıyor.
  CI: 50 context startup → her biri 15sn → 12.5 dakika sadece startup!

Neden?
  @MockBean context definition'ını değiştirir → cache key farklı → yeni context.

Çözümler:
  1. @MockBean yerine @SpyBean minimuma indir.
  2. Test slicing: @WebMvcTest (zaten mock'lu), @DataJpaTest kullan.
  3. Ortak @MockBean seti → abstract base test class → context paylaşılır.
  4. Testcontainers + @DynamicPropertySource: gerçek bağımlılık → mock gerekmez → tek context.
  5. application-test.properties ile mock bean'leri property bazlı disable et.
```

---

**Sorun 7: H2 ile test geçiyor, production PostgreSQL'de bozuluyor**

```
Senaryo:
  @DataJpaTest — H2 in-memory DB ile çalışıyor.
  Custom SQL sorgusu: PostgreSQL'e özgü JSONB operatörü veya window function.
  H2 bu syntax'ı desteklemiyor → test "hata" fırlatıyor.
  Geliştirici syntax'ı değiştiriyor, test geçiyor.
  Production PostgreSQL'de doğru syntax yok → feature bozuk.

Çözüm: Testcontainers
  @DataJpaTest
  @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
  @Testcontainers
  class OrderRepositoryTest {
      @Container
      static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15-alpine");
      
      @DynamicPropertySource
      static void props(DynamicPropertyRegistry r) {
          r.add("spring.datasource.url", pg::getJdbcUrl);
      }
      // Artık gerçek PostgreSQL → JSONB, window function vs. çalışır
  }

  Dezavantaj: Docker gerekli, biraz daha yavaş ama çok daha güvenilir.
```

---

**Sorun 8: Flakey test — zaman bağımlı intermittent başarısızlık**

```java
// Kötü test — sistem saatine bağımlı
@Test
void tokenShouldExpireAfterOneHour() {
    String token = tokenService.generate(user);
    Thread.sleep(3_601_000); // 1 saat 1 sn bekle (!)
    assertThat(tokenService.isExpired(token)).isTrue();
    // CI'da 3601 saniye beklemek kabul edilemez.
}

// Doğru: Clock'ı inject edilebilir yap
@Service
class TokenService {
    private final Clock clock;
    
    TokenService(Clock clock) { this.clock = clock; }  // inject edilebilir
    
    boolean isExpired(String token) {
        return Instant.now(clock).isAfter(extractExpiry(token));
    }
}

// Test: saati manuel kontrol et
@Test
void tokenShouldExpireAfterOneHour() {
    Clock fixedClock = Clock.fixed(Instant.now(), ZoneOffset.UTC);
    TokenService svc = new TokenService(fixedClock);
    String token = svc.generate(user, Duration.ofHours(1));

    Clock future = Clock.offset(fixedClock, Duration.ofHours(1).plusSeconds(1));
    TokenService futureSvc = new TokenService(future);
    assertThat(futureSvc.isExpired(token)).isTrue();
}
```

---

### Mülakat Soruları — Test

**Junior / Mid:**

10. `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest` ne zaman kullanılır?

    > **Beklenen:** @SpringBootTest: tüm uygulama entegrasyonu — servisler arası akış, E2E. Yavaş (10-30s). @WebMvcTest: sadece controller katmanı — validation, serialization, HTTP status. Hızlı (1-3s), Service otomatik mock edilmeli. @DataJpaTest: sadece JPA — custom query doğruluğu, H2 ile. Hızlı. Kural: mümkün olan en küçük slice → hızlı feedback. @SpringBootTest → gerçekten gerekiyorsa.

11. `@MockBean` ve `@Mock` (Mockito) farkı nedir?

    > **Beklenen:** `@Mock`: Mockito annotation, Spring yoktur, unit test. `@MockBean`: Spring context'e mock bean'i ekler, @Autowired ile inject edilir. @WebMvcTest'te Service'i mock'lamak için `@MockBean` kullanılır. Dikkat: @MockBean her kullanımda yeni context yükler → CI yavaşlar. Mümkünse unit test (Spring olmadan) tercih et.

12. Testcontainers neden H2'ye göre daha güvenilir?

    > **Beklenen:** H2 dialect farkları: JSONB, array operators, window functions, upsert syntax PostgreSQL/MySQL'de farklı → H2'de çalışmaz veya farklı davranır. Testcontainers: gerçek Docker image → production ortamıyla aynı davranış. Index, constraint, trigger davranışı da aynı. Dezavantaj: Docker gerekli, biraz yavaş. Avantaj: "Works on my machine" problemi ortadan kalkar.

---

**Senior / Architect:**

13. Test piramidini açıkla. Mevcut projede denge nasıl olmalı?

    > **Beklenen:** Unit (çok, hızlı, ucuz) → Integration/Slice (orta) → E2E (az, yavaş, pahalı). Spring Boot projesi için: Unit test (Mockito, Spring yok): iş mantığı, utility. Slice test (@WebMvcTest, @DataJpaTest): katman doğruluğu. Testcontainers: kritik entegrasyon (Kafka, DB). @SpringBootTest: az sayıda happy path E2E. Sorun: çoğu proje piramidi tersine çeviriyor (çok @SpringBootTest → CI saatler sürer).

14. Contract testing ne zaman gereklidir? Testcontainers ile farkı?

    > **Beklenen:** Testcontainers: bağımlılığın davranışını test et (DB, broker). Contract testing (Pact): servisler arası API kontratını test et — consumer expectations vs provider responses. Microservices'te: A servisi B'den veri bekliyor → B değişince A bozulabilir. Contract test: consumer test case yazar → provider bu test case'e karşı doğrulama yapar → "can I deploy?" kararı. Testcontainers bunu çözmez — farklı katman.

---

## Bölüm 4: Spring Batch

### Gerçek Hayat Sorunları

---

**Sorun 9: Chunk boyutu yanlış — OutOfMemory veya yavaş işlem**

```
Senaryo A — Chunk çok büyük:
  chunk(10_000): 10K kayıt belleğe alındı → işlem → tek transaction ile yaz.
  10M satır dosyası → her chunk 10K → 1000 transaction.
  Ama 10K satır aynı anda belleğe → her satır 5KB → 50MB per chunk → OOM riski.

Senaryo B — Chunk çok küçük:
  chunk(1): her satır ayrı transaction → 10M transaction overhead → 10 saat.

Doğru boyut bulma:
  Deneysel: 100, 500, 1000, 5000 ile load test.
  Formül yok — satır boyutu, işlem karmaşıklığı, DB insert throughput'a göre.
  Genellikle 500-2000 arası iyi başlangıç.
  JVM heap gözetilmeli: chunk × average_row_size << available_heap.

Paralel chunk:
  Partition kullan: 10M satır → 10 partition (her biri 1M) → 10 thread → paralel.
  Her thread kendi chunk'larını işler.
```

---

**Sorun 10: Job yeniden başlatma — aynı satırları tekrar işleme**

```
Senaryo:
  Gece batch: 5M satır CSV → DB'ye import.
  3M satır başarıyla işlendi, 4M'inci chunk'ta hata → job FAILED durumuna düştü.
  
  Sabah job tekrar çalıştırıldı:
  BAD: Baştan başladı → 3M satır TEKRAR işlendi → duplicate veri!

Neden?
  Job, aynı JobParameters ile çalıştırılırsa JobRepository önceki run'ı bulur.
  Ama restart değil, yeni run başlatıldıysa → baştan başlar.

Düzeltme:
  1. JobRepository kullan: Spring Batch otomatik yönetir.
     Aynı JobParameters + başarısız status → restart → kaldığı yerden devam.
     
  2. Idempotent write: her satır için upsert (INSERT ... ON CONFLICT UPDATE).
     Tekrar işlenirse → üzerine yazar, duplicate olmaz.
     
  3. Checkpoint: FlatFileItemReader lastReadPosition kaydeder.
     Restart: checkpoint'ten devam eder.
  
  4. JobParametersIncrementer: her run için unique parameter.
     Restart istiyorsan: önceki failed run ID ile launch et.
```

---

### Mülakat Soruları — Spring Batch

**Junior / Mid:**

15. Spring Batch'in temel bileşenleri nelerdir? Chunk-oriented processing nasıl çalışır?

    > **Beklenen:** Job → Step → (ItemReader, ItemProcessor, ItemWriter). Chunk: N kayıt okuma → işle → transaction içinde toplu yaz → commit. Başarısız: chunk rollback (sadece o chunk, tamamlanan chunk'lar korunur). Job → birden fazla Step → Step tamamlanmadan sonrakine geçilmez. JobRepository: her run'ın durumunu DB'de saklar.

16. `skip` ve `retry` ne zaman kullanılır? Farkı nedir?

    > **Beklenen:** `retry`: geçici hata (deadlock, network timeout) — aynı kaydı tekrar dene, maxAttempts kez. `skip`: kalıcı hata (validasyon hatası, hatalı veri) — bu kaydı atla, devam et, skipLimit'e kadar. Birlikte: önce retry, retry başarısız → skip. Dikkat: skip edilen kayıtları log'la veya bad records tablosuna yaz — işlenmemiş kayıtlar kaybolmasın.

---

**Senior / Architect:**

17. Spring Batch ile Quartz Scheduler farkı nedir? Hangisini ne zaman seçersin?

    > **Beklenen:** Spring Batch: toplu veri işleme framework'ü (okuma→işleme→yazma, chunk, retry, skip, restart). Quartz: zamanlama motoru (cron, trigger, job scheduling). Genellikle birlikte kullanılır: Quartz belirli saatte Spring Batch job'ı tetikler. Sadece schedule gerekiyorsa: @Scheduled yeterli. Büyük veri + hata toleransı + restart gerekiyorsa: Spring Batch. Kubernetes'te: K8s CronJob + Spring Batch kombinasyonu yaygın.

---

## Bölüm 5: Spring Cache

### Gerçek Hayat Sorunları

---

**Sorun 11: Cache stampede (Thundering Herd)**

```
Senaryo:
  "products" cache — 500 concurrent kullanıcı ürün listesini istiyor.
  Redis TTL doldu → cache boşaldı → 500 istek aynı anda DB'ye gitti.
  DB: 500 aynı sorgu → connection pool tüketildi → 503 hatası yağdı.

Çözüm 1: Probabilistic Early Expiration (PER)
  TTL dolmadan önce arka planda yenile.
  @Scheduled(fixedDelay=...) → cache'i expire olmadan refresh et.

Çözüm 2: TTL Jitter
  Her kaydın TTL'ine rastgele offset ekle:
  Duration ttl = Duration.ofMinutes(30).plus(
      Duration.ofSeconds(ThreadLocalRandom.current().nextInt(0, 300)));
  → Hepsi aynı anda expire olmuyor.

Çözüm 3: Caffeine + LoadingCache
  Caffeine: key yüklenirken single loader garantisi.
  İlk istek DB'ye gider, diğerleri bekler (aynı key için tek thread load eder).
  CaffeineSpec: expireAfterWrite=30m,maximumSize=10000

Çözüm 4: Soft TTL
  Cache değeri expire olunca direkt sil değil, "stale" işaretle.
  İlk istek stale veriyi döndürsün + arka planda yenilesin.
```

---

**Sorun 12: Cache inconsistency — stale veri**

```java
// Senaryo: fiyat güncellendi ama cache hâlâ eski fiyatı gösteriyor
@CachePut(value = "products", key = "#product.id")
Product updatePrice(Product product) {
    return productRepo.save(product);  // DB güncellendi + "products" cache güncellendi
}

// Ama başka bir method da aynı ürünü cache'liyorsa:
@Cacheable(value = "productList")   // farklı cache name!
List<Product> findAll() { return productRepo.findAll(); }

// updatePrice çağrıldı: "products::123" güncellendi.
// Ama "productList" cache'i hâlâ eski listeyi döndürüyor!

// Düzeltme: @Caching ile birden fazla evict
@Caching(
    put   = { @CachePut(value = "products", key = "#product.id") },
    evict = { @CacheEvict(value = "productList", allEntries = true) }
)
Product updatePrice(Product product) {
    return productRepo.save(product);
}

// Daha iyi yaklaşım: Event-based invalidation
// ProductUpdatedEvent → @EventListener → cache temizle
// → Cache invalidation mantığı tek yerde
```

---

### Mülakat Soruları — Spring Cache

**Junior / Mid:**

18. `@Cacheable`, `@CachePut`, `@CacheEvict` farkları nelerdir?

    > **Beklenen:** @Cacheable: cache varsa döndür, yoksa metodu çalıştır ve cache'e yaz. Okuma optimizasyonu. @CachePut: her zaman metodu çalıştır VE sonucu cache'e yaz. Yazma sonrası cache güncelleme. @CacheEvict: cache'ten sil. Veri silinince veya değişince. `allEntries=true`: tüm cache'i temizle. Üçünü birlikte kullanmak: @Caching ile birleştir.

19. Cache key nasıl tasarlanır? Kötü key örneği ver.

    > **Beklenen:** İyi key: unique, belirleyici, boyutu küçük. SpEL: `#id`, `#user.id`, `#status.name() + ':' + #page`. Kötü: `#request` (tüm request nesnesi serialize edilir — büyük, yavaş). Kötü: key yok + birden fazla parametre → tüm parametreler birleştirilir → bazen collision. userId tabanlı key: `"user:" + #userId + ":orders"` → farklı kullanıcıların cache'i ayrışır. Cache poisoning: kullanıcı A'nın verisi kullanıcı B'ye dönmesin.

---

**Senior / Architect:**

20. Distributed cache (Redis) ile local cache (Caffeine) arasında nasıl seçim yaparsın?

    > **Beklenen:** Caffeine: in-process, ultra-hızlı (<1ms), instance başına ayrı — multi-instance'ta stale veri riski yüksek. Redis: ağ round-trip (~1-5ms), tüm instance'lar aynı veriyi görür — tutarlı. Hybrid (L1+L2): Caffeine (L1, 30sn TTL) + Redis (L2, 30dk TTL). Cache invalidation: Redis pub/sub → her instance'a "bu key'i invalidate et" mesajı → L1 temizlenir. Senaryo: tek instance veya cache tutarlılığı gerekmiyorsa Caffeine. Multi-instance, tutarlılık önemliyse Redis.

---

## Bölüm 6: Spring Retry

### Gerçek Hayat Sorunları

---

**Sorun 13: İdempotent olmayan işlemde retry — duplicate ödeme**

```
Senaryo:
  Ödeme gateway'e HTTP POST yapılıyor.
  Ağ kesintisi: istek gönderildi ama yanıt gelmedi.
  @Retryable(retryFor=ResourceAccessException.class) → otomatik 3 kez tekrar denedi.
  Sonuç: 3 ödeme gerçekleşti!

Neden tehlikeli?
  Ödeme POST idempotent değil → her istek yeni ödeme oluşturuyor.

Çözüm:
  1. Idempotency key: her istek için UUID üret → header'da gönder.
     Gateway aynı key'i görürse → zaten işlendi → aynı sonucu döndür.
     headers.set("Idempotency-Key", UUID.randomUUID().toString())
     
  2. İstek öncesi unique transaction ID oluştur → DB'ye kaydet → retry'da aynı ID.
     Gateway duplicate'i reddedebilir.
  
  3. @Retryable'ı ödeme için kullanma, sadece idempotent operasyonlarda kullan.
     GET, basit sorgular için güvenli.
```

---

**Sorun 14: Yanlış exception retry — validation hatası sonsuz döngü**

```java
// YANLIŞ — tüm exception'ları retry ediyor
@Retryable(maxAttempts = 3)
void processOrder(Order order) {
    if (order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Geçersiz tutar");  // 3 kez retry → hep hata!
    }
    // ...
}

// DOĞRU — sadece geçici hataları retry et
@Retryable(
    retryFor = {HttpServerErrorException.class, ResourceAccessException.class},
    noRetryFor = {IllegalArgumentException.class, ValidationException.class}
)
void processOrder(Order order) { ... }

// Kalıcı hata (validation, auth, business rule) → retry anlamsız, hızlı fail.
// Geçici hata (network, rate limit, 503) → retry mantıklı.
```

---

### Mülakat Soruları — Spring Retry

**Junior / Mid:**

21. Exponential backoff nedir? Jitter neden eklenir?

    > **Beklenen:** Exponential backoff: her retry denemesinde bekleme süresi katlanarak artar (1s → 2s → 4s → 8s). Sistem recover olmasına zaman tanır. Jitter (rastgele offset): Tüm client'lar aynı anda retry ederse → aynı anda yük → sistem yeniden çöker. Jitter: her client farklı zamanda retry → yük yayılır → thundering herd önlenir. `random=true` parametresi bunu sağlar.

22. `@Recover` ne zaman çağrılır? Signature nasıl olmalıdır?

    > **Beklenen:** Tüm retry denemeleri başarısız olunca çağrılır. Signature: dönüş tipi orijinal metotla aynı, ilk parametre yakalanan exception tipi, sonraki parametreler orijinal metotla aynı. Spring Retry aynı exception hiyerarşisinde en spesifik @Recover metodunu seçer. @Recover yoksa son exception fırlatılır. Fallback stratejisi: dead letter queue, alert gönder, degrade yanıt dön.

---

## Bölüm 7: Spring Scheduling

### Gerçek Hayat Sorunları

---

**Sorun 15: Çoklu instance — cron 3 kez çalışıyor**

```
Senaryo:
  3 pod çalışıyor, "Her gece 02:00'de fatura oluştur" cron var.
  3 pod'da da cron tetiklendi → aynı fatura 3 kez oluşturuldu.
  Müşteriler 3 fatura aldı, muhasebe karıştı.

Çözüm 1: ShedLock — distributed lock
  @SchedulerLock(name = "generateInvoices", lockAtMostFor = "PT30M")
  @Scheduled(cron = "0 0 2 * * *")
  void generateInvoices() { ... }
  
  ShedLock: Redis veya DB'de lock tutup sadece bir instance çalıştırır.
  lockAtMostFor: crash durumunda lock sonsuz kalmaz (30dk sonra otomatik release).

Çözüm 2: Kubernetes CronJob
  Uygulamadan tamamen çıkar → K8s CronJob ayrı pod başlatır.
  Tek pod → duplicate risk sıfır.
  Dezavantaj: Spring context yeniden yükleme süresi.

Çözüm 3: Leader election
  Sadece lider pod cron çalıştırır.
  Spring Cloud Kubernetes Leader Election veya Zookeeper.
```

---

**Sorun 16: `fixedDelay` ve `fixedRate` karışıklığı**

```java
// fixedDelay: önceki bitiş → 60sn sonra başlar
@Scheduled(fixedDelay = 60_000)
void slowTask() {
    Thread.sleep(30_000);  // 30sn süren task
    // Toplam: 30sn çalışma + 60sn bekleme = her 90 saniyede bir
}

// fixedRate: önceki başlangıç → 60sn sonra başlar
@Scheduled(fixedRate = 60_000)
void rateTask() {
    Thread.sleep(30_000);  // 30sn süren task
    // 0:00 başladı → 0:30 bitti → 1:00 tekrar başlar (30sn gecikmeli)
    // UYARI: task 60sn'den uzun sürerse → birikme riski!
    // 0:00 başladı → 1:30 bitti → anında tekrar başlar (backlog var)
}

// Senaryo: fixedRate + task normalden uzun sürerse
// → Görevler birbirine giriyor (önceki bitmeden yenisi başlıyor)
// Çözüm: @Async + fixedRate → paralel çalışabilir
// Veya: fixedDelay kullan → her zaman öncekinin bitmesini bekler
```

---

### Mülakat Soruları — Scheduling

**Junior / Mid:**

23. `@Scheduled(fixedDelay)` ile `@Scheduled(fixedRate)` farkı nedir?

    > **Beklenen:** fixedDelay: önceki çalışmanın BİTİŞinden itibaren N ms sonra başlar. Görev ne kadar sürer → bekleme hep sabit. fixedRate: önceki çalışmanın BAŞLANGICINDEN itibaren N ms sonra başlar. Görev uzun sürerse → birikim olabilir (scheduler backlog). Ne zaman hangisi: Görevin tamamlanmasını garantilemek istiyorsan → fixedDelay. Sabit aralıkla çalışması kritikse (metric toplama) → fixedRate + @Async.

24. Çoklu Kubernetes pod'larında cron tekrar çalışmasını nasıl engellersin?

    > **Beklenen:** ShedLock: DB veya Redis'te distributed lock. `@SchedulerLock(name, lockAtMostFor)`. Kubernetes CronJob: uygulamadan kaldır, ayrı pod başlat. Leader election: Spring Integration veya Zookeeper ile lider seç, sadece lider çalıştırır. Quartz Cluster Mode: cluster-aware scheduling, yalnızca bir node tetikler. Tercih sırası: basit ise ShedLock, karmaşık workflow ise K8s CronJob.

---

## Bölüm 8: Spring AMQP (RabbitMQ)

### Gerçek Hayat Sorunları

---

**Sorun 17: Message requeue döngüsü — poison message**

```
Senaryo:
  Bozuk mesaj queue'ya geldi (parse edilemeyen JSON).
  Consumer: deserializasyon exception → basicNack + requeue=true.
  Mesaj hemen tekrar queue'ya döndü → tekrar tüketildi → tekrar hata → sonsuz döngü.
  Queue doluyor, CPU yüzde 100, diğer mesajlar işlenemiyor.

Düzeltme:
  1. requeue=false: hata durumunda queue'ya geri koyma → DLX'e düş.
     channel.basicNack(tag, false, false)  // ← requeue=false
  
  2. DLX (Dead Letter Exchange): başarısız mesajlar buraya → DLQ'dan manuel incele.
  
  3. x-death header: RabbitMQ kaç kez tekrar denendi sayar.
     3 deneme → DLQ gönder → alert.
  
  4. @RetryableTopic (Kafka'da) veya Spring AMQP retry interceptor:
     1s → 2s → 4s bekleyerek retry → son deneme DLX.
```

---

**Sorun 18: Publisher confirm olmadan mesaj kaybı**

```
Senaryo:
  Order service: RabbitMQ'ya sipariş event'i gönderdi.
  Uygulama "başarılı" varsaydı → sipariş DB'ye commit edildi.
  RabbitMQ: o anda network partition → mesaj kayboldu.
  Stok servisi event'i hiç almadı → stok güncellenmedi → overselling!

Sorun: fire-and-forget + no confirmation.

Çözümler:
  1. Publisher Confirms: broker acknowledge verince mesaj ulaştı sayılır.
     rabbitTemplate.invoke(ops -> {
         ops.convertAndSend(...);
         ops.waitForConfirms(5000);  // 5sn içinde confirm gelmezse exception
     });
  
  2. Transactional: AMQP transaction (çok yavaş, nadiren kullanılır).
  
  3. Outbox Pattern (en güvenilir):
     DB transaction içinde hem sipariş kaydet hem outbox tablosuna mesaj yaz.
     Ayrı thread outbox'ı okur → RabbitMQ'ya gönderir → gönderilince sil.
     DB transaction: atomik → ya ikisi de olur ya da hiçbiri.
```

---

### Mülakat Soruları — AMQP

**Junior / Mid:**

25. RabbitMQ'da `basicAck`, `basicNack(requeue=true)`, `basicNack(requeue=false)` ne zaman kullanılır?

    > **Beklenen:** basicAck: mesaj başarıyla işlendi, queue'dan sil. basicNack requeue=true: geçici hata (DB kesintisi), tekrar dene — dikkat: sonsuz döngü riski. basicNack requeue=false: kalıcı hata (parse hatası, bozuk mesaj) — DLX'e gönder, kayıp etme. Best practice: kalıcı hata → requeue=false + DLX yapılandır. Geçici hata → retry interceptor (belirli sayıda deneme) + sonunda DLX.

26. Direct, Topic, Fanout exchange farkı nedir?

    > **Beklenen:** Direct: routing key tam eşleşmesi. `order.created` → sadece `order.created` binding olan queue. Topic: routing key wildcard. `order.*` → order.created, order.cancelled. `#.critical` → herhangi bir prefix + critical. Fanout: routing key ignore — tüm bağlı queue'lara broadcast. Kullanım: Direct: tek consumer task queue. Topic: event filtering (her servis ilgilendiği event'i filtreler). Fanout: log broadcasting, cache invalidation.

---

## Bölüm 9: Spring Kafka

### Gerçek Hayat Sorunları

---

**Sorun 19: Consumer lag birikimi — rebalance fırtınası**

```
Senaryo:
  Topic: 12 partition, consumer group: 12 consumer thread.
  Bir consumer yavaşladı (dış servis timeout) → partition lag birikti.
  Kafka: consumer heartbeat timeout → rebalance tetikledi.
  Rebalance sırasında: tüm consumer'lar durdu → yeniden atama → tekrar başladı.
  Yavaş consumer tekrar lag'landı → tekrar rebalance → fırtına!

Sebepler:
  - session.timeout.ms çok küçük (varsayılan 45sn)
  - max.poll.interval.ms: poll() çağrıları arası max süre (varsayılan 5dk)
  - İşleme süresi > max.poll.interval.ms → consumer grubundan kicked

Düzeltme:
  max.poll.records: 500 → 50 (tek poll'da daha az kayıt)
  max.poll.interval.ms: 5dk → 10dk (uzun işleme için)
  İşlemeyi asenkron yap: poll → hızlı al → async işle → manuel ack.
  Concurrency artır: daha fazla consumer thread → partition başına daha az yük.
```

---

**Sorun 20: Deserializasyon hatası — topic bloklanması**

```
Senaryo:
  Producer: v2 formatında mesaj gönderdi (yeni alan eklendi).
  Consumer: v1 format için deserializasyon kodu → hata fırlattı.
  Consumer ack etmedi → aynı mesajı tekrar okudu → tekrar hata → topic bloklandı.
  O partition'daki sonraki tüm mesajlar işlenemiyor.

Sorun: Schema evolution yönetilmedi.

Çözüm 1: Schema Registry (Confluent veya Apicurio)
  Avro/Protobuf schema → registry'de versiyonla.
  Consumer: registry'den schema'yı çek → deserialize.
  Backward/Forward compatibility kontrolü.

Çözüm 2: @RetryableTopic + DLT
  Deserializasyon hatası → retry topic → 3 deneme → DLT.
  DLT: manuel inceleme kuyruğu.
  Diğer mesajlar bloklanmaz.

Çözüm 3: Tolerant reader pattern
  Jackson: FAIL_ON_UNKNOWN_PROPERTIES=false → yeni alan → ignore et.
  Consumer yeni alanı bilmese de hata vermez.
```

---

### Mülakat Soruları — Kafka

**Junior / Mid:**

27. `enable.auto.commit=false` neden önerilir? Manuel commit nasıl yapılır?

    > **Beklenen:** Auto commit: Kafka periyodik olarak offset'i commit eder (varsayılan 5sn). Sorun: mesaj alındı, işleme başlandı, auto commit oldu, sonra işleme hata verdi → offset commit edildi → mesaj kayboldu! Manuel commit: işlem başarılı olduktan SONRA `ack.acknowledge()`. At-least-once garantisi: bazen duplicate olabilir (uygulama çöktüyse), ama kayıp olmaz. Idempotency ile at-least-once + idempotent işlem = exactly-once semantics.

28. `@RetryableTopic` ne yapar? DLT nedir?

    > **Beklenen:** @RetryableTopic: başarısız mesajlar için otomatik retry topic zinciri oluşturur. `orders` → `orders-retry-0` → `orders-retry-1` → `orders.DLT`. Her retry arasında backoff (delay). Avantaj: orijinal topic bloklanmaz, başarısız mesajlar ayrı işlenir. DLT (Dead Letter Topic): tüm retry başarısız → buraya. Manuel inceleme, alert, reprocessing için. Kafka alternatifi: RabbitMQ'daki DLX/DLQ benzeri.

---

**Senior / Architect:**

29. Kafka'da exactly-once semantics nasıl sağlanır?

    > **Beklenen:** Producer tarafı: `enable.idempotence=true` + `acks=all` → at-least-once + duplicate engeli = exactly-once produce. Consumer + produce (Kafka Streams): `isolation.level=read_committed` + transactional producer → consume-transform-produce atomik. DB write + Kafka: Kafka transaction + DB transaction birlikte atomic değil → Outbox Pattern gerekir. "Exactly-once" farklı sistemlere (Kafka + DB) arasında mümkün değil native olarak.

30. Consumer group rebalance'ı nasıl minimize edersin?

    > **Beklenen:** (1) `session.timeout.ms` ve `heartbeat.interval.ms` optimize et. (2) `max.poll.interval.ms` artır veya per-poll işlemi azalt (max.poll.records). (3) Static membership: `group.instance.id` ile consumer kimliği sabit → rebalance yerine incremental cooperative rebalance. (4) Cooperative sticky assignor: `partition.assignment.strategy=CooperativeStickyAssignor` → tüm partition'lar değil, sadece taşınan partition'lar affected. (5) Consumer sayısını partition sayısından fazla tutma.

---

## Bölüm 10: Spring Session

### Gerçek Hayat Sorunları

---

**Sorun 21: Redis down — tüm kullanıcılar oturumu kaybetti**

```
Senaryo:
  Spring Session → Redis.
  Redis node tek nokta çöktü → session store ulaşılamaz.
  Tüm aktif kullanıcılar oturumlarını kaybetti → login sayfasına düştü.
  "Siteniz çöktü!" şikayetleri.

Redis tek nokta hatası çözümleri:
  1. Redis Sentinel: master + replica + sentinel → otomatik failover.
     Failover ~30sn → kullanıcılar kısa süre etkilenir.
  
  2. Redis Cluster: sharding + replica → yüksek availability.
  
  3. Fallback: Redis down → in-memory session (güvenli değil, session paylaşılmaz).
     Graceful degradation: login yeniden istenebilir, ama site çalışmaya devam eder.

Alternatif: JWT (stateless)
  Token client'ta → Redis yok → SPOF yok.
  Dezavantaj: token invalidation zor (blacklist → Redis lazım).
```

---

### Mülakat Soruları — Spring Session

**Junior / Mid:**

31. Spring Session olmadan çoklu instance'ta session sorunu neden yaşanır?

    > **Beklenen:** Varsayılan session JVM belleğinde. Instance A'da login → session Instance A'da. Load balancer bir sonraki isteği Instance B'ye yönlendirdi → session yok → login sayfası. Sticky session (aynı kullanıcı hep aynı instance'a) çözüm ama instance çökünce session kaybı. Spring Session: Redis gibi merkezi store → tüm instance aynı session'ı görür → instance fark etmez.

32. JWT ile Spring Session'ı karşılaştır. Ne zaman hangisi?

    > **Beklenen:** JWT (stateless): token client'ta, server'da state yok. Avantaj: ölçekleme kolay, Redis bağımlılığı yok, microservice'lerde kolay paylaşım. Dezavantaj: logout/invalidation zor (token expire olana kadar geçerli). Spring Session (stateful): server'da state. Avantaj: anlık logout, server'da kontrol. Dezavantaj: Redis bağımlılığı, her request Redis round-trip. Seçim: yatay ölçekleme + stateless → JWT. Anlık invalidation kritik, kompleks session state → Spring Session.

---

## Karma — Architect Seviyesi

33. **"Yüksek trafikli bir Spring Boot uygulamasının başlangıç süresini nasıl optimize edersin?"**

    > **Beklenen:** (1) `spring.main.lazy-initialization=true`: tüm bean'leri lazy yap, ilk kullanımda yükle. (2) Component scan daralt: `scanBasePackages="com.myapp.specific"`. (3) Gereksiz auto-config exclude et (`--debug` ile tespit et). (4) CDS (Class Data Sharing): `-Xshare:dump` ile class metadata cache. (5) Spring Boot 3.3+ AOT (Ahead-of-Time): build-time proxying, condition evaluation → runtime overhead kaldır. (6) GraalVM Native: JVM warm-up yok, anında başlangıç — ama build süresi ve reflection kısıtlamaları. (7) Virtual Thread + WebFlux: daha az thread → daha az memory → daha hızlı start.

34. **"Kafka consumer'ın işlediği mesaj sayısı 1 saatte 1M'den 100K'ya düştü. Nasıl debug edersin?"**

    > **Beklenen:** (1) Consumer lag: `kafka-consumer-groups.sh --describe` → lag artıyor mu? (2) Actuator metrics: `kafka.consumer.fetch-latency-avg`, `kafka.consumer.records-consumed-rate`. (3) Thread dump: consumer thread'ler bloklu mu? (4) GC: Full GC stop-the-world → consumer poll durakladı → rebalance mi tetiklendi? (5) Dış bağımlılık: DB yavaşladı mı? HTTP dış servis timeout? (6) Partition: partition sayısı < consumer thread sayısı → bazı thread'ler boşta. (7) max.poll.records: çok büyükse her poll uzun → interval aşılır → rebalance.

35. **"`@Transactional` Kafka'da nasıl çalışır? Outbox Pattern ile farkı?"**

    > **Beklenen:** Kafka transactional producer: `spring.kafka.producer.transaction-id-prefix` → `kafkaTemplate.executeInTransaction()` → ya tüm mesajlar commit olur ya hiçbiri. Sadece Kafka içinde atomik. Sorun: DB commit + Kafka commit iki ayrı system → iki fazlı commit olmadan atomik değil. DB commit oldu, Kafka commit olmadıysa → kayıp. Outbox Pattern: DB transaction içinde outbox tablosuna yaz (Kafka'ya değil). Debezium CDC outbox tablosunu okur → Kafka'ya publish eder. DB commit = Kafka publish garantisi. Tam atomik, Kafka transaction gerekmez.
