# 08 — System Design

## 1. System Design Interview Framework

Her system design sorusuna bu sırayla yaklaş:

```
1. Gereksinimleri netleştir (5 dk)
   - Functional requirements: ne yapmalı?
   - Non-functional: QPS, latency, availability, consistency
   - Scale: kaç kullanıcı? kaç istek/saniye?

2. Capacity estimation (5 dk)
   - Günlük aktif kullanıcı × ortalama istek = QPS
   - Depolama: veri boyutu × günlük kayıt × yıl

3. High-level tasarım (10 dk)
   - Temel bileşenler: client, API, servis, DB, cache

4. Deep dive (20 dk)
   - Darboğazları tespit et
   - Trade-off'ları tartış
   - Alternatifleri karşılaştır

5. Darboğaz & İyileştirme (5 dk)
   - SPOF (Single Point of Failure) var mı?
   - Nasıl scale edilir?
```

---

## 2. Capacity Estimation

**Sayılar ezber (10x yaklaşım yeterli):**
```
1 KB = 1,000 bytes
1 MB = 1,000 KB
1 GB = 1,000 MB

Hız:
Memory read:    100 ns
SSD read:       100 μs  (1000x memory)
Network:        1 ms round-trip (same DC)
HDD read:       10 ms   (100x SSD)

Günde saniye: 86,400 ≈ 100,000
Ayda saniye:  2,592,000 ≈ 3,000,000
```

**Twitter örneği:**
```
DAU (Daily Active Users): 300M
Tweet/gün: 300M × 3 = 900M tweet
QPS (write): 900M / 86400 ≈ 10,000 tweet/sn
QPS (read): 10,000 × 100 = 1,000,000 read/sn (okuma çok daha fazla)

Tweet boyutu: 280 karakter + metadata ≈ 1KB
Günlük depolama: 900M × 1KB = 900GB/gün
10 yıl: 900GB × 365 × 10 = 3.3PB (medya hariç!)
```

---

## 3. Scalability Patterns

### Horizontal vs Vertical Scaling

```
Vertical (scale up):
4 core, 16GB → 32 core, 128GB
Avantaj: Basit, uygulama değişmez
Dezavantaj: Limit var, tek SPOF, pahalı

Horizontal (scale out):
1 sunucu → 10 sunucu
Avantaj: Teorik sınırsız, HA
Dezavantaj: Stateless gerektirir, koordinasyon
```

### Sharding Stratejileri

**Range-based sharding:**
```
Shard 0: user_id 1 - 1,000,000
Shard 1: user_id 1,000,001 - 2,000,000
Shard 2: user_id 2,000,001 - ...

Sorun: Hot spot (yeni kullanıcılar son shard'a yığılır)
```

**Hash-based sharding:**
```
shard = hash(user_id) % shard_count
Avantaj: Uniform dağılım
Dezavantaj: Shard eklemek → rehashing (consistent hashing ile çöz)
```

**Directory-based sharding:**
```
Lookup service: user_id → shard mapping
Avantaj: Esnek (herhangi user'ı herhangi shard'a taşı)
Dezavantaj: Lookup service SPOF, her sorgu + 1 hop
```

---

## 4. Caching Strategies

### Hangi Cache Nerede?

```
Client cache (browser): static assets, TTL header
CDN: coğrafi dağıtık, static + dynamic content
Application cache (in-process): Caffeine, Guava (L1)
Distributed cache: Redis, Memcached (L2)
Database cache: PostgreSQL buffer pool, query cache

Okuma sırası: L1 (ms) → L2 (ms) → DB (10-100ms)
```

### Cache Invalidation

"There are only two hard things in CS: cache invalidation and naming things."

**TTL (Time-to-Live):** Belirli süre sonra otomatik expire.  
Sorun: TTL boyunca stale veri.

**Write-through invalidation:**
```
DB güncelle → cache sil (veya güncelle)
Sorun: Race condition:
1. Thread A → DB güncelle
2. Thread B → DB'den oku → cache yaz (eski değer!)
3. Thread A → cache sil (ama B az önce yazdı)

Çözüm: Cache-Aside + DB update + cache delete sırası
```

**Event-driven invalidation:**
```
OrderUpdated event → Cache invalidation listener → Redis DEL
Async, loose coupling, ama eventual
```

---

## 5. CDN Nasıl Çalışır

```
User (İstanbul) → CDN Edge (Frankfurt) → Origin Server (US)

1. İlk istek: CDN miss → origin'den çek → cache'le → kullanıcıya ver
2. Sonraki istekler: CDN hit → direkt ver (origin'e gitmez)

Cache-Control header:
Cache-Control: public, max-age=86400, s-maxage=604800
max-age: browser cache süresi
s-maxage: CDN cache süresi

CDN Ne zaman kullanılır?
✓ Static assets (JS, CSS, images)
✓ Video streaming
✓ Coğrafi dağıtık kullanıcı tabanı
✓ DDoS koruması
✓ API caching (okuma ağırlıklı, az değişen)
```

---

## 6. Klasik System Design: URL Shortener

**Gereksinimler:**
- Uzun URL → kısa URL oluştur (bit.ly/abc123)
- Kısa URL → yönlendir
- 100M URL/gün, 10B redirect/gün

**Capacity:**
```
Write QPS: 100M / 86400 ≈ 1200/sn
Read QPS:  10B / 86400 ≈ 115,000/sn (read-heavy)
Storage: 100M × 500bytes = 50GB/gün, 10 yıl = 182TB
```

**Tasarım:**
```
Client → Load Balancer → URL Service → Cache (Redis) → DB (MySQL)
                              ↓
                         Key Generation Service (KGS)

Redirect akışı:
1. GET bit.ly/abc123
2. Cache hit? → 301/302 Location: uzun-url
3. Cache miss → DB'den oku → Cache yaz → redirect

KGS (Key Generation Service):
- Pre-generated 6 karakter keys: [a-zA-Z0-9]^6 = 56.8 milyar
- Key database: used_keys, not_used_keys tabloları
- Startup'ta batch key al → memory'de tut (race condition yok)

Neden KGS? URL hash kullanmak yerine:
hash(uzun-url) → collision riski, uzun hash
pre-generated → garantili unique, hızlı
```

---

## 7. Klasik System Design: Twitter/Twitter-like Feed

**Write path:**
```
Tweet oluştur → Tweet DB → Fan-out Service
                                   ↓
                           Follower listesini çek
                                   ↓
                        Her follower'ın timeline cache'ine ekle (push model)
                                   ↓
                               Redis LPUSH user:123:timeline tweetId
```

**Read path:**
```
Timeline iste → Redis (user:123:timeline) → Tweet detayları çek → render
```

**Push vs Pull trade-off:**
```
Push (fan-out on write):
+ Okuma hızlı (cache hazır)
- Çok takipçili kullanıcı (celebrity) → milyon kişiye yazmak
  → Hot user problemi: Justin Bieber tweet'i 100M takipçiye push = yavaş

Pull (fan-out on read):
+ Write basit
- Read pahalı (her timeline'da tüm takip edilenlerin tweet'leri sorgulanır)

Hybrid (Twitter'ın gerçek çözümü):
- Normal kullanıcı → push
- Celebrity (>X takipçi) → pull (timeline okunurken dinamik merge)
```

---

## 8. Klasik System Design: Rate Limiter

```
Gereksinimler:
- 1000 request/dakika per user
- Distributed (birden fazla API server)
- Hızlı (eklenen latency < 1ms)

Tasarım:
Client → API Server → Rate Limit Middleware → Redis
                              ↓
                      Limit aşıldı → 429 Too Many Requests
                      Headers: X-RateLimit-Remaining: 0
                               X-RateLimit-Reset: 1705312800

Redis Token Bucket implementasyonu:
Lua script (atomik):
local tokens = tonumber(redis.call("GET", key) or limit)
if tokens > 0 then
    redis.call("SET", key, tokens - 1, "EX", 60)
    return 1  -- allowed
else
    return 0  -- rejected
end

Neden Lua? Client-side atomicity: GET + SET race condition'sız
```

---

## 9. High Availability Patterns

**SLA hesabı:**
```
99%    uptime → 3.65 gün/yıl downtime
99.9%  uptime → 8.76 saat/yıl
99.99% uptime → 52.6 dakika/yıl (four nines)
99.999% uptime → 5.26 dakika/yıl (five nines)
```

**SPOF Elimination:**
```
Her katmanda redundancy:

Load Balancer: Active-Active veya Active-Passive (VRRP/keepalived)
Application: N instance, stateless
Database: Primary-Replica, automatic failover (Patroni, RDS Multi-AZ)
Cache: Redis Sentinel veya Cluster
Message Queue: Kafka replication, RabbitMQ quorum queues
DNS: Multiple A records, health check
```

**Graceful degradation:**
```
Recommendation service çöktüğünde:
Normal: "Sizin için önerilen ürünler"
Degraded: "Popüler ürünler" (cache'ten)
Circuit open: "Diğer ürünlere göz atın" (static)
```
