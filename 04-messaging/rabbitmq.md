# 04b — RabbitMQ

## Ne?

AMQP protokolü üzerine kurulu **message broker**. Producer mesajı bir Exchange'e gönderir, Exchange routing kurallarına göre Queue'lara iletir, Consumer Queue'dan okur. Mesaj consume edilince silinir — Kafka gibi log değil, kuyruk semantiği.

---

## Neden?

**Çözdüğü problem:** Servisler arası görev dağıtımı, karmaşık routing, öncelikli kuyruklar, zamanlanmış mesajlar ve geçici task queue ihtiyaçları.

```
Kafka: "Bu event oldu, isteyen dinlesin"
RabbitMQ: "Bu işi biri yapsın" — task queue, RPC, complex routing

Örnek: Sipariş faturası oluşturma
Producer → [invoice-queue] → Worker 1 (meşgul)
                           → Worker 2 (alır, işler)
                           → Worker 3 (meşgul)
```

Erlang ile yazılmıştır; actor model ile yüksek eşzamanlılık sağlar. 2007'de Rabbit Technologies, 2010'da VMware/Pivotal satın aldı.

---

## Nasıl?

### AMQP Modeli

```
Producer → Exchange → Binding → Queue → Consumer
              │
         Routing Logic
         (type'a göre)
```

**Katmanlar:**
```
Connection → TCP bağlantısı (yeniden kullan — pahalı)
Channel    → Connection üzerinde sanal çoklama (lightweight, her thread için ayrı)
Exchange   → Mesajı hangi Queue'lara gönder kararı
Queue      → Mesajların bekletildiği FIFO yapısı
Binding    → Exchange ↔ Queue bağlantısı + routing key/header kuralı
```

---

### Exchange Türleri — Routing Mekanizması

**Direct Exchange:**
```
Exchange: "payments" (type: direct)
Binding:  "payments" → "visa-queue"   (routing key: "visa")
Binding:  "payments" → "paypal-queue" (routing key: "paypal")

Producer: routing_key = "visa"
  → sadece "visa-queue" alır

Kullanım: Kesin eşleşme, belirli worker'a yönlendirme
```

**Fanout Exchange:**
```
Exchange: "events" (type: fanout)
Binding:  "events" → "email-queue"
Binding:  "events" → "sms-queue"
Binding:  "events" → "analytics-queue"

Producer: routing_key önemsiz (yok sayılır)
  → TÜMÜNE kopyalanır

Kullanım: Pub/Sub, broadcast, notification
```

**Topic Exchange:**
```
Exchange: "logs" (type: topic)
Binding:  "logs" → "error-queue"   (routing key: "*.error")
Binding:  "logs" → "app-queue"     (routing key: "app.#")
Binding:  "logs" → "all-queue"     (routing key: "#")

Pattern kuralları:
  *  = tam olarak 1 kelime
  #  = 0 veya daha fazla kelime

routing_key = "app.payment.error"
  → "error-queue":  *.error → 2 kelime gerekir → HAYIR
  → "app-queue":    app.#   → app ile başlar   → EVET
  → "all-queue":    #        → her şey          → EVET

Kullanım: Log routing, event filtreleme, çok boyutlu routing
```

**Headers Exchange:**
```
Routing key yerine message header'ları kullanır

Queue binding: x-match=all, format=pdf, type=invoice
Message headers: format=pdf, type=invoice, customer=abc
  → Eşleşir (x-match=all: tümü uymalı)

Queue binding: x-match=any, priority=high, vip=true
Message headers: priority=high, customer=xyz
  → Eşleşir (x-match=any: biri yeterli)

Kullanım: Çok boyutlu, attribute tabanlı routing (nadiren kullanılır)
```

---

### Queue Türleri

**Classic Queue:**
```
Tek node'da çalışır
Mirroring (deprecated Quorum Queue lehine)
Eski sistemlerde karşılaşılır, yeni projede kullanma
```

**Quorum Queue (Önerilen):**
```
Raft consensus ile çoğunluk (quorum) onayı
  3 node → 2'si yazarsa commit (1 node çökse veri kaybolmaz)
  5 node → 3'ü yazarsa commit (2 node çökse veri kaybolmaz)

Özellikler:
  - Veri güvenliği öncelikli
  - Classic'ten ~%30 yavaş ama güvenilir
  - Exactly-once delivery semantics
  - Poison message handling yerleşik

Production'da her zaman Quorum Queue kullan
```

**Stream Queue:**
```
Kafka benzeri: kalıcı log, non-destructive consume
Consumer mesajı okur ama silinmez
Offset ile istediğin noktadan başla
Birden fazla consumer bağımsız okur

Fark: Classic/Quorum → consume = sil
      Stream        → consume = oku, mesaj kalır

Kullanım: Event replay, çok consumer, Kafka'ya hafif alternatif
```

---

### Message Durability — 3 Katman

```
Mesaj kaybolmasın için 3 şeyin birlikte olması şart:

1. Durable Exchange
   channel.exchangeDeclare("payments", "direct", true); // durable=true

2. Durable Queue
   channel.queueDeclare("invoice-queue", true, false, false, null); // durable=true

3. Persistent Message
   AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
       .deliveryMode(2)  // 2 = persistent (disk'e yaz)
       .build();
   channel.basicPublish("payments", "invoice", props, body);

Bonus: Publisher Confirms
   channel.confirmSelect();
   channel.basicPublish(...);
   channel.waitForConfirmsOrDie(5000); // broker disk'e yazdı mı?
```

---

### Dead Letter Exchange (DLX)

Başarısız mesajların gideceği yer — retry ve hata yönetiminin temeli.

```
Normal akış:
Producer → Exchange → Queue → Consumer (başarılı → ack → mesaj silindi)

Hata durumu:
Consumer nack() / reject() ile requeue=false → DLX devreye girer

DLX akış:
"orders-queue" → işlenemedi → "orders-dlx" → "orders-dlq" → hata işleme consumer

Queue tanımı:
{
  "x-dead-letter-exchange": "orders-dlx",
  "x-dead-letter-routing-key": "failed-order",
  "x-message-ttl": 60000,   // 60sn'de expire → DLX
  "x-max-length": 10000      // max mesaj → oldest → DLX
}
```

**Retry pattern (DLX ile):**
```
orders-queue (TTL: 5000ms)
  → expire → orders-retry-exchange
  → orders-retry-queue (TTL: 30000ms) — 30sn bekle
  → expire → orders-exchange
  → orders-queue  ← tekrar dene

3 retry sonrası:
  → orders-dead-exchange → orders-dead-queue ← insan inceler
```

---

### Consumer Acknowledgement

```java
// AUTO ACK (tehlikeli — asla production'da kullanma)
channel.basicConsume("queue", true, deliverCallback, cancelCallback);
// Mesaj teslimde SİLİNİR. Consumer crash → kayıp.

// MANUAL ACK (doğru yol)
channel.basicConsume("queue", false, (tag, delivery) -> {
    try {
        processMessage(delivery.getBody());
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        //                                                          ↑ false = sadece bu mesaj
        //                                                          true = bu + önceki tüm mesajlar

    } catch (BusinessException ex) {
        // Kalıcı hata → DLX'e gönder (requeue=false)
        channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);

    } catch (TransientException ex) {
        // Geçici hata → queue'ya geri koy (requeue=true)
        // DİKKAT: Sonsuz döngü riski! Retry sayacı tut.
        channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
    }
}, consumerTag -> {});
```

**basicQos — consumer throttling:**
```java
channel.basicQos(10); // maksimum 10 unacked mesaj
// Consumer işlemeden yeni mesaj almaz → backpressure mekanizması
// Çok büyük sayı → bir consumer tüm mesajları alır, diğerleri açken
// Çok küçük sayı → throughput düşer
// İyi başlangıç: prefetchCount = 2-10 (benchmark et)
```

---

### Priority Queue

```java
Map<String, Object> args = new HashMap<>();
args.put("x-max-priority", 10);  // 0-10 arası öncelik
channel.queueDeclare("priority-queue", true, false, false, args);

// Yüksek öncelikli mesaj gönder
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .priority(8)  // 0=düşük, 10=yüksek
    .build();
channel.basicPublish("", "priority-queue", props, body);

// VIP siparişler önce işlenir
```

---

## Ne zaman?

**RabbitMQ kullan:**
```
✓ Task queue — işi worker'lara dağıt
✓ Karmaşık routing (topic/headers exchange)
✓ Öncelikli kuyruklar (priority queue)
✓ Request-Reply / RPC pattern
✓ Delayed/scheduled mesajlar (TTL + DLX trick)
✓ Mesaj geçici — consume edilince gitsin
✓ Küçük/orta ölçek, kolay operasyon
✓ Multi-protocol (AMQP, MQTT, STOMP, HTTP)
```

**RabbitMQ kullanma:**
```
✗ Mesaj replay gerekiyorsa → Kafka (RabbitMQ siler)
✗ 100K+ msg/sn throughput → Kafka
✗ Birden fazla consumer aynı mesajı bağımsız okumalı → Kafka (Stream Queue kısmi çözüm)
✗ Event sourcing / audit log → Kafka
✗ Stream processing → Kafka Streams
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Esnek routing (exchange type'ları) | Kafka'dan düşük throughput |
| Push model → düşük consumer latency | Mesaj replay yok (Streams hariç) |
| Operasyonu görece kolay | Büyük cluster yönetimi karmaşık |
| DLX, priority, TTL yerleşik | Memory alarm → publisher block olur |
| Management UI hazır | Quorum Queue Classic'ten yavaş |
| Multi-protocol support | Partition/sharding manuel yapılır |
| Küçük mesajlar için verimli | Schema evrim yönetimi yok |

---

## Architect Kontrol Listesi

```
□ Production'da Quorum Queue (Classic değil)
□ Durable exchange + queue + persistent message (3'ü birlikte)
□ Publisher Confirms aktif (mesaj kaybı önlenir)
□ DLX her queue için tanımlı (başarısız mesajlar kaybolmaz)
□ basicQos/prefetch ayarı (consumer throttling)
□ requeue=true kullanırken retry sayacı var mı? (sonsuz döngü riski)
□ Memory/disk alarms için alerting (publisher block'u önce gör)
□ Shovel/Federation (multi-DC veya migration için)
□ Management plugin + monitoring (queue depth, consumer count)
□ Connection pool — connection'ı yeniden kullan, her request'te açma
```
