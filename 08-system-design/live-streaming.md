# 08q — System Design: Canlı Yayın (Twitch / YouTube Live)

## Gereksinimler

```
Functional:
  ✓ Yayıncı video akışını iletir
  ✓ İzleyiciler düşük latencyyle izler
  ✓ Canlı chat
  ✓ İzleyici sayısı
  ✓ Yayın kaydı (replay)
  ✗ Ödeme, subscription (out of scope)

Non-Functional:
  100K eş zamanlı yayın
  1M eş zamanlı izleyici
  Latency: yayıncı → izleyici < 10 saniye (low latency streaming)
  Availability: 99.99%
```

---

## Capacity Estimation

```
Yayın kaliteleri per yayın:
  1080p: 6 Mbps, 720p: 3 Mbps, 480p: 1.5 Mbps, 360p: 0.8 Mbps

Ingest bandwidth (yayıncılar → sunucu):
  100K yayın × 6 Mbps (1080p) = 600 Gbps ingest

Egress bandwidth (izleyiciler → CDN):
  1M izleyici × ortalama 3 Mbps = 3 Tbps egress (CDN dağıtır)

Storage (kayıt):
  100K yayın × ortalama 2 saat × 6 Mbps = 540 TB/gün
  Sıkıştırma sonrası: ~200 TB/gün
```

---

## High-Level Tasarım

```
Streamer (OBS Studio)
    │ RTMP (Real-Time Messaging Protocol)
    ▼
┌────────────────────────────────┐
│      Ingest Server             │
│  (RTMP receiver, Nginx RTMP    │
│   veya AWS IVS)                │
└────────────────┬───────────────┘
                 │ raw video stream
                 ▼
┌────────────────────────────────┐
│   Transcoding Service          │
│   FFmpeg: 1080p→720p→480p→360p │
│   H.264/H.265 encoding         │
│   HLS segmentation (2s chunks) │
└────────────────┬───────────────┘
                 │
    ┌────────────┼────────────────┐
    ▼            ▼                ▼
  ┌──────┐   ┌──────┐        ┌────────┐
  │  S3  │   │ CDN  │        │Recording│
  │(HLS  │──►│(Edge)│        │ Storage │
  │ segs)│   │      │        │  (S3)  │
  └──────┘   └──────┘        └────────┘
                 │
             Viewers
```

---

## RTMP Ingest

```
OBS Studio → RTMP → Ingest Server:
  URL: rtmp://live.mystream.com/live/{streamKey}

Ingest Server (Nginx RTMP Module):
  nginx.conf:
    rtmp {
      server {
        listen 1935;
        application live {
          live on;
          record all;
          record_path /tmp/recordings;
          on_publish http://localhost:8080/auth/stream;  # stream key doğrula
          exec_push ffmpeg -i rtmp://localhost/$app/$name
            -c:v libx264 -b:v 6000k -s 1920x1080 -f flv rtmp://localhost/hls/$name_1080p
            -c:v libx264 -b:v 3000k -s 1280x720  -f flv rtmp://localhost/hls/$name_720p
            -c:v libx264 -b:v 1500k -s 854x480   -f flv rtmp://localhost/hls/$name_480p;
        }
        application hls {
          live on;
          hls on;
          hls_path /tmp/hls;
          hls_fragment 2s;        # 2 saniyelik segment
          hls_playlist_length 30s; # son 15 segment playlist'te
        }
      }
    }
```

---

## HLS ile Düşük Latency

```
Normal HLS:
  Segment: 10s → 3 segment buffer = 30s latency

Low Latency HLS (LL-HLS):
  Segment: 2s → 3 segment buffer = 6s latency
  Partial segments (0.5s parça) → 2-3s latency

  master.m3u8:
    #EXT-X-TARGETDURATION:2
    #EXT-X-PART-INF:PART-TARGET=0.5    ← 0.5s partial segment
    
    #EXT-X-PART:DURATION=0.5,URI="part0.ts"  ← player partial segment okur
    #EXT-X-PART:DURATION=0.5,URI="part1.ts"
    #EXT-X-PART:DURATION=0.5,URI="part2.ts"
    #EXT-X-PART:DURATION=0.5,URI="part3.ts"
    #EXTINF:2.0,
    seg0.ts                                   ← tam segment (4 partial = 1 full)

Alternatif: WebRTC (sub-second latency)
  Yayıncı → WebRTC → SFU (Selective Forwarding Unit) → izleyiciler
  ✓ 0.5-1s latency (gerçek zamanlı etkileşim için)
  ✗ Ölçekleme zor (her izleyici için ayrı bağlantı)
  Kullanım: Gaming, interactive shows (çok düşük latency gerekli)
```

---

## CDN ile Ölçekleme

```
3 Tbps egress → CDN olmadan imkansız

CDN Pull:
  İzleyici → CDN Edge → S3 (segment yoksa)
  Segment önbelleklenmiş → Edge direkt verir
  Canlı segment (2s) → kısa TTL → Edge'de 2s cache
  
  Avantaj:
    Aynı segmenti 1M izleyici isterse → S3'ten 1 kez çek → CDN dağıtır
    Origin'e 1 istek = 1M izleyiciye hizmet

Multi-CDN:
  CloudFront + Akamai + Fastly
  İzleyici → GeoDNS → en yakın CDN provider
  Failover: CDN çöküşünde alternatif CDN

WebSocket push for segments:
  CDN gerçek zamanlı bildirim alamaz
  Alternatif: Server-Sent Events veya WebSocket ile player'a
    "Yeni segment hazır: seg_123.ts" → Player direkt ister
  → Player polling'den kurtulur (segment ready push)
```

---

## Canlı Chat

```
Chat ölçeği:
  1M izleyici → 100 mesaj/sn (aktif chat)
  Peak (büyük event): 10,000 msg/sn

Architecture:
  WebSocket (chat sunucuları)
  Redis Pub/Sub: channel = stream:{streamId}:chat
  Chat sunucusu → Redis SUBSCRIBE
  Mesaj gelince → Redis PUBLISH → tüm chat sunucuları → WebSocket push

Rate Limiting (spam önleme):
  Kullanıcı başına: max 1 mesaj/2 saniye
  Moderasyon: banned words filter, AutoMod

Ölçekleme:
  1M eş zamanlı WebSocket → chat sunucuları (sticky session)
  Redis Cluster: pub/sub için yeterli
  Kafka (yedek): chat logları için kalıcı depolama
```

---

## Yayın Durumu ve Stream Metadata

```
Yayın başladı → Stream Service:
  INSERT INTO streams (streamId, userId, title, startedAt, status=LIVE)
  Redis: SET stream:{streamId}:status "LIVE"
  WebSocket push → takipçilere "Yayın başladı"

İzleyici sayısı:
  WebSocket bağlantısı → INCR stream:{streamId}:viewers
  Bağlantı kapandı → DECR stream:{streamId}:viewers
  Her 5 saniyede DB'ye yaz (çok sık write yapma)

  HyperLogLog (daha doğru unique viewer):
    PFADD stream:{streamId}:unique_viewers {userId}
    PFCOUNT stream:{streamId}:unique_viewers → tahmini unique sayı

Yayın bitti:
  UPDATE streams SET status=ENDED, endedAt=NOW()
  VOD (Video on Demand) olarak kaydet → S3'te kalıcı
```

---

## Transcode Pipeline

```
Yayıncı 1080p yayın → 4 kalite üretilmesi gerekiyor

Seçenek 1: Segment başına transcode
  Her 2s segment gelince → 4 kaliteyi encode et → CDN'e push
  Latency: +2-4s (encoding süresi)
  
Seçenek 2: Paralel encoding (GOP alignment ile)
  GOP (Group of Pictures): Keyframe başlangıcı
  Aynı GOP'u 4 encoder paralel işler
  → Tüm kaliteler senkron → adaptive bitrate switching sorunsuz

AWS MediaLive / Elemental:
  Managed transcode service
  Otomatik ölçekleme (izleyici artınca daha fazla encode kapasitesi)

GPU kullanımı:
  NVENC (NVIDIA) → CPU'ya göre 5-10x hızlı encoding
  Maliyet: GPU instance (p3, g4) daha pahalı ama gerekli

Adaptive encoding:
  Yayıncının upload hızı düşerse → daha düşük kaliteye düş
  Ingest server bandwidth ölç → encoder'a feedback
```

---

## Veri Modeli

```sql
CREATE TABLE streams (
    stream_id       UUID PRIMARY KEY,
    streamer_id     UUID NOT NULL,
    title           VARCHAR(500),
    category        VARCHAR(100),    -- Gaming, Music, IRL
    status          VARCHAR(20),     -- LIVE, ENDED, ERROR
    started_at      TIMESTAMP,
    ended_at        TIMESTAMP,
    peak_viewers    INT,
    total_views     BIGINT,
    recording_url   TEXT,            -- S3 path (ended sonrası)
    stream_key      VARCHAR(100) UNIQUE,  -- hashed (güvenlik)
    INDEX idx_status_started (status, started_at)
);

CREATE TABLE stream_quality_urls (
    stream_id   UUID,
    quality     VARCHAR(10),  -- 1080p, 720p, 480p, 360p
    hls_url     TEXT,
    PRIMARY KEY (stream_id, quality)
);
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Ingest | RTMP | WebRTC | RTMP (yayıncı tool'ları RTMP) |
| Streaming | HLS (10s) | LL-HLS (2s) | LL-HLS (kullanıcı deneyimi) |
| Ultra low latency | LL-HLS | WebRTC | WebRTC (interactive, küçük ölçek) |
| CDN | Push | Pull | Pull (canlı = sürekli değişen) |
| Chat | HTTP polling | WebSocket+Redis | WebSocket+Redis |
