# 08g — System Design: Video Streaming (YouTube)

## Gereksinimler

```
Functional:
  ✓ Video yükleme
  ✓ Video izleme (akış)
  ✓ Farklı kalite seçeneği (360p, 720p, 1080p, 4K)
  ✓ Arama
  ✗ Canlı yayın, yorumlar, analitik (out of scope)

Non-Functional:
  DAU: 5M
  Upload: 500 saat video/dakika
  İzleme: 5M × 5 video/gün = 25M izleme/gün
  Latency: video başlama < 2s
  Availability: 99.99%
  Bandwidth: en büyük maliyet kalemi
```

---

## Capacity Estimation

```
Upload:
  500 saat/dakika = 500 × 60 = 30,000 saniye video/dakika
  Ortalama video: 300 MB (10 dakika, 1080p)
  Günlük upload: 500h × 60min × (300MB/10min) = 900,000 MB = 900 TB/gün

Encoding (her video → 5 kalite):
  360p: 50 MB, 480p: 100 MB, 720p: 300 MB, 1080p: 600 MB, 4K: 2 GB
  Her yüklenen video için: ~3 GB işleme çıktısı

İzleme bandwidth:
  25M izleme/gün, ortalama 10 dk, 1080p ≈ 600 MB/video
  25M × 600 MB = 15 PB/gün çıkış bandwidth → CDN olmadan imkansız

CDN maliyet:
  ~$0.01/GB CDN çıkış → 15 PB × $0.01 = $150,000/gün (CDN optimizasyon şart)
```

---

## High-Level Tasarım

```
          Upload Path                      View Path
               │                               │
        ┌──────┴──────┐               ┌────────┴────────┐
        │  Upload API  │               │    Client App   │
        └──────┬──────┘               └────────┬────────┘
               │                               │
        ┌──────▼──────┐               ┌────────▼────────┐
        │  Object     │               │      CDN        │
        │  Storage    │──────────────►│  (CloudFront/   │
        │  (S3 raw)   │               │   Akamai)       │
        └──────┬──────┘               └────────┬────────┘
               │                               │ miss
        ┌──────▼──────┐               ┌────────▼────────┐
        │  Encoding   │               │   Video Storage  │
        │  Service    │──────────────►│   (S3 encoded)  │
        │  (FFmpeg)   │               └─────────────────┘
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │  Metadata   │
        │  Service    │
        │  (MySQL)    │
        └─────────────┘
```

---

## Video Yükleme Akışı

```
1. Kullanıcı → Upload API: "Video yüklemek istiyorum" (metadata)
   → Upload Service → S3 presigned URL üret
   → Client URL'yi alır

2. Client → S3 presigned URL → Ham videoyu direkt yükle
   (multipart upload: her 5 MB chunk ayrı upload, paralel)

3. S3 → Upload tamamlandı → SNS/SQS event
   → Encoding Queue tetiklenir

4. Encoding Service:
   a. Ham videoyu S3'ten çek
   b. FFmpeg ile paralel encoding:
      360p:  ffmpeg -i raw.mp4 -vf scale=-1:360  output_360p.mp4
      720p:  ffmpeg -i raw.mp4 -vf scale=-1:720  output_720p.mp4
      1080p: ffmpeg -i raw.mp4 -vf scale=-1:1080 output_1080p.mp4
   c. HLS segmentlere böl (10 saniyelik .ts dosyaları)
   d. Manifest dosyası oluştur (master.m3u8)
   e. Tüm parçaları S3'e yükle

5. Metadata Service: video durumunu "READY" olarak güncelle
```

---

## HLS (HTTP Live Streaming) — Nasıl Çalışır?

```
Video → Segmentler

master.m3u8 (manifest):
  #EXTM3U
  #EXT-X-VERSION:3
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=1280x720
  video_720p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1920x1080
  video_1080p/index.m3u8

video_720p/index.m3u8:
  #EXTM3U
  #EXT-X-TARGETDURATION:10
  #EXTINF:10.0,
  segment_000.ts    ← 10 saniyelik video chunk
  #EXTINF:10.0,
  segment_001.ts
  #EXTINF:10.0,
  segment_002.ts
  ...

Player:
  1. master.m3u8 indir → kalite listesini gör
  2. Ağ hızına göre kalite seç (Adaptive Bitrate)
  3. video_720p/index.m3u8 indir → segment listesi
  4. segment_000.ts, segment_001.ts, ... sırayla indir
  5. Buffer dolunca oynat
  6. Ağ yavaşladı → 360p'ye düş (adaptive bitrate switching)
```

---

## Adaptive Bitrate Streaming (ABR)

```
Client, ağ hızını sürekli ölçer ve kaliteyi dinamik ayarlar:

Buffer < 5s  → bir kalite aşağı (1080p → 720p)
Buffer > 20s → bir kalite yukarı (720p → 1080p)
Ağ hızı ölçüm: her segment ne kadar sürede indi?

DASH (Dynamic Adaptive Streaming over HTTP):
  HLS'e benzer, MPEG-DASH standardı
  YouTube ve Netflix kullanır

HLS vs DASH:
  HLS: Apple'ın standardı, iOS native destekler
  DASH: Açık standart, Android ve web
  Modern sistemler ikisini de üretir
```

---

## CDN Stratejisi

```
Push CDN:
  Origin → CDN'in tüm edge node'larına proaktif yükle
  ✓ İlk istek hızlı
  ✗ Depolama maliyeti yüksek (popüler olmayan içerik de var)

Pull CDN (YouTube/Netflix için önerilen):
  İlk istek: CDN miss → Origin'den çek → CDN'de cache'le → kullanıcıya ver
  Sonraki istekler: CDN hit → direkt ver
  ✓ Popüler içerik otomatik cache'lenir
  ✗ İlk istek biraz yavaş (cold cache)

CDN cache key:
  /video/{videoId}/{quality}/segment_{n}.ts
  segment bazlı cache → parçalar bağımsız cache'lenir

Coğrafi optimizasyon:
  Türkiye kullanıcısı → Frankfurt/İstanbul CDN node
  ABD kullanıcısı → New York/LA CDN node
  GeoDNS: kullanıcı → en yakın CDN PoP
```

---

## Video Storage Organizasyonu

```
S3 bucket yapısı:
  videos-raw/
    {videoId}/raw.mp4              ← orijinal upload

  videos-encoded/
    {videoId}/
      master.m3u8                  ← manifest
      360p/
        index.m3u8
        segment_000.ts
        segment_001.ts
        ...
      720p/
        index.m3u8
        segment_000.ts
        ...
      1080p/
        index.m3u8
        ...
      thumbnail_default.jpg        ← 3 farklı thumbnail
      thumbnail_1.jpg
      thumbnail_2.jpg

S3 Storage Class:
  Son 30 gün: S3 Standard (sık erişim)
  30-180 gün: S3 Intelligent-Tiering
  180 gün+: S3 Glacier (eski videolar)
```

---

## Encoding Pipeline (Dağıtık)

```
Ham video büyük olabilir (2+ GB) → encoding uzun sürer
Paralel encoding:

Approach 1: Tek encoder
  1 video → 1 encoding job → 1080p → 720p → 360p sıralı
  ✗ Yavaş, seri

Approach 2: Paralel encoding (önerilen)
  1 video → 3 encoding job (360p, 720p, 1080p aynı anda)
  Her job farklı encoding worker'da

  Video → Pre-Processor (split) → 60 segment
  60 segment × 3 kalite = 180 paralel encoding task
  180 worker → segment bazlı encode → S3'e yükle → manifest oluştur

Araçlar:
  AWS MediaConvert: managed video encoding
  FFmpeg: open-source (kendi EC2 instance'larında)
  Zencoder, Mux: SaaS video encoding
```

---

## Arama

```
Video metadata → Elasticsearch indexi:
  {videoId, title, description, tags, channelId, viewCount, uploadedAt}

Arama: "java tutorial"
  → Elasticsearch full-text search
  → Score: title > description > tags
  → Ranking: relevance × viewCount × freshness

Autocomplete:
  Redis: ZADD autocomplete "java" → "java tutorial", "java spring"
  veya Elasticsearch completion suggester
```

---

## Metadata DB Şeması

```sql
CREATE TABLE videos (
    video_id        UUID         PRIMARY KEY,
    uploader_id     UUID         NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(20),  -- UPLOADING, PROCESSING, READY, FAILED
    duration_secs   INT,
    view_count      BIGINT       DEFAULT 0,
    like_count      BIGINT       DEFAULT 0,
    thumbnail_url   VARCHAR(500),
    created_at      TIMESTAMP,
    INDEX idx_uploader (uploader_id),
    INDEX idx_created_at (created_at)
);

CREATE TABLE video_qualities (
    video_id    UUID,
    quality     VARCHAR(10),  -- 360p, 720p, 1080p, 4K
    s3_path     VARCHAR(500),
    bitrate_kbps INT,
    ready       BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (video_id, quality)
);
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Upload | Direkt API | S3 Presigned | Presigned (API yük azalır) |
| Streaming | Progressive MP4 | HLS/DASH | HLS/DASH (ABR, CDN uyumlu) |
| CDN | Push | Pull | Pull (cost efficient) |
| Encoding | Seri | Paralel segment | Paralel (hız) |
| Storage | Tek tier | Intelligent-Tiering | Tiering (maliyet) |
