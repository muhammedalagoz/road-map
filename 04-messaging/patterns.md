# 04h — Mesajlaşma Pattern'ları

Belirli bir teknolojiye bağlı olmayan, her mesajlaşma sisteminde geçerli mimari pattern'lar.

---

## 1. Competing Consumers

### Ne?
Aynı kuyruğu birden fazla consumer'ın dinlemesi. Her mesaj yalnızca bir consumer tarafından işlenir.

### Neden?
Tek consumer işlem kapasitesini aşarsa queue birikir. Birden fazla consumer ile yük dağıtılır, throughput doğrusal artar.

### Nasıl?
```
Queue: order-processing-queue
├── Consumer-1 → msg1, msg4, msg7...
├── Consumer-2 → msg2, msg5, msg8...
└── Consumer-3 → msg3, msg6, msg9...

Kafka'da:
  1 partition = max 1 consumer (aynı group içinde)
  6 partition + 6 consumer = tam paralel

RabbitMQ'da:
  1 queue → N consumer → broker round-robin dağıtır
  basicQos(prefetch=10) → consumer kapasitesine göre dağıtım

SQS'de:
  1 queue → N consumer → her consumer ReceiveMessage çağırır
  Visibility timeout ile çift işlem önlenir
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Yatay ölçekleme | Mesaj sırası garantisi bozulur |
| Tek consumer çöküşü sistemi durdurmaz | İdempotency şart (duplicate riski) |
| Consumer sayısı dinamik artırılabilir | Stateful işlemler zorlaşır |

---

## 2. Message Ordering Garantileri

### Ne?
Mesajların gönderildiği sırada işlenmesi.

### Neden?
Sipariş durumu: `CREATED → PAID → SHIPPED`. Bunlar ters sırayla işlenirse tutarsız durum oluşur.

### Nasıl?
```
Global sıra (tüm mesajlar sıralı):
  Sadece 1 consumer, 1 partition → throughput kurbanlığı

Partition/key bazlı sıra (önerilen):
  Kafka:      aynı key → aynı partition → o partition içinde sıralı
  FIFO SQS:   MessageGroupId ile grup bazlı sıra
  RabbitMQ:   1 queue, 1 consumer (sıra kritikse)

Örnek — Kafka ile sipariş sırası:
  producer.send("orders", orderId, event)  // key = orderId
  → hash(orderId) % partitionCount → hep aynı partition
  → aynı orderId'nin tüm event'leri sıralı

Neye dikkat et:
  Bir sipariş için 10 event:  sıralı gelir (aynı partition)
  100 farklı sipariş:         birbirinden bağımsız, paralel işlenebilir
```

### Trade-off?
| Yöntem | Sıra Garantisi | Throughput |
|--------|---------------|-----------|
| Global sıra | Tüm mesajlar | Çok düşük |
| Key-based | Key başına | Yüksek |
| Best-effort | Yok | En yüksek |

---

## 3. Idempotent Consumer

### Ne?
Aynı mesajı birden fazla işlemek aynı sonucu üretir — tekrar zararsız.

### Neden?
At-least-once garantili sistemlerde duplicate mesaj kaçınılmazdır. Consumer çöküp kalkarsa mesaj tekrar gelir. İdempotency bunu güvenli hale getirir.

### Nasıl?
```
Duplicate kaynakları:
  Producer retry (acks=1, leader crash)
  Consumer crash (ack göndermeden önce)
  Network partition (mesaj iki kez iletildi)
  Rebalancing (offset geriye döndü)

Çözüm 1: Doğal idempotency
  PUT /orders/{id}/status → idempotent (aynı state'e set et)
  INSERT ... ON CONFLICT DO NOTHING → duplicate = no-op

Çözüm 2: Idempotency key tablosu
  processed_messages(message_id PK, processed_at)

  @Transactional
  void process(Message msg) {
      if (processedMessageRepo.existsById(msg.getId())) {
          return; // zaten işlendi
      }
      doActualWork(msg);
      processedMessageRepo.save(new ProcessedMessage(msg.getId()));
  }

Çözüm 3: Kafka transactional outbox
  Kafka transaction: consume + produce atomik
  consumer.commitSync ile DB + offset atomik

Dikkat:
  message_id tablosu sonsuza büyümesin → TTL ile temizle
  TTL = max mesaj retention süresinden uzun olmalı
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Güvenli retry ve redelivery | Ekstra DB sorgusu (her mesaj için) |
| At-least-once → effectively exactly-once | message_id tablosu yönetimi |
| Basit hata recovery | Distributed transaction gerektiren durumlar zor |

---

## 4. Dead Letter Queue (DLQ) Pattern

### Ne?
İşlenemeyen mesajların atıldığı özel kuyruk. Ana akışı bloke etmez, sorunlu mesajları izole eder.

### Neden?
Bir mesaj sürekli exception fırlatıyorsa (poison message) sonsuz retry ile queue bloke olur. DLQ bunu önler.

### Nasıl?
```
Normal akış:
  orders-queue → Consumer → başarılı → ack → mesaj silindi

Hata akışı (DLQ):
  orders-queue → Consumer → exception → nack → retry
  → maxRetry aşıldı → orders-dlq

DLQ'daki mesajlar için:
  1. Alert: monitoring sistemi uyarır
  2. İnceleme: mesajın neden başarısız olduğunu analiz et
  3. Düzelt: Kodu düzelt veya mesajı manuel düzelt
  4. Replay: orders-dlq → orders-queue (yeniden işle)

Kafka'da:
  @RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltTopicSuffix = ".DLT"
  )
  @KafkaListener(topics = "orders")
  void consume(OrderEvent event) { ... }
  // orders → orders-retry-0 → orders-retry-1 → orders.DLT

RabbitMQ'da:
  x-dead-letter-exchange: orders.dlx
  x-max-delivery-count: 3   // 3 kez → DLX

SQS'de:
  Redrive Policy: maxReceiveCount=3 → DLQ
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Poison message'ı izole eder | DLQ izlenmezse mesajlar sessizce kaybolur |
| Ana akışı bloke etmez | DLQ → replay mekanizması ayrıca kurulmalı |
| Forensics (ne başarısız oldu?) | DLQ büyürse alarm — köklü sorun |

---

## 5. Schema Registry & Mesaj Versiyonlama

### Ne?
Mesaj şemalarını (Avro, Protobuf, JSON Schema) merkezi bir yerde tutan, üretici ve tüketici arasında schema uyumunu zorlayan sistem.

### Neden?
```
Sorun: Producer şemayı değiştirdi, consumer eski şemayı bekliyor → deserialziation exception

Örnek:
  v1: { "orderId": "123", "amount": 99.99 }
  v2: { "orderId": "123", "totalAmount": 99.99 }  // "amount" → "totalAmount"

Consumer v1 şemasıyla v2 mesajı okursa → alan bulunamadı
```

### Nasıl?
```
Confluent Schema Registry (Kafka ile):

Producer flow:
  1. Şema → Schema Registry'ye kaydet
  2. Schema Registry → schema_id döner (int)
  3. Producer mesajı serialize eder:
     [magic byte][schema_id(4 byte)][avro binary]

Consumer flow:
  1. Mesajı al → schema_id oku
  2. Schema Registry'den şemayı çek (cache'li)
  3. O şema ile deserialize et

Compatibility modes:
  BACKWARD:  Yeni şema, ESKİ mesajı okuyabilir
             → Consumer güncelle, sonra producer güncelle
  FORWARD:   ESKİ şema, YENİ mesajı okuyabilir
             → Producer güncelle, sonra consumer güncelle
  FULL:      Her ikisi de → en güvenli, en kısıtlı
  NONE:      Uyumluluk kontrolü yok → tehlikeli
```

**Avro şema örneği:**
```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.example.orders",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "totalAmount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}
```

**Backward compatible değişiklikler:**
```
✓ Default değeri olan yeni alan ekle
✓ Alan sil (consumer okuyamaz ama patlamamalı)

Backward incompatible (yapma):
✗ Default değersiz yeni alan ekle
✗ Var olan alanın tipini değiştir
✗ Required alan sil
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Runtime serialize/deserialize hatası önlenir | Ek bileşen (Schema Registry) |
| Şema evolution güvenli | Avro/Protobuf öğrenme eğrisi |
| Merkezi şema katalog (discovery) | Registry HA kurulumu gerekir |
| Compact binary (Avro) | JSON Schema daha az compact |

---

## 6. Backpressure

### Ne?
Consumer mesajları işleyemiyorsa producer'ı yavaşlatma mekanizması. Sistemin taşmamasını sağlar.

### Neden?
```
Producer: 10.000 msg/sn
Consumer: 1.000 msg/sn
Fark:     9.000 msg/sn → queue/topic hızla dolar → memory → crash
```

### Nasıl?
```
Kafka backpressure:
  Consumer poll hızını ayarla:
  max.poll.records=500       → bir poll'da max 500 mesaj
  max.poll.interval.ms=30000 → 30sn içinde işle (aksi halde rebalance)
  Lag monitörle → Prometheus, Grafana alert

RabbitMQ backpressure:
  basicQos(prefetch=10) → 10 unacked mesaj → yeni mesaj gelmez
  Memory alarm → broker publisher'ı block eder

Reactive backpressure (WebFlux + Kafka):
  Flux<ConsumerRecord> records = kafkaReceiver.receive();
  records
    .limitRate(100)           // max 100 in-flight
    .flatMap(r -> process(r), 10)  // max 10 concurrent
    .subscribe();

SQS backpressure:
  Consumer hızını kontrol edersin (pull model)
  ReceiveMessage çağırmazsın → mesajlar SQS'de bekler (14 güne kadar)
  Visibility timeout → işlenemeyen mesajlar geri döner
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Sistemin taşmasını önler | Throughput azalır |
| Memory/CPU koruma | Gecikme artar |
| Graceful degradation | Ayarlama zor (too little → waste, too much → lag) |

---

## 7. Saga Pattern (Dağıtık Transaction)

### Ne?
Birden fazla servis boyunca yayılan bir işlemi lokal transaction'larla yönetme; başarısız adımı geri almak için compensating transaction çalıştırma.

### Neden?
2PC (Two-Phase Commit) dağıtık sistemlerde uygulanamaz veya çok kısıtlayıcı. Saga lokal transaction'larla ACID'e yakın sonuç üretir.

### Nasıl?
**Choreography (event-driven):**
```
OrderService    → order.created event
InventoryService→ inventory.reserved event
PaymentService  → payment.processed event
ShippingService → shipment.created event
                → Tamamlandı!

Hata: PaymentService → payment.failed event
  InventoryService → inventory.released (compensating)
  OrderService     → order.cancelled   (compensating)

Avantaj: Servisler bağımsız, loosely coupled
Dezavantaj: Akışı takip etmek zor, koordinasyon dağınık
```

**Orchestration (merkezi koordinatör):**
```
SagaOrchestrator:
  1. → InventoryService.reserve()    ✓
  2. → PaymentService.charge()       ✗ hata
  3. → InventoryService.release()    (compensating)
  4. → OrderService.cancel()         (compensating)

Avantaj: Akış tek yerde, debug kolay
Dezavantaj: Orchestrator bağımlılığı, tek nokta
```

```java
// Spring State Machine veya Temporal ile orchestration:
@Component
class OrderSaga {

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    void on(OrderCreatedEvent event) {
        inventoryCommandGateway.send(
            new ReserveInventoryCommand(event.getOrderId(), event.getItems())
        );
    }

    @SagaEventHandler(associationProperty = "orderId")
    void on(InventoryReservedEvent event) {
        paymentCommandGateway.send(
            new ProcessPaymentCommand(event.getOrderId(), event.getAmount())
        );
    }

    @SagaEventHandler(associationProperty = "orderId")
    void on(PaymentFailedEvent event) {
        // Compensating transaction
        inventoryCommandGateway.send(
            new ReleaseInventoryCommand(event.getOrderId())
        );
        orderCommandGateway.send(
            new CancelOrderCommand(event.getOrderId(), "Payment failed")
        );
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    void on(OrderCompletedEvent event) { }
}
```

### Trade-off?
| Avantaj | Dezavantaj |
|---------|-----------|
| Dağıtık transaction mümkün | Eventual consistency (anlık tutarsızlık) |
| Her servis bağımsız | Compensating transaction yazmak zor |
| 2PC'ye göre çok daha ölçeklenebilir | Debug ve monitoring karmaşık |
| Partial failure tolere edilir | Idempotency şart |

---

## 8. Event-Driven Architecture vs Request-Response

### Seçim Kılavuzu

```
Request-Response (REST/gRPC) — NE ZAMAN:
  ✓ Anlık cevap gerekiyor (kullanıcı bekliyor)
  ✓ Basit, doğrusal akış
  ✓ İşlem sonucu hemen gerekli
  ✓ Küçük sistem, az servis

Event-Driven (Kafka/RabbitMQ) — NE ZAMAN:
  ✓ Alıcı hazır olmak zorunda değil
  ✓ Birden fazla sistem aynı olayı işlemeli
  ✓ Yük spike'ı absorbe edilmeli
  ✓ Audit log / replay gerekli
  ✓ Loosely coupled sistem isteniyor

Hybrid (çoğu üretim sistemi):
  Kullanıcı isteği → REST API (senkron, anlık feedback)
  API → Event yayınla → async işlem (email, analytics, inventory)
```

---

## Genel Architect Kontrol Listesi

```
□ İdempotency sağlandı mı? (her consumer için)
□ DLQ tanımlı mı? (her queue/topic için)
□ Consumer lag monitörü var mı?
□ Schema Registry kullanılıyor mu? (Kafka için Avro/Protobuf ile)
□ Poison message nasıl handle ediliyor?
□ Sıra garantisi gerekiyor mu? (varsa partition/key stratejisi)
□ Backpressure mekanizması nedir?
□ Mesaj boyutu sınırı nedir? (Kafka default 1MB, binary > text)
□ Retry stratejisi: exponential backoff + jitter
□ Transaction boundary doğru çizilmiş mi? (outbox pattern gerekiyor mu?)
□ Dead letter → replay mekanizması kuruldu mu?
□ Monitoring: mesaj throughput, latency, error rate
```
