# 02b — Spring Framework: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Spring Core — IoC Container

### Gerçek Hayat Sorunları

---

**Sorun 1: Prototype bean singleton'a inject edildi — hiç değişmedi**

```java
// Senaryo: Her kullanıcı isteği için yeni bir "context" nesnesi istiyoruz.
// Scope'u prototype yaptık ama hep aynı nesneyi görüyoruz.

@Component
@Scope("prototype")
class RequestContext {
    private final UUID id = UUID.randomUUID();
    public UUID getId() { return id; }
}

@Service  // singleton
class OrderService {
    @Autowired
    private RequestContext ctx;  // YANLIŞ — bir kez inject edildi, hiç değişmez!

    void process(Order order) {
        log.info("ctx.id={}", ctx.getId());  // her çağrıda aynı UUID
    }
}

// Belirti: Her istek aynı UUID → context izolasyonu yok → veri karışması.

// DÜZELTME — ObjectProvider: her getObject() çağrısında yeni instance
@Service
class OrderService {
    @Autowired
    private ObjectProvider<RequestContext> ctxProvider;

    void process(Order order) {
        RequestContext ctx = ctxProvider.getObject();  // her seferinde yeni
        log.info("ctx.id={}", ctx.getId());
    }
}

// Alternatif: @Lookup metod injection
@Service
abstract class OrderService {
    @Lookup
    protected abstract RequestContext requestContext();  // Spring override eder

    void process(Order order) {
        RequestContext ctx = requestContext();  // her seferinde yeni prototype
    }
}
```

---

**Sorun 2: Circular dependency — constructor injection ile oluşturulamıyor**

```
Senaryo:
  OrderService → PaymentService (constructor inject)
  PaymentService → OrderService (constructor inject)

Belirti: Startup'ta BeanCurrentlyInCreationException:
  "The dependencies of some of the beans in the application context
   form a cycle: orderService -> paymentService -> orderService"

Spring 6+ varsayılan: constructor injection tercih edildiğinde circular dep tespit edilir.
Field injection (@Autowired field) ile Spring proxy kullanarak "çözer" — ama bu kötü tasarım sinyali.

DÜZELTME (doğru):
  1. Tasarımı düzelt: Ortak işlevi ayrı bir sınıfa taşı.
     OrderService + PaymentService → ikisi de OrderPaymentCoordinator'a bağımlı.

  2. Event pub/sub: OrderService event yayınlar → PaymentService dinler.
     Doğrudan bağımlılık ortadan kalkar.

  3. @Lazy (geçici çözüm — kötü tasarımı gizler):
     @Lazy @Autowired PaymentService payment;
     → İlk erişime kadar initialize etme → döngüyü kır.
     Production'da kullanmaktan kaçın, tasarım borcunu biriktiriyor.
```

---

**Sorun 3: @PostConstruct exception — bean yaratılmıyor, sessiz hata**

```java
// Senaryo: DataSource bağlantısını doğrulamak için @PostConstruct kullandık.
// DB down → startup fail ama hata mesajı karmaşık.

@Service
class ProductCatalogService {
    @Autowired DataSource dataSource;

    @PostConstruct
    void validateConnection() throws SQLException {
        dataSource.getConnection().close();  // DB down → exception
        // Spring: BeanCreationException: Error creating bean 'productCatalogService':
        //         Invocation of init method failed
        // Gerçek neden: log'larda birkaç satır aşağıda
    }
}

// DÜZELTME 1: Fail-fast istiyorsan bu doğru ama log seviyesine dikkat et.
// Spring Boot Actuator healthcheck daha iyi alternatif:
@Component
class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try (var conn = dataSource.getConnection()) {
            conn.isValid(1);
            return Health.up().build();
        } catch (SQLException e) {
            return Health.down(e).build();  // uygulama çalışır ama unhealthy raporlar
        }
    }
}
// K8s readinessProbe → unhealthy → trafik yönlendirilmez ama pod çalışmaya devam eder.

// DÜZELTME 2: @PostConstruct'ta DB bağlantısı test etme, ApplicationReadyEvent kullan:
@EventListener(ApplicationReadyEvent.class)
void onReady() { validateConnection(); }
// Startup tamamlandı → şimdi kontrol et → fail varsa log at ve alert gönder.
```

---

### Mülakat Soruları — Spring Core

**Junior / Mid:**

1. `@Autowired` field injection yerine constructor injection neden tercih edilir?

   > **Beklenen:** (1) Test: mock geçirmek için Spring context gerekmez — new MyService(mockDep) yeterli. Field injection: reflection ile inject gerekir veya Spring runner. (2) Immutability: `final` field olabilir — nesne oluşturulduğunda garanti. (3) Zorunluluk: Constructor null olamaz → null dep ile nesne yaratılamaz. Field: @Autowired null olabilir, runtime NullPointerException. (4) Circular dep: Constructor injection → Spring startup'ta tespit eder, field → gizler. Kural: zorunlu bağımlılık → constructor, opsiyonel → setter.

2. Bean scope'ları nelerdir? `singleton` ile `prototype` ne zaman hangisi?

   > **Beklenen:** Singleton: container başına 1 instance (default). Prototype: her inject/getBean'de yeni. Request: HTTP request başına. Session: HTTP session başına. Singleton: stateless service, repository, utility. Prototype: stateful nesne (her istek farklı state taşıyan), işlem başına context nesnesi. Tuzak: prototype'ı singleton'a inject → prototype hiç değişmez → ObjectProvider kullan.

3. `BeanFactory` ile `ApplicationContext` arasındaki fark nedir?

   > **Beklenen:** BeanFactory: temel DI container, lazy init (ilk kullanımda yükler). ApplicationContext: BeanFactory üstüne ekler — eager init (startup'ta hepsi hazır), i18n (MessageSource), event pub/sub, environment/property, @Scheduled/@Async desteği. Pratik: neredeyse her zaman ApplicationContext kullanılır. BeanFactory: çok kısıtlı ortam (embedded, Android). Spring Boot her zaman ApplicationContext sağlar.

---

**Senior / Architect:**

4. `BeanPostProcessor` ile `BeanFactoryPostProcessor` farkı nedir? Hangisini ne zaman yazarsın?

   > **Beklenen:** BeanFactoryPostProcessor: bean tanımları (BeanDefinition) oluşturulduktan sonra, bean'ler instantiate edilmeden önce çalışır. BeanDefinition'ı değiştirebilir — örnek: PropertySourcesPlaceholderConfigurer (${...} placeholder'ları property'lerle değiştirir). BeanPostProcessor: her bean oluşturulduktan sonra, before/afterInitialization. Proxy oluşturma burada olur — @Transactional, @Async, @Cacheable proxy'leri BeanPostProcessor tarafından sarılır. Ne zaman: meta-data değiştir → BeanFactoryPostProcessor. Bean instance'ını wrap et (proxy, decorator) → BeanPostProcessor.

5. Spring singleton bean thread-safe midir?

   > **Beklenen:** Hayır, doğası gereği değil. Singleton: tek instance → birden fazla thread aynı anda kullanabilir. State taşıyan field varsa → race condition. Örnek: `private List<String> cache = new ArrayList<>()` → concurrent modification. Doğru tasarım: singleton bean'ler stateless olmalı. State gerekliyse: (1) ThreadLocal, (2) synchronized, (3) AtomicReference, (4) prototype scope. Spring servisleri genellikle stateless tasarlandığı için sorun çıkmaz — ama yanlış kod → bug.

---

## Bölüm 2: Spring AOP

### Gerçek Hayat Sorunları

---

**Sorun 4: Self-invocation tuzağı — @Transactional çalışmadı**

```java
// Senaryo: Sipariş verilince hem order hem inventory kaydedilmeli.
// Hata durumunda ikisi de rollback olmalı. Ama inventory yazılmaya devam ediyor.

@Service
class OrderService {

    @Transactional
    public void placeOrder(OrderRequest req) {
        orderRepo.save(req.toOrder());
        this.updateInventory(req);  // YANLIŞ — this = gerçek nesne, proxy değil!
        // @Transactional burada ÇALIŞMAZ — yeni transaction açılmaz
        // Tüm işlem aynı transaction'da mı? Hayır — updateInventory'nin
        // kendi @Transactional'ı IGNORE edildi.
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateInventory(OrderRequest req) {
        // Bu annotation hiçbir zaman tetiklenmedi!
        inventoryRepo.update(req);
    }
}

// Belirti: updateInventory'de hata olsa bile placeOrder rollback olmuyor.
// Debug: transaction log yok, sanki annotation yokmuş gibi.

// DÜZELTME 1: Başka bir bean üzerinden çağır
@Service
class OrderService {
    @Autowired InventoryService inventoryService;  // proxy inject edildi

    @Transactional
    public void placeOrder(OrderRequest req) {
        orderRepo.save(req.toOrder());
        inventoryService.updateInventory(req);  // proxy üzerinden → AOP çalışır
    }
}

// DÜZELTME 2: ApplicationContext.getBean() (çirkin ama bazen gerekli)
@Autowired ApplicationContext ctx;

void placeOrder(OrderRequest req) {
    orderRepo.save(req.toOrder());
    ctx.getBean(OrderService.class).updateInventory(req);  // self proxy
}

// DÜZELTME 3: @EnableAspectJAutoProxy(exposeProxy=true) + AopContext
@Transactional
void placeOrder(OrderRequest req) {
    orderRepo.save(req.toOrder());
    ((OrderService) AopContext.currentProxy()).updateInventory(req);
}
```

---

**Sorun 5: @Async çalışmıyor — aynı thread'de çalışıyor**

```java
// Senaryo: Email gönderimi async yapıldı ama HTTP response yine de geç dönüyor.

@Service
class NotificationService {

    // Self-invocation ile çağrılıyor:
    public void processOrder(Order order) {
        orderRepo.save(order);
        this.sendEmail(order);   // YANLIŞ — proxy bypass → @Async yok sayıldı
    }

    @Async
    public void sendEmail(Order order) {
        // Uzun süren email gönderimi
    }
}

// Ayrıca: @EnableAsync anotasyonu @Configuration sınıfında yoksa @Async hiç çalışmaz
// — hata da fırlatmaz, sadece sync çalışır.

// DÜZELTME:
@SpringBootApplication
@EnableAsync  // ← bu olmazsa @Async hiç tetiklenmez
class App {}

@Service
class OrderService {
    @Autowired NotificationService notificationService;  // dışarıdan inject

    void processOrder(Order order) {
        orderRepo.save(order);
        notificationService.sendEmail(order);  // proxy → @Async tetiklenir
    }
}

// @Async exception yutma tuzağı:
// Async metod CompletableFuture döndürmüyorsa exception kaybolur!
@Async
public CompletableFuture<Void> sendEmail(Order order) {
    // Exception burada yakalanabilir
    try {
        emailClient.send(order);
        return CompletableFuture.completedFuture(null);
    } catch (Exception e) {
        log.error("Email failed", e);
        return CompletableFuture.failedFuture(e);
    }
}
```

---

### Mülakat Soruları — Spring AOP

**Junior / Mid:**

6. Spring AOP ile AspectJ farkı nedir?

   > **Beklenen:** Spring AOP: proxy tabanlı, runtime weaving. Sadece Spring bean'lerine uygulanır, method-level join point. JDK proxy (interface varsa) veya CGLIB (concrete class). AspectJ: compile-time veya load-time weaving, bytecode manipülasyonu. Her Java sınıfına uygulanabilir, field/constructor/static method join point destekler. Spring: @Around yetersiz kaldığında AspectJ kullanabilirsin (load-time weaving). Pratik: Spring'in 99% use case'i için Spring AOP yeterlidir.

7. `@Around` advice nedir? Diğer advice türlerinden farkı?

   > **Beklenen:** @Before: metoddan önce çalışır, durduramaz (void). @After: her durumda — return veya exception sonrası. @AfterReturning: sadece başarılı return'de, dönüş değerine erişim. @AfterThrowing: sadece exception'da. @Around: metodu tamamen sarar — pjp.proceed() ile devam et veya etme, dönüş değerini değiştirebilir, exception handle edebilir. Kullanım: timing, caching, transaction gibi "önce-yap / sonra-yap" gerektiren şeyler. Gücü: pjp.proceed() çağrılmazsa metod hiç çalışmaz.

---

**Senior / Architect:**

8. Bir metodun hem `@Transactional` hem `@Cacheable` hem `@PreAuthorize` annotation'ı var. Bunlar hangi sırada çalışır?

   > **Beklenen:** Hepsi AOP proxy üzerinden — sıra @Order veya default önceliğe göre. Spring Security: en dışta (çalışmazsa içeriye girilmez). Cache: transaction dışında → cache hit olursa DB'ye hiç gidilmez. Transaction: cache miss durumunda açılır. Sıra: @PreAuthorize → @Cacheable → @Transactional → gerçek metod. Bu önemlidir: @Cacheable transaction başlamadan cache'e bakar → transaction rollback olsa bile cache kirlenebilir. Çözüm: cache TTL kısa tut veya transactional event listener ile invalidate et.

9. `final` class'lara Spring AOP neden uygulanamaz? Alternatif nedir?

   > **Beklenen:** CGLIB proxy: hedef sınıfın subclass'ını üretir. final sınıf subclass'lanamaz → CGLIB başarısız. JDK proxy: interface tabanlı → interface varsa final sınıf için çalışır (çünkü interface'i implement eden yeni sınıf üretir, final sınıfı extend etmez). Alternatif: AspectJ load-time weaving — bytecode'u class loading sırasında değiştirir, subclass gerekmez. Spring için @EnableLoadTimeWeaving + AspectJ agent. Pratik: Kotlin data class'lar default final → Spring: `kotlin-spring` plugin açık class'a dönüştürür.

---

## Bölüm 3: Spring Event Sistemi

### Gerçek Hayat Sorunları

---

**Sorun 6: @EventListener sync — exception publisher'ı etkiledi**

```java
// Senaryo: Sipariş kaydedildi → analitik event fırlatıldı → analytics servisi down.
// Sipariş de kaydedilmedi (rollback oldu). Müşteri şikayeti.

@Service
class OrderService {
    @Transactional
    void placeOrder(Order order) {
        orderRepo.save(order);                                // DB'ye yazıldı
        publisher.publishEvent(new OrderPlacedEvent(order)); // ← burası throw etti
        // AnalyticsListener.onOrderPlaced() → HTTP call → timeout → exception
        // @Transactional: exception geldi → ROLLBACK
        // Sonuç: Sipariş kaydedilmedi ama müşteri "sipariş verildi" sayfasını gördü.
    }
}

@Component
class AnalyticsListener {
    @EventListener  // SYNC — publisher'ın transaction'ı içinde çalışıyor
    void onOrderPlaced(OrderPlacedEvent event) {
        analyticsClient.track(event.order()); // exception fırlatabilir!
    }
}

// DÜZELTME 1: @TransactionalEventListener — commit sonrası çalış
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void onOrderPlaced(OrderPlacedEvent event) {
    try {
        analyticsClient.track(event.order());
    } catch (Exception e) {
        log.error("Analytics failed — non-critical", e);
        // exception yutulabilir: analytics critical değil
    }
}

// DÜZELTME 2: @Async ile ayrı thread
@EventListener
@Async
void onOrderPlacedAsync(OrderPlacedEvent event) {
    analyticsClient.track(event.order()); // exception publisher'a dönmez
}
// Dikkat: @Async içinde @Transactional yeni transaction başlatır — parent'tan bağımsız.
```

---

**Sorun 7: @TransactionalEventListener — AFTER_COMMIT'te entity lazy load**

```java
// Senaryo: Commit sonrası email göndermek istiyoruz.
// Email içinde sipariş kalemleri (items) lazily loaded.

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void sendEmail(OrderPlacedEvent event) {
    Order order = event.order();
    // Transaction kapandı → Persistence Context yok
    order.getItems().size(); // LazyInitializationException!
    // "no Session" → items yüklü değil
}

// DÜZELTME 1: Event'e eager yüklü veriyi koy
record OrderPlacedEvent(Order order, List<OrderItem> items) {}
// Publisher: items'ı transaction içinde yükle, event'e ekle.

// DÜZELTME 2: Event'te sadece ID, listener yeni transaction açar
record OrderPlacedEvent(Long orderId) {}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)  // yeni transaction
void sendEmail(OrderPlacedEvent event) {
    Order order = orderRepo.findByIdWithItems(event.orderId()); // eager fetch
    emailService.send(order);
}

// DÜZELTME 3: DTO kullan — entity yerine serializable veri
record OrderPlacedEvent(OrderDTO dto) {}  // tüm veriler hazır, session yok
```

---

### Mülakat Soruları — Event Sistemi

**Junior / Mid:**

10. `@EventListener` ile `@TransactionalEventListener` farkı nedir?

    > **Beklenen:** @EventListener: event publish edildiği anda, sync, publisher'ın transaction'ı içinde çalışır. Exception → publisher'ı etkiler (rollback). @TransactionalEventListener: transaction'ın belirli aşamasında çalışır. AFTER_COMMIT (default): commit başarılıysa → güvenli. AFTER_ROLLBACK: rollback sonrası temizlik. BEFORE_COMMIT: commit öncesi son kontrol. Kullanım: email/bildirim → AFTER_COMMIT (commit garanti → gönder). Kritik iş adımı → @EventListener (transaction içinde, fail → rollback).

11. Spring Event'i Kafka/RabbitMQ ile nasıl karşılaştırırsın?

    > **Beklenen:** Spring Event: in-process, aynı JVM, kayıp riski var (JVM çöktü → event gitti), servis sınırlarını geçemez. Kafka/RabbitMQ: process'ler arası, kalıcı (at-least-once delivery), servisler arası. Kullanım: Spring Event → modüler monolith içi decoupling, aynı servis içi yan etki (email, analitik). Kafka/RabbitMQ → microservisler arası, kritik işlemler, replay gerektiğinde.

---

## Bölüm 4: Spring MVC

### Gerçek Hayat Sorunları

---

**Sorun 8: @ControllerAdvice exception handler sırası yanlış — genel handler özel'i eziyor**

```java
// Sorun: Her exception için genel 500 dönüyor, özel handler devreye girmiyor.

@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)   // ← en genel, üstte
    ResponseEntity<ErrorResponse> handleAll(Exception ex) {
        return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR"));
    }

    @ExceptionHandler(OrderNotFoundException.class)  // ← daha özel, altta
    ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("ORDER_NOT_FOUND"));
        // Bu hiç çalışmıyor — Exception handler her şeyi yakaladı
    }
}

// DÜZELTME: Özel'den genele doğru sıra — ya @Order ile veya ayrı class
@ControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)  // önce dene
class DomainExceptionHandler {
    @ExceptionHandler(OrderNotFoundException.class)
    ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) { ... }

    @ExceptionHandler(ValidationException.class)
    ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) { ... }
}

@ControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)   // son çare
class FallbackExceptionHandler {
    @ExceptionHandler(Exception.class)
    ResponseEntity<ErrorResponse> handleAll(Exception ex) { ... }
}
```

---

**Sorun 9: Filter vs Interceptor yanlış tercih — Spring context'e erişilemiyor**

```
Senaryo:
  JWT token doğrulaması için Servlet Filter yazdık.
  Filter içinde UserDetailsService'e @Autowired yaptık — null geldi.

Sorun:
  Servlet Filter: Servlet container yönetir, Spring context'i varsayılan olarak inject etmez.
  @Autowired Filter içinde null → NullPointerException.

Neden:
  Filter, Spring'in IoC container'ından önce başlatılır.
  Spring @Bean olarak tanımlanmış Filter: Spring yönetir → inject çalışır.
  @WebFilter + @ServletComponentScan ile tanımlanan Filter: Servlet yönetir → inject yok.

Çözümler:
  1. Filter'ı Spring @Bean olarak tanımla:
     @Bean FilterRegistrationBean<JwtFilter> jwtFilter() {
         FilterRegistrationBean<JwtFilter> reg = new FilterRegistrationBean<>(new JwtFilter(userDetailsService));
         reg.addUrlPatterns("/api/*");
         return reg;
     }
     
  2. OncePerRequestFilter extend et: Spring yönetimli, inject çalışır.
     @Component class JwtAuthFilter extends OncePerRequestFilter { }

  3. Auth/JWT için HandlerInterceptor kullan — Spring context var.
     preHandle() içinde SecurityContextHolder'ı doldur.
```

---

### Mülakat Soruları — Spring MVC

**Junior / Mid:**

12. `DispatcherServlet` ne işe yarar? Request nasıl Controller'a ulaşır?

    > **Beklenen:** DispatcherServlet: Spring MVC'nin Front Controller'ı. Tüm request'ler önce buraya gelir. Adımlar: (1) HandlerMapping ile URL + method → hangi Controller.method? (2) HandlerInterceptor.preHandle() → false ise dur. (3) HandlerAdapter.handle() → argument resolver'lar (@RequestBody, @PathVariable). (4) Controller.method() çalışır. (5) ReturnValueHandler → @ResponseBody ise Jackson serialize. (6) HandlerInterceptor.postHandle() + afterCompletion(). Tek giriş noktası → cross-cutting concern merkezi.

13. `Filter` ve `HandlerInterceptor` arasındaki farkı ne zaman önemlidir?

    > **Beklenen:** Filter: Servlet spec, Spring context yok (dikkat — @Bean olarak tanımlanırsa var). Her request'te çalışır, static resource dahil. Request/response byte seviyesinde değiştirilebilir. Kullanım: CORS header, gzip, request logging, encoding. HandlerInterceptor: Spring MVC, Spring context mevcut, sadece @Controller'lara eşleşen request'lerde. preHandle: false → controller çalışmaz. postHandle: ModelAndView erişim. afterCompletion: response yazıldı, view render edildi. Kullanım: auth, audit log, locale/timezone ayarlama.

---

**Senior / Architect:**

14. Spring MVC'de hata yanıtı standartlaştırma nasıl yapılır? `ProblemDetail` nedir?

    > **Beklenen:** @ControllerAdvice + @ExceptionHandler: merkezi hata handler. Tüm exception tiplerine özel response. Spring 6 / Boot 3: RFC 7807 Problem Details standartı → `ProblemDetail` sınıfı. `Content-Type: application/problem+json`. Alan: type, title, status, detail, instance. Avantaj: tüm API'ler aynı hata formatında → client tek format işler. Konfigürasyon: `spring.mvc.problemdetails.enabled=true`. Özel hata alanı: `ProblemDetail.setProperty("traceId", MDC.get("traceId"))` → client hataları loglarda bulabilir.

---

## Bölüm 5: Spring WebFlux

### Gerçek Hayat Sorunları

---

**Sorun 10: Blocking çağrı event loop thread'ini bloke etti**

```java
// Senaryo: WebFlux + R2DBC kullanıyoruz ama bir servis hâlâ JDBC ile çalışıyor.
// Sistem birkaç yüz request sonra yanıt vermez oldu.

@GetMapping("/products/{id}")
Mono<Product> getProduct(@PathVariable Long id) {
    // YANLIŞ — blocking call event loop thread'inde
    Product product = legacyJdbcRepo.findById(id);  // Thread bloke!
    return Mono.just(product);
    // Event loop: küçük thread havuzu (CPU core sayısı).
    // 8 thread × 1sn JDBC wait → 8 eş zamanlı request sonra sistem durur.
}

// Belirti:
// - İlk N request hızlı, sonra timeout
// - Thread dump: "reactor-http-nio" thread'leri WAITING/BLOCKED state
// - Metric: event.loop.blocked uyarısı

// DÜZELTME: boundedElastic scheduler
@GetMapping("/products/{id}")
Mono<Product> getProduct(@PathVariable Long id) {
    return Mono.fromCallable(() -> legacyJdbcRepo.findById(id))
               .subscribeOn(Schedulers.boundedElastic());
    // boundedElastic: blocking işler için ayrı thread havuzu (128 × CPU)
    // Event loop thread serbest → binlerce isteği işlemeye devam eder
}

// Uzun vadeli: legacyJdbcRepo → R2DBC'ye migrate et.
```

---

**Sorun 11: WebFlux + Spring Security — SecurityContext kayboldu**

```java
// Senaryo: Reactive endpoint'te mevcut kullanıcıyı almaya çalıştık.
// SecurityContextHolder.getContext() null döndü.

@GetMapping("/orders")
Mono<List<Order>> myOrders() {
    // YANLIŞ — ThreadLocal tabanlı, WebFlux'ta çalışmaz
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    // WebFlux: farklı operasyonlar farklı thread'lerde → ThreadLocal taşınmaz
    return orderRepo.findByUserId(auth.getName()); // NullPointerException!
}

// DOĞRU — ReactiveSecurityContextHolder
@GetMapping("/orders")
Mono<List<Order>> myOrders() {
    return ReactiveSecurityContextHolder.getContext()
        .map(ctx -> ctx.getAuthentication())
        .flatMap(auth -> orderRepo.findByUserId(auth.getName()));
}

// veya @AuthenticationPrincipal annotation:
@GetMapping("/orders")
Mono<List<Order>> myOrders(@AuthenticationPrincipal Mono<UserDetails> userMono) {
    return userMono.flatMap(user -> orderRepo.findByUserId(user.getUsername()));
}
```

---

### Mülakat Soruları — WebFlux

**Junior / Mid:**

15. `Mono` ve `Flux` nedir? `block()` ne zaman kullanılır?

    > **Beklenen:** Mono: 0 veya 1 element (Optional gibi ama async). Flux: 0..N element (List gibi ama async). Her ikisi de lazy — subscribe edilmeden çalışmaz. `block()`: Mono/Flux'ı sync bloke ederek değer alır. ASLA event loop thread'inde çağrılmaz → sistem durur. Ne zaman: test kodu, main metod, Spring MVC ile WebFlux karışık kullanım. Production reactive kod: block() görmek red flag. Alternatif: reactive chain devam eder, `.subscribe()` veya WebFlux controller Mono döndürür.

16. Spring MVC mi, WebFlux mi? Nasıl karar verirsin?

    > **Beklenen:** WebFlux seç: 10K+ eş zamanlı bağlantı, IO-bound (çok external API/DB çağrısı), streaming (SSE, WebSocket), Spring Cloud Gateway. MVC seç: ekip reactive'e yabancı, JPA/JDBC kullanımı gerekiyor (JPA reactive değil), CPU-bound iş (event loop avantaj sağlamaz), basit CRUD. Kural: blocking library (JPA, Hibernate) → MVC. Tüm stack reactive (R2DBC, WebClient) → WebFlux. Karışık: MVC + `Schedulers.boundedElastic()` geçici çözüm ama temiz değil.

---

**Senior / Architect:**

17. Reactive backpressure nedir? Pratikte neden önemlidir?

    > **Beklenen:** Backpressure: consumer üretimi kontrol eder — "şu an işleyebileceğimden fazla gönderme." Publisher hızlı, consumer yavaş → buffer doldu → OutOfMemory veya data loss. Reactive Streams spec: Subscriber.request(n) → "şimdi n tane verebilirsin." Örnek: Kafka → Flux<Message> → 1000 mesaj/sn üretiyor, DB yazma 100 mesaj/sn → buffer → OOM. Çözüm: `limitRate(100)`, `onBackpressureDrop`, `onBackpressureBuffer(maxSize)`. Pratik: WebFlux + database bağlantıları → bağlantı havuzu doldu → backpressure olmadan timeout storm.

---

## Bölüm 6: Spring WebSocket

### Gerçek Hayat Sorunları

---

**Sorun 12: Multi-pod deployment — WebSocket mesajı yanlış pod'a gitti**

```
Senaryo:
  Chat uygulaması: 2 pod. Kullanıcı A → Pod 1'e bağlı.
  Kullanıcı B, A'ya mesaj gönderdi → mesaj Pod 2'ye geldi.
  Pod 2: A'yı bilmiyor (farklı in-memory broker) → mesaj kayboldu.

Sorun: In-memory SimpleBroker → pod-local → diğer pod'ları bilmiyor.

Düzeltme: Shared STOMP Broker (RabbitMQ veya ActiveMQ)
  config.enableStompBrokerRelay("/topic", "/queue")
      .setRelayHost("rabbitmq")
      .setRelayPort(61613);
  
  Akış:
    Pod 2 mesaj alır → RabbitMQ STOMP relay'e iletir
    RabbitMQ → tüm subscriber'lara (Pod 1 dahil) dağıtır
    Pod 1 → A'ya iletir

Alternatif: Redis Pub/Sub
  Pod 2 mesaj → Redis channel'a publish
  Pod 1 aynı channel subscribe → alır → A'ya iletir
  (Spring'in built-in STOMP relay değil, custom implementasyon gerekir)

Not: Load balancer → sticky session (IP hash) → A her zaman Pod 1'e bağlı.
  Basit ama pod restart → bağlantı kopuyor.
```

---

### Mülakat Soruları — WebSocket

**Junior / Mid:**

18. WebSocket ve HTTP polling arasındaki fark nedir? WebSocket ne zaman tercih edilir?

    > **Beklenen:** HTTP polling: client her N saniyede GET isteği → çoğu response boş → sunucu + network yük, gecikme yüksek. Long polling: server gelene kadar bekler → daha iyi ama bağlantı sayısı artar. SSE: server→client tek yönlü push, basit. WebSocket: kalıcı TCP bağlantısı, çift yönlü, sub-millisecond gecikme. Tercih: gerçek zamanlı chat, oyun, canlı fiyat → WebSocket. Sunucu→client bildirim, az güncelleme → SSE yeterli. Gerektiğinde güncellenecek veriler → polling.

---

**Senior / Architect:**

19. 100K eş zamanlı WebSocket bağlantısı nasıl yönetilir?

    > **Beklenen:** (1) WebFlux reactive WebSocket: thread-per-connection değil, event loop → az thread, çok bağlantı. (2) Horizontal scale: shared broker (RabbitMQ STOMP cluster). (3) Connection pool: bağlantı başına durum küçük tutulur (sadece userId, subscriptions). (4) Load balancer: sticky session veya bağlantısız tasarım (shared broker aracı). (5) Heartbeat + reconnect: client-side exponential backoff ile yeniden bağlanma. (6) K8s: pod affinity değil, her pod tüm kullanıcılara hizmet edebilir (shared broker sayesinde). Monitoring: active connection count, message throughput, memory per connection.

---

## Bölüm 7: Spring Data JPA

### Gerçek Hayat Sorunları

---

**Sorun 13: LazyInitializationException — transaction dışında lazy load**

```java
// Senaryo: Sipariş listesini endpoint'ten döndürünce bazı alanlar null.
// Detaya bakılınca: "could not initialize proxy - no Session"

@Service
class OrderService {
    @Transactional
    public List<Order> getOrders() {
        return orderRepo.findAll();  // Order yüklendi
        // items: LAZY → henüz yüklenmedi (proxy)
    }                                // Transaction kapandı — Session yok
}

@RestController
class OrderController {
    void getOrders() {
        List<Order> orders = orderService.getOrders();
        orders.get(0).getItems().size(); // ← LazyInitializationException!
        // Jackson serialize etmeye çalışırken da olabilir
    }
}

// Çözüm 1: Eager fetch (dikkat: her zaman yükler — gereksiz)
@OneToMany(fetch = FetchType.EAGER)  // performans riski

// Çözüm 2: JOIN FETCH ile sadece gerektiğinde yükle
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// Çözüm 3: EntityGraph
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findAll();

// Çözüm 4: DTO projection — entity serialize etme, data transfer objesi dön
@Query("SELECT new com.example.OrderSummary(o.id, o.total, o.status) FROM Order o")
List<OrderSummary> findAllSummaries();  // LazyInit sorunu yok — field'lar String/Long

// Çözüm 5 (yanlış): open-session-in-view=true (default=true)
// Request bitene kadar session açık → N+1 sorunu gizlenir → production felaket
// KAPAT: spring.jpa.open-in-view=false
```

---

**Sorun 14: Dirty checking sürprizi — istemeden UPDATE üretildi**

```java
// Senaryo: Siparişi okuyup fiyat hesaplıyoruz ama DB'ye yazmıyoruz.
// Ama log'larda beklenmedik UPDATE sorguları görüldü.

@Transactional
public BigDecimal calculateDiscount(Long orderId) {
    Order order = orderRepo.findById(orderId).get();  // Managed entity
    order.setDiscountRate(0.1);  // Hesaplama için set ettik ama save() çağırmadık
    // ...
    BigDecimal discount = order.getTotal().multiply(BigDecimal.valueOf(order.getDiscountRate()));
    return discount;  // Transaction kapanır → dirty checking → UPDATE fırlatır!
}
// Sonuç: discountRate gereksiz yere DB'ye yazıldı.

// Düzeltme 1: @Transactional(readOnly = true) — dirty checking devre dışı
@Transactional(readOnly = true)
public BigDecimal calculateDiscount(Long orderId) {
    Order order = orderRepo.findById(orderId).get();
    order.setDiscountRate(0.1); // artık flush edilmez
    return order.getTotal().multiply(BigDecimal.valueOf(order.getDiscountRate()));
}

// Düzeltme 2: Entity'yi değiştirme — yerel değişken kullan
@Transactional(readOnly = true)
public BigDecimal calculateDiscount(Long orderId) {
    Order order = orderRepo.findById(orderId).get();
    double tempRate = 0.1;  // entity'ye dokunma
    return order.getTotal().multiply(BigDecimal.valueOf(tempRate));
}
```

---

### Mülakat Soruları — Spring Data JPA

**Junior / Mid:**

20. N+1 problemi nedir? Nasıl tespit edilir ve çözülür?

    > **Beklenen:** N+1: 1 sorgu ile N kayıt çekilir, her kayıt için 1 daha sorgu → N+1 toplam. Örnek: 100 sipariş → her siparişin müşterisi için ayrı SELECT → 101 sorgu. Tespit: (1) `spring.jpa.show-sql=true` ile sorgu sayısı, (2) Hibernate Statistics, (3) datasource-proxy + P6Spy, (4) Micrometer JDBC metrics. Çözüm: JOIN FETCH, @EntityGraph (seçici eager), @BatchSize (N+1'i N/batch+1'e düşürür). DTO projection: gerekli alanları tek sorguda çek.

21. `@Transactional(readOnly = true)` ne sağlar?

    > **Beklenen:** (1) Dirty checking kapalı → entity değişse bile flush edilmez → UPDATE üretilmez. (2) DB'de read-only transaction hint → bazı DB'ler (PostgreSQL) read replica'ya yönlendirebilir. (3) 1. seviye cache (persistence context) optimizasyonu. (4) Hibernate: entity immutable snapshot almaz → küçük bellek tasarrufu. Kullanım: GET endpoint, rapor sorguları, veri okuma servisleri. Yanlış algı: readOnly = faster değil, ama gereksiz write önler.

---

**Senior / Architect:**

22. Spring Data JPA yüksek yazma throughput'u için uygun mu? Alternatifleri nelerdir?

    > **Beklenen:** JPA yüksek yazma için zorlanır: (1) Dirty checking overhead: her entity snapshot karşılaştırması. (2) Persistence context belleği: çok entity → OOM. (3) Batch insert: `saveAll()` optimize değilse N ayrı INSERT. Iyileştirmeler: `@BatchSize`, `hibernate.jdbc.batch_size`, `@Modifying @Query` ile bulk update. Alternatifler: JOOQ (SQL type-safe, hibernate overhead yok), Spring JDBC (saf JDBC, çok daha hızlı batch), MyBatis, native SQL. Karar: <1000 kayıt/sn → JPA yeterli. Yüksek throughput → JDBC/JOOQ.

---

## Bölüm 8: Spring Data Redis

### Gerçek Hayat Sorunları

---

**Sorun 15: Default serializer — binary key, debug edilemiyor**

```java
// Sorun: Redis'e yazdığımız key'ler redis-cli'den okunemiyor.
// redis-cli: keys * → "\xac\xed\x00\x05t\x00\border:123"

// Neden: Default JdkSerializationRedisSerializer → Java serialization binary format
RedisTemplate<String, Object> template;
// template.opsForValue().set("order:123", order);
// Key: binary garbage → redis-cli, monitoring tool, başka uygulama okuyamaz

// DÜZELTME: Jackson serializer + string key
@Configuration
class RedisConfig {
    @Bean
    RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key: String
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value: JSON
        ObjectMapper om = new ObjectMapper()
            .activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
                                   ObjectMapper.DefaultTyping.NON_FINAL);
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer(om));
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer(om));

        return template;
    }
}
// Sonuç: redis-cli → GET order:123 → {"id":123,"status":"PENDING",...}
```

---

**Sorun 16: TTL unutuldu — Redis bellek doldu**

```
Senaryo:
  Kullanıcı profili cache'lendi → TTL yok.
  1 yıl sonra: Redis 32GB → bellek dolu → OOM → Redis crash.
  Uygulama: tüm Redis operasyonları fail → downtime.

Belirti: "OOM command not allowed when used memory > maxmemory"

Düzeltme:
  1. Her cache key'ine TTL belirle:
     redisTemplate.opsForValue().set("user:" + id, user, Duration.ofHours(1));
  
  2. maxmemory-policy: allkeys-lru (bellek dolarsa LRU key sil)
     redis.conf: maxmemory-policy allkeys-lru
     Uygulama cahce miss'e toleranslıysa: LRU güvenli.
  
  3. Key naming convention: "module:entity:id:version"
     Monitoring: keyspace notification → TTL olmayan key alert.
  
  4. @Cacheable ile Spring Cache → CacheManager TTL config:
     RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(30))
```

---

### Mülakat Soruları — Spring Data Redis

**Junior / Mid:**

23. `RedisTemplate` ile `StringRedisTemplate` farkı nedir?

    > **Beklenen:** StringRedisTemplate: `RedisTemplate<String, String>` — hem key hem value String, `StringRedisSerializer` default. En basit kullanım: counter, flag, basit string değerler. RedisTemplate: generic tür, serializer konfigürasyonu gerekir. Karmaşık nesne cache'lemek için. Kural: sadece string ise StringRedisTemplate, nesne cache'i için RedisTemplate + Jackson serializer.

24. Redis'te distributed lock nasıl yapılır? Sınırlamaları nelerdir?

    > **Beklenen:** `SET key value NX PX ttl` → atomik: "key yoksa set et, TTL koy." Başarılıysa lock alındı. Sınırlamalar: (1) Redis tek node → SPOF. (2) TTL dolmadan iş bitmedi → başka işlem lock alır, eski kilit sahibi farkında değil → conflict. (3) Redis master failover → slave henüz sync almadı → iki node lock sahibi olabilir. Çözüm: Redlock algoritması (N node'a yaz, majority'den ack al), Kubernetes: ShedLock (DB tabanlı), ZooKeeper (daha güçlü consistency).

---

**Senior / Architect:**

25. Redis cluster'da cross-slot işlem neden kısıtlıdır? Nasıl çözülür?

    > **Beklenen:** Redis Cluster: keyler hash slot'lara dağıtılır (16384 slot). Multi-key komut (MGET, MSET, transaction) — tüm key'ler aynı slot'ta olmalı. Farklı slot → "CROSSSLOT Keys in request don't hash to the same slot" hatası. Çözüm: Hash tag `{userId}:order:123` ve `{userId}:cart` → {} içi hash hesaplanır → aynı slot'a düşer. Pipeline: bağımsız key'leri ayrı komut olarak gönder, atomic gerekmiyorsa. Lua script: server-side atomic işlem. Tasarım: ilişkili key'leri aynı hash tag ile grupla.

---

## Bölüm 9: Spring Security

### Gerçek Hayat Sorunları

---

**Sorun 17: 401 ve 403 karıştırıldı — client yanlış davranış sergiledi**

```
Senaryo:
  Frontend: her 401'de login sayfasına yönlendir.
  Backend: yetkisiz kaynak erişiminde 401 döndürüyor.
  Kullanıcı giriş yapmış ama admin kaynağına erişmeye çalışıyor → 401 → login sayfası.
  Kullanıcı tekrar giriş yapıyor ama yine yönlendiriliyor → sonsuz döngü.

Doğru anlam:
  401 Unauthorized: "Kim olduğunu bilmiyorum — authenticate ol."
  403 Forbidden:    "Kim olduğunu biliyorum ama bu kaynağa erişim yok."

Spring Security default:
  Authenticate edilmemiş → 401
  Authenticate edilmiş ama yetkisiz → 403

Yaygın hata: Her ikisi için de 401 döndürmek.

Düzeltme Spring Security:
  .exceptionHandling(ex -> ex
      .authenticationEntryPoint((req, res, authEx) ->
          res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Authentication required"))
      .accessDeniedHandler((req, res, denyEx) ->
          res.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied"))
  )

Frontend:
  401 → login sayfasına yönlendir.
  403 → "Bu sayfaya erişim yetkiniz yok" mesajı göster, logout değil.
```

---

**Sorun 18: @PreAuthorize proxy bypass — method security çalışmıyor**

```java
// Senaryo: Admin endpoint'ini @PreAuthorize ile korumak istedik.
// Ama normal kullanıcılar da erişebildi.

@Service
class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllOrders() {
        orderRepo.deleteAll();
    }

    public void cleanup() {
        this.deleteAllOrders(); // YANLIŞ — self-invocation → proxy bypass → @PreAuthorize ÇALIŞMAZ
    }
}

// Başka sorun: @EnableMethodSecurity açılmadıysa @PreAuthorize hiç çalışmaz!
// Spring Boot 3+: @EnableMethodSecurity (prePostEnabled = true default)
// Spring Boot 2: @EnableGlobalMethodSecurity(prePostEnabled = true)

// DÜZELTME:
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // ← BU OLMADAN @PreAuthorize ÇALIŞMAZ
class SecurityConfig { }

// Self-invocation düzeltmesi:
@Service
class AdminService {
    @Autowired AdminService self;  // kendi proxy'si inject edildi

    public void cleanup() {
        self.deleteAllOrders();  // proxy üzerinden → @PreAuthorize çalışır
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllOrders() { ... }
}
```

---

### Mülakat Soruları — Spring Security

**Junior / Mid:**

26. Spring Security filter chain'ini özetle. En önemli filter'lar hangileri?

    > **Beklenen:** Her request sırayla tüm filter'lardan geçer. Kritikler: SecurityContextHolderFilter (SecurityContext yükle/temizle — request boyunca kim olduğu burada), CsrfFilter (token doğrulama), BearerTokenAuthenticationFilter (JWT parse + doğrulama), ExceptionTranslationFilter (401/403 üret), AuthorizationFilter (son kontrol — erişim var mı?). Sıra sabit ve önemli: auth önceden yapılır, authorization en sona.

27. `@PreAuthorize` ile `requestMatchers().hasRole()` farkı?

    > **Beklenen:** requestMatchers: URL tabanlı, SecurityFilterChain'de konfigüre edilir, HTTP method + URL pattern. Coarse-grained: "/admin/**" sadece ADMIN. @PreAuthorize: metod tabanlı, SpEL, ince taneli: kullanıcı kendi kaynağına erişebilir `#userId == authentication.principal.id`. İkisi birlikte: URL'de ilk filtre (public/auth ayrımı), metod'da fine-grained kontrol. @PreAuthorize @EnableMethodSecurity gerektirir.

---

**Senior / Architect:**

28. JWT token refresh stratejisi nasıl tasarlanır? Token çalınması nasıl önlenir?

    > **Beklenen:** Access token: kısa TTL (15dk), stateless. Refresh token: uzun TTL (7 gün), stateful (DB'de). Refresh token rotation: her kullanımda yeni refresh token → eski geçersiz. Token çalınma tespiti: aynı refresh token iki kez kullanıldı → her ikisini de iptal et → kullanıcıyı logout et. Güvenlik: refresh token HttpOnly cookie (JS erişemez), access token in-memory (localStorage tehlikeli). Logout: refresh token'ı DB'den sil (blacklist) → artık yeni access token üretemez. JTI (JWT ID) claim + Redis blacklist: logout sonrası kısa süreli access token'lar için.

29. Multi-tenant uygulamada Spring Security ile tenant izolasyonu nasıl sağlanır?

    > **Beklenen:** JWT claims'ine tenantId ekle. Filter: her request'te tenantId çıkart → ThreadLocal (veya SecurityContext'e custom principal). Repository katmanı: tüm sorgulara `WHERE tenant_id = :currentTenant` → Row-Level Security. @PreAuthorize: `authentication.principal.tenantId == #resource.tenantId`. Hibernate multi-tenancy: `CurrentTenantIdentifierResolver` → her request doğru şema/DB kullanır. Hata: filter'da tenantId çıkarılır ama JPA sorgusunda kullanılmazsa → tenant A'nın verisi tenant B'ye görünür. Audit: tenant bilgisi her log satırında bulunmalı.

---

## Karma — Architect Seviyesi

30. **"Spring'in AOP, transaction ve event sistemleri birbirine nasıl bağlıdır? Bir sipariş işlemini örnek alarak anlat."**

    > **Beklenen:** Adımlar: (1) `OrderController.placeOrder()` → HTTP request geldi. (2) Spring Security filter → JWT doğrula, SecurityContext'e yaz. (3) `@PreAuthorize("hasRole('USER')")` → AOP proxy kontrol. (4) `@Transactional` → AOP proxy transaction açar (propagation: REQUIRED). (5) `orderRepo.save()` → JPA → dirty checking başlar. (6) `publisher.publishEvent(new OrderPlacedEvent())` → listener'lar tetiklenir. (7) `@EventListener` sync → aynı transaction içinde çalışır. `@TransactionalEventListener(AFTER_COMMIT)` → queue'ya alınır. (8) Transaction commit → @TransactionalEventListener'lar çalışır → email gönder. (9) AOP proxy response'u döndürür. Hepsi AOP proxy katmanları — onion model.

31. **"Spring WebFlux + Spring Security + R2DBC kullanan bir serviste production'da memory leak var. Nasıl araştırırsın?"**

    > **Beklenen:** (1) Actuator metrics: `jvm.memory.used` artıyor mu? Hangi heap bölgesi (heap/non-heap)? (2) Thread dump: reactor-http-nio thread'leri ne yapıyor? BLOCKED/WAITING → blocking call tespiti. (3) Heap dump analiz (VisualVM/MAT): en çok yer kaplayan obje nedir? CompletableFuture/Mono → tamamlanmayan reactive chain? (4) Reactive context leak: `contextWrite()` ile context büyüyor mu? (5) R2DBC connection pool: connection'lar geri dönüyor mu? `r2dbc.pool.acquired` vs `r2dbc.pool.idle`. (6) Redis subscription: Flux.subscribe() çağrılmış ama dispose edilmemiş → leak. (7) Heap dump'ta `Disposable` veya `Subscription` birikmesi → subscribe edilip dispose edilmemiş akış.

32. **"Microservice'lerde Spring Security ile merkezi auth mi, her serviste auth mi? Hangi yaklaşımı seçersin?"**

    > **Beklenen:** Seçenekler: (1) Gateway merkezi: Gateway JWT doğrular → claims header'a ekler → servisler trust eder. Avantaj: tek doğrulama, servisler temiz. Risk: gateway bypass → korumasız. (2) Her serviste: defense in depth → gateway bypass edilse bile servis doğrular. Overhead: her servis JWKS endpoint'ini çeker, public key cache'ler. Önerilen hybrid: Gateway doğrular + claims header'a yazar. Servisler header'ı accept eder (full JWT parse etmez) AMA gateway'den geldiğini mTLS ile garanti eder. İç servise mTLS olmadan dışarıdan erişilemez → claims güvenilir. Servis mesh (Istio): mTLS otomatik, her servis müthiş izolasyon.
