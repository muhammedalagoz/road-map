# 08t — System Design: News Feed Sıralama (Facebook / LinkedIn)

## Gereksinimler

```
Functional:
  ✓ Kullanıcının bağlantılarından gelen içerikleri göster
  ✓ Puanlama ve sıralama (ML tabanlı)
  ✓ İçerik türleri: post, fotoğraf, video, paylaşım, link
  ✓ Gerçek zamanlı güncelleme (yeni post → feed'de görünsün)
  ✓ Sonsuz scroll (pagination)
  ✗ Stories, Reels, Gruplar (out of scope)

Non-Functional:
  DAU: 500M
  Feed açılma: 500M × 5/gün = 2.5B/gün
  Feed QPS: 2.5B / 86400 ≈ 29,000/sn
  Post QPS: 500M × 2/gün / 86400 ≈ 11,600 write/sn
  Feed yükleme: < 200ms
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
```

---

## High-Level Tasarım

```
           Client
              │ Feed isteği
              ▼
      ┌──────────────┐
      │  Feed Service │
      └──────┬───────┘
             │
    ┌─────────────────────────┐
    │    Feed Generation      │
    │  1. Candidate Selection │  ← arkadaşların son postları
    │  2. Filtering           │  ← blocked, hidden, seen
    │  3. Ranking (ML)        │  ← engagement prediction
    │  4. Merging             │  ← ads, recommended posts
    └──────────┬──────────────┘
               │
    ┌──────────┼──────────────┐
    ▼          ▼              ▼
[Post DB]  [User Graph]  [ML Ranking
(Cassandra) (Neo4j/Redis)  Service]
```

---

## Feed Oluşturma Pipeline

### Aşama 1: Candidate Selection

```
Kullanıcının 500 arkadaşından son 7 günün postları:

Naive:
  500 arkadaş × 10 post/gün × 7 gün = 35,000 post candidate
  Her biri için ML score hesapla → ağır

Optimizasyon:
  a. Son 48 saat ile sınırla (çok eski içerik irrelevant)
  b. Arkadaş yakınlık skoru (close friend > acquaintance)
     Yakınlık: son 30 günde kaç kez etkileşim?
     Düşük yakınlık → post candidate havuzuna dahil etme
  c. Son 1000 candidate → ranking'e gir (35,000 yerine)
```

### Aşama 2: Filtering

```
Kullanıcının işlevi:
  - Blocked user → postlar görünmez
  - Unfollowed (arkadaş ama feed'den gizle)
  - Daha önce görmüşüm → spam olur
  - İçerik türü tercih yok (video'yu gizle gibi)

seen_posts:
  Kullanıcı X, Post Y gördü → "seen" işaretle
  Redis BitSet: seen:{userId} → bit[postId] = 1
  Yeni feed → seen bit 0 olanları getir
```

### Aşama 3: ML Ranking

```
Her candidate post için engagement score:
  P(like) × like_weight
  + P(comment) × comment_weight
  + P(share) × share_weight
  + P(click) × click_weight
  + P(hide) × (-hide_weight)  ← negatif sinyal

Feature vector (her post için):
  - post_age (ne kadar yeni?)
  - creator_affinity_score (yazarla etkileşim geçmişi)
  - content_type (video, image, text — kullanıcı tercihi)
  - post_engagement (kaç like/comment aldı)
  - creator_following_count (popüler mi?)
  - time_of_day (sabah ne izleniyor vs akşam)
  - device_type (mobile video vs desktop)

Model: Gradient Boosting (LightGBM) veya Neural Network
  Train: Geçmiş etkileşim logları (offline)
  Update: Günlük model refresh

Skor → sort → top 200 → mix ile birleştir
```

### Aşama 4: Mixing & Diversity

```
Top 200 post → mix kuralları:

Aynı kişiden max 3 ardışık post gösterme
Her 10 posttan en az 2 farklı içerik türü (video, image, text)
Her 15 postta 1 reklam (ad network)
Her 20 postta 1 "önerilen" (arkadaş değil, discover içerik)

Son 20 post:
  [post1, post2, ad1, post3, post4, post5, video1, ...]

Diversity for filter bubbles:
  Sadece aynı görüşlü içerik gösterme → "çeşitlilik skoru" ekle
```

---

## Fan-Out Stratejisi

### Pull Model (Lazy Generation)

```
Feed isteği geldi → Arkadaşların postlarını çek → rank et → göster

Avantaj:
  ✓ Write basit (sadece Post DB'ye kaydet)
  ✓ Arkadaş sayısına bağımsız

Dezavantaj:
  ✗ Feed oluşturma her istekte → yavaş (500 arkadaş × DB query)
  ✗ Yüksek QPS'de DB yükü

Facebook yaklaşımı:
  Pull + Pre-computation hybrid
```

### Pre-Computed Feed Cache

```
Yeni post yazıldı → Fan-out Service:
  Yazarın arkadaşlarına (500 kişi) → feed_cache güncelle

Redis sorted set (score = ranking skoru):
  ZADD feed:{userId} {score} {postId}
  ZREVRANGE feed:{userId} 0 19  → top 20 post ID

Feed okuma:
  Redis'ten top 20 postId → Post DB'den batch fetch → Render

Sorun:
  500 arkadaş × her post = 500 Redis write → yönetilebilir
  500M friend olan celebrity → 500M Redis write → imkansız!

Hybrid (Facebook'un çözümü):
  Normal kullanıcı (< 1000 arkadaş): Fan-out on write
  Influencer / celebrity (> 5000): Fan-out on read
  Feed okurken: kendi cache + celebrity'ların son postları (pull) → merge
```

---

## Gerçek Zamanlı Feed Güncellemesi

```
Kullanıcı feed'e bakıyor → arkadaşı yeni post attı → feed'de görünsün

WebSocket / Long Polling:
  Client → WebSocket bağlantısı
  Yeni post → Fan-out → Redis pub/sub: PUBLISH feed:user-123 {postId}
  Feed Service → subscribe → WebSocket push → "Yeni post var" bildirimi
  Client: "Yenile" butonu göster veya otomatik refresh

Push sıklığı:
  Anlık her post → çok gürültülü
  Biriktir: "3 yeni post" → kullanıcı kaydırırken göster
```

---

## Veri Modeli

```sql
-- Post (Cassandra — yüksek write)
CREATE TABLE posts (
    post_id     TIMEUUID PRIMARY KEY,
    author_id   UUID,
    content     TEXT,
    media_urls  LIST<TEXT>,
    post_type   TEXT,        -- TEXT, IMAGE, VIDEO, SHARE, LINK
    like_count  COUNTER,
    share_count COUNTER,
    created_at  TIMESTAMP
);

-- Author'a göre son postlar (timeline query için)
CREATE TABLE posts_by_author (
    author_id UUID,
    post_id   TIMEUUID,
    PRIMARY KEY ((author_id), post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- Feed cache (Redis)
-- ZSET: feed:{userId} → (score=rankScore, member=postId)
-- Score: ML rank skoru (her feed refresh'te güncellenir)
```

---

## Ranking Sinyalleri Detayı

```
Affinity Score (Yakınlık):
  Son 30 günde:
    Comment: +10 puan
    Like:    +3 puan
    View:    +1 puan
    Share:   +7 puan
  Toplam / max → 0-1 normalize
  → "Bu kişiyle çok etkileşime girdim" → postları daha üste

Time Decay:
  Yeni post → yüksek skor
  rank_score = base_score × e^(-λ × age_in_hours)
  λ = 0.1 → 7 saat'te %50 skor kaybı

Virality Boost:
  Post çok paylaşılıyorsa → virality score artar
  Arkadaşlar beğendi → sosyal proof → üste çıkar

Content Type Preference:
  Kullanıcı son 30 günde video'ya daha fazla tıkladıysa
  → video content type multiplier artır
  Personalized content mix
```

---

## Trade-off Özeti

| Karar | Seçenek | Facebook Yaklaşımı |
|-------|---------|-------------------|
| Fan-out | On write | Hybrid (normal→write, celebrity→read) |
| Sıralama | Chronological | ML ranking |
| Feed cache | Her request generate | Pre-computed + merge |
| Realtime | Polling | WebSocket + aggregated push |
| Diversity | En yüksek score | Score + diversity rules |
