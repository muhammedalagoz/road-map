# 08r — System Design: E-Ticaret Platformu (Amazon)

## Gereksinimler

```
Functional:
  ✓ Ürün kataloğu ve arama
  ✓ Sepet yönetimi
  ✓ Sipariş oluşturma ve takip
  ✓ Stok yönetimi (overselling önleme)
  ✓ Fiyat ve kampanya yönetimi
  ✓ Ürün öneri motoru (recommendation)
  ✗ Ödeme, teslimat (out of scope)

Non-Functional:
  DAU: 10M
  Ürün: 100M ürün
  Peak: Black Friday → 10x normal trafik
  Ürün sayfası latency: < 200ms
  Sipariş oluşturma: < 3s
  Availability: 99.99%
  Stok tutarsızlığı: sıfır tolerans (oversell = gelir kaybı + itibar)
```

---

## Capacity Estimation

```
Trafik:
  10M DAU × 20 sayfa görüntüleme = 200M istek/gün
  200M / 86,400 ≈ 2,300 QPS ortalama
  Peak (Black Friday): 2,300 × 10 = 23,000 QPS

  Sipariş QPS: 10M × 0.05 (satın alma oranı) / 86,400 ≈ 6 sipariş/sn
  Peak sipariş: 6 × 10 = 60 sipariş/sn

Storage:
  Ürün metadata: 100M × 2 KB = 200 GB (PostgreSQL)
  Ürün görseli: 100M × 5 görsel × 500 KB = 250 TB (S3 + CDN)
  Elasticsearch index: 100M × 1 KB = 100 GB
  Sipariş (5 yıl): 6/sn × 86,400 × 365 × 5 × 500B ≈ 475 GB

Cache:
  Aktif ürün (hot): %1 × 100M = 1M ürün × 2KB = 2 GB Redis
  Sepet: 10M DAU × 1 KB = 10 GB Redis
  Session: 10M × 500 B = 5 GB Redis

Elasticsearch:
  100M ürün × 1 KB = 100 GB index
  3 shard × 2 replica = 6 node (16 GB RAM/node) → yeterli
```

---

## Microservice Mimarisi

```
          Client (Web/Mobile)
                │
         ┌──────┴──────────┐
         │   API Gateway    │  ← Rate limiting, Auth, SSL termination
         │   BFF (Web/Mob)  │  ← Web BFF: tam veri, Mob BFF: lightweight
         └──────┬───────────┘
                │
    ┌───────────┼──────────────────────────────────┐
    ▼           ▼          ▼         ▼       ▼     ▼
[Product    [Search    [Cart     [Order   [Inventory [Campaign
 Service]   Service]   Service]  Service]  Service]  Service]
    │           │          │         │        │          │
[PostgreSQL [Elastic   [Redis]  [PostgreSQL] [PostgreSQL [PostgreSQL
 + Redis]   search]             + Outbox]   + Redis]    + Redis]
    │
    ▼
[CDC/Debezium] → Kafka → [ES Indexer] → Elasticsearch
                        → [Cache Invalidator] → Redis
```

---

## Ürün Kataloğu Servisi

```
100M ürün → tek tablo imkansız, çok katmanlı veri yönetimi

Veri katmanları:
  PostgreSQL:     master veri (id, name, price, sellerId, stock_status)
  Elasticsearch:  arama ve filtreleme indexi
  Redis (L1):     hot ürün cache (son 24 saatte 100+ görüntüleme)
  S3 + CDN:       ürün görselleri (immutable URL, içerik hash)

Ürün görüntüleme QPS:
  23,000 QPS peak
  Cache hit rate %95 → DB: 1,150 QPS (yönetilebilir)
  Cache miss: Redis populate → DB yükü minimal

Çok dilli katalog:
  product_translations(product_id, lang, title, description)
  Kullanıcı dili → doğru translation → Elasticsearch locale-aware index
```

```java
@Service
class ProductService {

    Product getProduct(String productId, String lang) {
        String cacheKey = "product:" + productId + ":" + lang;

        // L1: Redis cache
        Product cached = redis.get(cacheKey, Product.class);
        if (cached != null) return cached;

        // L2: DB (master data + translation join)
        Product product = productRepo.findByIdWithTranslation(productId, lang)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // Cache süresi: hot ürün → 1 saat, normal → 5 dk
        Duration ttl = viewCountService.isHot(productId)
            ? Duration.ofHours(1)
            : Duration.ofMinutes(5);
        redis.setEx(cacheKey, product, ttl);

        return product;
    }

    // Fiyat/stok değişikliğinde cache invalidate
    @EventListener
    void onProductUpdated(ProductUpdatedEvent event) {
        // Tüm dil versiyonlarını invalidate
        supportedLangs.forEach(lang ->
            redis.delete("product:" + event.getProductId() + ":" + lang)
        );
        // Elasticsearch re-index (async)
        searchIndexer.reindex(event.getProductId());
    }
}
```

---

## Katalog-Elasticsearch Senkronizasyonu (CDC)

```
Problem: PostgreSQL'deki ürün değişiklikleri ES'e nasıl yansır?

Naif yöntem (çift yazma — dual write):
  Product Service → PostgreSQL yaz → Elasticsearch'e yaz.
  ✗ Atomik değil: Postgres başarılı, ES başarısız → tutarsızlık.
  ✗ Transaction yok: crash aralarında olursa → veri kaybı.

Doğru yöntem (Change Data Capture — Debezium):
  Debezium: PostgreSQL WAL (Write-Ahead Log) → her değişikliği yakalar.
  WAL event → Kafka topic (product.changes).
  ES Indexer consumer: Kafka → Elasticsearch upsert.

  Akış:
  Product Service → PostgreSQL UPDATE → WAL log güncellendi.
  Debezium (PostgreSQL Connector) → WAL okur → Kafka'ya publish:
    {
      "op": "u",          // update
      "before": {"price": 999, ...},
      "after":  {"price": 799, ...},
      "table": "products"
    }

  ES Indexer (Kafka consumer):
    Upsert: productId → ES belgesi güncelle.
    Batch: 500 event → bulk API → ES (çok daha verimli).

  Cache Invalidator (ayrı consumer):
    Aynı Kafka event → Redis key sil → sonraki istek DB'den.

Neden CDC üstün:
  Exactly-once (Kafka idempotent consumer): ES tutarlı.
  Gecikmeli ama güvenilir: ES 1-2 sn geride → kabul edilebilir.
  Replay: ES cluster yeniden kurulursa Kafka'dan replay → sıfırdan index.
  Decoupled: Product Service ES'i bilmez → bağımlılık yok.

Elasticsearch index yönetimi:
  Blue-green indexing: yeni mapping → yeni index → alias switch → sıfır downtime.
  Index alias: products_alias → products_v2 (production),
                                products_v3 (yeni, hazırlanıyor).
  Switch: POST /_aliases (atomic) → v3 aktif, v2 eski.
```

---

## Arama (Elasticsearch) — Detaylı

```json
// Ürün index mapping
{
  "mappings": {
    "properties": {
      "productId":       {"type": "keyword"},
      "title":           {"type": "text", "analyzer": "turkish",
                          "fields": {"keyword": {"type": "keyword"}}},
      "description":     {"type": "text", "analyzer": "turkish"},
      "brand":           {"type": "keyword"},
      "category":        {"type": "keyword"},
      "price":           {"type": "double"},
      "discountedPrice": {"type": "double"},
      "discountRate":    {"type": "integer"},
      "inStock":         {"type": "boolean"},
      "rating":          {"type": "float"},
      "reviewCount":     {"type": "integer"},
      "soldCount":       {"type": "integer"},
      "attributes":      {"type": "nested"},
      "sellerId":        {"type": "keyword"},
      "createdAt":       {"type": "date"}
    }
  }
}
```

```json
// Gelişmiş arama sorgusu: "nike koşu ayakkabısı" filtreli + sıralı + facet
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [{
            "multi_match": {
              "query": "nike koşu ayakkabısı",
              "fields": ["title^4", "brand^3", "category^2", "description"],
              "type": "best_fields",
              "fuzziness": "AUTO"   // yazım hatası toleransı
            }
          }],
          "filter": [
            {"range": {"discountedPrice": {"gte": 500, "lte": 2000}}},
            {"term":  {"attributes.size": 42}},
            {"term":  {"inStock": true}},
            {"term":  {"category": "Ayakkabı"}}
          ]
        }
      },
      "functions": [
        {"field_value_factor": {"field": "rating",     "factor": 1.2, "modifier": "sqrt"}},
        {"field_value_factor": {"field": "soldCount",  "factor": 0.8, "modifier": "log1p"}},
        {"field_value_factor": {"field": "reviewCount","factor": 0.5, "modifier": "log1p"}}
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  },
  "sort": [
    {"_score": "desc"},
    {"discountRate": "desc"},
    {"rating": "desc"}
  ],
  "aggs": {
    "brands":      {"terms": {"field": "brand", "size": 20}},
    "categories":  {"terms": {"field": "category"}},
    "price_range": {"histogram": {"field": "discountedPrice", "interval": 250}},
    "avg_rating":  {"avg": {"field": "rating"}},
    "in_stock_count": {"filter": {"term": {"inStock": true}}}
  },
  "highlight": {
    "fields": {"title": {}, "description": {"fragment_size": 150}}
  },
  "from": 0,
  "size": 24
}
```

```
Arama önerileri (Autocomplete):
  ES completion suggester veya search-as-you-type field.
  "nik" → ["Nike Air Max", "Nike React", "Nike Free Run"]
  Prefix match → edge n-gram analyzer.

Yazım hatası düzeltme:
  fuzziness: AUTO → 1 karakter hata tolere (5+ kelimede 2).
  "nkie" → "nike" → sonuçlar gelir.

Synonym:
  "laptop" = "dizüstü", "notebook"
  ES synonym filter → mapping'e ekle.
```

---

## Sepet Servisi

```
Sepet: geçici veri, yüksek okuma/yazma, kullanıcı başına izole

Redis Hash:
  Key:   cart:{userId}
  Field: {productId}:{variantId}
  Value: JSON {quantity, priceSnapshot, addedAt, sellerId}

  HSET cart:user-123 "p-456:v-red-42" '{"qty":2,"price":999.99,"addedAt":"..."}'
  HGETALL cart:user-123       → tüm ürünler
  HSET ... qty güncelle       → miktar değiştir
  HDEL cart:user-123 "p-456:v-red-42" → çıkar
  EXPIRE cart:user-123 604800         → 7 gün TTL

Fiyat snapshot politikası:
  Sepete eklerken: o anki discountedPrice snapshot'ı al.
  Ödeme aşamasında: anlık fiyatla karşılaştır.
    Düştüyse   → kullanıcı lehine güncel (ucuz) fiyat uygula.
    Arttıysa   → "Fiyat değişti: eski 999₺, yeni 1199₺" uyarısı.
    Stok bitti → "Bu ürün stokta kalmadı" uyarısı.

Misafir sepeti (guest cart):
  sessionId → cart:{sessionId} Redis'te.
  Login olunca: cart:{sessionId} → cart:{userId} merge.
  Çakışma: aynı ürün her ikisinde → qty topla veya kullanıcıya sor.
  TTL: misafir sepeti 24 saat, giriş yapılmış 7 gün.

Sepet validasyon (checkout öncesi):
  Her ürün için:
    ✓ Ürün hâlâ mevcut mu? (status=ACTIVE)
    ✓ Fiyat değişti mi? (snapshot vs güncel)
    ✓ Stok yeterli mi? (qty <= mevcut stok)
    ✓ Satıcı aktif mi?
  Sorun varsa → kullanıcıya önce göster → onay al → sonra sipariş.
```

---

## Stok Yönetimi — Overselling Önleme

```
Problem: 100 kişi aynı anda son 1 ürünü satın almak istiyor.

Katman 1 — Redis Atomic Decrement (yüksek throughput):
  Stok Redis'e yükle: SET inventory:p-456 100
  Sipariş gelince:
    result = DECRBY inventory:p-456 1
    IF result >= 0: ön onay → DB akışına devam
    IF result < 0:  INCRBY inventory:p-456 1 → stok yok!
  
  Avantaj: sub-ms → 100K req/sn kapasitesi.
  Dezavantaj: Redis crash → stok sayacı kaybolabilir.
  Çözüm: Redis + DB her zaman senkronize (CDC ile).

Katman 2 — DB Optimistic Lock (doğrulama):
  UPDATE inventory
  SET quantity = quantity - 1, version = version + 1
  WHERE product_id = ? AND quantity >= 1 AND version = ?
  -- Etkilenen satır = 0 → başka biri aldı → retry veya "stok yok"

  Neden optimistic: çoğu sipariş başarılı → lock bekleme yok.
  Pessimistic (FOR UPDATE): yüksek trafikte deadlock riski + yavaş.

Katman 3 — Rezervasyon (Flash Sale / Sepet tutma):
  Sepete ekleme → stok rezerve et (soft reserve):
    INSERT INTO inventory_reservations
      (reservation_id, product_id, quantity, user_id, expires_at)
    VALUES (uuid, p-456, 1, user-123, NOW() + INTERVAL '10 minutes')
  
  Aynı anda gerçek stok = toplam stok - aktif rezervasyon sayısı.
  10 dk içinde ödeme yoksa → scheduler → rezervasyon iptal → stok geri.
  Scheduler (Quartz/Redis ZSET TTL): her dakika süresi dolan rezervasyonu temizle.

  Kullanıcıya mesaj: "Sepetinizde tutuluyor: 09:47'ye kadar"
  Countdown timer: UX'te aciliyet hissi + sistemi koruma.

Stok senkronizasyonu (Redis ↔ DB):
  Güven: DB master.
  Redis: cache (hız).
  Her sipariş: DB güncellendi → CDC → Redis invalidate → yeniden yükle.
  Günde 1 kez: reconciliation job → DB stok ≠ Redis stok → alarm + sync.
```

---

## Sipariş Saga Pattern

```
Problem: Sipariş oluşturma birden fazla servisi kapsıyor.
  Stok rezerve et → Ödeme al → Sipariş onayla → Fulfillment başlat
  Herhangi bir adım başarısız → geri al (compensating transaction).

Orchestration Saga (Order Service yönetir):

  ┌──────────────┐
  │ Order Service │ (Saga Orchestrator)
  └──────┬───────┘
         │ 1. Reserve Stock
         ▼
  ┌──────────────────┐
  │ Inventory Service│ → OK: reserved / FAIL: out_of_stock
  └──────────────────┘
         │ 2. Process Payment
         ▼
  ┌──────────────────┐
  │ Payment Service  │ → OK: charged / FAIL: payment_declined
  └──────────────────┘
         │ 3. Create Shipment
         ▼
  ┌──────────────────┐
  │ Fulfillment Svc  │ → OK: shipment_created
  └──────────────────┘

Compensating transactions (başarısız adım → geri al):
  Payment FAIL:
    → Inventory: rezervasyonu iptal et (compensation).
    → Order: status = FAILED.

  Fulfillment FAIL (nadir):
    → Payment: iade et (refund).
    → Inventory: stok geri ver.
    → Order: status = FAILED.

Outbox Pattern (at-least-once delivery):
  Order Service → PostgreSQL transaction:
    1. orders tablosuna INSERT (status=PENDING).
    2. outbox tablosuna INSERT (event=OrderCreated).
    — Aynı transaction → atomik.

  Outbox publisher (ayrı process):
    Outbox tablosunu polling: yeni event var mı?
    → Kafka'ya publish → event gönderildi olarak işaretle.

  CREATE TABLE outbox (
    event_id    UUID PRIMARY KEY,
    event_type  VARCHAR(50),    -- OrderCreated, StockReserved, etc.
    payload     JSONB,
    created_at  TIMESTAMPTZ,
    published   BOOLEAN DEFAULT FALSE,
    published_at TIMESTAMPTZ
  );

Idempotency (duplicate event koruması):
  Her event: idempotency_key = orderId + eventType.
  Downstream servis: aynı key gelirse → ignore (Redis Set ile dedupe).
  Ödeme servisi: aynı orderId için iki kez ödeme alma → dedupe → bir kez al.
```

```java
@Transactional
public Order createOrder(CreateOrderRequest req) {
    // 1. Validate cart
    Cart cart = cartService.getAndValidate(req.getUserId());

    // 2. Create order (PENDING)
    Order order = Order.builder()
        .orderId(UUID.randomUUID())
        .userId(req.getUserId())
        .status(OrderStatus.PENDING)
        .items(cart.getItems())
        .totalAmount(cart.getTotalAmount())
        .build();
    orderRepo.save(order);

    // 3. Outbox event — SAME transaction (atomik!)
    outboxRepo.save(OutboxEvent.builder()
        .eventId(UUID.randomUUID())
        .eventType("OrderCreated")
        .payload(objectMapper.writeValueAsString(order))
        .build());

    // 4. Cart temizle
    cartService.clear(req.getUserId());

    return order;
    // Transaction commit → hem order hem outbox event kaydedildi.
    // Outbox publisher → Kafka → diğer servisler.
}
```

---

## Flash Sale (Black Friday)

```
10x normal trafik → 23,000 × 10 = 230,000 req/sn peak

Hazırlık (T-48 saat):
  Ürün sayfaları → CDN cache (TTL 1 dk: stok bilgisi canlı değil ama hız).
  Elasticsearch: flash sale ürünleri → ayrı index (öncelikli cache).
  Stok → Redis'e yükle: SET flash:{productId}:stock 1000
  Queue sistemi aktif et (bekletme ekranı hazır).
  Load test: 10x trafik → darboğaz tespit.

Ürün sayfası (T-0, satış başladı):
  Request → CDN (static page cache).
  "Stok: var/yok" → sadece bu kısım canlı (partial cache).
  SSE veya WebSocket: stok sayacı gerçek zamanlı güncelleme.

Sipariş akışı (rate-limited):
  Request → API Gateway → rate limit: kullanıcı başına 1 sipariş/dakika.
  → Queue (Redis List veya SQS): kullanıcı kuyruğa eklendi.
  → "3. sıradasınız, ~15 saniye" mesajı.
  → Queue worker: 1,000 sipariş/sn hızında işle.
  → DECRBY flash:{productId}:stock 1 → >= 0: devam, < 0: stok bitti.

Waiting Room (Cloudflare Queue-it benzeri):
  İlk N kullanıcı (örn. 10,000): doğrudan sitede.
  Sonrakiler: waiting room → sıra numarası → kademeli kabul.
  Bot koruması: CAPTCHA ilk girişte.

Önceden kayıt sistemi:
  48 saat öncesinde "hatırlat" → push notification.
  Kullanıcı önceden kayıt → ürünü sepetine ekle (rezervasyon yok, sadece pre-fill).
  Satış başlayınca: "Tek tık ile satın al" → ödeme sayfasına direkt.

İptal ve stok geri verme:
  10 dk içinde ödeme yapılmamış rezervasyon → scheduler iptal → stok geri.
  Geri dönen stok → hemen satışa açılmaz → 5 dk bekle (bot önleme).
  İkinci şans bildirimi: "Stok güncellendi" → waitlist kullanıcılarına push.
```

---

## Kampanya ve Fiyatlandırma Motoru

```
Kampanya türleri:
  DISCOUNT:      %20 indirim (ürün/kategori bazlı)
  BUY_X_GET_Y:  3 al 2 öde
  BUNDLE:        "Ayakkabı + Çorap" birlikte %15 indirim
  FREE_SHIPPING: 200₺ üzeri kargo bedava
  COUPON:        SUMMER2024 → %15 ekstra (tek kullanımlık)
  CASHBACK:      Ödeme sonrası puan iade
  FLASH:         30 dk için %50 indirim

Fiyat hesaplama sırası (Priority Stack):
  1. base_price                    (catalog price)
  2. seller_discount               (satıcı indirimi)
  3. category_discount             (kategori kampanyası)
  4. product_discount              (ürün özel kampanyası)
  5. coupon                        (kullanıcı kuponu — tek kullanımlık)
  6. loyalty_discount              (sadakat puanı)
  7. final_price = max(kombinasyon, min_floor_price)

  Kural: kampanyalar çakışırsa → "en avantajlı" veya "tüm geçerli kampanyalar"
         satıcı konfigürasyonuna göre değişir.

Rule Engine (Drools / Camunda / custom):
  Kampanya kuralları DB'de saklanır (no code deploy).
  Pazarlama ekibi: dashboard → yeni kampanya → canlıya al.
  A/B test: aynı ürün → iki farklı kampanya → hangisi daha iyi satıyor?

Fiyat geçmişi (yasal zorunluluk — AB/Türkiye):
  "30 günlük en düşük fiyat" gösterimi zorunlu.
  price_history(product_id, price, effective_from, effective_to).
  Her fiyat değişikliğinde: yeni satır INSERT.
  Gösterim: "İndirim öncesi fiyat: 1299₺ (son 30 günün en yüksek fiyatı değil, en düşüğü)."

Kampanya kötüye kullanım önleme:
  COUPON: user_id + coupon_code → unique constraint (tek kullanım).
  Cashback: aynı kart ile çok hesap → fraud detection.
  Flash sale: kullanıcı başına max 2 adet limit.
```

---

## Öneri Motoru (Recommendation)

```
Öneri türleri:
  "Bu ürünü alanlar şunu da aldı" (collaborative filtering)
  "Son görüntülediklerinize benzer" (content-based)
  "Sıkça birlikte alınan ürünler" (association rules)
  "Kişisel ana sayfa feed" (hybrid)

Collaborative Filtering (offline batch, Spark ML):
  Input: user_id, product_id, action (view=1, cart=3, purchase=5) → implicit feedback.
  ALS (Alternating Least Squares) matrix factorization.
  Çıktı: her kullanıcı için top-100 ürün önerisi.
  Batch: günde 1 kez (veya 6 saatte bir) Spark job → sonuçlar Redis'e.

  // Redis: öneri sonuçları
  ZADD recommendations:user-123 0.92 "p-456" 0.88 "p-789" ...
  ZREVRANGE recommendations:user-123 0 9 WITHSCORES → top-10

Item-Based Similarity (near-realtime, Kafka):
  "Bu ürünü görüntüleyen X ürününü de gördü."
  Co-occurrence matrix: görüntüleme eventlerinden.
  Kafka Streams: sliding window (1 saat) → aynı session'da görüntülenen çiftler.
  ES: more_like_this query → anlık benzerlik.

Frequently Bought Together:
  Sipariş verilerinden: aynı siparişte hangi ürünler bir arada?
  Apriori / FP-Growth algoritması.
  Batch: günlük → product_associations tablosu.
  Gösterim: "Bununla sıkça alınan: Çorap, Ayak Bezi"

Öneri servisinin mimarisi:
  Request → Recommendation Service → Redis (precomputed, <5ms).
  Redis miss → Fallback: ES benzerlik sorgusu (200ms, real-time).
  Cold start (yeni kullanıcı): popüler ürünler + kategori bazlı trending.
  Cold start (yeni ürün): content-based (kategori + marka + attribute benzerliği).
```

---

## CDN & Medya Yönetimi

```
250 TB ürün görseli → S3 + CDN zorunlu

Upload akışı (satıcı ürün ekliyor):
  1. Satıcı → Media Service: POST /media/upload-url
  2. Media Service → S3 pre-signed URL (5 dk geçerli).
  3. Satıcı → S3: PUT (direkt, Media Service bypass).
  4. S3 event → Lambda → thumbnail üret (200x200, 400x400, 800x800).
  5. Thumbnails → S3/CloudFront → CDN'e cache.

URL yapısı (immutable, content-addressed):
  https://cdn.myshop.com/products/{productId}/{hash}/{size}.webp
  Hash: SHA-256(original content) → içerik değişirse URL değişir.
  CDN: max-age=31536000 (1 yıl cache) → immutable.
  Silme: soft delete (DB'de flag) → CDN'den expire edilir.

Format optimizasyonu:
  WebP: JPEG'e göre %30 küçük, aynı kalite.
  AVIF: WebP'ye göre daha küçük ama eski tarayıcı desteği yok.
  Adaptive: Accept header → WebP destekleniyorsa WebP, yoksa JPEG.
  Lazy loading: viewport'ta görünene kadar yükleme yok.

Image SEO:
  Alt text: ürün adı + marka (crawler için).
  Structured data: JSON-LD → Google Shopping index.

Maliyet optimizasyonu:
  S3 Intelligent Tiering: çok erişilen → Standard, az erişilen → Glacier.
  Görseller 2 yıl sonra Glacier → %70 maliyet azaltma.
  CDN hit oranı hedef: %98+ → S3 egress maliyet minimal.
```

---

## Sipariş Takip & Durum Yönetimi

```sql
-- PostgreSQL: sipariş durumu FSM (Finite State Machine)
CREATE TABLE orders (
    order_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID NOT NULL,
    status           order_status NOT NULL DEFAULT 'PENDING',
    total_amount     DECIMAL(12,2) NOT NULL,
    discount_amount  DECIMAL(12,2) DEFAULT 0,
    shipping_amount  DECIMAL(12,2) DEFAULT 0,
    shipping_address JSONB NOT NULL,
    idempotency_key  VARCHAR(100) UNIQUE,  -- duplicate order önleme
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    updated_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE order_status AS ENUM (
    'PENDING',      -- oluşturuldu, ödeme bekleniyor
    'CONFIRMED',    -- ödeme alındı
    'PROCESSING',   -- depoda hazırlanıyor
    'SHIPPED',      -- kargoya verildi
    'DELIVERED',    -- teslim edildi
    'CANCELLED',    -- iptal edildi
    'REFUNDED'      -- iade edildi
);

CREATE TABLE order_items (
    order_id     UUID REFERENCES orders(order_id),
    product_id   UUID NOT NULL,
    variant_id   UUID,
    seller_id    UUID NOT NULL,
    quantity     INT NOT NULL CHECK (quantity > 0),
    unit_price   DECIMAL(10,2) NOT NULL,
    discount_amt DECIMAL(10,2) DEFAULT 0,
    total_price  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id, variant_id)
);

CREATE TABLE order_status_history (
    id          BIGSERIAL PRIMARY KEY,
    order_id    UUID REFERENCES orders(order_id),
    old_status  order_status,
    new_status  order_status NOT NULL,
    reason      TEXT,
    changed_by  VARCHAR(50),  -- system, user, admin
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);
-- Sipariş geçmişi: tüm durum geçişleri + kim/ne zaman değiştirdi.

-- Etkin arama indeksleri
CREATE INDEX idx_orders_user    ON orders (user_id, created_at DESC);
CREATE INDEX idx_orders_status  ON orders (status) WHERE status NOT IN ('DELIVERED','CANCELLED');
CREATE INDEX idx_orders_idem    ON orders (idempotency_key);
```

---

## Olası Sorunlar ve Çözümleri

### 1. Overselling — Redis Çöktü, Stok Negatife Düştü

```
Sorun:
  Stok Redis'te: inventory:p-456 = 10.
  Redis master çöktü → replica promote (30sn).
  Bu 30 sn: bazı istekler Redis'e ulaşamadı → DB'ye fallback.
  DB: FOR UPDATE → 15 sipariş aynı anda → sadece 10 geçmeli ama 12 geçti.
  Sebeep: failover sırasında iki farklı veri kaynağı → tutarsızlık.

Çözüm:
  a) Redis Cluster + Sentinel (HA):
     3 node: 1 master + 2 replica. Sentinel: master down → replica promote (~30sn).
     Bu 30sn: circuit breaker → DB katmanına geç (pessimistic lock).
     Redis geri gelince: DB'den stok yeniden yükle.

  b) Redis Lua script (atomic):
     local stock = tonumber(redis.call('GET', KEYS[1]))
     if stock <= 0 then return 0 end
     redis.call('DECR', KEYS[1])
     return 1
     -- GET + DECR atomik → race condition yok.

  c) DB level son güvence:
     UPDATE inventory SET quantity = quantity - 1
     WHERE product_id = ? AND quantity > 0
     -- Etkilenen satır = 0 → stok yok → Redis tutarsız → DB doğru.
     Reconciliation: her 5 dk → DB stok → Redis sync.

  d) Oversell tespit + acil müdahale:
     Monitoring: Redis stok < 0 → Slack alert → engineering on-call.
     Telafi: fazla sipariş alan → sipariş iptal + özür kupon.
     SLA: oversell tespitinden 1 saat içinde müşteri bilgilendirilmeli.
```

---

### 2. Flash Sale Anında Site Çöktü — Thundering Herd

```
Sorun:
  Büyük flash sale duyurusu: "Saat 12:00'de %70 indirim!"
  12:00:00.000: 500,000 kullanıcı aynı anda site açtı.
  API Gateway: 500K req/sn → rate limit konfigürasyonu yetmedi.
  DB connection pool: doldu → yeni bağlantı reddedildi.
  Uygulama: timeout → 503 → kullanıcılar sayfayı yeniledi → daha da kötü.
  Sonuç: 10 dk çökme → milyonlarca TL gelir kaybı.

Çözüm:
  a) CDN ile statik ürün sayfası:
     Flash sale ürün sayfası → CDN'e önceden cache.
     12:00'da: %95 istek CDN'den → DB'ye hiç gitmiyor.
     Sadece "stok var mı" API çağrısı canlı (lightweight).

  b) Waiting Room (Virtual Queue):
     İlk 10,000 kullanıcı → doğrudan geçer.
     Sonrakiler → waiting room sayfası (CDN'den, statik).
     Kademeli kabul: her 5 sn → 1,000 kullanıcı daha.
     Kullanıcı görür: "Sıra: 15,432 — Tahmini bekleme: 2 dk."
     Sistem yükü: sabit ve kontrollü.

  c) Auto-scaling önceden:
     CloudWatch / Prometheus alert: "Flash sale 1 saat sonra."
     Scheduled scaling: 11:00 → min instance 5x artır.
     Load test: 11:30 → 10x yük testi → bottleneck tespiti.

  d) Graceful degradation:
     DB aşırı yük → öneri servisi kapat (low priority).
     Review/rating servisi kapat.
     Sadece kritik path: ürün görüntüle, sepete ekle, ödeme → açık.
     Feature flag: anlık kapatma (Unleash/LaunchDarkly).
```

---

### 3. Elasticsearch ile PostgreSQL Tutarsızlığı — Stok Bitti Ama Arama "Var" Gösteriyor

```
Sorun:
  Ürün stoku bitti → PostgreSQL güncellendi (inStock=false).
  Debezium → Kafka → ES Indexer → ES güncellendi → 2-3 sn gecikme.
  Bu sürede: kullanıcı arama yaptı → ES "inStock=true" döndü → ürün sayfasına gitti.
  Ürün sayfası: "Stok yok" → hayal kırıklığı + kötü UX.
  Büyük flash sale'de: 100,000 kullanıcı stoksuz ürüne yönlendirildi.

Çözüm:
  a) ES lag'ı kabul et + ürün sayfasında kontrol:
     Arama sonucu "inStock=true" gösterebilir (stale).
     Ama ürün sayfasında: anlık DB kontrol → "Stok tükendi."
     Kullanıcı deneyimi: 1 ekstra tıklama → kabul edilebilir.

  b) Near-realtime stok güncellemesi:
     Stok değişti → doğrudan ES partial update (async, Kafka bypass):
     POST /products/_update/{id}
     {"doc": {"inStock": false, "stockCount": 0}}
     Sadece stok alanı → hızlı (tam re-index değil).
     Gecikme: 2-3sn → 200-500ms.

  c) Redis + ES hybrid stok kontrolü:
     Arama: ES sorgusu (hız + relevance).
     Stok bilgisi: ES'ten değil, Redis'ten (inventory:p-xxx).
     Response: ES sonuçları + Redis stok bilgisi merge.
     Gecikme: sıfır (Redis sub-ms).

  d) Flash sale için özel handling:
     Flash sale ürünleri: ayrı ES index (flash_products).
     Stok bitti → anında index'ten sil (ES delete immediate).
     Kullanıcı: artık arama sonuçlarında görmez.
```

---

### 4. Sipariş Saga Yarım Kaldı — Ödeme Alındı Ama Stok Rezerve Edilemedi

```
Sorun:
  Saga: Stok rezerve et → Ödeme al → Sipariş onayla.
  Ödeme alındı (Payment Service: success).
  Stok rezervasyonu timeout: Inventory Service 3sn cevap vermedi.
  Order Service: "Stok rezerve edilemedi" → siparişi iptal etti.
  Sonuç: kullanıcının parası çekildi, sipariş yok → müşteri şikayeti.

Çözüm:
  a) Saga sırası önemli:
     YANLIŞ sıra: Ödeme al → Stok rezerve et (ödeme sonra başarısız olabilir).
     DOĞRU sıra: Stok rezerve et → Ödeme al.
     İlk başarısız: ödeme hiç alınmadı → basit compensation.

  b) Compensating transaction zinciri:
     Inventory timeout → retry (3 kez, exponential backoff).
     3 retry sonra fail → Saga COMPENSATE:
       Ödeme alındıysa → iade et (Payment.refund).
       Stok rezerve edildiyse → geri ver (Inventory.release).
     Compensation da başarısız olursa → "pending_compensation" queue.
     Manual intervention queue: ops ekibi manuel çözüm.

  c) Idempotency her adımda:
     Her Saga adımı → idempotency_key = orderId + stepName.
     Tekrar denenirse → aynı sonucu döndür, duplicate işlem yok.
     Ödeme servisinde: orderId ile iki kez ödeme alma yok.

  d) Saga state machine monitoring:
     Her sipariş Saga state'i Cassandra'da izle.
     Stuck saga: 10 dk içinde tamamlanmadıysa → alert.
     Dashboard: "15 sipariş PENDING_COMPENSATION" → ops müdahalesi.
```

---

### 5. Fiyat Yarışması — İki Kullanıcı Aynı Kuponu Kullandı

```
Sorun:
  Kupon: SUMMER2024 → tek kullanımlık.
  İki kullanıcı (Ali ve Mehmet) neredeyse aynı anda kullandı.
  Her ikisi de checkout yapınca: kupon geçerli göründü → ikisi de indirim aldı.
  Race condition: iki istek paralel → kupon kullanıldı mı kontrolü → ikisi de hayır.

Çözüm:
  a) DB unique constraint + optimistic lock:
     coupon_usages(coupon_id, user_id) → unique.
     Ali: INSERT → success.
     Mehmet: INSERT → duplicate key → "kupon kullanılmış."

  b) Redis atomic SET NX (hızlı kontrol):
     SET coupon:used:SUMMER2024:user-123 "1" NX EX 3600
     NX: yoksa set et. Zaten varsa → 0 döner → "zaten kullandınız."
     Ön kontrol → DB yüküne gitmeden hızlı red.

  c) Global limit (tüm kullanıcılar):
     Kupon: max 1000 kullanım.
     Redis: INCR coupon:count:SUMMER2024
     count > 1000 → "kupon tükendi."
     Atomic → oversell yok.

  d) Audit:
     Her kupon kullanımı log: kim, ne zaman, hangi sipariş.
     Fraud detection: aynı IP, farklı hesaplar, aynı kupon → şüpheli.
```

---

### 6. Cache Stampede — Popüler Ürün Cache'den Silindi, DB Çöktü

```
Sorun:
  Black Friday saat 11:59 → ürün cache TTL doldu (5 dk TTL, yanlış zamanda).
  12:00 → 50,000 req/sn ürün sayfasına.
  Cache miss → hepsi DB'ye → DB bağlantı havuzu doldu → timeout.
  Cache doldurulamdı (DB cevap veremedi) → döngü.

Çözüm:
  a) Cache-aside + mutex (thundering herd prevention):
     Cache miss → Redis SET lock:product:{id} NX EX 5.
     Lock aldı → DB sorgu → cache doldur → lock sil.
     Lock alamadı → 100ms bekle → cache'i tekrar kontrol.
     Sadece 1 thread DB'ye gider → stampede yok.

  b) Stale-while-revalidate:
     Cache TTL doldu → eski veriyi dön (stale, anlık).
     Arka planda: async DB sorgusu → cache yenile.
     Kullanıcı: stale veri görür ama hızlı response alır.
     2 sn sonra: güncel veri.
     E-ticaret'te: fiyat 2 sn stale olabilir (tolere edilebilir).

  c) Jitter + uzun TTL:
     TTL = 3600 + random(0, 600) → tüm hot product aynı anda expire olmaz.
     Hot product: TTL 24 saat → flash sale süresince kesinlikle expire olmaz.
     Invalidation: fiyat/stok değişince explicit delete.

  d) Dedicated cache server (hot product):
     Top 1000 ürün → ayrı Redis cluster (daha büyük, daha hızlı).
     Bu ürünlerin cache'i asla expire olmaz → sadece explicit invalidation.
```

---

### 7. Yavaş Arama — 100M Ürün Elasticsearch'i Yavaşlattı

```
Sorun:
  100M ürün → ES index 100 GB.
  Arama sorgusu: function_score + bool + aggs → 3-5 sn.
  Kullanıcı: arama sonuçları 5 sn geliyor → %40 kullanıcı sayfayı terk etti.

Neden yavaş:
  function_score: tüm dökümanlar scoring → 100M × hesaplama.
  Aggregation: brand, category, price_range → tüm index tarama.
  Shard sayısı az → her shard çok veri → paralel işleme az.

Çözüm:
  a) Index sharding (düzgün):
     100M ürün / 5 primary shard = 20M/shard → optimal (Lucene performansı).
     2 replica → okuma paralel (3 kopya).
     10 node: her node ~2 shard → paralel arama.

  b) Filter önce, score sonra:
     Bool filter (cache'lenir): inStock=true, category=ayakkabı → küçük set.
     function_score: sadece filtreden geçenlere → 100M değil 10K üzerinde.
     ES filter cache: aynı filter → sonraki sorguda cache hit.

  c) Pre-computed relevance (offline):
     Batch job: her ürün için popularity_score = (soldCount × 0.6 + rating × 0.4).
     ES'e field olarak kaydet.
     Arama: function_score yerine sadece bu field → çok daha hızlı.

  d) Aggregation cache:
     "Tüm ayakkabı markaları" → nadiren değişir → Redis'e cache (1 saat).
     Her arama isteğinde agg hesaplamak yerine → cache'den.

  e) Index lifecycle:
     Aktif ürünler (inStock=true, 30 günde satılan): hot index (SSD).
     Pasif ürünler: warm index (HDD) → arama sonuçlarında düşük öncelik.
     Silinen ürünler: cold index (archive, aranmaz).
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Arama | DB LIKE/ILIKE | Elasticsearch | LIKE: index kullanamaz, 100M'de yavaş; ES: full-text, facet, Turkish analyzer, relevance scoring |
| Katalog sync | Dual write | CDC (Debezium) | Dual write: atomik değil, tutarsızlık riski; CDC: WAL'dan güvenilir, replay mümkün |
| Sepet | PostgreSQL | Redis Hash | Sepet geçici, yüksek okuma/yazma; Redis: sub-ms, TTL native; DB: gereksiz I/O |
| Stok kilit | FOR UPDATE (pessimistic) | Redis DECR + DB optimistic | Pessimistic: deadlock riski, yüksek trafikte yavaş; Redis DECR: sub-ms, 100K/sn |
| Fiyat hesap | DB stored procedure | Rule Engine (Drools) | Stored proc: deploy gerektirir, test zor; Rule engine: runtime değiştirilebilir, A/B test |
| Flash sale | Normal sipariş akışı | Queue + Waiting Room | Normal akış: thundering herd → çökme; Queue: kontrollü yük, UX korumalı |
| Saga | 2PC (distributed transaction) | Choreography/Orchestration Saga | 2PC: distributed lock, availability düşer; Saga: eventual consistency, compensating tx |
| Öneri | Real-time sadece | Offline batch + Real-time fallback | Real-time: 100M ürün × tüm kullanıcı → hesap yoğun; Batch ALS: doğru + Redis'ten hızlı |
| Cache stampede | Basit TTL | Mutex + stale-while-revalidate | Basit TTL: thundering herd; Mutex: tek DB sorgusu; Stale: kullanıcı hep hızlı cevap alır |
| ES-DB tutarsız | Senkron güncelleme | Async CDC (1-3sn lag) | Senkron: DB-ES aynı transaction → dağıtık transaction sorunu; Async: eventual, kabul edilebilir |
