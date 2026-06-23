# 02b — Spring Framework: Tüm Modüller

Her modül şu 5 soruya göre yapılandırılmıştır:
**Ne?** · **Neden?** · **Nasıl?** · **Ne zaman?** · **Trade-off?**

---

## 1. Spring Core — IoC Container

### Ne?
Uygulama bileşenlerini (bean) yaratan, bağımlılıklarını enjekte eden ve yaşam döngüsünü yöneten merkezi konteyner.

### Neden?
`new OrderService(new OrderRepository(new DataSource(...)))` zinciri kod içine gömüldüğünde test etmek, değiştirmek ve genişletmek zorlaşır. IoC (Inversion of Control) bu sorumluluğu framework'e devreder; sınıflar "nasıl yaratılacağını" değil "ne yapacağını" bilir.

### Nasıl?
```
1. @Configuration / @ComponentScan / XML okunur
2. BeanDefinition oluşturulur (class, scope, dependencies, init/destroy metod)
3. BeanDefinitionRegistry'ye kaydedilir
4. Dependency graph çözülür → topological sort
5. Bağımlılıklar önce instantiate edilir
6. Constructor / Field / Setter injection yapılır
7. BeanPostProcessor'lar devreye girer
8. @PostConstruct çağrılır
   ── Bean kullanıma hazır ──
9. Context kapanırken @PreDestroy çağrılır
```

**BeanFactory vs ApplicationContext:**
```
BeanFactory:
├── Lazy init (ilk getBean() çağrısında yükler)
└── Sadece temel DI

ApplicationContext (BeanFactory'yi genişletir):
├── Eager init (startup'ta tüm singleton bean'ler hazır)
├── MessageSource → i18n
├── ApplicationEventPublisher → event sistemi
├── ResourceLoader → classpath/dosya okuma
└── Environment & PropertySource → config yönetimi
```

**Bean Scope'ları:**
```java
@Scope("singleton")   // container başına 1 instance (varsayılan)
@Scope("prototype")   // her injection / getBean() çağrısında yeni instance
@Scope("request")     // HTTP request başına (web)
@Scope("session")     // HTTP session başına (web)
@Scope("application") // ServletContext başına (web)

// TUZAK: Prototype bean singleton'a inject edilirse
// Singleton bir kez yaratılır → prototype da bir kez inject edilir → hiç değişmez
// Çözüm:
@Autowired ObjectProvider<PrototypeBean> provider;
provider.getObject(); // her çağrıda yeni instance
```

**Circular Dependency:**
```java
// Constructor injection → Spring ÇÖZEMEZ → BeanCurrentlyInCreationException (doğru!)
// Field injection → proxy ile çözer (ama kötü tasarım sinyali)
// Gerçek çözüm: tasarımı düzelt, ortak bağımlılığı ayır veya event kullan
```

### Ne zaman?
Her Spring uygulamasında kullanılır — kaçınılmaz temel. `@Lazy` ile startup süresini kısaltmak istediğinde ya da `prototype` scope ile her çağrıda yeni nesne gerektiğinde özellikle dikkat edilmeli.

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Test edilebilirlik (mock inject edilebilir) | Reflection kullanımı → küçük startup maliyeti |
| Loose coupling | "Sihirli" injection — hata ayıklamak zor olabilir |
| Lifecycle yönetimi otomatik | Circular dependency tuzağı |
| Konfigürasyon merkezi | Annotation aşırı kullanımı karmaşıklık yaratır |

---

## 2. Spring AOP (Aspect Oriented Programming)

### Ne?
Logging, transaction, security, cache gibi "kesişen" (cross-cutting) konuları iş mantığından ayırıp merkezi bir yerde tanımlamayı sağlayan programlama modeli.

### Neden?
Her metoda `log.info("starting...")` ve `try/catch transaction` eklemek, aynı kodu yüzlerce yere kopyalamak anlamına gelir. AOP bu tekrarlı kodu tek bir yerde (Aspect) toplar; sınıflar sadece iş mantığını içerir.

### Nasıl?
**Temel kavramlar:**
```
Aspect    → Kesişen konuyu içeren sınıf (@Aspect)
JoinPoint → Aspect'in araya girebileceği an (genellikle method çağrısı)
Pointcut  → Hangi JoinPoint'lere uygulanacak (pattern/expression)
Advice    → Ne zaman ne yapılacak (@Before, @After, @Around...)
Weaving   → Aspect'i hedef kodla birleştirme
```

**Proxy mekanizması — Spring AOP'un kalbi:**
```
Sen inject edersin:   OrderService
Gerçekte gelir:       OrderService$$SpringCGLIB$$0  (proxy nesnesi)

Proxy.save(order):
  1. @Before advice çalışır
  2. Transaction açılır
  3. target.save(order) — gerçek kod
  4. @AfterReturning advice çalışır
  5. Transaction commit/rollback

İki proxy türü:
JDK Dynamic Proxy → sınıfın bir interface'i varsa kullanılır
CGLIB             → concrete class, bytecode manipülasyonu ile subclass üretir
Spring Boot varsayılanı: CGLIB (proxyTargetClass = true)
```

**Kritik kısıtlamalar:**
```java
// 1. final sınıf/metod → CGLIB subclass üretemez → AOP ÇALIŞMAZ
// 2. static metod → proxy intercept edemez → AOP ÇALIŞMAZ
// 3. Self-invocation → proxy bypass edilir → AOP ÇALIŞMAZ!

@Service
class OrderService {
    @Transactional
    public void save(Order o) { /* ... */ }

    public void process(Order o) {
        this.save(o); // YANLIŞ — proxy değil, this → transaction açılmaz!
        // Düzeltme: başka bir bean üzerinden çağır
    }
}
```

**Advice türleri:**
```java
@Aspect @Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    void logBefore(JoinPoint jp) {
        log.info("Calling: {}", jp.getSignature());
    }

    @AfterReturning(pointcut = "execution(* *.find*(..))", returning = "result")
    void logReturn(Object result) { log.info("Result: {}", result); }

    @AfterThrowing(throwing = "ex", pointcut = "within(com.example..*)")
    void logError(Exception ex) { log.error("Error: {}", ex.getMessage()); }

    @After("bean(orderService)")  // her durumda — return veya exception
    void logAfter() { }

    @Around("@annotation(Timed)")  // en güçlüsü — metodu tamamen sarar
    Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed(); // gerçek metodu çalıştır
        } finally {
            log.info("{}ms", System.currentTimeMillis() - start);
        }
    }
}
```

**Pointcut örnekleri:**
```java
execution(* com.example.service.*.*(..))  // paket + her sınıf + her metod + her arg
@annotation(Transactional)               // bu annotation olan her metod
within(com.example.service.*)            // bu paketteki her şey
args(Long, ..)                           // ilk arg Long olan metodlar
bean(orderService)                       // bu bean adındaki tüm metodlar

// Birleştirme
@Pointcut("within(com.example..*) && @annotation(Loggable)")
void loggable() {}
```

### Ne zaman?
- Logging, tracing, metrik toplama
- Transaction yönetimi (`@Transactional` arkada AOP'tur)
- Cache (`@Cacheable` arkada AOP'tur)
- Rate limiting, retry, circuit breaker
- Security (`@PreAuthorize` arkada AOP'tur)

### Ne zaman kullanma?
- Domain logic içinde — business kuralları AOP'a gömülmemeli
- Stateful işlemler — advice'lar genellikle stateless olmalı
- final/static metodlara — çalışmaz

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Cross-cutting concern'ler tek yerde | Akış takip etmek zorlaşır ("sihirli" davranış) |
| Sınıflar sadece iş mantığı içerir | Proxy kısıtlamaları (final, static, self-invocation) |
| Sonradan ekleme/çıkarma kolaylığı | Stack trace'lerde proxy sınıfları görünür |
| @Transactional / @Cacheable gibi powerful abstraction | Performans: proxy overhead (genellikle ihmal edilebilir) |

---

## 3. Spring Expression Language (SpEL)

### Ne?
Runtime'da Spring bean'leri, metod çağrıları, koşullar ve matematiksel ifadeleri değerlendiren ifade dili.

### Neden?
`@Value("${db.url}")` ile statik property inject edilir ama "bu bean'in şu alanını al ve 100 ile çarp" gibi dinamik ifadeler property dosyasıyla yapılamaz. SpEL bu boşluğu doldurur.

### Nasıl?
```java
// 1. Property inject (@Value)
@Value("${app.name}")                          // property dosyasından
@Value("#{T(Math).PI * 2}")                    // Java sınıfı + hesap
@Value("#{orderService.discountRate * 100}")   // başka bir bean'in alanı
@Value("#{systemProperties['user.home']}")     // system property
@Value("#{environment['spring.profiles.active']}")

// 2. Koşullu inject
@Value("#{${feature.enabled:false} ? 'v2' : 'v1'}")
String apiVersion;

// 3. Security'de — kimlik + rol kontrolü
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
void updateUser(@PathVariable Long userId) { }

// 4. Cache key üretimi
@Cacheable(value = "orders", key = "#status.name() + '_' + #customerId")
List<Order> findOrders(OrderStatus status, Long customerId) { }

// 5. Programatik kullanım
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext ctx = new StandardEvaluationContext(order);
Integer total = parser.parseExpression("items.size() * unitPrice")
                      .getValue(ctx, Integer.class);
```

### Ne zaman?
- `@Value` ile dinamik config değerleri
- `@PreAuthorize` / `@PostAuthorize` ile ince taneli güvenlik
- `@Cacheable` key ifadeleri
- `@ConditionalOnExpression` ile koşullu bean yaratımı

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Çok güçlü ve esnek | String içinde kod → IDE desteği zayıf |
| Compile-time bağımlılık yok | Runtime hatası → erken yakalanmaz |
| Security/cache için ideal | Karmaşık ifadeler okunaksız hale gelir |

---

## 4. Spring Context — Event Sistemi

### Ne?
Uygulama içi Observer pattern implementasyonu. Bir bileşen event yayınlar, ilgili bileşenler dinler — publisher ve listener birbirini tanımaz.

### Neden?
`OrderService` sipariş kaydedince email göndermek, stok düşmek ve analitik kayıt tutmak istiyorsun. Hepsini doğrudan çağırsan sıkı bağımlılık oluşur; OrderService'in 5 servisi bilmesi gerekir. Event ile OrderService sadece "sipariş verildi" der, geri kalanı listener'lar halleder.

### Nasıl?
```java
// 1. Event tanımla
record OrderPlacedEvent(Order order) {}  // POJO yeterli (Spring 6+)

// 2. Event yayınla
@Service
class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    void placeOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order));
        // Varsayılan: SYNC — listener'lar bu satırda, aynı thread'de çalışır
    }
}

// 3. Event dinle
@Component
class EmailListener {

    // Senkron listener — aynı transaction içinde
    @EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.order());
    }

    // Asenkron listener — ayrı thread (DB'ye bağımlı değilse ideal)
    @EventListener
    @Async
    void onOrderPlacedAsync(OrderPlacedEvent event) {
        analyticsService.track(event.order());
    }

    // Transaction COMMIT OLDUKTAN SONRA çalış (en güvenli)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void afterCommit(OrderPlacedEvent event) {
        // DB'ye yazıldığı garanti — şimdi email gönder
        emailService.sendConfirmation(event.order());
        // ROLLBACK olsaydı bu hiç çalışmayacaktı
    }
}
```

**Yerleşik Spring event'leri:**
```
ContextRefreshedEvent   → ApplicationContext tamamen yüklendi
ContextClosedEvent      → Context kapatılıyor
ApplicationStartedEvent → Spring Boot app başladı
ApplicationReadyEvent   → Tamamen hazır, istek kabul edebilir (buraya hook at)
ApplicationFailedEvent  → Startup başarısız
```

### Ne zaman?
- Birbirinden bağımsız olması gereken bileşenler arası iletişim
- Bir işlem sonrası yan etkiler (email, bildirim, analitik)
- `@TransactionalEventListener` ile "commit sonrası kesin çalış" garantisi
- Modüler monolith içi decoupling

### Ne zaman kullanma?
- Kritik iş akışı adımları — event kaybedilebilir, fallback yok
- Servisler arası iletişim — bunun için Kafka/RabbitMQ kullan

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Publisher listener'ı bilmez (loose coupling) | Akışı takip etmek zorlaşır |
| Yeni listener eklemek mevcut kodu değiştirmez | Sync listener'da exception → publisher etkilenir |
| `@Async` ile non-blocking | Async listener'da transaction yok — dikkat |

---

## 5. Spring MVC

### Ne?
HTTP request'leri Controller sınıflarına yönlendiren, argümanları çözümleyen, dönüş değerlerini serialize eden web çerçevesi. Temeli `DispatcherServlet`'tir.

### Neden?
HTTP'nin ham request/response API'si (HttpServletRequest, HttpServletResponse) ile çalışmak verbozdur. Spring MVC `@GetMapping`, `@RequestBody`, `@ResponseBody` gibi annotation'larla bu karmaşıklığı gizler; sınıflar POJO olarak kalır.

### Nasıl?
```
HTTP Request → Tomcat/Jetty → DispatcherServlet
                                      │
                          HandlerMapping.getHandler()
                          → RequestMappingHandlerMapping
                            (URL + Method + @RequestMapping eşleştirme)
                                      │
                          HandlerInterceptor.preHandle()
                          (false dönerse → dur, response yaz)
                                      │
                          HandlerAdapter.handle()
                          → ArgumentResolver'lar çalışır:
                            @RequestBody  → Jackson deserialize
                            @PathVariable → URL segmentinden parse
                            @RequestParam → query string
                            @RequestHeader → HTTP header
                            @AuthenticationPrincipal → SecurityContext
                                      │
                          Controller.method() çalışır
                                      │
                          ReturnValueHandler:
                            @ResponseBody → Jackson serialize → JSON/XML
                            ModelAndView  → ViewResolver
                                      │
                          HandlerInterceptor.postHandle()
                          HandlerInterceptor.afterCompletion() ← hata olsa da çalışır
                                      │
                                 HTTP Response
```

**@ControllerAdvice — global exception handler:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorResponse handleNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors()
            .stream().map(e -> e.getField() + ": " + e.getDefaultMessage()).toList();
        return new ErrorResponse("VALIDATION_FAILED", errors);
    }
}
```

**Interceptor vs Filter:**
```
Filter (Servlet spec):       Her request, Spring context'i yok
  → CORS, request logging, auth token okuma

HandlerInterceptor (Spring): @Controller'dan önce/sonra, Spring context var
  → Yetkilendirme, audit log, locale değiştirme
  → preHandle(): false dönerse controller çalışmaz
  → postHandle(): model'e veri ekle
  → afterCompletion(): view render sonrası (kaynak temizle)
```

### Ne zaman?
Senkron, HTTP tabanlı REST API geliştirirken. JPA/JDBC gibi blocking kütüphaneler kullanıyorsan MVC doğru seçim.

### Ne zaman kullanma?
- 10K+ eş zamanlı bağlantı gerekiyorsa → WebFlux
- Streaming (SSE, WebSocket ağırlıklı) → WebFlux

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Olgun, geniş ekosistem | Thread-per-request → eş zamanlılık Tomcat thread sayısıyla sınırlı |
| JPA/JDBC ile sorunsuz çalışır | IO-bound yükte WebFlux'tan daha az verimli |
| Öğrenmesi kolay | Reactive paradigmaya geçiş gerektirmez (avantaj da, kısıt da) |

---

## 6. Spring WebFlux

### Ne?
Non-blocking, reactive programlama modeli üzerine kurulu web çerçevesi. Netty event loop üzerinde çalışır; `Mono<T>` ve `Flux<T>` ile asenkron veri akışı sağlar.

### Neden?
Spring MVC'de her request bir thread tutar. 200 thread → 200 eş zamanlı istek. Thread IO beklerken (DB, HTTP) boşta durur ama bellek tüketir. WebFlux thread'i bekletmek yerine callback ile devam eder → az thread, çok eş zamanlı bağlantı.

### Nasıl?
```
Thread-per-request (MVC):
Request → Thread A → DB sorgusu bekle... → cevap → Response
          (Thread A bloke, başka iş yapamıyor)

Event Loop (WebFlux):
Request → Event Loop Thread → DB sorgusu gönder → başka request işle
       ← DB cevabı geldi → callback → Response tamamla
(Aynı thread binlerce isteği yönetir)
```

```java
// Mono: 0 veya 1 eleman (nullable alternatifi)
// Flux: 0..N eleman (List alternatifi)

@RestController
class OrderController {

    @GetMapping("/orders/{id}")
    Mono<Order> getOrder(@PathVariable Long id) {
        return orderRepo.findById(id)           // Mono<Order>
            .switchIfEmpty(Mono.error(new OrderNotFoundException(id)));
    }

    @GetMapping("/orders")
    Flux<Order> listOrders() {
        return orderRepo.findAll();             // Flux<Order>
    }

    // Server-Sent Events — client bağlantıyı açık tutar, veriler akar
    @GetMapping(value = "/orders/live", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<Order> streamLiveOrders() {
        return orderRepo.findNewOrdersStream();
    }
}

// Zincir (pipeline) — subscribe edilene kadar HİÇBİR ŞEY ÇALIŞMAZ (lazy)
Mono<OrderDTO> result = orderRepo.findById(orderId)
    .filter(o -> o.getStatus() == ACTIVE)
    .flatMap(o -> paymentService.getPayment(o.getPaymentId())
        .map(p -> new OrderDTO(o, p)))
    .switchIfEmpty(Mono.error(new NotFoundException()));
```

**Blocking call tuzağı:**
```java
// YANLIŞ — event loop thread'i bloke eder → tüm sistem durur
Mono<Order> getOrder(Long id) {
    Order order = jdbcTemplate.queryForObject(...); // BLOCKING!
    return Mono.just(order);
}

// DOĞRU — blocking kodu ayrı thread pool'a kaçır
Mono<Order> getOrder(Long id) {
    return Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
               .subscribeOn(Schedulers.boundedElastic());
}
```

### Ne zaman?
- 10K+ eş zamanlı bağlantı
- IO-bound yük (DB, external API çağrıları ağır basıyor)
- Streaming (SSE, WebSocket)
- Spring Cloud Gateway (WebFlux üzerine kurulu)

### Ne zaman kullanma?
- Ekip reactive programlamaya yabancıysa — öğrenme eğrisi dik
- JPA/JDBC kullanıyorsan — bunların reactive sürümü yoktur (R2DBC gerekir)
- CPU-bound işler — event loop avantaj sağlamaz

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Az thread → çok eş zamanlı bağlantı | Öğrenmesi zor (reactive paradigma) |
| IO-bound yükte çok daha verimli | Debugging zor (stack trace kopuk) |
| Backpressure — alıcı hızına göre üretim | Blocking library ile karışınca felaket |
| Streaming yetenekleri | JPA/Hibernate ile doğrudan çalışmaz |

---

## 7. Spring WebSocket

### Ne?
HTTP'nin tek yönlü istek-cevap modelinin ötesinde, client ile server arasında kalıcı, çift yönlü bağlantı kuran modül. STOMP protokolü ile mesajlaşma desteği içerir.

### Neden?
Chat uygulaması, canlı bildirim, borsa fiyatları, oyun gibi durumlar için client sürekli polling yapar (her 2 saniyede GET) — hem sunucu yükü artar hem gecikme olur. WebSocket kalıcı bağlantı kurar; sunucu istediği zaman client'a veri iter.

### Nasıl?
```
HTTP Upgrade Handshake:
  Client → GET /ws HTTP/1.1
           Upgrade: websocket
           Connection: Upgrade
  Server → 101 Switching Protocols

Bağlantı kuruldu → Kalıcı TCP bağlantısı (bidirectional)

STOMP (mesajlaşma protokolü WebSocket üzerinde):
  CONNECT    → auth
  SUBSCRIBE  → /topic/orders kanalına abone ol
  SEND       → /app/orders.create mesaj gönder
  MESSAGE    → server'dan mesaj al
  DISCONNECT → bağlantıyı kes
```

```java
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // In-memory broker (geliştirme)
        config.enableSimpleBroker("/topic", "/queue");
        // Production: RabbitMQ STOMP (çok server instance için)
        // config.enableStompBrokerRelay("rabbitmq-host", 61613);
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .withSockJS(); // WebSocket desteklemiyorsa HTTP fallback
    }
}

@Controller
class NotificationController {

    @MessageMapping("/orders.create")   // client → /app/orders.create
    @SendTo("/topic/orders")            // → tüm subscriber'lara broadcast
    OrderEvent createOrder(CreateOrderRequest req) {
        return orderService.create(req);
    }

    // Belirli kullanıcıya
    @MessageMapping("/orders.status")
    @SendToUser("/queue/notifications")  // sadece mesajı gönderen kullanıcıya
    OrderStatus getStatus(StatusRequest req, Principal user) {
        return orderService.getStatus(req.orderId());
    }
}

// Controller dışından push (sunucu tarafı)
@Service
class PushNotificationService {
    @Autowired SimpMessagingTemplate messagingTemplate;

    void notifyOrderShipped(Order order) {
        messagingTemplate.convertAndSend("/topic/orders", new OrderShippedEvent(order));
        messagingTemplate.convertAndSendToUser(
            order.getCustomerId().toString(),
            "/queue/notifications",
            new PersonalNotification("Siparişin kargoya verildi!")
        );
    }
}
```

### Ne zaman?
- Gerçek zamanlı chat, canlı bildirim, kolaboratif araçlar
- Anlık fiyat/durum güncellemeleri
- Oyun, canlı dashboard

### Ne zaman kullanma?
- Tek yönlü sunucu→client push yeterliyse → Server-Sent Events (daha basit)
- Nadiren güncellenen veriler → polling veya cache yeterli

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Gerçek zamanlı, düşük gecikme | Kalıcı bağlantı → sunucu kaynak tüketimi |
| Çift yönlü iletişim | Load balancer'da sticky session veya shared broker gerekir |
| HTTP polling'e göre çok verimli | Firewall/proxy WebSocket'i kesebilir (SockJS fallback çözer) |

---

## 8. Spring Data JPA

### Ne?
JPA (Java Persistence API) üzerine kurulu repository soyutlaması. Interface tanımla, SQL yazma — Spring implementasyonu otomatik üretir.

### Neden?
`EntityManager` ile doğrudan çalışmak tekrarlı ve hata eğilimlidir. Spring Data JPA method isimlendirmesinden query üretir, pagination/sorting otomatik sağlar, transaction yönetimini entegre eder.

### Nasıl?
**Repository hiyerarşisi:**
```
Repository (marker)
  └── CrudRepository<T,ID>         → save, findById, findAll, delete, count
        └── PagingAndSortingRepository → findAll(Pageable), findAll(Sort)
              └── JpaRepository         → flush, saveAndFlush, deleteAllInBatch
                    └── JpaSpecificationExecutor → Criteria API desteği
```

**Query üretme yolları:**
```java
interface OrderRepository extends JpaRepository<Order, Long> {

    // 1. Method isminden türetme — Spring parse eder, JPQL üretir
    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);
    Optional<Order> findFirstByCustomerIdOrderByCreatedAtDesc(Long customerId);
    boolean existsByCustomerIdAndStatus(Long id, OrderStatus status);
    long countByStatus(OrderStatus status);

    // 2. JPQL — entity adı, kolon adı (tablo adı değil)
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.total > :min")
    List<Order> findHighValueOrdersWithItems(@Param("min") BigDecimal min);

    // 3. Native SQL — tablo ve kolon adları
    @Query(value = "SELECT * FROM orders WHERE YEAR(created_at) = :year",
           nativeQuery = true)
    List<Order> findByYear(@Param("year") int year);

    // 4. Batch güncelleme
    @Modifying @Transactional
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") OrderStatus s);

    // 5. Projection — sadece gerekli alanlar, tam entity yükleme
    @Query("SELECT o.id as id, o.total as total FROM Order o WHERE o.customerId = :id")
    List<OrderSummary> findSummaries(@Param("id") Long customerId);

    // 6. Sayfalama
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    // Kullanım: PageRequest.of(0, 20, Sort.by("createdAt").descending())
}
```

**Persistence Context & Dirty Checking:**
```
Persistence Context = 1. Seviye Cache (transaction scope)

Entity state'leri:
  Transient → new Order()        — context bilmiyor
  Managed   → findById() sonrası — context takip ediyor
  Detached  → transaction bitti  — context takip etmiyor
  Removed   → delete() sonrası  — commit'te silinecek

Dirty Checking:
  Transaction başında: entity snapshot alınır
  Flush/commit'te: snapshot vs current state karşılaştırılır
  Fark varsa: otomatik UPDATE üretilir — sen save() çağırmak zorunda değilsin!
```

**N+1 Problemi ve Çözümü:**
```java
// PROBLEM: 100 sipariş → her sipariş için ayrı customer sorgusu = 101 sorgu
List<Order> orders = orderRepo.findAll();
orders.forEach(o -> System.out.println(o.getCustomer().getName())); // N+1!

// ÇÖZÜM 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();

// ÇÖZÜM 2: EntityGraph
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findAll();

// ÇÖZÜM 3: @BatchSize — N+1'i N/batch_size+1'e düşürür
@OneToMany @BatchSize(size = 25)
List<OrderItem> items;
```

### Ne zaman?
RDBMS kullanan, CRUD ağırlıklı, orta karmaşıklıkta domain modeli olan Spring uygulamaları.

### Ne zaman kullanma?
- Çok karmaşık sorgular → native SQL veya JOOQ daha uygun
- Yüksek yazma throughput'u → JDBC/JOOQ daha hızlı
- Reactive stack → R2DBC kullan (JPA blocking'dir)

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Hızlı geliştirme, az boilerplate | N+1, lazy loading tuzakları |
| Pagination/sorting otomatik | Hibernate davranışını bilmeden kullanmak tehlikeli |
| Transaction entegrasyonu | Karmaşık sorgularda JPQL yetersiz kalır |
| Şema migration desteği (Flyway/Liquibase) | Büyük batch işlemler için uygun değil |

---

## 9. Spring Data Redis

### Ne?
Redis'e erişim için Spring soyutlaması. `RedisTemplate` ile düşük seviye erişim, `@RedisHash` ile ORM benzeri mapping.

### Neden?
Redis'in Java client'ı (Jedis, Lettuce) doğrudan kullanılabilir ama serialization, connection yönetimi, exception handling tekrarlı iştir. Spring Data Redis bunları standardize eder.

### Nasıl?
```java
// RedisTemplate: tüm Redis veri yapılarına erişim
@Service
class RedisService {
    @Autowired RedisTemplate<String, Object> redis;

    // String (cache, counter, lock)
    void cache(String key, Object val, Duration ttl) {
        redis.opsForValue().set(key, val, ttl);
    }
    Long increment(String key) { return redis.opsForValue().increment(key); }
    Boolean lock(String key, Duration ttl) {             // distributed lock
        return redis.opsForValue().setIfAbsent(key, "locked", ttl);
    }

    // Hash (nesne cache — kısmi güncelleme)
    void setUserField(Long userId, String field, Object val) {
        redis.opsForHash().put("user:" + userId, field, val);
    }
    Object getUserField(Long userId, String field) {
        return redis.opsForHash().get("user:" + userId, field);
    }

    // List (queue, recent items)
    void pushToQueue(String queue, Object item) { redis.opsForList().rightPush(queue, item); }
    Object popFromQueue(String queue) { return redis.opsForList().leftPop(queue); }

    // Set (unique membership)
    void addTag(String key, Object... vals) { redis.opsForSet().add(key, vals); }
    boolean isMember(String key, Object val) { return redis.opsForSet().isMember(key, val); }

    // Sorted Set (leaderboard, delayed queue)
    void addScore(String board, String player, double score) {
        redis.opsForZSet().add(board, player, score);
    }
    Set<Object> topN(String board, int n) {
        return redis.opsForZSet().reverseRange(board, 0, n - 1);
    }

    // Pipeline: N komutu tek round-trip (RTT tasarrufu)
    redis.executePipelined((RedisCallback<Object>) conn -> {
        for (var entry : map.entrySet())
            conn.set(entry.getKey().getBytes(), serialize(entry.getValue()));
        return null;
    });
}

// @RedisHash: entity'yi Redis'e hash olarak sakla
@RedisHash(value = "session", timeToLive = 1800)
class UserSession {
    @Id String id;
    Long userId;
    List<String> roles;
}
interface SessionRepository extends CrudRepository<UserSession, String> {}
```

### Ne zaman?
- Session cache, distributed cache
- Rate limiting (INCR + EXPIRE)
- Distributed lock (SET NX PX)
- Leaderboard (Sorted Set)
- Pub/Sub (lightweight)
- Geçici veri (TTL kritikse)

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Sub-millisecond latency | In-memory → RAM sınırı |
| Zengin veri yapıları | Persistence yapılandırması dikkat ister (RDB/AOF) |
| Atomic operasyonlar | Cluster'da cross-slot işlem kısıtları |
| TTL desteği | Strong consistency yok (eventual) |

---

## 10. Spring Data MongoDB

### Ne?
MongoDB ile çalışmak için Spring repository soyutlaması ve `MongoTemplate` API'si.

### Neden?
Esnek şema, hiyerarşik veri (iç içe objeler), ve yüksek yazma throughput'u gerektiren durumlar için JPA/RDBMS ağırdır. MongoDB bu senaryolar için daha uygun; Spring Data bunu Spring ekosistemiyle entegre eder.

### Nasıl?
```java
@Document(collection = "products")
class Product {
    @Id String id;             // MongoDB ObjectId
    String name;
    @Indexed(unique = true) String sku;
    List<String> tags;
    Map<String, Object> attributes; // dinamik alanlar
    List<Review> reviews;           // embed (join yok)
}

interface ProductRepository extends MongoRepository<Product, String> {
    List<Product> findByTagsContaining(String tag);
    List<Product> findByPriceGreaterThan(BigDecimal price);
}

// MongoTemplate: aggregation ve karmaşık sorgular
@Service
class ProductService {
    @Autowired MongoTemplate mongo;

    List<Product> search(String keyword, List<String> tags) {
        Query query = new Query()
            .addCriteria(Criteria.where("name").regex(keyword, "i")
                .and("tags").all(tags)
                .and("stock").gt(0))
            .with(Sort.by("popularity").descending())
            .limit(20);
        return mongo.find(query, Product.class);
    }

    // Atomic upsert (race condition yok)
    void incrementView(String id) {
        mongo.updateFirst(
            Query.query(Criteria.where("_id").is(id)),
            new Update().inc("viewCount", 1).set("lastViewedAt", Instant.now()),
            Product.class
        );
    }

    // Aggregation pipeline
    List<CategoryStats> statsPerCategory() {
        return mongo.aggregate(
            Aggregation.newAggregation(
                Aggregation.match(Criteria.where("status").is("ACTIVE")),
                Aggregation.group("category").count().as("total").avg("price").as("avgPrice"),
                Aggregation.sort(Sort.by("total").descending())
            ),
            "products", CategoryStats.class
        ).getMappedResults();
    }
}
```

### Ne zaman?
- Esnek/değişen şema (katalog, CMS, kullanıcı profili)
- Hiyerarşik veri (embed + join yoktur)
- Yüksek yazma throughput (log, event, IoT)
- Polimorfik veri modeli

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Esnek şema | ACID transaction sınırlı (4.0+ multi-doc transaction var ama ağır) |
| Horizontal sharding kolayca | JOIN yok — uygulama katmanında çözülür |
| Hiyerarşik veri doğal | Şema tutarlılığı uygulamaya bırakılmış |
| Yüksek yazma performansı | Aggregation karmaşık olabilir |

---

## 11. Spring Security

### Ne?
Kimlik doğrulama (authentication), yetkilendirme (authorization) ve yaygın güvenlik açıklarına karşı koruma sağlayan kapsamlı güvenlik çerçevesi.

### Neden?
Her uygulamada login, session yönetimi, CSRF koruması, CORS başlıkları, şifre hashleme ve JWT doğrulama sıfırdan yazmak hem zaman kaybı hem güvenlik riski. Spring Security bu altyapıyı hazır sunar ve yanlış yapılandırma zor olacak şekilde tasarlanmıştır.

### Nasıl?

**Filter Chain — her request bu sırayı geçer:**
```
HTTP Request
    ↓
DelegatingFilterProxy (Servlet → Spring köprüsü)
    ↓
FilterChainProxy
    ↓
SecurityFilterChain (sırayla her filter çalışır):
  1. SecurityContextHolderFilter  → SecurityContext yükle/temizle
  2. CorsFilter                   → CORS headers
  3. CsrfFilter                   → CSRF token doğrula
  4. LogoutFilter                 → /logout yakala
  5. UsernamePasswordAuthFilter   → form login
  6. BasicAuthenticationFilter    → HTTP Basic
  7. BearerTokenAuthFilter        → JWT (OAuth2 Resource Server)
  8. AnonymousAuthenticationFilter→ kimlik doğrulanmadıysa anonymous
  9. ExceptionTranslationFilter   → 401 / 403 üret
 10. AuthorizationFilter          → URL/method yetkilendirme
```

**Authentication akışı:**
```
Kullanıcı → username + password gönderir
    ↓
UsernamePasswordAuthenticationFilter
    ↓
AuthenticationManager.authenticate()
    ↓
DaoAuthenticationProvider
    ↓
UserDetailsService.loadUserByUsername()  ← sen implement edersin
    ↓
PasswordEncoder.matches(raw, hash)       ← BCrypt
    ↓
Authentication nesnesi → SecurityContextHolder
    ↓
JWT üretilir ve client'a dönülür
```

**Konfigürasyon (Spring Security 6):**
```java
@Configuration @EnableWebSecurity @EnableMethodSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // JWT + stateless → CSRF'e gerek yok
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // work factor: 2^12 iterasyon
    }
}

// Method-level security
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
void updateUser(Long userId, UserDTO dto) { }

@PostAuthorize("returnObject.ownerId == authentication.principal.id")
Order getOrder(Long orderId) { }  // dönen nesneye göre kontrol
```

### Ne zaman?
Her production Spring uygulamasında. Basit uygulamalarda bile default ayarlar CSRF, XSS başlıkları gibi temel güvenliği sağlar.

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Kapsamlı, battle-tested | Konfigürasyon karmaşıklığı — yanlış anlaşılması kolay |
| Filter chain esnek ve genişletilebilir | "Magic" çok — ne olduğunu anlamak zaman alır |
| OAuth2, JWT, SAML desteği | Debug güç: 401/403 neden geldi? (DEBUG log aç) |
| Method-level security (SpEL) | Her major sürümde API değişiyor (5→6 büyük kırılma) |
