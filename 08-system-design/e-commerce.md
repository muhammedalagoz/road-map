# 08r — System Design: E-Ticaret Platformu (Amazon)

## Gereksinimler

```
Functional:
  ✓ Ürün kataloğu ve arama
  ✓ Sepet yönetimi
  ✓ Sipariş oluşturma ve takip
  ✓ Stok yönetimi (overselling önleme)
  ✓ Fiyat ve kampanya yönetimi
  ✗ Ödeme, teslimat (out of scope)

Non-Functional:
  DAU: 10M
  Ürün: 100M ürün
  Peak: Black Friday → 10x normal trafik
  Sipariş latency: < 3s
  Availability: 99.99%
  Stok tutarsızlığı: sıfır tolerans
```

---

## Microservice Mimarisi

```
          Client (Web/Mobile)
                │
         ┌──────┴───────┐
         │  API Gateway  │
         │  BFF (Web/Mob)│
         └──────┬────────┘
                │
    ┌───────────┼────────────────────────────┐
    ▼           ▼          ▼         ▼       ▼
[Product    [Search    [Cart     [Order   [Inventory
 Service]    Service]   Service]  Service] Service]
    │           │          │         │        │
[Product    [Elastic   [Redis]  [PostgreSQL] [PostgreSQL
   DB]       search]                          + Redis]
```

---

## Ürün Kataloğu Servisi

```
100M ürün → tek tablo imkansız

Veri bölümleme:
  PostgreSQL: master veri (id, name, price, sellerId)
  Elasticsearch: arama indexi (title, description, category, attributes)
  Redis: hot ürün cache (son 7 günde 1000+ görüntüleme)
  CDN: ürün görselleri (S3 + CloudFront)

Ürün görüntüleme QPS:
  10M × 20 sayfa görüntüleme / 86400 ≈ 2,300 QPS
  Cache hit rate %95 → DB: 115 QPS (yönetilebilir)
```

```java
@Service
class ProductService {

    Product getProduct(String productId) {
        // L1: Redis
        Product cached = redis.get("product:" + productId);
        if (cached != null) return cached;

        // L2: DB
        Product product = productRepo.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // Cache: hot product için 1 saat, diğer 5 dk
        Duration ttl = isHotProduct(product) ? Duration.ofHours(1) : Duration.ofMinutes(5);
        redis.setEx("product:" + productId, product, ttl);

        return product;
    }
}
```

---

## Arama (Elasticsearch)

```json
// Ürün indexi
{
  "productId": "p-123",
  "title": "Nike Air Max 270",
  "description": "Hafif koşu ayakkabısı...",
  "category": ["Ayakkabı", "Spor", "Nike"],
  "brand": "Nike",
  "price": 1299.99,
  "discountedPrice": 999.99,
  "inStock": true,
  "rating": 4.5,
  "reviewCount": 2847,
  "attributes": {
    "color": ["Siyah", "Beyaz"],
    "size": [36, 37, 38, 39, 40, 41, 42, 43]
  },
  "images": ["img1.jpg", "img2.jpg"]
}

// Arama: "nike ayakkabı" fiyat:500-1500 beden:42 inStock
GET /products/_search
{
  "query": {
    "bool": {
      "must": [{"multi_match": {"query": "nike ayakkabı", "fields": ["title^3", "description", "brand^2"]}}],
      "filter": [
        {"range": {"discountedPrice": {"gte": 500, "lte": 1500}}},
        {"term": {"attributes.size": 42}},
        {"term": {"inStock": true}}
      ]
    }
  },
  "sort": [{"_score": "desc"}, {"rating": "desc"}],
  "aggs": {
    "brands":    {"terms": {"field": "brand"}},
    "price_range": {"histogram": {"field": "discountedPrice", "interval": 250}}
  }
}
```

---

## Sepet Servisi

```
Sepet: geçici veri, hızlı okuma/yazma, kullanıcı başına

Redis Hash:
  Key: cart:{userId}
  Field: productId
  Value: {quantity, price_snapshot, addedAt}

  HSET cart:user-123 p-456 '{"qty":2,"price":999.99}'
  HGETALL cart:user-123
  HDEL cart:user-123 p-456
  EXPIRE cart:user-123 604800  (7 gün TTL)

Fiyat snapshot:
  Sepete eklerken fiyatı kaydet (kampanya bitince fiyat değişebilir)
  Ödeme aşamasında güncel fiyatla karşılaştır:
    Düştüyse → güncel (ucuz) fiyatı kullan
    Arttıysa → kullanıcıya bildir (fiyat değişti)

Misafir sepeti:
  Giriş yapmadan sepete ekle → sessionId ile
  Login olunca → session sepeti → kullanıcı sepetine merge et
```

---

## Stok Yönetimi (Overselling Önleme)

```
Problem: 100 kişi aynı anda son 1 ürünü satın almak istiyor

Çözüm 1: Pessimistic Lock (FOR UPDATE)
  BEGIN;
  SELECT stock FROM inventory WHERE product_id = ? FOR UPDATE;
  IF stock > 0:
    UPDATE inventory SET stock = stock - 1;
    INSERT INTO orders ...
    COMMIT;
  ELSE:
    ROLLBACK;

Çözüm 2: Redis Atomic Decrement (Önerilen — yüksek throughput)
  DECRBY inventory:p-456 1 → sonuç
  IF sonuç >= 0: OK → DB'yi async güncelle
  IF sonuç < 0: INCRBY inventory:p-456 1 → stok yok!

Çözüm 3: Pre-allocation (Flash Sale için)
  Satıştan önce: sepete ekle rezervasyonla
  10 dk içinde ödeme yapılmazsa → rezervasyon iptal → stok geri al
  "Sepetinize eklendi, 10 dk içinde ödeme yapın"

Rezervasyon:
  inventory_reservations: {reservationId, productId, quantity, expiresAt}
  Scheduler: Süresi dolan rezervasyonları iptal et → stok geri ver
```

---

## Sipariş Servisi (Order Service)

```
Sipariş oluşturma akışı:

1. POST /orders
   {cartId, shippingAddress, paymentMethodId}

2. Order Service:
   a. Cart validate → ürünler hâlâ müsait mi?
   b. Stok rezerve et (Inventory Service)
   c. Fiyat hesapla (kampanya, kupon)
   d. ORDER kaydı oluştur (status=PENDING)
   e. Outbox event: OrderCreated

3. Ödeme (Payment Service):
   Ödeme OK → ORDER status=CONFIRMED → Fulfillment başlat
   Ödeme FAIL → ORDER status=FAILED → stok rezervasyonu iptal

4. Fulfillment:
   Warehouse → paketleme → kargo
   Status: PROCESSING → SHIPPED → DELIVERED
```

```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    status          VARCHAR(30),     -- PENDING, CONFIRMED, SHIPPED, DELIVERED
    total_amount    DECIMAL(12,2),
    shipping_address JSONB,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_user_orders (user_id, created_at DESC)
);

CREATE TABLE order_items (
    order_id    UUID,
    product_id  UUID,
    quantity    INT,
    unit_price  DECIMAL(10,2),
    total_price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

---

## Flash Sale (Black Friday)

```
10x normal trafik → 100K req/sn

Öncesinde:
  Ürün sayfası statik → CDN cache (TTL 1 dk)
  Stok Redis'e yükle: SET flash:p-123:stock 1000
  Queue sistemi aktif et

Sırasında:
  Request → Queue (Redis List veya SQS)
  Queue worker → DECRBY flash:p-123:stock 1
  IF >= 0: sipariş oluştur
  IF < 0: "Stok tükendi"

Queue-based throttle:
  Kullanıcıyı bekleme kuyruğuna al
  "3. sıradasınız, ~30 saniye"
  → Gerçek sipariş QPS kontrollü (örn: 1000/sn)

Önceden kayıt:
  48 saat öncesinde "hatırlat" → push notification
  Sepete önceden ekle → ödeme sayfasına anlık yönlendir
```

---

## Kampanya ve Fiyatlandırma Motoru

```
Kampanya türleri:
  DISCOUNT:    %20 indirim
  BUY_X_GET_Y: 3 al 2 öde
  FREE_SHIPPING: 200 TL üzeri kargo bedava
  COUPON:      SUMMER2024 → %15 ekstra indirim

Fiyat hesaplama sırası:
  1. base_price
  2. category_discount (kategori kampanyası)
  3. product_discount (ürün kampanyası)
  4. coupon (kullanıcı kuponu)
  5. final_price = max(all discounts combined, min_price)

Rule engine (Drools veya custom):
  Kampanyalar öncelik sırasıyla uygulanır
  "En avantajlı tek kampanya" vs "tüm kampanyalar"

Fiyat geçmişi (yasal zorunluluk):
  Türkiye: "İndirim öncesi 30 günün en düşük fiyatı" gösterilmeli
  price_history tablosu: {productId, price, validFrom, validTo}
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Arama | DB LIKE | Elasticsearch | ES (full-text, facet) |
| Sepet | DB | Redis | Redis (geçici, hızlı) |
| Stok lock | FOR UPDATE | Redis DECR | Hybrid (normal=Redis, flash=queue) |
| Fiyat hesap | DB stored proc | Rule engine | Rule engine (esnek) |
| Flash sale | Normal flow | Queue-based | Queue (controlled load) |
