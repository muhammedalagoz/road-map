# 01 — Temeller: CS, OOP, Design Patterns

## 1. Algoritma & Veri Yapıları (Architect Perspektifi)

Architect olarak algoritmaları ezberlemek değil, **karmaşıklığı ve trade-off'ları** anlamak gerekir.

### Zaman & Alan Karmaşıklığı

```
O(1)    → HashMap.get(), array index erişimi
O(log n)→ Binary search, balanced BST (TreeMap)
O(n)    → Tek geçişli tarama
O(n log n) → Merge sort, heap sort
O(n²)  → İç içe döngüler — büyük veri setlerinde kabul edilemez
```

**Architect olarak ne bilmem gerekiyor?**
- Bir API'nin zaman karmaşıklığını tahmin edebilmek
- "Bu sorgu 10M satır için çalışır mı?" sorusunu cevaplayabilmek
- Hangi veri yapısının hangi erişim pattern'ına uygun olduğunu seçmek

### Kritik Veri Yapıları ve Kullanım Alanları

| Veri Yapısı | Ortalama Erişim | Kullanım Alanı |
|-------------|-----------------|----------------|
| Array/ArrayList | O(1) | Sıralı erişim, index bazlı okuma |
| LinkedList | O(n) | Sık insert/delete, queue implementasyonu |
| HashMap | O(1) amortized | Cache, lookup table, frequency count |
| TreeMap | O(log n) | Sıralı key traversal, range query |
| HashSet | O(1) | Unique kontrolü, membership test |
| PriorityQueue (Heap) | O(log n) | En büyük/küçük N eleman, scheduling |
| Stack | O(1) | DFS, expression parsing, undo/redo |
| Queue | O(1) | BFS, işlem sırası, rate limiting |
| Trie | O(k) k=key length | Autocomplete, prefix search |

---

## 2. OOP Prensipleri

### SOLID

#### S — Single Responsibility Principle
Bir sınıfın değişmesi için **tek bir neden** olmalı.

```java
// YANLIŞ — UserService hem auth hem email hem de DB işlemi yapıyor
class UserService {
    void register(User user) {
        validateUser(user);          // validation
        sendWelcomeEmail(user);      // email
        userRepository.save(user);  // persistence
        auditLog.log("registered");  // logging
    }
}

// DOĞRU — her sorumluluk ayrı sınıfta
class UserRegistrationService {
    void register(User user) {
        userValidator.validate(user);
        userRepository.save(user);
        eventPublisher.publish(new UserRegisteredEvent(user));
    }
}
// Email, audit log → event listener'lar handle eder
```

#### O — Open/Closed Principle
Sınıflar genişletmeye açık, değiştirmeye kapalı olmalı.

```java
// YANLIŞ — yeni ödeme yöntemi eklemek için mevcut kodu değiştirmek gerekiyor
class PaymentService {
    void pay(String method, double amount) {
        if (method.equals("credit")) { /* ... */ }
        else if (method.equals("paypal")) { /* ... */ }
        // yeni eklemek için buraya if yazmak gerekiyor — OCP ihlali
    }
}

// DOĞRU — yeni ödeme yöntemi yeni sınıf ekleyerek gelir
interface PaymentProcessor {
    void process(double amount);
}
class CreditCardProcessor implements PaymentProcessor { /* ... */ }
class PayPalProcessor implements PaymentProcessor { /* ... */ }
```

#### L — Liskov Substitution Principle
Alt sınıf, üst sınıfın yerine geçtiğinde sistem doğru çalışmalı.

```java
// KLASİK İHLAL — Square extends Rectangle
class Rectangle {
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    @Override
    void setWidth(int w) { this.width = w; this.height = w; } // LSP ihlali!
}

// Şimdi Rectangle kullanan kod:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(3);
// r.area() → 9 beklenir ama 9 döner (Square her ikisini de 3 yaptı)
// Davranış beklenenden farklı → LSP ihlali
```

#### I — Interface Segregation Principle
Büyük interface yerine küçük, özelleşmiş interface'ler.

```java
// YANLIŞ
interface Worker {
    void work();
    void eat();
    void sleep();
}

// Robot eat() ve sleep() implement etmek zorunda kalır
class Robot implements Worker {
    void work() { /* ok */ }
    void eat() { throw new UnsupportedOperationException(); } // saçma
    void sleep() { throw new UnsupportedOperationException(); } // saçma
}

// DOĞRU
interface Workable { void work(); }
interface Eatable { void eat(); }
class Human implements Workable, Eatable { /* ... */ }
class Robot implements Workable { /* ... */ }
```

#### D — Dependency Inversion Principle
Yüksek seviye modüller düşük seviye modüllere bağımlı olmamalı; ikisi de soyutlamaya bağımlı olmalı.

```java
// YANLIŞ
class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository(); // concrete'e bağımlı
}

// DOĞRU — soyutlamaya bağımlı, Spring @Autowired bunu sağlar
class OrderService {
    private final OrderRepository repository; // interface
    
    OrderService(OrderRepository repository) { // constructor injection
        this.repository = repository;
    }
}
```

### DRY, KISS, YAGNI

| Prensip | Açıklama | Architect için önemi |
|---------|----------|---------------------|
| **DRY** (Don't Repeat Yourself) | Bilgi tek yerde olmalı. Copy-paste değil, soyutlama | Tutarsızlık riski azaltır, bakım kolaylaşır |
| **KISS** (Keep It Simple, Stupid) | Basit çözüm tercih edilmeli | Over-engineering mimari karmaşıklık yaratır |
| **YAGNI** (You Aren't Gonna Need It) | Şu an gerekli olmayan özelliği ekleme | Gereksiz karmaşıklık ve teknik borç önler |

---

## 3. Design Patterns

### Creational Patterns (Nesne Yaratma)

#### Singleton
Uygulama boyunca tek örnek.

```java
// Thread-safe Singleton (double-checked locking)
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    
    private DatabaseConnection() {}
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) { // double-check
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}
```

**Ne zaman kullan?** Config manager, connection pool, logger  
**Anti-pattern uyarısı:** Test edilmesi zor, global state yaratır. Spring'de `@Bean` singleton varsayılandır, ayrıca implement etmeye gerek yok.

#### Factory Method
Nesne yaratmayı alt sınıflara bırak.

```java
abstract class NotificationSender {
    abstract Notification createNotification(); // factory method
    
    void send(String message) {
        Notification n = createNotification();
        n.send(message);
    }
}

class EmailNotificationSender extends NotificationSender {
    @Override
    Notification createNotification() { return new EmailNotification(); }
}
```

#### Builder
Karmaşık nesne yaratımı için adım adım inşa.

```java
// Lombok @Builder bunu generate eder
User user = User.builder()
    .name("Ali")
    .email("ali@example.com")
    .role(Role.ADMIN)
    .build();
```

**Ne zaman kullan?** 4+ parametre olan constructor'lar, opsiyonel parametreler çok olduğunda.

### Structural Patterns (Yapı)

#### Adapter
Uyumsuz interface'leri birbirine bağla.

```java
// Eski sistem XML döndürüyor, yeni sistem JSON istiyor
interface JsonDataSource {
    JsonObject getData();
}

class XmlToJsonAdapter implements JsonDataSource {
    private LegacyXmlService xmlService;
    
    @Override
    public JsonObject getData() {
        XmlDocument xml = xmlService.fetchData();
        return convertXmlToJson(xml); // dönüşüm burada
    }
}
```

#### Decorator
Nesneye dinamik olarak davranış ekle (inheritance olmadan).

```java
// Java I/O'nun temeli bu pattern
InputStream stream = new BufferedInputStream(
                        new GZIPInputStream(
                            new FileInputStream("data.gz")));
// Her decorator bir özellik ekliyor: buffering, decompression, file reading
```

#### Proxy
Gerçek nesnenin önüne geçerek erişimi kontrol et.

```java
// Spring AOP bu pattern'ı kullanır — @Transactional, @Cacheable, @Async
// Sen OrderService inject ettiğinde aslında proxy gelir:
//   proxy.save() → begin transaction → orderService.save() → commit/rollback
```

**Proxy türleri:**
- **Virtual Proxy** — lazy loading (Hibernate entity lazy fetch)
- **Protection Proxy** — erişim kontrolü
- **Remote Proxy** — uzak nesneye yerel erişim (RPC)
- **Caching Proxy** — sonuçları cache'le (`@Cacheable`)

### Behavioral Patterns (Davranış)

#### Strategy
Algoritmaları çalışma zamanında değiştirilebilir hale getir.

```java
interface SortStrategy {
    void sort(int[] data);
}

class QuickSort implements SortStrategy { /* ... */ }
class MergeSort implements SortStrategy { /* ... */ }

class Sorter {
    private SortStrategy strategy;
    
    void setStrategy(SortStrategy strategy) { this.strategy = strategy; }
    void sort(int[] data) { strategy.sort(data); }
}

// Kullanım: veri boyutuna göre strateji seç
sorter.setStrategy(data.length < 1000 ? new QuickSort() : new MergeSort());
```

#### Observer
Bir nesne değiştiğinde bağlı nesneleri otomatik bilgilendir.

```java
// Spring ApplicationEvent bu pattern'ı implement eder
@Component
class OrderService {
    @Autowired ApplicationEventPublisher publisher;
    
    void placeOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order)); // observer'lara bildir
    }
}

@Component
class EmailListener {
    @EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.getOrder());
    }
}
```

#### Command
İşlemi nesne olarak kapsülle — undo/redo, queue, log için.

```java
interface Command {
    void execute();
    void undo();
}

class TransferMoneyCommand implements Command {
    void execute() { fromAccount.debit(amount); toAccount.credit(amount); }
    void undo()    { fromAccount.credit(amount); toAccount.debit(amount); }
}
// Queue'ya ekleyip async çalıştır, başarısız olursa undo() çağır
```

#### Template Method
Algoritma iskeletini üst sınıfta tanımla, adımları alt sınıflara bırak.

```java
abstract class DataProcessor {
    // template method — sıra değişmez
    final void process() {
        readData();     // → alt sınıf implement eder
        processData();  // → alt sınıf implement eder
        writeData();    // → alt sınıf implement eder
    }
    
    abstract void readData();
    abstract void processData();
    abstract void writeData();
}

class CsvDataProcessor extends DataProcessor {
    void readData() { /* CSV oku */ }
    void processData() { /* dönüştür */ }
    void writeData() { /* kaydet */ }
}
```

#### Circuit Breaker (Enterprise Pattern)
Başarısız servise yapılan çağrıları kes, sistemi koru.

```
CLOSED (normal) → hatalar eşiği aşınca → OPEN (çağrıları reddet)
OPEN → belirli süre sonra → HALF-OPEN (test çağrısı)
HALF-OPEN → başarılıysa → CLOSED
           → başarısızsa → OPEN
```

---

## 4. Clean Code Prensipleri

### İsimlendirme
```java
// YANLIŞ
int d; // elapsed time in days
List<int[]> getThem() { ... }

// DOĞRU
int elapsedTimeInDays;
List<GameCell> getFlaggedCells() { ... }
```

### Fonksiyon Büyüklüğü
- Bir fonksiyon **bir şey** yapmalı
- 20 satırdan uzunsa genellikle bölünmeli
- Abstraction seviyesi tutarlı olmalı (bir metodun içinde hem HTTP çağrısı hem SQL sorgusu olmamalı)

### Sihirli Sayılardan Kaçın
```java
// YANLIŞ
if (user.getAge() > 18) { ... }
if (status == 3) { ... }

// DOĞRU
private static final int LEGAL_AGE = 18;
if (user.getAge() > LEGAL_AGE) { ... }
if (status == OrderStatus.SHIPPED) { ... }
```

### Yorum Satırları
```java
// YANLIŞ — kod ne yapıyor zaten belli
// increment i by 1
i++;

// DOĞRU — neden böyle yapıldığı belli değil, yorum gerekli
// RFC 2616 §14.9.3: must revalidate if response is stale
response.setHeader("Cache-Control", "no-cache, must-revalidate");
```
