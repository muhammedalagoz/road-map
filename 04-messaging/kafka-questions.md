# 04a — Apache Kafka: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Producer

### Gerçek Hayat Sorunları

---

**Sorun 1: `acks=1` + leader crash → sipariş verisi kayboldu**

```
Senaryo:
  E-ticaret: OrderService sipariş event'ini Kafka'ya yazıyor.
  Producer konfigürasyonu: acks=1 (sadece leader onayı).
  
  Akış:
    1. Producer → Broker 1 (leader, partition 0) → acks=1 → "başarılı"
    2. Broker 1 → Broker 2 (follower) replikasyonu: HENÜz tamamlanmadı
    3. Broker 1 crash etti → Broker 2 leader oldu
    4. Broker 2: o mesajı hiç almamıştı → kayıp
  
  Sonuç: Sipariş "başarılı" gönderildi ama Kafka'da yok.
  InventoryService mesajı hiç görmedi → stok düşmedi → tutarsızlık.

Doğru konfigürasyon:
  props.put("acks", "all");
  props.put("enable.idempotence", "true");
  props.put("retries", Integer.MAX_VALUE);
  props.put("max.in.flight.requests.per.connection", "5"); // idempotence ile güvenli

  # Broker tarafında (topic veya cluster):
  min.insync.replicas=2  # en az 2 ISR yazmalı, aksi halde ProducerException
  replication.factor=3   # 1 broker çöksün, 2 hâlâ ISR

Sonuç:
  acks=all + min.insync.replicas=2:
  Leader yazar → en az 1 follower daha yazar → sonra ACK.
  Leader crash: follower leader olur, veri zaten orada.
```

---

**Sorun 2: Partition'a tüm mesaj gitti — tek consumer bunaldı**

```java
// Senaryo: OrderService mesajları gönderirken key göndermedi.
// Bir süre sonra consumer-1 log'ları akarken consumer-2 ve consumer-3 idle.

// YANLIŞ: key=null → sticky partitioning
producer.send(new ProducerRecord<>("orders", null, orderJson));
// Sonuç: Batch dolana kadar aynı partition'a → tüm yük tek consumer'a

// Belirti:
// - consumer-1 lag: 500K mesaj
// - consumer-2 lag: 0
// - consumer-3 lag: 0

// DÜZELTME 1: İş mantığına uygun key seç
// Aynı müşterinin siparişleri sıralı olsun istiyorsan:
producer.send(new ProducerRecord<>("orders", customerId.toString(), orderJson));
// hash(customerId) % numPartitions → aynı müşteri hep aynı partition

// DÜZELTME 2: Round-robin (sıra önemsiz, load balance kritik):
// Kafka 2.4+: key=null → sticky (batch bazlı). Tam round-robin için:
props.put("partitioner.class", "org.apache.kafka.clients.producer.RoundRobinPartitioner");

// DÜZELTME 3: Custom partitioner
public class RegionPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String region = extractRegion(value);
        return region.equals("EU") ? 0 : region.equals("US") ? 1 : 2;
    }
}
```

---

**Sorun 3: `linger.ms=0` + `batch.size` küçük → throughput düşük**

```
Senaryo:
  Yeni Kafka producer: varsayılan ayarlar → saniyede 5K mesaj üretebiliyor.
  Gereksinim: 100K mesaj/sn.
  
  Sorun: Her mesaj ayrı network round-trip → overhead.

Varsayılan:
  batch.size    = 16384 (16KB)
  linger.ms     = 0 (batch dolmasını bekleme, hemen gönder)
  compression   = none

Her mesaj küçükse (200 byte): batch dolmadan gönderiliyor → N mesaj = N round-trip.

Düzeltme: Yüksek throughput için
  batch.size            = 131072   # 128KB — daha dolu batch
  linger.ms             = 20       # 20ms bekle, batch biriktir
  compression.type      = lz4      # hız/sıkıştırma dengesi (snappy de iyi)
  buffer.memory         = 67108864 # 64MB producer buffer
  max.request.size      = 1048576  # 1MB max request

  Sonuç: 20ms'de biriken mesajlar bir batch → tek round-trip → throughput artışı.
  Trade-off: latency 0ms → 20ms (çoğu event-driven sistemde kabul edilebilir).
```

---

### Mülakat Soruları — Producer

**Junior / Mid:**

1. `acks=0`, `acks=1`, `acks=all` farkları nelerdir? Ne zaman hangisi?

   > **Beklened:** `acks=0`: Producer ACK beklemez, "fire and forget". En hızlı, veri kaybı yüksek. Kullanım: metrik, log (kayıp tolere edilebilir). `acks=1`: Sadece leader ACK. Orta güvenlik, orta hız. Leader crash before replica sync → kayıp. `acks=all`: Tüm ISR (min.insync.replicas kadar) yazar → ACK. En güvenli, yavaş. Finansal işlem, sipariş. Üçlü kural: `acks=all` + `min.insync.replicas=2` + `replication.factor=3` → production minimum güvenlik standardı.

2. Idempotent producer nedir? Neden önemlidir?

   > **Beklened:** Network hatası → producer retry → aynı mesaj tekrar gönderilir → duplicate. Idempotent producer: her mesaja PID (Producer ID) + sequence number eklenir. Broker: aynı PID + sequence gelirse → duplicate → reddeder, ACK döner. Sonuç: retry olsa bile mesaj bir kez yazılır. Aktif etmek: `enable.idempotence=true`. Bu `acks=all` + `max.in.flight.requests.per.connection=5`'i de zorunlu kılar. Sadece partition bazında idempotentlik sağlar — farklı partition'lara gönderimde transactional API gerekir.

---

**Senior / Architect:**

3. `batch.size` ve `linger.ms` arasındaki gerilim nedir? Throughput ve latency nasıl dengelenir?

   > **Beklened:** batch.size: buffer dolunca gönder. linger.ms: dolmasa bile bu kadar bekle. linger.ms=0: ilk mesaj gelince hemen gönder → düşük throughput. linger.ms=50: 50ms biriktir → dolu batch → yüksek throughput ama +50ms latency. Trade-off: event-driven analytics → linger.ms=20-50 kabul edilebilir. Kullanıcı bekleyen senkron işlem → linger.ms küçük tut. compression.type: snappy/lz4 → CPU overhead ama network tasarrufu → net throughput artışı (network genellikle bottleneck). Gerçek: linger.ms ve batch.size yükle birlikte ayarla — load test ile belirle.

---

## Bölüm 2: Consumer & Rebalancing

### Gerçek Hayat Sorunları

---

**Sorun 4: `max.poll.interval.ms` çok düşük — rebalance fırtınası**

```
Senaryo:
  Consumer: her mesaj için DB yazıyor + harici API çağrısı (bazen 45sn).
  max.poll.interval.ms = 30000 (default).
  
  Akış:
    1. Consumer batch alır (100 mesaj).
    2. 60. mesajı işlerken harici API 45sn + DB 10sn = 55sn.
    3. 30sn geçti → coordinator "consumer öldü" sayar → rebalance tetiklenir.
    4. Bu consumer group'tan çıkarılır → partition başka consumer'a verilir.
    5. Yeni consumer aynı offset'ten başlar → zaten işlenen 60 mesaj tekrar işlenir.
    6. Yeni consumer yine 30sn'de timeout → rebalance → sonsuz döngü.
    5. Consumer lag artıyor ama hiç ilerlenemiyor.

Teşhis: consumer_lag artıyor, rebalance_rate yüksek ama aslında işleme yapılıyor.

Düzeltme 1: max.poll.interval.ms artır
  max.poll.interval.ms = 300000 (5dk)
  
Düzeltme 2: max.poll.records küçült — her batch'te daha az işle
  max.poll.records = 10 (default 500)
  → Her poll daha az mesaj → poll() sık sık çağrılır → heartbeat kesilmez
  
Düzeltme 3: İşlemi async yap
  Consumer → mesajı queue'ya at → worker thread işlesin → tamamlanınca commit
  Dikkat: commit sırası ve hata yönetimi karmaşıklaşır.
  
Düzeltme 4: Cooperative sticky rebalancing
  props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
  Rebalance olsa bile sadece etkilenen partition duruyor.
```

---

**Sorun 5: Auto-commit açık — crash sonrası mesaj kayboldu**

```java
// Senaryo: Sipariş mesajları işleniyor, auto-commit açık.
// Consumer crash etti → bazı siparişler kayboldu (işlenmedi ama commit edildi).

props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000); // 5 saniyede bir commit

// Akış:
// T=0: 100 mesaj alındı
// T=0-3s: 50 mesaj işlendi
// T=5s: AUTO COMMIT → offset 101 commit edildi (henüz işlenmeyen 51-100 dahil!)
// T=6s: Consumer crash
// T=7s: Yeni consumer offset 101'den başlar → 51-100 hiç işlenmedi → kayıp!

// Bu: at-most-once (en fazla bir kez) semantiği

// DÜZELTME: Manual commit — at-least-once
props.put("enable.auto.commit", false);

while (true) {
    ConsumerRecords<String, OrderEvent> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, OrderEvent> record : records) {
        try {
            processOrder(record.value());
        } catch (RetryableException e) {
            // Yeniden deneme kuyruğuna at, bu mesajı atla
            deadLetterProducer.send("orders-dlq", record);
        }
    }
    
    // BATCH sonunda commit — at-least-once
    // Crash olursa: tüm batch tekrar işlenir → idempotency gerekli
    consumer.commitSync();
}

// İdempotency: DB'de unique constraint veya idempotency key
// INSERT INTO orders (idempotency_key, ...) ON CONFLICT DO NOTHING
```

---

**Sorun 6: Poison message — partition durdu, consumer loop'a girdi**

```java
// Senaryo: Bir mesaj deserializasyon hatası veriyor (schema uyumsuzluğu).
// Consumer defalarca retry yapıyor ama ilerleyemiyor.

// YANLIŞ: Exception yakalamazsan consumer durur
consumer.poll(Duration.ofMillis(100)).forEach(record -> {
    OrderEvent event = deserialize(record.value()); // ClassCastException!
    // Exception → consumer.poll() çağrılmıyor → heartbeat yok → rebalance
});

// Yanlış düzeltme — sonsuz retry (poison message loop):
while (true) {
    try {
        processOrder(record.value());
        break;
    } catch (Exception e) {
        Thread.sleep(1000); // her saniye deniyor → sonsuza kadar
    }
}
// Bu partition'daki sonraki mesajlar ASLA işlenmez!

// DOĞRU: Dead Letter Topic (DLQ) pattern
for (ConsumerRecord<String, byte[]> record : records) {
    try {
        OrderEvent event = deserialize(record.value());
        processOrder(event);
    } catch (DeserializationException e) {
        // İşlenemeyen mesaj DLQ'ya
        dlqProducer.send(new ProducerRecord<>("orders-dlq",
            record.key(),
            record.value()
        ));
        log.error("Poison message sent to DLQ: offset={}", record.offset(), e);
        // Commit et ve geç — partition ilerler
    }
}
consumer.commitSync();

// Spring Kafka: @RetryableTopic ile daha temiz:
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltTopicSuffix = "-dlq"
)
@KafkaListener(topics = "orders")
void handle(OrderEvent event) { ... }
// 3 deneme sonrası otomatik "orders-dlq" topic'ine yönlendirir
```

---

### Mülakat Soruları — Consumer

**Junior / Mid:**

4. Consumer group ve partition arasındaki ilişkiyi açıkla. Consumer sayısı partition'dan fazla olursa ne olur?

   > **Beklened:** Consumer group: aynı topic'i beraber tüketen consumer'lar. Her partition aynı grup içinde sadece 1 consumer tarafından okunabilir. Paralel işlem: 6 partition → 6 consumer mümkün. Consumer > partition: fazla consumer'lar idle bekler — işe yaramaz ama zarar vermez. Consumer < partition: bir consumer birden fazla partition okur — yük dengesiz olabilir. Ölçekleme: partition sayısı = max paralel consumer. Başlangıçta az partition → sonradan artırılabilir (azaltılamaz!).

5. Auto-commit ile manual commit farkı nedir? At-least-once ve at-most-once ne anlama gelir?

   > **Beklened:** Auto-commit: periyodik (5sn) offset commit → crash durumunda bazı mesajlar işlendi ama commit edilmedi → tekrar işlenir (at-least-once). Veya bazıları commit edildi ama henüz işlenmemişti → kayıp (at-most-once riski). Manual commit: işlem tamamlanınca commit → at-least-once garantisi. At-least-once: mesaj en az 1 kez işlenir, crash durumunda duplicate mümkün → idempotency ile yönet. At-most-once: mesaj en fazla 1 kez işlenir, kayıp mümkün. Exactly-once: transactional API + idempotency → en karmaşık.

6. Rebalancing neden olur? Sistemi nasıl etkiler?

   > **Beklened:** Tetikleyiciler: yeni consumer join, consumer crash/timeout, heartbeat gelmemesi, subscription değişimi, topic partition artışı. Eager rebalancing (eski default): tüm consumer'lar partition'larını bırakır → grup yeniden atama yapar → devam. Downtime: büyük grupta dakikalar. Cooperative rebalancing: sadece taşınan partition'lar durdurulur, diğerleri çalışır. Sorun: heartbeat thread ≠ işleme thread. max.poll.interval.ms: işlem süresi bu'nu aşarsa → "consumer öldü" → rebalance. Çözüm: max.poll.records küçült + max.poll.interval.ms artır.

---

**Senior / Architect:**

7. Cooperative sticky assignor ile eager assignor farkı nedir? Ne zaman önemlidir?

   > **Beklened:** Eager (RangeAssignor, RoundRobinAssignor): rebalance → tüm consumer partition'larını bırakır → koordinatör yeniden atar → tüm consumer başlar. "Stop-the-world" → büyük grupta (100+ consumer) dakikalar sürer. Cooperative sticky (IncrementalCooperativeStickyAssignor): rebalance → sadece taşıması gereken partition'lar bırakılır → diğerleri çalışmaya devam eder → iki tur protokol. Önemli: büyük consumer group, düşük latency SLA. Kafka 3.x: varsayılan cooperative. Yapılandırma: `partition.assignment.strategy = CooperativeStickyAssignor`. Dikkat: eager'dan cooperative'e geçiş rolling restart gerektirir.

8. Consumer lag'ı nasıl izlersin? Lag artışı neye işaret eder?

   > **Beklened:** Consumer lag = latest offset − consumer's committed offset. Metric: `kafka.consumer_group.lag` (Kafka JMX) → Prometheus + Grafana. Araç: `kafka-consumer-groups.sh --describe` veya Burrow, Cruise Control. Lag artışı: (1) Consumer çok yavaş (DB bottleneck, yavaş API). (2) Producer hızı > consumer hızı — geçici spike. (3) Consumer death/rebalance loop. (4) Partition sayısı yetersiz. Aksiyon: lag > eşik → alert. Consumer sayısını artır (partition sayısı izin veriyorsa). İşleme paralelize et. Yavaş consumer → profil al, bottleneck bul.

---

## Bölüm 3: Broker & Partition Tasarımı

### Gerçek Hayat Sorunları

---

**Sorun 7: Partition sayısı az — sonradan artırınca sıra bozuldu**

```
Senaryo:
  "orders" topic: 3 partition. Müşteri bazlı key → aynı müşteri aynı partition.
  Trafik arttı → 3 partition yetmedi → 9'a çıkardık.

  Sorun:
  hash("customer-42") % 3 = 1 → Partition 1 (eski)
  hash("customer-42") % 9 = 7 → Partition 7 (yeni)

  Müşteri 42'nin eski mesajları Partition 1'de.
  Yeni mesajlar Partition 7'de.
  InventoryService: customer-42'nin mesajlarını farklı partition'lardan okuyor.
  Sıra garantisi bozuldu → stok önce düşürüldü, sonra eklendi gibi işlendi.

Çözüm:
  1. Baştan doğru partition sayısı belirle.
     Formül: peak_throughput / single_consumer_throughput.
     Örnek: 100K mesaj/sn ÷ 10K mesaj/sn = 10 partition.
     Yuvarlak yüksek güç: 12 → gelecekte 12, 24, 48'e kolayca artır.

  2. Partition sayısını artırmak zorundaysan:
     Geçiş dönemi: eski ve yeni consumer'ları birlikte çalıştır.
     Key bazlı sıra kritikse: topic'i yeniden oluştur, yeni topic'e migrate et.
     Compacted topic: partition artışı + offset reset + replay ile yeniden populate et.

  3. Sıra kritik değilse: artırım sorunsuz — sadece load balance bozulabilir.
```

---

**Sorun 8: Under-replicated partition — ISR küçüldü, veri kaybı riski**

```
Senaryo:
  3 broker, replication factor=3. Broker 2 GC pause yaptı → ISR'dan düştü.
  ISR: [broker1, broker3].
  min.insync.replicas=2 → hâlâ çalışıyor (2 ISR var).
  Broker 3 de ağ sorunu → ISR: [broker1].
  min.insync.replicas=2 → bu karşılanmıyor → producer NotEnoughReplicasException!
  Yazma durdu.

Under-replicated partition (URP): replication factor < beklenen.
Kritik alarm: URP > 0 → broker veya ağ sorunu işareti.

Monitoring:
  kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
  Alert: URP > 0 için immediate alert

Önlem:
  replication.factor=3 (minimum production)
  min.insync.replicas=2 (1 broker çöksün, 2 hâlâ yeter)
  unclean.leader.election.enable=false (ISR dışı broker leader olmasın → kayıp önle)
  replica.lag.time.max.ms=30000 → 30sn lag'da ISR'dan çıkar

Recovery:
  Lag'daki broker: otomatik olarak ISR'a geri döner (veriler sync olunca).
  Zorlama: kafka-reassign-partitions.sh ile partition'ı sağlıklı broker'a taşı.
```

---

**Sorun 9: Log compaction — beklenmedik mesaj kaybı**

```
Senaryo:
  "user-profiles" topic: cleanup.policy=compact.
  InventoryService: kullanıcı profili güncellemelerini bu topic'ten okuyor.
  Yeni consumer başladı: auto.offset.reset=earliest → baştan oku.
  Compaction zaten çalışmış: her key'in sadece son değeri var.
  
  Sorun değil mi? — Bazen problem:
  1. Null value (tombstone): key'i sil demek.
     Compaction: tombstone'u bir süre tutar, sonra siler.
     Yeni consumer hızlı okuyorsa: tombstone'u göremedi → sanki o kayıt hiç silinmedi.
  
  2. Compaction lag: son yazılan mesajlar henüz compact edilmedi.
     "Head" (uncompacted): tüm mesajlar.
     "Tail" (compacted): sadece son değerler.
     Yeni consumer: head'deki eski değerleri de okur → sonra güncellemeyi de → net sıra doğru.
     Ama işlem süresi artar.

  3. Compaction'ın ne zaman tetiklendiği belirsiz:
     min.cleanable.dirty.ratio=0.5 → %50 dirty segment → compaction başlar.
     Aktif yazmada compaction gecikmesi olabilir.

Net kural:
  Compacted topic: "son state" için. Tam tarih gerekmiyorsa tercih et.
  Event history gerekmiyorsa compaction + retention kombinasyonu kullan.
  delete.retention.ms: tombstone ne kadar süre tutulsun (default 24 saat).
```

---

### Mülakat Soruları — Broker & Partitioning

**Junior / Mid:**

9. Partition sayısını nasıl belirlersin?

   > **Beklened:** Formül: `max(consumer_parallelism, throughput/single_consumer_throughput)`. Throughput: hedef mesaj/sn ÷ bir consumer'ın işleyebildiği mesaj/sn. Consumer parallelism: kaç thread'e ihtiyaç var. Başlangıç: az partition başla (değiştirmek zorunda kalınca sıra bozulabilir). Artırım kolayca yapılabilir (azaltılamaz!). Production minimum: trafik tahminine göre + %20-30 buffer. Broker sayısı: ideal partition/broker = dengeli. Tek broker'a çok partition → hotspot.

10. ISR nedir? Neden önemlidir?

    > **Beklened:** In-Sync Replicas: leader ile senkronize olan follower'lar. Senkron = son N ms içinde leader'a yetişmiş (replica.lag.time.max.ms). ISR önemi: `acks=all` → tüm ISR yazar → ACK. ISR'da sadece leader varsa ve min.insync.replicas=2 → write reddedilir (koruyucu). `unclean.leader.election.enable=false`: ISR dışı broker leader seçilirse → o broker'da olmayan mesajlar kaybolur. Disable → kayıp önlenir ama partition leader seçilene kadar unavailable. Karar: finans → unclean election false. Yüksek availability > durability → true (ama kayıp riski).

11. Log compaction ile time-based retention farkı nedir?

    > **Beklened:** Time-based (delete policy): retention.ms sonra eski mesajlar silinir. Tüm geçmiş, belirli süre. Event log, audit trail için uygun. Log compaction: her key için son değer tutulur, eskiler silinir. Anahtar: key ile çalışılır (null key → compaction işlemez). Kullanım: kullanıcı profili, konfigürasyon state, CDC son state. Null value (tombstone): key'i "sil" işareti. Kombinasyon: delete + compact → hem eski sil hem son değer tut. Event sourcing: tüm event geçmişi lazımsa delete; son state lazımsa compact.

---

**Senior / Architect:**

12. `unclean.leader.election` trade-off'u nedir? Finansal sistemde nasıl karar verirsin?

    > **Beklened:** unclean.leader.election.enable=true: ISR dışı (lag'da olan) broker leader olabilir. Availability artar ama o broker'da olmayan mesajlar kaybolur. false: ISR'da broker yoksa partition unavailable → durability > availability. Finansal sistem: false zorunlu. Para mesajı kaybolursa → reconciliation sorunu, müşteri şikayeti, yasal sorun. Availability kaybı: partition unavailable → write fail → application error → ama en azından veri tutarlı. Recovery: ISR broker'lardan biri kurtarılınca → leader election → normal. Monitoring: URP alert → broker hızla düzelt.

---

## Bölüm 4: Exactly-Once Semantics

### Gerçek Hayat Sorunları

---

**Sorun 10: Ödeme duplicate — at-least-once ile çift işlem**

```java
// Senaryo: PaymentService Kafka'dan ödeme komutunu okuyor.
// At-least-once: crash durumunda mesaj tekrar işlendi → çift ödeme.

// PROBLEM AKIŞI:
// 1. Consumer mesajı aldı
// 2. Ödeme API çağrıldı: payment-gateway.charge(100$) → başarılı
// 3. Consumer DB'ye kaydettikten sonra commit EDİLEMEDEN crash etti
// 4. Yeni consumer aynı offset'ten başladı → charge(100$) tekrar → çift ödeme!

// ÇÖZÜM 1: Idempotency key
void processPayment(ConsumerRecord<String, PaymentCommand> record) {
    String idempotencyKey = record.topic() + "-" + record.partition() + "-" + record.offset();
    
    // DB'de idempotency key var mı?
    if (paymentRepo.existsByIdempotencyKey(idempotencyKey)) {
        log.info("Duplicate payment skipped: {}", idempotencyKey);
        return; // duplicate → atla
    }
    
    // Yoksa işle
    gateway.charge(command.getAmount(), command.getCardToken());
    paymentRepo.save(new Payment(idempotencyKey, command));
    // commit → idempotency key kaydedildi → tekrarda atlanır
}

// ÇÖZÜM 2: Kafka Transactions (read-process-write döngüsü)
producer.initTransactions();
while (true) {
    ConsumerRecords<String, PaymentCommand> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    try {
        for (var record : records) {
            PaymentResult result = processPayment(record.value());
            producer.send(new ProducerRecord<>("payment-results", result));
        }
        // Consumer offset'ini transaction içine al
        producer.sendOffsetsToTransaction(
            currentOffsets(records), consumer.groupMetadata()
        );
        producer.commitTransaction(); // atomik: sonuç yazıldı + offset commit
    } catch (Exception e) {
        producer.abortTransaction(); // her ikisi de geri alındı
    }
}
// Consumer: isolation.level=read_committed → sadece committed mesajlar okunur
```

---

### Mülakat Soruları — Exactly-Once

**Junior / Mid:**

13. At-least-once, at-most-once ve exactly-once semantikleri nasıl sağlanır?

    > **Beklened:** At-most-once: commit first, then process → crash: mesaj kaybolur. En basit, kayıp riski. At-least-once: process then commit → crash: tekrar işlenir. Default Kafka. Idempotency ile güvenli. Exactly-once: (1) Idempotent producer (retry = no duplicate) + (2) Consumer offset'ini transactional olarak commit et (sonucu yaz + offset commit atomik). Tam exactly-once: `enable.idempotence=true` + `transactional.id` + `isolation.level=read_committed`. Maliyet: throughput düşer ~%10-20. Karar: finansal → exactly-once. Analytics/log → at-least-once (idempotency ile).

---

**Senior / Architect:**

14. Kafka transactional API nasıl çalışır? Ne zaman kullanırsın?

    > **Beklened:** transactional.id: producer'a sabit ID. initTransactions(), beginTransaction(), send(), sendOffsetsToTransaction(), commitTransaction() veya abortTransaction(). Broker: transaction coordinator — commit/abort kararını log'a yazar. Consumer: isolation.level=read_committed → sadece committed mesajları görür → uncommitted transaction mesajları görünmez (buffer'da bekler). Kullanım: read-process-write: Kafka'dan oku → işle → Kafka'ya yaz + offset commit atomik. Mesaj → DB + başka topic: 2-phase commit. Overhead: transaction log + coordinator → ~10-20% throughput kayıp. Karar: banking, inventory deduction → evet. Log analytics → hayır.

---

## Bölüm 5: Schema & Operasyon

### Gerçek Hayat Sorunları

---

**Sorun 11: Schema değişikliği — consumer crash etti**

```
Senaryo:
  OrderService (producer): OrderEvent JSON gönderiyordu.
  v1: {"orderId": 1, "amount": 100.0}
  
  Geliştirici: "customerId alanı ekleyelim" → v2 deploy:
  v2: {"orderId": 1, "amount": 100.0, "customerId": 42}
  
  InventoryService (consumer) hâlâ eski kod:
  OrderEvent.class: orderId, amount alanları var, customerId YOK.
  JSON deserialize: ek alan → Jackson varsayılan olarak ignore eder → tamam.
  
  Ama: "email" alanının tipini String'den Long'a değiştirdik:
  v3: {"orderId": 1, "amount": 100.0, "customerId": 42, "email": 12345}
  Consumer: email String bekliyor, Long geldi → JsonMappingException → crash!

  Sorun: Schema versiyonlama yok, breaking change kontrolsüz yapıldı.

Çözüm: Schema Registry + Avro/Protobuf
  1. Schema Registry (Confluent): her schema versiyonu merkezi kayıt.
  2. Compatibility kontrolü:
     BACKWARD: yeni schema, eski mesajı okuyabilir (alan sil/varsayılan ekle)
     FORWARD: eski schema, yeni mesajı okuyabilir (alan ekle)
     FULL: her ikisi birden (en güvenli)
  
  // Avro schema: alan ekleme — backward compat
  {
    "type": "record", "name": "OrderEvent",
    "fields": [
      {"name": "orderId", "type": "long"},
      {"name": "amount", "type": "double"},
      {"name": "customerId", "type": ["null", "long"], "default": null} // yeni, opsiyonel
    ]
  }
  // Schema Registry: eski consumer yeni mesajı okur, customerId null gelir → OK
  // Breaking change (type değiştirme) → registry reddeder → deploy durdurulur
```

---

**Sorun 12: Disk doldu — Kafka broker çöktü**

```
Senaryo:
  Kafka retention: 7 gün. Trafik normalde 100GB/gün → 700GB yeterli.
  Black Friday: trafik 10× arttı → günde 1TB → retention süresinde 7TB.
  Disk: 2TB → 2 günde doldu → broker I/O error → ISR'dan düştü → URP alarm.

Teşhis:
  df -h /kafka-logs  → %95+ doluluk
  kafka-log-dirs.sh --describe → topic/partition bazlı alan kullanımı

Acil:
  # Retention süresini kıs (topic seviyesinde)
  kafka-configs.sh --alter --entity-type topics --entity-name orders \
    --add-config retention.ms=86400000  # 1 güne düşür
  # Hemen silinmez, log cleaner döngüsünde siler (segment bazlı)
  
  # Compaction varsa: compaction hızlandır
  log.cleaner.io.max.bytes.per.second → artır

Kalıcı önlem:
  1. Topic bazlı retention izleme dashboard'u.
  2. Disk kullanımı %70 → alert, %85 → critical.
  3. Tiered Storage (Kafka 3.6+): eski segment'leri S3'e taşı → sınırsız retention.
  4. Broker storage planlaması: peak × 2 (retention gün sayısı) + %20 buffer.
```

---

### Mülakat Soruları — Schema & Operasyon

**Junior / Mid:**

15. Kafka'da Schema Registry neden kullanılır? Olmadan ne olur?

    > **Beklened:** Schema Registry olmadan: producer ve consumer arasında "hangi format" sözleşmesi yoktur. Producer JSON yapısını değiştirirse → consumer crash (tip değişikliği, alan silme). Schema Registry: her schema versiyonu merkezi kayıt. Compatibility mode: BACKWARD/FORWARD/FULL. Breaking change detect → deploy reddedilir (CI/CD gate). Avro/Protobuf: binary format → JSON'dan küçük (3-10×), hızlı serialize. Sadece schema ID mesaja eklenir → consumer schema'yı registry'den çeker. Olmadan: JSON + manuel sözleşme → coordination overhead, runtime hata riski.

16. Kafka topic'te `retention.ms` ve `retention.bytes` nasıl çalışır?

    > **Beklened:** retention.ms: bu süre geçen segment'ler silinir (segment bazlı, dosya kapanınca). Varsayılan: 7 gün (604800000ms). retention.bytes: partition başına max boyut. Aşılınca en eski segment silinir. İkisi aynı anda aktif: hangisi önce tetiklenirse. Segment bazlı: aktif (yazılan) segment silinmez — kapanana kadar. Dolayısıyla anlık silme değil, segment rotasyonu beklenir (`log.segment.bytes`, `log.roll.ms`). Infinite retention: `retention.ms=-1`. Monitoring: topic'in gerçek disk kullanımı vs planlanan.

---

**Senior / Architect:**

17. Outbox Pattern Kafka ile nasıl implemente edilir? Neden gereklidir?

    > **Beklened:** Problem: DB kaydı + Kafka mesajı atomik olmalı. `save(order)` → başarılı. `kafkaProducer.send()` → fail → veri tutarsızlığı. Outbox Pattern: (1) Sipariş kaydı + outbox tablosu kaydı aynı DB transaction'ında. (2) Ayrı process (Debezium veya polling) outbox tablosunu okur → Kafka'ya yazar. (3) Kafka yazma başarılı → outbox kaydı sil/işaretle. Debezium CDC: PostgreSQL WAL → outbox tablosu change event → Kafka connector → topic. Avantaj: exactly-once garantisi olmayan sistem için güvenilir delivery. Dezavantaj: ek tablo, ek servis (connector), gecikme (ms-sn). Alternatiif: Kafka Transactions (uygulama seviyesinde ama 2PC olmadan).

---

## Bölüm 6: Kafka Streams

### Gerçek Hayat Sorunları

---

**Sorun 13: Kafka Streams state store — disk doldu**

```
Senaryo:
  Kafka Streams uygulaması: siparişleri window'da aggregate ediyor.
  State store (RocksDB): local disk'e yazıyor.
  Changelog topic: state değişikliklerini Kafka'da saklıyor.
  
  6 ay sonra: RocksDB dizini 500GB → disk doldu → uygulama çöktü.

Neden büyüdü:
  Windowed aggregate: pencere kapansa da state temizlenmedi.
  Konfigürasyon yanlışlığı: grace period çok uzun → eski state tutuldu.

Teşhis:
  du -sh /tmp/kafka-streams/  → büyük
  State store boyutu: RocksDB stats

Düzeltme:
  // Pencere ve grace period doğru ayarla
  .windowedBy(
    TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1))  // grace=0: hemen temizle
    // veya
    TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(10)) // 10sn grace
  )

  // State store için retention sınırı:
  Stores.persistentWindowStore("order-counts",
    Duration.ofMinutes(60),  // retention: 60dk eski state sil
    Duration.ofMinutes(1),   // window size
    false
  )
  
  // Compaction'ı agresifleştir:
  default.windowed.key.serde.inner = Serdes.String()
  rocksdb.config.setter = CustomRocksDBConfig  // compaction tuning

  // Monitoring: state store boyutunu metric olarak al
  streams.metrics() → kafkastreams.state-store.total-size-bytes
```

---

### Mülakat Soruları — Kafka Streams

**Junior / Mid:**

18. Kafka Streams ile Consumer API arasında nasıl seçim yaparsın?

    > **Beklened:** Consumer API: basit consume-process-produce. State'i kendin yönetirsin (DB, Redis). Tam kontrol, esneklik. Join, aggregation → elle yazılır. Kafka Streams: stateful processing — join, aggregate, windowing built-in. State: RocksDB (local, hızlı). Exactly-once built-in. Scaling: partition'a göre otomatik. Karar: "Kafka'dan oku → DB'ye yaz" → Consumer API yeterli. "Stream-stream join", "son 1 dakikadaki sipariş sayısı" → Streams. "Flink/Spark zaten var" → Consumer API yeterli. Overhead: Streams uygulaması state store, changelog topic → ek disk, ek topic.

---

## Karma — Architect Seviyesi

19. **"Bir e-ticaret sisteminde sipariş akışını Kafka ile tasarla. Partition sayısı, consumer group, DLQ ve exactly-once nasıl kurulur?"**

    > **Beklened:** Topic tasarımı: `orders` (sipariş komutları), `order-events` (durum değişiklikleri, fan-out), `orders-dlq` (başarısız mesajlar). Partition sayısı: peak throughput / consumer throughput. 50K mesaj/sn ÷ 5K/consumer = 10 partition. Key: customerId → aynı müşteri sıralı. Consumer groups: inventory-service, email-service, analytics-service — her biri bağımsız. Replication: factor=3, min.insync.replicas=2, acks=all. Exactly-once: Outbox pattern (DB transaction + CDC) → güvenilir Kafka yazma. Consumer: idempotency key (topic-partition-offset), DLQ sonrası alert. Monitoring: consumer lag per group, URP, disk kullanımı. Schema Registry: Avro + FULL compatibility.

20. **"Kafka consumer lag'ı artıyor. Debug planın ne?"**

    > **Beklened:** (1) Hangi consumer group ve hangi partition'da lag var? `kafka-consumer-groups.sh --describe` veya Grafana dashboard. (2) Lag sabit mi artıyor mu? Artıyorsa: consumer hızı < producer hızı. Sabit: gecikmeli ama yetişiyor. (3) Consumer log'ları: exception var mı? Rebalance log'u fazla mı? (4) Consumer processing time: tek mesaj işlemesi ne kadar sürüyor? DB yavaş mı? Harici API timeout mu? (5) max.poll.records: çok yüksekse, her poll'da çok mesaj alıp işleyemiyor → azalt. (6) Partition sayısı: consumer sayısına eşit mi? Fazla consumer var ama boşta mı? (7) Çözüm seçenekleri: a) Consumer sayısı artır (partition izin veriyorsa). b) Partition sayısı artır + consumer sayısı artır. c) Processing'i optimize et (batch DB insert, async harici API). d) Mesajı filtrele — gereksiz mesajları atla.

21. **"Kafka'yı RabbitMQ ile karşılaştır. Her ikisinin de bulunduğu bir mimaride hangi iş yükü nerede?"**

    > **Beklened:** Kafka güçlü: yüksek throughput (milyon msg/sn), log retention + replay, birden fazla consumer group bağımsız, event sourcing, CDC, stream processing. RabbitMQ güçlü: kompleks routing (exchange/binding), mesaj önceliği, request-reply pattern, per-message TTL/expiry, küçük team / basit kurulum, geçici mesajlar (consume edilince gitsin). Karma mimari: order-service → Kafka (sipariş event, yüksek volume, audit). email-service ← RabbitMQ (öncelikli kuyruk, routing per email type). payment-gateway ← RabbitMQ (request-reply, güvenilir delivery, az mesaj). analytics ← Kafka (replay, stream processing, BI entegrasyon). Karar: "mesajı birden fazla servis bağımsız okumalı + geçmiş lazım" → Kafka. "kompleks routing + az mesaj" → RabbitMQ.

22. **"Kafka'da exactly-once semantiği sağlandı mı demek için ne gerekir?"**

    > **Beklened:** 3 katman: (1) Producer exactly-once: `enable.idempotence=true` → retry'da duplicate yok. (2) Transaction: `transactional.id` → birden fazla partition'a yazma atomik. (3) Consumer exactly-once: `isolation.level=read_committed` + `sendOffsetsToTransaction()` → offset commit ile mesaj yazma atomik. Üçü birden: read-process-write loop'u exactly-once. Sınırlar: producer → broker arası. Consumer → external system (DB) arası değil! DB write + Kafka commit atomik değil → idempotency key veya outbox pattern zorunlu. "Kafka exactly-once = Kafka içindeki mesaj akışı için geçerli, dış sistemlerle sınırda uygulama seviyesinde garanti gerekir."
