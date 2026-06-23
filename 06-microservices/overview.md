# 06 — Microservices & Domain-Driven Design

## 1. Monolith → Microservices

### Ne Zaman Microservices?

**Monolith ile başla** eğer:
- Ekip < 10 developer
- Domain henüz netleşmemiş
- İlk ürün, pivot olabilir

**Microservices'e geç** eğer:
- Ekipler bağımsız deploy edemiyor
- Belirli modüller farklı ölçekleme ihtiyacı
- Domain boundaries netleşti
- Bağımsız teknoloji seçimi gerekiyor

### Strangler Fig Pattern

```
Mevcut monolith'i adım adım söküp ayır:

1. Yeni servis yaz (mesela OrderService)
2. API Gateway ekle — yeni endpoint → yeni servis
3. Monolith'teki ilgili kodu deaktive et / sil

    ┌─────────────┐
    │   Client    │
    └──────┬──────┘
           ↓
    ┌─────────────┐
    │ API Gateway │
    └──┬──────┬───┘
       │      │
       ↓      ↓
  [OrderSvc] [Monolith]  ← zamanla monolith küçülür
```

---

## 2. Domain-Driven Design (DDD)

### Temel Kavramlar

**Ubiquitous Language:** Ekip (developer + business) aynı dili konuşur. Kod'daki isimler iş dünyasıyla aynı.

```
YANLIŞ: class TblUserOrdr { int usrId; int prodId; }
DOĞRU:  class Order { Customer customer; List<OrderItem> items; }
```

**Bounded Context:** Modelin geçerli olduğu sınır.

```
E-ticaret:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Order Context  │  │ Catalog Context  │  │ Payment Context  │
│                  │  │                  │  │                  │
│ Order            │  │ Product          │  │ Payment          │
│ OrderItem        │  │ Category         │  │ Invoice          │
│ Customer (basit) │  │ Product (zengin) │  │ Customer (basit) │
└─────────────────┘  └─────────────────┘  └─────────────────┘

"Customer" her context'te farklı anlam taşır. Bu normal!
Order context: sadece {id, name, address}
CRM context: {id, name, email, phone, history, segment, ...}
```

**Aggregate:** Birlikte tutarlı olması gereken entity grubu. Dışarıdan sadece root'a erişilir.

```java
// Order Aggregate
class Order {           // Aggregate Root
    OrderId id;
    List<OrderItem> items; // Aggregate içi entity
    Money totalAmount;
    OrderStatus status;
    
    void addItem(Product product, int quantity) {
        // İş kuralları burada enforce edilir
        if (status != OrderStatus.DRAFT) throw new IllegalStateException();
        items.add(new OrderItem(product, quantity));
        recalculateTotal();
    }
    
    void confirm() {
        if (items.isEmpty()) throw new OrderException("Boş sipariş onaylanamaz");
        status = OrderStatus.CONFIRMED;
        // Domain event ekle
        registerEvent(new OrderConfirmedEvent(this.id));
    }
}

// OrderItem doğrudan manipüle edilmez, sadece Order üzerinden
```

**Domain Event:** Iş açısından önemli bir şey oldu.

```java
// Order onaylandığında
class OrderConfirmedEvent {
    OrderId orderId;
    CustomerId customerId;
    Money totalAmount;
    Instant occurredAt;
}

// OrderService bu event'i yayınlar
// EmailService, InventoryService, PaymentService dinler
```

### Context Map — Servisler Arası İlişki

```
Order Context ──Upstream──→ Shipping Context
               (Order, Shipping'i besler)

Anti-Corruption Layer (ACL):
Dış sistem farklı model kullanıyorsa:
Dış API → ACL (Translator) → İç model
Dış API değişse bile iç model korunur
```

---

## 3. Service Boundaries

**Hatalı ayrım (teknik sınır):**
```
DatabaseService  ← veri katmanı servis değil
UIService        ← presentation katmanı servis değil
```

**Doğru ayrım (iş yeteneği bazlı):**
```
OrderManagement  ← sipariş al, güncelle, iptal
Inventory        ← stok takip, rezervasyon
Fulfillment      ← paketleme, kargo
Customer         ← kayıt, profil, adres
Notification     ← email, SMS, push
```

**Heuristic:** Bir değişiklik birden fazla servisi etkiliyorsa, yanlış ayrım.

---

## 4. Inter-Service Communication

### Sync: REST vs gRPC

**REST:**
```
HTTP/1.1 + JSON
Avantaj: Evrensel, browser erişilebilir, human-readable
Dezavantaj: JSON parsing overhead, HTTP/1.1 head-of-line blocking
```

**gRPC:**
```
HTTP/2 + Protocol Buffers (binary)

Proto tanımı:
syntax = "proto3";
service OrderService {
    rpc GetOrder (GetOrderRequest) returns (Order);
    rpc StreamOrders (StreamRequest) returns (stream Order); // server streaming
}

message Order {
    string id = 1;
    string customerId = 2;
    double totalAmount = 3;
}

Avantaj:
- Binary → 3-10x daha küçük payload
- HTTP/2 → multiplexing, server push
- Streaming (server, client, bidirectional)
- Strong typing, code generation
- 5-10x daha hızlı (JSON vs protobuf parse)

Dezavantaj:
- Browser direkt desteklemez (gRPC-Web gerekir)
- Human-readable değil (debug için araç gerekir)
- Proto şema yönetimi

Kullanım: Internal microservice iletişimi, yüksek throughput gereken yerler
```

### Async: Event-Driven

```
OrderService → Kafka/RabbitMQ → [InventoryService, EmailService, AnalyticsService]

Avantaj:
- Temporal decoupling (alıcı hazır olmak zorunda değil)
- Load leveling
- Publisher alıcıyı bilmez

Dezavantaj:
- Eventual consistency
- Debugging zor (distributed tracing şart)
- Idempotency zorunlu
```

---

## 5. API Gateway Pattern

```
Client → API Gateway → [OrderService, PaymentService, UserService]

API Gateway sorumlulukları:
- Routing (hangi servis?)
- Authentication (JWT doğrulama)
- Rate Limiting
- SSL Termination
- Request/Response transformation
- Load Balancing
- Caching
- Logging & Monitoring
- Circuit Breaking

Araçlar: Kong, AWS API Gateway, Nginx, Spring Cloud Gateway
```

---

## 6. Outbox Pattern

**Problem:** Veritabanına kaydet + mesaj gönder — ikisi atomik değil.

```
// YANLIŞ — iki ayrı sistem, biri başarılı diğeri başarısız olabilir
@Transactional
void processOrder(Order order) {
    orderRepo.save(order);
    kafkaTemplate.send("order-topic", event); // exception → DB rollback olmaz!
}
```

**Outbox çözümü:**
```
@Transactional
void processOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxMessage("order-topic", event)); // aynı transaction!
}

// Ayrı process: Outbox tablosunu polling eder ve Kafka'ya gönderir
// Debezium (CDC) ile: outbox tablosundaki değişiklik → Kafka (daha iyi)

Outbox tablosu:
| id | aggregate_type | aggregate_id | event_type    | payload | sent_at |
|----|----------------|--------------|---------------|---------|---------|
| 1  | Order          | 123          | OrderCreated  | {...}   | null    |
```

**CDC (Change Data Capture) ile Outbox:**
```
PostgreSQL WAL → Debezium (CDC connector) → Kafka
Polling gerekmez, gerçek zamanlı, düşük latency
```

---

## 7. CQRS (Command Query Responsibility Segregation)

**Temel fikir:** Read ve Write modeli ayır.

```
Write Side (Command):
Command → CommandHandler → Aggregate → Domain Events → Event Store/DB

Read Side (Query):
Events → Projections → Read Model (optimize edilmiş view) → Query Handler → Client

Örnek:
Write: Order tablosu (normalized, ACID)
Read:  OrderSummaryView (denormalized, hızlı okuma)
       CustomerOrdersView
       DashboardStatsView
```

**Ne zaman CQRS?**
- Read/Write oranı çok farklı (100:1 gibi)
- Read modeli farklı optimizasyon gerektiriyor
- Event Sourcing kullanıyorsun
- Karmaşık domain logic var

**Ne zaman kullanma?**
- Basit CRUD uygulaması
- Küçük ekip, karmaşıklık kaldırmaz
- Read/Write benzer yük

---

## 8. Event Sourcing

**Temel fikir:** State'i kaydetme, event'leri kaydet. State = event'lerin replay'i.

```
Klasik:
ORDER tablosu: {id: 123, status: SHIPPED, total: 150}

Event Sourcing:
events tablosu:
1. OrderCreated    {orderId: 123, customer: Ali, items: [...]}
2. PaymentReceived {orderId: 123, amount: 150}
3. OrderShipped    {orderId: 123, trackingCode: TRK001}

Current state = 1+2+3 replay edilince elde edilir
```

**Avantajlar:**
- Tam audit log (ne zaman ne oldu)
- Herhangi bir anki state'e gidebilirsin
- Yeni projeksiyon ekleyebilirsin (geçmişi replay et)
- Event-driven mimaride doğal

**Dezavantajlar:**
- Karmaşık
- Eventual consistency
- Event schema değişimi zor (upcasting gerekir)
- Sorgu zorluğu (projeksiyon olmadan okuyamazsın)

```java
// Aggregate event sourcing ile:
class Order {
    List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    static Order reconstitute(List<DomainEvent> history) {
        Order order = new Order();
        history.forEach(order::apply);
        return order;
    }
    
    void confirm() {
        apply(new OrderConfirmedEvent(this.id, Instant.now()));
    }
    
    private void apply(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
        uncommittedEvents.add(event);
    }
}
```

---

## 9. Service Mesh (Istio)

**Problem:** Her servise circuit breaker, retry, mTLS, tracing eklemek.  
**Çözüm:** Sidecar proxy (Envoy) her pod'a eklenir.

```
Pod:
┌─────────────────────────────┐
│  OrderService Container     │
│  +                          │
│  Envoy Sidecar Proxy        │  ← traffic burada intercept edilir
└─────────────────────────────┘

Özellikler (uygulama kodu değişmeden):
- mTLS (servisler arası şifreleme)
- Circuit breaking
- Retry
- Load balancing (L7)
- Distributed tracing (Jaeger entegrasyonu)
- Traffic mirroring (canary testing)
- Timeout

Kontrol plane (Istio):
istiod → tüm Envoy proxy'lere config push eder
```

**Service mesh ne zaman?**
- 10+ microservice
- mTLS zorunluysa (compliance)
- Fine-grained traffic control gerekiyorsa
- Observability merkezi olmalı

**Dezavantaj:** Karmaşıklık, latency overhead (~1ms), kaynak tüketimi (her pod'da Envoy)
