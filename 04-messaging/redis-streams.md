# 04c — Redis Streams

## Ne?

Redis 5.0 ile gelen, Kafka'ya benzer ama çok daha hafif **append-only log** veri yapısı. Her entry bir ID (timestamp-sequence) ve field-value çiftlerinden oluşur. Consumer group desteği ile paralel okuma, ACK mekanizması ile at-least-once semantics sağlar.

---

## Neden?

**Çözdüğü problem:** Kafka kurmak fazla operasyonel yük, RabbitMQ'nun replay yeteneği yok. Zaten Redis altyapısı varsa, aynı Redis instance'ında Kafka benzeri streaming yapısı oluşturabilirsin.

```
Kafka için: Zookeeper/KRaft + 3+ Broker + Schema Registry + kafesin kurulumu
Redis Streams için: Zaten çalışan Redis instance'ı + XADD komutu

Senaryolar:
- Aynı mikroservisteki modüller arası event akışı
- Audit log, activity feed
- Rate limiting + event'leri birleştir
- Kafka'dan önce hızlı prototip
```

---

## Nasıl?

### Temel Komutlar

```
Stream ID formatı: <millseconds>-<sequence>
Örnek: 1705312800000-0   → zaman sırasına göre otomatik sıralı
       *                 → Redis otomatik üretir (önerilen)
```

```bash
# Entry ekle (Producer)
XADD orders * customer_id 123 product_id 456 amount 99.99
# Döner: "1705312800000-0"  ← entry ID

# Belirtilen ID ile
XADD orders 1705312800000-1 customer_id 124 product_id 789 amount 49.99

# Stream uzunluğunu sınırla
XADD orders MAXLEN ~ 10000 * customer_id 125 ...
# MAXLEN ~ = yaklaşık trim (tam değil, performans için)
# MAXLEN   = kesin trim (yavaş)

# Stream'i oku
XREAD COUNT 10 STREAMS orders 0  # 0'dan itibaren max 10 entry
XREAD COUNT 10 STREAMS orders $  # $ = sadece yeni (blocking olmayan)

# Blocking read (yeni entry gelene kadar bekle)
XREAD COUNT 10 BLOCK 0 STREAMS orders $
# BLOCK 0 = sonsuza kadar bekle
# BLOCK 5000 = 5 saniye bekle

# Range oku
XRANGE orders - +              # tümü
XRANGE orders 1705312800000 +  # belirli timestamp'ten sonrası
XREVRANGE orders + -           # ters sıra (son entry önce)

# Stream bilgisi
XLEN orders            # entry sayısı
XINFO STREAM orders    # detay bilgi
```

---

### Consumer Group — Paralel İşlem

```
Stream: orders
Consumer Group: "inventory-processors"
├── Consumer inventory-1 → entry'leri kısmen okur
├── Consumer inventory-2 → entry'leri kısmen okur
└── Consumer inventory-3 → entry'leri kısmen okur

Her entry sadece bir consumer'a gider (Kafka gibi)
Consumer crash → entry pending listesinde bekler → başkası alabilir
```

```bash
# Consumer group oluştur
XGROUP CREATE orders inventory-processors $ MKSTREAM
#                                          ↑ $ = şu andan itibaren
#                                          0 = baştan okumak için

# Consumer olarak oku
XREADGROUP GROUP inventory-processors consumer-1 COUNT 10 STREAMS orders >
# > = gruba assign edilmemiş yeni entry'leri al

# Entry işlendi → ACK et
XACK orders inventory-processors 1705312800000-0

# Pending (ACK edilmemiş) entry'leri gör
XPENDING orders inventory-processors - + 10
# Döner: entry ID, consumer adı, idle time, delivery count

# Başka consumer'a devret (crash recovery)
XCLAIM orders inventory-processors consumer-2 60000 1705312800000-0
# 60000ms'den fazla idle → consumer-2'ye devret
```

**Crash recovery flow:**
```
1. consumer-1 entry'yi aldı, işlemeye başladı
2. consumer-1 crash oldu → ACK gelmedi
3. consumer-2 veya yönetici XPENDING ile bekleyen entry'yi tespit eder
4. XCLAIM ile consumer-2'ye devredilir
5. consumer-2 işler, XACK eder
```

---

### Pub/Sub vs Streams vs List Karşılaştırması

```
Redis Pub/Sub:
  - Fire and forget (subscriber yoksa mesaj kaybolur)
  - Persistence yok
  - Consumer offline → mesaj yok
  Kullanım: Gerçek zamanlı bildirim, cache invalidation signal

Redis List (LPUSH/RPOP):
  - Simple queue, FIFO
  - Persistence var (AOF/RDB)
  - Consumer group yok (tek consumer)
  - Replay yok
  Kullanım: Basit task queue

Redis Streams:
  - Persistent log
  - Consumer group → paralel işlem
  - ACK mekanizması
  - Replay (ID'den başla)
  - Zaman serisi desteği
  Kullanım: Event log, multi-consumer, at-least-once delivery
```

---

### Spring Data Redis Streams

```java
// Producer
@Service
class OrderEventPublisher {
    @Autowired RedisTemplate<String, Object> redis;

    void publish(OrderCreatedEvent event) {
        redis.opsForStream().add(
            MapRecord.create("orders", Map.of(
                "orderId",    event.getOrderId().toString(),
                "customerId", event.getCustomerId().toString(),
                "amount",     event.getAmount().toString(),
                "timestamp",  Instant.now().toString()
            ))
        );
    }
}

// Consumer (manual)
@Service
class OrderConsumer {
    @Autowired RedisTemplate<String, Object> redis;

    void consume() {
        List<MapRecord<String, Object, Object>> records = redis.opsForStream()
            .read(Consumer.from("inventory-processors", "consumer-1"),
                  StreamReadOptions.empty().count(10).block(Duration.ofSeconds(5)),
                  StreamOffset.create("orders", ReadOffset.lastConsumed()));

        for (MapRecord<String, Object, Object> record : records) {
            processOrder(record.getValue());
            redis.opsForStream().acknowledge("orders", "inventory-processors", record.getId());
        }
    }
}

// Consumer (declarative — Spring integration)
@Component
class StreamConsumer {

    @StreamListener("orders")
    void handle(MapRecord<String, String, String> record) {
        String orderId = record.getValue().get("orderId");
        processOrder(orderId);
    }
}

// StreamMessageListenerContainer — otomatik okuma ve ACK
@Bean
StreamMessageListenerContainer<String, MapRecord<String, String, String>> container(
    RedisConnectionFactory factory) {

    StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> options =
        StreamMessageListenerContainerOptions.builder()
            .pollTimeout(Duration.ofSeconds(1))
            .build();

    StreamMessageListenerContainer<String, MapRecord<String, String, String>> container =
        StreamMessageListenerContainer.create(factory, options);

    container.receive(
        Consumer.from("inventory-processors", "consumer-1"),
        StreamOffset.create("orders", ReadOffset.lastConsumed()),
        record -> {
            processOrder(record.getValue().get("orderId"));
            // ACK otomatik yapılmaz, manuel gerekir
        }
    );

    return container;
}
```

---

## Ne zaman?

**Redis Streams kullan:**
```
✓ Zaten Redis altyapısı varsa (ek bileşen yok)
✓ Küçük-orta ölçek mesajlaşma (< 100K msg/sn)
✓ Hızlı prototip veya MVP
✓ Audit log, activity feed (Redis'te sorgulanabilir)
✓ Aynı servis içi modüller arası event akışı
✓ Basit stream processing
✓ IoT cihaz verisi toplama (zaman serisi ile birlikte)
```

**Redis Streams kullanma:**
```
✗ Yüksek throughput (100K+ msg/sn) → Kafka
✗ Uzun süreli mesaj saklama (hafta/ay) → Kafka (Redis RAM sınırlı)
✗ Mesaj boyutu büyükse → Redis memory baskısı
✗ Karmaşık routing → RabbitMQ
✗ Multi-datacenter replikasyon → Kafka MirrorMaker
✗ Stream processing (join, window) → Kafka Streams / Flink
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Ek bileşen yok (Redis zaten var) | RAM sınırlı → mesaj saklama kısıtlı |
| Basit API (Redis komutları) | Kafka kadar yüksek throughput değil |
| Consumer group + ACK | Consumer rebalancing manuel |
| Zaman serisi ile doğal entegrasyon | Partition/sharding yok (tek key → tek shard) |
| Düşük latency (sub-ms) | Cluster'da cross-slot sınırlaması |
| Replay (ID bazlı okuma) | Monitoring araçları Kafka kadar olgun değil |
