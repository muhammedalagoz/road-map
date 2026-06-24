# 08c — System Design: URL Shortener (bit.ly)

## Gereksinimler

```
Functional:
  ✓ Uzun URL → kısa URL oluştur
  ✓ Kısa URL → orijinal URL'ye yönlendir (redirect)
  ✓ Link analitik (tıklama sayısı, coğrafi dağılım)
  ✗ Kullanıcı yönetimi (out of scope)

Non-Functional:
  100M URL oluşturma/gün
  10B redirect/gün (100:1 read/write ratio)
  Latency: redirect < 50ms
  Availability: 99.99%
  URL yaşam süresi: 5 yıl
```

---

## Capacity Estimation

```
Write QPS: 100M / 86400 ≈ 1,200 URL/sn
Read QPS:  10B  / 86400 ≈ 115,000 redirect/sn
Peak:      115,000 × 3 ≈ 350,000 QPS

Storage:
  Bir URL kaydı: 500 bytes (uzun URL 2000 char max + metadata)
  5 yıl: 100M × 365 × 5 × 500 bytes = 91 TB

Bandwidth:
  Write: 1,200 × 500B ≈ 600 KB/sn
  Read:  115,000 × 500B ≈ 57 MB/sn (redirect response küçük: sadece header)

Cache:
  Hot URL'ler %20'si → %80 traffic
  Cache: 115,000 QPS × 0.2 × 500B ≈ 12 GB → RAM'e sığar
```

---

## High-Level Tasarım

```
                         ┌─────────────────────────────────────┐
                         │           API Gateway               │
                         │   Rate Limiting + Auth              │
                         └───────────────┬─────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              ▼                          ▼                          ▼
    ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
    │  URL Service     │     │ Redirect Service  │     │ Analytics Service│
    │  (Write path)    │     │ (Read path)       │     │                  │
    └────────┬─────────┘     └────────┬─────────┘     └──────────────────┘
             │                        │
             ▼                        ▼
    ┌──────────────────┐     ┌──────────────────┐
    │  Key Generation  │     │  Redis Cache     │
    │  Service (KGS)   │     │  (hot URLs)      │
    └──────────────────┘     └────────┬─────────┘
             │                        │ miss
             ▼                        ▼
    ┌──────────────────────────────────────────────────┐
    │              Database (MySQL/Cassandra)           │
    │  short_url | long_url | created_at | user_id     │
    └──────────────────────────────────────────────────┘
```

---

## Write Path: URL Kısaltma

### Seçenek 1: Hash (MD5/SHA)

```
short_key = MD5(long_url)[0:7]   → 7 karakter base62

Sorun:
  hash("https://example.com") → "3d4f8a1"
  hash("https://example.com") → "3d4f8a1" (aynı URL = aynı key ✓)
  hash("https://other.com")   → "3d4f8a2"
  hash("https://third.com")   → "3d4f8a1" → COLLISION! ✗

Collision handling:
  +1 suffix → rehash → tekrar dene → yavaş, karmaşık
```

### Seçenek 2: Key Generation Service (KGS) — Önerilen

```
[a-zA-Z0-9]^7 = 62^7 ≈ 3.5 trilyon unique key (yeterli)

KGS nasıl çalışır?
  1. Offline olarak tüm 7-char kombinasyonları üret → DB'ye yaz
     (not_used_keys tablosu)
  2. Uygulama başladığında: 1000 key al → in-memory buffer
  3. URL oluştur → buffered key kullan → used_keys'e taşı

Race condition yok:
  KGS startup'ta: SELECT ... FOR UPDATE LIMIT 1000 → mutex
  Her instance farklı key batch'i alır → çakışma yok

Avantajlar:
  ✓ Collision yok (pre-generated unique)
  ✓ Hızlı (memory'den al, DB round-trip yok)
  ✓ Basit
  
KGS DB şeması:
  not_used_keys: (key VARCHAR(7) PK)
  used_keys:     (key VARCHAR(7) PK)

Tablo boyutu:
  3.5 trilyon × 7 bytes ≈ 24.5 TB → çok büyük
  
  Çözüm: Sadece 100M key önden üret (runtime'da ek üret)
  100M × 7 bytes = 700 MB → yönetilebilir
```

### Seçenek 3: Auto-Increment + Base62 Encoding

```
DB auto-increment ID: 1, 2, 3, ...
Base62 encode: 1 → "1", 10 → "A", 61 → "z", 62 → "10"

ID 123456789 → base62 → "8M0kX"

Avantaj: Basit, DB'ye bağlı
Dezavantaj:
  Sıralı ID → URL tahmin edilebilir (güvenlik)
  Distributed sistemde auto-increment zor (sharding)
  
Çözüm: ID'yi scramble et (Hashids gibi kütüphane)
```

---

## Read Path: Redirect

```
GET https://short.ly/abc1234

Redirect Service:
1. Cache kontrolü (Redis): key = "abc1234"
   HIT → 301/302 Location: long_url → End

2. DB sorgusu:
   SELECT long_url FROM urls WHERE short_key = 'abc1234'
   MISS → 404 Not Found

3. Cache yaz: SET abc1234 long_url EX 86400 (1 gün TTL)

4. 301 vs 302:
   301 Permanent Redirect: Browser cache'ler → gelecekte direkt gider
     ✓ Server yükü azalır (browser tekrar sormaz)
     ✗ Analytics bozulur (tıklama sayılamaz)
   
   302 Temporary Redirect: Browser her seferinde server'a sorar
     ✓ Her tıklamayı analitik sistemi yakalar
     ✗ Her request server'a gelir

   Bit.ly gibi sistemler: 302 kullanır (analytics için)
   Link expire/update gerekiyorsa: 302
```

---

## Database Tasarımı

```sql
CREATE TABLE urls (
    short_key   CHAR(7)      PRIMARY KEY,
    long_url    VARCHAR(2048) NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMP,
    click_count BIGINT       NOT NULL DEFAULT 0,
    
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

-- Analitik (ayrı tablo / ayrı servis)
CREATE TABLE click_events (
    id          BIGINT       AUTO_INCREMENT PRIMARY KEY,
    short_key   CHAR(7)      NOT NULL,
    clicked_at  TIMESTAMP    NOT NULL,
    ip_address  VARCHAR(45),
    country     CHAR(2),
    user_agent  VARCHAR(512),
    referrer    VARCHAR(2048),
    
    INDEX idx_short_key_clicked (short_key, clicked_at)
) -- Partitioned by clicked_at (aylık)
```

---

## Ölçekleme

```
Bottleneck 1: Read QPS (115K/sn)
  Çözüm: Redis cluster (read cache)
  Cache hit rate %99 hedefi → DB 1,150 QPS (yönetilebilir)

Bottleneck 2: Write (1200 URL/sn)
  Çözüm: MySQL → Horizontal sharding (user_id by hash)
  Veya: Cassandra (write-optimized)

Bottleneck 3: Analytics (10B event/gün)
  Çözüm: Kafka → ClickHouse (OLAP)
  click_events DB'ye direkt yazılmaz → event stream

Bottleneck 4: KGS single point of failure
  Çözüm: KGS replicated (ZooKeeper ile leader election)
  Her instance startup'ta key batch alır → KGS çöküşü → batch bitene kadar çalışır
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B |
|-------|-----------|-----------|
| Key üretim | Hash (basit) | KGS (collision-free) |
| Redirect | 301 (cache, hızlı) | 302 (analitik) |
| DB | MySQL (ACID) | Cassandra (write scale) |
| Cache | Redis (distributed) | Memcached (simpler) |

---

## API Tasarımı

```
POST /api/v1/urls
Body:  { "long_url": "https://...", "custom_alias": "mylink", "expires_in": 86400 }
Resp:  { "short_url": "https://short.ly/abc1234", "short_key": "abc1234", "expires_at": "..." }

GET  /api/v1/urls/{short_key}
Resp: { "short_key": "abc1234", "long_url": "...", "click_count": 42, "created_at": "..." }

DELETE /api/v1/urls/{short_key}
Resp:  204 No Content

GET  /api/v1/urls/{short_key}/analytics?from=2024-01-01&to=2024-01-31
Resp: { "total_clicks": 1042, "by_country": [...], "by_day": [...] }
```

---

## Karşılaşılabilecek Sorunlar ve Çözümleri

### 1. Cache Stampede (Thundering Herd)

```
Senaryo:
  - Viral URL'nin cache TTL'i doluyor
  - Aynı anda 10,000 istek → cache MISS → hepsi DB'ye gidiyor
  - DB eziliyor

Çözüm 1: Mutex/Lock (Single Flight Pattern)
  - İlk MISS gelen thread DB'ye gider + kilit alır
  - Diğerleri kilidi bekler → DB'den dönen sonucu kullanır
  - Redis: SET lock:abc1234 1 NX EX 5

Çözüm 2: Probabilistic Early Expiration
  TTL dolmadan rastgele %5 ihtimalle cache'i yenile
  → Hiçbir zaman sıfırdan fetch gerekmez

Çözüm 3: Stale-While-Revalidate
  - Expired cache'i hemen döndür
  - Arka planda async DB fetch + cache yenile
  - Kullanıcı stale data alır ama latency düşük kalır
```

### 2. Hotspot / Viral URL Sorunu

```
Senaryo:
  - Twitter'da viral olan 1 URL → anlık 500K req/sn
  - Tek Redis node'u eziyor

Çözüm: Local In-Process Cache
  Her uygulama instance'ında Caffeine/Guava cache (JVM heap)
  → İlk katman: process memory (~100MB, 1M entry)
  → İkinci katman: Redis cluster
  → Üçüncü katman: DB

  Read path:
    L1 hit → < 1ms
    L2 hit → 1-2ms (Redis)
    L3 hit → 5-10ms (DB)

  TTL: L1=10sn, L2=1saat, L3=∞
```

### 3. KGS (Key Generation Service) Hataları

```
Senaryo A: KGS çöktü, in-memory buffer'daki key'ler kayboldu
  Sonuç: O batch key'ler hem used hem not_used değil → "lost"
  Çözüm: Her KGS instance'ı key'leri batch olarak "reserved" state'e alır
          Kullandıkça "used" olarak işaretler
          Crash'te reserved ama unused key'ler → background job geri alır

Senaryo B: KGS restart'ta eski buffer tekrar dağıtıldı → DUPLICATE KEY
  Çözüm: Batch'i DB'den sil/reserved_at timestamp koy → idempotent al

KGS HA Pattern:
  ┌──────────┐    ┌──────────┐
  │  KGS-1   │    │  KGS-2   │  (hot standby)
  │ (leader) │    │(follower)│
  └────┬─────┘    └──────────┘
       │ ZooKeeper leader election
       ▼
  not_used_keys DB (PostgreSQL, replicated)
```

### 4. URL Validation ve Güvenlik

```
Tehditler:
  a) Phishing / Malware URL'leri kısaltma
  b) Spam kampanyaları (rate abuse)
  c) URL tahmin saldırısı (sequential ID enumeration)
  d) DDoS via redirect amplification

Çözüm A: Malicious URL Tespiti
  - Google Safe Browsing API entegrasyonu (write path'te async check)
  - URL blacklist (Redis set: "blocked_urls")
  - Yeni URL oluştururken: async scan → suspicious flag
  - Redirect'te: flagged URL → interstitial uyarı sayfası

Çözüm B: Rate Limiting (çok katmanlı)
  API Gateway:   IP başına 100 URL/saat (token bucket)
  User bazlı:    Authenticated user: 10K URL/gün
  Redis'te:      INCR rate:ip:{ip}:{hour} → expire 3600

Çözüm C: Enumeration Önleme
  - KGS: random UUID-like key (ardışık değil)
  - Auto-increment kullanılıyorsa: Hashids ile scramble
  - Private URL'ler için: 128-bit token (ek auth gerektirir)

Çözüm D: Spam Detection
  - Aynı long_url'i saatte 100+ kez kısaltan IP → block
  - Honeypot: bilinen spam domain'leri → 403
```

### 5. Database Sorunları

```
Sorun: 91TB storage, sharding nasıl yapılır?

Sharding Stratejisi — short_key'e göre hash:
  shard_id = hash(short_key) % num_shards

  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Shard 0   │  │  Shard 1   │  │  Shard 2   │
  │ a-m keys   │  │ n-z keys   │  │ 0-9 keys   │
  └────────────┘  └────────────┘  └────────────┘

Hotspot shard sorunu:
  - Viral URL'lerin short_key'leri aynı shard'a düştü → o shard eziliyor
  - Çözüm: Consistent hashing → shard ekleme/çıkarmada rehash minimize

Read replica:
  Master: write
  3x Read replica: redirect sorguları (eventual consistency kabul edilir)
  Lag < 100ms → redirect için sorun yok
```

### 6. Analytics Pipeline Sorunları

```
Sorun: 10B tıklama/gün → direkt DB yazma imkansız (115K write/sn)

Çözüm: Event Streaming
  Click event → Kafka topic "click-events"
    → Flink / Spark Streaming (real-time aggregation)
      → ClickHouse (OLAP, partitioned by day)
    → S3 (raw events, long-term storage)

Veri kaybı riski:
  Kafka consumer lag → batch retry
  At-least-once delivery → ClickHouse'da dedup (event_id unique key)

Approximate counting (yüksek trafik için):
  Redis HyperLogLog → click_count estimate (%0.81 error)
  SET url:abc1234:clicks → PFADD → PFCOUNT
  Günlük batch job: HLL count → DB'ye persist
```

### 7. URL Expiration ve Cleanup

```
Sorun: 5 yıl sonra expired URL'ler DB'de birikir → 91TB şişer

Cleanup Stratejileri:

Lazy Deletion:
  - Redirect geldiğinde expires_at < NOW() → 410 Gone
  - DB'den hemen silme → background job
  
Active Cleanup (Scheduled Job):
  Gece 02:00 batch:
  DELETE FROM urls WHERE expires_at < NOW() - INTERVAL 30 DAY
  LIMIT 10000;  ← küçük batch, index-friendly

Soft Delete Pattern:
  - deleted_at column ekle → hard delete yerine soft delete
  - Silinen key → KGS not_used_keys'e GERİ EKLENEBILIR (key recycling)
  - Dikkat: Eski link'i bilen biri yeni URL'ye gidebilir → 6 ay quarantine
```

---

## Genişletilmiş Mimari (Production-Ready)

```
                    ┌─────────────────────────────────────────┐
                    │         CDN (CloudFront/Akamai)          │
                    │  Static assets + popular redirect cache  │
                    └──────────────────┬──────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────┐
                    │           API Gateway / LB               │
                    │  Rate Limit · Auth · SSL Termination     │
                    └────┬──────────────┬────────────┬────────┘
                         │              │            │
              ┌──────────▼──┐  ┌────────▼──┐  ┌────▼──────────┐
              │ URL Service │  │ Redirect  │  │  Analytics    │
              │ (Write)     │  │ Service   │  │  Service      │
              │ 10 pods     │  │ 50 pods   │  │  5 pods       │
              └──────┬──────┘  └─────┬─────┘  └──────┬────────┘
                     │               │                │
              ┌──────▼──────┐  ┌─────▼────────┐      │
              │ KGS Cluster │  │ Redis Cluster│      │
              │ (2 nodes)   │  │ 6 nodes      │      ▼
              └──────┬──────┘  │ L2 Cache     │  ┌──────────┐
                     │         └──────┬────────┘  │  Kafka   │
                     │           miss │           │  Cluster │
                     ▼                ▼           └────┬─────┘
              ┌──────────────────────────────┐         │
              │     MySQL Cluster            │    ┌────▼──────┐
              │  1 Master + 3 Read Replicas  │    │ClickHouse │
              │  Sharded (consistent hash)   │    │ (OLAP)    │
              └──────────────────────────────┘    └───────────┘
                     │
              ┌──────▼──────┐
              │  S3 Backup  │
              │  (daily)    │
              └─────────────┘
```

---

## High Availability ve Disaster Recovery

```
Multi-Region Deployment:
  Primary: us-east-1
  Secondary: eu-west-1 (hot standby, async replication lag ~100ms)
  
  DNS Failover: Route53 health check → failover 30sn içinde

RPO (Recovery Point Objective): < 1 dakika
RTO (Recovery Time Objective): < 5 dakika

DB Replication:
  MySQL Binlog → cross-region replica
  Redis: AOF + RDB snapshot → S3'e her 5 dakikada bir

Chaos Engineering Testleri:
  - Redis node kill → L1 cache devreye giriyor mu?
  - KGS crash → yeni URL üretimi duruyor mu?
  - DB master fail → replica promotion süresi?
```

---

## Monitoring ve Alerting

```
Key Metrics:
  p50/p95/p99 redirect latency    → alert: p99 > 100ms
  Cache hit rate                  → alert: < 95%
  KGS buffer doluluk oranı        → alert: < 10%
  Error rate (4xx/5xx)            → alert: > 0.1%
  Kafka consumer lag              → alert: > 1M messages

Dashboards:
  - Redirect QPS (real-time)
  - Top 100 hot URLs (by click/min)
  - Shard distribution balance
  - KGS key stock (kaç key kaldı)

Distributed Tracing:
  Her request'e trace-id → Jaeger/Zipkin
  Yavaş redirect nedenini tespit: L1 miss? L2 miss? DB slow query?
```

---

## Gelişmiş Özellikler (Nice-to-Have)

```
Custom Alias:
  POST body'de custom_alias: "myproduct" → short.ly/myproduct
  Conflict check: SELECT COUNT(*) FROM urls WHERE short_key = 'myproduct'
  Reserved words: ["api", "admin", "login", "static"] → reject

Password-Protected URLs:
  extra column: password_hash (bcrypt)
  Redirect: 401 → enter password → bcrypt verify → redirect

QR Code Generation:
  POST /api/v1/urls → response'a qr_code_url ekle
  QR: CDN'de cached PNG (lazy generate on first request)

Link Preview (Open Graph):
  GET short.ly/abc1234+  → preview sayfası (destination URL metadata)
  Bot detection: User-Agent check → bots için interstitial yok
```
