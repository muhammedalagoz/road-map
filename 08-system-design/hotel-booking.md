# 08l — System Design: Otel/Etkinlik Rezervasyon Sistemi (Airbnb / Biletix)

## Gereksinimler

```
Functional:
  ✓ Müsait oda/koltuk arama (tarih, kapasite filtresi)
  ✓ Rezervasyon oluşturma
  ✓ Çift rezervasyon önleme (overbooking)
  ✓ İptal ve iade
  ✓ Rezervasyon onayı (email/SMS)
  ✗ Ödeme, yorumlar (out of scope)

Non-Functional:
  DAU: 10M
  Search QPS: 10,000/sn (yoğun okuma)
  Booking QPS: 100/sn (yazma az)
  Latency: arama < 200ms, rezervasyon < 3s
  Çift rezervasyon: kesinlikle önlenmeli (zero tolerance)
```

---

## Capacity Estimation

```
Search:
  10M × 10 arama/gün / 86400 ≈ 1,160 QPS (ortalama)
  Peak (tatil sezonu): 10,000 QPS

Booking:
  10M × 1 rezervasyon / 30 gün / 86400 ≈ 4 booking/sn
  Peak: 100 booking/sn (kampanya, konser bileti)

Storage:
  Hotel: 1M otel × 1KB = 1 GB
  Room: 1M × 50 oda = 50M kayıt × 200B = 10 GB
  Reservations: 4 × 86400 × 365 = 126M/yıl × 1KB = 126 GB/yıl
```

---

## High-Level Tasarım

```
         Client
           │
    ┌──────┴──────────────────────────┐
    │           API Gateway           │
    └──────┬──────────────────┬───────┘
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │   Search    │    │  Booking    │
    │   Service   │    │  Service    │
    └──────┬──────┘    └──────┬──────┘
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │Elasticsearch│    │ PostgreSQL  │
    │ (search)    │    │ (inventory) │
    └─────────────┘    └──────┬──────┘
                              │
                       ┌──────▼──────┐
                       │    Redis    │
                       │  (cache +   │
                       │   lock)     │
                       └─────────────┘
```

---

## Müsaitlik Modeli

### Seçenek 1: Pre-computed Availability Table

```sql
-- Her oda/tarih kombinasyonu için bir satır
CREATE TABLE room_availability (
    hotel_id    UUID,
    room_type   VARCHAR(50),
    date        DATE,
    total_rooms INT,
    booked_rooms INT DEFAULT 0,
    price       DECIMAL(10,2),
    PRIMARY KEY (hotel_id, room_type, date)
);

-- Müsait arama:
SELECT * FROM room_availability
WHERE hotel_id IN (SELECT hotel_id FROM hotels WHERE city='Istanbul')
  AND date BETWEEN '2024-12-24' AND '2024-12-27'
  AND (total_rooms - booked_rooms) >= 1  -- müsait var mı?
  AND date = EVERY date in range;  -- tüm geceler müsait mi?

-- Sorun: N gece × M oda tipi = büyük JOIN
-- Avantaj: O(1) availability check, basit güncelleme
```

### Seçenek 2: Booking-Based (Event Sourcing)

```sql
-- Sadece rezervasyonları sakla, availability türetilir
CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    hotel_id       UUID,
    room_type      VARCHAR(50),
    check_in       DATE,
    check_out      DATE,
    guest_id       UUID,
    status         VARCHAR(20)  -- PENDING, CONFIRMED, CANCELLED
);

-- Müsaitlik:
SELECT total_rooms -
  COUNT(CASE WHEN r.status NOT IN ('CANCELLED') THEN 1 END)
FROM rooms r
LEFT JOIN reservations res ON r.hotel_id = res.hotel_id
  AND r.room_type = res.room_type
  AND res.check_in < :checkOut
  AND res.check_out > :checkIn
WHERE r.hotel_id = :hotelId AND r.room_type = :roomType;

-- Avantaj: Rezervasyon tarihi değiştirilirse otomatik müsaitlik güncellenir
-- Dezavantaj: Hesaplama pahalı (her sorguda COUNT)
```

**Seçilen: Seçenek 1 (Pre-computed) + trigger ile güncelleme**

---

## Çift Rezervasyon Önleme

### Kritik Problem: Race Condition

```
Senaryo:
  Saat 14:00:00.001 → Kullanıcı A: "Hotel X, 24 Aralık, 1 oda müsait mi?" → EVET
  Saat 14:00:00.002 → Kullanıcı B: "Hotel X, 24 Aralık, 1 oda müsait mi?" → EVET
  Saat 14:00:00.003 → Kullanıcı A: Rezervasyon oluştur
  Saat 14:00:00.004 → Kullanıcı B: Rezervasyon oluştur
  → İKİ rezervasyon, ama 1 oda kaldı!
```

### Çözüm 1: Pessimistic Locking (DB Lock)

```sql
BEGIN TRANSACTION;

-- Lock: diğer transaction'lar bu satırı kilitli görür, bekler
SELECT total_rooms, booked_rooms
FROM room_availability
WHERE hotel_id = :hotelId AND room_type = :type AND date = :date
FOR UPDATE;  -- ROW LEVEL LOCK

-- Müsait mi?
IF (total_rooms - booked_rooms) >= :requestedRooms THEN
  UPDATE room_availability
  SET booked_rooms = booked_rooms + :requestedRooms
  WHERE hotel_id = :hotelId AND room_type = :type AND date = :date;

  INSERT INTO reservations (...) VALUES (...);
  COMMIT;
ELSE
  ROLLBACK;
  RAISE EXCEPTION 'No rooms available';
END IF;
```

**Avantaj:** Basit, garantili, DB tek kaynak.  
**Dezavantaj:** Yüksek contention → throughput düşer, lock timeout riski.

### Çözüm 2: Optimistic Locking (Version Check)

```sql
-- version kolonu ile
UPDATE room_availability
SET booked_rooms = booked_rooms + 1, version = version + 1
WHERE hotel_id = :hotelId AND date = :date
  AND (total_rooms - booked_rooms) >= 1
  AND version = :expectedVersion;  -- stale check

-- Etkilenen satır = 0 → başka biri önce aldı → RETRY veya hata
IF affected_rows == 0:
  throw new RoomNoLongerAvailableException();
```

**Avantaj:** Lock yok → yüksek throughput.  
**Dezavantaj:** Yoğun contention'da çok retry.

### Çözüm 3: Distributed Lock + DB (Öneri)

```java
// Flash sale / konser bileti gibi yüksek contention senaryosu
String lockKey = "booking-lock:" + hotelId + ":" + roomType + ":" + date;

boolean locked = redisLock.tryLock(lockKey, Duration.ofSeconds(10));
if (!locked) {
    throw new ServiceBusyException("Çok fazla eş zamanlı istek");
}
try {
    // Lock altında availability check + reservation create
    return createReservationInDB(request);
} finally {
    redisLock.unlock(lockKey, requestId);
}
```

---

## Arama (Elasticsearch)

```json
// Hotel indexi
{
  "hotelId": "h-123",
  "name": "Grand Hotel Istanbul",
  "city": "Istanbul",
  "country": "TR",
  "location": { "lat": 41.01, "lon": 28.97 },
  "stars": 5,
  "amenities": ["pool", "spa", "wifi", "parking"],
  "priceFrom": 150.00,
  "availableRoomTypes": ["STANDARD", "DELUXE"],
  "rating": 4.7,
  "reviewCount": 1420
}

// Arama: İstanbul, 2 kişi, 24-27 Aralık, havuzlu, max 300 TL
GET /hotels/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"city": "Istanbul"}},
        {"terms": {"amenities": ["pool"]}},
        {"range": {"priceFrom": {"lte": 300}}}
      ],
      "filter": [
        {"term": {"availableRoomTypes": "STANDARD"}}
      ]
    }
  },
  "sort": [
    {"rating": "desc"},
    {"priceFrom": "asc"}
  ]
}
```

**Availability'yi Elasticsearch'te canlı tutma:**
```
room_availability tablosu değişti → Debezium CDC
→ Kafka → Elasticsearch updater → index güncelle

Güncelleme sıklığı: her rezervasyon/iptal için anlık
availableRoomTypes: müsait oda tiplerinin listesi
```

---

## Rezervasyon Akışı

```
1. Kullanıcı "Rezervasyon Yap" → POST /bookings
   {hotelId, roomType, checkIn, checkOut, guestCount}

2. Booking Service:
   a. Müsaitlik kontrolü (PostgreSQL SELECT FOR UPDATE)
   b. Ücret hesapla (gece sayısı × fiyat × surge varsa)
   c. Pending rezervasyon oluştur (status=PENDING, expires 15 dk)
   d. Ödeme servisine yönlendir

3. Kullanıcı ödeme yapar:
   Ödeme OK → reservation status=CONFIRMED
   Ödeme FAIL → reservation status=CANCELLED → müsaitlik geri ver

4. Pending süresi dolan rezervasyonlar (scheduler):
   Her dakika: DELETE FROM reservations WHERE status=PENDING AND expires_at < NOW()
   → room_availability güncelle (booked_rooms azalt)

5. Onay email/SMS → Notification Service
```

---

## Veri Modeli

```sql
CREATE TABLE hotels (
    hotel_id    UUID PRIMARY KEY,
    name        VARCHAR(200),
    city        VARCHAR(100),
    address     TEXT,
    stars       INT,
    latitude    DECIMAL(10,7),
    longitude   DECIMAL(10,7)
);

CREATE TABLE room_types (
    hotel_id    UUID,
    room_type   VARCHAR(50),
    total_rooms INT,
    base_price  DECIMAL(10,2),
    max_guests  INT,
    PRIMARY KEY (hotel_id, room_type)
);

CREATE TABLE room_availability (
    hotel_id    UUID,
    room_type   VARCHAR(50),
    date        DATE,
    booked_rooms INT DEFAULT 0,
    price       DECIMAL(10,2),     -- dynamic pricing
    version     BIGINT DEFAULT 0,  -- optimistic lock
    PRIMARY KEY (hotel_id, room_type, date)
);

CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    hotel_id       UUID,
    room_type      VARCHAR(50),
    guest_id       UUID,
    check_in       DATE,
    check_out      DATE,
    total_nights   INT,
    total_price    DECIMAL(10,2),
    status         VARCHAR(20),    -- PENDING, CONFIRMED, CANCELLED
    expires_at     TIMESTAMP,      -- PENDING timeout
    created_at     TIMESTAMP,
    INDEX idx_hotel_dates (hotel_id, room_type, check_in, check_out),
    INDEX idx_guest (guest_id),
    INDEX idx_pending_expires (status, expires_at)
);
```

---

## Trade-off Özeti

| Karar | Seçenek | Gerekçe |
|-------|---------|---------|
| Locking | Pessimistic (FOR UPDATE) | Otel: düşük contention, garanti şart |
| Flash sale | Distributed lock (Redis) | Yüksek contention, hız kritik |
| Search | Elasticsearch | Full-text, geo, facet filter |
| Availability sync | CDC (Debezium) | ES'yi gerçek zamanlı güncelle |
| Pending timeout | Scheduler | Garanti edilmiş temizlik |
