# 08q — System Design: Canlı Yayın (Twitch / YouTube Live)

## Gereksinimler

```
Functional:
  ✓ Yayıncı video akışını iletir (RTMP/SRT)
  ✓ İzleyiciler düşük latency ile izler (< 10sn)
  ✓ Çoklu kalite (1080p, 720p, 480p, 360p) — Adaptive Bitrate
  ✓ Canlı chat (10,000 msg/sn peak)
  ✓ Eş zamanlı izleyici sayısı
  ✓ Yayın kaydı ve VOD (replay)
  ✓ Stream DVR (canlı yayını geri sarma, son 30 dk)
  ✓ Klip oluşturma (son 30 sn kaydet)
  ✓ Stream keşfi (öneri, kategori)
  ✓ Ultra low latency modu (< 1sn, interactive show için)
  ✗ Ödeme, subscription (out of scope)

Non-Functional:
  100K eş zamanlı yayın
  1M eş zamanlı izleyici
  Latency: yayıncı → izleyici < 10 sn (LL-HLS), < 1 sn (WebRTC modu)
  Availability: 99.99% (canlı yayın kesilmemeli)
  Transcoding: 1080p ingest → 4 kalite < 5 sn (segment süresi kadar)
  Chat: mesaj alındıktan < 500ms'de tüm izleyicilere ulaşmalı
```

---

## Capacity Estimation

```
Yayın kaliteleri per stream:
  1080p: 6 Mbps, 720p: 3 Mbps, 480p: 1.5 Mbps, 360p: 0.8 Mbps

Ingest bandwidth (yayıncılar → sunucu):
  100K yayın × 6 Mbps (1080p ingest) = 600 Gbps ingest
  100K ingest server: 600 Gbps / 10 Gbps NIC = 60 server min
  Her server 1,666 yayın: 1,666 × 6 Mbps = 10 Gbps → tam kapasite
  Güvenlik payı 2x: 120 ingest server

Egress bandwidth (CDN → izleyiciler):
  1M izleyici × 3 Mbps (720p ortalama) = 3 Tbps
  CDN olmadan: 3 Tbps / 10 Gbps NIC = 300 origin server → imkansız!
  CDN ile: origin'e 100K stream × 6 Mbps × 4 kalite = 2.4 Tbps origin pull
  CDN edge'de cache hit %95: origin → 2.4 × 0.05 = 120 Gbps → yönetilebilir

Transcoding:
  100K stream × 4 kalite encode = 400K encoding işlemi
  1 CPU core: 1 stream encoding (yazılım H.264)
  GPU (NVENC): 1 GPU → 16 stream encoding
  400K / 16 = 25,000 GPU → bulut (AWS Media Services / spot instance)

Storage (kayıt):
  100K yayın × 2 saat ort. × 6 Mbps × 3600 / 8 = 540 TB/gün raw
  4 kalite: 540 × (6+3+1.5+0.8)/6 ≈ 540 × 1.88 = 1,015 TB/gün
  S3 + sıkıştırma (H.265 %40 tasarruf): ~600 TB/gün

DVR (son 30 dk, tüm aktif yayınlar):
  100K yayın × 30 dk × 6 Mbps / 8 = 135 TB anlık DVR buffer

Chat:
  10,000 msg/sn peak × 200B = 2 MB/sn → trivial
  Chat sunucu: 1M WebSocket = 1M × 10 KB kernel buffer = 10 GB RAM
  20 chat server (50K conn/server)
```

---

## High-Level Mimari

```
Streamer (OBS / Streamlabs / Mobile)
    │  RTMP / SRT / WebRTC
    ▼
┌──────────────────────────────────────────┐
│          Ingest Layer (HA)               │
│  GeoDNS → en yakın ingest cluster        │
│  Primary ingest + backup ingest (hot)    │
│  Stream Key doğrulama → Auth Service     │
└────────────────┬─────────────────────────┘
                 │ raw stream (RTMP internal)
                 ▼
┌──────────────────────────────────────────┐
│        Transcoding Pipeline              │
│  FFmpeg / NVENC GPU                      │
│  ABR Ladder: 1080p→720p→480p→360p        │
│  GOP alignment (keyframe sync)           │
│  HLS / LL-HLS segmentation (2s)          │
└────────────────┬─────────────────────────┘
                 │
    ┌────────────┼───────────────────┐
    ▼            ▼                   ▼
 ┌──────┐    ┌──────┐          ┌──────────┐
 │Origin│    │  S3  │          │ DVR buf  │
 │ HTTP │    │(segs,│          │(S3 ring  │
 │server│    │ VOD) │          │ buffer)  │
 └──┬───┘    └──────┘          └──────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│              CDN Layer                   │
│  CloudFront + Akamai + Fastly (Multi-CDN)│
│  Edge cache: segment TTL = 2s            │
│  GeoDNS: izleyici → en yakın edge        │
└────────────────┬─────────────────────────┘
                 │ HLS / LL-HLS
                 ▼
             Viewers (hls.js, native player)

Yan servisler:
  Stream Service:     metadata, status, izleyici sayısı
  Chat Service:       WebSocket + Redis Pub/Sub + Kafka
  Analytics Service:  gerçek zamanlı metrikler (Flink + ClickHouse)
  Notification Svc:   "Yayın başladı" push (FCM/APNs)
  Discovery/Search:   Elasticsearch (kategori, isim, dil)
  Clip Service:       async klip oluşturma (DVR buffer'dan)
  Moderation Svc:     AutoMod, banned word, ML spam detection
```

---

## Ingest Layer (Yüksek Erişilebilirlik)

### RTMP vs SRT Karşılaştırması

```
RTMP (Real-Time Messaging Protocol):
  Port: 1935 (TCP).
  Yaygınlık: OBS, Streamlabs default → tüm yayıncılar biliyor.
  Dezavantaj: TCP → paket kaybında yeniden iletim → latency artışı.
  Jitter: kötü bağlantıda takılma, donma.

SRT (Secure Reliable Transport):
  UDP tabanlı → paket kaybında ARQ (selective retransmission).
  Kötü ağda (yayıncı ev interneti) RTMP'den çok daha iyi.
  Şifreleme: AES-128/256 built-in.
  Twitch, YouTube: SRT desteğini artırdı.
  Dezavantaj: OBS'te ek konfigürasyon gerekli.

WebRTC (ultra low latency):
  < 500ms latency.
  Browser native (ek yazılım yok).
  Ölçekleme: her izleyici için ayrı peer connection → SFU gerekli.
  Kullanım: interaktif show, Q&A, web-based streaming.
```

### Ingest HA (Yayın Kesilmemesi)

```
Problem: Ingest server çöktü → yayın kesildi → izleyiciler "Bağlantı hatası" gördü.

Primary + Backup Ingest:
  Yayıncı OBS → 2 URL yapılandırması:
    Primary:  rtmp://ingest-1.mystream.com/live/{key}
    Backup:   rtmp://ingest-2.mystream.com/live/{key}
  OBS: primary bağlantı kesildi → otomatik backup'a geç (RTMP reconnect).
  Her ikisinde de stream alınıyor (redundant ingest).
  Sadece primary kullanılıyor, backup pasif bekleniyor.

Failover akışı:
  Ingest-1 down → Health check (her 5sn) tespit eder.
  Stream Coordinator: "ingest-1'deki stream-abc artık ingest-2'den alınacak."
  Transcoding pipeline: kaynak değiştirir → izleyiciler fark etmez.
  Yayıncı: OBS backup'a geçmiş, yayın devam ediyor.

GeoDNS ile coğrafi dağıtım:
  Türkiye yayıncı → ingest-istanbul.mystream.com → İstanbul cluster.
  ABD yayıncı → ingest-us-east.mystream.com → Virginia cluster.
  Avantaj: yayıncı → ingest arası round trip azalır → latency düşer.
  Transcoding: ingest cluster yerel → segment S3'e → CDN'e global dağıtım.

Stream key doğrulama:
  RTMP on_publish hook:
    POST /auth/stream { streamKey, ip, timestamp }
    Auth Service: key geçerli mi? yayıncı ban'lı mı? → 200 / 403
    Ingest server: 403 → bağlantıyı kes.

  Stream key güvenliği:
    Hash: DB'de key'in SHA-256 hash'i saklanır (düz metin değil).
    Sızdırıldıysa: kullanıcı dashboard → "Yeni key oluştur."
    Unauthorized stream: IP analizi + rate limit.
    RTMPS (TLS): stream key transit şifreli.
```

---

## Transcoding Pipeline

### ABR Ladder (Adaptive Bitrate)

```
Gelen 1080p stream → 4 kalite paralel encode:

Kalite    Çözünürlük   Bitrate   FPS    Codec
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1080p     1920×1080    6 Mbps    60     H.264 / H.265
720p      1280×720     3 Mbps    60     H.264 / H.265
480p      854×480      1.5 Mbps  30     H.264
360p      640×360      0.8 Mbps  30     H.264

Neden H.265 de:
  H.265 (HEVC): aynı kalite için H.264'e göre %40-50 daha küçük.
  Bandwidth tasarrufu: 3 Mbps H.264 ≈ 1.8 Mbps H.265.
  Sorun: eski cihazlar H.265 decode edemeyebilir → fallback H.264.
  Strateji: modern cihaz → H.265, eski → H.264 (codec negotiation).

VP9 / AV1 (YouTube/Netflix):
  AV1: H.265'ten %20 daha verimli, royalty-free.
  Encoding: çok yavaş (canlı için H.264/H.265 daha pratik, AV1 sadece VOD).
```

### GOP Alignment (Kritik!)

```
GOP (Group of Pictures):
  I-frame (keyframe): bağımsız tam çerçeve.
  P-frame: önceki frame'e göre fark.
  B-frame: önceki + sonraki frame'e göre fark.
  GOP: I-P-B-B-P-B-B... (I'dan I'ya kadar).

Neden GOP alignment şart:
  720p encode: I-frame her 2 sn → keyframe.
  480p encode: I-frame 2.3 sn → farklı zamanda.
  ABR switch: player 720p'den 480p'ye geçiyor.
  Keyframe uyuşmuyorsa → video donması, siyah ekran.

  ZORUNLU: tüm kaliteler aynı GOP sınırlarına sahip olmalı.
  Çözüm: forced keyframe her tam saniyede:
    FFmpeg -force_key_frames "expr:gte(t,n_forced*2)"
    Her 2 sn: tüm encoder aynı anda keyframe üretir.

Paralel encoding:
  Tek FFmpeg process 4 kalite → sıralı → yavaş.
  4 FFmpeg process paralel (veya GPU NVENC) → aynı anda 4 kalite.
  GOP alignment ile → tüm kaliteler aynı anda tamamlanır → sync push.

NVENC (GPU encoding):
  Yazılım H.264: 1 CPU core → ~1 stream.
  NVENC (NVIDIA T4): 1 GPU → 16 stream × 4 kalite = 64 encode.
  Maliyet: GPU instance pahalı ama 64x kapasite → toplam maliyette avantajlı.
```

---

## HLS ve Düşük Latency

### Normal HLS vs LL-HLS

```
Normal HLS:
  Segment: 10s → player 3 segment buffer bekler → latency: 30+ sn.
  Yayıncı ve izleyici arasında 30 sn → spoiler riski (Twitter'da gol gördü).

Low Latency HLS (Apple, RFC 8216bis):
  Segment: 2s, partial segment: 0.5s.
  Player 3 partial segment buffer: 1.5s latency.
  Blocking playlist reload: player "henüz hazır olmayan segmenti" bekliyor.
    GET /playlist.m3u8?_HLS_msn=42&_HLS_part=3
    Server: segment 42, part 3 hazır olana kadar bekle (long poll).
    Hazır → hemen cevap ver → player anında indirir.
  Sonuç: 2-4s latency (Twitch LL-HLS hedefi).

LL-HLS playlist örneği:
  #EXTM3U
  #EXT-X-TARGETDURATION:2
  #EXT-X-VERSION:9
  #EXT-X-PART-INF:PART-TARGET=0.5
  #EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0

  #EXT-X-PART:DURATION=0.5,URI="seg42_part0.m4s"
  #EXT-X-PART:DURATION=0.5,URI="seg42_part1.m4s"
  #EXT-X-PART:DURATION=0.5,URI="seg42_part2.m4s"
  #EXT-X-PART:DURATION=0.5,URI="seg42_part3.m4s"
  #EXTINF:2.000,
  seg42.m4s                       ← tam segment (DVR ve clip için)

  #EXT-X-PRELOAD-HINT:TYPE=PART,URI="seg43_part0.m4s"  ← henüz yok, hazırlanıyor

Push notification (CDN'e):
  Transcoder: segment hazır → CDN'e PURGE/UPDATE bildir.
  CDN: eski segment cache'ini sil → yeni segment'i çek → izleyiciye ver.
  Alternatif: CDN push (segment hazır → CDN'e gönder, pull bekleme).
```

### WebRTC (Ultra Low Latency)

```
< 500ms latency → interaktif show, Q&A, gerçek zamanlı etkileşim.

Mimari: SFU (Selective Forwarding Unit):
  Yayıncı → SFU → her izleyiciye ayrı stream.
  SFU: video paketlerini decode etmez, sadece yönlendirir → CPU düşük.
  MCU (Multipoint Control Unit): paketleri birleştirir → CPU çok yüksek → tercih edilmez.

  Yayıncı: 1 WebRTC bağlantı → SFU.
  İzleyici: her biri SFU'ya 1 bağlantı.
  1,000 izleyici → 1,000 peer connection → SFU'da 1,000 outbound stream.

Ölçekleme zorluğu:
  1M izleyici → 1M WebRTC bağlantı → 1,000 SFU server gerekli.
  HLS: 1M izleyici → CDN alır, origin'e minimal yük.
  WebRTC: her bağlantı stateful → SFU horizontal ölçek zor.

Hybrid yaklaşım (Twitch):
  Ultra low latency (< 1sn): WebRTC modu → küçük, interaktif yayınlar.
  Normal yayın: LL-HLS → büyük kitleler.
  Player: bant genişliğine göre mod seçer, izleyici sayısına göre de.
  Eşik: 10,000 izleyici → WebRTC mod kapat → LL-HLS'e geç.

WHIP (WebRTC-HTTP Ingestion Protocol):
  Yayıncı tarafında WebRTC (tarayıcıdan yayın).
  OBS WebRTC plugin → WHIP → ingest server.
  Avantaj: RTMP yerine WebRTC → daha düşük latency ingest.
```

---

## CDN ile Ölçekleme

```
3 Tbps egress → kendi data center'ımızla imkansız.

CDN Pull Modeli:
  İzleyici → CDN Edge → cache var? → edge'den serve.
  Cache yok → origin HTTP server → segment çek → edge cache → izleyiciye ver.
  Aynı segment 1M izleyici isterse → origin'den 1 kez çekme.
  CDN, cache'i 1M'e dağıtır → origin'e 1 istek.

Segment cache TTL:
  Canlı segment (2s): TTL = 2s (çabuk eskir, yeni segment gelecek).
  Eski segment (VOD): TTL = 86400s (1 gün, değişmez).
  Playlist (.m3u8): TTL = 1s (çok sık değişir, playlist her 2s güncellenir).
  Partial segment: TTL = 5s (kısa, ama birden fazla izleyiciye hizmet eder).

Multi-CDN failover:
  Primary: CloudFront (geniş ağ, iyi fiyat).
  Secondary: Akamai (Asya'da güçlü).
  Tertiary: Fastly (Avrupa).
  GeoDNS: izleyici lokasyonu → en yakın CDN.
  Failover: CDN hit rate < %70 → diğer CDN'e yönlendir.
  Maliyet: farklı CDN farklı fiyat → trafik yönlendirme optimizasyonu.

Origin shield:
  CDN → doğrudan origin değil → "shield" node'u.
  Shield: tek noktadan origin → CDN'e; CDN node'ları birbirine değil shield'e sorar.
  Origin istek sayısı: N CDN PoP → 1 Shield → origin.
  Bant genişliği tasarrufu: origin'e %10 istek bile yetebilir.
```

---

## Stream DVR ve Klip Oluşturma

### DVR (Geriye Sarma)

```
Kullanım: izleyici yayına geç katıldı → 10 dk geriye sar.
"Buffer": son 30 dakikanın segmentleri sürekli tutulur.

S3 Ring Buffer:
  Her segment (2s): S3'e yükle.
  Key: streams/{streamId}/dvr/{segmentIndex}.ts
  30 dk = 900 segment → maksimum 900 segment tut.
  Eski segment (900+'dan önceki): S3'ten sil veya Glacier'a taşı.

  Lifecycle rule (S3):
    Prefix: streams/*/dvr/*
    Expiration: 30 dk (1800 sn) → otomatik sil.

DVR playlist oluşturma:
  Normal HLS: sadece son N segment playlist'te.
  DVR playlist: tüm son 30 dk segment listesi → player istediği yerden başlar.
    #EXT-X-PLAYLIST-TYPE:EVENT  (canlı + geçmiş)
    ...tüm DVR segmentleri...
    #EXT-X-ENDLIST (yayın bittiyse)

  DVR Service: GET /dvr/{streamId}?offset=-600 → 10 dk öncesinden playlist.
  offset = 0 → canlı kenar, offset = -1800 → 30 dk önce.

DVR + CDN:
  DVR segmentleri: CDN'de cache (değişmez) → hızlı erişim.
  "Bu yayın zaten CDN'de" → DVR için ek pull yok.
```

### Klip Oluşturma

```
Kullanım: "Son 30 saniyeyi kaydet ve paylaş."

Akış:
  1. Kullanıcı: POST /clips { streamId, durationSec: 30 }
  2. Clip Service: DVR buffer'dan son 30 saniyenin segment listesini bul.
  3. Async worker (Kafka consumer):
     a. S3'ten ilgili segmentleri indir (paralel).
     b. FFmpeg: birleştir → tek MP4 dosyası → trim (başlangıç, bitiş).
     c. Thumbnail: 5. sn → JPEG → S3.
     d. MP4 → S3: clips/{clipId}.mp4
     e. metadata → DB kaydet.
  4. Kullanıcıya: "Klibin hazır: /clips/{clipId}"

MP4 transmux (encode değil):
  Segmentler zaten H.264 → sadece HLS TS → MP4 container (transmux).
  FFmpeg -c copy: yeniden encode yok → hızlı (CPU az), kalite kaybı yok.
  30 sn klip: ~2-5 sn işlem süresi.

Paylaşım URL:
  https://cdn.mystream.com/clips/{clipId}.mp4
  CDN: TTL = 86400 (değişmez veri).
  Public: herkese açık (link bilen izler).
  Private: signed URL (sadece izin verilene).

Rate limiting:
  Kullanıcı başına: 3 klip/dakika (spam önleme).
  Popüler stream: çok klip → S3 cost artabilir → monitör et.
```

---

## Canlı Chat

### Mimari

```
Chat ölçeği:
  1M eş zamanlı izleyici → 100 msg/sn ortalama.
  Peak (büyük event, gol anı): 10,000 msg/sn.
  Mesaj boyutu: 200B → 10,000 × 200B = 2 MB/sn → küçük.

Servis mimarisi:
  Chat Gateway (WebSocket sunucuları):
    İzleyici → WebSocket bağlantısı → Chat Gateway.
    Chat Gateway → Redis Pub/Sub SUBSCRIBE stream:{streamId}:chat.
    1M bağlantı / 50K/server = 20 Gateway server.

  Yayın akışı:
    Kullanıcı mesaj gönderir → Chat Gateway → doğrulama + rate limit.
    Geçtiyse → Redis PUBLISH stream:{streamId}:chat {msg}.
    Tüm o channel'ı subscribe eden Gateway server'lar → mesajı alır → WebSocket push.

  Redis Pub/Sub ölçeği:
    Tek Redis: 1M subscriber → CPU darboğazı.
    Redis Cluster: stream ID hash → farklı node.
    Her stream farklı node'da → yük dağıtımı.

  Kalıcı depolama (Kafka + ClickHouse):
    Chat mesajı → Kafka topic: chat-messages.
    Consumer: ClickHouse'a yaz → analytics, arama.
    VOD replay: o zamandaki chat mesajlarını getir → timestamp eşle.
```

### Chat Moderasyon

```
Banned word filter:
  Kelime listesi → Redis Set: chat:banned_words.
  Mesaj → her kelimeyi kontrol et → O(N) ama N küçük (birkaç bin kelime).
  Regex: "r@cist" → "racist" algıla (leetspeak bypass önleme).
  False positive: "class" → "ass" içeriyor → context-aware gerekli.

AutoMod (ML tabanlı):
  Model: BERT/DistilBERT → mesajı classify et (spam/toxic/ok).
  Threshold: confidence > 0.95 → otomatik sil; > 0.80 → moderatöre gönder.
  Realtime: her mesaj → model inference → 10ms → karar.
  Model güncellemesi: yeni toksik dil → fine-tune → re-deploy.

Rate limiting (spam önleme):
  Kullanıcı başına: max 1 mesaj/2 sn (sliding window).
  Yeni hesap (< 7 gün): max 1 mesaj/5 sn.
  Ban: geçici (1 saat), kalıcı (account ban).
  Redis: INCR user:{userId}:chat_count EX 2 → count > 1 → reddedile

Slow mode (büyük event):
  Kanal yayıncısı aktive eder: "30 sn slow mode."
  Herkes sadece 30 sn'de bir mesaj gönderebilir.
  Peak 10,000 msg/sn → slow mode → birkaç yüz msg/sn → yönetilebilir.

Subscriber-only mode:
  Sadece abone olanlar chat yazabilir → spam %80 azalır.
  İzleyici: okuyabilir, yazamaz.

Chat replay (VOD'da):
  Kayıt sırasındaki mesajlar → timestamp ile Kafka'ya.
  VOD oynatılırken: video timestamp → o zamandaki chat mesajları.
  Örn: video 00:03:45 → chat o dakikadaki mesajları göster.
  ClickHouse: stream_id + video_timestamp range query → hızlı.
```

---

## Analytics Pipeline

### Gerçek Zamanlı Metrikler

```
İzleyici sayısı (anlık):
  WebSocket bağlantısı açıldı → Redis INCR stream:{id}:viewers.
  Bağlantı kapandı → Redis DECR.
  Sorun: CDN üzerinden izleyici → WebSocket bağlantısı yok → sayılmıyor.

  Çözüm: HLS player heartbeat:
    Player: her 30 sn → POST /analytics/heartbeat { streamId, sessionId }.
    Analytics Service: INCR stream:{id}:hls_viewers (TTL 60sn per session).
    Toplam viewer = WebSocket viewers + HLS viewers.

Unique viewer (HyperLogLog):
  PFADD stream:{streamId}:unique {userId/sessionId}
  PFCOUNT stream:{streamId}:unique → tahmini unique izleyici.
  Hafıza: 12 KB ile milyonlarca unique tahmin.

Peak viewer:
  Her 5sn: current_viewers → peak ile karşılaştır → peak > current → peak güncelle.
  Redis: SET stream:{id}:peak_viewers {count} (sadece arttıkça).
  Yayın bitti → peak DB'ye kaydet.

Analytics pipeline (Kafka → Flink → ClickHouse):
  Events: StreamStarted, ViewerJoined, ViewerLeft, ChatMessage, ClipCreated.
  Kafka: topic = stream-events.
  Flink: 1 dk sliding window → viewer count, chat rate, engagement score.
  ClickHouse: aggregated metrics → dashboard sorguları.
```

### Yayın Kalite Metrikleri

```
Player tarafı metrikler (hls.js events):
  Buffer stall: buffering kaç kez, kaç sn sürdü.
  Quality switch: kaç kez kalite değişti, hangi yönde (up/down).
  Startup time: play tuşundan ilk frame'e kadar geçen süre.
  Error rate: player hata sayısı (manifest yüklenemedi, segment 404).

  Her event → POST /analytics/quality { streamId, sessionId, eventType, data }.
  Kafka → Flink: aggregation (ortalama, P99).
  Dashboard: "Bugün %3.5 izleyici 5sn+ buffer yaşadı."

Origin health monitoring:
  Prometheus: ingest server → active_streams, encoding_lag, dropped_frames.
  Transcoding lag: ingest timestamp - encode complete timestamp → target < 2sn.
  CDN hit rate: (CDN serve) / (CDN serve + origin pull) → hedef %95+.
  Alarm: hit rate < %80 → "CDN cache sorunlu, origin yük artıyor."
```

---

## Stream Keşfi ve Öneri

```
Kategori bazlı keşif:
  Elasticsearch index (streams):
    { streamId, streamerId, title, category, language, viewerCount,
      startedAt, thumbnailUrl, tags, isLive }
  Arama: "valorant türkçe" → title + tags + category match.
  Sıralama: viewer_count DESC (popüler önce) veya started_at DESC (yeni).

Öneri (recommendation):
  Collaborative filtering: "Bu yayını izleyenler şunu da izledi."
  Takip tabanlı: takip ettiğin yayıncı canlıya geçti → bildirim + keşfette göster.
  Benzer kategori: "Valorant izliyordun, şu CS:GO yayını da sana uyar."

Keşif feed algoritması:
  Kişiselleştirilmiş: geçmiş izleme geçmişi + dil + saat.
  Trending: son 1 saatte en hızlı büyüyen yayınlar.
  Yeni yayıncılar: ilk 10 kez yayın → keşif sayfasında boost (topluluk büyütme).

Thumbnail:
  Her 30 sn: transcoder → anlık ekran görüntüsü → S3 (CDN TTL 30sn).
  İzleyici: thumbnail ön izleme (hover) → ilgi çekici an.
```

---

## Veri Modeli

```sql
CREATE TABLE streams (
    stream_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    streamer_id      UUID NOT NULL,
    title            VARCHAR(500),
    description      TEXT,
    category         VARCHAR(100),     -- Gaming, Music, IRL, Talk, Education
    language         CHAR(5),          -- tr, en, de
    status           VARCHAR(20) DEFAULT 'OFFLINE'
                     CHECK (status IN ('LIVE','OFFLINE','ERROR','PROCESSING')),
    started_at       TIMESTAMPTZ,
    ended_at         TIMESTAMPTZ,
    duration_secs    INT,              -- hesaplanan süre
    peak_viewers     INT DEFAULT 0,
    total_views      BIGINT DEFAULT 0,
    unique_viewers   BIGINT DEFAULT 0,
    chat_messages    BIGINT DEFAULT 0,
    recording_url    TEXT,             -- S3 path (ended sonrası)
    thumbnail_url    TEXT,             -- anlık thumbnail
    is_dvr_enabled   BOOLEAN DEFAULT TRUE,
    stream_key_hash  CHAR(64) UNIQUE,  -- SHA-256(streamKey), düz metin değil!
    tags             TEXT[],
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_streams_live   ON streams (status, category, started_at DESC)
  WHERE status = 'LIVE';
CREATE INDEX idx_streams_streamer ON streams (streamer_id, started_at DESC);

CREATE TABLE stream_quality_segments (
    stream_id    UUID REFERENCES streams(stream_id),
    quality      VARCHAR(10),    -- 1080p, 720p, 480p, 360p
    segment_idx  BIGINT,
    s3_key       TEXT NOT NULL,
    duration_ms  INT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (stream_id, quality, segment_idx)
);

CREATE TABLE clips (
    clip_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id      UUID REFERENCES streams(stream_id),
    creator_id     UUID NOT NULL,
    title          VARCHAR(200),
    s3_key         TEXT,
    thumbnail_url  TEXT,
    duration_secs  INT,
    start_offset   INT,           -- yayındaki başlangıç saniyesi
    view_count     BIGINT DEFAULT 0,
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE stream_viewers (
    stream_id    UUID REFERENCES streams(stream_id),
    user_id      UUID,
    session_id   UUID,
    joined_at    TIMESTAMPTZ,
    left_at      TIMESTAMPTZ,
    watch_secs   INT,             -- kaç saniye izledi
    max_quality  VARCHAR(10),     -- izlediği en yüksek kalite
    PRIMARY KEY (stream_id, session_id)
) PARTITION BY RANGE (joined_at);

CREATE TABLE chat_messages (
    message_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id    UUID NOT NULL,
    user_id      UUID NOT NULL,
    content      TEXT NOT NULL,
    sent_at      TIMESTAMPTZ DEFAULT NOW(),
    is_deleted   BOOLEAN DEFAULT FALSE,
    deleted_by   UUID
) PARTITION BY RANGE (sent_at);
-- Aylık partition → eski mesajlar arşiv

CREATE INDEX idx_chat_stream ON chat_messages (stream_id, sent_at);
```

---

## Olası Sorunlar ve Çözümleri

### 1. Ingest Server Çöktü — Yayın Kesildi

```
Sorun:
  Yayıncı 3 saatlik turnuva yayını yapıyor.
  Ingest server hardware arızası → saat 2:15'te çöktü.
  İzleyiciler: "Bağlantı kesildi" hatası → yayın sayfasında "Offline."
  Yayıncı: OBS "Stream disconnected" → reconnect deniyor → farklı server gerekiyor.
  Sonuç: 5,000 izleyici yayını kaçırdı → chat "N** nerede?"

Çözüm:
  a) OBS failover URL (birincil koruma):
     OBS Settings: Stream → "Use authentication" → Primary + Backup URL.
     Primary: rtmp://ingest-1.mystream.com/live/{key}
     Backup:  rtmp://ingest-2.mystream.com/live/{key}
     OBS: primary 30sn cevap vermezse → backup'a otomatik geç.
     Yayıncı tarafı: kesintisiz geçiş, OBS halleder.

  b) Hot standby (sunucu tarafı):
     Her aktif ingest → paralel olarak backup ingest'e stream kopyalanır.
     Primary down → Coordinator: stream kaynağını backup'a yönlendir.
     Transcoder: source URL değişir → izleyiciler fark etmez.
     Geçiş süresi: < 5sn (izleyici: 1-2 segment atlayabilir).

  c) HLS buffer (izleyici tarafı):
     Player: 3 segment (6sn) buffer tutar.
     Ingest down → transcoding dur → yeni segment gelmiyor.
     Player: buffer bitti → "buffering" gösterir.
     Failover 5sn'de tamamlanırsa → player buffer bitene kadar bekledi → sorunsuz.
     Failover > 15sn → player "stream ended" kabul eder → kullanıcıya "yenile" mesajı.

  d) Heartbeat monitoring:
     Her ingest server → her 5sn → health check endpoint.
     3 ardışık miss (15sn) → "UNHEALTHY" → otomatik failover tetikle.
     PagerDuty: ops uyarısı → sunucu incele.
```

---

### 2. Viral An — Ani 100x İzleyici Artışı, CDN Cache Miss

```
Sorun:
  Normal: 500 izleyici. Büyük streamer: "Şu yayına bakın!" → retweet.
  10 sn içinde: 500 → 50,000 izleyici.
  Hepsi aynı anda segment istedi.
  CDN: segment cache yok (soğuk) → 50,000 istek origin'e → origin overload.
  Origin: CPU %100 → yavaş cevap → CDN timeout → izleyiciler buffering.
  Domino etkisi: CDN retry → origin daha da kötü → cascade failure.

Çözüm:
  a) Request coalescing (CDN katmanı):
     Aynı segment için 1,000 istek geldi → CDN 1 istek origin'e gönderir.
     1,000 istek bekler → origin cevap → 1,000'e aynı anda dağıt.
     Cloudfront: collapse_concurrent_requests → built-in.
     Origin tek istek görür, CDN binlercesini halleder.

  b) Origin shield:
     CDN node → doğrudan origin değil → shield node.
     50,000 CDN istek → shield'e → shield 1 istek origin'e.
     Origin: sadece shield görür → overload yok.

  c) Pre-warm (viral tahmin):
     ML model: "Bu yayın viral olabilir" → retweet/mention artışı tespit.
     Pre-warm: CDN edge'lerine segmentleri önceden push et.
     Gerçek izleyici geldiğinde: cache hit → origin bypass.
     Gerçek zamanlı alarm: "Bu yayın trend oluyor → auto-scale transcoder."

  d) Graceful degradation:
     Origin overload → 480p ve 360p servis et (1080p ve 720p kapat geçici).
     Bant genişliği %50 azaldı → origin kurtuldu.
     Player: ABR otomatik düşük kaliteye iner → buffering azalır.
     5 dk sonra: origin toparlandı → kalite normale döner.
```

---

### 3. Transcoding Lag — Yayın 30 Sn Gecikmeli

```
Sorun:
  Normal latency: 6sn (LL-HLS).
  Sabah 03:00: batch transcoding gorevi yanlış configure edildi → transcoding server'larda %80 CPU.
  Canlı transcoding: CPU alamıyor → her segment encode süresi 8sn (target: 2sn).
  Yayıncı → izleyici arası: 30sn gecikme.
  Sporcular: "Gol attı, 30sn sonra chat'te gördüm" → spoiler.

Çözüm:
  a) Resource isolation:
     Canlı transcoding: dedicated instance (yüksek öncelik, spot değil on-demand).
     Batch job (VOD transcode): spot instance (ucuz ama kesintiye uğrayabilir).
     Kubernetes: canlı transcoding pod → guaranteed QoS (CPU/memory guaranteed).
     Batch: BestEffort → kaynak varsa çalışır.

  b) Encoding lag monitoring:
     Prometheus: encode_lag_secs = now - segment_start_time.
     Lag > 3sn → alert → "Transcoding geride kalıyor."
     Lag > 8sn → auto-scale: yeni GPU instance ekle.
     KEDA: Kafka queue depth / Prometheus metric → HPA.

  c) Adaptive GOP:
     Normal: GOP 2sn (60 frame @30fps).
     Lag yüksek → GOP kısalt (GOP 1sn) → encoder daha erken flush → latency azalır.
     Trade-off: küçük GOP → daha az compression → bitrate artar.

  d) Hardware encoder önceliği:
     NVENC: CPU encode'un 10x hızlı.
     Tüm ingest → NVENC GPU instance (spot yerine reserved).
     CPU encode: fallback (NVENC kapasitesi aşılırsa).
```

---

### 4. Chat Thundering Herd — Viral An'da Chat Patladı

```
Sorun:
  Büyük turnuva finali: gol anı.
  1M izleyici → aynı anda mesaj gönderdi → 1M msg/sn.
  Redis Pub/Sub: 1M publish → 20 chat server her biri 1M mesaj aldı.
  Her chat server: 1M mesaj × 50K bağlantı = impossible.
  Gerçek: Redis Pub/Sub CPU %100 → mesajlar drop ediliyor.
  İzleyici: mesajlar gitmiyor, chat donmuş gibi.

Çözüm:
  a) Fan-out throttling:
     Chat gateway: kullanıcı başına max 1 msg/2sn (katı).
     1M kullanıcı → max 500K msg/2sn = 250K msg/sn → yönetilebilir.
     Slow mode: yayıncı aktif eder → herkes 30sn'de bir.

  b) Message batching + sampling:
     Viral an: mesajları 100ms'de bir batch yap → tek Redis publish.
     100ms × 1,000 msg → 1 batch publish → 10,000 batch/sn (1M yerine).
     İzleyici: 100ms gecikmeli chat → fark edilmez.

  c) Chat message sampling (görünür chat):
     1M msg/sn → izleyici ekranda sadece 50 msg/sn görebilir (hız okuma).
     Sampling: her 20 mesajdan 1'ini göster (rastgele).
     Twitch: "Yavaş chat" = sampling aktif → "Bu kanal çok popüler."
     Pubsub: azaltılmış mesaj → CDN/WS yükü düşer.

  d) Redis Cluster (pub/sub sharding):
     Channel = stream:{streamId}:chat → her stream farklı shard.
     1M eş zamanlı yayın → 1M channel → cluster'a dağılır.
     Tek yayın: tek Redis shard → 1M subscriber problem hâlâ var → sampling şart.
```

---

### 5. Stream Key Sızdı — Yetkisiz Yayın

```
Sorun:
  Yayıncı stream key'ini yanlışlıkla ekran görüntüsünde yayınladı.
  Kötü aktör: bu key ile yayıncı adına sahte içerik yayınladı.
  Gerçek yayıncı: kendi hesabından sahte bir yayın gördü → panik.
  İzleyiciler: zararlı içerik izledi → şikayet.

Çözüm:
  a) Anlık key sıfırlama:
     Dashboard: "Stream Key'imi Sıfırla" → 1 tık.
     Eski key: DB'de geçersiz → ingest server auth red.
     Unauthorized stream: 5sn içinde kesilir.
     Yayıncı: yeni key ile yeniden başlatır.

  b) Key rotation (periyodik):
     Her 90 günde bir: "Key'inizi yenilemenizi öneririz" uyarısı.
     Yüksek güvenlik: key her yayında otomatik değişir (RTMPS + JWT).

  c) Geo-IP kısıtlama:
     Yayıncı: "Sadece Türkiye IP'den bağlanabilirim."
     Unauthorized: farklı ülke IP → auth red.
     Dezavantaj: VPN kullanan gerçek yayıncıyı engelleyebilir.

  d) Double-connect detection:
     Aynı key ile aynı anda 2 bağlantı → ikinci bağlantıyı kes.
     Birincisi yayıncı, ikincisi saldırgan → ikincisi kesilir.
     Yayıncı: kesintisiz devam.
     Sorun: hangisi gerçek yayıncı? → IP whitelist ile birleştir.

  e) Content audit (ML):
     Yayın içeriği → frame sampling → NSFW detection.
     İhlal → otomatik askıya al → yayıncıya email.
     Hızlı: < 30sn'de tespit (frame her 30sn bir analiz).
```

---

### 6. Yayın Kaydı Bozuldu — VOD Oynatılamıyor

```
Sorun:
  3 saatlik büyük turnuva → VOD izleyici 2 dk sonra takılıyor.
  Segment 60 corrupted → FFmpeg encode hatası → 2 saniyelik segment bozuk.
  HLS player: bozuk segment → HTTP 200 ama geçersiz data → decode hatası → player crash.
  3 saat yayın: 1 bozuk segment yüzünden izlenemiyor.

Çözüm:
  a) Segment doğrulama (encode sonrası):
     Her encode → FFprobe ile doğrula:
       ffprobe -v error -show_entries packet=pts_time segment.ts
     Hata → yeniden encode et (retry 2 kez).
     Hâlâ hata → skip segment + gap marker:
       #EXT-X-GAP → player: bu segment yok, atla.
     İzleyici: 2 sn atlama (3 saatlik videoda kabul edilebilir).

  b) Checksum (S3 integrity):
     S3 upload: ETag = MD5(segment).
     Player fetch: ETag kontrol → uyuşmuyorsa → CDN purge → yeniden fetch.
     Corruption: S3 veya transit bozulması → tespit edilir.

  c) Parallel recording (redundancy):
     İki farklı format: HLS TS segmentleri + MP4 fragmented.
     HLS bozulsa → MP4'ten dönüştür.
     Ayrı S3 prefix: recordings/hls/ ve recordings/mp4/.

  d) VOD post-processing:
     Yayın bitti → tüm HLS segment → tek MP4 (FFmpeg concat + remux).
     Kesintisiz dosya: bozuk segment → zaten doğrulanmış segment var.
     MP4 index (moov atom): başa taşı → seek edilebilir (MP4 progressive download).
     İzleyici: MP4'ü istediği noktadan başlatabilir → HLS playlist gereksiz (VOD'da).
```

---

### 7. Encoding Farm Kapasitesi Yetersiz — Yeni Yayınlar Bekliyor

```
Sorun:
  Büyük e-spor etkinliği: 5,000 yeni yayın aynı anda başladı.
  Transcoding farm: 1,000 GPU kapasiteli (normal için yeterli).
  5,000 yayın → 4,000 yayın kuyruğa girdi → ortalama bekleme 8 dk.
  8 dk boyunca izleyici: "Yayın bulunamadı" → başka yere gitti.

Çözüm:
  a) Auto-scaling (KEDA):
     Kafka topic: encoding-queue.
     KEDA: queue depth > 100 → yeni transcoder pod ekle.
     5,000 yayın → queue spike → KEDA → 5,000 pod talep.
     AWS GPU Spot: 5,000 × 4 = 20,000 NVENC encode (5,000 stream × 4 kalite).
     Spot instance: %70 ucuz ama kesintiye uğrayabilir → spot + on-demand mix.

  b) Graceful degradation (kalite kısıtlama):
     Normal: 4 kalite (1080p, 720p, 480p, 360p).
     Kapasite %80+: 2 kalite (720p + 360p) → kaynak %50 azaldı.
     Kapasite %100: sadece 480p → en az kaynak.
     İzleyici: en popüler kanallar önce full kalite, küçükler kısıtlı.

  c) Priority queue:
     Büyük streamer (100K+ takipçi): yüksek öncelik encode.
     Yeni yayıncı: düşük öncelik (izleyicisi az → bekleme tolere edilebilir).
     Premium hesap: guaranteed encoding (SLA).

  d) Önceden ölçeklendirme:
     E-spor takvimi → bilinen büyük etkinlikler → 2 saat önce kapasiteyi 5x artır.
     AWS: scheduled scaling (CloudWatch Events → Lambda → Auto Scaling).
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Ingest protokolü | RTMP (TCP) | RTMP + SRT seçenek | RTMP: uyumluluk; SRT: kötü ağda daha iyi |
| Streaming protokolü | Normal HLS (10sn) | LL-HLS (2sn) | Kullanıcı deneyimi: spoiler, etkileşim |
| Ultra low latency | LL-HLS | WebRTC + SFU | WebRTC: < 1sn, etkileşim; LL-HLS: 1M+ ölçek |
| Encoding | CPU yazılım | GPU NVENC | GPU: 10x hızlı, maliyet avantajlı toplamda |
| CDN | Origin push | Pull + Origin Shield | Pull: dinamik cache, shield: origin koruması |
| Chat | HTTP polling | WebSocket + Redis Pub/Sub | Polling: gecikme, yük; WS: anlık, verimli |
| Conflict (multi-device) | LWW timestamp | Kafka sequential | Kafka sıralı → chat mesaj sırası garanti |
| DVR storage | Ayrı kayıt | HLS segment ring buffer | Ayrı kayıt: 2x depolama; Ring: zaten var |
| Ingest HA | Tek server | Primary + Backup + Coordinator | Tek: SPOF; Primary-Backup: kesintisiz failover |
| Viral scale | Sabit kapasite | KEDA + auto-scale + origin shield | Sabit: viral'de çöker; KEDA: esnek kapasite |
| VOD format | HLS (segmentli) | MP4 (remux, tek dosya) | HLS: streaming; MP4: seek, download, compat |
