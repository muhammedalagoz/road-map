# 06a — Database per Service Pattern

## Ne?

Her microservice kendi veritabanını yönetir. Başka bir servis bu veritabanına **doğrudan** erişemez; sadece o servisin API'si üzerinden erişilir.

```
Monolith:
  [OrderService]─┐
  [PaymentService]├─→ Tek PostgreSQL DB (paylaşımlı)
  [UserService]──┘

Microservices (Database per Service):
  [OrderService]   → PostgreSQL (order_db)
  [PaymentService] → PostgreSQL (payment_db)  ← ayrı şema, ayrı bağlantı
  [UserService]    → MongoDB    (user_db)
  [CatalogService] → Elasticsearch (catalog_index)
```

---

## Neden?

**Çözdüğü problemler:**

```
1. Bağımsız ölçekleme:
   Catalog servisi yoğun okuma → Elasticsearch
   Order servisi ACID gerektiriyor → PostgreSQL
   Session servisi hız kritik → Redis
   Tek DB kullanılırsa: hepsine aynı çözüm, uygun olmayabilir

2. Bağımsız deployment:
   Paylaşımlı DB'de bir servis şema değiştirirse → diğerleri etkilenir
   Ayrı DB: migration diğer servisleri bloke etmez

3. Hata izolasyonu:
   Paylaşımlı DB'de bir servis tüm bağlantıları tüketirse → diğerleri çöker
   Ayrı connection pool: izole

4. Teknoloji özgürlüğü:
   Her servis en uygun DB'yi seçebilir
   (Catalog → Elasticsearch, Graph → Neo4j, IoT → TimescaleDB)
```

---

## Nasıl?

### Veri Paylaşım Stratejileri

**Sorun:** `OrderService` bir ürünün fiyatını nasıl öğrenir? `CatalogService`'in DB'sine doğrudan erişemez.

#### 1. API Çağrısı (Senkron)

```java
// OrderService → CatalogService REST/gRPC çağrısı
@Service
class OrderService {

    @Autowired CatalogClient catalogClient;

    Order createOrder(CreateOrderRequest req) {
        // Her ürün için fiyat çek
        List<ProductDto> products = req.getProductIds().stream()
            .map(catalogClient::getProduct)   // HTTP çağrısı
            .collect(Collectors.toList());

        // Siparişi oluştur
        return buildOrder(req, products);
    }
}

// Avantaj: Her zaman güncel fiyat
// Dezavantaj: CatalogService down → OrderService çalışmaz (tight coupling)
```

#### 2. Event-Driven Replikasyon (Önerilen)

```java
// CatalogService → fiyat değişince event yayınlar
class CatalogService {
    void updatePrice(ProductId id, Money newPrice) {
        productRepo.updatePrice(id, newPrice);
        eventBus.publish(new ProductPriceUpdatedEvent(id, newPrice));
    }
}

// OrderService bu event'i dinler ve kendi read model'ine kopyalar
@KafkaListener(topics = "catalog.product-price-updated")
void onPriceUpdated(ProductPriceUpdatedEvent event) {
    // OrderService'in kendi DB'sinde product_prices tablosu
    localProductPriceRepo.upsert(event.getProductId(), event.getNewPrice());
}

// Order oluştururken local kopyayı kullan
Order createOrder(CreateOrderRequest req) {
    List<ProductPrice> prices = localProductPriceRepo
        .findAllById(req.getProductIds());
    return buildOrder(req, prices);
}

// Avantaj: CatalogService down olsa da çalışır
// Dezavantaj: Eventual consistency — fiyat anlık olmayabilir
```

#### 3. Shared Database (Anti-Pattern — sadece geçiş döneminde)

```
[ServiceA] ──→ Shared schema (read-only)
[ServiceB] ──→ Shared schema (write)

KÖTÜ UYGULAMA:
  Yalnızca migration döneminde kabul edilebilir
  Long-term: coupling, bağımsız deploy imkansız
  Çözüm: Zamanla ayrıştır, Event-Driven replikasyona geç
```

---

### Veri Tutarlılığı: Saga Pattern

`OrderService` + `PaymentService` + `InventoryService` — hepsinde ayrı DB, atomik işlem nasıl?

```
2PC kullanamayız (distributed lock, blocking, microservices'e uygun değil)
Çözüm: Saga (04h-messaging-patterns.md'de detaylı)

Choreography örneği:
  OrderService    → OrderCreated event       → order_db: PENDING
  PaymentService  → PaymentProcessed event   → payment_db: CHARGED
  InventoryService→ InventoryReserved event  → inventory_db: RESERVED
  ShippingService → ShipmentCreated event    → shipping_db: IN_PROGRESS

Hata (Compensating Transaction):
  InventoryService → InventoryFailed event
  PaymentService   → PaymentRefunded (compensate)
  OrderService     → OrderCancelled (compensate)
```

---

### Migration Stratejisi

**Paylaşımlı DB'den ayrı DB'ye geçiş:**

```
Adım 1 — Strangler Fig ile şema ayrımı:
  Tek DB'de ayrı şema oluştur (hâlâ aynı server)
  orders_schema.orders
  payments_schema.payments

Adım 2 — Servis katmanı izolasyonu:
  PaymentService sadece payments_schema'ya erişir
  Cross-schema JOIN'leri kaldır

Adım 3 — Fiziksel ayrım:
  payments_schema → ayrı DB instance'a taşı
  Dual-write: bir süre eski DB + yeni DB'ye yaz
  Verification: iki taraf tutarlı mı?
  Cutover: yeni DB'ye geç, eski'yi kapat
```

---

### Polylgot Persistence Örnekleri

```
E-ticaret platformu:

UserService       → PostgreSQL     (ACID, relational, auth)
CatalogService    → Elasticsearch  (full-text search, filter)
OrderService      → PostgreSQL     (transaction, ACID)
CartService       → Redis          (hız, TTL, ephemeral)
SessionService    → Redis          (token, TTL)
AnalyticsService  → ClickHouse     (OLAP, aggregation)
RecommendationSvc → Neo4j          (graph, user-product ilişkisi)
NotificationSvc   → MongoDB        (flexible schema, log)
PaymentService    → PostgreSQL     (ACID, audit log)
```

---

## Ne zaman?

```
Database per Service kullan:
✓ Servisler bağımsız ölçeklenmeli
✓ Her servis farklı DB teknolojisi ihtiyacı
✓ Bağımsız deployment kritik
✓ Takımlar bağımsız çalışmalı

Geçici olarak paylaşımlı DB kabul edilebilir:
✓ Monolith → microservices geçiş döneminde
✓ Takım küçük, yönetim overhead çok
✓ Domain henüz netleşmemiş

Zorluğa hazır ol:
  ✗ Cross-service JOIN yok → API çağrısı veya event replikasyonu
  ✗ Distributed transaction yok → Saga pattern
  ✗ Data consistency eventual → business'ın kabul etmesi gerekir
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Bağımsız ölçekleme (her servis farklı DB) | Cross-service JOIN yok |
| Teknoloji özgürlüğü (polyglot persistence) | Distributed transaction karmaşık (Saga) |
| Hata izolasyonu | Data replikasyonu gerekebilir (eventual consistency) |
| Bağımsız migration/deployment | Operasyonel karmaşıklık (birçok DB yönet) |
| Takım bağımsızlığı | Veri tutarsızlığı olasılığı (compensating logic) |
