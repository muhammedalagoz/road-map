# 04d — AWS SQS & SNS

## Ne?

**SQS (Simple Queue Service):** AWS'nin yönetilen mesaj kuyruğu. Producer mesaj koyar, consumer alır, işler, siler. Pull-based.

**SNS (Simple Notification Service):** AWS'nin yönetilen pub/sub servisi. Publisher bir topic'e gönderir, tüm subscriber'lar alır. Push-based fanout.

**İkisi birlikte:** SNS → birden fazla SQS → farklı consumer'lar. AWS'nin "Kafka + RabbitMQ" kombinasyonu.

---

## Neden?

**Çözdüğü problem:** Kafka veya RabbitMQ kurmak, yönetmek, ölçeklendirmek operasyonel yük. AWS'de çalışıyorsan SQS/SNS fully managed, sunucusuz, sonsuz ölçeklenebilir, SLA garantili.

```
Kafka cluster = 3+ broker + ZK/KRaft + monitoring + upgrade + disk yönetimi
SQS           = AWS konsolunda 3 tıklama, 0 operasyon, otomatik ölçekleme
```

---

## Nasıl?

### SQS — İç Mekanizma

```
Producer → SQS Queue → Consumer
                ↓
         Visibility Timeout
         (mesaj diğer consumer'lardan gizlenir)

Akış:
1. Consumer ReceiveMessage çağırır
2. SQS mesajı döndürür + visibility timeout başlar (varsayılan: 30sn)
3. Consumer mesajı işler
4. Consumer DeleteMessage çağırır → mesaj silinir
5. Eğer timeout dolmadan DeleteMessage gelmezse → mesaj tekrar görünür
   (başka consumer alabilir veya aynı consumer tekrar alır)
```

**SQS Queue türleri:**

| | Standard Queue | FIFO Queue |
|-|---------------|-----------|
| **Throughput** | Neredeyse sınırsız | 300 msg/sn (batching ile 3000) |
| **Sıra** | Best-effort | Kesin FIFO (MessageGroupId bazında) |
| **Duplicate** | Olabilir (at-least-once) | Tam deduplicate (exactly-once) |
| **Kullanım** | Çoğu senaryo | Finansal işlem, sıra kritik |
| **Fiyat** | Ucuz | Biraz pahalı |

```java
// Spring Boot + SQS (Spring Cloud AWS)
@SqsListener("https://sqs.eu-west-1.amazonaws.com/123456/orders-queue")
void processOrder(OrderCreatedEvent event) {
    // Auto-delete: başarılıysa SQS mesajı siler
    // Exception fırlatırsa: visibility timeout dolar → tekrar gelir
    orderService.process(event);
}

// Manuel ReceiveMessage + Delete
@Autowired SqsTemplate sqsTemplate;

void manualConsume() {
    SqsTemplate.SqsReceiveResponse<OrderCreatedEvent> response =
        sqsTemplate.receive(r -> r.queue("orders-queue").maxNumberOfMessages(10));

    for (Message<OrderCreatedEvent> msg : response.messages()) {
        try {
            orderService.process(msg.getPayload());
            sqsTemplate.delete("orders-queue", msg);
        } catch (Exception e) {
            // Silme → visibility timeout → retry
        }
    }
}

// Mesaj gönder
sqsTemplate.send(to -> to
    .queue("orders-queue")
    .payload(new OrderCreatedEvent(orderId, customerId, amount))
    .delaySeconds(60)          // 60sn sonra görünür (scheduled message)
    .messageGroupId("customer-" + customerId)     // FIFO için
    .messageDeduplicationId(orderId.toString())   // FIFO duplicate önleme
);
```

---

### Dead Letter Queue (DLQ)

```
Normal:
orders-queue → Consumer (başarılı → DeleteMessage)

Başarısız (maxReceiveCount aşıldı):
orders-queue → [3 kez başarısız] → orders-dlq → alert + inceleme

SQS Redrive Policy:
{
  "deadLetterTargetArn": "arn:aws:sqs:...:orders-dlq",
  "maxReceiveCount": 3  // 3 kez başarısızsa DLQ'ya gönder
}
```

---

### SNS — Fanout Mekanizması

```
Publisher → SNS Topic → [Subscription 1: SQS orders-inventory-queue]
                      → [Subscription 2: SQS orders-email-queue]
                      → [Subscription 3: Lambda]
                      → [Subscription 4: HTTP endpoint]
                      → [Subscription 5: Email]
                      → [Subscription 6: SMS]

Yeni servis ekle → sadece topic'e subscribe ol → mevcut kod değişmez
```

```java
// SNS'e yayınla
@Autowired SnsTemplate snsTemplate;

void publishOrderEvent(Order order) {
    snsTemplate.sendNotification(
        "arn:aws:sns:eu-west-1:123456:order-events",
        new OrderCreatedEvent(order),
        "OrderCreated"  // subject
    );
}

// SNS → SQS fanout konfigürasyonu (Terraform ile):
/*
resource "aws_sns_topic_subscription" "inventory" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.inventory_queue.arn

  filter_policy = jsonencode({  // sadece belirli event'leri al
    eventType = ["OrderCreated", "OrderCancelled"]
  })
}
*/
```

**SNS Message Filtering:**
```json
Subscription Filter Policy (inventory-service için):
{
  "eventType": ["OrderCreated"],
  "amount": [{"numeric": [">", 100]}],
  "region": ["EU", "US"]
}

Sadece: eventType=OrderCreated VE amount>100 VE region=EU veya US
Diğerleri SQS'e ULAŞMAZ → consumer işlemez → maliyet azalır
```

---

### SNS + SQS Kombine Pattern

```
OrderService → SNS:order-events
                      │
         ┌────────────┼────────────┐
         ↓            ↓            ↓
SQS:inventory  SQS:email-notif  SQS:analytics
      │               │               │
  InventoryService  EmailService  AnalyticsService
  (3 replica)       (Lambda)      (Batch)

Avantajlar:
- SNS: fanout (bir gönderim, çok alıcı)
- SQS: buffer (alıcı yavaşsa queue biriktirir)
- SQS: retry (başarısız işlem tekrar denenir)
- Bağımsız scaling (her servis kendi kapasitesinde)
```

---

### SQS Long Polling vs Short Polling

```
Short Polling (varsayılan):
  ReceiveMessage → anında döner → mesaj yoksa boş → tekrar call
  Sorun: Boş call = maliyet + CPU waste

Long Polling (önerilen):
  ReceiveMessage(WaitTimeSeconds=20) → 20 saniye bekler → mesaj gelince döner
  Faydası: Daha az boş call → %90'a kadar maliyet azalması

Spring Cloud AWS default: long polling (otomatik)
```

---

### SQS vs Kafka Karşılaştırması

| | SQS | Kafka |
|-|-----|-------|
| **Mesaj saklama** | Consume edilince siler (max 14 gün) | Kalıcı log (sonsuza kadar) |
| **Replay** | Yok | Evet (offset ile) |
| **Throughput** | Çok yüksek (managed) | Çok yüksek (kendi yönet) |
| **Operasyon** | Sıfır (managed) | Yüksek (self-managed) |
| **Ordering** | FIFO Queue ile | Partition içinde |
| **Consumer** | Pull | Pull |
| **Retention** | Max 14 gün | Sonsuza kadar |
| **Maliyet** | Pay-per-use | Sabit altyapı |
| **Ekosistem** | AWS | Cross-cloud |

---

## Ne zaman?

**SQS/SNS kullan:**
```
✓ AWS ortamında çalışıyorsun (IAM entegrasyonu, VPC, Lambda trigger)
✓ Operasyon yükü istemiyorsun (fully managed)
✓ Değişken yük (autoscaling otomatik)
✓ Lambda ile serverless event processing
✓ Basit fanout (SNS → birden fazla SQS)
✓ Multi-region (SNS cross-region delivery)
✓ Compliance (AWS SOC, PCI, HIPAA sertifikaları)
```

**SQS/SNS kullanma:**
```
✗ Mesaj replay gerekiyorsa → Kafka veya Kinesis
✗ Stream processing (join, window) → Kinesis Data Streams
✗ Cross-cloud / on-premise → Kafka
✗ Çok düşük latency (<5ms) → Redis Streams veya NATS
✗ 14 günden uzun retention → Kafka
✗ Vendor lock-in kabul edilemiyorsa → Kafka
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Sıfır operasyon (fully managed) | AWS vendor lock-in |
| Sonsuz ölçekleme | Mesaj replay yok (Standard Queue) |
| IAM ile güvenlik entegrasyonu | 14 gün max retention |
| Lambda trigger native | FIFO'da throughput limiti (300 msg/sn) |
| Pay-per-message (düşük trafik ucuz) | Kafka'ya göre daha yüksek latency |
| SNS ile kolay fanout | Stream processing desteği yok (Kinesis lazım) |
| DLQ, visibility timeout yerleşik | Kafka ekosistemi kadar zengin değil |
