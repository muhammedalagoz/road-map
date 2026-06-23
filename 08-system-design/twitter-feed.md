# 08d — System Design: Twitter/Social Feed

## Gereksinimler

```
Functional:
  ✓ Tweet yayınla (metin + medya)
  ✓ Timeline: takip ettiklerinin tweet'lerini gör
  ✓ Takip et / takipten çık
  ✓ Like, retweet
  ✗ DM, trending, arama (out of scope)

Non-Functional:
  DAU: 300M
  Write: 10,000 tweet/sn
  Read:  100,000 timeline/sn (read-heavy)
  Latency: timeline < 200ms
  Availability: 99.99%
  Eventual consistency kabul edilebilir
```

---

## Capacity Estimation

```
Write: 300M × 3 tweet/gün / 86400 ≈ 10,000/sn
Read:  300M × 30 okuma/gün / 86400 ≈ 100,000/sn
Ratio: 1:10 (read-heavy)

Storage:
  Tweet: 1 KB (text + metadata)
  Medya: %10 tweet → 500 KB ortalama
  Günlük text: 10,000 × 86400 × 1KB ≈ 860 GB/gün
  Günlük medya: 1,000 × 86400 × 500KB ≈ 43 TB/gün

Timeline cache:
  Kullanıcı başına 200 tweet (last feed)
  300M × 200 × 1KB ≈ 60 TB → Redis cluster
```

---

## High-Level Tasarım

```
          Client
            │
       ┌────┴────┐
       │   CDN   │ ← medya (fotoğraf, video)
       └────┬────┘
            │
      ┌─────┴──────┐
      │ API Gateway │
      └─────┬──────┘
            │
  ┌─────────┼─────────┬──────────────┐
  ▼         ▼         ▼              ▼
Tweet     User      Timeline      Search
Service   Service   Service       Service
  │         │          │
  ▼         ▼          ▼
Tweet DB  User DB   Redis Cache
(Cassandra) (MySQL)  (Timeline)
  │
  ▼
Fan-out
Service
  │
  ▼
Kafka
(tweet.created)
```

---

## Write Path: Tweet Oluşturma

```
1. Client → POST /tweets (text, media_ids)

2. Tweet Service:
   a. Tweet'i kaydet → Cassandra (tweets by user_id)
   b. Medya varsa S3'e upload (CDN üzerinden serve)
   c. Kafka'ya event yayınla: tweet.created {tweetId, userId, text, timestamp}

3. Fan-out Service (Kafka consumer):
   a. Kullanıcının follower listesini çek (User Graph DB)
   b. Her follower'ın timeline'ına bu tweet'i push et
```

---

## Fan-Out Stratejileri

### Push Model (Fan-Out on Write)

```
Tweet yazıldığında → hemen tüm follower timeline'larına ekle

Adımlar:
  Tweet oluşturuldu → Fan-out Service
  → follower_ids çek (User Graph: 200 follower ortalama)
  → Redis: LPUSH user:{followerId}:timeline {tweetId}
  → LTRIM user:{followerId}:timeline 0 999  (son 1000 tweet)

Avantaj:
  ✓ Okuma anında hazır (pre-computed)
  ✓ Timeline okuma basit: Redis LRANGE

Dezavantaj:
  ✗ Celebrity problemi: 100M takipçisi olan kullanıcı tweet attı
    → 100M × Redis write → fan-out servisi çöker
  ✗ Write yavaş (senkron olursa)
```

### Pull Model (Fan-Out on Read)

```
Timeline okunurken → takip edilenlerin tweet'lerini gerçek zamanlı çek

Adımlar:
  Timeline isteği → User Graph'ten following listesi
  → Her kullanıcı için son N tweet'i çek
  → Merge et, sırala (timestamp desc)

Avantaj:
  ✓ Write basit (sadece tweet kaydet)
  ✓ Celebrity problemi yok

Dezavantaj:
  ✗ Read pahalı (N following × DB sorgu → N+1 problem)
  ✗ 500 kişiyi takip ediyorsun → 500 DB sorgu → birleştir
```

### Hybrid Model (Twitter'ın Gerçek Çözümü)

```
Normal kullanıcı (< 1M follower): Push model
Celebrity (> 1M follower): Pull model

Timeline okuma:
  1. Kendi cache'imi oku (push'lanmış normal tweet'ler)
  2. Takip ettiğim celebrity'lerin son tweet'lerini çek (pull)
  3. İkisini merge et → timestamp'e göre sırala → döndür

Threshold: 1M follower → "celebrity" → pull
Takip ettiğin celebrity sayısı genellikle az → pull cost az
```

---

## Read Path: Timeline

```
GET /timeline (ilk yükleme veya sayfalama)

Timeline Service:
1. Redis'ten: LRANGE user:{userId}:timeline 0 19 (ilk 20 tweet ID)
2. Tweet ID listesi → Tweet Service → batch fetch tweet detayları
3. Kullanıcı bilgileri → User Service → cache hit (Redis)
4. Medya URL'leri → CDN URL oluştur (S3 presigned veya public CDN)
5. Assemble & return

Cache miss (yeni kullanıcı veya cache evict):
  → Fan-out'tan rebuild: takip edilenlerin son N tweet'ini çek → cache doldur
```

---

## Veri Modelleri

### Cassandra (Tweet Storage)

```sql
-- tweets_by_user: kullanıcının tweet'leri (write path)
CREATE TABLE tweets_by_user (
    user_id    UUID,
    tweet_id   TIMEUUID,   -- zaman bazlı UUID → otomatik sıralı
    text       TEXT,
    media_urls LIST<TEXT>,
    like_count COUNTER,
    PRIMARY KEY ((user_id), tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);

-- tweets_by_id: tweet detay çekme (read path)
CREATE TABLE tweets_by_id (
    tweet_id   UUID PRIMARY KEY,
    user_id    UUID,
    text       TEXT,
    created_at TIMESTAMP,
    media_urls LIST<TEXT>
);
```

### Redis (Timeline Cache)

```
Key: user:{userId}:timeline
Type: List (sorted by insertion = time order)
Value: [tweetId1, tweetId2, ...]
TTL: 7 gün (aktif kullanıcı)

LPUSH user:123:timeline tweetId    → başa ekle (newest first)
LTRIM user:123:timeline 0 999      → max 1000 tweet tut
LRANGE user:123:timeline 0 19      → ilk 20'yi oku (pagination)
```

### User Graph (Neo4j veya Cassandra)

```sql
-- Takipçi ilişkisi
CREATE TABLE follows (
    follower_id UUID,
    following_id UUID,
    created_at TIMESTAMP,
    PRIMARY KEY ((follower_id), following_id)
);

-- Reverse: Kim takip ediyor? (fan-out için)
CREATE TABLE followers_by_user (
    user_id     UUID,
    follower_id UUID,
    PRIMARY KEY ((user_id), follower_id)
);
```

---

## Medya Yükleme

```
Client → Presigned S3 URL → Direkt S3 upload → CDN cache

Akış:
1. POST /media/upload-url → Media Service
2. Media Service → S3 presigned URL üret → client'a döndür
3. Client → S3'e direkt upload (Media Service üzerinden geçmez)
4. S3 → Upload tamamlandı event → Thumbnail Service (Lambda)
5. Thumbnail Service → farklı boyutlar üret (400px, 800px, original)
6. CDN: S3 → CloudFront → kullanıcıya

Avantaj: Büyük dosya Media Service'ten geçmez → bandwidth tasarrufu
```

---

## Ölçekleme

```
Bottleneck 1: Fan-out (celebrity tweet)
  → Kafka partition'larına göre paralel fan-out worker
  → Celebrity için async/delayed fan-out (kritik değil)
  → Rate limiting fan-out (burst kontrolü)

Bottleneck 2: Timeline cache boyutu (60 TB)
  → Redis Cluster (64 node × 1 TB = 64 TB)
  → Inactive kullanıcı TTL: 7 gün (90 günden fazla giriş yoksa cache sil)

Bottleneck 3: Tweet DB
  → Cassandra: yatay sharding, replication factor 3
  → Write: ANY consistency (hızlı) → Read: LOCAL_QUORUM

Bottleneck 4: User Graph (kim kimi takip ediyor)
  → Redis'te cache (kullanıcı → follower list)
  → Cassandra'da source of truth
```

---

## Trade-off Özeti

| Karar | Push | Pull | Hybrid |
|-------|------|------|--------|
| Write cost | Yüksek | Düşük | Orta |
| Read cost | Çok düşük | Yüksek | Düşük |
| Celebrity sorunu | Var | Yok | Yok (pull ile çöz) |
| Gerçek zamanlılık | Anlık | Anlık | Orta (merge latency) |
