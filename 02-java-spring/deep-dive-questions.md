# 02 — Java & Spring Deep Dive: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: JVM Internals

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 1: Serverless / container ortamında JVM warm-up gecikmesi**

```
Senaryo:
  AWS Lambda üzerinde Spring Boot çalışıyor.
  İlk istek 8-12 saniye sürüyor, sonraki istekler 50ms.
  "Cold start" problemi — müşteri şikayeti.

Neden?
  JVM bytecode'u önce interpret eder (~10.000 çağrı eşiği).
  Spring context initialization (bean scan, proxy oluşturma).
  JIT henüz native code üretmedi.

Çözümler:
  1. GraalVM Native Image: AOT compilation → JIT warm-up yok, anında hız
     Dezavantaj: reflection kısıtlı, build süresi çok uzun (10-20 dk)

  2. Spring Boot 3 + CDS (Class Data Sharing):
     -Xshare:dump → class metadata cache → yeniden kullan
     Warm-up %30-50 azalır

  3. Provisioned Concurrency (AWS Lambda): instance'ı sıcak tut, ücretlisi

  4. Kubernetes: readinessProbe başarılı olana kadar trafik gitme
     → JVM ısındıktan sonra servis devreye girer
```

---

**Sorun 2: Memory Leak — ClassLoader sızıntısı**

```
Senaryo:
  Uygulama günlerce çalışınca Metaspace dolup OutOfMemoryError fırlatıyor.
  "Metaspace: java.lang.OutOfMemoryError: Metaspace"

Neden?
  Custom ClassLoader'lar her deploy'da yeni sınıflar yüklüyor ama eski
  ClassLoader GC edilemiyor → sınıf metadata biriktiyor.
  Yaygın: JDBC driver, web container (Tomcat) hotdeploy, OSGi.

Tespit:
  jmap -clstats <pid>  → ClassLoader istatistikleri
  VisualVM → Metaspace kullanımı grafiği → sürekli artıyor?

Düzeltme:
  ClassLoader sızıntısı: static alanda ClassLoader referansı tutmama.
  JDBC: Driver.deregister() ile kaldır (@PreDestroy).
  Boyut artırma (geçici): -XX:MaxMetaspaceSize=512m
```

---

**Sorun 3: Full GC Storm — Stop-the-World felaketi**

```
Senaryo:
  Production servis günde 1-2 kez 5-10 saniye cevap vermiyor.
  GC log'larında: "Full GC (Allocation Failure)" — 8 saniye pause.
  Timeout olan istekler → kullanıcı hataları.

Neden?
  Old Generation doldu → Full GC → tüm thread'ler durdu (STW).
  Genellikle: memory leak, çok büyük nesneler, yanlış GC ayarı.

Tespit:
  GC log aktif et:
  -Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m
  
  GC Viewer veya GCeasy.io ile analiz.
  
  jstat -gcutil <pid> 1000 → her saniye GC istatistiği

Düzeltme:
  Heap yeterliyse ama Full GC varsa: memory leak araştır.
  jmap -histo:live <pid> → en çok yer kaplayan sınıflar.
  G1GC geç: -XX:+UseG1GC -XX:MaxGCPauseMillis=200
  Heap'i uygun ayarla: Xms=Xmx (GC resize overhead kaldırılır).
```

---

**Sorun 4: volatile olmadan visibility bug**

```java
// Gerçek production bug örneği
public class ServiceRegistry {
    private static boolean initialized = false;  // volatile yok!
    private static Config config;

    public static void init(Config c) {
        config = c;
        initialized = true;  // Thread A yazdı
    }

    public static Config getConfig() {
        if (!initialized) throw new IllegalStateException();  // Thread B eski değeri görür!
        return config;
    }
}

// Sonuç:
// Thread A init() çağırdı, initialized = true yaptı.
// Thread B CPU cache'inden false okuyabilir → IllegalStateException
// Hata intermittent (ara ara) → debug edilmesi çok zor

// Düzeltme:
private static volatile boolean initialized = false;
private static volatile Config config;
// VEYA: AtomicReference<Config> kullan
// VEYA: double-checked locking with volatile
```

---

### Mülakat Soruları — JVM

**Junior / Mid:**

1. ClassLoader parent delegation model neden önemli?

   > **Beklenen:** Güvenlik — kötü amaçlı kod `java.lang.String`'i kendi implementasyonuyla ezleyemez. Bootstrap ClassLoader her zaman önce kontrol edilir. Aynı sınıfın iki kez yüklenmesini önler. Follow-up: Custom ClassLoader ne zaman yazılır? → Farklı versiyonları aynı JVM'de çalıştırmak (OSGi, plugin sistemi).

2. JIT compilation nedir? "Warm-up" neden önemli?

   > **Beklenen:** JVM önce interpret eder (~10K çağrıda JIT devreye girer). JIT: inlining, dead code elimination, escape analysis. Warm-up: İlk dakikada performans düşük, sonra yükselir. Production: load balancer'dan trafiği kademeli ver (gradual rollout), hazır olmadan trafik gitmesin (readinessProbe).

3. Minor GC ile Full GC arasındaki fark nedir? Hangisi daha tehlikeli?

   > **Beklenen:** Minor GC: sadece Young Generation, hızlı (<10ms). Full GC: tüm heap, Stop-the-World, saniyeler sürebilir. Full GC tehlikeli: tüm thread'ler durur → timeout, timeout retry → daha fazla yük → cascade failure. Çözüm: G1/ZGC gibi concurrent collector kullan.

---

**Senior / Architect:**

4. G1GC ile ZGC arasında nasıl seçim yaparsın?

   > **Beklenen:** G1GC: heap'i region'lara böler, öngörülebilir pause (<200ms hedef), 4-32GB heap için iyi. ZGC (Java 15+): concurrent compaction, pause <10ms, heap boyutundan bağımsız, terabayt heap bile olsa aynı pause. Seçim: Latency SLA <100ms ve büyük heap → ZGC. Genel servis → G1GC yeterli. Throughput > latency ise → Parallel GC.

5. "Uygulamam bazen 5 saniye donuyor" — JVM açısından nasıl debug edersin?

   > **Beklenen:** (1) GC log aktif et: `-Xlog:gc*` → Full GC var mı? (2) Thread dump al: `jstack <pid>` → tüm thread'ler lock bekliyor mu? (3) JFR (Java Flight Recorder): `jcmd <pid> JFR.start duration=60s` → en detaylı profil. (4) jstat: heap kullanımı trend. (5) CPU spike değil memory spike: heap dolup GC fırtınası mı?

6. Escape Analysis nedir? Neden önemli?

   > **Beklenen:** JIT'in bir optimizasyonu: nesnenin method dışına "kaçıp kaçmadığını" analiz eder. Kaçmıyorsa → heap yerine stack'e koyar (stack allocation) → GC baskısı sıfır. Lambda'lar, küçük value object'ler bundan yararlanır. Record ve sealed class → escape analysis için daha iyi ipuçları verir.

---

## Bölüm 2: Concurrency

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 5: Deadlock**

```java
// Klasik deadlock — production'da gerçekleşti
public class AccountTransferService {
    void transfer(Account from, Account to, int amount) {
        synchronized (from) {        // Thread A: from=Acc1, to=Acc2 → Acc1'i kilitliyor
            synchronized (to) {      // Thread A: Acc2'yi bekliyor
                from.debit(amount);
                to.credit(amount);
            }
        }
    }
}

// Thread A: transfer(Acc1, Acc2) → Acc1 kilitledi, Acc2 bekliyor
// Thread B: transfer(Acc2, Acc1) → Acc2 kilitledi, Acc1 bekliyor
// → Deadlock! İkisi de birbirini bekliyor, sistem dondu.

// Belirti:
// - Thread dump: "waiting to lock" zincirleri
// - Uygulama cevap vermiyor ama CPU düşük

// Düzeltme: Kilitleme sırasını sabitlemek
void transfer(Account from, Account to, int amount) {
    Account first  = from.getId() < to.getId() ? from : to;   // küçük ID her zaman önce
    Account second = from.getId() < to.getId() ? to   : from;

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
// Alternatif: tryLock() ile timeout → deadlock yerine hata fırlat
```

---

**Sorun 6: Thread pool yanlış boyutlandırma**

```
Senaryo A — Thread Starvation:
  REST API → ThreadPool(size=10) ile dış servise HTTP çağrısı.
  Dış servis yavaşladı → 10 thread hepsi bloklandı (I/O bekleme).
  Yeni istekler: "Queue is full" hatası → kullanıcıya 503.

  Çözüm: I/O-bound iş için ayrı thread pool.
  Veya: WebClient (non-blocking reactive) kullan → thread'i bloklamaz.

Senaryo B — Çok fazla thread (context switch overhead):
  Executors.newCachedThreadPool() → sınırsız thread oluşturur.
  1000 eş zamanlı istek → 1000 thread → %80 zaman context switch.
  CPU yüksek ama throughput düşük.

  Çözüm: newFixedThreadPool(N_CPU * 2) ile sınırla.
  CallerRunsPolicy: queue dolunca caller thread işi yapar → backpressure.
```

---

**Sorun 7: Race condition — count kaybı**

```java
// Gerçek bug: sayaç doğru değil
public class MetricsService {
    private int requestCount = 0;

    public void recordRequest() {
        requestCount++;  // YANLIŞ: 3 ayrı operasyon (read, increment, write)
        // Thread A: oku (5) → Thread B: oku (5) → her ikisi 6 yazar
        // Sonuç: 1 kayıp sayım
    }
}

// Düzeltme 1: AtomicInteger
private AtomicInteger requestCount = new AtomicInteger(0);
requestCount.incrementAndGet(); // CAS ile atomik

// Düzeltme 2: LongAdder (çok yüksek contention için daha hızlı)
private LongAdder requestCount = new LongAdder();
requestCount.increment();
long total = requestCount.sum(); // sonucu okumak için

// LongAdder vs AtomicLong:
// AtomicLong: tek değer, high contention'da CAS retry yoğun
// LongAdder: her thread kendi cell'inde sayar, sum() toplar → daha hızlı
```

---

**Sorun 8: ThreadLocal sızıntısı (özellikle thread pool'da)**

```java
// Thread pool'da ThreadLocal tehlikeli
public class UserContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

    public static void set(User user) { currentUser.set(user); }
    public static User get() { return currentUser.get(); }
}

// Sorun:
// Thread pool thread'leri yeniden kullanılır.
// Request 1: ThreadLocal'a User A koyuldu.
// Request işlendi, thread pool'a döndü.
// Request 2: ThreadLocal.remove() çağrılmadıysa User A hâlâ orada!
// → Request 2 yanlış kullanıcı context'iyle çalışıyor.

// Düzeltme: request sonunda temizle
try {
    UserContext.set(user);
    processRequest();
} finally {
    UserContext.remove();  // kritik!
}

// Daha iyi: Spring'in RequestScope bean'i kullan → otomatik temizlik
```

---

### Mülakat Soruları — Concurrency

**Junior / Mid:**

7. `synchronized` ve `ReentrantLock` arasındaki fark nedir?

   > **Beklenen:** synchronized: JVM keyword, basit, exception'da otomatik unlock. ReentrantLock: tryLock(timeout), lockInterruptibly(), fair lock (FIFO sırası), Condition (ince grained wait/signal). Seçim: Basit kritik bölge → synchronized yeterli. Timeout, interrupt veya fairness gerekiyorsa → ReentrantLock.

8. `volatile` neyi garanti eder, neyi garanti etmez?

   > **Beklenen:** Garanti eder: visibility (tüm thread'ler güncel değeri görür), ordering (reorder edilmez). Garanti etmez: atomicity. `volatile int count; count++` hâlâ thread-safe değil (read-modify-write). Tek bir read veya write atomik → volatile yeterli. Compound operasyon → AtomicInteger veya synchronized.

9. `ExecutorService`'te thread pool boyutunu nasıl seçersin?

   > **Beklenen:** CPU-bound: `N_CPU + 1` (bir thread I/O page fault'ta beklese bile CPU boşta kalmaz). I/O-bound: `N_CPU × (1 + bekleme_süresi / işlem_süresi)`. Örnek: CPU=8, I/O bekleme 90ms, işlem 10ms → 8 × (1 + 9) = 80 thread. Ama pratik: load test ile ölç, teorik formüle körü körüne güvenme.

---

**Senior / Architect:**

10. Deadlock nasıl tespit edilir ve önlenir?

    > **Beklenen:** Tespit: `jstack <pid>` → "Found 1 deadlock" çıktısı. JFR lock analizi. Önleme stratejileri: (1) Kilit sırasını sabitlemek (ID sırasına göre lock). (2) tryLock(timeout) → timeout'ta vazgeç, retry. (3) Tüm kilitleri tek seferde al (`tryLock()` birden fazla kaynak için). (4) Kilit granülaritesini azaltmak (daha kısa kritik bölge). (5) Lock-free veri yapıları (ConcurrentHashMap, AtomicReference).

11. ConcurrentHashMap Java 7 ve Java 8 arasındaki implementasyon farkı nedir?

    > **Beklenen:** Java 7: Segment-level locking — 16 segment, her segment ayrı lock → maksimum 16 concurrent write. Java 8: CAS (Compare-And-Swap) + synchronized bucket-level lock — her bucket ayrı lock → çok daha yüksek concurrency. Ayrıca linked list → balanced BST (8+ entry) → O(n) → O(log n) okuma. Java 8 versiyonu neredeyse lock-free için optimize edilmiş.

12. CompletableFuture ile paralel API çağrıları nasıl yapılır?

    ```java
    // Beklenen cevabın içermesi gereken pattern:
    CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> userService.getUser(id), executor);

    CompletableFuture<List<Order>> ordersFuture =
        CompletableFuture.supplyAsync(() -> orderService.getOrders(id), executor);

    CompletableFuture<UserDashboard> result =
        userFuture.thenCombine(ordersFuture,
            (user, orders) -> new UserDashboard(user, orders));

    // Hata yönetimi:
    result.exceptionally(ex -> fallbackDashboard())
          .get(5, TimeUnit.SECONDS); // timeout
    ```

    > **Dikkat noktaları:** Default ForkJoinPool.commonPool() yerine custom executor kullan (I/O işlemler için). `get()` blocking — virtual thread varsa sorun değil, yoksa `thenApply` chain'i devam ettir.

---

## Bölüm 3: Spring Internals

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 9: Self-invocation — @Transactional çalışmıyor**

```java
// Çok yaygın ve sinsi bir bug
@Service
public class OrderService {

    @Transactional
    public void saveOrder(Order order) {
        orderRepo.save(order);
        auditRepo.save(new AuditLog(order));
    }

    public void processAndSave(Order order) {
        validateOrder(order);
        this.saveOrder(order);  // YANLIŞ! Transaction açılmıyor.
        // 'this' proxy değil, gerçek nesne → AOP bypass!
    }
}

// Sonuç:
// saveOrder() çağrıldı, exception oldu, rollback olmadı.
// orderRepo.save() persist oldu, auditRepo.save() olmadı → tutarsız veri.

// Düzeltmeler:
// 1. processAndSave()'i de @Transactional yap
// 2. Self-inject et (application context üzerinden):
@Autowired OrderService self;  // Spring proxy inject eder
self.saveOrder(order);

// 3. Spring AspectJ weaving kullan (proxy değil, bytecode weaving)
// spring.aop.proxy-target-class=true + aspectj dependency

// 4. Mantığı ayır: processAndSave → ayrı bean'de
```

---

**Sorun 10: Circular dependency — bean oluşturulamıyor**

```
Hata:
  The dependencies of some of the beans in the application context
  form a cycle: ServiceA → ServiceB → ServiceA

Senaryo:
  ServiceA → ServiceB inject eder
  ServiceB → ServiceA inject eder
  Spring biri oluşturmadan diğerini oluşturamıyor.

Çözüm 1 (önerilen): Tasarımı gözden geçir
  Circular dependency genellikle kötü tasarım işareti.
  Ortak mantığı üçüncü bir sınıfa çıkar (ServiceC).

Çözüm 2: @Lazy injection
  @Autowired
  @Lazy
  private ServiceB serviceB;  // ilk kullanımda inject edilir

Çözüm 3: Setter injection (constructor injection kullanma)
  Spring field injection ile çözebilir ama constructor cycle çözemez.

Çözüm 4: Event-driven mimariye geç
  A → B call yerine A → event publish → B event consume
  → bağımlılık yönü tek yön
```

---

**Sorun 11: N+1 JPA sorunu — production yavaşlaması**

```
Senaryo:
  Admin endpoint: "tüm siparişleri getir"
  Monitoring: endpoint 50ms'den 8 saniyeye çıktı.

  Tespit:
  Hibernate istatistikleri: 1 + 1000 = 1001 sorgu!
  spring.jpa.properties.hibernate.generate_statistics=true

  Çıktı:
  Queries executed to database: 1001

Düzeltme:
```

```java
// Kötü (N+1):
@OneToMany(fetch = FetchType.LAZY)
List<OrderItem> items;

orderRepo.findAll().forEach(o -> o.getItems().size()); // her order için +1 sorgu

// İyi (JOIN FETCH):
@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);

// Veya @EntityGraph:
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findAll();

// DTO projection (daha performanslı — entity materialize etme):
@Query("SELECT new com.app.dto.OrderSummary(o.id, o.total, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();
```

---

**Sorun 12: Transaction isolation yanlış seçimi — dirty read**

```
Senaryo:
  Online rezervasyon sistemi: READ_UNCOMMITTED kullanılmış (hız için).

  Thread A: room 101'i rezerve ediyor (transaction başladı, henüz commit yok)
  Thread B: müsait oda arıyor → 101'i "müsait" görüyor → rezerve ediyor
  Thread A: rollback yaptı (ödeme başarısız)
  → İki müşteri aynı odayı rezerve etti!

Kural:
  READ_UNCOMMITTED: asla production'da.
  READ_COMMITTED: genellikle yeterli (Oracle default).
  REPEATABLE_READ: aynı transaction'da tutarlı okuma (MySQL InnoDB default).
  SERIALIZABLE: maksimum izolasyon, maximum lock contention — sadece zorunluysa.

Rezervasyon için:
  REPEATABLE_READ + SELECT FOR UPDATE (pessimistic)
  veya optimistic locking (@Version) + retry on OptimisticLockException.
```

---

### Mülakat Soruları — Spring Internals

**Junior / Mid:**

13. Spring Bean Lifecycle'ını açıkla. `@PostConstruct` ne zaman çağrılır?

    > **Beklenen:** Instantiation → Property injection → BeanPostProcessor before → @PostConstruct / afterPropertiesSet → BeanPostProcessor after → kullanıma hazır → @PreDestroy (kapanışta). @PostConstruct: injection tamamlandıktan sonra, bean kullanıma açılmadan önce. DB bağlantısı, cache warmup gibi init işlemleri için kullanılır.

14. `@Transactional` neden `private` metotlarda çalışmaz?

    > **Beklenen:** Spring AOP, proxy oluşturarak çalışır. CGLIB proxy subclass oluşturur ve metodu override eder. Private metotlar override edilemez → proxy intercept edemez → @Transactional görmezden gelinir. Çözüm: metodu public yap veya package-private (AspectJ weaving ile private de çalışır ama daha kompleks setup gerekir).

15. `@Autowired` field injection yerine constructor injection neden tercih edilir?

    > **Beklenen:** (1) Test edilebilirlik: Spring olmadan new MyService(mockRepo) yapabilirsin. (2) Immutability: field `final` olabilir. (3) Circular dependency erken tespiti: Spring başlarken hata verir, field injection'da runtime'da olabilir. (4) Açıklık: bağımlılıklar constructor'da görünür. Spring resmi dökümantasyonu da constructor injection önerir.

---

**Senior / Architect:**

16. Spring AOP'un kısıtlamaları nelerdir? Gerçek AOP (AspectJ) ile farkı?

    > **Beklenen:** Spring AOP kısıtlamaları: (1) Sadece Spring bean'lerine uygulanabilir (Spring dışı nesne → AOP yok). (2) Self-invocation çalışmaz (proxy bypass). (3) Private/final metotlar intercept edilemez. (4) Sadece runtime proxy → performans overhead. AspectJ: bytecode weaving (compile-time veya load-time), tüm nesnelere uygulanabilir, private metotlar da intercept edilebilir, zero overhead (compile time). Dezavantaj: build pipeline daha karmaşık.

17. `BeanPostProcessor` ve `BeanFactoryPostProcessor` farkı nedir? Ne zaman kullanırsın?

    > **Beklenen:** BeanFactoryPostProcessor: bean'ler instantiate edilmeden önce çalışır — BeanDefinition'ları değiştirir. PropertySourcesPlaceholderConfigurer bunu kullanır (`${property}` yerleştirme). BeanPostProcessor: her bean instantiate edildikten sonra çalışır — nesneyi wrap edebilir veya modifiye edebilir. Spring AOP proxy oluşturma bu şekilde yapılır. Kendi framework'ünüzde cross-cutting işlemler için kullanılır.

18. Spring Security Filter Chain'ine custom filter nasıl eklenir ve neden sıra önemlidir?

    ```java
    // Beklenen implementasyon pattern:
    @Component
    public class JwtAuthFilter extends OncePerRequestFilter {
        @Override
        protected void doFilterInternal(HttpServletRequest req,
                                        HttpServletResponse res,
                                        FilterChain chain) throws IOException, ServletException {
            String token = extractToken(req);
            if (token != null && jwtService.isValid(token)) {
                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(
                        jwtService.getUser(token), null,
                        jwtService.getAuthorities(token));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
            chain.doFilter(req, res);  // zinciri devam ettir
        }
    }

    // SecurityConfig'de:
    http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    ```

    > **Sıra neden önemli:** UsernamePasswordAuthenticationFilter'dan önce JWT kontrolü yapılmalı. Aksi hâlde Spring form login denemesi yapar. ExceptionTranslationFilter'dan sonra gelen filtreler 401/403'ü doğru handle eder.

---

## Bölüm 4: Auto-Configuration & Spring Boot

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 13: İstenmeyen auto-configuration — DataSource bean çakışması**

```
Hata:
  Failed to configure a DataSource: 'url' attribute is not specified.

Senaryo:
  spring-boot-starter-data-jpa dependency'si var ama DB kullanmıyoruz (mock service).
  Spring Boot JPA görüyor → DataSource konfigürasyonu zorunlu tutuyor.

Çözümler:
  1. Exclude et:
  @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})

  2. Properties ile:
  spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

  3. Gerçekten JPA gerekiyorsa: properties'e url ekle.

Debug: --debug flag → Conditions Evaluation Report:
  "DataSourceAutoConfiguration matched" veya "did not match" satırını bul.
```

---

**Sorun 14: Application property yanlış ortam — production'da dev ayarı**

```
Senaryo:
  application-dev.properties: logging.level.root=DEBUG, H2 console açık
  Production'da SPRING_PROFILES_ACTIVE set edilmedi → default profile aktif
  Hem H2 console public açık hem de DEBUG log: sensitive veri log'a düştü!

Düzeltme:
  Secure defaults: application.properties'e production-safe değerler koy.
  Dev/test overrides: application-dev.properties'de gevşek ayarlar.
  Kubernetes: env var ile SPRING_PROFILES_ACTIVE=production set et.
  
  Spring Boot Actuator:
  GET /actuator/env → hangi property nereden geliyor görürsün
  (ama /env'i production'da kapat!)
```

---

### Mülakat Soruları — Auto-Configuration

**Senior / Architect:**

19. `@ConditionalOnClass` ile `@ConditionalOnBean` farkı nedir?

    > **Beklenen:** `@ConditionalOnClass(Foo.class)`: classpath'te Foo sınıfı varsa bu bean oluşturulur (dependency var mı kontrolü). `@ConditionalOnBean(Foo.class)`: Spring context'te Foo bean'i varsa oluşturulur (başka bir bean oluşturuldu mu kontrolü). Auto-configuration yazarken sıra önemli: `@AutoConfigureAfter` ile bağımlı config'den sonra çalıştır.

20. Spring Boot uygulaman çok yavaş başlıyor. Nasıl optimize edersin?

    > **Beklenen:** (1) `spring.main.lazy-initialization=true` — tüm bean'leri lazy yap. (2) Component scan kapsamını daralt: `@SpringBootApplication(scanBasePackages = "com.myapp")`. (3) Gereksiz auto-config'leri exclude et. (4) Startup Actuator: `/actuator/startup` ile hangi adım ne kadar sürüyor. (5) CDS (Class Data Sharing): `-Xshare:dump`. (6) Virtual Thread + WebFlux → thread blocking yok.

---

## Bölüm 5: Performans & Connection Pool

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 15: HikariCP connection pool tükenmesi**

```
Hata:
  HikariPool-1 - Connection is not available, request timed out after 30000ms.

Senaryo:
  maximum-pool-size: 10
  Yavaş bir dış servis çağrısı içinde DB transaction tutuyoruz.
  Dış servis 30 saniye yavaşladı → 10 thread 30 saniye bloklandı
  → Yeni istekler 30 saniye bekleyip timeout → cascade failure.

Kök neden: Transaction içinde uzun süre bekleme (I/O, dış servis çağrısı).

Düzeltme:
  1. Transaction dışında dış servis çağrısı yap:
     data = externalService.fetch();   // transaction dışında
     @Transactional processAndSave(data);  // sadece DB işlemi

  2. Pool size artır (yeterli DB bağlantısı varsa).
     Ama: DB max_connections da sınırlı! Tüm servislerin toplamı aşmamalı.

  3. Circuit breaker: dış servis yavaşlayınca hızlı fail, pool serbest kal.

  4. Connection timeout kısalt: bekleme süresini 30s → 3s. Hızlı hata daha iyi.
```

---

**Sorun 16: Pool size ile thread pool uyumsuzluğu**

```
Senaryo:
  REST API: thread pool = 200 (embedded Tomcat default)
  HikariCP: maximum-pool-size = 10
  Yük altında: 200 istek aynı anda → 190 tanesi connection için bekliyor.
  Timeout hatası yağıyor.

Kural:
  pool_size ≥ işlemci thread sayısı
  Ama: DB'ye gitmeden önce iş yapılıyorsa daha az yeterli.

Pratik formül (HikariCP önerisi):
  connections = (core_count × 2) + effective_spindle_count
  Örnek: 4 core, SSD (spindle=1) → (4×2)+1 = 9 → 10 yeterli.

  Ama 200 thread 10 connection için yarışıyorsa → ayarla:
  - Thread pool küçült (işin gerçekten ne kadar CPU kullandığına bak)
  - Async/reactive geç → thread başına bağlantı bağımlılığını kır
```

---

### Mülakat Soruları — Performans

**Senior / Architect:**

21. HikariCP pool size nasıl belirlenir? "Daha fazla = daha iyi" mi?

    > **Beklenen:** Hayır. DB'nin de maksimum bağlantı limiti var. Pool çok büyükse: DB tarafında lock contention, memory kullanımı. Çok küçükse: connection bekler, throughput düşer. Formül: `(core_count × 2) + spindle_count`. SSD: spindle=1. 4 core SSD → ~10. Load test ile gerçek değeri ölç. Her mikroservis instance'ı için düşün: 5 pod × 10 connection = 50 DB bağlantısı → DB max_connections'ı geçiyor mu?

22. `@Transactional` metodun içine uzun süren bir işlem koymanın tehlikeleri nelerdir?

    > **Beklenen:** Transaction açık kaldıkça DB bağlantısı tutulur (HikariCP connection serbest bırakılmaz). Lock tutulur (UPDATE/SELECT FOR UPDATE → başka transaction bekler). Uzun transaction = DB lock contention = cascading slowdown. Çözüm: Transaction'ı mümkün olduğunca kısa tut. Dış servis çağrısını transaction dışına al. Büyük batch işlemini chunk'lara böl (Spring Batch).

---

## Karma — Architect Seviyesi Zor Sorular

23. **"JVM'in Garbage Collector'ı neden durması (Stop-the-World) gerekiyor?"**

    > **Beklenen:** Compaction (object'leri belleği sıkıştırma) sırasında nesnelerin pointer'ları değişir. Eğer thread'ler çalışmaya devam ederse: pointer güncellenirken okunabilir → yanlış nesneye erişim → memory corruption. Modern GC'ler (G1, ZGC): concurrent marking (STW olmadan) + çok kısa STW sadece final'de. ZGC: pointer coloring (load barrier) ile STW'yi <10ms'ye indirdi.

24. **"Spring'de `@Transactional(propagation=REQUIRES_NEW)` ne zaman kullanılmalı? Gerçek bir senaryo ver."**

    > **Beklenen:** Dış transaction'dan bağımsız commit/rollback gerektiğinde. Senaryo: Her işlem denemesini audit_log tablosuna yaz — işlem başarısız olsa bile audit kaydı commit edilmeli. `@Transactional(propagation=REQUIRES_NEW)` → audit kayıt kendi transaction'ında → outer transaction rollback yapsa bile audit persist olur. Dikkat: REQUIRES_NEW her zaman yeni DB connection açar — pool baskısı oluşturabilir.

25. **"Reactive programming (WebFlux) ne zaman tercih edilmeli, ne zaman edilmemeli?"**

    > **Beklenen:** Tercih et: Çok sayıda eş zamanlı I/O-bound bağlantı (WebSocket, SSE, streaming), az CPU - çok bekleme senaryosu, thread pool starvation sorunu yaşıyorsan. Tercih etme: CPU-bound işlemler (ML inference, görüntü işleme → thread pool daha iyi), takım reaktif programlamaya yabancıysa (learning curve yüksek), tüm stack'in reactive olması gerekiyor (JDBC senkron — R2DBC gerekir). Java 21 Virtual Thread: I/O-bound sorununu senkron kod ile çözer → WebFlux ihtiyacı azaldı.

26. **"Bir production JVM uygulamasında memory leak var. Nasıl bulursun?"**

    > **Beklenen:** (1) Heap kullanımı trend'i: `jstat -gcutil <pid> 5000` → Old Gen sürekli artıyor mu? (2) Heap dump al: `jmap -dump:live,format=b,file=heap.hprof <pid>` (3) Eclipse MAT veya VisualVM ile analiz: Dominator Tree → en çok hafıza tutan nesne hangisi? (4) Şüpheli: static Map/List'e sürekli ekleme (temizleme yok), ThreadLocal (remove() eksik), listener/observer deregistration eksik, ClassLoader sızıntısı, Cache (boyutsuz büyüyen), unclosed stream/connection.

27. **"Spring Security'de 401 ile 403 farkı nedir? Spring hangisinde ne yapar?"**

    > **Beklenen:** 401 Unauthorized: kimlik doğrulanmadı (token yok / geçersiz). 403 Forbidden: kimlik doğrulandı ama yetki yok. Spring: `AuthenticationException` → ExceptionTranslationFilter → 401. `AccessDeniedException` → 403. Önemli nüans: Spring Security anonim kullanıcıya da 403 değil 401 döner (çünkü authenticate olursa erişebilir olabilir). `AuthenticationEntryPoint` → 401. `AccessDeniedHandler` → 403. İkisini özelleştirmek için her ikisini de configure et.
