# 08e — System Design: Chat Sistemi (WhatsApp)

## Gereksinimler

```
Functional:
  ✓ 1-1 mesajlaşma
  ✓ Grup chat (max 500 üye)
  ✓ Mesaj okundu/iletildi bilgisi (read receipt)
  ✓ Online/son görülme göstergesi
  ✓ Medya gönderme (resim, video, dosya)
  ✓ Multi-device sync (aynı hesap, birden fazla cihaz)
  ✓ Mesaj arama
  ✗ Video call, ses kaydı (out of scope)

Non-Functional:
  DAU: 100M
  Ortalama mesaj: 50/gün/kullanıcı → 5B mesaj/gün
  Latency: mesaj iletimi < 500ms
  Availability: 99.99%
  Mesajlar kalıcı (5 yıl)
  E2EE: sunucu mesaj içeriğini okuyamaz
```

---

## Capacity Estimation

```
Mesaj QPS:
  5B / 86400 ≈ 58,000 mesaj/sn ortalama
  Peak: 58,000 × 3 ≈ 175,000/sn

Storage:
  Mesaj boyutu: ~100 bytes (text + metadata)
  5B × 100B = 500 GB/gün
  5 yıl: 500GB × 365 × 5 ≈ 913 TB → Cassandra + tiered storage

Medya (%30 mesaj, ortalama 200 KB):
  5B × 0.3 × 200 KB = 300 TB/gün → S3 + CDN
  Deduplicate: aynı dosya (hash ile) → tek kopya → gerçek 30-50 TB/gün

WebSocket bağlantıları:
  100M DAU, aynı anda %10 online = 10M eş zamanlı bağlantı
  Netty/WebFlux (non-blocking I/O): 1 server → 100K-500K bağlantı
  10M / 300K = ~34 Chat Server (overhead ile 50 server)

Presence Redis:
  100M kullanıcı × 50 bytes = 5 GB → tek Redis Cluster'a sığar

Kafka throughput:
  175,000 msg/sn × 500 bytes = 87.5 MB/sn → 10 broker rahatlıkla taşır
```

---

## Protokol Seçimi: Polling vs WebSocket vs SSE

```
Short Polling:
  Client → "Yeni mesaj var mı?" → Server her X saniyede yanıt.
  ✗ %95 istek boş → CPU israfı. Latency: polling aralığı kadar.

Long Polling:
  Client istek gönderir → server mesaj gelene kadar tutar → yanıtlar.
  ✓ Boş istek azaldı.
  ✗ HTTP/1.1: her yanıt sonrası yeni bağlantı. Head-of-line blocking.
  ✗ Bağlantı havuzu tükebilir (binlerce bekleyen istek).

SSE (Server-Sent Events):
  Server → Client tek yönlü stream. HTTP/2 üzerinde çalışır.
  ✓ Firewall/proxy uyumlu (HTTP).
  ✗ Client → Server için ayrı HTTP POST gerekir (gerçek iki yönlü değil).
  Kullanım: bildirim sistemleri, feed güncellemeleri.

WebSocket (seçilen):
  Tek TCP bağlantısı → iki yönlü sürekli iletişim.
  HTTP Upgrade handshake → ws:// veya wss://.
  ✓ Gerçek zamanlı, düşük latency (<50ms).
  ✓ Her mesaj için yeni bağlantı yok.
  ✓ Server → Client push.
  ✗ LB: sticky session gerekir (bağlantı durum bilgili).
  ✗ Bazı corporate proxy WebSocket'i bloklar → SSE fallback.
  ✗ Reconnect yönetimi uygulamaya bırakılmış.

Fallback zinciri:
  WebSocket → SSE → Long Polling
  (auto-detect: bağlantı başarısız → bir sonrakini dene)
```

---

## High-Level Tasarım

```
   Alice (mobil)                                    Bob (mobil)
        │                                                │
        │ wss://chat-1.myapp.com                        │ wss://chat-7.myapp.com
        ▼                                                ▼
┌───────────────┐    ┌─────────────────┐    ┌───────────────────┐
│  Chat Server 1│    │  Message Service │    │  Chat Server 7    │
│  (Netty)      │───►│  (Kafka + logic) │───►│  (Netty)          │
└───────────────┘    └────────┬────────┘    └───────────────────┘
        │                     │
        │             ┌───────┴────────┐
        │             │   Cassandra    │ ← mesaj storage
        │             └────────────────┘
        │
┌───────┴──────────────────────────────────────┐
│            Chat Server Registry               │
│  "Alice → Chat Server 1" (Redis Hash)         │
│  "Bob   → Chat Server 7" (Redis Hash)         │
└───────────────────────────────────────────────┘
        │
┌───────┴──────────────────────────────────────┐
│  Supporting Services                          │
│  Presence Service  → Redis (online/offline)   │
│  Notification Svc  → FCM / APNs               │
│  Media Service     → S3 + CDN                 │
│  Search Service    → Elasticsearch            │
└───────────────────────────────────────────────┘
```

---

## Chat Server Routing — Kullanıcı Hangi Sunucuda?

```
Problem: Alice mesaj gönderiyor, Bob'un hangi Chat Server'da olduğu bilinmeli.

Çözüm: Redis Hash — session registry

  Kullanıcı bağlandığında:
    HSET user_sessions bob "chat-server-7:node-id-xyz"
    EXPIRE user_sessions:bob 3600  (1 saat — heartbeat ile yenilenir)

  Mesaj yönlendirme (Delivery Service):
    target = HGET user_sessions bob
    → "chat-server-7"
    → Chat Server 7'ye internal HTTP/gRPC: "Bob'a bu mesajı ilet"
    → Chat Server 7 → Bob'un WebSocket bağlantısına push

  Bob offline (key yok veya TTL doldu):
    → Mesajı Cassandra'ya "PENDING" olarak yaz
    → Push notification: FCM (Android) / APNs (iOS)

  Multi-device:
    HSET user_sessions:bob "phone" "chat-server-7"
    HSET user_sessions:bob "laptop" "chat-server-3"
    → Delivery: her iki cihaza ilet

  Chat Server crash:
    Redis key'leri TTL ile expire → 60 sn sonra "offline" kabul.
    Bağlantısı kopan kullanıcı: reconnect → yeni Chat Server'a → registry güncellendi.
```

---

## Mesaj Gönderme Akışı (Detaylı)

```
1. Alice → Chat Server 1 (WebSocket):
   {
     type: "message",
     clientMsgId: "c-uuid-1234",   ← client üretir, idempotency için
     to: "bob-uuid",
     conversationId: "conv-uuid",
     text: "Merhaba",              ← E2EE ile şifrelenmiş ciphertext
     timestamp: 1719000000000
   }

2. Chat Server 1 → Cassandra: mesajı kaydet
   INSERT INTO messages (conversation_id, message_id, sender_id, text, status)
   VALUES (conv-uuid, now(), alice-uuid, ciphertext, 'SENT')
   → message_id = TIMEUUID (zaman sıralı, sunucu üretir)

3. Chat Server 1 → Kafka: "messages" topic'ine publish
   partition key = conversationId → aynı konuşma aynı partition → sıra garantisi

4. Chat Server 1 → Alice: ACK
   {type: "ack", clientMsgId: "c-uuid-1234", serverMsgId: "timeuuid-xyz", status: "SENT"}
   Alice: clientMsgId → serverMsgId eşleştir → mesaj gönderildi ✓

5. Delivery Service (Kafka consumer):
   Bob online mı? → HGET user_sessions:bob
   ├── Online (chat-server-7):
   │     → gRPC: ChatServer7.deliver(message)
   │     → Bob'a WebSocket push
   │     → Bob ACK → Cassandra: status = DELIVERED
   │     → Alice'e WebSocket: {type: "delivery", msgId: "xyz", status: "DELIVERED"}
   └── Offline:
         → Cassandra: status = PENDING (zaten SENT durumunda)
         → FCM/APNs: push notification (şifreli, içerik görünmez)
         → "1 yeni mesaj" bildirimi

6. Bob mesajı okudu:
   Bob → Chat Server 7: {type: "read", conversationId: "conv-uuid", upToMsgId: "timeuuid-xyz"}
   → Cassandra: status = READ
   → Alice'e: {type: "read_receipt", conversationId: "conv-uuid", readBy: "bob", upTo: "timeuuid-xyz"}
```

---

## Mesaj Sıralama Garantisi

```
Problem: Alice hızlı arka arkaya mesaj gönderdi.
  M1 (t=100ms) ve M2 (t=110ms) → ağ koşulları → M2 önce ulaştı?

Çözüm 1 — TIMEUUID (Cassandra native):
  Cassandra TIMEUUID: 60-bit timestamp + 64-bit unique node ID.
  Sunucu üretir → nanosaniye hassasiyeti → monoton artan.
  Aynı milisaniyede: node ID ile ayrıştırılır → sıra korunur.
  PRIMARY KEY ((conversation_id), message_id) + CLUSTERING ORDER BY DESC
  → Cassandra verileri fiziksel olarak sıralı depolar.

Çözüm 2 — Sequence number per conversation:
  Redis: INCR "seq:conv-uuid" → 1, 2, 3 ...
  Her mesaja sequence ekle.
  Client: seq eksikse → sunucudan o mesajı iste (gap detection).

Çözüm 3 — Kafka partition sıralaması:
  partition key = conversationId → aynı konuşma tek partition → FIFO.
  Delivery Service: partition içi sıra korunur.
  Farklı partition → farklı konuşma → sıra birbirini etkilemez.

Client-side ordering:
  Client: serverMsgId (TIMEUUID) ile sırala.
  clientMsgId: geçici (ağda) → serverMsgId gelince replace et.
  Optimistic UI: mesaj anında göster (pending) → ACK gelince confirm.
```

---

## Bağlantı Kopma & Reconnect Stratejisi

```
Senaryo: Alice tünel girdi → WebSocket bağlantısı kesildi.

Heartbeat mekanizması:
  Client → Server: her 30 saniyede ping frame.
  Server → Client: pong frame.
  Server: 60 sn içinde ping gelmedi → bağlantıyı kapat → registry temizle.

Client reconnect (Exponential Backoff):
  Bağlantı kesildi → 1sn bekle → dene.
  Başarısız → 2sn → dene.
  Başarısız → 4sn → dene.
  Max: 30sn. Jitter ekle: random(0, 1) × backoff → thundering herd önleme.

  // Client (JavaScript)
  function reconnect(attempt) {
    const delay = Math.min(1000 * Math.pow(2, attempt), 30000);
    const jitter = Math.random() * 1000;
    setTimeout(() => connectWebSocket(), delay + jitter);
  }

Reconnect sonrası mesaj senkronizasyonu:
  Client → Server: {type: "sync", lastSeenMsgId: "timeuuid-xyz"}
  Server → Cassandra:
    SELECT * FROM messages
    WHERE conversation_id IN (alice_conversations)
      AND message_id > lastSeenMsgId
    LIMIT 100
  → Eksik mesajları push et → client UI güncelle.

Chat Server değiştiğinde:
  Alice yeni Chat Server'a bağlandı (load balancer farklı yönlendirdi).
  Yeni server: HSET user_sessions:alice "phone" "chat-server-5"
  Eski server: session temizlendi (TTL expire veya disconnect event).
  Teslimat: Delivery Service → yeni server'a ilet → sorunsuz.
```

---

## Multi-Device Sync

```
Senaryo: Alice hem telefonda hem laptop'ta aktif.

Registry:
  HSET user_sessions:alice "phone"  "chat-server-1"
  HSET user_sessions:alice "laptop" "chat-server-3"

Mesaj teslimi:
  Delivery Service: tüm alice device'larına ilet.
  Phone ve laptop'a aynı mesaj → her ikisinde görünür.

Okundu durumu senkronizasyonu:
  Alice laptop'ta mesajı okudu → READ gönderdi.
  Server: phone'a da "bu mesaj okundu" push et → phone'daki unread badge temizlendi.

  Device-specific vs account-level read:
    WhatsApp: herhangi bir cihazda okunursa → READ.
    Telegram: cihaz bazlı (phone okusa laptop hâlâ unread gösterebilir).
    Seçim: account-level → kullanıcı dostu ama karmaşık sync.

Cassandra'da device tracking:
  ALTER TABLE messages ADD delivered_to SET<UUID>;  -- device ID set
  Tüm device'lar DELIVERED → mesaj DELIVERED kabul.

Yeni cihaz eklendi (initial sync):
  Alice yeni telefon kurdu → son 30 günlük mesaj tarihi.
  Cassandra: son 30 gün, her konuşma için son 100 mesaj.
  Büyük tarih → on-demand load (scroll up yaptıkça).
```

---

## Offline Mesajlar & Push Notification

```
Bob offline:
  Mesaj Cassandra'da status=SENT/PENDING.
  Push notification → Bob'u uyarır.

FCM (Android) / APNs (iOS) entegrasyonu:
  Notification Service: Bob offline → token al → push gönder.
  E2EE nedeniyle: bildirim içeriği şifreli veya "1 yeni mesaj" (içerik yok).
  WhatsApp: "Mesajlarınız uçtan uca şifreli" → içerik bildirimde görünmez.

  // Notification payload
  {
    "to": "fcm-device-token-bob",
    "data": {
      "conversationId": "conv-uuid",
      "senderId": "alice-uuid",
      "badge": 3  // unread count
    }
    // body yok (E2EE)
  }

Bob çevrimiçi oldu:
  1. WebSocket bağlandı → Chat Server'a register.
  2. Sync isteği: {lastSeenAt: "2026-06-24T10:00:00Z"}
  3. Server → Cassandra:
     SELECT * FROM messages
     WHERE conversation_id IN (bob_conversations)
       AND created_at > lastSeenAt
     ORDER BY message_id ASC
     LIMIT 500  -- çok mesaj varsa sayfalı
  4. Mesajlar Bob'a push.
  5. Bob ACK → status = DELIVERED → göndericilere bildir.

Push quota yönetimi:
  FCM: saniyede 1M push → kolayca aşılabilir.
  Batch: aynı kullanıcıya 5 mesaj → 5 ayrı push değil, 1 badge push.
  Cooling: son 5 dk içinde zaten push gönderildiyse → tekrar gönderme.
```

---

## Medya Gönderme Akışı

```
Doğrudan Chat Server üzerinden medya göndermek:
  ✗ Chat Server bellek/CPU yoğun → mesaj latency'i etkiler.
  ✗ 300 TB/gün → Chat Server'da saklama imkansız.

Pre-signed URL yaklaşımı (AWS S3):

1. Alice: "Fotoğraf göndermek istiyorum" (dosya seçti)
   → Media Service: POST /media/upload-url
   → {uploadUrl: "https://s3.../presigned?...", mediaId: "m-uuid"}

2. Alice → S3: PUT {uploadUrl} + fotoğraf (direkt, Chat Server bypass)
   Avantaj: Chat Server yük almaz, S3 bandwidth'i kullanılır.

3. Alice yükleme tamamlandı → Chat Server'a mesaj:
   {type: "message", mediaId: "m-uuid", mediaType: "IMAGE", text: "Bak!"}

4. Chat Server → Message Service → Delivery (normal mesaj akışı)

5. Bob mesajı aldı:
   → Media Service: GET /media/m-uuid → pre-signed download URL (5 dk geçerli)
   → CDN üzerinden indir (S3 direkt değil → CDN cache → hız + maliyet)

Medya deduplication:
  Alice gönderdi: SHA-256(file) = "abc123"
  Mehmet aynı fotoğrafı gönderiyor: SHA-256 = "abc123" → zaten S3'te!
  Media Service: hash → S3 key map → yeni upload yok → aynı mediaId dön.
  S3 tasarrufu: %40-60 gerçek fotoğraflarda deduplicate.

CDN stratejisi:
  CloudFront / Akamai: S3 önünde.
  İlk istek: CDN miss → S3'ten çek → CDN cache (popüler medya).
  Sonraki istekler: CDN hit → düşük latency, S3 maliyet yok.
  TTL: 7 gün (medya değişmez, immutable URL).
  Private medya: pre-signed URL → CDN imzalı → yetkisiz erişim yok.

Thumbnail:
  Upload sonrası: Lambda trigger → thumbnail üret (200×200px).
  Mesaj önizlemesi: thumbnail (küçük) → tam çözünürlük istenince indir.
```

---

## Presence (Online/Offline) Servisi

```
Online detection:
  WebSocket bağlandı → Presence Service:
    SET "presence:alice" "online" EX 60
  Her 30sn heartbeat → EXPIRE "presence:alice" 60  (yenile)
  Bağlantı kesildi → TTL expire → 60sn sonra "offline"
  Immediate offline: disconnect event → DEL "presence:alice"

Son görülme:
  Disconnect → SET "lastSeen:alice" {timestamp}
  Privacy ayarı:
    "Herkese göster" → normal.
    "Sadece kontaklarıma" → Redis'te flag → sadece contact ise dön.
    "Kimseye gösterme" → hep "son görülme bilinmiyor".

Presence yayımı (Publisher/Subscriber):
  Alice online oldu → Redis PUBLISH "presence-events" "alice:online"
  Alice'in kontaklarının Chat Server'ları: SUBSCRIBE "presence-events"
  Bob'un Chat Server'ı: mesajı alır → Bob'a WebSocket push:
    {type: "presence", user: "alice", status: "online"}

Ölçek sorunu (100M kullanıcı):
  Her kullanıcı → kontaklarının presence'ını subscribe.
  Ortalama 200 contact × 100M = 20B subscription → Redis Cluster ile ölçekle.
  Optimizasyon: yalnızca aktif conversation'daki kullanıcıların presence'ı.
  (Sadece açık konuşma ekranındaki kullanıcıyı izle — tüm kontakları değil)
```

---

## Veri Modeli (Cassandra)

```sql
-- Mesajlar (conversation bazlı partition)
CREATE TABLE messages (
    conversation_id UUID,
    message_id      TIMEUUID,        -- sunucu üretir, zaman sıralı, globally unique
    sender_id       UUID,
    text            TEXT,            -- E2EE ciphertext
    media_id        UUID,            -- medya varsa
    message_type    TEXT,            -- TEXT, IMAGE, VIDEO, FILE, AUDIO
    status          TEXT,            -- SENT, DELIVERED, READ
    delivered_to    SET<UUID>,       -- hangi device'lara teslim edildi
    read_by         SET<UUID>,       -- grup: kimler okudu
    client_msg_id   TEXT,            -- idempotency (client UUID)
    created_at      TIMESTAMP,
    PRIMARY KEY ((conversation_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND gc_grace_seconds = 864000;     -- 10 gün: tombstone temizleme

-- Kullanıcının konuşma listesi (inbox)
CREATE TABLE user_conversations (
    user_id         UUID,
    conversation_id UUID,
    last_message_id TIMEUUID,
    last_message_at TIMESTAMP,
    last_message_preview TEXT,       -- şifreli snippet (thumbnail)
    unread_count    COUNTER,
    is_muted        BOOLEAN,
    PRIMARY KEY ((user_id), last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC);

-- Konuşma metadata
CREATE TABLE conversations (
    conversation_id UUID PRIMARY KEY,
    type            TEXT,            -- DIRECT, GROUP
    name            TEXT,            -- grup adı (1-1 için null)
    created_at      TIMESTAMP,
    created_by      UUID
);

-- Grup üyeliği
CREATE TABLE group_members (
    group_id        UUID,
    user_id         UUID,
    joined_at       TIMESTAMP,
    role            TEXT,            -- ADMIN, MEMBER
    last_read_msg   TIMEUUID,        -- bu üye buraya kadar okudu
    PRIMARY KEY ((group_id), user_id)
);

-- Device token (push notification)
CREATE TABLE user_devices (
    user_id     UUID,
    device_id   UUID,
    platform    TEXT,               -- IOS, ANDROID, WEB
    push_token  TEXT,               -- FCM / APNs token
    last_active TIMESTAMP,
    PRIMARY KEY ((user_id), device_id)
);
```

---

## Grup Mesajlaşma

```
Grup mesajı (500 üye):

Fan-out on Write (küçük grup < 50 üye):
  Kafka → Delivery Service → 50 üyeye paralel WebSocket push.
  Online üyeler anında alır.
  ✓ Düşük okuma latency.
  ✗ 500 üyede 500 delivery → yavaş + kaynak yoğun.

Fan-out on Read (büyük grup > 50 üye):
  Kafka → sadece grup mesaj tablosuna yaz.
  Üye konuşmayı açınca → sunucudan çeker.
  ✓ Write ucuz ve hızlı.
  ✗ Okuma gecikmeli → "yeni mesaj var" bildirimi gerekli.

Hybrid (seçilen):
  Küçük grup (< 50): Fan-out on Write.
  Büyük grup (≥ 50): Fan-out on Read + push notification.

Mesaj tekili (single copy):
  Grup mesajı: 1 kez Cassandra'ya yaz.
  500 ayrı kopya değil → 1 mesaj + 500 delivery status.
  group_members.last_read_msg: per-member okuma pozisyonu.

Grup read receipt:
  Mesajı 50 üye okudu → "50/500 okudu" gösterimi.
  Cassandra: read_by SET<UUID> → SET boyutu izle.
  Performans: büyük grupta SET → 500 UUID → 8KB veri.
  Alternatif: ayrı tablo:
    CREATE TABLE message_reads (
        message_id  TIMEUUID,
        user_id     UUID,
        read_at     TIMESTAMP,
        PRIMARY KEY ((message_id), user_id)
    );
  Okuma sayısı: SELECT COUNT(*) → pahalı.
  Counter column veya materialized summary tercih edilir.
```

---

## End-to-End Encryption (E2EE)

```
Signal Protocol (WhatsApp kullanır):

Key Exchange (X3DH — Extended Triple Diffie-Hellman):
  Alice ve Bob, uygulama kurulumunda:
    Identity Key (IK): uzun vadeli key pair
    Signed Prekey (SPK): orta vadeli (haftalık rotation)
    One-time Prekeys (OPK): tek kullanımlık key listesi

  Server: public key'leri saklar (private key'ler cihazda).
  Alice → Bob'a mesaj göndermek istiyor:
    Bob'un public key'lerini sunucudan çek.
    X3DH: 4 Diffie-Hellman işlemi → shared secret.
    Shared secret → AES-256-GCM session key.

Double Ratchet Algorithm:
  Her mesaj → yeni şifreleme anahtarı türetir.
  Forward secrecy: bir anahtar ele geçirilse → önceki mesajlar açılamaz.
  Break-in recovery: gelecek mesajlar da korumalı (anahtar senkronize edilince).

Sunucu ne görür:
  from, to, timestamp, message_size → metadata.
  Mesaj içeriği → ciphertext (opaque byte array).
  Sunucu plaintext hiç görmez.

Metadata koruması (opsiyonel):
  Tor/onion routing ile kimden-kime bilgisi de gizlenebilir.
  Signal App: Sealed Sender → alıcı bile göndereni bilmeyebilir (advanced).

Grup E2EE:
  Sender Key: gönderen bir grup session key üretir.
  Grup üyelerine: kendi E2EE kanallarından sender key'i ilet.
  Sonraki mesajlar: tek şifreleme → her üye sender key ile çözer.
  (500 üyeye 500 ayrı şifreleme değil → 1 şifreleme + 500 key dağıtımı)
```

---

## Mesaj Arama

```
Problem: Cassandra partition-içi arama → conversationId bilmeden arama yok.
  "Geçen ay 'toplantı' yazdım, hangi konuşmada?" → Cassandra karşılayamaz.

Çözüm: Elasticsearch:
  Her mesaj Kafka'dan tüketilir → ES'e index.
  Index: user_id (erişim kontrolü), conversation_id, text, timestamp.
  Arama: kullanıcı sadece kendi mesajlarında arama yapabilir.

  Mapping:
  {
    "mappings": {
      "properties": {
        "conversation_id": {"type": "keyword"},
        "sender_id":       {"type": "keyword"},
        "participants":    {"type": "keyword"},  // kim erişebilir
        "text":            {"type": "text", "analyzer": "turkish"},
        "timestamp":       {"type": "date"},
        "message_type":    {"type": "keyword"}
      }
    }
  }

  Sorgu:
  GET /messages/_search
  {
    "query": {
      "bool": {
        "must": [{"match": {"text": "toplantı"}}],
        "filter": [{"term": {"participants": "alice-uuid"}}]  // sadece kendi mesajları
      }
    },
    "sort": [{"timestamp": "desc"}],
    "size": 20
  }

E2EE ile arama sorunu:
  Mesajlar şifreli → ES plaintext indexleyemez.
  Çözümler:
  a) Client-side search: mesajlar cihazda local SQLite → cihaz içi arama.
     WhatsApp: bu yöntemi kullanır (sunucu arama yok).
  b) Keyword index (sunucu tarafı, E2EE kırılır): sunucu şifreli mesajları indexler.
     Güvenlik vs özellik trade-off → çoğu E2EE app → client-side tercih eder.
  c) Private Set Intersection (PSI): kriptografik protokol → sunucu arama.
     Araştırma aşamasında, henüz pratikte yaygın değil.
```

---

## Rate Limiting & Spam Önleme

```
Sorunlar:
  Spam bot: 1 kullanıcı → 10,000 mesaj/dk → diğer kullanıcıları rahatsız.
  Broadcast: 1 mesaj → 10,000 konuşmaya gönderme (politika ihlali).
  Media flood: çok büyük dosya veya çok sık upload → S3 maliyet.

Rate limiting (Redis + Sliding Window):
  Mesaj: kullanıcı başına max 60 mesaj/dakika.
    Key: "rate:msg:{userId}:{minute}"
    INCR → > 60 → 429 Too Many Requests.
  Media upload: 10 dosya/dakika/kullanıcı.
  Grup mesajı: 1 grup → max 20 mesaj/saniye (bot önleme).

Anti-spam:
  Content hash: aynı mesaj kısa sürede çok kez → spam şüphesi.
    SHA256(text) → SET içinde var mı? → "duplicate broadcast" flag.
  Recipient limit: 1 mesaj → max 5 farklı konuşmaya forward.
  New account: ilk 24 saat → günde max 20 yabancıya mesaj.

Account score:
  Şikayet alındı → score düş.
  Uzun süreli hesap + normal davranış → score yüksek.
  Düşük score → daha kısıtlı rate limit.
  Çok düşük score → hesap geçici askıya al → human review.

Media scanning (E2EE olmayan durumlarda):
  Şifreli içerik → scan yapılamaz (E2EE).
  Client-side PhotoDNA hash: bilinen illegal içerik → hash eşleşme → report.
  Gönderilmeden önce client'ta hash kontrol → sunucuya şikayet (içerik değil hash).
```

---

## Ölçekleme

```
10M eş zamanlı WebSocket bağlantısı:
  Netty / WebFlux (non-blocking I/O): event loop → thread başına 10K+ bağlantı.
  50 Chat Server × 200K bağlantı/server = 10M.
  K8s: HPA → bağlantı sayısına göre scale.
  Load Balancer: sticky session (IP hash veya cookie) → aynı kullanıcı aynı server.

Kafka throughput:
  175K msg/sn: 10 broker × 30 partition = 300 partition → rahatça karşılar.
  Replication factor: 3 (HA).
  Consumer group per service: delivery, aggregation, search indexing.

Cassandra cluster:
  20 node, replication factor 3, consistency:
    Write: ANY → en hızlı (lider bile yoksa kabul eder) → < 1ms.
    Read: LOCAL_QUORUM → 2/3 node onayı → tutarlılık + hız dengesi.
  Partition boyutu: conversation başına max 100K mesaj → ~10 MB → sağlıklı.
  TTL: eski mesajlar → TWCS (TimeWindowCompactionStrategy) → otomatik sil.

Presence:
  Redis Cluster: 10 node.
  100M × 50 bytes = 5 GB → tek node'a sığar, Cluster HA için.

Horizontal scale özeti:
  Chat Server:   50 node (WebSocket)    → stateless değil, sticky LB
  Delivery Svc:  20 node (Kafka consumer) → stateless → kolayca scale
  Presence:      10 Redis node            → shard by userId
  Cassandra:     20 node                  → shard by conversation_id
  Elasticsearch: 10 node (3 shard × 3 rep) → search index
```

---

## Olası Sorunlar ve Çözümleri

### 1. Chat Server Crash — Aktif Bağlantılar Koptu, Mesajlar Kayboldu

```
Sorun:
  Chat Server 1 crash → 300K kullanıcının WebSocket bağlantısı koptu.
  Bu sunucu aracılığıyla iletilmekte olan mesajlar → yarıda kesildi.
  Redis'te session kaydı hâlâ "chat-server-1" gösteriyor.
  Delivery Service → chat-server-1'e ilet → bağlantı hatası → mesaj kayboldu.

Çözüm:
  a) Session TTL + heartbeat:
     Chat Server: her 30sn → Redis session key'ini yenile.
     Crash → heartbeat durdu → 60sn → TTL expire → Bob "offline" kabul edildi.
     Delivery Service: "offline" → Cassandra'ya PENDING yaz → push notification.
     Kaybedilen mesaj yok, gecikme: max 60sn.

  b) Client reconnect:
     Bağlantı koptu → exponential backoff → yeni Chat Server'a bağlan.
     Yeni sunucu → Redis'e register → Delivery Service doğru yönlendirir.
     Reconnect sync: lastSeenMsgId → eksik mesajları çek.

  c) Graceful shutdown:
     Chat Server stop sinyali aldı (SIGTERM) → yeni bağlantı kabul etme.
     Mevcut bağlantılar: "Yeniden bağlanın" mesajı → client reconnect başlasın.
     K8s terminationGracePeriodSeconds: 60sn → client'lar yönlendirilebilir.

  d) Health check:
     K8s liveness probe: WebSocket server ayakta mı?
     Başarısız → pod restart → session TTL expire → trafiği başka sunucuya.
```

---

### 2. Thundering Herd — Grup Mesajı 500 Üyeyi Aynı Anda Uyandırdı

```
Sorun:
  500 üyeli grup: gece 03:00'da aktif değil.
  Sabah 09:00'da: moderatör mesaj attı.
  500 push notification → 500 kullanıcı uyandı → aynı anda Chat Server'a bağlandı.
  Delivery Service: 500 mesaj aynı anda → Redis + Cassandra → spike.
  500 reconnect → LB → tüm Chat Server'lar → connection spike.

Çözüm:
  a) Staggered push notification:
     500 push → rastgele 0-5 sn gecikme ile gönder.
     Kullanıcılar farklı zamanlarda uyanır → bağlantı spike dağılır.

  b) Fan-out throttling:
     Delivery Service: büyük grup → max 50 teslim/sn.
     Token bucket: 500 teslim → 10 sn'de tamamlanır → spike yok.
     Kabul edilebilir gecikme: 500 ms vs sistem stabilitesi.

  c) Grup message queue per recipient:
     Her kullanıcı için inbox queue → consumer hızına göre tüket.
     Backpressure: yüklü sunucu → yavaş tüket → doğal throttle.

  d) Circuit breaker (Delivery Service → Chat Server):
     Chat Server aşırı yüklendi → circuit OPEN → mesajları Cassandra'da beklet.
     Yük azalınca → circuit CLOSED → teslim devam.
```

---

### 3. Message Storm — Bir Kullanıcı Saniyede Binlerce Mesaj Gönderdi

```
Sorun:
  Bot hesabı veya hatalı client: saniyede 5,000 mesaj → grup konuşmasına.
  Kafka: mesajları kabul etti → topic doldu.
  Delivery Service: 500 üye × 5,000 msg/sn = 2.5M teslim/sn → çöktü.
  Diğer kullanıcıların mesajları gecikti → sistem degredasyonu.

Çözüm:
  a) API Gateway rate limiting (ilk savunma hattı):
     WebSocket frame rate: max 60 frame/dakika/bağlantı.
     Aşım → 429 → bağlantıyı kes.
     Nginx / Envoy: WebSocket message rate limiting.

  b) Chat Server seviyesinde token bucket:
     Her WebSocket bağlantısı → 1 token/sn üretir, max 60 token.
     Mesaj gönderilince 1 token harcanır.
     Token yoksa → mesaj drop + client'a uyarı.

  c) Otomatik hesap askıya alma:
     Son 1 dk > 100 mesaj → "şüpheli aktivite" flag.
     Bot detection: mesaj içeriği tekrar ediyor, insanüstü hız.
     Hesap geçici kısıtlama → human review queue.

  d) Kafka topic quota:
     producer.quotas: kullanıcı başına max 1 MB/sn write.
     Aşımda: producer throttle (delay) → otomatik yavaşlatma.
```

---

### 4. Presence Yanlış Gösterdi — Kullanıcı Online Görünüyor Ama Değil

```
Sorun:
  Alice'in telefonu uçak moduna geçti → WebSocket bağlantısı aniden kesildi.
  TCP: RST paketi gönderelemedi (ağ kopuk) → server bağlantıyı hemen bilmiyor.
  Server: Alice hâlâ "online" gösteriyor → 60sn TTL dolana kadar.
  Bob: "Alice online, neden cevap vermiyor?" → kötü UX.

Çözüm:
  a) Agresif heartbeat timeout (60sn → 15sn):
     Client: her 10sn ping.
     Server: 15sn ping gelmedi → offline kabul.
     Tradeoff: daha sık heartbeat → daha fazla ağ trafiği (küçük, kabul edilebilir).

  b) TCP keepalive:
     SO_KEEPALIVE: TCP seviyesinde bağlantı sağlığı kontrolü.
     İşletim sistemi: boşta bağlantıya probe gönderir → cevap yoksa → kapat.
     Server: bağlantı kapatıldı → disconnect event → presence güncelle.

  c) Presence "probably online":
     "Online" yerine "Az önce aktifti (2 dk önce)" gösterimi.
     Kesin online değil, son aktivite zamanı → daha dürüst UX.

  d) Presence privacy:
     Kullanıcı ayarı: "Online durumumu gösterme."
     → Presence hesaplanır ama paylaşılmaz → UX sorunu yok.
```

---

### 5. Read Receipt Fırtınası — Büyük Grup Cassandra'yı Dövdü

```
Sorun:
  500 üyeli grup: popüler mesaj.
  500 üye mesajı okudu → 500 ayrı Cassandra UPDATE (status=READ, read_by ekleme).
  Aynı satıra 500 concurrent write → Cassandra hot row → write contention.
  Latency spike: 500 UPDATE → sıralanmak zorunda (LWT veya seri).

Çözüm:
  a) Counter tablosu yerine append-only:
     Her read → yeni satır (UPDATE değil INSERT):
     message_reads(message_id, user_id, read_at) → her biri farklı satır.
     Cassandra: append → hızlı. Read count: SELECT COUNT → ayrı.

  b) Read receipt batching:
     Client: her mesajı tek tek "read" göndermez.
     Batch: "conv-uuid içinde timeuuid-xyz'ye kadar okudum."
     Server: tek UPDATE → unread mesajların hepsini işaretle.
     500 satır yerine 1 satır güncelleme.

  c) Read receipt async (Kafka):
     READ eventi → Kafka → consumer → Cassandra batch write.
     Spike absorb: Kafka buffer → Cassandra'ya kontrollü yazma.
     Tradeoff: read receipt gecikebilir (< 2sn) → kabul edilebilir.

  d) Approximate count (grup için):
     Büyük grupta "500/500 okudu" yerine "100+ okudu" göster.
     HyperLogLog (Redis): yaklaşık unique count → düşük bellek + hızlı.
     Kesin count: sadece küçük grup için tam hesapla.
```

---

### 6. Cassandra Hot Partition — Popüler Grup Konuşması Tek Node'u Dövüyor

```
Sorun:
  1M üyeli süper grup (Telegram benzeri kanal).
  conversation_id = partition key → tüm yazma tek partition → tek node → bottleneck.
  Mesaj hızı: 10,000/dk → o node'un write kapasitesi aşıldı → lag + timeout.

Çözüm:
  a) Partition key bucket'lama:
     conversation_id + toDate(message_id) → günlük bucket.
     conversation_id + (message_id.hashCode() % 10) → 10 sub-partition.
     Tek node → 10 node'a dağılım.
     Dezavantaj: sıralı okuma için tüm bucket'lardan merge.

  b) Conversation sharding:
     Büyük konuşma → "large conversation" flag.
     conversation_id yerine: conversation_shard_id (0-9).
     Mesaj shard = (sequence_number % 10).
     Okuma: 10 shard → paralel sorgu → merge → sıralı.

  c) Materialized view / Summary table:
     Son 1000 mesaj: ayrı summary table → hot path için.
     Eski mesajlar: ana tablo → scroll up ile lazy load.

  d) Kafka Streams pre-aggregation:
     Mesajları hemen Cassandra'ya yazmak yerine:
     Batch: 100ms → 100 mesajı birden yazma.
     Cassandra: batch insert → daha verimli.
```

---

### 7. Multi-Device Sync Tutarsızlığı — Telefonda Okundu, Laptop'ta Hâlâ Okunmadı

```
Sorun:
  Alice telefonda mesajı okudu → READ gönderdi.
  Laptop'ta: hâlâ "2 okunmamış" gösteriyor.
  Sync mesajı laptop'a ulaşmadı → bağlantı sorunu.

  Daha kötü senaryo:
  Alice telefonu: 5 mesaj → okudu → READ ACK gönderdi → Kafka'ya gitti.
  Alice laptop'u o sırada offline → READ eventi kaçırdı.
  Laptop online olunca: sync isteği → server "son sync'ten bu yana" değişiklik gönder.
  READ eventi Kafka'da artık yok (retention doldu) → laptop senkronize olamadı.

Çözüm:
  a) Conversation state snapshot (Cassandra):
     user_conversations tablosu: last_read_msg per device değil, per account.
     Alice telefonda okudu → last_read_msg = timeuuid-xyz.
     Laptop online → user_conversations'ı çek → last_read_msg = timeuuid-xyz.
     Laptop: bu message_id'ye kadar okundu olarak işaretle.

  b) Device sync topic (Kafka):
     "sync:alice" topic → Alice'in tüm device'larına gidecek eventler.
     READ eventi → sync topic'e de yaz.
     Laptop online → sync topic'ten son N eventi replay → güncel duruma gel.
     Retention: 30 gün → 30 gün offline kalan cihaz bile senkronize olabilir.

  c) Tombstone mesaj:
     Server: Alice READ gönderdi → laptop'a özel sync mesajı sakla.
     Laptop bağlanınca: bekleyen sync mesajları → gönder.
     Cassandra: device_sync(device_id, event_id, event_data) → TTL 30 gün.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Protokol | Long Polling | WebSocket | Gerçek zamanlı, çift yönlü; 10ms'de mesaj push. Long polling: her yanıt sonrası yeni bağlantı |
| DB | MySQL | Cassandra | Write-heavy (58K msg/sn); Cassandra: append-only, linearly scalable. MySQL: write bottleneck |
| Mesaj sırası | Client timestamp | Server TIMEUUID | Client saati güvenilmez (manipüle edilebilir); TIMEUUID: monoton, globally unique |
| Grup fan-out | Sadece on-write | Hybrid (< 50: write, ≥ 50: read) | 500 üye → 500 push → yavaş. Hybrid: küçük grup hızlı, büyük grup ölçeklenebilir |
| Şifreleme | Server-side | E2EE (Signal Protocol) | Sunucu mesajları okuyamaz; ihlal durumunda content leak yok |
| Medya | Chat Server üzerinden | Pre-signed URL + S3 | Chat Server bypass → latency sıfır; S3: petabyte ölçek |
| Arama | Cassandra scan | Elasticsearch | Cassandra: partition dışı arama yok; ES: full-text, Turkish analyzer |
| Presence timeout | 300sn | 60sn heartbeat | 300sn → kullanıcı 5dk yanlış "online" görünür; 60sn: daha doğru ama %50 fazla heartbeat |
| Read receipt (grup) | Her üye ayrı UPDATE | Append-only + batch | Cassandra hot row; append: yüksek concurrency, batch: I/O azaltır |
| Offline sync | Kafka replay | Cassandra snapshot + device sync topic | Kafka retention sınırlı; Cassandra snapshot: her zaman doğru son durum |
