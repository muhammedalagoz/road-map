# 01 — Temeller: Mülakat Soruları & Gerçek Hayat Sorunları

## Bölüm 1: Algoritma & Veri Yapıları

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 1: Production'da yavaş API — yanlış veri yapısı seçimi**

```
Senaryo:
  Her istek geldiğinde kullanıcı izinlerini kontrol eden bir endpoint var.
  İzin listesi bir ArrayList'te tutuluyor.
  contains() çağrısı her seferinde O(n) tarama yapıyor.
  Kullanıcı başına 50+ izin varsa → her API isteğinde onlarca O(n) çağrı.

Belirti:
  API 100ms altındayken birden 800ms'e çıkıyor.
  Profiler: izin kontrolünde zaman harcanıyor.

Düzeltme:
  ArrayList → HashSet (contains: O(n) → O(1))
  Set<String> permissions = new HashSet<>(user.getPermissions());
  permissions.contains("ORDER_READ"); // O(1)
```

---

**Sorun 2: Pagination olmadan büyük veri çekme (N+1 eşdeğeri)**

```
Senaryo:
  Admin paneli: "tüm kullanıcıları getir" → 2M kullanıcı heap'e yüklendi
  OutOfMemoryError: Java heap space

Belirti:
  Servis düzensiz aralıklarla OOM ile çöküyor.
  Sadece belirli endpoint kullanıldığında olıyor.

Düzeltme:
  Pageable kullan: Page<User> findAll(Pageable pageable)
  Akış: Stream ile lazy fetch (JPA ScrollableResults)
  Batch export: chunk bazlı işle (Spring Batch)
```

---

**Sorun 3: N+1 Query Problemi**

```
Senaryo:
  Order listesi getiriliyor → her sipariş için ayrıca müşteri bilgisi sorgulanıyor.
  100 sipariş = 1 (liste) + 100 (müşteri) = 101 sorgu!

  List<Order> orders = orderRepo.findAll();     // 1 sorgu
  for (Order o : orders) {
      o.getCustomer().getName();               // her seferinde 1 sorgu → N sorgu
  }

Düzeltme:
  @Query("SELECT o FROM Order o JOIN FETCH o.customer")
  veya
  @EntityGraph(attributePaths = "customer")
  → Tek sorguda JOIN ile getir
```

---

**Sorun 4: Yanlış zaman karmaşıklığı tahmini**

```
Senaryo:
  "Sadece 1000 kayıt var, O(n²) sorun olmaz" → 6 ay sonra 500K kayıt
  → 250 milyar operasyon → servis dondu

Ders:
  Veri büyüme hızını tahmin et.
  O(n²) olan her yer: iki iç içe döngü, Cartesian product, vs.
  Erken refactor etmek geç refactor'dan ucuzdur.
```

---

### Mülakat Soruları — Algoritma & Veri Yapıları

**Junior / Mid seviye:**

1. HashMap ve TreeMap arasındaki fark nedir? Hangisini ne zaman kullanırsın?

   > **Beklenen cevap:** HashMap O(1) get/put, sırasız. TreeMap O(log n), key'e göre sıralı (SortedMap). Range query veya sıralı traversal gerekiyorsa TreeMap, sadece hız gerekiyorsa HashMap.

2. ArrayList vs LinkedList — gerçek hayatta hangisini tercih edersin, neden?

   > **Beklenen cevap:** Neredeyse her zaman ArrayList. Random access O(1) ve cache locality iyi. LinkedList sadece başa/sona sık ekleme/silme varsa (deque olarak). Gerçek uygulamada LinkedList nadiren avantajlı.

3. Bir listedeki en sık tekrar eden elemanı O(n) karmaşıklıkla bul.

   > **Beklenen cevap:** HashMap<Element, Integer> frequency count. Tek geçişte çözülür. Follow-up: Top K için PriorityQueue (MinHeap boyutu K).

4. Stack ne zaman kullanılır? Gerçek hayat örneği ver.

   > **Beklenen cevap:** DFS traversal, expression parsing (JSON/XML), undo/redo, method call stack (JVM'in yaptığı), balanced parentheses check.

---

**Senior / Architect seviye:**

5. "Bu API'nin zaman karmaşıklığı nedir ve 10M kullanıcı için ölçeklenir mi?" — nasıl yaklaşırsın?

   > **Beklenen cevap:** Endpoint'in iç operasyonlarını analiz et. DB sorgusu O(log n) mi (index var mı?), bellek kullanımı sabit mi yoksa O(n) mi büyüyor, N+1 var mı, pagination var mı. Capacity estimation: 10M × average payload × peak QPS = bandwidth, bellek, DB yükü.

6. Trie veri yapısını ne zaman kullanırsın? Alternatifi nedir?

   > **Beklenen cevap:** Prefix search (autocomplete), IP routing tablosu, dictionary. Alternatif: Elasticsearch completion suggester (distributed, persistence var), Redis SCAN (basit ama yavaş), sorted set (lexicographic range).

7. Production'da HashMap kullanırken karşılaşılabilecek tehlikeler nelerdir?

   > **Beklenen cevap:** (1) Hash collision DoS: özelleşmiş input ile aynı bucket'a düşür → O(n) bozulur. Java 8+ HashMap: linked list → balanced BST (Java 8 treeify) ile azaltılmış. (2) Thread-safety: HashMap thread-safe değil, concurrent ortamda ConcurrentHashMap kullan. (3) mutable key: key nesnesini değiştirirsen hashCode değişir → kayıt bulunamaz.

---

## Bölüm 2: OOP & SOLID

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 5: God Class — SRP ihlali**

```
Senaryo:
  UserService: kullanıcı kayıt, login, email gönderme, rapor üretme,
               fatura oluşturma, log yönetimi... hepsi tek sınıfta.

  Sonuç:
    - 5000 satır sınıf
    - Bir değişiklik her yerde yan etki yaratıyor
    - Unit test yazmak imkansız (tüm bağımlılıkları mock etmek gerekiyor)
    - Takım çatışmaları: herkes aynı dosyayı değiştiriyor, sürekli conflict

Düzeltme:
  Etki alanlarına ayır: AuthService, EmailService, ReportService
  Event-driven: UserRegistered event → başka servisler kendi işini yapar
```

---

**Sorun 6: OCP ihlali — if/else cehennemi**

```
Senaryo:
  Ödeme sistemi: kredi kartı, PayPal, EFT, kripto → hepsi if/else.
  Yeni ödeme tipi eklemek = mevcut kodu değiştirmek = regresyon riski.
  Test coverage zor, her branch için test.

Belirti:
  "Yeni özellik ekleyince eski özellikler bozuluyor."

Düzeltme:
  PaymentProcessor interface + Strategy pattern.
  Yeni tip: yeni sınıf ekle, mevcut koda dokunma.
  Spring: @Qualifier veya Map<String, PaymentProcessor> injection.
```

---

**Sorun 7: LSP ihlali — UnsupportedOperationException sürprizi**

```
Senaryo:
  ReadOnlyList, List'i extend ediyor ama add() fırlatıyor UnsupportedOperationException.
  Kod List<> ile çalışıyor ama runtime'da patlıyor.
  Collections.unmodifiableList()'in yaptığı tam olarak bu.

Sonuç:
  "Neden çalışmıyor?" — compile time hata yok, sadece runtime.
  Testte görmemişler çünkü test mock kullandı.

Düzeltme:
  ReadableList interface: sadece get() metotları.
  add() metodu List'in parçası olmamalı bu hiyerarşide.
  Composition kullan inheritance yerine.
```

---

**Sorun 8: DIP ihlali — test edilemeyen kod**

```
Senaryo:
  OrderService içinde direkt new MySQLOrderRepository() yapılıyor.
  Unit test yazmaya çalışıyorsun ama her test gerçek DB bağlantısı açıyor.
  CI'da DB yok → testler fail → "sadece local'de çalışıyor"

Düzeltme:
  Interface inject et: OrderRepository (interface)
  Test: MockOrderRepository veya in-memory H2
  Production: MySQLOrderRepository
  Spring @Autowired / constructor injection bunu sağlar.
```

---

### Mülakat Soruları — OOP & SOLID

**Junior / Mid seviye:**

8. SOLID'in "S"sini açıkla. Kötü bir örnek ver.

   > **Beklenen cevap:** Single Responsibility — bir sınıfın değişmesi için tek bir neden olmalı. Kötü örnek: `ReportService` hem veriyi çekiyor, hem formatıyor, hem email gönderiyor. Email değişirse, formatı değiştirmek için aynı sınıfa girmek gerekiyor.

9. Composition over Inheritance ne demek? Neden önemli?

   > **Beklenen cevap:** Inheritance ("is-a") yerine bileşim ("has-a") kullan. Inheritance kırılgan hiyerarşi yaratır. Alt sınıf üst sınıfın implementation'ına sıkı bağlı. Composition: davranışı interface olarak enjekte et → test edilebilir, değiştirilebilir. Örnek: Stack extends Vector (Java'da mevcut, hatalı tasarım) vs Stack has-a Deque.

10. Interface ve Abstract Class farkı nedir? Hangisini ne zaman kullanırsın?

    > **Beklenen cevap:** Interface: pure contract, çoklu implement edilebilir, durum yok (Java 8+ default method var ama). Abstract class: kısmi implementation, shared state, "template" mantığı. Kural: "is-a" relationship + shared implementation → abstract class. "can-do" behavior → interface. Genelde interface tercih edilir (DIP için).

11. `equals()` ve `hashCode()` neden birlikte override edilmeli?

    > **Beklenen cevap:** HashMap/HashSet contract'ı: equals true olan iki nesnenin hashCode'u da eşit olmalı. Sadece equals override edersen: HashSet'e ekleyip bulamazsın çünkü farklı bucket'a gidebilir. Sadece hashCode override edersen: equals yanlış sonuç verebilir. İkisi birlikte mantıklı bir bütün oluşturur.

---

**Senior / Architect seviye:**

12. "Bize yeni bir ödeme yöntemi eklememiz gerekiyor" — OCP'ye uygun nasıl tasarım yaparsın?

    > **Beklenen cevap:** `PaymentProcessor` interface. Her yöntem ayrı sınıf. Factory veya Spring `@Component` + Map<String, PaymentProcessor>. Yeni yöntem = yeni class ekle, konfigürasyon güncelle, mevcut koda dokunma. Follow-up: Strategy vs Factory Method farkı.

13. Dependency Injection ve Dependency Inversion aynı şey mi?

    > **Beklenen cevap:** Farklı. DIP: prensip — yüksek seviye modüller soyutlamaya bağlı olmalı. DI: teknik — bağımlılıkları dışarıdan enjekte etme mekanizması. DI, DIP'i implement etmenin yolu. Spring IoC Container DI yapar, bu DIP prensibini hayata geçirir.

14. "God Object" ile nasıl mücadele edersin? 10,000 satırlık bir servisi nasıl refactor edersin?

    > **Beklenen cevap:** (1) Sorumluluklarını listele. (2) En bağımsız olanı çıkar (az bağımlılık). (3) Extract Class → yeni servis. (4) Event-driven: yeni sorumlulukları event listener'lara taşı. (5) Strangler Fig pattern: yavaş yavaş, büyük bang refactor değil. (6) Her adımda test ekle.

---

## Bölüm 3: Design Patterns

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 9: Singleton ile thread-safety problemi**

```java
// BUG — thread-safe değil
public class Config {
    private static Config instance;

    public static Config getInstance() {
        if (instance == null) {         // Thread A buraya girdi
            instance = new Config();    // Thread B de buraya girdi!
        }                               // → iki farklı instance oluştu
        return instance;
    }
}

// Sonuç: Config değerleri farklı thread'lerde farklı görünüyor.
// Debug edilmesi çok zor (race condition).

// Düzeltme 1: volatile + double-checked locking
private static volatile Config instance;

// Düzeltme 2: Initialization-on-demand holder (en clean)
private static class Holder {
    static final Config INSTANCE = new Config();
}
public static Config getInstance() { return Holder.INSTANCE; }

// Düzeltme 3: Spring kullanıyorsan @Bean ile singleton Spring yönetir
```

---

**Sorun 10: Factory yerine if/else ile type-based dispatch**

```java
// Yaygın anti-pattern
Notification send(String type) {
    if (type.equals("EMAIL")) return new EmailNotification();
    if (type.equals("SMS")) return new SmsNotification();
    if (type.equals("PUSH")) return new PushNotification();
    throw new IllegalArgumentException("Unknown type: " + type);
}

// Sorun: yeni tip = bu metodu değiştir = OCP ihlali
// Spring çözümü:
@Component("EMAIL") class EmailNotification implements Notification {}
@Component("SMS")   class SmsNotification implements Notification {}

// Servis içinde:
@Autowired Map<String, Notification> notificationMap;
// notificationMap.get("EMAIL") → Spring otomatik inject etti
```

---

**Sorun 11: Observer pattern ile hafıza sızıntısı**

```java
// EventBus veya custom observer listesi
List<EventListener> listeners = new ArrayList<>();
listeners.add(new UserDashboard(userId)); // GUI component

// Kullanıcı sayfadan ayrıldı ama listener hâlâ listede!
// GC: listener → UserDashboard → tüm grafik bileşenler → hafıza sızıntısı

// Düzeltme:
// WeakReference kullan
List<WeakReference<EventListener>> listeners = new ArrayList<>();

// Veya açıkça unsubscribe et:
bus.unsubscribe(listener); // lifecycle sonunda
```

---

**Sorun 12: Strategy pattern olmadan test edilemez kod**

```java
// Strateji hardcoded
class PriceCalculator {
    double calculate(Order order) {
        // discount algoritması burada gömülü
        if (order.isVip()) return order.getTotal() * 0.8;
        if (order.isMember()) return order.getTotal() * 0.9;
        return order.getTotal();
    }
}
// Yeni indirim tipi = bu sınıfı değiştir.
// Test: her kombinasyonu bu sınıf üzerinden test etmek gerekiyor.

// Düzeltme: Strategy
interface DiscountStrategy {
    double apply(double total);
}
class VipDiscount implements DiscountStrategy {
    public double apply(double total) { return total * 0.8; }
}
// Test: sadece VipDiscount'u test et, PriceCalculator'ı ayrı test et.
```

---

### Mülakat Soruları — Design Patterns

**Junior / Mid seviye:**

15. Singleton pattern'ın dezavantajları nelerdir?

    > **Beklenen cevap:** (1) Global state — yan etkiler takip edilmesi zor. (2) Test zorluğu — mock etmek için reflection veya özel teknikler gerekir. (3) Tight coupling — kullananlar doğrudan bağımlı. (4) Thread-safety dikkat gerektirir. Modern yaklaşım: Spring IoC container singleton scope'u yönetir, siz ayrıca implement etmezsiniz.

16. Builder pattern ne zaman kullanılır? Telescoping Constructor ile farkı nedir?

    > **Beklenen cevap:** 4+ parametreli constructor'lar okunaksız ve hataya açık olur (hangi int ne anlama geliyor?). Builder: adım adım inşa, opsiyonel parametreler temiz, immutable nesne oluşturulabilir. Lombok `@Builder` veya Record yapısı. Telescoping constructor: her kombinasyon için ayrı constructor → N! overload.

17. Proxy ve Decorator farkı nedir?

    > **Beklenen cevap:** İkisi de aynı interface'i implement eder ve başka nesneyi wrap eder. Fark: intent. Proxy: erişim kontrolü, lazy init, remote access (gerçek nesneyi bilmeyebilir). Decorator: davranış eklemek, runtime'da özellik katmak. Proxy genellikle client tarafından oluşturulmaz (framework yapar). Decorator client oluşturur. `@Transactional` → Proxy. Java I/O → Decorator.

---

**Senior / Architect seviye:**

18. Spring'de hangi design pattern'lar kullanılıyor?

    > **Beklenen cevap:** Singleton (bean scope), Factory (BeanFactory, ApplicationContext), Proxy (AOP — @Transactional, @Cacheable), Template Method (JdbcTemplate, RestTemplate), Observer (ApplicationEvent/Listener), Strategy (PasswordEncoder), Decorator (BeanPostProcessor), Chain of Responsibility (Filter chain), Builder (UriComponentsBuilder).

19. Event-driven mimari Observer pattern'ın büyük ölçekli halidir — avantajları ve dezavantajları nelerdir?

    > **Beklenen cevap:** Avantaj: loose coupling, publisher-subscriber birbirini bilmez, yeni listener eklemek mevcut kodu değiştirmez, async işleme (Kafka). Dezavantaj: Debug zor (hangi listener neyi tetikledi?), event sıralaması garanti değil (Kafka partition içinde garantili), eventual consistency (event işlenene kadar state eski), error handling karmaşık (listener başarısız olursa?).

20. Command pattern ile Event Sourcing arasındaki ilişkiyi açıkla.

    > **Beklenen cevap:** Command: işlemi nesne olarak kapsülle, undo/redo/queue için. Event Sourcing: state'i event'ların akümülasyonu olarak sakla, her state değişikliği bir Command/Event. Command → execute → Event oluşur → Event Store'a yazılır → state replay mümkün. CQRS ile birleşince: Command (write), Query (read — event'lardan türetilmiş projection).

---

## Bölüm 4: Clean Code

### Karşılaşılabilecek Gerçek Sorunlar

---

**Sorun 13: Anlaşılmaz değişken isimleri**

```java
// Gerçek hayat örneği (production codebases'de görülür)
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<>();
    for (int[] x : theList) {
        if (x[0] == 4) list1.add(x);
    }
    return list1;
}

// 3 ay sonra: 'x' ne? '4' ne? 'theList' ne? 'list1' ne döndürüyor?

// Düzeltme:
public List<Cell> getFlaggedCells() {
    return gameBoard.stream()
        .filter(cell -> cell.isFlagged())
        .collect(toList());
}
```

---

**Sorun 14: Magic number / string — bakım kabusu**

```java
// Production'daki gerçek hata:
if (order.getStatus() == 3) {  // 3 ne demek?
    sendShippingEmail();
}

// 6 ay sonra yeni geliştirici: "3'ü 4'e değiştireyim, aynı şey" → EMAIL GİTMEDİ

// Düzeltme:
enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
if (order.getStatus() == OrderStatus.SHIPPED) {
    sendShippingEmail();
}
```

---

**Sorun 15: Uzun metot — test edilemez, anlaşılmaz**

```java
// 200 satır processOrder() metodu:
void processOrder(Order order) {
    // 40 satır validation
    // 30 satır inventory check
    // 50 satır payment processing
    // 40 satır fulfillment
    // 40 satır notification
}

// Sorun: her şeyi test etmek için tüm bağımlılıkları mock etmek gerekiyor.
// Bir adım başarısız olunca hangi adımda? Debug zor.

// Düzeltme: Her adım ayrı metot/sınıf
void processOrder(Order order) {
    orderValidator.validate(order);
    inventoryService.reserve(order);
    paymentService.charge(order);
    fulfillmentService.initiate(order);
    notificationService.notifyOrderPlaced(order);
}
```

---

### Mülakat Soruları — Clean Code & Genel

**Junior / Mid seviye:**

21. "Clean Code" ne demek? Bir cümleyle tanımla.

    > **Beklenen cevap:** Yazıldıktan 6 ay sonra başkası (veya siz) okuduğunda ne yaptığını anlaması için minimum çaba gerektiren kod.

22. Yorum satırı ne zaman yazılmalı?

    > **Beklenen cevap:** Kod "ne yaptığını" değil, "neden öyle yaptığını" açıklamalı yorum. `// increment i` → gereksiz. `// RFC 2616 gereği cache header ekleniyor` veya `// race condition önlemek için lock sırası önemli` → gerekli. İyi isimlendirme çoğu yorumun önüne geçer.

23. DRY prensibini ihlal ettiğin bir örnek ver, nasıl düzeltirsin?

    > **Beklenen cevap:** Aynı validation mantığı 5 farklı endpoint'te copy-paste. → Ortak `@Validator` bean veya utility metot. Dikkat: DRY "bilgiyi" tekrarlamamak demek, bazen benzer görünen kod farklı bilgileri temsil eder — o zaman ayrı kalabilir.

---

**Senior / Architect seviye:**

24. Technical debt (teknik borç) nedir? Nasıl yönetilir?

    > **Beklenen cevap:** Hızlı çözüm için alınan kısa vadeli kararların uzun vadeli bakım maliyeti. Türleri: kasıtlı-bilinçli (deadline baskısı, sonra düzeltilecek), kasıtlı-ihmalkâr (biliyoruz ama umursamıyoruz), kasıtsız (o anki bilgi eksikliği). Yönetim: ADR (Architecture Decision Record) ile belgeleme, teknik borç backlog, her sprintte %20 iyileştirme zamanı, refactor → test coverage artışı.

25. Code review'da en çok nelere dikkat edersin?

    > **Beklenen cevap:** (1) Correctness: edge case, null, concurrency. (2) Güvenlik: injection, sensitive data exposure, auth kontrol. (3) Performance: N+1, O(n²), unbounded collection. (4) Testability: mock edilebilir mi, test var mı. (5) Design: SRP ihlali var mı, soyutlama doğru mu. (6) Readability: isimler açık mı, metot uzunluğu. NOT söylemesi gereken: "naming" gibi küçük şeylere takılmak değil, önemli şeylere odaklanmak.

---

## Karma — Architect Seviyesi Zorlu Sorular

26. **"HashMap neden thread-safe değil?"** — derinlemesine açıkla.

    > **Beklenen cevap:** Internal resize (rehashing): eleman sayısı threshold'u aşınca yeni array oluşturulur ve tüm elemanlar yeniden konumlandırılır. İki thread aynı anda resize yaparsa → sonsuz döngü (Java 7'de infinite loop, Java 8+'da yine veri kaybı). Compound operations da atomic değil: check-then-act (get then put). Çözüm: ConcurrentHashMap — segment-level lock (Java 7) → CAS + synchronized bucket (Java 8+).

27. **"SOLID prensiplerini ihlal etmeden microservice mimarisi nasıl tasarlanır?"**

    > **Beklenen cevap:** SRP → her servis tek bounded context. OCP → event-driven: yeni servis event'a subscribe olur, publisher değişmez. LSP → API contract: provider versioning ile backward compat. ISP → API endpoint'leri fine-grained (BFF pattern). DIP → servislerin birbirinin concrete URL'ine değil, servis discovery (Consul, Kubernetes Service) üzerinden bulduğu soyutlamaya bağlı. Anti-corruption layer → dış sistemlerle entegrasyonda DIP.

28. **"Strategy, Template Method ve Command pattern'larını karşılaştır. Hangisini ne zaman seçersin?"**

    > **Beklenen cevap:** Strategy: algoritmayı tamamen değiştir (runtime), client seçer. Template Method: algoritma iskeletini sabitle, adımları değiştir (inheritance, compile-time). Command: işlemi nesne olarak kapsülle, undo/queue/log gerekiyorsa. Seçim: Runtime değişiklik → Strategy. Sabit akış, değişen adımlar → Template. Undo/Redo/Queue → Command.

29. **"Bir microservice 10 saniyede başlıyor, nasıl debug edersin?"**

    > **Beklenen cevap:** Spring Actuator startup metrics (`/actuator/startup`). Spring Boot 2.4+: startup step recording. `--debug` flag ile auto-configuration raporu. JVM profiler (JFR): sınıf yükleme, bean init. Lazy bean init: `spring.main.lazy-initialization=true`. Şüpheli: büyük bean graph, eager Elasticsearch/DB bağlantısı, component scan kapsamı (tüm classpath yerine spesifik paket).

30. **"Immutability neden önemli? Architect olarak nerede tercih edersin?"**

    > **Beklenen cevap:** Thread-safety otomatik (synchronization gerekmez). Cache güvenli (nesne değişmeyeceği için). Side-effect yok. Dezavantaj: her değişiklik yeni nesne → GC baskısı (String, wrapper klas interning bunu azaltır). Kullanım: DTO, Value Object, Event (Event Sourcing event'ları immutable olmalı — geçmiş değiştirilmez), functional programlama. Java: `record`, Lombok `@Value`, Guava ImmutableList.
