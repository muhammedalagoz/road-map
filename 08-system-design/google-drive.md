# 08h — System Design: Google Drive / Dropbox

## Gereksinimler

```
Functional:
  ✓ Dosya yükleme, indirme, silme
  ✓ Dosya senkronizasyonu (birden fazla cihaz)
  ✓ Klasör yapısı
  ✓ Dosya paylaşımı (link, kullanıcı ile)
  ✓ Versiyon geçmişi
  ✓ Dosya arama (isim + içerik)
  ✓ Thumbnail / önizleme
  ✗ Gerçek zamanlı ortak düzenleme (out of scope)

Non-Functional:
  50M kullanıcı, 10M aktif/gün
  Her kullanıcı: 15 GB ücretsiz depolama
  Upload: 10M × 2 dosya/gün = 20M upload/gün
  Availability: 99.99%
  Durability: 11 nines (veri kaybı sıfır tolerans)
  Upload latency: büyük dosyalarda kesilebilir + devam edilebilir
  Sync latency: değişiklikten cihaza ulaşma < 5 sn
```

---

## Capacity Estimation

```
Storage:
  50M kullanıcı × 15 GB = 750 PB kapasite
  Kullanım oranı %30 → 225 PB gerçek veri
  Dedup ile %30 kazanç → net: ~160 PB S3'te

Upload:
  20M upload / 86400 ≈ 230 upload/sn
  Ortalama dosya: 500 KB
  Upload bandwidth: 230 × 500 KB ≈ 115 MB/sn
  Peak (3x): 345 MB/sn → CDN edge upload + S3 Transfer Acceleration

Download:
  Read:Write = 10:1 → 1.15 GB/sn → CDN olmadan imkansız
  CDN cache hit oranı hedef: %90+ (popüler shared dosyalar)

Metadata:
  Dosya başına: ~1 KB (ad, boyut, hash, path, owner, permissions, chunk list)
  20M upload/gün × 1 KB = 20 GB/gün yeni metadata
  50M kullanıcı × ortalama 500 dosya × 1 KB = 25 TB toplam metadata

Chunk DB:
  20M upload/gün × ortalama 10 chunk (50 KB/chunk) = 200M chunk/gün
  Dedup sonrası: ~140M yeni chunk (geri kalan referans sayar)

WebSocket (sync):
  10M aktif cihaz × 2 cihaz/kullanıcı = 20M WebSocket bağlantısı
  Her bağlantı: ~10 KB kernel buffer → 20M × 10KB = 200 GB RAM
  WebSocket gateway: 50K bağlantı/server → 400 server gerekli
```

---

## High-Level Mimari

```
         Desktop / Mobile / Web Client
                      │
              ┌───────┴────────┐
              │  Client Sync   │  ← local SQLite, file watcher, upload queue
              │     Agent      │
              └───────┬────────┘
                      │
              ┌───────▼────────┐
              │   API Gateway  │  ← auth, rate limit, routing
              └───────┬────────┘
                      │
     ┌────────┬────────┼──────────┬──────────┐
     ▼        ▼        ▼          ▼          ▼
┌─────────┐ ┌──────┐ ┌────────┐ ┌────────┐ ┌────────┐
│  Block  │ │Meta  │ │Sharing │ │Search  │ │Notif.  │
│ Service │ │Serv. │ │Service │ │Service │ │Service │
└────┬────┘ └──┬───┘ └────────┘ └────────┘ └────┬───┘
     │          │                                 │
     ▼          ▼                                 ▼
┌─────────┐ ┌─────────────┐              ┌──────────────┐
│   S3    │ │  PostgreSQL │              │ Redis Pub/Sub│
│(chunks) │ │ (metadata,  │              │ + WebSocket  │
│         │ │  versions,  │              │   Servers    │
└────┬────┘ │  sharing)   │              └──────────────┘
     │      └─────────────┘
     ▼
  CDN Edge                   Async Workers:
(download)              ┌──────────────────────────┐
                        │ Thumbnail Generator       │
                        │ Full-text Indexer (Tika) │
                        │ Virus Scanner (ClamAV)   │
                        │ Chunk GC                 │
                        └──────────────────────────┘
```

---

## Block Storage: Chunk Bazlı Upload

### Neden Chunk?

```
Büyük dosyalar (5 GB video):
  Tek parça upload → ağ kesintisi → baştan başla → kötü UX
  Chunk upload: her 4 MB ayrı chunk → kesilirse sadece o chunk tekrar

Delta sync:
  10 GB dosyada 1 KB değişti → sadece değişen chunk'ı yükle
  Bandwidth tasarrufu: %99.99 (değişim küçükse)

Paralel upload:
  N chunk → N paralel HTTP istek → bant genişliği tam kullanım
  4 GB dosya / 4 MB chunk = 1,000 chunk → 10 paralel = 100x hız
```

### Chunk Upload Akışı (Detaylı)

```
1. Pre-upload check (dedup fırsatı):
   Client:
     file_hash = SHA-256(entire_file)
     chunk_hashes[] = [SHA-256(chunk_0), SHA-256(chunk_1), ...]

   POST /upload/prepare
   { fileName, fileSize, fileHash, chunkHashes[], mimeType }

   Server yanıtı:
   {
     uploadId: "upld-abc123",
     existingChunks: ["h1", "h2", "h5"],  // bunlar zaten var, yükleme
     presignedUrls: {                       // bunlar yeni chunk
       "h3": "https://s3.../chunk-h3?...",
       "h4": "https://s3.../chunk-h4?..."
     }
   }
   → Sadece yeni chunk'lar yüklenir → dedup önceden!

2. Chunk yükleme (paralel, S3'e direkt):
   for each new_chunk in parallel:
     PUT presignedUrls[chunk.hash] (chunk_bytes)
     S3 response: ETag = chunk hash doğrulama

3. Upload tamamlama:
   POST /upload/complete
   { uploadId, allChunkHashes[] }
   Server: tüm chunk'ların S3'te mevcut olduğunu doğrula → file_metadata kaydet.

4. Resume: yarıda kesildiyse:
   GET /upload/{uploadId}/status
   Server: tamamlananlar ve eksik chunk listesi döndür.
   Client: sadece eksik chunk'ları yükle.
```

### S3 Multipart Upload

```
S3 kendi multipart upload API'si var (5 MB min chunk boyutu):

1. Initiate: POST /bucket/key?uploads → UploadId alınır.
2. Upload parts: PUT /bucket/key?partNumber=1&uploadId=xxx (her chunk)
3. Complete: POST /bucket/key?uploadId=xxx (tüm ETag listesi)

Neden S3 multipart kullan:
  Kesintiye dayanıklı: AWS tarafında parçalar 7 gün tutulur.
  Paralel: birden fazla thread aynı anda farklı part'ları yükler.
  5 TB'a kadar tek dosya destekler.

Chunk boyutu seçimi:
  Küçük chunk (1 MB): daha fazla HTTP istek overhead → yüksek QPS
  Büyük chunk (100 MB): ağ kesilirse daha fazla tekrar yükle
  Optimum: 4-8 MB (Dropbox de bu değeri kullanır)
```

### Deduplication

```
Dosya seviyesi dedup:
  fileHash → daha önce yüklendi mi? → direkt referans ver.
  Aynı Ubuntu ISO'yu 1M kullanıcı yüklerse → 1 kopya S3'te.
  Potansiyel tasarruf: çok büyük (özellikle kurumsal).

Chunk seviyesi dedup (daha ince):
  4 MB chunk hash → aynı hash → farklı dosyalarda paylaşılan chunk.
  Tipik tasarruf: %30-40 (Dropbox rakamları).
  Neden daha iyi: büyük dosyalarda küçük değişiklik → %95+ chunk paylaşımı.

Güvenlik (cross-user dedup riski):
  "Hash ile dedup yapıyoruz" → bir kullanıcı hash'i bilirse başkasının dosyasına erişebilir mi?
  Hash proof: client dosya içeriğini gönderdiğini kanıtlamadan referans alamaz.
  Convergent encryption: client kendi key ile şifreler → aynı şifreli chunk = aynı key?
  Pratik çözüm: farklı kullanıcılar arası dedup'ı kapat, aynı kullanıcı içinde uygula.

Reference counting (GC için):
  chunks tablosu: ref_count → her chunk'ı kullanan dosya/versiyon sayısı.
  Dosya/versiyon silinince: ref_count-- → 0 ise → GC sırasında S3'ten sil.
```

---

## Client Sync Agent

```
Senkronizasyonun kalbi: yerel değişiklikleri yakalar, sunucuyla senkronize eder.

Yerel SQLite (local state):
  files_table: (path, hash, last_modified, server_version, sync_status)
  pending_uploads: (path, chunk_hashes[], priority, retry_count)
  pending_downloads: (file_id, version, chunks_needed[])

File Watcher (OS native):
  Linux:   inotify (kernel event: CREATE, MODIFY, DELETE, MOVE)
  macOS:   FSEvents API (Core Services framework)
  Windows: ReadDirectoryChangesW (Win32 API)

  Event aldığında:
    MODIFY → hash hesapla → eski hash'le karşılaştır → farklıysa upload queue'ya.
    CREATE → yeni dosya → upload queue'ya.
    DELETE → Meta Service'e bildir.
    RENAME/MOVE → Meta Service'e yeni path bildir (içerik değişmedi).

Upload Queue (öncelikli):
  Priority: kullanıcının şu an açtığı dosya → önce.
  Bandwidth throttle: %70 max upload bant genişliği (kullanıcı internet yavaşlamasın).
  Retry: üstel geri çekilme (1s, 2s, 4s, max 5 dk).

Bandwidth limiti (kullanıcı kontrolü):
  "Max upload speed: 1 Mbps" → ayarlanabilir.
  Background sync vs aktif kullanım: akıllı throttle.

Senkronizasyon döngüsü:
  1. File watcher event → değişen dosyaları tespit et.
  2. Chunking + hash hesaplama.
  3. Server'a "hangi chunk'lar eksik?" sor → sadece yenileri yükle.
  4. Upload tamamlanınca → Meta güncelle.
  5. Sunucu push notification gelirse → indir + birleştir.
  6. Local SQLite güncelle.

Offline mod:
  Bağlantı yok → değişiklikler local SQLite'ta pending olarak bekler.
  İnternet gelince → queue işle → sunucula sync.
  Çakışma olasılığı artar (offline düzenleme + sunucu güncellemesi).
```

---

## Senkronizasyon

### Sync Akışı

```
Laptop'ta değişiklik:
  File watcher → event → değişen dosya chunk'lara bölündü → diff bulundu.
  Değişen chunk'lar yüklendi → Meta güncellendi.

Meta Service → Notification Service:
  "user:123'ün file.pdf güncellendi, version 5"

Notification Service → Redis Pub/Sub:
  PUBLISH "user:123:sync" '{"fileId":"abc","version":5}'

WebSocket Server → kullanıcının tüm bağlı cihazları:
  Mobil telefon, tablet, ev bilgisayarı → bildirim alır.

Cihaz sync başlatır:
  "Son versiyonum 4, server'da 5 var → değişen chunk'ları al."
  GET /file/{id}/diff?fromVersion=4&toVersion=5
  Cevap: değişen chunk hash listesi + presigned download URL'leri.
  Paralel indir → birleştir → dosya güncellendi.
```

### Conflict Resolution

```
Senaryo: Laptop ve telefon offline iken aynı dosyayı düzenledi.
Laptop önce online oldu → version 5 yükledi.
Telefon online oldu → version 4'ten güncellemek istiyor.

Dropbox yaklaşımı (basit, kullanıcı dostu):
  İkisini de sakla:
    "document.pdf" → laptop versiyonu (son yüklenen = sunucu versiyonu)
    "document (telefon çakışan kopya 2024-01-15 14:23).pdf" → telefon versiyonu
  Kullanıcı hangisini tutacağını seçer → diğerini siler.
  Avantaj: veri kaybı yok, kullanıcı karar verir.

Google Drive yaklaşımı (versiyon geçmişi):
  Her değişiklik = yeni versiyon.
  Revision history'de her ikisi görülür → kullanıcı seçer.
  Avantaj: versiyon geçmişi temiz.

Timestamp-based (basit ama hatalı):
  Daha yeni timestamp kazanır.
  Sorun: sistem saatleri farklıysa → yanlış karar.
  Clock drift: NTP ile bile ±sn fark olabilir.
  Lamport clock veya vector clock → daha doğru ama karmaşık.

CRDT (Conflict-free Replicated Data Type):
  Metin düzenleme için (Google Docs tarzı gerçek zamanlı).
  Operation Transform (OT): her operasyon (insert, delete) → transform → merge.
  Dosya senkronizasyonu için gerekli değil (chunk bazlı).
  Out of scope (bu sistemde gerçek zamanlı ortak düzenleme yok).
```

---

## Dosya Paylaşımı

```
Link ile paylaşım:
  POST /share → token üret → URL: https://drive.example.com/s/{token}
  token = cryptographically secure random (32 byte → base64url)
  
  share_links tablosu:
  CREATE TABLE share_links (
    token          VARCHAR(50) PRIMARY KEY,
    file_id        UUID REFERENCES files(file_id),
    owner_id       UUID NOT NULL,
    permission     ENUM('READ', 'WRITE') DEFAULT 'READ',
    expires_at     TIMESTAMPTZ,
    max_downloads  INT,   -- NULL = sınırsız
    download_count INT DEFAULT 0,
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    is_revoked     BOOLEAN DEFAULT FALSE
  );

  Erişim kontrolü:
    token geçerli mi? (revoked değil, süresi dolmadı)
    max_downloads aşıldı mı?
    download_count++

Kullanıcı ile paylaşım (ACL):
  file_permissions tablosu:
  CREATE TABLE file_permissions (
    file_id   UUID REFERENCES files(file_id),
    user_id   UUID NOT NULL,
    role      ENUM('VIEWER', 'COMMENTER', 'EDITOR', 'OWNER'),
    granted_by UUID NOT NULL,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (file_id, user_id)
  );

Klasör paylaşımı → kalıtım:
  Klasör paylaşılırsa → içindeki tüm dosyalar da o izinle paylaşılır.
  İzin kontrolü: klasör hiyerarşisini yürü (parent → root).
  Optimizasyon: her dosyada "effective_permissions" cache (Redis).
  Cache invalidation: klasör izni değişince → tüm alt dosyalar invalidate.

ACL kontrol sırası (her erişimde):
  1. Owner mi? → izin ver.
  2. file_permissions: doğrudan yetki var mı?
  3. Parent klasörlerin izni var mı? (hiyerarşi yürüme).
  4. Share link ile mi? → token kontrol.
  5. Hiçbiri → 403 Forbidden.
```

---

## Versiyon Geçmişi

```
Her dosya güncellemesi yeni bir versiyon:
  v1: [chunk_h1, chunk_h2, chunk_h3]
  v2: [chunk_h1, chunk_h2, chunk_h4]  ← sadece son chunk değişti
  v3: [chunk_h5, chunk_h2, chunk_h4]  ← ilk chunk değişti

Storage tasarrufu:
  chunk_h1, chunk_h2 paylaşılıyor → 1 kopya S3'te.
  Sadece gerçekten değişen chunk'lar ek depolama kullanıyor.
  10 versiyon × 1 GB dosya, %5 değişimle → ~1.5 GB (10 GB değil).

Versiyon saklama politikası (Google Drive benzeri):
  Son 30 gün: tüm versiyonlar → S3 Standard.
  30-90 gün: günlük checkpoint (gün başındaki son versiyon) → S3 IA.
  90+ gün: haftalık checkpoint → S3 Glacier.
  
  Ücretli plan: sınırsız versiyon geçmişi.
  Ücretsiz plan: 30 gün.

Restore:
  Kullanıcı v3'e dönmek istiyor → v3 chunk listesi alınır → indirilir.
  Restore aslında yeni bir versiyon oluşturur (v4 = v3'ün kopyası).
  Neden: geçmiş değişmez (immutable audit log).

file_versions tablosu:
  CREATE TABLE file_versions (
    file_id        UUID REFERENCES files(file_id),
    version_number INT,
    chunk_hashes   TEXT[],     -- sıralı chunk hash listesi
    total_size     BIGINT,
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    created_by     UUID,
    comment        TEXT,        -- "Yanlışlıkla silindi, restore edildi"
    is_deleted     BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (file_id, version_number)
  );
```

---

## Metadata DB Şeması

```sql
CREATE TABLE files (
    file_id         UUID         PRIMARY KEY,
    owner_id        UUID         NOT NULL,
    parent_folder   UUID         REFERENCES files(file_id),  -- klasör hiyerarşisi
    name            VARCHAR(500) NOT NULL,
    size_bytes      BIGINT,
    mime_type       VARCHAR(100),
    file_hash       CHAR(64),    -- SHA-256 of entire file (dedup)
    current_version INT DEFAULT 1,
    is_folder       BOOLEAN DEFAULT FALSE,
    is_deleted      BOOLEAN DEFAULT FALSE,
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),

    CONSTRAINT unique_name_per_folder UNIQUE (owner_id, parent_folder, name)
);

-- Hiyerarşi sorgusunu hızlandır
CREATE INDEX idx_owner_folder ON files (owner_id, parent_folder);
CREATE INDEX idx_file_hash    ON files (file_hash);   -- dedup lookup
CREATE INDEX idx_deleted      ON files (owner_id, is_deleted, deleted_at);

CREATE TABLE chunks (
    chunk_hash    CHAR(64)     PRIMARY KEY,
    s3_key        VARCHAR(500) NOT NULL,   -- s3://bucket/chunks/{chunk_hash}
    size_bytes    INT          NOT NULL,
    ref_count     INT          DEFAULT 1,  -- GC için
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE file_chunks (
    file_id       UUID    REFERENCES files(file_id),
    version       INT,
    chunk_index   INT,
    chunk_hash    CHAR(64) REFERENCES chunks(chunk_hash),
    PRIMARY KEY (file_id, version, chunk_index)
);

CREATE TABLE file_versions (
    file_id        UUID REFERENCES files(file_id),
    version_number INT,
    total_size     BIGINT,
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    created_by     UUID,
    storage_tier   ENUM('STANDARD', 'IA', 'GLACIER') DEFAULT 'STANDARD',
    PRIMARY KEY (file_id, version_number)
);

CREATE TABLE share_links (
    token          VARCHAR(50)  PRIMARY KEY,
    file_id        UUID         REFERENCES files(file_id),
    owner_id       UUID         NOT NULL,
    permission     VARCHAR(10)  DEFAULT 'READ',
    expires_at     TIMESTAMPTZ,
    max_downloads  INT,
    download_count INT          DEFAULT 0,
    is_revoked     BOOLEAN      DEFAULT FALSE,
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE file_permissions (
    file_id    UUID        REFERENCES files(file_id),
    user_id    UUID        NOT NULL,
    role       VARCHAR(20) NOT NULL,  -- VIEWER, EDITOR, OWNER
    granted_by UUID        NOT NULL,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (file_id, user_id)
);

CREATE TABLE user_storage (
    user_id       UUID   PRIMARY KEY,
    used_bytes    BIGINT DEFAULT 0,
    quota_bytes   BIGINT DEFAULT 16106127360,  -- 15 GB
    updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Quota Yönetimi

```
Kullanıcı başına 15 GB ücretsiz depolama:
  user_storage tablosu: used_bytes, quota_bytes.
  Upload öncesi: used_bytes + file_size > quota_bytes → 402 / 413 reddet.

Atomik güncelleme:
  UPDATE user_storage
  SET used_bytes = used_bytes + :delta
  WHERE user_id = :userId AND (used_bytes + :delta) <= quota_bytes
  -- Etkilenen satır = 0 → kota aşıldı.

Redis ile hızlı kota kontrolü:
  INCRBY user:quota:{userId} {file_size}
  Sonuç > quota_limit → geri al, reddet.
  DB ile periyodik sync (DB master kaynak).

Silinmiş dosyalar:
  Çöp kutusu (30 gün) → kullanılan kotadan sayılır (Google Drive gibi).
  Kesin silinince → used_bytes azalır.

Quota uyarıları:
  %80 dolduğunda: e-posta uyarısı.
  %95: uygulama içi banner.
  %100: yalnızca indirme ve silme izni, yükleme reddedilir.
```

---

## Thumbnail & Önizleme Oluşturma

```
Akış (async, upload sonrası):
  1. Upload tamamlandı → Kafka event: FileUploaded {fileId, mimeType, s3Key}.
  2. Thumbnail Worker (consumer):
      image/*  → Pillow/ImageMagick → 200x200, 400x400 thumbnail.
      PDF      → Ghostscript → ilk sayfa → JPEG thumbnail.
      video/*  → FFmpeg → 5. saniye kare → JPEG thumbnail.
      doc/xlsx → LibreOffice headless → PDF'e → thumbnail.
  3. Thumbnail → S3: thumbnails/{fileId}_200x200.webp
  4. Meta güncelle: thumbnail_url = CDN URL.
  5. Client bir sonraki listede thumbnail'ı görür.

Neden async:
  Thumbnail üretimi CPU yoğun → senkron yapılırsa upload yavaşlar.
  Worker sayısı bağımsız ölçeklenir.
  Hata durumunda retry (Kafka consumer group).

WebP formatı:
  JPEG'e göre %30 küçük → CDN bandwidth tasarrufu.
  Önizleme: her zaman WebP (legacy browser'da JPEG fallback).

Önizleme URL güvenliği:
  CDN URL signed (CloudFront signed URL): paylaşılmayan dosya → sadece sahibi görebilir.
  Paylaşılan dosya → public thumbnail URL (token ile).
  TTL: 1 saat (signed URL).
```

---

## Dosya Arama

```
İsim bazlı arama:
  PostgreSQL: ILIKE '%query%' → küçük ölçekte yeterli (milyonlarca dosyada yavaş).
  Elasticsearch: dosya adı, path, mime_type → trigram analizi → hızlı autocomplete.
  Öneri: Elasticsearch ile isim bazlı tam metin arama.

İçerik bazlı arama (full-text):
  Text/PDF/DOCX → Apache Tika → metin çıkar → Elasticsearch'e index.
  Uygulama: "finansal rapor" → içinde bu kelimeler geçen PDF'leri bul.
  
  Akış:
    Upload → Kafka → Content Extractor Worker:
      Apache Tika → mime_type'a göre metin ayıkla.
      Metin → Elasticsearch: POST /file-contents/_doc/{fileId}
      { content: "...", ownerId, fileName, mimeType, createdAt }
  
  Arama:
    GET /search?q=finansal+rapor&userId=123
    → ES: bool query (owner + full-text + isim) → sonuç.

OCR (görsel → metin):
  Taranmış PDF veya JPEG → Tesseract OCR → metin → index.
  Yavaş (5-30 sn/sayfa) → tamamen async, arka planda.
  Desteklenen dil: TR, EN + 100+ dil.

Arama güvenliği:
  Elasticsearch index: her doküman owner_id içerir.
  Sorgu: her zaman owner_id filtreli → başkasının dosyası çıkmaz.
  Paylaşılan dosyalar: ayrı "shared_files" index → paylaşan ile alıcı görür.
```

---

## Şifreleme

```
At-rest şifreleme (S3 tarafında):
  S3 SSE-S3: AWS yönetimli AES-256 → otomatik, şeffaf.
  S3 SSE-KMS: AWS KMS key → key rotasyon, denetim logu.
  Seçim: SSE-KMS → her dosya için ayrı data key (DEK), DEK → KMS ile şifreli.

In-transit şifreleme:
  TLS 1.3: Client ↔ API Gateway, Client ↔ S3 (HTTPS).
  HSTS: başlık ile zorunlu.

Client-side şifreleme (E2EE — opsiyonel, kurumsal):
  Client: dosyayı kendi key'i ile şifreler (AES-256-GCM).
  Şifreli veri → S3'e yüklenir.
  Sunucu düz metin görmez → thumbnail veya full-text arama mümkün değil.
  Trade-off: güvenlik ↑, özellik ↓ (arama, preview yok).

Key yönetimi (E2EE olmadan):
  Master Key (MK): AWS KMS'te saklanır.
  Data Encryption Key (DEK): her dosya için benzersiz → MK ile şifreli → DB'de saklanır.
  Akış: S3'ten al → DEK ile çöz → DEK → KMS ile çöz.
  Key rotation: DEK değişse → sadece şifreli DEK'i DB'de güncelle (S3 verisi değişmez).
```

---

## Çöp Toplama (Garbage Collection)

```
Problem: Dosya silindi → chunk ref_count 0 → ama S3'ten hemen silinmemeli
  Neden: anlık silme → düşük ref_count gözlemlenirse silinecek ama başka transaction kullanıyor olabilir.

Soft delete akışı:
  files tablosunda: is_deleted = TRUE → 30 gün çöp kutusu.
  30 gün doldu → kesin silme işlemi başlat.

Kesin silme (GC job, günlük çalışır):
  1. is_deleted=TRUE ve deleted_at < NOW()-30gün olan dosyaları bul.
  2. Her dosya chunk listesini al.
  3. Her chunk için: ref_count-- (transaction içinde).
  4. ref_count = 0 olan chunk'lar → S3 delete batch.
  5. files, file_chunks, file_versions tablosundan sil.

Orphan chunk tespiti (haftalık):
  S3'teki tüm chunk key'leri listele.
  DB'deki chunks tablosuyla karşılaştır.
  S3'te olup DB'de olmayan → orphan → S3'ten sil.
  Sebep: upload yarıda kesildi → S3'e gitti ama DB'ye kayıt olmadı.
  S3 Lifecycle Rule: 7 gün upload tamamlanmamış multipart → otomatik sil.

Referans sayacı tutarsızlığı:
  Consistency için: ref_count güncelleme ve dosya silme aynı DB transaction'ı.
  İki phase: önce ref_count-- → commit → sonra S3 delete.
  S3 delete başarısız → retry job → ref_count zaten azaltıldı → orphan değil.
```

---

## Bildirim (Sync Notification)

```
10M aktif cihaz × 2 cihaz = 20M WebSocket bağlantısı

WebSocket Server mimarisi:
  Stateful: her WebSocket server bağlı client'ları bilir.
  Horizontal ölçek: 20M / 50K = 400 server (Netty/Node.js).
  Load balancer: IP hash veya sticky session (aynı client = aynı server).

Bildirim akışı:
  Meta Service → Kafka: FileUpdated {userId, fileId, version}
  WebSocket Dispatcher (consumer): userId → hangi WS server? → forward.
  WS Server → client.

Bağlantı kaydı:
  Redis Hash: user_ws_mapping
    HSET "user:123:devices" "deviceId-abc" "ws-server-5"
  WS Dispatcher: HGETALL user:123:devices → hangi server → TCP/gRPC ile ilet.

Fallback (WebSocket yoksa):
  Long Polling: GET /changes?since={timestamp}&userId={id}
  Server: bekle (30 sn) → değişiklik varsa hemen dön.
  Client: döndü → yeniden bağlan.
  Push Notification: mobil offline → FCM/APNs.

Presence & heartbeat:
  Her 30 sn: client → server "hâlâ bağlıyım" ping.
  30 sn gelmezse → bağlantı temizle → Redis'ten sil.
```

---

## Ölçekleme

```
Upload bottleneck:
  Block Service stateless → yatay ölçek (istenen kadar instance).
  S3 presigned URL → client direkt S3'e → Block Service yük almaz.
  S3 Transfer Acceleration: upload → nearest AWS edge → backbone → S3.
  Bölgesel S3 bucket: TR kullanıcısı → eu-central-1 → düşük latency.

Metadata bottleneck:
  PostgreSQL okuma ağırlıklı → read replica (1 master + 3 replica).
  Büyük ölçekte: kullanıcı ID bazlı sharding (Vitess / Citus).
  Sık okunan metadata (folder list) → Redis cache + invalidation.

Sync bildirim:
  20M WebSocket → 400 WebSocket server (Netty, async I/O).
  Redis Pub/Sub → horizontal mesaj yayınlama.
  WebSocket server çöktüğünde → client reconnect + full sync.

Download:
  CDN (CloudFront): en yakın edge'den servis → global.
  Hot dosyalar (shared, popüler): CDN cache hit → %90.
  Cold dosyalar: CDN → S3 → CDN'e cache.
  Signed URL: güvenlik + CDN cache uyumlu.

Search ölçeği:
  Elasticsearch cluster: 10 node, 5 shard, 2 replica.
  Per-user index isolation yerine: tek index + owner_id filter (daha verimli).
```

---

## Olası Sorunlar ve Çözümleri

### 1. Büyük Dosya Upload Kesildi — Kullanıcı Baştan Başlamak Zorunda Kaldı

```
Sorun:
  Kullanıcı 5 GB video yüklüyor, saat 2 sonra bağlantı kesildi.
  Uygulama yeniden başlatılınca → yüklem baştan başladı → 2 saat daha bekledi.
  Kullanıcı şikayeti: "Yükleme sürekli sıfırlanıyor."

Çözüm:
  a) Resumable upload (server-side state):
     Chunk'lar ayrı yüklendiğinden her chunk upload bilgisi S3'te saklanır.
     Upload state: uploadId → hangi chunk'lar tamamlandı → SQLite local DB.
     Uygulama restart: uploadId mevcut → sadece eksik chunk'ları yükle.
     "Kaldığı yerden devam et" butonu → UX'te açıkça göster.

  b) S3 Multipart lifecycle rule:
     Tamamlanmamış multipart upload → 7 gün sonra S3 otomatik temizler.
     7 gün sonra upload devam edilemez → kullanıcıya bildir: "Upload süresi doldu."

  c) Background upload:
     Uygulama minimize edildiğinde → background service olarak devam etsin.
     iOS: background fetch, Android: WorkManager.
     Upload tamamlanınca → bildirim: "Dosyanız yüklendi."

  d) Progress göster:
     Chunk bazlı: %43 tamamlandı, 12 dk kaldı.
     Anlık hız: "Upload hızı: 1.2 MB/sn."
     Motivasyon: kullanıcı iptale daha az meyilli.
```

---

### 2. Sync Storm — 10,000 Kullanıcı Aynı Anda Sync İstedi

```
Sorun:
  Şirket 5,000 çalışanı Google Drive benzeri sisteme geçti.
  Çalışanlar sabah 09:00'da geldi → hepsi sync başlattı.
  API: 50,000 req/sn (her çalışan birden fazla cihaz) → DB connection pool doldu.
  Cevap: timeout → client retry → daha da kötü → cascade failure.

Çözüm:
  a) Jitter ile sync başlatma:
     Client başlatılınca: random(0, 60sn) bekle → sonra ilk sync.
     50K client × uniform dağılım → saniyede ~830 client → yönetilebilir.

  b) Sync frekansı sınırlama:
     "Delta sync" değil: belirli aralıklarla tam kontrol.
     Sık değişen dosyalar: her 30 sn.
     Değişmeyen: her 5 dk.
     Sadece metadata poll (hangi dosya değişti) → hafif istek.

  c) Delta endpoint (DB yükünü azalt):
     GET /changes?since={lastSyncTimestamp}&userId={id}
     Sadece o zamandan beri değişen dosyaları döndür → az veri.
     Cassandra: (user_id, changed_at) partition → hızlı range query.

  d) Event-driven (polling yerine push):
     WebSocket bildirim → sync sadece değişiklik olduğunda.
     Polling yerine: %95 daha az istek.
     WebSocket bağlantısı: lightweight (Netty, async).
```

---

### 3. Dedup Tutarsızlığı — Aynı Chunk İki Kez Yüklendi

```
Sorun:
  User A ve User B aynı anda "ubuntu.iso" yüklüyor.
  İkisi de "bu hash var mı?" sordu → ikisi de "yok" cevabı aldı.
  İkisi de S3'e yükledi → 2 kopya + ref_count karışıklığı.

Neden: race condition — aynı anda iki upload → ikisi de "yok" gördü.

Çözüm:
  a) DB unique constraint:
     chunks(chunk_hash) PRIMARY KEY → duplicate insert → conflict.
     INSERT INTO chunks ... ON CONFLICT (chunk_hash) DO UPDATE SET ref_count = ref_count + 1
     PostgreSQL upsert: atomik → race condition yok.

  b) Distributed lock (Redis):
     SETNX lock:chunk:{hash} "uploading" EX 60
     Lock alan → upload eder → DB'ye yazar → lock serbest.
     Lock alamayan → bekler → tekrar kontrol eder → varsa referans sayar.
     Dezavantaj: Redis SPOF → Sentinel ile gider.

  c) S3 idempotent:
     S3'e aynı key ile iki kez yazılırsa → son yazılan kazanır (overwrite).
     Veri aynı olduğundan sorun yok, ek maliyet: bir kez fazla S3 PUT.
     DB race: upsert ile çöz, S3 race: S3 idempotent garantisi yeterli.
```

---

### 4. Chunk GC Hatalı Silme — Aktif Dosyanın Chunk'ı Silindi

```
Sorun:
  GC job çalıştı: chunk_hash "abc123" ref_count = 0 → S3'ten sildi.
  Aynı anda: başka bir upload bu chunk'ı referans etmeye başladı.
  Yeni dosya: chunk "abc123"e referans var ama S3'te yok → download hata.

Neden: GC ve upload arasında race condition.

Çözüm:
  a) Grace period (en güvenilir):
     Chunk ref_count = 0 olunca hemen silme → "deleted_at" zamanı yaz.
     7 gün sonra GC silsin.
     7 gün içinde yeni referans gelirse: "deleted_at" temizle.
     Risk penceresi çok küçük → pratik çözüm.

  b) Mark-and-sweep:
     Phase 1 (Mark): tüm file_chunks tarama → kullanılan chunk hash seti oluştur.
     Phase 2 (Sweep): chunks tablosu - kullanılan set = orphan → sil.
     İki phase arasında: yeni upload eklenebilir → Phase 1'de olmayan chunk eklenebilir.
     Çözüm: Phase 2'den önce Phase 1'i tekrar çalıştır (çift kontrol).

  c) S3 versioning:
     S3 versioning aktif: "silme" → soft delete (delete marker).
     Gerçek silme: lifecycle rule 30 gün sonra.
     GC hatası → 30 gün içinde recover.
```

---

### 5. Storage Maliyeti Patladı — 225 PB → Beklenti Aşıldı

```
Sorun:
  Hesaplama: 50M × 15GB × %30 kullanım = 225 PB.
  Gerçek: versiyon geçmişi + thumbnail + içerik kopyaları → 400 PB.
  AWS S3 maliyeti: 400 PB × $0.023/GB/ay = ~$9.2M/ay (bütçe: $5M).

Çözüm:
  a) S3 Intelligent Tiering (otomatik, 0 maliyet):
     30 gün erişilmemeyen chunk → IA (Infrequent Access) → %40 ucuz.
     90 gün → Glacier Instant → %68 ucuz.
     Erişilince → anında Standard'a geçer.
     Versiyon geçmişi: 90%+ chunk'ı Glacier'a taşır.

  b) Daha agresif dedup:
     Cross-user dedup şüpheli ama şirket içinde: aynı şirketin çalışanları → güvenli dedup.
     Google Workspace: şirket account'larında cross-user dedup.
     %30 → %50 tasarruf.

  c) Compression (upload öncesi):
     Client-side: dosya compress edilip yüklenir → S3'te sıkıştırılmış.
     Metin dosyaları: %60-80 küçülür.
     Görseller: zaten compress → beklenti düşük.
     Video: zaten H.264/H.265 → yeniden sıkıştırma kötü kalite.

  d) Thumbnail agresif temizle:
     Eski thumbnail'lar: dosya silinince thumbnail da silinsin.
     Versiyon thumbnail: sadece son 5 versiyon thumbnail tut.

  e) Versiyon geçmişi kısıtla:
     Ücretsiz: 30 gün.
     Dosya başına max 100 versiyon.
     Büyük dosyalar (>1 GB): sadece son 5 versiyon.
```

---

### 6. Dosya Arama Yavaş — 500M Dosya ES'i Bunalttı

```
Sorun:
  Kullanıcı sayısı arttı → Elasticsearch 500M doküman.
  "rapor" araması → 3-4 sn cevap → kullanıcı şikayet.
  ES CPU: %90 → neredeyse full.

Çözüm:
  a) Index sharding (kullanıcı bazlı):
     Şu an: tek büyük index → tüm 500M doküman.
     Yeni: kullanıcı ID'ye göre shard routing.
     Arama: sadece ilgili shard'a gider → 500M / N shard yük.

  b) Öne arama sonuçlarını kısıtla:
     İsim bazlı önce (PostgreSQL, hızlı, her zaman çalışır).
     İçerik bazlı sonra (ES, ağır, ayrı butona taşı "içerik ara").
     %80 kullanıcı isim araması yapar → ES yüküne gitmiyor.

  c) ES kaynak yönetimi:
     Her kullanıcı sorgusu: max shard = 5 (tüm index taranmasın).
     Query complexity limit: nested query derinliği 3 ile sınırla.
     Cache: aynı sorgu 5 dk içinde tekrar → ES request cache.

  d) Precomputed indexing:
     İçerik extraction: 24 saat önce (yeni upload, hemen aranmaz genellikle).
     Düşük öncelikli queue → off-peak saatlerde işle.
```

---

### 7. Klasör Paylaşımı Silindi — Alt Dosyalar Erişilemez Kaldı

```
Sorun:
  Alice klasörü Bob ile paylaştı (EDITOR).
  Bob: alt klasördeki dosyayı 3. kişiyle paylaştı.
  Alice: klasör paylaşımını iptal etti.
  Bob artık erişemiyor ama 3. kişiye verilen share link hâlâ çalışıyor.
  3. kişi Alice'in dosyasına Alice'in haberi olmadan erişiyor.

Çözüm:
  a) Cascading revoke:
     Klasör paylaşımı iptal → alt tüm dosyaların doğrudan ve dolaylı izinlerini iptal et.
     Recursive propagation: BFS/DFS ile tüm alt dosya ve klasörleri geç.
     Alt izinler: parent'tan türemişse → sil. Bağımsız verilmişse → koru.

  b) Share link bağımlılığı:
     Share link oluşturulurken: "Bu link hangi izin sayesinde mümkün?" bağı kur.
     İzin iptal → o izne bağlı tüm share link'leri de iptal et (cascade).

  c) Permission inheritance cache:
     Effective permission = dosya izni + tüm parent klasör izinleri.
     Redis cache: "user X, file Y için izni: READ."
     Klasör izni değişince: tüm alt dosya permission cache'lerini invalidate.
     TTL: 5 dk (eventual consistency kabul edilebilir düzeyde).

  d) Audit log:
     Kim kiminle paylaşmış, ne zaman, ne zaman iptal.
     Alice: paylaşım geçmişini görebilir → "Bob 2024-01-15'te 3. kişiyle paylaştı."
     Compliance: GDPR kapsamında veri erişimi denetimi.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Upload yöntemi | Tek parça | Chunk bazlı (4 MB) | Kesintide resume, delta sync, dedup |
| Deduplication | Dosya seviyesi | Chunk seviyesi | Chunk: kısmi değişimde çok daha iyi tasarruf |
| Conflict | Last-write-wins | Çift kopya | Veri kaybı sıfır; kullanıcı karar verir |
| Sync bildirim | Polling (30sn) | WebSocket + push | WebSocket: anlık, daha az istek; polling fallback |
| Metadata DB | MySQL | PostgreSQL | JSONB, array type, upsert ON CONFLICT |
| Büyük metadata | PostgreSQL only | PostgreSQL + Redis | Okuma cache ile P99 < 50ms |
| Şifreleme | SSE-S3 | SSE-KMS | KMS: key rotation, denetim logu, per-file DEK |
| Arama | DB LIKE | Elasticsearch | Full-text, isim + içerik, relevance |
| Thumbnail | Senkron upload | Async Kafka worker | Senkron: upload yavaşlar; async: bağımsız |
| Versiyon storage | Tam kopya her versiyon | Chunk paylaşımı + ref count | %80-90 storage tasarrufu |
| GC | Anlık silme | Grace period (7 gün) | Race condition → aktif chunk silinmez |
| CDN güvenlik | Public URL | Signed URL (1 saat) | Paylaşılmayan dosya → yetkisiz erişim yok |
