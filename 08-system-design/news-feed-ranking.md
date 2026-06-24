# 08t — System Design: News Feed Sıralama (Facebook / LinkedIn)

## Gereksinimler

```
Functional:
  ✓ Kullanıcının bağlantılarından gelen içerikleri göster
  ✓ Puanlama ve sıralama (ML tabanlı)
  ✓ İçerik türleri: post, fotoğraf, video, paylaşım, link
  ✓ Gerçek zamanlı güncelleme (yeni post → feed'de görünsün)
  ✓ Sonsuz scroll (cursor-based pagination)
  ✓ Negatif sinyal (gizle, şikayet, takipten çık)
  ✓ Reklam entegrasyonu (her 15 posttan biri)
  ✗ Stories, Reels, Gruplar (out of scope)

Non-Functional:
  DAU: 500M
  Feed açılma: 500M × 5/gün = 2.5B/gün
  Feed QPS: 2.5B / 86400 ≈ 29,000/sn
  Post write: 500M × 2/gün / 86400 ≈ 11,600 post/sn
  Fan-out: 11,600 × ortalama 300 arkadaş = 3.5M feed write/sn (burst)
  Feed yükleme: < 200ms (p99)
  ML Scoring: < 50ms (top 1000 candidate için)
```

---

## Capacity Estimation

```
Post storage:
  11,600 post/sn × 500 byte (text + metadata) = 5.8 MB/sn → 500 GB/gün
  Media (img/video): ayrı CDN, bu hesapta yok.
  Cassandra: 3 replica × 500 GB = 1.5 TB/gün → 45 TB/ay.

Feed cache (Redis):
  500M kullanıcı × 200 postId (cache'teki feed) × 8 byte (postId) = 800 GB.
  Sorted set overhead: ~2x → ~1.6 TB Redis RAM → cluster şart.
  Eviction: 72 saat görülmeyen feed → TTL ile sil (aktif kullanıcılar önce).

Fan-out write yükü:
  Normal kullanıcı (300 arkadaş): post → 300 Redis ZADD.
  Celebrity (10M follower): fan-out on READ (çözüm aşağıda).
  Ortalama: 11,600/sn × 300 = 3.5M Redis write/sn → cluster + pipeline.

Affinity score storage:
  500M kullanıcı × ortalama 300 arkadaş = 150B çift.
  Her çift: 8 byte (score) → 1.2 TB → Redis Cluster veya DynamoDB.
  Haftalık recompute: Spark batch job.

ML inference:
  29,000 feed QPS × ortalama 1,000 candidate = 29M scoring/sn.
  LightGBM inference: ~0.01ms/post → 29M × 0.01ms = 290,000ms/sn.
  10,000 CPU core ile: 29 ms/core/sn → pratik (horizontal scale).
```

---

## Twitter Feed'den Farkı

```
Twitter (08d): Takip et → chronological → basit
Facebook: Friend graph (çift taraflı) + ML ranking → karmaşık

Farklar:
  1. İki yönlü friend (A-B arkadaş → ikisi de birbirini görür)
     Twitter: tek yönlü follow

  2. Sıralama: Kronolojik değil, "senin için en iyi" (ML)
     Etkileşim oranı, içerik türü tercihi, yakınlık skoru

  3. İçerik çeşitliliği: post + video + share + event + ad
     Mix oranı önemli (çok ad → kullanıcı kaçar)

  4. Graph karmaşıklığı: 500 arkadaş × post → çok fazla içerik
     Filtreleme ve ranking kritik

  5. Gizlilik: arkadaşın arkadaşı görmemeli, visibility kontrol.
     Twitter: public by default.
```

---

## High-Level Mimari

```
             Client (Mobile / Web)
                  │  GET /feed?cursor=...
                  ▼
         ┌─────────────────┐
         │   Feed Service  │  ← stateless, horizontal scale
         └────────┬────────┘
                  │
     ┌────────────▼────────────────────────┐
     │         Feed Generation Pipeline    │
     │                                     │
     │  1. [Cache Hit?] Redis feed:userId  │  ← P99: 5ms
     │       ↓ Miss                        │
     │  2. Candidate Selection             │  ← Graph DB + Post DB
     │  3. Filtering (blocked, seen, nsfw) │
     │  4. ML Ranking (Two-Tower + Rerank) │  ← 50ms budget
     │  5. Diversity & Mixing              │
     │  6. Ad Insertion (auction)          │
     │  7. Cache Write (Redis ZADD)        │
     └────────────┬────────────────────────┘
                  │
    ┌─────────────┼──────────────┬──────────────┐
    ▼             ▼              ▼              ▼
[Post DB]    [Graph DB]     [ML Serving]   [Ad Service]
(Cassandra)  (Redis/Neo4j)  (TensorFlow    (Auction +
                             Serving)       Budget Ctrl)

Write path (yeni post):
  Client → Post Service → Cassandra → Kafka "new-post" topic
                                           │
                                    ┌──────▼───────┐
                                    │  Fan-out Svc  │
                                    └──────┬────────┘
                                           │ Redis feed:userId ZADD
                                    ┌──────▼────────┐
                                    │  WebSocket Svc │ → "Yeni post" push
                                    └───────────────┘
```

---

## Feed Oluşturma Pipeline

### Aşama 1: Candidate Selection

```
Kullanıcının arkadaş listesi → Graph DB veya Redis:
  friends:{userId} = [friendId1, friendId2, ..., friendId500]

Her arkadaş için: son 48 saatteki postlar (Cassandra range query):
  SELECT post_id FROM posts_by_author
  WHERE author_id = :friendId
  AND created_at > NOW() - 48h
  LIMIT 20;  ← arkadaş başına max 20

Paralel execution:
  500 arkadaş → 500 Cassandra sorgusu paralel → async batch.
  Toplam: ~35,000 candidate → filtreleme + ranking'e gir.

Optimizasyon:
  a. Yakınlık skoru düşük arkadaşlar: son 24 saat ile sınırla.
  b. Aktif arkadaşlar (bugün post attı): önce kontrol et.
  c. PostId'leri zaten feed cache'te varsa: skip (seen).
  d. Hedef: 35,000 → 1,000 candidate (filtreleme öncesi ML budget).

Ek kaynaklar (discovery):
  Arkadaşların beğendiği postlar (2nd degree):
    friend beğendi → bu post'u %20 ihtimalle candidate havuzuna ekle.
  Önerilen sayfalar (Page rank, topic matching):
    Kullanıcının ilgi alanları → benzer topic'teki public postlar.
```

---

### Aşama 2: Filtering

```
Sıralı filtreler (hızlıdan yavaşa):

1. Seen filter (Redis BitSet, çok hızlı):
   seen:{userId} BitSet → bit[postId % 2^32] == 1 → skip.
   BitSet boyutu: 2^32 bit = 512 MB per user → çok büyük.
   Alternatif: Bloom Filter (false positive %1 kabul edilebilir, "gördüm" dedi ama görmedi → ok).
   seen_posts:{userId} Bloom Filter → 1 MB per user → 500M × 1MB = 500 TB → hâlâ fazla.
   Pratik: son 7 günde görülen postId listesi → Redis List, max 500 item → LPOS.

2. Block filter (DynamoDB, fast KV):
   blocked_users:{userId} set → author_id → blocked mi?

3. Unfollow filter:
   unfollowed:{userId} set → feed'den gizlendi mi?

4. Content type filter:
   user_preferences:{userId} → video_hidden=true → tüm VIDEO postlar çıkar.

5. NSFW filter (ML tabanlı, async):
   Post moderasyon servisi: text + image → offensive score > 0.9 → çıkar.
   Bu asenkron: post yazılınca → moderasyon → etiket → candidate selection'da filtrele.

6. Low quality filter:
   post_engagement_rate < 0.01 (çok az etkileşim) → %80 ihtimalle çıkar.
   Yeni post (<1 saat): etkileşim süresi dolmadı → bu filtre uygulanmaz.

Sonuç: 35,000 → 1,000 candidate → ML Ranking'e gir.
```

---

### Aşama 3: ML Ranking (İki Aşamalı)

#### Two-Tower Model (Embedding-Based Retrieval)

```
Amaç: 1000 candidate'ı hızlıca 200'e indirge.

User Tower:                    Content Tower:
  userId → embedding           postId → embedding
  (kullanıcı tercihleri,       (post text, media type,
   etkileşim geçmişi,          creator özellikleri,
   demografik)                  engagement velocity)
  → 128-dim vector             → 128-dim vector

Benzerlik: dot product → cosine similarity.
ANN (Approximate Nearest Neighbor): FAISS → en yakın 200 post.
Avantaj: O(N) yerine O(log N) → büyük candidate havuzunda hızlı.

Günlük offline training:
  User/Content embedding'leri → Spark job → güncelle.
  Embedding store: Redis veya FAISS index (memory-mapped).
```

#### Reranker (Gradient Boosting / Neural)

```
200 candidate → tam feature hesapla → LightGBM veya 2-layer NN.

Feature vector (her post için):
  ┌─ Creator özellikler ─────────────────────────────────┐
  │  creator_affinity_score    (bu kullanıcıyla etkileşim)│
  │  creator_post_frequency    (spam mi?)                 │
  │  creator_avg_engagement    (popüler mi?)              │
  ├─ Post özellikler ────────────────────────────────────┤
  │  post_age_hours            (ne kadar yeni?)           │
  │  post_type                 (VIDEO, IMAGE, TEXT)       │
  │  post_like_velocity        (ilk 1 saatte kaç like?)  │
  │  post_comment_velocity                                │
  │  post_share_velocity       (viral mı?)                │
  │  media_quality_score       (blur, low-res → düşür)   │
  ├─ User özellikler ────────────────────────────────────┤
  │  user_content_type_pref    (VIDEO sever mi?)          │
  │  user_time_of_day          (sabah feed vs akşam)      │
  │  user_device               (mobile → kısa video)      │
  │  user_session_length       (ne kadar scroll yapıyor?) │
  ├─ Contextual ─────────────────────────────────────────┤
  │  friend_reactions          (5 arkadaş beğendi)        │
  │  topic_match               (kullanıcı ilgi alanları)  │
  └──────────────────────────────────────────────────────┘

Çıkış (multi-task learning — tek model, çok görev):
  P(like)    × w1
  P(comment) × w2
  P(share)   × w3
  P(click)   × w4
  P(hide)    × (-w5)   ← negatif sinyal
  P(long_view) × w6   ← video için izlenme süresi
  ─────────────────────
  engagement_score     → sıralama

Weight'ler: product goal'e göre tune edilir.
  "Daha fazla paylaşım" hedefi → w3 artır.
  "Kullanıcı mutluluğu" → hide/unfollow negatif signal ağırlığı artır.
```

---

### Aşama 4: Ranking Sinyalleri Detayı

```
Affinity Score (Yakınlık):
  Son 30 günde kullanıcı A → kullanıcı B için:
    Comment: +10 puan
    Like:    +3 puan
    View (3sn+): +1 puan
    Share:   +7 puan
    Profile visit: +5 puan
  Toplam normalize → 0-1
  Depolama: affinity:{userId}:{friendId} → Redis Hash veya DynamoDB.
  Haftalık Spark job: tüm affinity'leri yeniden hesapla.

Time Decay:
  Yeni post → yüksek tavan skoru.
  rank_score = base_score × e^(-λ × age_in_hours)
  λ = 0.1 → 7 saatte %50 skor kaybı (haber tipi).
  λ = 0.02 → 35 saatte %50 (evergreen içerik — tutorial, albüm).
  İçerik türüne göre farklı λ.

Virality Boost:
  Post paylaşımları hızla artıyorsa → viral sinyal.
  share_velocity = share_count / max(post_age_hours, 1)
  Üstel boost: virality_score = min(log(share_velocity + 1) / 5, 1.0)

Social Proof:
  "5 arkadaşın beğendi" → güçlü sinyal → friend_reaction_bonus.
  Arkadaşın yorumu → çok güçlü sinyal → hemen öne çıkar.

Content Type Preference:
  Kullanıcının son 30 günlük etkileşimi:
    video_clicks / total_clicks → video_preference_ratio.
  Feature olarak modele girer → model otomatik ağırlıklandırır.
```

---

### Aşama 5: Mixing & Diversity

```
Top 200 ML-scored post → diversity kuralları:

1. Same creator cap: aynı kişiden max 3 ardışık post.
   Önce: [A, A, A, A, B] → Sonra: [A, A, A, B, A]

2. Content type balance: her 10 posttan en az 2 farklı tür.
   Kullanıcı sadece TEXT görüyorsa → 1-2 video ekle.

3. Topic diversity: aynı konu çok fazlaysa → diğer konuları dahil et.
   Seçim sorunu: "Görülmesi gereken her şey görüldü mü?"
   Maximal Marginal Relevance (MMR): relevance - similarity_to_seen_items.

4. Time diversity: çok eski postlar varsa → yen post tanıt.

5. Filter bubble önleme (Facebook tartışmalı karar):
   Diversity score ekle: farklı görüşlü kaynaklardan içerik.
   Pratikte: politics konusunda az uygulandı (engagement düşüyor).

Reklam ekleme (sonraki bölüm ayrı ele alındı):
  Her 15 posttan 1 reklam → pozisyon: {5, 20, 35, ...}.
  Reklam kalitesi çok düşükse → atlat, bir sonraki pozisyona.
```

---

## Fan-Out Stratejisi

### Pull Model (Lazy Generation)

```
Feed isteği geldi → Arkadaşların postlarını çek → rank et → göster.

Avantaj:
  ✓ Write basit (sadece Post DB'ye kaydet).
  ✓ Arkadaş sayısına yazma bağımsız.

Dezavantaj:
  ✗ Feed oluşturma her istekte → yavaş (500 arkadaş × DB query).
  ✗ Yüksek QPS'de DB yükü: 29,000 QPS × 500 arkadaş = 14.5M Cassandra query/sn.

Kullanım: küçük sistemler, anlık doğruluk önemli.
```

### Pre-Computed Feed Cache

```
Yeni post yazıldı → Fan-out Service (Kafka consumer):
  Yazarın arkadaşlarına (500 kişi) → feed_cache güncelle.

Redis sorted set (score = ML ranking skoru):
  ZADD feed:{userId} {score} {postId}
  TTL: 72 saat (pasif kullanıcı için cache expire).
  ZREVRANGE feed:{userId} 0 19  → top 20 postId.

Feed okuma (P99: 5ms):
  1. Redis ZREVRANGE feed:{userId} 0 199 → top 200 postId.
  2. Filtering (seen, blocked) → 200 → 50.
  3. Post DB'den batch fetch → Cassandra IN sorgusu.
  4. Render → response.

Fan-out write optimizasyon:
  Kafka: "new-post" topic → fan-out consumer group.
  Paralel partition işleme: 10,000 arkadaşlı yazar → 10 partition × 1,000 paralel.
  Pipeline: Redis ZADD pipeline → 500 write → 1 RTT (pipeline batching).
```

### Hybrid (Celebrity Problem Çözümü)

```
Sorun:
  Cristiano Ronaldo (150M follower) post attı
  → 150M Redis ZADD → dakikalarca sürer → stale feed.

Çözüm (Hybrid Fan-out):
  Normal kullanıcı (< 5,000 arkadaş/takipçi): Fan-out on WRITE.
    Post yazıldı → anında tüm arkadaşların feed cache'i güncellenir.

  Celebrity / Influencer (> 5,000): Fan-out on READ.
    Post yazıldı → sadece Post DB'ye yaz.
    Feed okuma: kendi pre-computed cache + celebrity postları (pull) → merge.

  Feed oluşturma (hybrid):
    1. Redis feed:{userId} → cached posts (normal arkadaşlar).
    2. Celebrity listesi → son 1 saatlik postları Cassandra'dan çek.
    3. İki listeyi merge et → rank → serve.

  Celebrity listesi:
    Kullanıcının takip ettiği celebrity'lar → Redis Set: celebrities:{userId}.
    Feed request'inde: bu listeyi dön, celebrity postlarını çek, merge et.

  Threshold: 5,000 follower → elastik, trafik yüküne göre ayarlanabilir.
```

---

## Reklam Entegrasyonu

```
Reklam sıralaması (Auction):
  Reklam slot'u açıldı (her 15 posttan biri):
    Advertiser bid (MAX CPC) × predicted CTR (ML) = auction score.
    En yüksek score → kazanır.
    Gerçek maliyet: 2. en yüksek + $0.01 (Vickrey auction).

  Predicted CTR modeli:
    User features + Ad features → P(click).
    Benzer kullanıcılar bu reklamı tıkladı mı?
    Ad image quality, title relevance → CTR artırır.

  Quality score:
    Düşük CTR expected → ad gösterilmez (kalite filtreleme).
    Kullanıcı bu tür reklamı daha önce gizlediyse → score düşer.

  Budget kontrolü:
    Günlük / aylık bütçe → harcandı → kampanya duraklar.
    Pacing: bütçeyi gün boyu yay, sabah bitirme.

  Feed'e entegrasyon:
    Reklam ID'leri de ZADD feed:{userId} ile cache'e eklenir.
    Score: yeterince yüksek olmalı (kullanıcı deneyimi koruma).
    Reklam / organik oranı: max %10 (her 10 postta 1).
```

---

## Impression Tracking ve Feedback Loop

```
Kullanıcı feed'i scroll ederken:
  Görülen post → "impression" → log (Kafka → ClickHouse).
  3sn+ görülen post → "view" (scroll ile geçip gitmedi).
  Tıklama → "click".
  Like, Comment, Share → engagement.
  "Gizle" → "hide" (güçlü negatif sinyal).
  "Şikayet et" → "report" (moderasyon kuyruğuna).

Client-side tracking:
  IntersectionObserver API: post %50+ viewport'ta → impression başladı.
  Süre ölçümü: 3sn geçti → "view" event gönder.
  Batch send: her 30sn veya 10 event → POST /events (gürültüyü azalt).

Training data pipeline:
  Kafka → Spark Streaming → feature store.
  Etiket: like=1, hide=-1, view=0.1, click=0.5.
  Günlük: dün'ün etkileşim logları → yeni model eğitimi.
  A/B test: yeni model → %5 kullanıcı → CTR/engagement karşılaştır.

Bias sorunları:
  Position bias: üstteki post daha çok tıklanır (sıralama avantajı değil).
    Düzeltme: Inverse Propensity Scoring (IPS) → pozisyon etkisini çıkar.
  
  Engagement bias: "like" kolay, "hide" zor → gerçek memnuniyet değil.
    Düzeltme: survey (random kullanıcı) → "Bu post iyi miydi?" → ground truth.

Cold Start:
  Yeni kullanıcı: etkileşim geçmişi yok → affinity skor yok.
    Çözüm: onboarding → ilgi alanı seçimi → warm start.
    Başlangıç: popüler postlar (viral + yüksek engagement) göster.
    İlk 7 gün: hızlı öğrenme (her etkileşim daha ağırlıklı).

  Yeni post: sıfır engagement → ML skoru belirsiz.
    Çözüm: yeni post boost (ilk 30 dk: tüm takipçilere göster).
    Exploration: Thompson Sampling → "test et, öğren, karar ver."
    Velocity izle: ilk 1 saatte hızla like alıyorsa → viral pool'a al.
```

---

## Gerçek Zamanlı Feed Güncellemesi

```
Kullanıcı feed'e bakıyor → arkadaşı yeni post attı → bildir.

WebSocket mimarisi:
  Client → WebSocket bağlantısı (sticky session → LB).
  Yeni post → Fan-out Svc → Kafka → Notification Svc.
  Notification Svc → Redis pub/sub: PUBLISH ws:user-123 {type:"new_post", count:3}
  WebSocket Svc → subscribe → push → client.

Push stratejisi (gürültü azaltma):
  Her post anında push → çok gürültülü, CPU drain.
  Aggregate: "3 yeni post" → 60sn biriktir → tek bildirim.
  Kullanıcı aktifse (scroll yapıyor): "Yeni postlar ↑" banner göster.
  Kullanıcı uygulamayı kapattıysa: mobil push notification (FCM/APNs).

Long Polling (fallback):
  WebSocket desteği yok → her 30sn GET /feed/new-count.
  Server: son kontrol zamanından sonra yeni post var mı? → count.

Presence service:
  Kullanıcı aktif mi? → WebSocket bağlantısı var mı?
  Aktif → anlık push.
  Pasif → notification biriktir → geri döndüğünde "X yeni post" göster.
```

---

## Cursor-Based Pagination (Infinite Scroll)

```
Problem: offset-based pagination → veri değişince post kaymalar.
  GET /feed?page=2 → arada yeni post eklendi → aynı post iki kez görünür.

Cursor-based çözüm:
  İlk istek: GET /feed → {posts: [...], next_cursor: "eyJwb3N0SWQiOiIxMjMiLCJzY29yZSI6MC43OH0="}
  Sonraki: GET /feed?cursor=eyJ... → cursor'dan itibaren devam et.

Cursor içeriği (base64 encoded JSON):
  {
    "last_post_id": "123e4567-e89b-12d3-a456-426614174000",
    "last_score": 0.78,
    "session_id": "session-abc",
    "generated_at": 1705312800
  }

Redis implementasyonu:
  ZADD feed:{userId} {score} {postId}
  Cursor: son görülen postun score'u → ZREVRANGEBYSCORE ile devam.
  
  # Cursor'dan itibaren sonraki 20:
  ZREVRANGEBYSCORE feed:{userId} {last_score} -inf LIMIT 0 21
  # 21 al, 21. = next_cursor için kullan (slice: ilk 20'yi göster).

Cache miss (cursor geçerliliği süresi dolmuş):
  72 saat sonra feed cache expire → cursor geçersiz.
  Çözüm: cursor'daki generated_at → 72 saat geçmişse → fresh feed üret.
  Kullanıcıya: "Feed yenilendi" bildirimi.

Session snapshot:
  Feed açıldığında: o anki top 200 post ID → session cache'e yaz.
  Scroll: session cache'ten devam → yeni post eklense bile önceki sıra korunur.
  Yenile: yeni session → güncel feed.
```

---

## A/B Test ve Model Deployment

```
Yeni ranking modeli test etme:

Traffic split:
  %90: mevcut model (control).
  %5: yeni model (treatment-A).
  %5: yeni model v2 (treatment-B).
  
  Sticky assignment: kullanıcı → her zaman aynı grup (deneyim tutarlılığı).
  Grup ataması: hash(userId) % 100 → hangi bucket?

Metrikler (7 günlük test):
  Primary: engagement rate (like+comment+share / impression).
  Secondary: session length, retention (7. gün dönüş).
  Guardrail: hide rate artmamalı, report rate artmamalı.
  Revenue: reklam tıklama oranı (CPM).

Statistical significance:
  p-value < 0.05 ve pratik anlamlılık (>2% relative improvement).
  Minimum detectable effect: %1 → N kullanıcı (power analysis).
  Erken durdurma: guardrail metric kötüleşiyorsa → rollback.

Canary deployment:
  Yeni model → önce %1 trafik → metric dashboard izle → sorun yok → %5 → %10 → %100.
  Model versioning: model-v42, model-v43 → çabuk rollback.

Shadow mode:
  Yeni model: gerçek trafik ile paralel hesapla (sonuç gösterme).
  Loglara yaz → offline karşılaştır → güvenli test.
```

---

## Veri Modeli

```sql
-- Post (Cassandra — yüksek write throughput)
CREATE TABLE posts (
    post_id     TIMEUUID PRIMARY KEY,
    author_id   UUID,
    content     TEXT,
    media_urls  LIST<TEXT>,
    post_type   TEXT,             -- TEXT, IMAGE, VIDEO, SHARE, LINK
    visibility  TEXT DEFAULT 'FRIENDS',  -- PUBLIC, FRIENDS, CUSTOM
    like_count  COUNTER,
    comment_count COUNTER,
    share_count COUNTER,
    view_count  COUNTER,
    engagement_rate FLOAT,        -- batch job ile güncellenir
    nsfw_score  FLOAT,            -- moderation ML skoru
    created_at  TIMESTAMP
);

-- Author'a göre son postlar (fan-out için hızlı okuma)
CREATE TABLE posts_by_author (
    author_id UUID,
    post_id   TIMEUUID,
    PRIMARY KEY ((author_id), post_id)
) WITH CLUSTERING ORDER BY (post_id DESC)
  AND default_time_to_live = 604800;  -- 7 gün TTL

-- Friend graph (PostgreSQL veya Neo4j)
CREATE TABLE friendships (
    user_id_1   UUID,
    user_id_2   UUID,             -- her zaman user_id_1 < user_id_2
    status      VARCHAR(20),      -- PENDING, ACCEPTED, BLOCKED
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id_1, user_id_2)
);

-- Affinity scores (DynamoDB veya Redis Hash)
CREATE TABLE affinity_scores (
    viewer_id   UUID,
    creator_id  UUID,
    score       DECIMAL(5,4),     -- 0.0000 - 1.0000
    last_updated TIMESTAMPTZ,
    PRIMARY KEY (viewer_id, creator_id)
);

-- User preferences (feed kişiselleştirme)
CREATE TABLE user_feed_preferences (
    user_id      UUID PRIMARY KEY,
    hide_video   BOOLEAN DEFAULT FALSE,
    hide_reposts BOOLEAN DEFAULT FALSE,
    topics_muted TEXT[],           -- ["politics", "sports"]
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Impressions (ClickHouse — append-only, analitik)
CREATE TABLE feed_impressions (
    user_id     UUID,
    post_id     UUID,
    session_id  UUID,
    position    SMALLINT,          -- feed'deki sıra (1-based)
    impressed_at DATETIME,
    view_duration_ms INT,          -- 0 = scroll, >3000 = long view
    action      VARCHAR(20)        -- NONE, LIKE, COMMENT, SHARE, HIDE, REPORT
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(impressed_at)
ORDER BY (user_id, impressed_at);

-- Feed cache (Redis — Sorted Set)
-- ZADD feed:{userId} {ml_score} {postId}
-- EXPIRE feed:{userId} 259200  (72 saat)
-- ZREVRANGE feed:{userId} 0 199 → top 200 post

-- Session feed snapshot (anlık görüntü)
-- SETEX feed_session:{sessionId} 1800 {json_of_post_ids}
```

---

## Olası Sorunlar ve Çözümleri

### 1. Celebrity Fan-Out Patlaması — Feed Günlerce Yavaş

```
Sorun:
  Kim Kardashian (250M follower) yeni post attı.
  Fan-out Service: 250M Redis ZADD → dakikalar sürer.
  Bu sürede: takipçiler eski feed görüyor, yeni post yok.
  Spike: 250M × 1 ZADD = 250M Redis operation → Redis cluster çöktü.

Neden kritik:
  Dünya genelinde 100 celebrity günde ortalama 10 post.
  = 1,000 celebrity post × 100M average follower = 100B fan-out/gün.
  Normal: 11,600 post/sn × 300 = 3.5M/sn.
  Celebrity spike: tek post → saniyede 10M+ Redis write.

Çözüm:
  a) Celebrity eşiği:
     > 5,000 follower → "celebrity" flag → fan-out on READ.
     Post DB'ye yaz, Redis cache'e YAZMA.
     Eşik dinamik: trafik yüküne göre otomatik ayarla.

  b) Read-time merge:
     Feed isteği: kendi cache (normal arkadaşlar) + celebrity son postları.
     Celebrity postları: Cassandra'dan çek, merge et, re-rank et.
     Cache: "celebrity_posts:{userId}" → 5 dk TTL → çok sayıda kullanıcı → birlikte çeker.

  c) Tiered fan-out:
     Fan-out: online kullanıcılara (son 24 saatte aktif) → anında yaz.
     Offline kullanıcılar: açınca pull yap → "24 yeni post" göster.
     Azaltma: 250M → %10 aktif = 25M write → 10x daha az.

  d) Rate limit fan-out:
     Celebrity post → fan-out queue → 10,000/sn hızında işle (cluster korunur).
     En erken 25 dakikada tamamlanır → kabul edilebilir.
```

---

### 2. Stale Feed — Kullanıcı Çok Eski Post Görüyor

```
Sorun:
  Feed cache TTL: 72 saat.
  Kullanıcı 3 günde bir açıyor → cache expire → fresh hesaplama.
  Ama: kullanıcı günlük açıyor, cache var → yeni postlar fan-out edilmedi.
  
  Senaryo: arkadaş 100 post attı, hepsi fan-out edildi ama:
    a. Redis ZADD: yeni score eskinin altında → sıralamada kayıp.
    b. Arkadaşı unfriend oldu → cache'teki eski postlar hâlâ var.
    c. Post silindi → cache'te postId var, DB'de yok → boş render.

Çözüm:
  a) Tombstone postlar:
     Post silindi → Kafka "post-deleted" event → feed cache consumer.
     ZREM feed:* {postId} → tüm cache'lerden sil.
     Pratik: sadece aktif kullanıcıları güncelle, pasifler pull'da temizlenir.

  b) Soft delete + client filter:
     Post silindi → deleted_at set.
     Feed client: post_id fetch → 404 → bu postu gösterme.
     Daha basit: 404'ler logla, düzeltme async.

  c) Unfriend event:
     Unfriend → Kafka → fan-out svc → ZREM feed:{userId} (eski arkadaşın postları).
     Kaldırma pahalı (hangi postlar onundu?) → alternatif:
     Feed render: author_id → friendship table → hâlâ arkadaş mı? → değilse atla.
     Tradeoff: her render'da friendship check → DB yükü.

  d) Cache version:
     feed_version:{userId} counter → her unfriend/block → +1.
     Cache'teki version ≠ current version → cache geçersiz → yenile.
```

---

### 3. Filter Bubble / Echo Chamber

```
Sorun:
  Ranking: engagement maksimize → kullanıcı beğendiği şeyi daha çok gördü.
  → Sadece kendi görüşüne uyan içerik → "herkesle hem fikir görünüyor."
  → Aşı karşıtı → giderek daha fazla aşı karşıtı içerik → radicalization.
  → Seçim döneminde: yanıltıcı siyasi içerik viral → dış müdahale riski.

Çözüm:
  a) Diversity score:
     Feed'deki topic dağılımına bak: 90% aynı topic → bonus ver farklı topic'e.
     Maximal Marginal Relevance: relevance_score - alpha × max_similarity_to_seen.
     alpha: ne kadar diversity zorla → product değeri kararı.

  b) Authoritative content boost:
     Güvenilir kaynak (CDC, WHO, major newspaper) → sağlık konusunda boost.
     Misinformation etiketleme: 3rd party fact-checker → false → uyarı göster.

  c) "Explore" section:
     Ana feed: %80 kişiselleştirilmiş.
     Explore section: %20 discovery → kullanıcının normal çemberinin dışı.
     Opt-out: kullanıcı kapatabilir.

  d) Metrik çeşitliliği:
     Sadece engagement optimize etme → satisfaction + diversity metric ekle.
     Kullanıcı anketi (random sample): "Bu içerikten memnun musun?"
     Memnuniyet × diversity → joint optimization.
```

---

### 4. Model Training Bias — Engagement ≠ Kullanıcı Memnuniyeti

```
Sorun:
  Model: engagement (like, comment, share) maximize ediyor.
  Ama: öfke içeriği → çok comment → yüksek engagement → üste çıkar.
  Sonuç: sinir bozucu, aşırılıkçı içerik → feed'e hâkim.
  
  Başka sorun: "Clickbait" başlık → yüksek CTR ama kullanıcı hayal kırıklığı.
  Model: CTR yüksek → iyi sanıyor, kullanıcı mutsuz.

Çözüm:
  a) Dwell time kullan:
     Tıkladıktan sonra ne kadar kaldı? → 2sn sonra geri dönüş → kötü.
     "Dwell time > 30sn" → pozitif sinyal, "immediate bounce" → negatif.

  b) Negative reaction takibi:
     "Gizle", "Rahatsız edici bul", "Takipten çık" → güçlü negatif label.
     Bu sinyalleri model'e agresif gir (yüksek negatif weight).

  c) User satisfaction survey:
     Her hafta: random 1,000 kullanıcı → "Bugünkü feedden memnun musun?"
     Survey sonuçlarıyla korelasyon → hangi içerik tipi kötü hissettiriyor.
     Survey labelled veri → model training'e ekle.

  d) Constrained optimization:
     Sadece engagement değil: engagement SUBJECT TO hate_speech < X, misinformation < Y.
     Constraint satisfaction → engagement içerik kalite sınırı içinde maksimize et.
```

---

### 5. Feed Kalitesi Sorunu — Çok Fazla Reklam

```
Sorun:
  Gelir hedefi artınca → reklam yoğunluğu %5'ten %15'e çıktı.
  Kullanıcı: "Feed artık reklam doldu, organik içerik yok."
  DAU düşmeye başladı → gelir de uzun vadede düşecek.
  
  Aynı zamanda: reklam kalitesi düşük (irrelevant, spam) → kullanıcı daha da rahatsız.

Çözüm:
  a) Ad/Organic oranı gardrail:
     Max %10 ad oranı → A/B test → eşik bulma.
     Organik engagement düşüyorsa → reklam azalt.

  b) Ad kalite skoru:
     Kullanıcı "Bu reklamı gizle" dedi → bu reklamcının global score düşer.
     Düşük CTR reklam → gösterilmez (sadece para ödüyor diye değil).
     Relevance: kullanıcının ilgi alanlarıyla eşleştir → irrelevant reklam = spam gibi.

  c) Native ad experience:
     Reklam organik içerik gibi görünsün (format, font, layout).
     "Sponsorlu" etiketi küçük ama görünür (şeffaflık).
     Hikaye formatı: organik post deneyimiyle benzer.

  d) Frequency capping:
     Aynı reklamı 3'ten fazla gösterme (7 günde).
     Kullanıcı "Gizle" dedi → bir daha gösterme.
     Reklamcı: reach metric → "kaç benzersiz kullanıcıya ulaştım."
```

---

### 6. Cold Start — Yeni Kullanıcı Boş Feed

```
Sorun:
  Yeni kullanıcı: arkadaşı yok, etkileşim geçmişi yok.
  Feed: boş veya alakasız içerik → kötü ilk deneyim → churn riski.
  İlk 7 gün: %40 kullanıcı ayrılıyor (industry benchmark).

Çözüm:
  a) Onboarding interest selection:
     Kayıtta: "Seni en çok ne ilgilendiriyor?" → 5+ kategori seç.
     Anında: seçilen kategorilerden popüler public postlar → feed dolu.

  b) Suggested connections:
     Email: kontak listesinden zaten üye olanları bul → hemen bağlantı öner.
     Telefon: numara hash'i → eşleştir → öneri.
     Facebook: 10 arkadaş ekle → feed anlamlı hale gelir.

  c) Trending & Popular:
     Ülke / dil bazında trending postlar → herkes için ilgili.
     "Bölgenden popüler" → lokal içerik.
     Viral içerik: çok paylaşılan → yeni kullanıcıya da göster.

  d) Hızlı öğrenme modu:
     İlk 7 gün: her etkileşim çok daha ağırlıklı (exploration priority).
     "Bu konuyu gizle" → anında feed'den kaldır.
     Explicit feedback: "Daha fazla göster / Daha az göster" butonu → prominent.

  e) Yeni post cold start:
     Yeni post: 0 like → ML skoru düşük → kimseye gösterilmez → kimse like etmez.
     Çözüm: her yeni post → yazarın aktif takipçilerine %100 göster (kesin).
     Velocity: ilk 1 saatte like velocity → yüksekse → viral pool'a ekle.
```

---

### 7. Ranking Servisi Yavaşladı — P99 > 200ms

```
Sorun:
  ML ranking servisi: normalde P99 40ms.
  Model güncellemesi + feature set genişletildi → P99 450ms.
  Feed SLA: 200ms → ihlal → 29,000 QPS × %15 = 4,350 kullanıcı/sn bozuk deneyim.

Neden:
  Feature hesaplama: affinity_score → her candidate için DynamoDB lookup.
  1,000 candidate × 1 DynamoDB read = 1,000 seri read → 200ms.

Çözüm:
  a) Feature pre-computation:
     Affinity score: saatte bir batch hesapla → Redis'e yaz.
     Ranking sırasında: Redis'ten O(1) lookup (DynamoDB seri değil).
     Tradeoff: 1 saatlik gecikme (affinity güncelleme).

  b) Batch inference:
     1,000 candidateı tek seferde → model inference batch API.
     PyTorch Serving: batch=1,000 → 1 GPU forward pass → tümü birden.
     1,000 seri inference → 1 batch inference → 10-50x hızlanma.

  c) Model simplification:
     Feature sayısını azalt: 50 feature → 20 feature (PCA, feature importance).
     2-layer NN → LightGBM → daha hızlı inference.
     Accuracy tradeoff: %0.5 engagement düşüşü → 5x hız kazanımı → kabul.

  d) Tiered ranking:
     Tüm 1,000 candidate → cheap model (LightGBM, 5ms) → top 100 seç.
     Top 100 → expensive model (NN, 40ms) → final sıralama.
     Net: 1,000 × LightGBM + 100 × NN = çok daha hızlı.

  e) Caching ML scores:
     Aynı post → birden fazla kullanıcı için score → base score cache.
     Cache key: postId → post_base_score.
     Kişiselleştirme: base_score × user_affinity × user_preference_multiplier.
     Tamamen farklı skor değil ama hesaplama paylaşılıyor.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Fan-out | On write (tümü) | Hybrid (normal→write, celebrity→read) | Tek strateji: celebrity spike → sistem çöker |
| Sıralama | Chronological | ML ranking | Kronolojik: 500 arkadaşın her postu = kaos |
| Feed cache | Her request üret | Pre-computed Redis ZADD | Her request: 500 arkadaş × DB = çok yavaş |
| Realtime update | Polling (30sn) | WebSocket + aggregated push | Polling: DB yükü; WS: düşük latency |
| Diversity | Sadece yüksek score | Score + MMR diversity | Tekrar: filter bubble; MMR: çeşitlilik dengesi |
| ML modeli | Tek büyük model | Two-tower + LightGBM reranker | Tek: yavaş; İki aşama: hız + kalite dengesi |
| Pagination | Offset-based | Cursor-based | Offset: post kayması; Cursor: tutarlı scroll |
| Celebrity eşiği | Sabit (5,000) | Dinamik (trafik yüküne göre) | Sabit: spike hâlâ olabilir; Dinamik: uyarlanabilir |
| Engagement label | Sadece like/comment | Like + dwell time + satisfaction survey | Sadece engagement: clickbait optimize eder |
| Ad oranı | Gelir odaklı (%15) | Guardrail ile sınırlı (%10) | %15: DAU düşüşü → uzun vadede daha az gelir |
| Cold start | Boş feed | Trending + interest selection | Boş: ilk hafta churn %40; Trending: anlık değer |
