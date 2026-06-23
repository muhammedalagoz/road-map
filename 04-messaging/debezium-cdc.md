# 04f — Debezium & Change Data Capture (CDC)

## Ne?

**CDC (Change Data Capture):** Veritabanındaki her INSERT, UPDATE, DELETE değişikliğini gerçek zamanlı olarak yakalayan teknik.

**Debezium:** Red Hat'in open-source CDC platformu. Veritabanının **transaction log'unu** okur (PostgreSQL WAL, MySQL binlog, MongoDB oplog) ve her değişikliği bir Kafka event'ine dönüştürür. Uygulamaya dokunmaz, veritabanını polling etmez.

---

## Neden?

**Çözdüğü problem:** Outbox pattern'i doğru implement etmek, servisler arası veri senkronizasyonu ve zero-downtime migration.

**Outbox pattern sorunu olmadan:**
```java
@Transactional
void placeOrder(Order order) {
    orderRepo.save(order);                    // DB'ye yaz
    kafkaTemplate.send("orders", event);      // Kafka'ya gönder
    // SORUN: DB commit → Kafka fail → tutarsızlık
    // VEYA: DB rollback → Kafka gönderildi → ghost event
}
```

**Outbox ile çözüm (ama polling gerekir):**
```java
@Transactional
void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("OrderCreated", order)); // aynı transaction
}
// Ayrı servis outbox tablosunu 1sn'de bir polling yapar → Kafka'ya gönderir
// Sorun: Polling DB'ye yük bindirir, gecikme vardır
```

**CDC ile en iyi çözüm:**
```
@Transactional
void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent(...)); // aynı transaction — bitti!
}
// Debezium WAL'ı okur → outbox tablosundaki değişikliği yakalar
// → Kafka'ya push eder (polling yok, gecikme ~ms)
```

---

## Nasıl?

### Veritabanı Transaction Log Nedir?

```
PostgreSQL WAL (Write-Ahead Log):
  Her veri değişikliği önce WAL'a yazılır
  WAL → crash recovery + replikasyon için kullanılır
  Debezium WAL'ı okur (veritabanına yük bindirmez, replikasyon gibi davranır)

MySQL Binlog:
  Statement-based (SQL statement loglanır)
  Row-based (satır değişikliği loglanır — Debezium bunu kullanır)

MongoDB Oplog:
  Tüm write operasyonlarının capped collection log'u
```

```
Normal replikasyon:    DB Primary → WAL → Replica
Debezium:             DB Primary → WAL → Debezium → Kafka
                                          (logical replication slot)
```

### Debezium Mimarisi

```
PostgreSQL Primary
      │
      │ logical replication slot (pg_create_logical_replication_slot)
      ↓
Debezium Connector (Kafka Connect worker içinde çalışır)
      │  Her değişiklik için event üretir:
      │  { "op": "c/u/d/r", "before": {...}, "after": {...}, "source": {...} }
      ↓
Kafka Topic: "db.public.orders"  (server.schema.table formatı)
      │
      ↓
Consumer Servisler (InventoryService, SearchIndexer, Analytics, Cache Invalidator...)
```

**Event yapısı:**
```json
{
  "schema": {...},
  "payload": {
    "before": null,
    "after": {
      "id": "ord-123",
      "customer_id": "cust-456",
      "total": 99.99,
      "status": "CONFIRMED"
    },
    "source": {
      "db": "orders_db",
      "schema": "public",
      "table": "orders",
      "txId": 1234,
      "lsn": 24023128,
      "ts_ms": 1705312800000
    },
    "op": "c",   // c=create, u=update, d=delete, r=read (snapshot)
    "ts_ms": 1705312800100
  }
}
```

**op değerleri:**
```
c → INSERT (create)
u → UPDATE
d → DELETE
r → READ (initial snapshot sırasında mevcut satırlar)
```

---

### PostgreSQL Kurulum

```sql
-- WAL level logical replication aktif et
ALTER SYSTEM SET wal_level = logical;
-- Restart gerekir

-- Replication slot oluştur (Debezium bunu yapar ama kontrol et)
SELECT pg_create_logical_replication_slot('debezium_slot', 'pgoutput');

-- Debezium için kullanıcı izni
CREATE ROLE debezium WITH LOGIN PASSWORD 'secret' REPLICATION;
GRANT USAGE ON SCHEMA public TO debezium;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;
```

**Debezium Connector konfigürasyonu:**
```json
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres.internal",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    "table.include.list": "public.orders,public.outbox_events",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_orders_slot",
    "publication.name": "debezium_publication",
    "snapshot.mode": "initial",
    "heartbeat.interval.ms": "10000",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "type:header:eventType"
  }
}
```

---

### Outbox Pattern — Debezium ile Production Implementasyonu

```sql
-- Outbox tablosu
CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(255) NOT NULL,  -- "Order", "Payment"
    aggregate_id   VARCHAR(255) NOT NULL,  -- "ord-123"
    event_type     VARCHAR(255) NOT NULL,  -- "OrderCreated", "OrderShipped"
    payload        JSONB NOT NULL,
    created_at     TIMESTAMPTZ DEFAULT NOW()
);
```

```java
@Service
class OrderService {
    @Transactional
    void placeOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));

        // Outbox'a yaz — aynı transaction içinde (atomic!)
        outboxRepo.save(OutboxEvent.builder()
            .aggregateType("Order")
            .aggregateId(order.getId().toString())
            .eventType("OrderCreated")
            .payload(objectMapper.valueToTree(OrderCreatedEvent.from(order)))
            .build());

        // Debezium outbox tablosundaki INSERT'i yakalar
        // → Kafka topic: "orders.OrderCreated"
        // → Consumer'lar alır
    }
}
```

**Debezium Outbox Router transform:**
```
outbox_events tablosuna INSERT → Debezium yakaladı →
EventRouter transform:
  payload.aggregate_type = "Order"
  payload.event_type = "OrderCreated"
  →  Kafka topic: "Order" (aggregate_type)
  →  Key: aggregate_id
  →  Value: payload

Sonuç: Her aggregate type için ayrı topic → isteyen subscribe eder
```

---

### Snapshot vs Streaming

```
Debezium başlatıldığında iki aşama:

1. Initial Snapshot:
   - Mevcut tüm satırları okur (READ event, op=r)
   - Büyük tablo → uzun sürer (snapshot.mode ayarlanabilir)
   - snapshot.mode:
     initial          → her başlatmada snapshot (tabloyu kopyala)
     initial_only     → sadece snapshot, stream etme
     never            → snapshot atla, sadece WAL'dan stream et
     when_needed      → slot yoksa snapshot

2. Streaming:
   - Snapshot sonrası WAL'dan gerçek zamanlı okuma
   - LSN (Log Sequence Number) takip eder (offset gibi)
```

---

### Kullanım Senaryoları

```
1. Outbox Pattern (en yaygın):
   Service → DB + Outbox tablosu → Debezium → Kafka

2. Database'ler arası senkronizasyon:
   PostgreSQL (primary) → Debezium → Kafka → Elasticsearch (search index)
                                            → MongoDB (read model)
                                            → Redis (cache invalidation)
                                            → Analytics DB

3. Audit Log:
   Her tablo değişikliği → Debezium → Kafka → Audit storage
   (uygulama kodu değişmez, DB katmanında yakalanır)

4. Cache Invalidation:
   Product tablosu UPDATE → Debezium → Kafka → Cache Consumer → Redis.DEL(productKey)

5. Zero-downtime migration:
   Eski DB → Debezium → Kafka → Yeni DB
   Çift yazma süresi → traffic switch → eski DB kapat
```

---

## Ne zaman?

**Debezium/CDC kullan:**
```
✓ Outbox pattern'i polling olmadan implement etmek
✓ DB değişikliklerini birden fazla sisteme iletmek
✓ Audit log (uygulama kodu değiştirmeden)
✓ Cache invalidation (DB değişince cache'i otomatik invalidate et)
✓ Search index senkronizasyonu (Elasticsearch güncelleme)
✓ Zero-downtime database migration
✓ Microservice'e dönüşüm (monolith'in DB'sini event'e çevir)
```

**Debezium kullanma:**
```
✗ Basit messaging (Kafka/RabbitMQ direkt yeterli)
✗ DB WAL desteği yoksa (bazı managed DB'ler kısıtlar)
✗ Yüksek tablo sayısı ve karmaşık filtering gerektiriyorsa (performans)
✗ Operasyonel karmaşıklığı kabul edemiyorsan (Kafka Connect + Debezium kurulumu)
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Uygulama kodu değişmez | Kafka Connect + Debezium kurulumu karmaşık |
| DB polling yok → düşük gecikme (ms) | WAL/binlog boyutu artar |
| Outbox pattern'ın en iyi implementasyonu | Replication slot disk dolabilir (lag izle) |
| Tüm değişiklikler (before/after) | Snapshot büyük tablolarda uzun sürer |
| Otomatik failover (offset tracking) | Schema değişimi (kolon ekleme) dikkat ister |
| Birden fazla DB destekler (PG, MySQL, Mongo, Oracle...) | Yalnızca Kafka ile değil, Kafka Connect gerekir |
