# 04a — Apache Kafka

## Ne?

Yüksek throughput, düşük gecikme ve kalıcı mesaj saklama özelliğine sahip dağıtık **event streaming platformu**. Kafka mesajları consumer tarafından okunduktan sonra silmez — log olarak saklar, tekrar okunabilir.

---

## Neden?

**Çözdüğü problem:** Servisler birbirinden bağımsız çalışmalı, anlık yük spike'ları absorbe edilmeli, mesajlar kaybolmamalı ve aynı mesaj birden fazla sistem tarafından okunabilmeli.

```
Kafka öncesi: Servis-to-servis direkt çağrı
OrderService → HTTP → InventoryService  (inventory çöktüyse ne olur?)
OrderService → HTTP → EmailService      (email servisi yavaşsa ne olur?)
OrderService → HTTP → AnalyticsService  (her yeni servis = yeni bağımlılık)

Kafka sonrası:
OrderService → [orders topic] → InventoryService  (bağımsız, kendi hızında)
                              → EmailService
                              → AnalyticsService
                              → AuditService       (yeni servis → sadece subscribe et)
```

LinkedIn 2011'de geliştirdi: günde 1 trilyon mesaj işleme ihtiyacından doğdu.

---

## Nasıl?

### Temel Yapı

```
Producer → [Topic: orders] → Consumer Group A (InventoryService)
                           → Consumer Group B (EmailService)

Topic
├── Partition 0  [msg1, msg4, msg7]  → Broker 1 (Leader)
│                                    → Broker 2 (Follower/ISR)
├── Partition 1  [msg2, msg5, msg8]  → Broker 2 (Leader)
│                                    → Broker 3 (Follower/ISR)
└── Partition 2  [msg3, msg6, msg9]  → Broker 3 (Leader)
                                     → Broker 1 (Follower/ISR)
```

**Terimler:**
```
Topic      → Mesajların kategorize edildiği kanal (veritabanındaki tablo gibi)
Partition  → Topic'in paralel bölümü; sıra ve ölçekleme birimi
Offset     → Partition içindeki mesaj pozisyonu (0, 1, 2...) — immutable, artan
Segment    → Partition'ın disk'teki dosya parçası
ISR        → In-Sync Replicas: leader ile senkron olan follower listesi
```

---

### Producer İçi Mekanizma

```
ProducerRecord (topic, key, value, headers)
        ↓
Serializer (key + value → bytes)
        ↓
Partitioner
  key varsa   → hash(key) % numPartitions  (aynı key hep aynı partition)
  key yoksa   → sticky partitioning (batch dolana kadar aynı partition)
        ↓
RecordAccumulator (in-memory buffer, batch.size kadar biriktirir)
        ↓  linger.ms beklenir
Sender Thread → Broker'a batch gönderir
        ↓
Broker → acks bekler
```

**Kritik producer parametreleri:**

| Parametre | Değer | Etki |
|-----------|-------|------|
| `acks=0` | No wait | En hızlı, veri kaybı riski |
| `acks=1` | Leader yazar | Hızlı, leader crash'inde kayıp |
| `acks=all` | Tüm ISR yazar | Güvenli, yavaş |
| `batch.size=16384` | 16KB batch | Büyük batch = yüksek throughput |
| `linger.ms=5` | 5ms bekle | Daha dolu batch, yüksek latency |
| `compression.type=snappy` | Sıkıştır | CPU vs network trade-off |
| `enable.idempotence=true` | Tekrar önle | Exactly-once producer garantisi |

**Idempotent Producer nasıl çalışır:**
```
1. Producer'a unique PID (Producer ID) verilir
2. Her mesaja sequence number eklenir (PID, partition, sequence)
3. Broker: aynı PID + sequence → duplicate → reddeder
4. Sonuç: producer retry yapsa bile mesaj bir kez yazılır
```

---

### Consumer İç Mekanizma

```
Consumer Group: "inventory-service"
├── Consumer-1 → Partition 0
├── Consumer-2 → Partition 1
└── Consumer-3 → Partition 2

Kural: 1 partition → max 1 consumer (aynı group içinde)
Consumer > Partition sayısı → bazı consumer'lar boşta
Consumer < Partition sayısı → bir consumer birden fazla partition okur
```

**Offset Commit Stratejileri:**
```java
// 1. Auto commit (riskli)
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);
// Sorun: işlemeden commit → crash → at-most-once (kayıp)

// 2. Manual commit — at-least-once
while (true) {
    ConsumerRecords<K, V> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<K, V> record : records) {
        process(record);
    }
    consumer.commitSync(); // batch sonrası commit — crash → duplicate
}

// 3. Per-message commit (yavaş ama güvenli)
for (ConsumerRecord<K, V> record : records) {
    process(record);
    consumer.commitSync(Map.of(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)
    ));
}
```

**Rebalancing:**
```
Tetikleyiciler: consumer join, crash, heartbeat timeout, subscription değişimi

Stop-the-world rebalancing (eski):
  Tüm consumer'lar durur → koordinatör yeniden atar → devam
  Sorun: büyük group'ta dakikalar sürebilir

Cooperative rebalancing (Kafka 2.4+, IncrementalCooperativeStickyAssignor):
  Sadece etkilenen partition'lar yeniden atanır
  Diğer consumer'lar çalışmaya devam eder → daha az downtime
```

---

### Broker İçi Mekanizma

**Log yapısı (disk):**
```
/kafka-logs/orders-0/   (topic: orders, partition: 0)
├── 00000000000000000000.log      ← mesajlar binary olarak append
├── 00000000000000000000.index    ← offset → file position (O(1) lookup)
├── 00000000000000000000.timeindex← timestamp → offset mapping
└── 00000000000004321000.log      ← yeni segment (önceki kapandı)

Segment kapanma koşulu:
  log.segment.bytes=1GB veya log.roll.ms=7gün
```

**Sequential write neden hızlı:**
```
Random write (HDD): ~100 I/O/sn  → yavaş
Sequential write (HDD): ~1M byte/sn → Kafka bunu kullanır
OS page cache: write → önce cache, sonra disk (async flush)
Zero-copy: disk → network, kernel bypass (sendfile syscall)
```

**Retention politikaları:**
```
Time-based:   retention.ms=604800000   (7 gün)
Size-based:   retention.bytes=10737418240 (10 GB)
Compaction:   cleanup.policy=compact   → her key'in son değeri saklanır

Log Compaction ne zaman:
  event sourcing, config değişikliği, user profili gibi
  "son state önemli, geçmiş önemli değil" senaryoları

Compaction öncesi:  [A:1] [B:2] [A:3] [C:4] [B:5]
Compaction sonrası: [A:3] [C:4] [B:5]
```

---

### Exactly-Once Semantics

```
Problem: Producer retry → duplicate
         Consumer crash → reprocess → duplicate

Çözüm: Idempotent producer + Transactional API

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", key, value));
    producer.send(new ProducerRecord<>("audit", key, audit));
    // Consumer offset'i de transaction içinde commit et
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction(); // atomik rollback
}

Isolation level consumer'da:
props.put("isolation.level", "read_committed");
// Sadece committed transaction'ların mesajlarını görür
```

---

### KRaft (Zookeeper'dan Geçiş)

```
Eski mimari (Zookeeper):
Kafka Broker → Zookeeper (leader election, metadata, config)
Sorunlar:
  - Ek bağımlılık (Zookeeper operasyonu ayrı iş)
  - Metadata bottleneck (200K partition limiti)
  - Controller election: saniyeler

KRaft (Kafka 3.3+ GA):
Kafka Broker → Raft consensus (kendi içinde)
  - Tek sistem → daha az operasyon
  - Milyonlarca partition desteği
  - Controller election: ~milisaniye
  - Snapshot ile hızlı recovery
```

---

### Kafka Streams vs Consumer API

| | Consumer API | Kafka Streams |
|-|-------------|---------------|
| **Kullanım** | Oku, işle, yaz | Stream processing, join, aggregate |
| **State** | Uygulama yönetir | RocksDB ile otomatik (local) |
| **Exactly-once** | Manuel kurulum | Yerleşik destek |
| **Join** | Manuel implement | Built-in (stream-stream, stream-table) |
| **Windowing** | Manuel | Tumbling, hopping, sliding, session |
| **Scaling** | Manuel thread | Partition'a göre otomatik |

```java
// Kafka Streams örneği: sipariş sayısını 1 dakikalık window'da say
StreamsBuilder builder = new StreamsBuilder();
builder.stream("orders", Consumed.with(Serdes.String(), orderSerde))
    .filter((key, order) -> order.getStatus() == COMPLETED)
    .groupBy((key, order) -> order.getCategory())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count(Materialized.as("order-counts"))
    .toStream()
    .to("order-counts-per-minute");
```

---

## Ne zaman?

**Kafka kullan:**
```
✓ Yüksek throughput (100K+ mesaj/sn)
✓ Mesaj geçmişini saklamak ve tekrar okumak (replay)
✓ Birden fazla consumer aynı mesajı okumalı (fanout)
✓ Event sourcing, audit log, CDC pipeline
✓ Stream processing (Kafka Streams, Flink)
✓ Microservisler arası decoupling, event-driven mimari
✓ Log aggregation (ELK'e alternatif)
```

**Kafka kullanma:**
```
✗ Mesaj routing karmaşıksa (topic/partition basit routing yapar) → RabbitMQ
✗ Çok düşük latency gerekiyorsa (<1ms) → NATS veya Redis Pub/Sub
✗ Küçük team, basit kullanım → RabbitMQ daha kolay yönetilir
✗ Geçici mesajlar (consume edilince gitsin) → RabbitMQ daha uygun
✗ Tek mesaj için RPC pattern → gRPC veya HTTP
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Çok yüksek throughput (milyon msg/sn) | Kurulum ve operasyon karmaşık |
| Kalıcı log — replay, audit, CDC | Küçük mesajlar için overhead büyük |
| Horizontal scaling (partition ekle) | Kesin global sıra yok (partition içi var) |
| Consumer group ile paralel işlem | Consumer offset yönetimi dikkat ister |
| Birden fazla consumer group bağımsız | Schema yönetimi (Avro + Schema Registry şart) |
| Retention: günler/haftalar/sonsuza | Rebalancing downtime (cooperative ile azaldı) |
| Kafka Streams ile stream processing | At-least-once varsayılan, exactly-once kurulumu karmaşık |

---

## Architect Kontrol Listesi

```
□ Partition sayısı: consumer sayısı kadar (sonradan artırılabilir, azaltılamaz)
□ Replication factor: 3 (prod) — 2 broker'ın çökmesini tolere eder
□ acks=all + enable.idempotence=true (veri kaybı riski sıfıra indirilir)
□ Consumer max.poll.interval.ms > işlem süresi (rebalance fırtınası önlenir)
□ Schema Registry kurulumu (Avro/Protobuf şema evrim yönetimi)
□ Log retention politikası belirle (storage planlama)
□ Monitoring: consumer lag, under-replicated partition, ISR shrink
□ Dead Letter Topic (başarısız mesajlar için)
□ Compaction vs deletion (use-case'e göre)
□ KRaft mi Zookeeper mi (yeni kurulum → KRaft)
```
