# 08e — System Design: Chat Sistemi (WhatsApp)

## Gereksinimler

```
Functional:
  ✓ 1-1 mesajlaşma
  ✓ Grup chat (max 500 üye)
  ✓ Mesaj okundu/iletildi bilgisi (read receipt)
  ✓ Online/son görülme göstergesi
  ✓ Medya gönderme
  ✗ Video call, ses kaydı (out of scope)

Non-Functional:
  DAU: 100M
  Ortalama mesaj: 50/gün/kullanıcı → 5B mesaj/gün
  Latency: mesaj iletimi < 500ms
  Availability: 99.99%
  Mesajlar kalıcı (5 yıl)
```

---

## Capacity Estimation

```
Mesaj QPS: 5B / 86400 ≈ 58,000 mesaj/sn
Peak:      58,000 × 3 ≈ 175,000/sn

Storage:
  Mesaj boyutu: ~100 bytes (text) 
  5B × 100B = 500 GB/gün
  5 yıl: 500GB × 365 × 5 ≈ 913 TB

Medya (%30 mesaj, ortalama 200 KB):
  58,000 × 0.3 × 86400 × 200KB ≈ 300 TB/gün → CDN + S3

WebSocket bağlantıları:
  100M DAU, aynı anda %10 online = 10M eş zamanlı bağlantı
  1 server: ~65,000 max TCP bağlantısı
  10M / 65,000 ≈ 154 chat server
```

---

## Protokol Seçimi: Polling vs WebSocket

```
Short Polling:
  Client → "Yeni mesaj var mı?" → Server → Hayır / [mesajlar]
  Her X saniyede bir
  ✗ Çoğu istek boş → CPU israfı, yüksek latency

Long Polling:
  Client → istek gönderir → Server: mesaj gelene kadar bekler → cevap
  ✓ Az boş istek
  ✗ Sunucu bağlantı havuzu dolar, HTTP/1.1 head-of-line blocking

WebSocket (seçilen):
  Tek TCP bağlantısı → iki yönlü sürekli iletişim
  ✓ Gerçek zamanlı, düşük latency
  ✓ Her mesaj için yeni bağlantı yok
  ✓ Server → Client push (mesaj gelince direkt gönder)
  ✗ Uzun süreli bağlantı → LB affinity (sticky session) gerekir
  ✗ Firewall/proxy sorunları (WebSocket'i bloklayabilir → fallback SSE/polling)
```

---

## High-Level Tasarım

```
   Alice                                               Bob
     │                                                   │
     │ WebSocket                                WebSocket│
     ▼                                                   ▼
┌──────────┐   ┌──────────────────────┐    ┌──────────────┐
│  Chat    │   │   Message Service    │    │   Chat       │
│  Server  │──►│   (Kafka + DB)       │───►│   Server     │
│  (A)     │   └──────────────────────┘    │   (B)        │
└──────────┘           │                   └──────────────┘
     │           ┌─────┴──────┐
     │           │  Cassandra  │ ← mesaj storage
     │           └─────────────┘
     │
┌────┴─────────────────────────────────────┐
│            Service Discovery             │
│  (Alice hangi Chat Server'da? → Zookeeper)
└──────────────────────────────────────────┘
```

---

## Mesaj Gönderme Akışı

```
1. Alice → Chat Server A (WebSocket):
   {type: "message", to: "bob", text: "Merhaba", clientMsgId: "uuid"}

2. Chat Server A:
   a. Mesajı Cassandra'ya kaydet (message_id, from, to, text, status=SENT)
   b. Kafka'ya yayınla: topic = "messages" (partition key = conversation_id)
   c. Alice'e "SENT" ACK gönder

3. Delivery Service (Kafka consumer):
   a. Bob online mı? → Presence Service'e sor
   b. Online: Bob'un Chat Server'ını bul (ZooKeeper/Redis)
      → Chat Server B'ye ilet → Bob'a WebSocket üzerinden push
   c. Offline: Mesajı "pending" olarak işaretle
      → Push notification gönder (APNs/FCM)

4. Bob mesajı aldı:
   → Chat Server B → Delivery Service → "DELIVERED" → Cassandra güncelle
   → Alice'e "DELIVERED" bildir (WebSocket)

5. Bob mesajı okudu:
   → "READ" → Alice'e bildir
```

---

## Offline Mesajlar

```
Bob offline:
  Mesaj Cassandra'da bekler (status=SENT)
  Push notification → Bob'u uyarır (FCM/APNs)

Bob çevrimiçi oldu:
  Bob → Chat Server → "Son görülme" zamanından sonra gelen mesajları çek
  → Cassandra: SELECT * FROM messages WHERE recipient=bob AND created_at > lastSeen
  → Mesajları WebSocket üzerinden Bob'a gönder
  → Bob aldı → "DELIVERED" → göndericilere bildir
```

---

## Presence (Online/Offline) Servisi

```
Online detection:
  WebSocket bağlandı → Presence Service: SET user:alice:status "online" EX 60
  Her 30s heartbeat → EXPIRE user:alice:status 60 (yenile)
  Bağlantı kesildi → key expire → "offline"

Son görülme:
  Bağlantı kapandı → SET user:alice:lastSeen {timestamp}

Online/offline izleme:
  Bob → "Alice online mı?" → Redis GET user:alice:status
  Yayın: Alice online → Alice'in contact'larına WebSocket push
  (Pub/Sub: Alice bağlandı → Redis PUBLISH presence:alice "online")
```

---

## Veri Modeli (Cassandra)

```sql
-- Mesajlar (conversation bazlı partition)
CREATE TABLE messages (
    conversation_id UUID,
    message_id      TIMEUUID,        -- zaman sıralı
    sender_id       UUID,
    text            TEXT,
    media_url       TEXT,
    message_type    TEXT,            -- TEXT, IMAGE, VIDEO, FILE
    status          TEXT,            -- SENT, DELIVERED, READ
    created_at      TIMESTAMP,
    PRIMARY KEY ((conversation_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Kullanıcının konuşma listesi
CREATE TABLE user_conversations (
    user_id         UUID,
    conversation_id UUID,
    last_message_id TIMEUUID,
    last_message_at TIMESTAMP,
    unread_count    INT,
    PRIMARY KEY ((user_id), last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC);

-- Grup üyeliği
CREATE TABLE group_members (
    group_id        UUID,
    user_id         UUID,
    joined_at       TIMESTAMP,
    role            TEXT,  -- ADMIN, MEMBER
    PRIMARY KEY ((group_id), user_id)
);
```

---

## Grup Mesajlaşma

```
Grup mesajı (500 üye):
  Akış 1: Fan-out on write (her üyeye ayrı delivery)
    → Kafka → Delivery Service → 500 üye için paralel delivery
    ✓ Online üyeler anında alır
    ✗ 500 push → kaynak yoğun, büyük gruplarda yavaş

  Akış 2: Fan-out on read (üyeler okuyunca çekerler)
    → Sadece grup inbox'ına yaz
    → Üye conversation'ı açınca çeker
    ✓ Write ucuz
    ✗ Okuma daha yavaş (her seferinde çek)

  Çözüm: Hybrid
    Küçük grup (< 50): Fan-out on write
    Büyük grup (> 50): Fan-out on read + push notification

Mesaj tekili (single copy):
  WhatsApp: Grup mesajı tek kez kaydedilir → her üye okuma sayacı
  (500 kopya değil, 1 mesaj + 500 delivery status)
```

---

## End-to-End Encryption (E2EE)

```
Signal Protocol (WhatsApp kullanır):

Key Exchange:
  Alice ve Bob'un her biri → public/private key çifti üretir
  Public key'ler sunucuda saklanır
  Alice → Bob'un public key'ini çeker → shared secret türetir (Diffie-Hellman)
  Sunucu mesajın içeriğini okuyamaz

Double Ratchet Algorithm:
  Her mesaj için yeni şifreleme anahtarı türetilir (forward secrecy)
  Bir anahtar ele geçirilse → önceki mesajlar deşifre edilemez

Implementasyon özeti:
  Mesaj client'ta şifrelenir → sunucu ciphertext depolar → alıcı client'ta deşifre eder
  Sunucuda plaintext hiçbir zaman görünmez
```

---

## Ölçekleme

```
10M eş zamanlı WebSocket bağlantısı:
  Özel Chat Server'lar (C10K → C1M sorunu)
  Netty/WebFlux (reactive, non-blocking) → 1M bağlantı/server
  10 server yeterli (load balanced, sticky session)

Mesaj throughput (175K/sn):
  Kafka: 10 broker × 30 partition/topic → 300K msg/sn kapasitesi

Message storage:
  Cassandra cluster: 20 node, replication factor 3
  Write: ANY consistency (hız), Read: LOCAL_QUORUM (tutarlılık)

Presence:
  Redis Cluster: 10 node
  100M kullanıcı × 50 bytes = 5 GB → RAM'e sığar
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Protokol | Long polling | WebSocket | WebSocket |
| DB | MySQL | Cassandra | Cassandra (write scale) |
| Grup fan-out | On write | On read | Hybrid |
| Şifreleme | Server-side | E2EE | E2EE (güvenlik) |
