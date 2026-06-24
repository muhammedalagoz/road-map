# 08d — System Design: Twitter/Social Feed

## Gereksinimler

```
Functional:
  ✓ Tweet yayınla (metin + medya: foto, video)
  ✓ Timeline: takip ettiklerinin tweet'lerini gör
  ✓ Takip et / takipten çık
  ✓ Like, retweet, quote tweet
  ✓ Reply ve thread (tweet zinciri)
  ✓ Tweet silme (kendi tweet'ini)
  ✓ Bildirimler (like, retweet, follow, reply)
  ✓ Kronolojik + algoritma bazlı timeline seçeneği
  ✗ DM, trending topics, global arama (out of scope — ayrı servisler)

Non-Functional:
  DAU: 300M
  Write: 10,000 tweet/sn (peak: 30,000/sn — büyük etkinlik)
  Read:  100,000 timeline/sn (read-heavy)
  Ratio: 1:10 (read >> write)
  Latency: timeline < 200ms (P99)
  Availability: 99.99%
  Eventual consistency kabul edilebilir (like sayısı, feed gecikmesi)
```

---

## Capacity Estimation

```
Tweet write:
  300M DAU × 3 tweet/gün / 86400 ≈ 10,000 tweet/sn.
  Peak (büyük maç, seçim): 3x → 30,000 tweet/sn.

Tweet storage:
  Tweet: ~1 KB (280 karakter + metadata + indeks).
  Günlük: 10,000 × 86400 × 1 KB ≈ 860 GB/gün.
  5 yıl: 860 GB × 365 × 5 ≈ 1.6 PB → Cassandra cluster.

Medya storage:
  Tweet'lerin %10'u fotoğraf, %5'i video.
  Fotoğraf: ortalama 500 KB (orijinal) + thumbnail (50 KB × 3 boyut).
  Video: ortalama 5 MB (HLS segment, 720p).
  Günlük: 1,000/sn × 500 KB (foto) + 500/sn × 5 MB (video)
        = 43 GB/sn + 2.5 GB/sn = 45.5 GB/sn... değil:
        1,000 × 86400 × 500 KB = 43 TB/gün (fotoğraf)
        500 × 86400 × 5 MB = 216 TB/gün (video) → toplam: ~260 TB/gün.
  5 yıl medya: 260 TB × 365 × 5 ≈ 475 PB → S3 (yaşam döngüsü ile arşivle).

Social graph:
  300M kullanıcı, ortalama 500 takip → 150B follows edge.
  Her edge: 16 B (follower_id) + 16 B (following_id) + 8 B (ts) = 40 B.
  Toplam: 150B × 40 B = 6 TB → Cassandra.

Timeline cache:
  Aktif kullanıcı (DAU): son 200 tweet ID × 8 B = 1.6 KB/kullanıcı.
  300M × 1.6 KB = 480 GB → Redis Cluster (tweet metadata de eklersek ~60 TB).
  Optimizasyon: sadece tweet ID tut → batch fetch metadata → 480 GB.

Fan-out amplification:
  10,000 tweet/sn, ortalama 500 follower.
  Normal kullanıcı fan-out: 10,000 × 500 × 8 B = 40 MB/sn Redis write.
  Celebrity (30M follower): tek tweet → 30M Redis write → sorun (ayrı çözüm).
```

---

## High-Level Mimari

```
         Client (browser / mobile)
              │
    ┌─────────┴──────────┐
    │       CDN          │ ← medya (fotoğraf, video), statik asset
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │    API Gateway     │ ← rate limit, auth (JWT), routing
    └──┬─────┬─────┬─────┘
       │     │     │
 ┌─────▼──┐ ┌▼───────┐ ┌▼──────────────┐
 │ Tweet  │ │ User   │ │   Timeline    │
 │Service │ │Service │ │   Service     │
 └───┬────┘ └───┬────┘ └──────┬────────┘
     │          │             │
     │      ┌───▼────┐   ┌────▼────────┐
     │      │User DB │   │ Redis Cache │
     │      │(MySQL) │   │ (Timeline)  │
     │      └───┬────┘   └─────────────┘
     │          │
     ▼      ┌───▼────────────┐
  Tweet DB  │  User Graph DB │ ← Social Graph (Cassandra / Neo4j)
 (Cassandra)└────────────────┘
     │
  ┌──▼──────┐
  │  Kafka  │ ← tweet.created, tweet.liked, tweet.retweeted, tweet.deleted
  └──┬──────┘
     │
  ┌──▼──────────┐   ┌─────────────────┐
  │ Fan-out Svc │   │Notification Svc │
  └─────────────┘   └─────────────────┘
     │
  ┌──▼──────────┐
  │Media Process│ ← Video transcode, thumbnail
  └─────────────┘
```

---

## Write Path: Tweet Oluşturma

```
POST /tweets {text, media_ids[], reply_to_tweet_id?}

1. Tweet Service:
   a. Rate limit: kullanıcı başına 300 tweet/3saat → Redis INCR + EXPIRE.
   b. Tweet içerik moderasyonu: spam filter, banned word check (async OK).
   c. Tweet kaydı:
      - tweet_id: Snowflake ID (zaman bazlı 64-bit → sıralama + dağıtık).
      - Cassandra'ya yaz: tweets_by_user + tweets_by_id.
   d. Medya varsa: media_ids zaten S3'te → URL'leri tweet'e bağla.
   e. Reply ise: parent_tweet'in reply_count'unu artır (Redis INCR).

2. Kafka event yayınla (async, non-blocking):
   Topic: tweet.created
   {tweetId, userId, text, parentTweetId, mediaIds, createdAt, isRetweet}

3. Hemen response: 201 Created {tweetId, status: "published"}
   (Fan-out async — kullanıcı beklemez)

Fan-out Service (Kafka consumer):
   a. follower_ids çek: Cassandra followers_by_user.
   b. Shard by followerId: paralel Kafka consumer group (32 partition).
   c. Normal user (< 1M follower):
      - Redis PIPELINE: LPUSH + LTRIM her follower için.
      - Batch: 1000 follower per pipeline → 1 network roundtrip.
   d. Celebrity (> 1M follower): Fan-out ATLA → pull-on-read (hybrid).
   e. Inactive user (son 30 gün giriş yok): Fan-out ATLA → istek gelince lazy rebuild.
```

### Snowflake ID Yapısı

```
64-bit Snowflake ID:
  [41 bit timestamp | 10 bit machine_id | 12 bit sequence]
  
  Timestamp: epoch'tan millisecond → 69 yıl.
  Machine ID: 1024 farklı node.
  Sequence: ms başına 4096 ID.
  
  Toplam: 4096 × 1024 = 4M ID/ms → 4B ID/sn → yeterli.
  
  Avantaj:
    ✓ Merkezi koordinatör yok → dağıtık.
    ✓ Zaman sıralı → Cassandra'da clustering order ID'ye göre → efficient.
    ✓ K-sort özelliği → B-tree index'de sıralı insert → page splits az.
```

---

## Fan-Out Stratejileri (Detay)

### Push Model (Fan-Out on Write)

```
Tweet yazılır → hemen tüm follower timeline'larına ekle.

Kafka consumer (Fan-out Worker):
  Partition sayısı: 32 → 32 paralel worker.
  Her worker: 1 Kafka partition consume eder.
  
  for followerId in follower_ids:
    redis.pipeline()
      .lpush(f"user:{followerId}:timeline", tweetId)
      .ltrim(f"user:{followerId}:timeline", 0, 999)
    .execute()
  
  Batch size: 1000 follower/pipeline → latency düşer.
  Toplam süre (500 follower, 50ms Redis): ~50ms.
  
Avantaj:
  ✓ Okuma anında hazır: LRANGE O(N).
  ✓ Timeline Service basit: Redis oku → batch hydrate.

Dezavantaj:
  ✗ Celebrity (100M follower): 100M Redis write → 100M / 1000 = 100,000 pipeline → dakikalarca.
  ✗ Write amplification: 10,000 tweet/sn × 500 avg follower = 5M Redis write/sn.
```

### Pull Model (Fan-On on Read)

```
Timeline isteği gelince → takip edilenlerin son tweet'lerini çek.

Timeline Service:
  following_ids = graph.getFollowing(userId)  ← cache'li
  tweets = []
  for uid in following_ids:
    tweets += cassandra.query("SELECT * FROM tweets_by_user WHERE user_id=? LIMIT 30", uid)
  tweets.sort(by=created_at, desc)
  return tweets[:20]

Avantaj:
  ✓ Write basit: sadece Cassandra'ya kaydet.
  ✓ Celebrity sorun yok.

Dezavantaj:
  ✗ N+1 problem: 500 kişi takip → 500 Cassandra sorgu → merge.
  ✗ Latency: 500 × 5ms = 2.5 sn → kabul edilemez (< 200ms hedef).
  ✗ Çözüm: paralel sorgular → async 500 sorgu paralel → ~20ms (IO-bound).
     Ama yine de Redis'ten yavaş.
```

### Hybrid Model (Twitter'ın Gerçek Yaklaşımı)

```
Normal kullanıcı (< 1M follower): Fan-out on Write (Push).
Celebrity (≥ 1M follower): Fan-out on Read (Pull).

Timeline okuma:
  1. Redis'ten push-cache oku: LRANGE user:{userId}:timeline 0 199.
     → Normal kullanıcıların tweet'leri.
  2. Takip ettiğim celebrity'leri bul (User Graph cache).
  3. Her celebrity'nin son N tweet'ini Cassandra'dan çek (paralel).
  4. Push + Pull sonuçları → merge sort (timestamp DESC) → dedupe.
  5. Top 20 döndür.

Uygulama:
  Celebrity threshold: 1M follower (config olarak ayarlanabilir).
  "Takip ettiğim celebrity sayısı": genellikle < 10 → pull cost az.
  
  Merge algoritması:
    push_tweets = redis_timeline[:200]     → son 200 tweet ID → hydrate
    pull_tweets = parallel_fetch(celebrities[:10])  → her birinden 20 tweet
    all_tweets = merge_sort(push_tweets + pull_tweets)
    return dedupe(all_tweets)[:20]

Ek optimizasyon:
  "Senin için" (algorithmic) timeline: ayrı ranking service → ML model.
  "Takip edilenler" (chronological) timeline: yukarıdaki hybrid.
  Kullanıcı geçiş yapabilir.
```

---

## Read Path: Timeline Detay

```
GET /timeline?cursor={lastTweetId}&limit=20

Timeline Service:
1. Cursor decode: lastTweetId → Snowflake timestamp → zaman damgası.
2. Redis okuma:
   a. Push cache: LRANGE user:{userId}:timeline 0 199 → tweet ID listesi.
   b. Cursor'dan sonraki ID'leri al (Snowflake sıralı → binary search).
3. Celebrity pull (paralel async):
   CompletableFuture'larla tüm celebrity'ler aynı anda sorgulanır.
4. Merge + dedup + limit 20.
5. Hydrate tweet detayları:
   tweet_ids → Tweet Service batch get → Redis multi-get.
   Cache miss olanlar → Cassandra batch read.
6. User bilgileri:
   user_ids → User Service → Redis user:{id} cache hit.
7. CDN URL üret: {cdn_base}/{mediaKey}?w=800&h=600&fit=crop.
8. Return: {tweets[], nextCursor: lastTweetId_in_response, hasMore: bool}.

Cache miss (yeni kullanıcı veya expired):
  Timeline cache boş → rebuild:
    following_ids çek → her birinin son 200 tweet ID'si → merge sort → Redis'e yükle.
  Rebuild süresi: ~500ms (kabul edilebilir, ilk yüklemede).
  Arka planda async warm: kullanıcı giriş yaptığında trigger.
```

---

## Like / Retweet / Quote Tweet Mekaniği

### Like

```
POST /tweets/{tweetId}/like

Like Service:
  1. Deduplication kontrolü:
     Redis SISMEMBER user:{userId}:likes {tweetId}
     → zaten beğenmişse → 409 Conflict.

  2. Redis'e kaydet (hızlı yol):
     SADD user:{userId}:likes {tweetId}
     INCR tweet:{tweetId}:like_count
     SET tweet:{tweetId}:like_count {count} XX (sadece varsa güncelle)

  3. Async DB persist (Kafka):
     Topic: tweet.liked {userId, tweetId, timestamp}
     Consumer → MySQL likes tablosuna kaydet (sorgulama için).

  4. Notification:
     Tweet sahibine bildirim → Notification Service.
     "Ali beğendi" → push notification (eğer sahibi aktif değilse).

Unlike:
  SREM user:{userId}:likes {tweetId}
  DECR tweet:{tweetId}:like_count
  Kafka: tweet.unliked → DB'den sil.

Like sayısı okuma:
  GET /tweets/{tweetId} → tweet:{tweetId}:like_count Redis'ten → anlık.
  DB: eventual consistency (Kafka consumer gecikmeli sync).
  Tutarsızlık: 1-2sn → kabul edilebilir.

Viral tweet (10M like):
  like_count Redis INCR → tek key → hot key sorun.
  Çözüm: sharded counter.
    INCR tweet:{tweetId}:like_count:{shard_id}  (shard_id = rand 0-9)
    Okuma: MGET 10 shard key → topla → total.
    10x throughput artışı.
```

### Retweet

```
Retweet = Yeni tweet + parent_tweet referansı.

POST /tweets/{tweetId}/retweet

Retweet Service:
  1. Dedup: user:{userId}:retweets SET → zaten retweeted?
  2. Yeni tweet oluştur:
     {type: "RETWEET", original_tweet_id: tweetId, user_id: userId}
  3. Fan-out: yeni tweet olarak fan-out işlemi → takipçilere push.
  4. Original tweet'in retweet_count'u artır: INCR tweet:{tweetId}:rt_count.
  5. Tweet sahibine bildirim.

Timeline'da gösterim:
  type=RETWEET → "X retweeted" başlığı + original tweet içeriği.
  Silme: retweet silinirse → original tweet'in RT sayısı azalır.

Quote Tweet (Retweet with Comment):
  {type: "QUOTE", text: "Harika!", quoted_tweet_id: tweetId}
  Bağımsız tweet gibi davranır ama referans içerir.
  Fan-out: normal tweet fan-out.
```

---

## Reply / Thread (Tweet Zinciri)

```
Twitter thread = hiyerarşik reply ağacı.

Veri modeli:
  tweet {
    tweet_id, user_id, text,
    parent_tweet_id (NULL → root tweet),
    root_tweet_id (thread kökü, hızlı traversal için),
    reply_count, depth (0=root, 1=reply, 2=reply-to-reply)
  }

Thread okuma (GET /tweets/{tweetId}/thread):
  1. Root tweet çek.
  2. Replies: tweets WHERE parent_tweet_id = tweetId ORDER BY created_at.
  3. Her reply'ın kendi reply'ları (depth 2): lazy load (expand on click).
  4. Pagination: cursor-based (sonraki 20 reply).

Cassandra sorgusu:
  Tablo: replies_by_tweet (parent_tweet_id, tweet_id, created_at, user_id, text).
  PRIMARY KEY: (parent_tweet_id, tweet_id) CLUSTERING ORDER BY created_at ASC.
  
  SELECT * FROM replies_by_tweet WHERE parent_tweet_id = ? ORDER BY tweet_id ASC LIMIT 20.

Reply bildirimi:
  parent tweet sahibine → notification.
  "@ mention" olan kullanıcılara → mention notification.

Reply count güncelleme:
  INCR tweet:{parentTweetId}:reply_count (Redis, hızlı).
  Kafka event: sync to DB (async).
```

---

## Medya İşleme Pipeline

```
Fotoğraf yükleme:
  1. Client: POST /media/presigned-url → Media Service.
  2. Media Service: S3 presigned PUT URL üret → client'a döndür.
  3. Client: Direkt S3'e upload (Media Service'ten geçmez → bandwidth tasarrufu).
  4. S3 event → SNS/SQS → Image Processor (Lambda):
     a. Metadata çıkar (EXIF: konum, tarih).
     b. EXIF gizlilik: konum bilgisini sil.
     c. Boyutlar üret: 400px (mobil küçük), 800px (mobil büyük), 1200px (web).
     d. Format: WebP (Chrome/Android) + JPEG fallback (iOS < 14).
     e. CDN'e yükle: cdn.example.com/media/{mediaId}/{size}.webp.
  5. Client: media_id → tweet'e ekle → POST /tweets.

Video yükleme (daha karmaşık):
  1. Multipart upload: büyük dosya → S3 multipart.
  2. Video Processor (EC2 / ECS, CPU-heavy):
     a. Input: mp4 (herhangi).
     b. Output: HLS (m3u8 playlist + .ts segment) — adaptif bitrate.
        - 360p (600 Kbps), 480p (1 Mbps), 720p (2.5 Mbps), 1080p (4 Mbps).
        - Her 6 saniyede 1 segment.
     c. Thumbnail: 0s, 2s, 4s karelerden 3 thumbnail.
  3. CDN: video/{videoId}/playlist.m3u8 → stream.
  4. Player: HLS.js (web) / AVPlayer (iOS) / ExoPlayer (Android).
     → bandwidth'e göre otomatik kalite seç.

CDN stratejisi:
  Origin: S3 (us-east-1).
  Origin Shield: tek bir region CDN katmanı → S3'e origin pressure azaltır.
  Edge: 200+ PoP → kullanıcıya yakın.
  Cache-Control: immutable, max-age=31536000 (medya değişmez — content hash URL).
  URL: cdn.example.com/media/{sha256_hash}_{size}.webp → hash = içerik belirlenir, cache güvenli.
```

---

## Timeline Ranking (Algoritma Bazlı)

```
İki mod:
  1. "Takip Edilenler" (chronological): fan-out/pull merge, timestamp DESC.
  2. "Senin İçin" (For You): ML ranking model.

"Senin İçin" pipeline:

Adım 1 — Candidate retrieval:
  Push cache (normal users): son 500 tweet ID.
  Celebrity pull: son 50 tweet her celebrity.
  Trending: viral tweetler (like velocity).
  Toplam: ~1000 aday tweet.

Adım 2 — Feature extraction (per tweet):
  User features: takip süresi, etkileşim geçmişi bu kullanıcıyla.
  Tweet features: like_count, retweet_count, reply_count, medya var mı?.
  Context features: günün saati, cihaz tipi, session uzunluğu.
  Social proof: takip ettiğin kişiler beğendi mi?.

Adım 3 — Scoring (ML model):
  Model: LightGBM veya XGBoost (hızlı inference, < 20ms).
  Output: P(engagement) — kullanıcının tweet ile etkileşime gireceği olasılık.
  P(like) × 1 + P(reply) × 3 + P(retweet) × 2 + P(click) × 1 = final score.
  Reply en değerli (derin etkileşim).

Adım 4 — Filters:
  Blocked users: çıkar.
  Muted keywords: çıkar.
  Görülen tweet'ler (son 24 saat impression): tekrar gösterme.

Adım 5 — Diversification:
  Aynı kullanıcıdan max 2 tweet (sıkıcı olmasın).
  Medya çeşitliliği: tüm video veya tüm metin olmasın.
  Zamanlama: son 24 saat ağırlıklı.

Adım 6 — Return top 20.

Latency bütçesi:
  Candidate retrieval: 20ms (Redis + paralel celebrity pull).
  Feature extraction: 10ms (Redis multi-get).
  ML inference: 20ms (CPU, LightGBM).
  Filter + diversification: 5ms.
  Toplam: ~55ms (200ms SLA'nın altında).
```

---

## Tweet Silme

```
DELETE /tweets/{tweetId}

Tweet Service:
  1. Yetki kontrolü: tweet sahibi mi? (admin override için ayrı endpoint).
  2. Soft delete: Cassandra UPDATE → deleted_at = now(), is_deleted = true.
     (Hard delete yapmıyoruz → fan-out undo gerekirse, audit trail için).

Async cleanup (Kafka: tweet.deleted):
  a. Fan-out cleanup:
     Takipçilerin timeline'ından tweetId'yi sil.
     Sorun: 10M follower → 10M LREM → çok yavaş.
     Çözüm: filtreleme (read time):
       Timeline'dan tweet ID geldi → Tweet Service'ten fetch → deleted_at SET?
       → varsa: timeline'da gösterme, ID'yi temizle.
       Lazy cleanup: read sırasında silinen tweetleri filtrele.
       Background job: gecenin ilerleyen saatlerinde gerçek LREM.

  b. Cache invalidation:
     DEL tweet:{tweetId} (Redis tweet detay cache).
     DEL tweet:{tweetId}:like_count.

  c. Medya silme:
     S3: silmiyoruz (CDN cache expire olana kadar erişilebilir → kabul edilebilir).
     Gizlilik talebi (GDPR): S3 lifecycle → immediate deletion.

  d. Notification:
     Silen kişiye onay.
     Bildirimlerde görünen "tweet silinmiş" → "Bu tweet mevcut değil."

Retweet/Like cascade:
  Original tweet silindi → retweet'ler de silinmeli (görsel olarak).
  likes/retweets tablosunda: foreign key veya read-time kontrol.
  tweet.deleted event → likes + retweets soft delete (Kafka consumer).
```

---

## Notification Entegrasyonu

```
Notification türleri (sosyal feed özelinde):
  LIKE       → "Ali tweet'ini beğendi"
  RETWEET    → "Zeynep tweet'ini retweet etti"
  REPLY      → "Emre tweet'ine yanıt verdi: ..."
  FOLLOW     → "Can seni takip etmeye başladı"
  MENTION    → "Fatma senden bahsetti: ..."
  QUOTE      → "Ahmet tweet'ini alıntıladı"

Notification Service pipeline:
  Kafka event (tweet.liked, tweet.retweeted, etc.)
  → Notification Service → preference check (kullanıcı bu tür bildirimi istiyor mu?)
  → Push (FCM/APNs) veya in-app.

Batching (anti-spam):
  10 kişi aynı tweet'i beğendi → tek bildirim: "Ali ve 9 kişi beğendi".
  Sliding window: 30sn içinde aynı tweet + aynı action → batch.
  Redis: LPUSH notif:{userId}:{tweetId}:likes {actorId} EX 30.
  30sn sonra: list uzunluğu > 1 → "X ve Y kişi daha beğendi" → tek push.

Önceliklendirme:
  P0: mention, direct reply (hemen).
  P1: like, retweet (dakika içinde batch).
  P2: new follower, trending mention (saatlik digest).

Quiet hours:
  23:00 - 08:00 IANA timezone → push notification gönderme.
  In-app notification kuyruğunda tut → sabah iletilsin.
```

---

## Rate Limiting ve Anti-Abuse

```
Tweet rate limiting:
  Kullanıcı başına: 300 tweet / 3 saat.
  Redis: INCR user:{userId}:tweet_count:{window} → EXPIRE 10800.
  Aşılırsa: 429 Too Many Requests + Retry-After header.

Like/Retweet limitleri:
  1000 like/gün (bot önlemi).
  100 follow/gün (spam follow önlemi).
  Redis sliding window counter.

Spam tespiti (async):
  Kafka: tweet.created → Spam Detector:
    - Aynı URL'yi 10 tweet → spam.
    - Yeni hesap (< 1 hafta) + 100 tweet/gün → şüpheli.
    - ML model: text + metadata → P(spam).
    - Spam ise: shadow ban (tweet görünür ama yayılmaz) veya suspend.

IP-based rate limiting:
  Registration: 5 hesap / IP / gün (bot hesap açma önleme).
  Anonymous read: 100 request / IP / dakika.
  JWT bearer: kullanıcı kimliği ile limit.

Content moderation:
  Tweet oluşturulurken async:
    → Text: banned keyword list → otomatik filtre.
    → Medya: nudity/violence detection (ML, rekabetçi öneri: AWS Rekognition).
    → Report: kullanıcı şikayet → moderatör kuyruğu.
  Moderatör aksiyon: delete, shadow ban, suspend, warn.
```

---

## Cache Warming Stratejisi

```
Sorun 1: Yeni kullanıcı (ilk giriş) → timeline cache boş.
Çözüm: Registration sonrası async warm:
  Kafka: user.registered → Timeline Warmer Service.
  Warmer: onboarding'de önerilen hesapları takip etti → onların son 50 tweet'ini çek.
  Redis'e yaz: user:{userId}:timeline → pre-populated.
  Kullanıcı timeline açınca: cache hit → anlık.

Sorun 2: Uzun süredir giriş yapmamış kullanıcı (cache expired).
Çözüm: Login event → async warm (cache miss'i önce kabul et):
  Login → warm job tetikle (Kafka: user.loggedin).
  İlk istek: cache miss → pull rebuild (500ms latency kabul edilebilir, nadir durum).
  Background: warm tamamlandı → sonraki istekler cache hit.

Sorun 3: Sayfanın alt kısmı (scroll down, daha eski tweet'ler).
Çözüm: infinite scroll cursor + on-demand Cassandra fetch:
  Redis'te son 1000 tweet ID var → sonrası Cassandra'dan çek.
  Pre-fetch: kullanıcı 80% scroll'da → arka planda sonraki sayfayı yükle.

Sorun 4: Deployment sonrası warm Redis'i doldurma.
Çözüm: Gradual traffic shifting:
  Yeni Redis Cluster → önce %10 traffic → warm up 6 saat → %100.
```

---

## Social Graph Ölçeği

```
150B follows edge → nasıl sakla ve sorgula?

Cassandra (source of truth):
  follows tablosu: (follower_id, following_id, ts) → 150B row → 6 TB.
  followers_by_user: (user_id, follower_id) → fan-out için.
  following_by_user: (user_id, following_id) → timeline için.

  Sorgu: "Kim takip ediyor?" → followers_by_user WHERE user_id = X LIMIT 1000.
  Celebrity (100M follower): sayfalama gerekir.

Redis Cache (sık kullanılan grafikler):
  user:{userId}:following → Sorted Set: {following_id → follow_ts}.
  user:{userId}:followers_count → INT (INCR/DECR).
  TTL: aktif kullanıcı için 24 saat.
  Invalidation: follow/unfollow eventi → Redis güncelle.

Fan-out için follower fetch:
  Celebrity: 100M follower → Cassandra'dan okumak için sayfalı query.
    LIMIT 10,000 → 10,000 iterasyon → seri çok yavaş.
  Çözüm:
    a. Kafka partition key: follower_id → paralel N consumer.
    b. Pre-computed follower shard: follower listesini önceden shard listelerine böl.
    c. Fan-out worker: her shard → paralel pipeline → toplam süre: ~5sn for 100M.
    d. 5sn gecikmeli fan-out: celebrity tweet → 5sn sonra herkese ulaşır → kabul edilebilir.

Neo4j (graph traversal):
  "Arkadaşlarının arkadaşları" (2nd degree connection): Neo4j güçlüdür.
  BFS traversal: MATCH (a)-[:FOLLOWS]->(b)-[:FOLLOWS]->(c) WHERE a.id=X.
  Twitter scale'de: Neo4j değil, FlockDB (Twitter'ın özel graph DB) veya Cassandra.
  Alternatif: Redis + Cassandra + uygulama katmanında BFS.
```

---

## Cursor-Based Pagination

```
Sorun: LIMIT/OFFSET → büyük offset yavaş ("offset 1000 → 1000 row skip").
Çözüm: Cursor (keyset pagination).

Cursor = son görülen tweet'in Snowflake ID'si.

İlk sayfa:
  GET /timeline → {tweets[20], nextCursor: "1234567890"}

Sonraki sayfa:
  GET /timeline?cursor=1234567890
  → Tweet ID'leri Snowflake'e göre azalan sırada.
  → WHERE tweet_id < 1234567890 LIMIT 20.
  → Cassandra: TIMEUUID'ye göre range scan → verimli.

Redis timeline cursor:
  LRANGE user:{userId}:timeline 0 999 → 1000 tweet ID'yi bellek'te tut.
  Cursor: array index değil, tweet_id → binary search → pozisyon bul.
  Sonraki sayfa: cursor pozisyonundan sonraki 20.

Timestamp manipülasyon önlemi:
  Kullanıcı cursor'ı değiştirirse (farklı tweet_id) → sadece kendi timeline'ındaki ID'leri görebilir.
  Başkasının tweet ID'sini cursor olarak kullanırsa → erişim kontrolü gerekli.

Pagination metadata:
  {
    tweets: [...],
    nextCursor: "sf8h3k2j...",    ← base64 encoded tweet_id
    hasMore: true,
    generatedAt: 1705312800       ← cache freshness
  }
```

---

## Veri Modeli

```sql
-- Tweet storage (Cassandra)
CREATE TABLE tweets_by_user (
    user_id    UUID,
    tweet_id   BIGINT,         -- Snowflake ID
    text       TEXT,
    type       TEXT,           -- 'TWEET', 'RETWEET', 'QUOTE', 'REPLY'
    parent_tweet_id BIGINT,
    root_tweet_id   BIGINT,
    original_tweet_id BIGINT,  -- RETWEET için
    media_keys LIST<TEXT>,
    is_deleted BOOLEAN DEFAULT false,
    deleted_at TIMESTAMP,
    created_at TIMESTAMP,
    PRIMARY KEY ((user_id), tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);

-- Tweet detail lookup (Cassandra)
CREATE TABLE tweets_by_id (
    tweet_id   BIGINT PRIMARY KEY,
    user_id    UUID,
    text       TEXT,
    type       TEXT,
    parent_tweet_id BIGINT,
    root_tweet_id   BIGINT,
    original_tweet_id BIGINT,
    media_keys LIST<TEXT>,
    is_deleted BOOLEAN DEFAULT false,
    created_at TIMESTAMP
);

-- Reply lookup (Cassandra)
CREATE TABLE replies_by_tweet (
    parent_tweet_id BIGINT,
    tweet_id        BIGINT,
    user_id         UUID,
    text            TEXT,
    created_at      TIMESTAMP,
    PRIMARY KEY ((parent_tweet_id), tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id ASC);

-- Social graph (Cassandra)
CREATE TABLE followers_by_user (
    user_id     UUID,
    follower_id UUID,
    created_at  TIMESTAMP,
    PRIMARY KEY ((user_id), follower_id)
);

CREATE TABLE following_by_user (
    user_id      UUID,
    following_id UUID,
    created_at   TIMESTAMP,
    PRIMARY KEY ((user_id), following_id)
);

-- Likes (MySQL — daha az volume, ACID gerekli)
CREATE TABLE likes (
    user_id    BINARY(16) NOT NULL,
    tweet_id   BIGINT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    INDEX idx_tweet_id (tweet_id)
) ENGINE=InnoDB;

-- Media metadata (MySQL)
CREATE TABLE media (
    media_id   VARCHAR(64) PRIMARY KEY,
    tweet_id   BIGINT NOT NULL,
    user_id    BINARY(16) NOT NULL,
    type       ENUM('IMAGE', 'VIDEO', 'GIF') NOT NULL,
    s3_key     VARCHAR(255) NOT NULL,
    width      INT,
    height     INT,
    duration_s INT,     -- video için
    status     ENUM('PENDING', 'PROCESSING', 'READY', 'FAILED') DEFAULT 'PENDING',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_tweet_id (tweet_id)
);

-- Redis key referansı:
-- user:{userId}:timeline           → LIST [tweetId ...] (max 1000, TTL 7gün)
-- tweet:{tweetId}:like_count:{shard}→ INT (sharded counter, 10 shard)
-- tweet:{tweetId}:rt_count         → INT
-- tweet:{tweetId}:reply_count      → INT
-- tweet:{tweetId}                  → HASH {userId, text, createdAt, type} TTL 3600
-- user:{userId}:likes              → SET {tweetId ...} (TTL 24saat, son 1000)
-- user:{userId}:following          → ZSET {followingId → followTs} TTL 24saat
-- user:{userId}:followers_count    → INT
-- rate:{userId}:tweets:{window}    → INT (TTL 10800)
```

---

## Olası Sorunlar ve Çözümleri

### 1. Celebrity Fan-Out Fırtınası — Beyoncé 100M Takipçiye Tweet Attı

```
Sorun:
  100M takipçiye sahip celebrity tweet attı.
  Fan-out worker: 100M × Redis LPUSH → 100M işlem.
  Redis cluster: saniyede 1M yazma kapasitesi → 100 saniye sürer.
  Bu süre zarfında: fan-out kuyruk birikir, diğer kullanıcılar gecikmeli.

Çözüm:
  a) Hybrid model (birincil):
     Celebrity (> 1M follower) için fan-out YOK.
     Timeline okuma: celebrity'nin son tweetleri pull with merge.
     100M LPUSH → 0. Sorun çözüldü.

  b) Async delayed fan-out (gerekirse):
     Fan-out'u hemen değil, 30sn içinde tamamla.
     Rate limited fan-out: 100K write/sn → 100M / 100K = 1000sn (~17dk).
     Acceptable: celebrity tweet'i 17dk içinde herkese ulaşır.

  c) Kafka fan-out parallelizm:
     32 partition → 32 consumer → 32 paralel Redis pipeline.
     32 × 1000 write/pipeline = 32,000 pipeline/sn.
     100M / (32 × 1000) = 3125 pipeline → daha hızlı.

  d) Throttle: celebrity fan-out için ayrı "low priority" Kafka topic.
     Normal kullanıcı fan-out önce işlenir, celebrity arka planda.
```

---

### 2. Timeline Tutarsızlığı — Push + Pull Merge'de Sıra Bozuluyor

```
Sorun:
  Push cache: normal kullanıcı tweet'leri (Snowflake ID sıralı).
  Pull: celebrity'nin Cassandra'dan gelen tweet'leri (farklı zaman damgası formatı).
  Merge sort: zaman damgası saniye hassasiyeti → aynı saniyede 2 tweet → sıra deterministik değil.
  Sonuç: her timeline refresh'te farklı sıra → kullanıcı "tweet kayboldu" hisseder.

Çözüm:
  a) Snowflake ID ile merge sort (milisaniye hassasiyeti):
     Cassandra'daki tweet'ler Snowflake ID ile depolanır.
     Merge: tweet_id DESC → milisaniye hassasiyetinde deterministik sıra.

  b) Deduplication:
     Aynı tweet_id push cache + pull'da olabilir.
     Set kullan: seen = Set(), tweet_id in seen → atla.

  c) Stable sort guarantee:
     Aynı timestamp → tweet_id büyük olan önce (ID monoton artar).
     Deterministik sıra → kullanıcı deneyimi tutarlı.

  d) Merge cache:
     Merged timeline'ı Redis'te cache'le (30sn TTL).
     30sn içinde aynı kullanıcı refresh → aynı sonuç (stable UX).
```

---

### 3. Viral Tweet Cache Stampede

```
Sorun:
  Tweet aniden viral: 10M kullanıcı aynı anda timeline'ı açıyor.
  Bu tweet → Redis'te cache'li değil (yeni) → 10M eşzamanlı Cassandra sorgusu.
  Cassandra: OOM veya timeout → tüm servis yavaşlar.

Çözüm:
  a) Read-through cache (auto-populate):
     Tweet fetch → Redis cache miss → Cassandra'dan çek → Redis'e yaz → döndür.
     İlk istek: Cassandra. Sonrakiler: Redis cache hit.
     Sorun: 10M aynı anda ilk istek → 10M Cassandra sorgu (hâlâ sorun).

  b) Mutex lock (single population):
     Cache miss → Redis SETNX tweet:{tweetId}:loading 1 EX 5.
     SETNX başarılı → sen yükle, Redis'e yaz.
     SETNX başarısız → başkası yüklüyor → 50ms bekle → retry.
     Sonuç: sadece 1 Cassandra sorgu, geri kalanı bekler.

  c) Probabilistic early expiry (PER):
     Cache TTL dolmadan: (kalan_süre < rastgele_süre) ise → yenile.
     Yenileme: 1 thread erken → TTL dolmadan yüklüyor.
     Diğerleri: eski cache okumaya devam → stampede yok.

  d) CDN + Edge cache:
     Public tweet (herkese açık): CDN'de cache (5sn TTL).
     Viral tweet → CDN hit → Cassandra'ya sıfır istek.
     Sadece private veya auth-gated tweet için origin hit.
```

---

### 4. Hot Partition — Kafka'da Celebrity Tweet Fan-Out Tıkandı

```
Sorun:
  Kafka topic: tweet.created, partition key: user_id.
  Celebrity user_id → her tweeti aynı partition'a → hot partition.
  Bu partition: consumer bir tane → bottleneck.
  Diğer partitionlar: boş bekliyor.

Çözüm:
  a) Fan-out için partition key: followerId, tweet_id değil.
     Fan-out mesajları: her follower için ayrı message (tweet.created'da değil,
     fan-out.tasks topic'inde) → partition key = followerId → dağıtık.

  b) Celebrity için ayrı topic:
     tweet.celebrity.created → 64 partition (normal'den daha fazla).
     Consumer group: daha fazla consumer.
     İzole: celebrity storm normal fan-out'u etkilemez.

  c) Sticky partition'a sahip olmayan producer:
     Round-robin partition → her partition eşit yük.
     Sorun: sıra garantisi kaybolur → kabul edilebilir (fan-out için sıra önemli değil).

  d) Kafka Streams'de repartition:
     celebrity tweet event → repartition by followerId → dağıtık consumer.
```

---

### 5. Like Sayısı Tutarsızlığı — Viral Tweet'te 1M Like Gözüküyor, DB'de 900K

```
Sorun:
  Like count: Redis INCR (hızlı, eventual) + DB persist (Kafka, gecikmeli).
  Redis count: 1M (gerçek zamanlı).
  DB (MySQL likes tablo): 900K (Kafka consumer 100K geride).
  Rapor/analitik: DB'den okuyor → yanlış rakam.

Çözüm:
  a) UI her zaman Redis'ten okur → kullanıcı doğru sayı görür.
     Analytics/rapor: eventual consistency kabul eder → Kafka backlog eridiğinde senkronize.

  b) Dual write with reconciliation:
     Her 5 dakikada bir: Redis count vs DB count → fark varsa → alarm.
     Günlük reconciliation job: DB'de eksik like'ları Redis'ten düzelt.

  c) Sharded counter (hem hız hem doğruluk):
     10 shard: INCR tweet:{id}:like_count:{0-9}.
     Okuma: MGET 10 key → topla → doğru sayı.
     DB persist: her shard ayrı → paralel → daha hızlı senkron.

  d) Read repair pattern:
     Okuma sırasında: Redis count > DB count → farkı DB'ye ekle.
     Eventual convergence: her read → DB ve Redis senkronize yaklaşır.
```

---

### 6. Inactive User Cache Şişirmesi — 60 TB Redis, %40'ı Ölü Hesap

```
Sorun:
  300M DAU derken 300M Redis key → 60 TB.
  Ama gerçek aktif kullanıcı: 50M / gün.
  Geri 250M: aylardır giriş yapmıyor → timeline cache'i boşuna tutuyor.
  Redis: 60 TB × %40 = 24 TB israf.

Çözüm:
  a) TTL stratejisi (birincil):
     Timeline cache TTL: kullanıcının son aktivitesine göre.
     Son 7 gün aktif: TTL 7 gün.
     7-30 gün: TTL 1 gün (giriş yapınca rebuild).
     30+ gün: cache hiç tutma (giriş gelince on-demand rebuild).
     Sonuç: Redis → 480 GB (sadece aktif kullanıcı).

  b) Activity tracking:
     user.last_active → Redis hash → günlük güncelle.
     Fan-out: user.last_active < now - 30 gün → SKIP (cache'e yazma).
     Giriş yapınca: warm timeline on-demand.

  c) Tiered storage:
     Aktif (< 7 gün): Redis (hızlı).
     Yarı aktif (7-30 gün): Redis + disk-backed (Redis on Flash / KeyDB).
     Inactive (30+ gün): silindi → login gelince Cassandra'dan rebuild.

  d) Monitoring:
     Redis key count, bellek kullanımı → alarm: bellek > %80.
     Düzenli LRU eviction policy: allkeys-lru → Redis otomatik eskiyi siler.
```

---

### 7. Tweet Silme Propagasyonu — 30M Takipçi Timeline'ından Silmek

```
Sorun:
  @ünlü 30M takipçiye sahip tweet atti sonra sildi.
  Fan-out: tweet tüm 30M timeline'a push'landı.
  Silme: 30M Redis LREM (tweetId listeden sil) → imkansız süre.

Çözüm:
  a) Lazy deletion (birincil — en pratik):
     Tweet soft delete → is_deleted=true, deleted_at=now.
     Timeline okuma: tweet ID'lerini hydrate ederken → is_deleted=true → filtrele.
     Kullanıcı timeline'ında görünmez → "deleted" flag kontrol yeterli.
     Cost: her tweet hydration'da 1 flag check → trivial.

  b) Redis LREM (arka planda, yavaşça):
     Kafka: tweet.deleted → low-priority consumer.
     Background job: 30M timeline'dan yavaş yavaş LREM.
     Rate limited: 100K LREM/sn → 30M / 100K = 300sn (~5dk).
     Bu süre: lazy deletion zaten kapsıyor → kullanıcı silinmiş tweet görmüyor.

  c) Delete event'ı timeline'a ekle:
     Redis: özel "DELETED:{tweetId}" token → timeline listesine ekle.
     Okuma: "DELETED" token → o pozisyona "tweet silinmiş" yaz (veya atla).
     Avantaj: LREM'e gerek yok, O(1) operasyon.

  d) Bloom filter for deleted tweets:
     Silinen tweet ID'leri → Bloom Filter.
     Timeline hydration: ID → bloom filter → muhtemelen silinmiş → DB kontrol.
     Fast path: bloom filter miss → kesinlikle silinmemiş → direkt serve.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Fan-out modeli | Push (on write) | Hybrid (push normal, pull celebrity) | Push: celebrity 100M write; Pull: 500 sorgu; Hybrid: ikisinin avantajı |
| Tweet ID | UUID (random) | Snowflake ID (zaman bazlı) | UUID: sıralanabilir değil; Snowflake: sıralı, dağıtık, zaman bilgisi dahil |
| Tweet storage | MySQL | Cassandra | MySQL: yatay scale zor; Cassandra: write-heavy, yatay scale |
| Social graph | Neo4j | Cassandra + Redis cache | Neo4j: graph traversal güçlü ama 150B edge'de ölçek zor; Cassandra yeterli |
| Like counter | DB count | Redis INCR + async DB | DB: her like transaction; Redis: < 1ms, async persist |
| Timeline ranking | Kronolojik | Hybrid (chrono + ML "For You") | Kronolojik: basit; ML: daha iyi engagement ama bias riski |
| Cache strategy | Cache all users | TTL-based (aktif kullanıcı) | All: 60 TB israf; TTL: 480 GB → 125x tasarruf |
| Tweet silme | Hard delete | Soft delete + lazy cleanup | Hard delete: fan-out undo imkansız; Soft: basit, hızlı, audit trail |
| Medya upload | Server-side | Presigned S3 direct upload | Server-side: bandwidth waste; Presigned: client direkt upload |
| Video format | MP4 | HLS (adaptive bitrate) | MP4: tek kalite; HLS: bandwidth'e göre otomatik kalite seç |
| Pagination | LIMIT/OFFSET | Cursor (Snowflake ID) | OFFSET: büyük offset'te O(N) tarama; Cursor: O(log N), tutarlı |
| Notification batch | Her event | Sliding window 30sn batch | Her event: spam; Batch: "X ve 9 kişi beğendi" → daha iyi UX |
