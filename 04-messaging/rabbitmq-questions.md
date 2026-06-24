# 04b — RabbitMQ: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Mesaj Dayanıklılığı (Durability)

### Gerçek Hayat Sorunları

---

**Sorun 1: RabbitMQ yeniden başladı — tüm mesajlar kayboldu**

```java
// Senaryo: Bakım için RabbitMQ node restart edildi.
// Restart sonrası: queue boş, işlenmemiş 50K fatura mesajı kayıp.

// YANLIŞ: Non-durable queue + transient mesaj
channel.queueDeclare("invoices", false, false, false, null);
//                                ^^^^^ durable=false → restart'ta silinir

channel.basicPublish("", "invoices", null, body);
//                                   ^^^^ properties=null → deliveryMode=1 (transient)

// Durability 3 katmanlıdır — hepsi bir arada olmalı:

// DOĞRU 1: Durable exchange
channel.exchangeDeclare("billing", "direct", true);
//                                           ^^^^^ durable=true

// DOĞRU 2: Durable queue
channel.queueDeclare("invoices",
    true,  // durable → restart'ta hayatta kalır
    false, // exclusive
    false, // autoDelete
    null);

// DOĞRU 3: Persistent mesaj
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .deliveryMode(2)  // 1=transient (RAM), 2=persistent (disk)
    .build();
channel.basicPublish("billing", "invoice", props, body);

// BONUS: Publisher confirms — broker disk'e yazdı mı?
channel.confirmSelect();
channel.basicPublish("billing", "invoice", props, body);
boolean acked = channel.waitForConfirms(5000);
if (!acked) {
    log.error("Broker mesajı onaylamadı! Retry gerekiyor.");
    // retry logic veya dead letter
}
```

---

**Sorun 2: Publisher confirms yok — mesajlar sessizce kayboldu**

```
Senaryo:
  Payment servisi ödeme onayını "payment-complete" queue'ya gönderiyor.
  basicPublish() çağrısı başarılı dönüyor ama mesaj bazen ulaşmıyor.
  
  Neden:
    basicPublish() — fire-and-forget. Broker'a gönderildi ama:
    a) Queue dolu → overflow policy: drop → mesaj düşürüldü.
    b) Broker disk I/O hata → persistent write başarısız.
    c) Network paketi kayboldu.
    basicPublish() bunların hiçbirini bilmez → her zaman "başarılı" gibi davranır.
  
  Sonuç: Ödeme onayı işlenemedi → sipariş "beklemede" kaldı.

ÇÖZÜM A: Publisher Confirms (async, yüksek throughput)
  channel.confirmSelect();
  
  channel.addConfirmListener(
    (deliveryTag, multiple) -> log.info("ACK: {}", deliveryTag),   // broker kabul etti
    (deliveryTag, multiple) -> {                                    // broker reddetti
        log.error("NACK: {} — retry!", deliveryTag);
        retryMessage(deliveryTag);
    }
  );
  
  // Gönder — async devam et
  channel.basicPublish("payments", "complete", props, body);

ÇÖZÜM B: waitForConfirmsOrDie (sync, basit ama blocking)
  channel.confirmSelect();
  channel.basicPublish("payments", "complete", props, body);
  channel.waitForConfirmsOrDie(5000); // 5sn içinde ACK gelmezse exception

ÇÖZÜM C: Outbox Pattern
  DB'ye yaz (transaction) + outbox tablosu → scheduler Rabbit'e gönder
  → en güçlü garanti, ek altyapı gerektirir
```

---

### Mülakat Soruları — Durability

**Junior / Mid:**

1. Mesaj dayanıklılığı için hangi üç şeyin birlikte sağlanması gerekir?

   > **Beklened:** (1) Durable exchange: `exchangeDeclare(..., durable=true)` → restart'ta exchange hayatta kalır. (2) Durable queue: `queueDeclare(..., durable=true)` → queue ve içindeki mesajlar restart'ta korunur. (3) Persistent mesaj: `deliveryMode=2` → mesaj RAM'de değil disk'e yazılır. Birini unutursan: exchange/queue restart'ta kaybolursa mesaj da gider; mesaj transient ise queue kalsa bile RAM'deydi → yeniden başlatmada kaybolur. Bonus: Publisher Confirms → broker disk'e yazdığını onaylar.

2. Publisher Confirms nedir? Ne zaman şarttır?

   > **Beklened:** basicPublish() fire-and-forget'tir — gönderildi diye broker'a ulaştı veya disk'e yazıldı anlamına gelmez. Publisher Confirms: `channel.confirmSelect()` → broker mesajı kabul (ACK) veya reddettiğinde (NACK) bildirim. Şart: finansal işlem, sipariş, envanter güncellemesi — mesaj kaybı kabul edilemez. Sync: `waitForConfirmsOrDie()` — basit ama blocking. Async: `addConfirmListener()` — yüksek throughput. Outbox pattern: DB transaction + outbox → en güçlü garanti, Confirms olmadan da kullanılabilir.

---

**Senior / Architect:**

3. Classic Queue, Quorum Queue ve Stream Queue arasındaki farklar nelerdir? Hangisi ne zaman?

   > **Beklened:** Classic Queue: single node, eski mirroring (deprecated). Yeni projede kullanma. Quorum Queue: Raft consensus, çoğunluk yazarsa commit. 3 node → 1 çöksün veri kalır. Production standardı. %30 daha yavaş ama güvenilir. Poison message handling built-in. Stream Queue: Kafka benzeri — consume = silme değil. Offset tabanlı okuma, birden fazla consumer bağımsız. Replay mümkün. Ne zaman: task queue, work dispatch → Quorum. Fan-out, replay, Kafka hafif alternatif → Stream. Eski kod → Classic (ama migrate et). Karar: veri güvenliği kritikse → Quorum. Çok consumer, replay → Stream.

---

## Bölüm 2: Consumer Acknowledgement & Prefetch

### Gerçek Hayat Sorunları

---

**Sorun 3: Auto ACK açık — consumer crash sonrası veri kayboldu**

```java
// Senaryo: Fatura oluşturma servisi mesajı alır almaz ACK'liyor.
// İşlem ortasında crash → fatura oluşturulmadı ama mesaj silindi.

// YANLIŞ: auto-ack = true
channel.basicConsume("invoices",
    true,  // autoAck — mesaj teslimde anında silinir
    (tag, delivery) -> {
        generateInvoice(delivery.getBody()); // crash burada → mesaj zaten silindi!
    },
    consumerTag -> {}
);

// Belirti:
// "invoices queue boş ama fatura oluşmadı" → kayıp.

// DOĞRU: Manual ACK
channel.basicConsume("invoices",
    false, // autoAck=false → mesaj işlenene kadar unacked kalır
    (tag, delivery) -> {
        try {
            generateInvoice(delivery.getBody());
            // Başarılı → ack et → mesaj silinir
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);

        } catch (PermanentException e) {
            // Kalıcı hata (bad data) → DLX'e gönder, yeniden deneme
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
            // requeue=false → DLX devreye girer

        } catch (TransientException e) {
            // Geçici hata (DB timeout) → queue'ya geri koy
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
            // requeue=true — DİKKAT: aşağıda sonsuz döngü riskine bak
        }
    },
    consumerTag -> {}
);
```

---

**Sorun 4: `requeue=true` ile sonsuz döngü — queue doldu, sistem çöktü**

```java
// Senaryo: Bağımlı bir servis birkaç saatliğine down oldu.
// Geçici hata → requeue=true → mesaj geri döndü → tekrar alındı → tekrar fail.
// Binlerce mesaj saniyede defalarca dönüp durdu → CPU %100, queue flood.

// YANLIŞ: Sonsuz retry
} catch (Exception e) {
    channel.basicNack(tag, false, true); // her zaman requeue → sonsuz döngü
}

// DOĞRU: Retry sayacı ile sınırlı deneme
} catch (Exception e) {
    int retryCount = getRetryCount(delivery); // header'dan oku
    if (retryCount < 3) {
        // Header'a retry sayacı ekleyerek geri koy
        Map<String, Object> headers = new HashMap<>(delivery.getProperties().getHeaders());
        headers.put("x-retry-count", retryCount + 1);
        AMQP.BasicProperties retryProps = delivery.getProperties().builder()
            .headers(headers).build();
        // Önce nack (requeue=false), sonra yeniden publish
        channel.basicNack(tag, false, false);
        channel.basicPublish("", "invoices", retryProps, delivery.getBody());
    } else {
        // Max retry → DLX'e gönder
        channel.basicNack(tag, false, false);
    }
}

// DAHA İYİ: DLX + TTL retry pattern (Spring AMQP @RetryableTopic benzeri)
// invoices-queue → fail → invoices-wait-queue (TTL: 30sn, no consumer)
//               → expire → invoices-retry-exchange → invoices-queue
// Her geçişte header'a sayaç ekle, N kez sonra dead-queue'ya gönder
```

---

**Sorun 5: `basicQos` ayarı yok — tek consumer tüm mesajları aldı**

```java
// Senaryo: 3 consumer, 10K mesaj.
// Consumer-1 tüm mesajları aldı (prefetch sınırı yok).
// Consumer-2 ve Consumer-3 boşta bekledi.
// Consumer-1 yavaş işledi → mesajlar gecikti.

// Neden: basicQos(prefetchCount) ayarlanmadı.
// RabbitMQ: consumer'ın kanalı boştaysa mümkün olduğunca mesaj iter.
// Consumer-1 ilk bağlanan → round-robin öncesi queue doluysa tümünü alabilir.

// YANLIŞ: Sınırsız prefetch
// (basicQos çağrılmadı)

// DOĞRU: Prefetch sınırla
channel.basicQos(10); // max 10 unacked mesaj al, onları işle, sonra yeni al
// Her consumer en fazla 10 mesaj tutar → 3 consumer dengeli dağılır
// Consumer-1 10 işledi → ack → 10 daha alır
// Bu sürede Consumer-2 ve Consumer-3 da 10'ar mesaj alır

// Yanlış değer:
// basicQos(1): Her mesaj için network round-trip → throughput düşer
// basicQos(1000): Tekrar tek consumer sorunu
// İyi başlangıç: 10-50, yük testinde optimize et

// Spring AMQP:
// spring.rabbitmq.listener.simple.prefetch=20
```

---

### Mülakat Soruları — ACK & Prefetch

**Junior / Mid:**

4. `basicAck`, `basicNack`, `basicReject` farkları nelerdir?

   > **Beklened:** basicAck: mesaj başarıyla işlendi → RabbitMQ'dan sil. `multiple=true` → bu delivery tag dahil önceki unacked tümünü ack et. basicNack: mesaj işlenemedi. `requeue=true` → queue'ya geri koy (tekrar dene). `requeue=false` → DLX varsa DLX'e gönder, yoksa sil. `multiple=true` → toplu nack. basicReject: basicNack'in single message versiyonu (`multiple` yok). requeue=true/false aynı mantık. Kural: kalıcı hata (parse hatası, bad data) → requeue=false + DLX. Geçici hata (DB timeout) → requeue=true ama retry sayacı zorunlu.

5. `basicQos` / prefetchCount neden önemlidir? Yanlış ayarlanınca ne olur?

   > **Beklened:** Prefetch: consumer'ın aynı anda kaç unacked mesaj tutabileceği. Sınırsız (0): RabbitMQ tüm mesajları iter → bir consumer hepsini alır → diğerleri boşta → load imbalance. Çok küçük (1): Her mesaj için ack bek → network round-trip → düşük throughput. İdeal: işleme süresine göre. Hızlı işlem (1ms) → prefetch=50-100. Yavaş işlem (1sn) → prefetch=5-10. Spring: `spring.rabbitmq.listener.simple.prefetch=20`. basicQos aynı zamanda backpressure: consumer yavaşlarsa RabbitMQ daha fazla mesaj itmez → queue'da birikir → producer yavaşlar (memory alarm → publish block).

---

**Senior / Architect:**

6. Retry mekanizmasını DLX ile nasıl tasarlarsın? `requeue=true`'nun tehlikesi nedir?

   > **Beklened:** requeue=true tehlikesi: bağımlı servis down → mesaj geri dön → anında retry → geri dön → döngü. CPU %100, queue flood, diğer mesajlar işlenemiyor. Doğru retry: DLX + TTL wait queue. `invoices-queue` → hata → nack(requeue=false) → DLX → `invoices-wait` (TTL=30sn, consumer yok) → expire → `invoices-exchange` → `invoices-queue`. Her geçişte header'a sayaç. 3 denemeden sonra → `invoices-dead-queue` → alarm + insan müdahalesi. Avantajlar: exponential backoff (her retry bekleme artabilir), döngü yok, monitoring kolay (dead queue depth → alert).

---

## Bölüm 3: Exchange Routing

### Gerçek Hayat Sorunları

---

**Sorun 6: Topic exchange pattern yanlış — mesaj hiçbir queue'ya gitmedi**

```
Senaryo:
  Log routing sistemi: topic exchange.
  Binding: "*.error" → error-queue (tüm uygulamaların hataları)
  Mesaj gönderildi: routing_key = "app.payment.error"
  
  Sonuç: error-queue'ya ulaşmadı → kayıp.

Neden:
  "*.error" pattern'ı: * = TAM OLARAK 1 kelime
  "app.payment.error" = 3 kelime ("app", "payment", "error")
  "*.error" = 1 kelime + ".error" → yani "app.error" eşleşir ama "app.payment.error" EŞLEŞMEZ

Çözüm:
  "#.error" kullan — # = 0 veya daha fazla kelime
  "#.error" → "app.error" ✓, "app.payment.error" ✓, "x.y.z.error" ✓
  
  Binding örnekleri ve davranışları:
  "app.#"      → "app.payment.error" ✓  "app" ✓  "app.x.y.z" ✓
  "*.*.error"  → "app.payment.error" ✓  "x.y.error" ✓  "app.error" ✗
  "#"          → her şey ✓ (fanout gibi davranır)
  "app.*.error"→ "app.payment.error" ✓  "app.x.error" ✓  "app.x.y.error" ✗

Test: Her binding'i rabbitmq yönetim UI'ında "publish message" ile doğrula.
Monitoring: Publish rate > route rate → bazı mesajlar routing'e girmiyor alarm.
```

---

**Sorun 7: Fanout exchange — yeni servis bağlanmadan önce gönderilen mesajlar kayboldu**

```
Senaryo:
  "order-events" fanout exchange → email-queue, sms-queue bağlı.
  Yeni gereksinim: analytics-service de bu event'leri dinlesin.
  Analytics-service deploy edildi, analytics-queue oluşturuldu, exchange'e bağlandı.
  
  Sorun: Bağlantıdan önce gönderilmiş tüm order event'leri analytics-service'e gitmedi.
  Fanout: o an bağlı queue'lara gönderir — sonradan bağlanan queue geçmişi göremez.

RabbitMQ mesaj semantiği:
  "Şu an bağlı ve binding'i olan queue'ya gönder."
  Geçmişi replay etme özelliği YOK (Stream Queue hariç).

Seçenekler:
  1. Geçmişi kabul et: analytics-service sadece bağlandıktan sonrasını işler.
     Tarihsel veri için ayrı kaynak (PostgreSQL, data warehouse).
     
  2. Stream Queue kullan:
     "order-events-stream" → Stream Queue → consumer offset ile istediği yerden başlar.
     Yeni servis: offset=0 → tüm geçmişi okur.
     Trade-off: mesajlar hiç silinmez → storage yönetimi gerekir.
     
  3. Kafka ile entegrasyon:
     Fanout → geçmiş de lazım → Kafka consumer group semantiği daha uygun.
     Her consumer group bağımsız offset tutar, geçmişi replay edebilir.

Mimari ders:
  "Yeni servis eklenince geçmişe ihtiyaç var mı?" → Evet → Kafka / Stream Queue.
  "Sadece şimdiden itibaren yeterli" → RabbitMQ Fanout.
```

---

### Mülakat Soruları — Exchange & Routing

**Junior / Mid:**

7. Dört exchange tipini açıkla. Her biri için gerçek bir kullanım örneği ver.

   > **Beklened:** Direct: routing key tam eşleşme. Örnek: ödeme tipine göre yönlendirme — "visa" → visa-queue, "paypal" → paypal-queue. Fanout: tüm bağlı queue'lara kopyala, routing key yok sayılır. Örnek: sipariş event'ini email + sms + analytics'e aynı anda gönder. Topic: wildcard pattern (* = 1 kelime, # = N kelime). Örnek: log routing — "app.payment.error" → "#.error" bağlı queue'ya. Headers: mesaj header'larına göre routing. Örnek: format=pdf AND type=invoice olan mesajları PDF-işleme kuyruğuna gönder. Nadiren kullanılır (topic daha yaygın).

8. RabbitMQ ile Kafka'nın temel farkı nedir? "Bu işi biri yapsın" ve "Bu event oldu" ayrımını açıkla.

   > **Beklened:** RabbitMQ: task queue semantiği. Mesaj bir consumer tarafından alınır, işlenir, silinir. "Bu faturayı oluştur" — tek bir worker yapmalı. Karmaşık routing, öncelik, TTL. Kafka: event log semantiği. Mesaj silinmez, birden fazla consumer group bağımsız okur. "Sipariş verildi" — inventory, email, analytics hepsi ayrı ayrı okur, geçmişe gidebilir. Pratik: task distribution, RPC, routing karmaşıksa → RabbitMQ. Fan-out + replay + yüksek throughput → Kafka. Her ikisi de ekibinde varsa: iş dağıtımı → RabbitMQ, event streaming → Kafka.

---

**Senior / Architect:**

9. Headers exchange yerine topic exchange neden genellikle tercih edilir?

   > **Beklened:** Topic exchange: routing key string pattern, lightweight, performanslı. Çoğu use case için yeterli. Headers exchange: mesaj header map'i — binary (byte array header), type-safe matching. `x-match=all` (AND) veya `x-match=any` (OR). Avantajı: routing key'e sığmayan çok boyutlu kriter. Dezavantajı: topic'ten yavaş (header map parse), string olmayan değerler. Kullanım: format=pdf + type=invoice + region=EU gibi üç boyutlu koşul → headers. Bunların hepsi routing key'e kodlanabiliyorsa (örn: "EU.pdf.invoice") → topic tercih et. Gerçek hayatta headers exchange nadiren gerekir.

---

## Bölüm 4: Dead Letter Exchange & Retry

### Gerçek Hayat Sorunları

---

**Sorun 8: DLX tanımlanmadı — başarısız mesajlar kayboldu**

```java
// Senaryo: Ödeme servisi bazen down oluyor.
// İşlenemeyen mesaj nack edildi, requeue=false → mesaj silindi.
// "Ödeme mesajları nereye gitti?" → hiçbir yere!

// YANLIŞ: DLX olmadan nack
channel.basicNack(deliveryTag, false, false);
// DLX yok → mesaj direkt silinir → iz yok, recovery yok

// DOĞRU: DLX tanımlı queue
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "payments-dlx");     // başarısız → bu exchange'e
args.put("x-dead-letter-routing-key", "failed-payment"); // bu routing key ile
args.put("x-message-ttl", 30000);                       // 30sn işlenmezse → DLX
args.put("x-max-length", 50000);                        // max 50K mesaj → eskisi DLX'e

channel.queueDeclare("payments", true, false, false, args);

// DLX exchange ve queue oluştur
channel.exchangeDeclare("payments-dlx", "direct", true);
channel.queueDeclare("payments-dlq", true, false, false, null);
channel.queueBind("payments-dlq", "payments-dlx", "failed-payment");

// Şimdi nack(requeue=false) → payments-dlq'ya düşer
// DLQ consumer: alert gönder, insan inceler, gerekirse retry

// Spring AMQP:
@Bean
Queue paymentsQueue() {
    return QueueBuilder.durable("payments")
        .withArgument("x-dead-letter-exchange", "payments-dlx")
        .withArgument("x-dead-letter-routing-key", "failed-payment")
        .withArgument("x-message-ttl", 30000)
        .build();
}

@RabbitListener(queues = "payments-dlq")
void handleDeadLetter(Message message) {
    alertService.notify("Dead letter: " + new String(message.getBody()));
    // Monitoring → PagerDuty, Slack alert
}
```

---

**Sorun 9: TTL + DLX ile zamanlanmış mesaj — gecikmiş bildirim sistemi**

```java
// Kullanım senaryosu: Siparişi 30dk sonra hatırlatma email'i gönder.
// RabbitMQ delayed message plugin yoksa TTL+DLX trick:

// Adım 1: "Bekletme" queue'su — TTL=30dk, consumer YOK, DLX=notification-exchange
Map<String, Object> waitArgs = new HashMap<>();
waitArgs.put("x-message-ttl", 1800000);        // 30 dakika
waitArgs.put("x-dead-letter-exchange", "notification-exchange");
waitArgs.put("x-dead-letter-routing-key", "reminder");
// Consumer yok → mesaj birikir → TTL dolunca DLX'e gider

channel.queueDeclare("order-reminder-wait", true, false, false, waitArgs);

// Adım 2: Gerçek notification queue
channel.queueDeclare("notifications", true, false, false, null);
channel.queueBind("notifications", "notification-exchange", "reminder");

// Adım 3: Producer — mesajı bekletme queue'ya gönder
producer.send("", "order-reminder-wait", props, reminderBody);
// 30dk sonra → TTL → notification-exchange → notifications → consumer işler

// SORUN: Farklı TTL gerekliyse farklı wait queue gerekir.
// Çözüm: rabbitmq-delayed-message-exchange plugin → per-message gecikme desteği
// Production: plugin daha temiz ama ek bağımlılık.
```

---

### Mülakat Soruları — DLX & Retry

**Junior / Mid:**

10. DLX (Dead Letter Exchange) nedir? Hangi durumlarda tetiklenir?

    > **Beklened:** DLX: başarısız mesajların gittiği exchange. Tetikleyiciler: (1) `basicNack/basicReject` ile `requeue=false`. (2) Mesajın TTL'i doldu (x-message-ttl veya per-message expiration). (3) Queue max uzunluğuna ulaştı (x-max-length) → en eski mesaj DLX'e. Queue tanımında `x-dead-letter-exchange` ve opsiyonel `x-dead-letter-routing-key` set edilir. DLQ (Dead Letter Queue): DLX'e bağlı queue — başarısız mesajlar burada birikir. Monitoring: DLQ derinliği → alert. İnsan/operatör inceler, düzeltilmiş mesajları yeniden gönderir.

11. `x-message-ttl` ile `expiration` (per-message) farkı nedir?

    > **Beklened:** `x-message-ttl` (queue argument): tüm queue için geçerli, tüm mesajlara uygulanır. `expiration` (per-message property): sadece o mesaj için — queue TTL'den kısa olursa geçerli. Her ikisi de mevcutsa: daha kısa olan kazanır. DLX kombinasyonu: TTL dolunca mesaj DLX'e gönderilir. Kullanım: queue TTL → genel kural. Per-message TTL → bazı mesajlar daha kısa süre önemliyse (flash sale, OTP kodu). Dikkat: per-message TTL, sıradaki mesajın önünde bekliyorsa önden silinmez — sıra beklenir (kısıtlama).

---

**Senior / Architect:**

12. RabbitMQ ile request-reply (RPC) pattern nasıl implemente edilir? Sorunları neler?

    > **Beklened:** Pattern: producer mesaj gönderir + `reply_to` header'ına geçici queue adı + `correlation_id` koyar. Consumer işler + cevabı `reply_to` queue'ya correlation_id ile gönderir. Producer `reply_to` queue'yu dinler, correlation_id ile eşleştirir. Sorunlar: (1) Consumer down → cevap gelmez → producer timeout bekler → client bloklanır. (2) Geçici queue birikimi → silinmezse memory. (3) Latency ekledi: async sisteme sync bekleme. (4) Ölçekleme zor — hangi instance cevapladı? Alternatif: gRPC (typed contract, streaming, timeout built-in), HTTP REST (simpler), Feign client. Kullanım: RabbitMQ'da RPC → nadiren iyi fikir. gRPC veya HTTP tercih et.

---

## Bölüm 5: Operasyon & Güvenilirlik

### Gerçek Hayat Sorunları

---

**Sorun 10: Memory alarm tetiklendi — tüm publisher'lar bloklandı**

```
Senaryo:
  RabbitMQ node bellek kullanımı %40'ı geçti → memory alarm tetiklendi.
  Tüm AMQP connection'ların yazma tarafı bloklandı (flow control).
  OrderService: basicPublish() çağrısı dondu → HTTP request timeout → kullanıcıya hata.
  
  Neden: Queue'lar dolu, consumer yavaş işliyor → mesajlar RAM'de birikti.

RabbitMQ memory management:
  vm_memory_high_watermark = 0.4 (default: toplam RAM'in %40)
  %40 → alarm → tüm publisher'lar blok
  Disk: disk_free_limit (1GB minimum) → disk dolunca da blok

Teşhis:
  Management UI → Overview → Memory ve Disk kullanımı
  rabbitmqctl list_connections | grep blocked → bloklanmış connection'lar
  rabbitmqctl list_queues name messages → queue derinliği
  
Acil çözüm:
  1. Consumer kapasitesini artır (daha fazla consumer instance).
  2. Kısa vadeli: mesaj hızını düşür (producer tarafında rate limit).
  3. queue TTL ekle → eski mesajlar DLX'e → RAM boşalt.

Kalıcı önlem:
  Prefetch doğru ayarla → consumer'lar dengeli yük taşısın.
  Lazy queue: mesajları RAM yerine disk'e yaz (x-queue-mode=lazy).
  vm_memory_high_watermark artır veya node memory artır.
  Monitoring: queue depth > threshold → alert (alarm gelmeden önce gör).

Spring AMQP — publisher blocked eventi:
  @EventListener(ListenerContainerConsumerFailedEvent.class)
  void onBlockedConnection(ListenerContainerConsumerFailedEvent event) {
      alertService.notify("RabbitMQ publisher blocked!");
  }
```

---

**Sorun 11: Connection-per-request — TCP storm, RabbitMQ çöktü**

```java
// Senaryo: Her HTTP isteğinde yeni RabbitMQ connection açılıp kapatılıyor.
// Yüksek trafik: saniyede 500 istek → saniyede 500 TCP connection → RabbitMQ aşırı yük.

// YANLIŞ: Her istekte yeni connection
@PostMapping("/orders")
ResponseEntity<?> createOrder(@RequestBody OrderRequest req) {
    try (Connection conn = factory.newConnection(); // TCP handshake ~10ms
         Channel ch = conn.createChannel()) {
        ch.basicPublish("orders", "new", props, body);
    }
    // Connection kapatıldı → sonraki request yeniden açar
    // 500 RPS → 500 TCP conn/sn → RabbitMQ file descriptor limiti
}

// DOĞRU: Connection pool — bir kez aç, tekrar kullan
// Spring AMQP: ConnectionFactory bean — otomatik connection/channel pooling
@Configuration
class RabbitConfig {
    @Bean
    CachingConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory("rabbitmq-host");
        factory.setChannelCacheSize(25);     // kanal pool boyutu
        factory.setCacheMode(CacheMode.CHANNEL); // channel'ları cache'le, connection'ı değil
        return factory;
    }
    
    @Bean
    RabbitTemplate rabbitTemplate(CachingConnectionFactory factory) {
        RabbitTemplate template = new RabbitTemplate(factory);
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) log.error("NACK: {}", cause);
        });
        return template;
    }
}
// Uygulama: birkaç connection + channel pool → binlerce istek

// KURAL:
// Connection → uygulama başına 1-2 (TCP, pahalı)
// Channel → thread/işlem başına (lightweight, ucuz sanal çoklama)
```

---

### Mülakat Soruları — Operasyon

**Junior / Mid:**

13. `Connection` ve `Channel` farkı nedir? Neden her request'te connection açmamalısın?

    > **Beklened:** Connection: TCP bağlantısı. Pahalı: handshake, authentication ~10ms. Uygulama başına az sayıda. Channel: connection üzerinde lightweight sanal multiplexing. Ucuz, thread başına ayrı channel. Neden her request'te açmamalı: TCP overhead — yüksek RPS'de bağlantı sayısı patlıyor. RabbitMQ file descriptor limiti → "too many connections" hatası. Connection pool: Spring AMQP `CachingConnectionFactory` → connection'ı bir kez aç, channel'ları pool'la. Channel thread-safe değil — thread başına ayrı channel kullan.

14. RabbitMQ memory alarm nedir? Tetiklenince ne olur?

    > **Beklened:** RabbitMQ node RAM'in belirli oranını (%40 default) aşınca memory alarm tetikler. Sonuç: tüm AMQP connection'ların yazma yönü bloklanır (flow control). Publisher blok → `basicPublish()` askıda kalır → HTTP request timeout → cascading failure. Disk alarm: `disk_free_limit` altına düşünce → benzer blok. Çözüm: consumer hızını artır (daha fazla worker), queue TTL ile eski mesajları DLX'e at, lazy queue (RAM → disk), node belleği artır. Monitoring: alarm gelmeden önce queue depth ve memory alertleri kur.

---

**Senior / Architect:**

15. Quorum Queue'yu Classic Queue'dan üstün kılan teknik fark nedir?

    > **Beklened:** Classic Queue: eski mirroring (synchronous, tüm mirror'lara yaz). Mirror sayısı = n → n bağlantı → yavaş. Failover süreci tanımsız. Quorum Queue: Raft consensus — çoğunluk (quorum) yazar → commit. 3 node: 2'si yeterli → 1 node çökse devam eder. Avantajlar: kesin veri güvenliği (split-brain koruması), poison message built-in tracking (`x-delivery-limit`), daha öngörülebilir failover. Dezavantaj: ~%30 daha yavaş (Raft overhead), daha fazla disk I/O. Stream Queue ek karşılaştırma: non-destructive consume, Kafka benzeri offset. Production kararı: yeni proje → Quorum. Classic'i migrate et. Stream → replay gerektiren fanout.

16. RabbitMQ'da "shovel" ve "federation" ne işe yarar?

    > **Beklened:** Shovel: queue veya exchange'den başka bir RabbitMQ instance'a mesaj kopyalar (farklı DC, migration). Tek yönlü, mesaj taşıma. Kullanım: RabbitMQ versiyonu yükseltirken canlı migration, farklı data center'a fan-out. Federation: exchange veya queue'yu remote'dan "çek" — upstream'den downstream'e upstream'in bant genişliğini kullanarak. Shovel'dan farkı: federation daha gevşek bağlı, her iki taraf bağımsız çalışmaya devam edebilir. Kullanım: coğrafi dağıtım, çok bölgeli mimari, mesh network. Her ikisi de plugin gerektirir.

---

## Karma — Architect Seviyesi

17. **"Bir e-ticaret sisteminde sipariş akışını RabbitMQ ile tasarla. Exchange türü, queue yapısı, DLX, retry ve monitoring nasıl kurulur?"**

    > **Beklened:** Exchange tasarımı: `orders-exchange` (topic) → routing key: `order.created`, `order.cancelled`, `order.shipped`. Consumer'lar: `invoice-queue` (binding: `order.created`), `notification-queue` (binding: `order.#`), `inventory-queue` (binding: `order.created`). Her queue: durable + DLX set edilmiş. DLX: `orders-dlx` (direct) → her queue'dan gelen başarısız mesajlar → ilgili DLQ (invoice-dlq, notification-dlq...). Retry: DLX + TTL wait pattern (3 deneme, her seferinde artan gecikme). Publisher confirms: tüm publish işlemleri → ACK beklenir. Prefetch: her consumer servise özel (invoice: 5, notification: 20). Monitoring: queue depth per queue → alert (threshold'u işleme süresine göre ayarla), DLQ derinliği > 0 → immediate alert, memory/disk alarms, consumer count (0 consumer → alarm).

18. **"RabbitMQ mi, Kafka mı? Ekibinde her iki sistem de var. Hangi iş yükü nereye?"**

    > **Beklened:** RabbitMQ güçlü: karmaşık routing (topic/headers exchange), öncelikli kuyruk (priority queue), TTL + delayed message, request-reply pattern, küçük/orta throughput, task distribution (her görevi bir worker alsın), mesaj consume edilince gitsin. Kafka güçlü: çok consumer group bağımsız → birden fazla servis aynı event'i okusun, replay (yeni servis geçmişi okusun), yüksek throughput (100K+ msg/sn), event sourcing, audit log, stream processing. Karma mimari: order-service → Kafka `order-events` (inventory + email + analytics bağımsız okur). invoice-service → RabbitMQ task queue (yoğun fatura işleme, öncelik desteği). notification-service → RabbitMQ (routing key ile email vs SMS, TTL). analytics → Kafka (replay, Flink entegrasyonu). Karar sorusu: "Mesaj consume edilince gitsin mi?" → RabbitMQ. "Birden fazla consumer bağımsız mı?" → Kafka.

19. **"RabbitMQ cluster'ın bir node'u düştü. Ne olur ve nasıl recovery yaparsın?"**

    > **Beklened:** Classic Queue — node affinity: Queue'nun bulunduğu node düştüyse → o queue unavailable (mirror yoksa). Mirroring (deprecated): mirror'lar promote edilir ama senkron değilse veri kaybı mümkün. Quorum Queue — Raft: quorum varsa (3 node'da 2) cluster çalışır. Düşen node'un partition'ları kalan node'lar tarafından devralınır. Veri güvenli (quorum'a yazılmıştı). Recovery adımları: (1) Node'un neden düştüğünü bul (disk, memory, network). (2) `rabbitmqctl cluster_status` → hangi node'lar up. (3) Node'u geri getir → otomatik cluster'a katılır, quorum queue'ları sync eder. (4) Uzun süre down olduysa: `rabbitmq-queues rebalance all` → partition liderliklerini dengele. (5) Rolling restart için: quorum'u koru (3 node'da her seferinde 1 node).

20. **"Bir servis RabbitMQ queue'sunu tüketiyor ama mesaj sayısı düşmüyor. Debug planın ne?"**

    > **Beklened:** (1) Management UI → queue: messages, messages_ready, messages_unacknowledged. Unacked yüksekse: consumer mesajı alıyor ama ack etmiyor → işleme takıldı. Ready yüksekse: consumer mesajı almıyor → consumer count'u kontrol et. (2) Consumer count = 0 → consumer down veya hiç bağlanmadı. (3) basicQos/prefetch: yüksek prefetch → bir consumer tümünü aldı, diğerleri idle. Çözüm: prefetch düşür. (4) Consumer log: işleme hatası var mı? DB timeout? Harici API timeout? (5) Message rate in vs message rate out: in > out → throughput yetersiz → consumer say artır. (6) Requeue loop: nack+requeue → mesaj döngüde, düşmüyor. Header'da retry sayacı var mı? (7) Memory alarm: publisher bloklu → mesaj gönderilemiyor ama görünüyor mesaj düşmüyor. Confirm: `rabbitmqctl list_connections | grep blocked`.
