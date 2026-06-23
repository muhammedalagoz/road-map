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
