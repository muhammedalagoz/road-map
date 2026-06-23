# 02 — Java & Spring Ekosistemi Deep Dive

## 1. JVM Nasıl Çalışır

### Bytecode ve Classloader

```
Java Kodu (.java)
      ↓ javac (compiler)
Bytecode (.class)
      ↓ JVM
ClassLoader → Bytecode Verifier → Interpreter/JIT → Native Code → CPU
```

**ClassLoader hiyerarşisi (parent delegation model):**
```
Bootstrap ClassLoader (rt.jar, java.lang.*)
        ↑
Extension ClassLoader (jre/lib/ext)
        ↑
Application ClassLoader (classpath'indeki sınıflar)
        ↑
Custom ClassLoader (uygulama özelinde)
```

Bir sınıf yüklenirken önce **parent'a sorulur**, parent bulamazsa child dener. Bu güvenlik için önemlidir — java.lang.String'i kendi ClassLoader'ınla ezleyemezsin.

### JIT Compilation (Just-In-Time)

JVM başlangıçta bytecode'u **interpret** eder (yavaş ama hemen başlar). Sık çalışan metodları tespit edip **native machine code'a çevirir** (hızlı). Bu "warm-up" süreci serverless ortamlarda önemli bir problemdir.

```
Method çağrıları sayılır
→ Eşik aşılır (varsayılan ~10000)
→ JIT compiler devreye girer
→ Method native code olarak cache'lenir
→ Sonraki çağrılar direkt native çalışır
```

**JIT optimizasyonları:**
- **Inlining** — kısa metodları çağıran koda gömer
- **Dead code elimination** — hiç ulaşılmayan kodu siler
- **Escape analysis** — heap yerine stack'e nesne koyar
- **Loop unrolling** — döngü overhead'ini azaltır

### Garbage Collection

#### Heap Yapısı
```
JVM Heap
├── Young Generation (kısa ömürlü nesneler)
│   ├── Eden Space (yeni nesneler buraya gelir)
│   ├── Survivor 0 (S0)
│   └── Survivor 1 (S1)
└── Old Generation / Tenured (uzun yaşayan nesneler)

Non-Heap
└── Metaspace (sınıf metadata — Java 8+ ile PermGen yerine)
```

#### GC Döngüsü
1. Yeni nesne Eden'a eklenir
2. Eden dolunca **Minor GC** tetiklenir
3. Canlı nesneler S0'a taşınır
4. Birkaç Minor GC sonrası S0→S1→Old Generation'a promotion
5. Old Generation dolunca **Major/Full GC** tetiklenir (uzun sürer, "Stop the World")

#### GC Algoritmaları

| GC | Ne zaman? | Özellik |
|----|-----------|---------|
| **Serial GC** | Küçük heap, tek thread | Basit, production'da kullanma |
| **Parallel GC** | Throughput odaklı, batch işler | Multiple thread, Stop-the-World |
| **G1 GC** | Büyük heap, düşük latency | Heap'i region'lara böler, öngörülebilir pause |
| **ZGC** | Ultra-low latency (< 10ms) | Concurrent, heap boyutundan bağımsız pause |
| **Shenandoah** | Benzer ZGC, Red Hat geliştirdi | Concurrent compaction |

**Architect kararı:** Web servisleri için **G1 GC** (varsayılan Java 9+), çok büyük heap (>32GB) veya latency kritik sistemler için **ZGC**.

### Java Memory Model

```
Thread Stack (her thread için ayrı)
├── Local variables
├── Method parameters
└── Stack frames

JVM Heap (tüm thread'ler paylaşır)
├── Object instances
└── Arrays

Metaspace
└── Class definitions, method bytecode
```

**Visibility problemi:**
CPU'ların kendi cache'leri var. Bir thread bir değeri değiştirirse, diğer thread CPU cache'inden eski değeri okuyabilir.

```java
// BU KOD YANLIŞ — flag değişimi görünmeyebilir
boolean flag = false;

// Thread 1
flag = true;

// Thread 2
while (!flag) { } // sonsuza kadar döngüde kalabilir (CPU cache)
```

**volatile** anahtar kelimesi: Her read/write direkt main memory'den yapılır, CPU cache bypass edilir.

**happens-before ilişkisi:** JMM'nin temel garantisi — A işlemi B'den önce görünür garanti edilmişse, B A'nın sonuçlarını görecektir.

---

## 2. Concurrency Derinlemesi

### synchronized vs Lock

```java
// synchronized — JVM keyword, basit
synchronized (this) {
    count++;
}

// ReentrantLock — daha esnek
Lock lock = new ReentrantLock();
lock.lock();
try {
    count++;
} finally {
    lock.unlock(); // finally önemli!
}
```

**ReentrantLock avantajları:**
- `tryLock(timeout)` — belirli süre bekle, timeout'ta vazgeç
- `lockInterruptibly()` — interrupt edilebilir bekleme
- `Condition` — daha ince grained wait/notify

### Atomic Classes

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // CAS (Compare-And-Swap) kullanır, lock gerektirmez

// CAS mekanizması:
// 1. Mevcut değeri oku (expected)
// 2. Hesapla (newValue)
// 3. Eğer hâlâ expected ise, newValue ile değiştir (atomik CPU operasyonu)
// 4. Aksi hâlde retry
```

### volatile, synchronized, Lock Seçim Kılavuzu

```
Tek değişken → sadece read/write → volatile
Tek değişken → increment gibi compound op → AtomicXxx
Birden fazla değişken → synchronized veya Lock
Okuma çok, yazma az → ReadWriteLock
Sıra önemli, fairness → ReentrantLock(true)
```

### ExecutorService & Thread Pool

```java
// Thread pool: thread yaratma maliyetini amortize eder
ExecutorService executor = Executors.newFixedThreadPool(10);

// Daha iyi: ThreadPoolExecutor ile full kontrol
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,              // corePoolSize: her zaman yaşayan thread sayısı
    20,             // maximumPoolSize: maksimum thread
    60, TimeUnit.SECONDS, // keepAliveTime: idle thread ne kadar yaşar
    new ArrayBlockingQueue<>(100), // work queue
    new ThreadPoolExecutor.CallerRunsPolicy() // queue dolunca ne yap
);
```

**Architect sorusu:** Thread pool boyutu nasıl seçilir?
- **CPU-bound:** `N_CPU + 1` thread (CPU waste azaltılır)
- **I/O-bound:** `N_CPU * (1 + wait_time/compute_time)` — genellikle 2-10x CPU sayısı

---

## 3. Spring Framework Internals

### IoC Container Nasıl Çalışır

```
ApplicationContext oluşturulur
        ↓
BeanDefinition'lar yüklenir (XML, @Component scan, @Bean metodlar)
        ↓
BeanDefinitionRegistry'ye kaydedilir
        ↓
BeanFactory sıraya göre instantiate eder
  → Constructor injection
  → BeanPostProcessor'lar devreye girer
  → @PostConstruct çağrılır
  → Bean kullanıma hazır
        ↓
ApplicationContext kapanınca:
  → @PreDestroy çağrılır
  → DisposableBean.destroy()
```

### Bean Lifecycle Detayı

```
1. Instantiation (constructor çağrısı)
2. Property Population (@Autowired field injection)
3. BeanNameAware.setBeanName()
4. ApplicationContextAware.setApplicationContext()
5. BeanPostProcessor.postProcessBeforeInitialization()
6. @PostConstruct / InitializingBean.afterPropertiesSet()
7. BeanPostProcessor.postProcessAfterInitialization()
   ← Bean burada kullanıma hazır
8. @PreDestroy / DisposableBean.destroy() (container kapanırken)
```

### Spring AOP: Proxy Mekanizması

Spring AOP'un tüm sihri **proxy** nesnelerde gizli.

```
Sen inject edersin: OrderService
Gerçekte gelir: OrderService$$SpringCGLIB$$0 (proxy)

Proxy.save(order):
    1. advice before()  ← @Before
    2. begin transaction
    3. target.save(order) ← gerçek kod
    4. commit transaction
    5. advice after()   ← @After
```

**İki proxy türü:**
- **JDK Dynamic Proxy** — interface'e sahipse kullanılır (`java.lang.reflect.Proxy`)
- **CGLIB** — concrete class'sa, bytecode'u runtime'da değiştirerek subclass oluşturur

```java
// @Transactional aslında AOP proxy'sidir:
@Service
class OrderService {
    @Transactional // → proxy intercept eder → transaction başlatır
    public void save(Order order) {
        repo.save(order);
    }
}

// UYARI: Self-invocation AOP'u bypass eder!
@Service
class OrderService {
    @Transactional
    public void save(Order order) { /* ... */ }
    
    public void process(Order order) {
        this.save(order); // YANLIŞ — proxy bypass edilir, transaction açılmaz
        // Doğrusu: başka bean'den çağır, veya self-inject et
    }
}
```

### Spring Boot Auto-Configuration

```
@SpringBootApplication
    └── @EnableAutoConfiguration
            └── AutoConfigurationImportSelector
                    └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports okur
                            └── Koşullara göre (@ConditionalOnClass, @ConditionalOnMissingBean) bean'leri oluşturur
```

```java
// Bir auto-configuration nasıl yazılır:
@AutoConfiguration
@ConditionalOnClass(DataSource.class)      // classpath'te varsa
@ConditionalOnMissingBean(DataSource.class) // kullanıcı tanımlamamışsa
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
            .url(props.getUrl())
            .build();
    }
}
```

**Debug için:** `--debug` flag ile çalıştır → hangi auto-config devreye girdi, hangisi neden girmedi görürsün.

### Spring MVC Request Lifecycle

```
HTTP Request
      ↓
DispatcherServlet (front controller)
      ↓
HandlerMapping → hangi @Controller @RequestMapping ile eşleşiyor?
      ↓
HandlerAdapter → @RequestBody, @PathVariable gibi argümanları resolve eder
      ↓
Interceptor.preHandle()
      ↓
Controller.method() → ResponseBody veya ModelAndView
      ↓
Interceptor.postHandle()
      ↓
ViewResolver (JSON için: Jackson HttpMessageConverter)
      ↓
Interceptor.afterCompletion()
      ↓
HTTP Response
```

### Spring Data JPA — N+1 Problemi

```java
// N+1 problemi:
List<Order> orders = orderRepo.findAll(); // 1 sorgu
for (Order order : orders) {
    System.out.println(order.getCustomer().getName()); // her Order için 1 sorgu!
    // 100 order varsa → 1 + 100 = 101 sorgu
}

// Çözüm 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();

// Çözüm 2: @EntityGraph
@EntityGraph(attributePaths = {"customer"})
List<Order> findAll();

// Çözüm 3: Batch size (N+1'i azaltır ama elimine etmez)
@OneToMany
@BatchSize(size = 20) // 20'şerli gruplarla yükle
List<OrderItem> items;
```

### Transaction Management

```java
@Transactional(
    propagation = Propagation.REQUIRED,    // varsayılan: mevcut transaction'a katıl veya yeni aç
    isolation = Isolation.READ_COMMITTED,  // dirty read engeller
    rollbackFor = Exception.class,         // checked exception'da da rollback
    timeout = 30                           // 30 saniye sonra timeout
)
public void processOrder(Order order) { ... }
```

**Propagation türleri:**

| Propagation | Davranış |
|-------------|----------|
| REQUIRED | Mevcut varsa katıl, yoksa yeni aç (default) |
| REQUIRES_NEW | Her zaman yeni transaction, mevcut suspend edilir |
| SUPPORTS | Mevcut varsa katıl, yoksa transaction'sız çalış |
| MANDATORY | Mevcut olmalı, yoksa exception fırlat |
| NEVER | Transaction olmamalı, varsa exception fırlat |
| NESTED | Savepoint kullanan nested transaction |

**Isolation seviyeleri:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| READ_UNCOMMITTED | ✓ olabilir | ✓ olabilir | ✓ olabilir |
| READ_COMMITTED | ✗ | ✓ olabilir | ✓ olabilir |
| REPEATABLE_READ | ✗ | ✗ | ✓ olabilir |
| SERIALIZABLE | ✗ | ✗ | ✗ |

---

## 4. Spring Security

### Filter Chain

```
HTTP Request → FilterChain
    SecurityContextPersistenceFilter  (session'dan context yükle)
    UsernamePasswordAuthenticationFilter (login)
    BasicAuthenticationFilter (Basic auth header)
    BearerTokenAuthenticationFilter (JWT)
    ExceptionTranslationFilter (auth/authz exception → 401/403)
    FilterSecurityInterceptor (metod/URL bazlı yetkilendirme)
```

### Authentication vs Authorization Akışı

```
Authentication (Kimsin?):
1. UsernamePasswordAuthenticationFilter
2. AuthenticationManager.authenticate(token)
3. UserDetailsService.loadUserByUsername(username)
4. PasswordEncoder.matches(raw, encoded)
5. Başarılıysa: SecurityContextHolder'a Authentication koyulur

Authorization (Ne yapabilirsin?):
1. Method veya URL'ye erişim
2. SecurityContext'ten Authentication alınır
3. GrantedAuthority'ler kontrol edilir
4. @PreAuthorize("hasRole('ADMIN')") veya antMatchers().hasRole()
```

---

## 5. Performans İpuçları

### Connection Pool (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10    # thread pool size ile uyumlu olmalı
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

**Formül:** `pool_size = (core_count * 2) + effective_spindle_count`

### GC Tuning Başlangıç Noktası

```bash
java -Xms2g -Xmx2g \         # heap boyutu, min=max (GC'yi stabilize eder)
     -XX:+UseG1GC \           # G1 kullan
     -XX:MaxGCPauseMillis=200 \ # hedef pause süresi
     -XX:G1HeapRegionSize=16m \ # büyük nesneler için
     -jar app.jar
```
