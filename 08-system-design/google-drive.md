# 08h — System Design: Google Drive / Dropbox

## Gereksinimler

```
Functional:
  ✓ Dosya yükleme, indirme, silme
  ✓ Dosya senkronizasyonu (birden fazla cihaz)
  ✓ Klasör yapısı
  ✓ Dosya paylaşımı (link, kullanıcı ile)
  ✓ Versiyon geçmişi
  ✗ Gerçek zamanlı ortak düzenleme (out of scope)

Non-Functional:
  50M kullanıcı, 10M aktif/gün
  Her kullanıcı: 15 GB ücretsiz depolama
  Upload: 10M × 2 dosya/gün = 20M upload/gün
  Availability: 99.99%
  Veri kaybı: sıfır (11 nines durability)
```

---

## Capacity Estimation

```
Storage:
  50M kullanıcı × 15 GB = 750 PB (toplam kapasite)
  Kullanım oranı %30 → 225 PB gerçek veri

Upload QPS:
  20M / 86400 ≈ 230 upload/sn
  Ortalama dosya: 500 KB
  Upload bandwidth: 230 × 500 KB = 115 MB/sn

Download bandwidth:
  Read:Write = 10:1 → 1.15 GB/sn → CDN ile dağıtılır

Metadata:
  Dosya başına: 1 KB (ad, boyut, hash, path, owner, permissions)
  20M × 1 KB = 20 GB/gün metadata
```

---

## High-Level Tasarım

```
         Desktop/Mobile Client
                 │
          ┌──────┴───────┐
          │    Client    │
          │    Sync      │
          │    Agent     │
          └──────┬───────┘
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
┌─────────┐ ┌────────┐ ┌──────────┐
│  Block  │ │ Meta   │ │ Sharing  │
│  Serv.  │ │ Serv.  │ │ Service  │
└────┬────┘ └────┬───┘ └──────────┘
     │           │
     ▼           ▼
┌─────────┐ ┌────────┐
│   S3    │ │ MySQL/ │
│ (files) │ │ Cassandra│
└─────────┘ └────────┘
     │
     ▼
   CDN
(download)
```

---

## Block Storage: Chunk Bazlı Upload

### Neden Chunk?

```
Büyük dosyalar (5 GB video):
  Tek parça upload → ağ kesintisi → baştan başla → kötü UX
  Chunk upload: her 4 MB ayrı chunk → kesilirse sadece o chunk tekrar

Delta sync:
  10 GB dosyada 1 KB değişti → tüm dosyayı tekrar yükleme
  Sadece değişen chunk'ı yükle → bandwidth tasarrufu
```

### Chunk Upload Akışı

```
Dosya = N × 4 MB chunk

1. Client → Meta Service: "file.pdf upload etmek istiyorum"
   → Dosya hash'i gönder (SHA-256 of entire file)
   → Server: "Bu hash zaten var mı?" → var → deduplication (upload etme)

2. Her chunk için:
   chunk_hash = SHA-256(chunk_bytes)
   → Block Service: "Bu chunk var mı?" (hash kontrolü)
   → Yok: S3 presigned URL al → chunk'ı direkt S3'e yükle
   → Block Service: "chunk {hash} hazır" kaydını güncelle

3. Tüm chunk'lar yüklendi:
   → Meta Service: file manifest kaydet
   {fileId, userId, path, totalSize, chunkHashes: [h1, h2, h3, ...]}

4. Download:
   → Meta'dan chunk listesi al
   → Her chunk → CDN/S3'ten paralel indir → birleştir
```

### Deduplication

```
İki kullanıcı aynı dosyayı (Ubuntu ISO) yüklerse:
  User A → hash: "abc123" → S3'e yükle
  User B → hash: "abc123" → "Zaten var" → upload etme

Chunk bazlı deduplication:
  4 MB chunk hash → aynı hash → aynı S3 key → tek kopya

Storage kazancı:
  %30-40 storage tasarrufu tipik (özellikle kurumsal kullanımda)

Reference counting:
  chunk_ref_count: her chunk'ın kaç dosya tarafından kullanıldığını tut
  Dosya silindiğinde → ref count azalt → 0 ise chunk'ı sil (GC)
```

---

## Senkronizasyon

### Sync Akışı

```
Laptop'ta değişiklik:
  Client Sync Agent → dosya değişikliklerini izle (inotify/FSEvents/ReadDirectoryChangesW)
  Değişen dosya → chunk'lara böl → değişen chunk'ları tespit et (diff)
  Değişen chunk'ları yükle → Meta güncelle

Mobil cihazda güncelleme:
  Long polling veya WebSocket ile değişiklik bildirimi al
  Notification: "Dizüstü bilgisayarda file.pdf güncellendi"
  → Güncel chunk listesini al → değişen chunk'ları indir

Sync servisi bildirim yayını:
  File updated → Redis Pub/Sub: PUBLISH "user:123:sync" {fileId, version}
  → Client'ın WebSocket bağlantısı → anında sync başlat
```

### Conflict Resolution

```
Senaryo: Aynı dosyayı offline iken iki cihaz düzenledi

Dropbox yaklaşımı:
  İki versiyonu da sakla:
  - "document.pdf" (sunucudaki son versiyon)
  - "document (Laptop çakışan kopya 2024-01-15).pdf"
  Kullanıcı hangisini tutacağını seçer

Google Drive yaklaşımı:
  Her düzenleme ayrı versiyon olarak kaydedilir
  Versiyon geçmişinde tüm versiyonlar görülür

Timestamp-based (basit ama hatalı):
  Daha yeni timestamp kazanır → clock sync sorunu

CRDT (Conflict-free Replicated Data Type):
  Metin için operational transform (Google Docs)
  Gerçek zamanlı ortak düzenleme → sıradışı sorun, ağırlıklı Dropbox bu yolu seçmez
```

---

## Dosya Paylaşımı

```
Link ile paylaşım:
  GET /share/{token}
  token = random 20 char (UUID yeterince güvenli)
  
  share_links tablosu:
    {token, fileId, ownerId, permissions (READ/WRITE), expiry, maxDownloads}

Kullanıcı ile paylaşım:
  file_permissions tablosu:
    {fileId, userId, role (VIEWER/EDITOR/OWNER)}

ACL kontrolü (her dosya erişiminde):
  1. Owner mı? → izin ver
  2. file_permissions tablosunda var mı? → role'e göre
  3. Link ile mi geliyor? → share_links token kontrol
  4. Hiçbiri → 403 Forbidden
```

---

## Versiyon Geçmişi

```
Her dosya güncellemesinde:
  Yeni versiyon = yeni chunk seti
  Eski chunk'lar silinmez (versiyon için tutulur)

file_versions tablosu:
  {fileId, versionNumber, chunkHashes[], createdAt, size}

30 gün versiyon saklama:
  30 günden eski versiyonlar → S3 Glacier (ucuz arşiv)
  Kullanıcı restore isterse → S3 Glacier'dan geri getir (3-12 saat)

Storage maliyet optimizasyonu:
  Değişmeyen chunk'lar tüm versiyonlarca paylaşılır (ref counting)
  Sadece gerçekten değişen kısmı sakla
```

---

## Metadata DB Şeması

```sql
CREATE TABLE files (
    file_id       UUID         PRIMARY KEY,
    owner_id      UUID         NOT NULL,
    parent_folder UUID,
    name          VARCHAR(500) NOT NULL,
    size_bytes    BIGINT,
    mime_type     VARCHAR(100),
    file_hash     CHAR(64),    -- SHA-256 of entire file
    current_version INT DEFAULT 1,
    is_deleted    BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP,
    updated_at    TIMESTAMP,

    INDEX idx_owner_path (owner_id, parent_folder),
    INDEX idx_hash (file_hash)           -- deduplication lookup
);

CREATE TABLE file_chunks (
    file_id     UUID,
    version     INT,
    chunk_index INT,
    chunk_hash  CHAR(64),    -- SHA-256 of chunk
    chunk_size  INT,
    s3_key      VARCHAR(500),
    PRIMARY KEY (file_id, version, chunk_index)
);

CREATE TABLE chunks (
    chunk_hash    CHAR(64) PRIMARY KEY,
    s3_key        VARCHAR(500) NOT NULL,
    size_bytes    INT,
    ref_count     INT DEFAULT 1,    -- kaç dosya kullanıyor
    created_at    TIMESTAMP
);
```

---

## Ölçekleme

```
Upload bottleneck:
  Block Service → stateless, yatay ölçeklenebilir
  S3: teorik sınırsız upload (presigned URL → direkt yükle)

Metadata bottleneck:
  MySQL → read replicas (read-heavy)
  Büyük dosya listesi → Cassandra (user_id bazlı sharding)

Sync notification:
  100M cihaz WebSocket → WebSocket sunucuları (Netty)
  Redis Pub/Sub → değişiklik bildirimi
  Long polling fallback (WebSocket yoksa)

CDN:
  Download → CloudFront (presigned S3 URL → CDN edge)
  S3 Transfer Acceleration: upload için AWS backbone kullan
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Upload | Tek parça | Chunk bazlı | Chunk (delta sync, résumé) |
| Dedup | Dosya level | Chunk level | Chunk (daha iyi tasarruf) |
| Conflict | Last-write-wins | Çift kopya | Çift kopya (veri kaybı yok) |
| Sync | Polling | WebSocket | WebSocket (gerçek zamanlı) |
| Metadata DB | MySQL | Cassandra | MySQL (ACID, küçük ölçek) |
