# 08l — System Design: Otel/Etkinlik Rezervasyon Sistemi (Airbnb / Booking.com)

## Gereksinimler

```
Functional:
  ✓ Müsait oda/koltuk arama (tarih, konum, kapasite filtresi)
  ✓ Coğrafi (harita) bazlı arama
  ✓ Rezervasyon oluşturma
  ✓ Çift rezervasyon önleme (overbooking yok)
  ✓ İptal ve iade (politikaya göre hesaplama)
  ✓ Rezervasyon değiştirme (tarih / oda tipi)
  ✓ Rezervasyon onayı (email/SMS)
  ✓ Dinamik fiyatlandırma (sezon, doluluk)
  ✓ Bekleme listesi (tam dolu otelde)
  ✗ Ödeme işleme (out of scope)
  ✗ Yorum/değerlendirme (out of scope)

Non-Functional:
  DAU: 10M
  Search QPS: 10,000/sn (yoğun okuma, yüksek öncelik)
  Booking QPS: 100/sn (yazma az ama kritik)
  Latency: arama < 200ms P99, rezervasyon < 3s P99
  Çift rezervasyon: kesinlikle önlenmeli (zero tolerance)
  Availability: 99.99%
  Pending timeout garantisi: 15 dk ± 30sn tolerans
```

---

## Capacity Estimation

```
Search:
  10M DAU × 10 arama/gün / 86400 ≈ 1,160 QPS ortalama
  Peak (tatil sezonu, akşam 20:00): 10,000 QPS
  Bandwidth: 10,000 × 5 KB (arama sonuçları) = 50 MB/sn → CDN ile dağıt

Booking:
  10M DAU × 1 rezervasyon / 30 gün / 86400 ≈ 4 booking/sn ortalama
  Peak (uçuş indirimi duyurusu, konser bileti): 100 booking/sn
  Critical path: DB write → lock → unlock → 100/sn yönetilebilir

Storage:
  Otel: 1M × 1 KB = 1 GB (PostgreSQL)
  Oda tipi: 1M otel × 10 tip = 10M × 200B = 2 GB
  Müsaitlik tablosu: 10M oda × 365 gün × 50B = 182 GB/yıl (pre-computed)
  Rezervasyon: 4/sn × 86400 × 365 = 126M/yıl × 1KB = 126 GB/yıl
  Elasticsearch index: 1M otel × 2 KB = 2 GB

Cache (Redis):
  Hot otel availability (İstanbul merkez, sonraki 30 gün):
    5,000 otel × 10 tip × 30 gün × 20B = 30 MB → çok ucuz
  Tüm aktif oteller Redis'te: 1M × 30 gün × 10 tip × 20B = 6 GB
```

---

## High-Level Mimari

```
              Client (Web / Mobile)
                       │
               ┌───────▼────────┐
               │   API Gateway  │  ← auth, rate limit, routing
               └───────┬────────┘
                        │
     ┌──────────┬────────┼───────────┬──────────┐
     ▼          ▼        ▼           ▼          ▼
┌─────────┐ ┌──────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Search  │ │Hotel │ │Booking │ │Pricing │ │Notif.  │
│ Service │ │Mgmt  │ │Service │ │Service │ │Service │
└────┬────┘ └──┬───┘ └────┬───┘ └────┬───┘ └────┬───┘
     │          │          │           │          │
     ▼          ▼          ▼           ▼          ▼
┌─────────┐ ┌──────┐ ┌──────────┐ ┌──────┐ ┌────────┐
│Elastic  │ │Postgres│ PostgreSQL │ │Redis │ │Kafka   │
│search   │ │(hotels)│(reserv.,  │ │(lock,│ │(events)│
│         │ │        │ avail.)   │ │cache)│ │        │
└─────────┘ └──────┘ └──────────┘ └──────┘ └────────┘
                          │
                   ┌──────▼──────┐
                   │   Debezium  │  → Kafka → ES Updater
                   │    (CDC)    │          → Cache Invalidator
                   └─────────────┘

Async Workers (Kafka consumers):
  Pending Expiry Worker:  süresi dolan rezervasyonları temizle
  Notification Worker:    e-posta / SMS gönder
  Waitlist Worker:        iptal → bekleme listesini kontrol et
  Price Sync Worker:      dinamik fiyat → ES güncelle
```

---

## Müsaitlik Modeli

### Seçenek 1: Pre-computed Availability Table (Seçilen)

```sql
-- Her oda tipi + tarih kombinasyonu için bir satır
CREATE TABLE room_availability (
    hotel_id     UUID,
    room_type    VARCHAR(50),
    date         DATE,
    total_rooms  INT NOT NULL,
    booked_rooms INT DEFAULT 0,
    price        DECIMAL(10,2),    -- o gün için dinamik fiyat
    version      BIGINT DEFAULT 0, -- optimistic lock
    PRIMARY KEY (hotel_id, room_type, date)
);

-- Çok gecelik müsaitlik: her gece için kontrol et
-- 24-27 Aralık = 24, 25, 26 Aralık geceleri (3 gece)
SELECT date, (total_rooms - booked_rooms) AS available
FROM room_availability
WHERE hotel_id = :hotelId
  AND room_type = :roomType
  AND date >= :checkIn AND date < :checkOut
  AND (total_rooms - booked_rooms) >= :requestedRooms;
-- Dönen satır sayısı = (checkOut - checkIn) gün sayısı olmalı
-- Herhangi bir gece müsait değilse → "o tarihte dolu"

-- Avantaj: O(1) availability check, basit güncelleme, fiyat snapshot
-- Dezavantaj: Büyük tablo (1M otel × 10 tip × 365 gün = 3.65B satır)
-- Çözüm: Sadece sonraki 365 günü sakla → eski satırlar arşiv/sil
```

### Seçenek 2: Booking-Based (Event Sourcing)

```sql
-- Sadece rezervasyonları sakla, availability türetilir
-- Avantaj: Rezervasyon değiştirilirse otomatik müsaitlik güncellenir
-- Dezavantaj: Her availability sorgusunda COUNT → 10K QPS × JOIN = çok ağır
-- Sonuç: Search Service için uygun değil. Booking Service'te doğrulama için kullanılabilir.

SELECT total_rooms - COUNT(*) AS available
FROM room_types rt
LEFT JOIN reservations res ON rt.hotel_id = res.hotel_id
  AND rt.room_type = res.room_type
  AND res.check_in < :checkOut
  AND res.check_out > :checkIn
  AND res.status NOT IN ('CANCELLED')
WHERE rt.hotel_id = :hotelId AND rt.room_type = :roomType
GROUP BY rt.total_rooms;
```

### Hibrit Yaklaşım (Gerçek Dünya)

```
Arama (yüksek QPS): Pre-computed table + Redis cache → hızlı.
Rezervasyon (düşük QPS): Pre-computed + SELECT FOR UPDATE → güvenli.
Doğrulama: Her rezervasyonda booking-based COUNT → çift güvence.
```

---

## Arama (Elasticsearch + Geo)

### Otel Index Şeması

```json
{
  "hotelId":    "h-123",
  "name":       "Grand Hotel Istanbul",
  "city":       "Istanbul",
  "country":    "TR",
  "location":   { "lat": 41.0082, "lon": 28.9784 },
  "stars":      5,
  "amenities":  ["pool", "spa", "wifi", "parking", "gym", "breakfast"],
  "priceFrom":  150.00,
  "priceCalendar": {
    "2024-12-24": { "STANDARD": 180.0, "DELUXE": 280.0 },
    "2024-12-25": { "STANDARD": 210.0, "DELUXE": 320.0 }
  },
  "availableRoomTypes": ["STANDARD", "DELUXE"],
  "rating":     4.7,
  "reviewCount": 1420,
  "tags":       ["merkezi konum", "boğaz manzarası", "evcil hayvan dostu"],
  "cancelPolicy": "FREE_CANCELLATION"
}
```

### Arama Sorgusu (Konum + Filtre + Sıralama)

```json
// İstanbul, Taksim'e 5 km içinde, havuzlu, max 3 kişi, 24-27 Aralık
GET /hotels/_search
{
  "query": {
    "bool": {
      "must": [
        { "term":  { "availableRoomTypes": "DELUXE" }},
        { "range": { "priceFrom": { "lte": 500 }}},
        { "terms": { "amenities": ["pool"] }}
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": { "lat": 41.0368, "lon": 28.9850 }
          }
        },
        { "range": { "stars": { "gte": 4 }}}
      ],
      "should": [
        { "term": { "cancelPolicy": "FREE_CANCELLATION" }},
        { "term": { "amenities": "breakfast" }}
      ],
      "minimum_should_match": 0
    }
  },
  "sort": [
    { "_score": "desc" },
    {
      "_geo_distance": {
        "location": { "lat": 41.0368, "lon": 28.9850 },
        "order": "asc",
        "unit": "km"
      }
    },
    { "rating": "desc" }
  ],
  "aggs": {
    "by_stars":       { "terms": { "field": "stars" }},
    "by_amenities":   { "terms": { "field": "amenities", "size": 20 }},
    "price_range":    { "histogram": { "field": "priceFrom", "interval": 100 }},
    "cancel_policy":  { "terms": { "field": "cancelPolicy" }}
  },
  "from": 0,
  "size": 20
}
```

### Harita Görünümü (Map Search)

```
Kullanıcı haritayı kaydırıyor → viewport (sol-alt, sağ-üst koordinatları) değişiyor.

GET /hotels/map
{ "topLeft": {"lat": 41.1, "lon": 28.8},
  "bottomRight": {"lat": 40.9, "lon": 29.1},
  "checkIn": "2024-12-24", "checkOut": "2024-12-27" }

ES geo_bounding_box query:
{
  "filter": {
    "geo_bounding_box": {
      "location": {
        "top_left":     { "lat": 41.1, "lon": 28.8 },
        "bottom_right": { "lat": 40.9, "lon": 29.1 }
      }
    }
  }
}

Sonuç: pin listesi (hotel_id, lat, lon, priceFrom, stars)
Kullanıcı pin'e tıklar → detay modal → availability check (anlık, DB'den).

Zoom out (çok otel):
  Cluster aggregation: ES geo_centroid / geo_grid agg.
  Küçük haritada binlerce pin yerine → gruplar (cluster) göster.
  "Bu bölgede 47 otel, 120 TL'den başlayan fiyatlarla."
```

### ES ↔ PostgreSQL Senkronizasyonu

```
Değişiklik tetikleyen olaylar:
  Rezervasyon oluşturuldu → availableRoomTypes listesi değişti.
  Rezervasyon iptal edildi → tekrar müsait.
  Fiyat güncellendi → priceFrom değişti.
  Yeni oda tipi eklendi (hotel management).

CDC Akışı:
  PostgreSQL (room_availability, room_types) → Debezium → Kafka
  ES Updater (consumer):
    partial update: POST /hotels/_update/{hotelId}
    {"doc": {"priceFrom": 180, "availableRoomTypes": ["STANDARD", "DELUXE"]}}
  
  Gecikme: ~1-2 sn (CDC lag) → kabul edilebilir (arama stale ama rezervasyon DB'den doğrular).

ES'te availability saklama stratejisi:
  availableRoomTypes: müsait oda tiplerinin listesi (STANDARD, DELUXE...).
  Dar tarih aralığı için: priceFrom (minimum fiyat, seçilen tarihler için).
  priceCalendar: önümüzdeki 90 gün detaylı fiyat takvimi.
  Kesin availability: ES'ten değil → DB'den (rezervasyon öncesi).
```

---

## Dinamik Fiyatlandırma

```
Fiyat = base_price × demand_multiplier × season_multiplier × occupancy_multiplier

demand_multiplier:
  Arama çok, rezervasyon az → fiyat artır (talep yüksek, arz az).
  Arama az, çok oda boş → fiyat düşür (doluluk artırma).
  Hesap: son 1 saatteki arama sayısı / ortalama arama (moving avg).
  Eşik: >2x → 1.2 multiplier; >5x → 1.5 multiplier.

season_multiplier (önceden tanımlı):
  Normal: 1.0
  Yılbaşı (24-31 Aralık): 1.8
  Yaz (Temmuz-Ağustos): 1.5
  Bayram tatili: 2.0
  Düşük sezon (Ocak-Şubat): 0.8

occupancy_multiplier:
  Doluluk % < 50  → 0.9 (boş oda sat)
  Doluluk % 50-70 → 1.0 (normal)
  Doluluk % 70-85 → 1.2 (yüksek talep)
  Doluluk % 85-95 → 1.5 (neredeyse dolu)
  Doluluk % > 95  → 2.0 (son oda premium)

Son dakika indirimi:
  Check-in 24 saat kaldı, %30 oda boşsa → %20 indirim (boş geçirmektense).
  Scheduler: her saat → doluluk hesapla → fiyat güncelle.

Early bird:
  60+ gün önceden → %15 indirim (ödeme garantisi için).
  30 gün → normal fiyat.

Fiyat güncelleme akışı:
  Pricing Service (Kafka consumer + cron):
    her saat → doluluk hesapla → yeni fiyat → room_availability güncelle.
    Güncelleme → CDC → ES güncelleme (priceFrom, priceCalendar).
```

---

## Çift Rezervasyon Önleme

### Kritik Problem: Race Condition

```
Senaryo:
  14:00:00.001 → Kullanıcı A: "Hotel X, 24 Aralık, 1 oda müsait mi?" → EVET
  14:00:00.002 → Kullanıcı B: "Hotel X, 24 Aralık, 1 oda müsait mi?" → EVET
  14:00:00.003 → Kullanıcı A: Rezervasyon oluştur
  14:00:00.004 → Kullanıcı B: Rezervasyon oluştur
  → İKİ rezervasyon, ama 1 oda kaldı → overbooking!
```

### Çözüm 1: Pessimistic Locking (DB Lock) — Normal Otel İçin

```sql
BEGIN TRANSACTION;

-- Tüm geceleri lock'la (multi-night booking için döngü)
SELECT hotel_id, room_type, date, total_rooms, booked_rooms
FROM room_availability
WHERE hotel_id = :hotelId
  AND room_type = :roomType
  AND date >= :checkIn AND date < :checkOut
ORDER BY date           -- deadlock önleme: her zaman aynı sırada lock
FOR UPDATE;             -- ROW LEVEL LOCK → diğer tx bekler

-- Tüm gecelerde müsait mi?
-- (uygulama katmanında kontrol, SQL'de HAVING ile de yapılabilir)
IF herhangi_bir_gece_dolu THEN
  ROLLBACK;
  RAISE EXCEPTION 'No rooms available for selected dates';
END IF;

-- Tüm geceleri güncelle
UPDATE room_availability
SET booked_rooms = booked_rooms + :requestedRooms
WHERE hotel_id = :hotelId
  AND room_type = :roomType
  AND date >= :checkIn AND date < :checkOut;

INSERT INTO reservations (reservation_id, hotel_id, room_type,
  guest_id, check_in, check_out, total_nights, total_price,
  status, expires_at)
VALUES (gen_random_uuid(), :hotelId, :roomType,
  :guestId, :checkIn, :checkOut, :nights, :price,
  'PENDING', NOW() + INTERVAL '15 minutes');

COMMIT;
```

```
Avantaj: Basit, garantili, DB tek kaynak, çok gecelik için güvenli.
Dezavantaj: Peak'te contention → throughput düşer → lock timeout.
Uygun: Normal otel rezervasyonu (100 booking/sn → yeterli).
```

### Çözüm 2: Optimistic Locking — Düşük Contention

```sql
-- Her gece için ayrı optimistic update (batch)
UPDATE room_availability
SET booked_rooms = booked_rooms + :rooms,
    version = version + 1
WHERE hotel_id = :hotelId
  AND room_type = :roomType
  AND date = :date
  AND (total_rooms - booked_rooms) >= :rooms
  AND version = :expectedVersion;

-- Etkilenen satır = 0 → başka biri önce aldı veya stale read → RETRY

-- Çok gecelik: 3 gün → 3 ayrı UPDATE → hepsi başarılı olmalı.
-- Kısmi başarı (1. gün OK, 2. gün fail) → 1. günü geri al → compensating.
-- Karmaşıklık artar → pessimistic genellikle daha temiz.
```

### Çözüm 3: Redis Distributed Lock — Flash Sale / Konser Bileti

```java
// Çok yüksek contention: konser, büyük kampanya
// Aynı oda tipine saniyede 10,000 istek → DB lock yetmez

String lockKey = "avail-lock:" + hotelId + ":" + roomType + ":" + checkIn;
String requestId = UUID.randomUUID().toString(); // idempotency için

boolean locked = redisTemplate.opsForValue()
    .setIfAbsent(lockKey, requestId, Duration.ofSeconds(10));

if (!locked) {
    // Hız sınırı aşıldı → kullanıcıya "Şu an yoğunuz, biraz bekleyin"
    throw new ServiceBusyException("Çok fazla eş zamanlı istek");
}

try {
    // Lock altında: sadece bir thread çalışıyor
    AvailabilityResult result = checkAndReserveInDB(hotelId, roomType, dates, rooms);
    return result;
} finally {
    // Lock'ı sadece alan kişi serbest bıraksın (requestId ile)
    String script = "if redis.call('get', KEYS[1]) == ARGV[1] " +
                    "then return redis.call('del', KEYS[1]) " +
                    "else return 0 end";
    redisTemplate.execute(new DefaultRedisScript<>(script, Long.class),
        List.of(lockKey), requestId);
}
```

```
Avantaj: DB'yi yüksek concurrent istekten korur, hız gerektirir.
Dezavantaj: Redis SPOF (Sentinel ile gider), Redlock daha karmaşık.
Ne zaman: kampanya, konser bileti, flash sale.
```

### Çok Gecelik Rezervasyon (Multi-Night) Özel Durumu

```
3 gecelik rezervasyon: 24, 25, 26 Aralık geceleri.
Her gece ayrı kayıt → hepsini atomik olarak kontrol et ve güncelle.

Deadlock önleme (çok thread aynı geceleri kilitliyorsa):
  Her zaman tarih sırasına göre lock: 24 → 25 → 26.
  Thread A: 24, 25, 26 sırasında.
  Thread B: 24, 25, 26 sırasında. (Rastgele sıra olsaydı: A=25 alır, B=24 alır → deadlock)

Kısmi müsaitlik:
  24 ve 25 Aralık müsait, 26 Aralık dolu → tüm rezervasyon reddedilir.
  Kullanıcıya göster: "26 Aralık için oda kalmamış. 23-25 Aralık arası müsait."
  Alternatif öneri: Search Service → alternatif oteller veya alternatif tarihler.
```

---

## Rezervasyon Akışı (Detaylı)

```
1. Kullanıcı tarih ve otel seçer → Müsaitlik ön kontrol (Redis cache):
   GET availability:{hotelId}:{roomType}:{date} → hızlı "boş/dolu" göster.
   Redis miss → DB → cache'e yaz (TTL: 30 sn, sık değişiyor).

2. "Rezervasyon Yap" tıklama → POST /bookings
   { hotelId, roomType, checkIn, checkOut, guestCount, rateCode }

3. Booking Service:
   a. Rate plan seç (rateCode: FLEXIBLE, NON_REFUNDABLE, EARLY_BIRD).
   b. Her gece için fiyat snapshot al (o anki dinamik fiyat).
   c. İptal politikasını belirle (rate plan'a göre).
   d. Pessimistic lock ile availability kontrol + booked_rooms güncelle.
   e. PENDING reservation oluştur (expires_at = NOW() + 15 dk).
   f. Kafka event: ReservationPending {reservationId, guestId, ...}

4. Ödeme (Payment Service, out of scope):
   Ödeme OK:
     → Booking Service: PUT /bookings/{id}/confirm
     → status = CONFIRMED
     → Kafka: ReservationConfirmed

   Ödeme FAIL / Timeout:
     → status = PAYMENT_FAILED
     → booked_rooms azalt (availability geri ver)
     → Kafka: ReservationCancelled

5. PENDING timeout (15 dk):
   Pending Expiry Worker → her dakika çalışır:
     SELECT * FROM reservations WHERE status='PENDING' AND expires_at < NOW()
     → status = EXPIRED → booked_rooms azalt → waitlist kontrol et.
   
   Redis ZSET alternatifi (daha verimli):
     Rezervasyon oluşunca: ZADD pending_reservations {expires_at_unix} {reservationId}
     Worker: ZRANGEBYSCORE pending_reservations -inf {now_unix} → süresi dolmuşları al.
     DB güncelle → ZREM → boiled room geri ver.

6. Onay bildirimi:
   Kafka: ReservationConfirmed → Notification Worker:
     E-posta: onay, QR kodu, check-in bilgisi, iptal linki.
     SMS: özet (oda no, tarih, toplam tutar).
     Push: mobil uygulama bildirimi.
   48 saat önce hatırlatıcı (ayrı Kafka delayed message).
```

---

## Rezervasyon Değiştirme (Modification)

```
Kullanıcı rezervasyonu değiştirmek istiyor: 24-27 Aralık → 25-28 Aralık.

Availability delta hesabı:
  Eski geceler: 24, 25, 26 Aralık.
  Yeni geceler: 25, 26, 27 Aralık.
  Serbest bırakılacak: 24 Aralık → booked_rooms--.
  Rezerve edilecek: 27 Aralık → booked_rooms++ (müsait olmalı!).
  Değişmeyenler: 25, 26 Aralık → dokunma.

Transaction:
  BEGIN;
  SELECT ... FROM room_availability WHERE date = '2024-12-27' FOR UPDATE;
  -- 27 Aralık müsait mi?
  UPDATE room_availability SET booked_rooms = booked_rooms + 1 WHERE date = '2024-12-27';
  UPDATE room_availability SET booked_rooms = booked_rooms - 1 WHERE date = '2024-12-24';
  UPDATE reservations SET check_in = '2024-12-25', check_out = '2024-12-28',
    total_nights = 3, total_price = :newPrice WHERE reservation_id = :id;
  COMMIT;

Fiyat farkı:
  Yeni fiyat > eski fiyat → kullanıcı fark öder.
  Yeni fiyat < eski fiyat → iade (rate plan'a göre).
  NON_REFUNDABLE plan: değişiklikte iade yapılmaz, fark tahsil edilir.

Rate plan değişikliği:
  FLEXIBLE → NON_REFUNDABLE değişikliği: fiyat düşer, iadelik hakları kaybolur.
  Kullanıcı onaylamalı: "Değiştirirseniz ücretsiz iptal hakkınızı kaybedersiniz."
```

---

## İptal Politikaları ve İade

```
Rate Plan Türleri:
  FLEXIBLE:
    Check-in'den 24 saat öncesine kadar ücretsiz iptal.
    24 saatten az: 1 gece tutulur, kalan iade.
    Fiyat: en yüksek.

  MODERATE:
    5 gün öncesine kadar ücretsiz.
    3-5 gün: %50 iade.
    3 günden az: iade yok.

  NON_REFUNDABLE:
    Hiç iade yok (ama daha ucuz → %15-20 indirim).
    İptal edilse bile ödeme alınır.
    Force majeure (doğal afet): özel değerlendirme.

  EARLY_BIRD:
    60+ gün öncesi rezervasyon → %15 indirim.
    İptal: check-in 30 gün öncesine kadar %80 iade.

İade Hesabı (servis):
  cancellation_date = NOW()
  days_to_checkin = checkIn - cancellation_date

  switch(rate_plan):
    FLEXIBLE:
      days > 1 → full_refund
      days <= 1 → total_price - one_night_price

    MODERATE:
      days > 5 → full_refund
      2 < days <= 5 → total_price × 0.5
      days <= 2 → 0

    NON_REFUNDABLE → 0

  Vergi iadesi: vergiler her zaman iade edilir (ülke mevzuatı).
  Ödeme iadesi: Payment Service → kredi kartına 3-5 iş günü.

İptal tetikleyen availability:
  reservation status = CANCELLED → booked_rooms azalt.
  Waitlist varsa: Waitlist Worker → ilk bekleyene bildirim.
```

---

## Bekleme Listesi (Waitlist)

```
Otel tam dolu → kullanıcı "Beni Bilgilendir" diyebilir.

waitlist tablosu:
  (waitlist_id, hotel_id, room_type, check_in, check_out,
   guest_id, requested_at, notified_at, status)

İptal gerçekleşti → Kafka: ReservationCancelled {hotelId, roomType, dates}

Waitlist Worker (consumer):
  1. İlgili tarihler için bekleyenleri sıra ile al.
  2. İlk N kişiye bildirim gönder (N = müsait hale gelen oda sayısı).
  3. "24 Aralık için istediğiniz oda müsait oldu. 10 dk içinde rezerve etmezseniz başkasına gösteriyoruz."
  4. 10 dk içinde rezervasyon → waitlist = CONVERTED.
  5. 10 dk geçti → bir sonraki waitlist kullanıcısına bildirim.

Kuyruğun öncelik sırası:
  Önce: daha erken kayıt.
  Alternatif: en uzun süredir bekleyen (fairness).
  Premium üyelere öncelik (business model kararı).
```

---

## Overbooking Stratejisi (Bilinçli Fazla Satış)

```
Havayolları ve bazı oteller bilerek fazla satar:
  %5-10 iptal / no-show oranı tarihsel veri → fazla bu kadar sat.
  Amaç: doluluk oranını %100 yapmak.

Risk:
  Herkes gelirse → biri odası yok → telafi gerekli.
  Telafi: üst kategori oda yükseltme (upgrade), yakın otele transfer, tazminat.

Implementasyon (opsiyonel özellik):
  total_rooms: fiziksel oda sayısı.
  sellable_rooms: total_rooms × overbooking_rate (ör. 1.05).
  booked_rooms: sellable_rooms'a kadar satılabilir.

  Eğer overbooking aktifse:
    IF booked_rooms < sellable_rooms → satış yap.
    IF booked_rooms >= sellable_rooms → dolu.

Biz bu sistemde overbooking YAPMIYOR'uz (zero-tolerance):
  booked_rooms < total_rooms → koşul.
  Daha basit, müşteri memnuniyeti öncelikli.
```

---

## Veri Modeli (Genişletilmiş)

```sql
CREATE TABLE hotels (
    hotel_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name          VARCHAR(200) NOT NULL,
    city          VARCHAR(100) NOT NULL,
    country       CHAR(2),
    address       TEXT,
    stars         INT CHECK (stars BETWEEN 1 AND 5),
    latitude      DECIMAL(10,7),
    longitude     DECIMAL(10,7),
    check_in_time TIME DEFAULT '14:00',
    check_out_time TIME DEFAULT '11:00',
    amenities     TEXT[],      -- ["pool", "spa", "wifi", "parking"]
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE room_types (
    hotel_id      UUID REFERENCES hotels(hotel_id),
    room_type     VARCHAR(50),
    total_rooms   INT NOT NULL,
    max_guests    INT NOT NULL,
    base_price    DECIMAL(10,2) NOT NULL,
    description   TEXT,
    amenities     TEXT[],
    images        TEXT[],
    PRIMARY KEY (hotel_id, room_type)
);

CREATE TABLE room_availability (
    hotel_id      UUID,
    room_type     VARCHAR(50),
    date          DATE,
    total_rooms   INT NOT NULL,
    booked_rooms  INT DEFAULT 0 CHECK (booked_rooms >= 0),
    price         DECIMAL(10,2) NOT NULL,
    version       BIGINT DEFAULT 0,
    PRIMARY KEY (hotel_id, room_type, date),
    CONSTRAINT no_overbooking CHECK (booked_rooms <= total_rooms)
);

CREATE INDEX idx_avail_search ON room_availability (hotel_id, room_type, date, booked_rooms);

CREATE TABLE rate_plans (
    plan_code         VARCHAR(30) PRIMARY KEY,  -- FLEXIBLE, MODERATE, NON_REFUNDABLE
    name              VARCHAR(100),
    price_multiplier  DECIMAL(4,3) DEFAULT 1.0,  -- NON_REFUNDABLE = 0.85
    cancel_policy     JSONB,   -- {"free_until_days": 1, "penalty_pct": 100}
    is_refundable     BOOLEAN DEFAULT TRUE
);

CREATE TABLE reservations (
    reservation_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hotel_id         UUID REFERENCES hotels(hotel_id),
    room_type        VARCHAR(50),
    guest_id         UUID NOT NULL,
    check_in         DATE NOT NULL,
    check_out        DATE NOT NULL,
    total_nights     INT NOT NULL,
    rooms_count      INT DEFAULT 1,
    rate_plan        VARCHAR(30) REFERENCES rate_plans(plan_code),
    price_per_night  DECIMAL(10,2)[],  -- her gece ayrı fiyat snapshot
    total_price      DECIMAL(10,2) NOT NULL,
    tax_amount       DECIMAL(10,2) DEFAULT 0,
    status           VARCHAR(20) DEFAULT 'PENDING'
                     CHECK (status IN ('PENDING','CONFIRMED','MODIFIED',
                                       'CANCELLED','EXPIRED','PAYMENT_FAILED')),
    expires_at       TIMESTAMPTZ,           -- PENDING → 15 dk
    confirmed_at     TIMESTAMPTZ,
    cancelled_at     TIMESTAMPTZ,
    cancel_reason    TEXT,
    refund_amount    DECIMAL(10,2),
    idempotency_key  VARCHAR(100) UNIQUE,   -- duplicate submission önleme
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    updated_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reservation_guest   ON reservations (guest_id, created_at DESC);
CREATE INDEX idx_reservation_hotel   ON reservations (hotel_id, check_in, check_out);
CREATE INDEX idx_reservation_pending ON reservations (status, expires_at)
  WHERE status = 'PENDING';

CREATE TABLE waitlist (
    waitlist_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hotel_id      UUID REFERENCES hotels(hotel_id),
    room_type     VARCHAR(50),
    check_in      DATE,
    check_out     DATE,
    guest_id      UUID NOT NULL,
    requested_at  TIMESTAMPTZ DEFAULT NOW(),
    notified_at   TIMESTAMPTZ,
    status        VARCHAR(20) DEFAULT 'WAITING'  -- WAITING, NOTIFIED, CONVERTED, EXPIRED
);

CREATE INDEX idx_waitlist_lookup ON waitlist (hotel_id, room_type, check_in, status, requested_at);
```

---

## Availability Cache (Redis)

```
Arama QPS: 10,000/sn → DB'ye her sorguda gitme.
Redis cache: otel-oda-tarih bazlı basit availability.

Cache yapısı:
  Key: avail:{hotelId}:{roomType}:{date}
  Value: available_count (integer)
  TTL: 30 sn (sık rezervasyon olan tarihlerde kısa TTL)

Cache flow:
  Arama: Redis'ten available_count bak.
    >= 1 → "Müsait" göster.
    = 0  → "Dolu" göster.
    Key yok (miss) → DB'den al → cache'e yaz.
  
  Rezervasyon: DB'yi güncelle → Redis key sil (invalidate).
  İptal: DB'yi güncelle → Redis key sil.

Neden DELETE (invalidate), UPDATE değil:
  DB transaction ve cache sync aynı anda olmuyor.
  DB commit başarılı → Redis update crash → tutarsızlık.
  Cache-aside (lazy): invalidate → bir sonraki istek DB'den gelir → doğru.

Özel durum (flash sale):
  Cache'den "müsait" gözüktü → kullanıcı tıkladı → DB'de dolu çıktı.
  Kullanıcı: "Müsait diyordu, neden dolu?"
  Çözüm: arama sayfasında "Fiyatlar ve müsaitlik değişebilir" notu.
          Rezervasyon anında DB doğrulaması → güvenilir.

Tüm otellerin özet cache'i:
  Hash: hotel_avail_summary:{date}
    HSET hotel_avail_summary:2024-12-24 "h-123" "3"  (3 müsait oda)
    HSET hotel_avail_summary:2024-12-24 "h-456" "0"  (dolu)
  Arama sonuçları için hızlı "dolu/müsait" filtreleme.
```

---

## Olası Sorunlar ve Çözümleri

### 1. Race Condition — İki Kullanıcı Aynı Son Odayı Aldı

```
Sorun:
  Hotel X, 24 Aralık: 1 oda kaldı.
  Kullanıcı A ve B aynı anda Redis cache'i okudu → ikisi de "1 oda var" gördü.
  İkisi de "Rezervasyon Yap"a tıkladı → ikisi de Booking Service'e ulaştı.
  SELECT FOR UPDATE bir transaction aldı, diğeri bekledi.
  İlk transaction: booked_rooms = 1 → total_rooms = 1 → dolu → COMMIT.
  İkinci transaction: booked_rooms artık 1 = total_rooms → "müsait değil" → ROLLBACK.
  Sonuç: doğru! İkinci kullanıcıya "Üzgünüz, az önce doldu" mesajı.

Bu senaryoda sistem doğru çalışıyor, ama kullanıcı deneyimi kötü:
  Kullanıcı B "müsait" gördü, ödeme sayfasına gitti, "dolu" mesajı aldı.

İyileştirmeler:
  a) Optimistic UI lock (15 dk):
     Kullanıcı checkout sayfasına girince: soft reserve yap (PENDING oluştur).
     15 dk o kullanıcıya "tutuldu" → ya ödeme yapar ya tutuklama sona erer.
     Diğer kullanıcılar: o oda sayılmaz → gerçekçi müsaitlik.

  b) Arama sonucunda "sadece X oda kaldı!" uyarısı:
     available_count <= 3 → "Son 2 oda!" banner.
     Kullanıcı acele eder → daha az "görmek" ile "bulunmamak" arasındaki fark.

  c) Waitlist otomatik öneri:
     "Bu oda az önce doldu. Müsait olursa bilgilendirilmek ister misiniz?"
     1 tık → waitlist'e ekle.
```

---

### 2. Pending Reservation Spam — Botlar 1000 Oda Tutukladı

```
Sorun:
  Rakip firma/bot: 1000 kullanıcı hesabı → her biri PENDING rezervasyon oluşturdu.
  1000 PENDING × 15 dk = 15 dk boyunca tüm oda envanteri kilitli.
  Gerçek kullanıcılar: "Müsait oda yok" görüyor → başka otele gidiyor.
  Bot: 15 dk sonra ödeme yapmıyor → rezervasyonlar expired.
  Rakip otel: gerçek kullanıcı aldı. Saldırı başarılı!

Çözüm:
  a) Kullanıcı başına PENDING limit:
     Aynı kullanıcı → aynı anda max 2 PENDING rezervasyon.
     3. PENDING denemesi → "Önce bekleyen rezervasyonunuzu tamamlayın."
     Redis SET: pending_count:{userId} → INCR ile say, DECR ile düşür.

  b) IP bazlı rate limiting:
     1 IP → 5 dk içinde max 10 rezervasyon denemesi.
     Bot genellikle aynı IP bloğunu kullanır.
     Proxy/VPN: CAPTCHA tetikle.

  c) Hesap doğrulama önce:
     Ödeme yöntemi kayıtlı değilse → PENDING oluşturma.
     Kredi kartı kaydı → bot engeli (sahte kart → şüpheli).
     SMS doğrulama → bir telefon numarası → bir hesap.

  d) PENDING süresi kısalt:
     15 dk → 10 dk (konser bileti gibi yoğun sistemde 5 dk).
     Kısa süre → bot yetişemiyor → gerçek kullanıcı daha hızlı.

  e) Behavioral analysis:
     Rezervasyon → ödeme arasındaki süre: normal kullanıcı 3-8 dk.
     Hiç ödeme olmayan PENDING → şüpheli hesap işaretle.
     Tekrarlayan pattern → otomatik ban + insan incelemesi.
```

---

### 3. Pending Expiry Worker Gecikmesi — Oda 30 dk Ekstra Kilitli Kaldı

```
Sorun:
  PENDING reservation: expires_at = 14:00.
  Worker çöktü → 14:15'te yeniden başladı.
  14:00-14:15 arası: oda gerçekte boş ama sistemde dolu görünüyor.
  Bu 15 dk: 50+ kullanıcı "dolu" mesajı aldı → kaçtı.
  Hotel: 50 potansiyel müşteri kaybı.

Çözüm:
  a) Worker HA (yüksek erişilebilirlik):
     Kubernetes: 2 replica (active + standby).
     Standby: primary çöktüğünde < 30 sn devreye girer.
     Leader election: Redis SETNX ile → sadece bir worker aktif (duplicate işlem yok).

  b) Redis ZSET (DB polling yerine):
     ZADD pending_expirations {expires_at_unix} {reservationId}
     Worker: ZRANGEBYSCORE pending_expirations -inf {now_unix}
     → süresi dolan reservation listesi → anında işle.
     DB polling: her dakika → ZSET: her sn ← daha hassas.

  c) Kafka delayed message:
     PENDING oluşunca: Kafka'ya 15 dk sonra tüketilecek mesaj koy.
     Consumer: mesaj geldi → DB kontrol → hâlâ PENDING mi? → expire et.
     DB kontrol şart: ödeme yapılmışsa CONFIRMED → expire etme.
     Kafka offset retention: mesajlar kaybolmaz, worker restart sonrası devam.

  d) Kullanıcıya uyarı:
     Checkout sayfası: geri sayım sayacı "14:27'ye kadar tutuldu."
     Süre dolmadan 2 dk: "Az kaldı! Lütfen ödemeyi tamamlayın."
     Süre doldu: "Rezervasyon süresi doldu, yeniden arama yapın."
```

---

### 4. Elasticsearch ile DB Tutarsızlığı — Dolu Otel "Müsait" Göründü

```
Sorun:
  Hotel X, 25 Aralık: son oda rezerve edildi → DB: booked_rooms = total_rooms.
  CDC → Kafka → ES Updater → 2 sn gecikme.
  Bu 2 sn: kullanıcı arama yaptı → ES "1 oda var" döndü → rezervasyona gitti.
  Booking Service: DB'yi kontrol etti → "dolu" → kullanıcıya hata.
  Kullanıcı: "Müsait diyordu, neden dolu?"

Çözüm:
  a) İki aşamalı: arama ES'ten, gerçek kontrol DB'den (mevcut tasarım):
     Kullanıcı deneyimi sorunsuz: arama hızlı, rezervasyon anında doğrulanır.
     "Müsait diyordu" mesajını kabul et: "Arama sırasında müsaitti, az önce doldu."
     Booking.com da aynı yaklaşımı kullanır.

  b) ES'te özel "son oda" durumu:
     available_count <= 0 → "availableRoomTypes" listesinden çıkar (anlık).
     Sadece bu güncelleme: partial ES update → hızlı (tam re-index değil).
     Booking gerçekleşince: doğrudan ES partial update çağrısı (CDC beklemeden).
     CDC de gelir → idempotent → sorun yok.

  c) Stale sonuç uyarısı:
     ES sorgu dönünce: "Müsaitlik son 2 saniyede değişmiş olabilir."
     Otel detay sayfasına tıklayınca: DB anlık kontrol.
     Böylece arama sayfası her zaman güncel olmak zorunda değil.
```

---

### 5. Dinamik Fiyat Çakışması — Fiyat Değişti Ama Kullanıcı Eski Fiyata Rezervasyon Yaptı

```
Sorun:
  Kullanıcı 09:00'da arama yaptı: "1 gece 200 TL."
  Fiyatlandırma motoru 09:05'te fiyatı 280 TL'ye çıkardı.
  Kullanıcı 09:10'da "Rezervasyon Yap"a tıkladı.
  Booking Service: o anki fiyat 280 TL → kullanıcı 200 TL bekliyordu.

Seçenekler:
  a) Arama fiyatını kilitle (price lock, 15 dk):
     Kullanıcı arama yapınca: fiyat snapshot alınır, 15 dk geçerli.
     price_lock:{sessionId} = {hotelId, roomType, date, price} → Redis TTL 15 dk.
     Rezervasyon: price_lock kontrolü → var → snapshot fiyat.
     Yok (15 dk geçti) → anlık fiyat göster → kullanıcı onaylamalı.

  b) Fiyat artışını bildir, onay iste:
     Booking anında: "Fiyat değişti: 200 TL → 280 TL. Devam etmek ister misiniz?"
     Kullanıcı onaylarsa → yeni fiyatla rezervasyon.
     Kullanıcı reddederse → arama sayfasına dön.

  c) %5'e kadar fark sessizce uygula:
     200 TL → 210 TL (+%5): sessizce uygula, kullanıcıya gösterme.
     200 TL → 280 TL (+%40): kesinlikle bildir + onay.
     İş kararı: hangi eşik uygun?

Seçilen: (a) + (b) hibrit:
  15 dk içinde: price lock → aynı fiyat.
  15 dk sonra: fiyat farkı > %10 → bildir + onay.
```

---

### 6. DB Connection Pool Tükendi — Peak'te Tüm Rezervasyonlar Timeout

```
Sorun:
  Tatil sezonu kampanyası: saniyede 500 rezervasyon denemesi.
  Her rezervasyon: BEGIN...SELECT FOR UPDATE...UPDATE...INSERT...COMMIT.
  Ortalama transaction süresi: 150 ms.
  500 req/sn × 150 ms = 75 aktif connection → pool: 50 → 25 istek kuyrukta.
  Kuyruk: 5 sn bekledi → timeout → "Sistem meşgul" hatası.
  Kullanıcılar: hata görüyor → panik → yeniden deniyor → daha da kötü.

Çözüm:
  a) Redis Distributed Lock (önce eleme):
     Lock alamayan → anında "Şu an yoğunuz, lütfen bekleyin" → DB'ye hiç gitme.
     Lock kapasitesi = DB pool kapasitesi kadar → DB korunur.
     Kullanıcıya: "Kuyruğa alındınız, sıra numaranız: 15."

  b) PgBouncer (connection pooler):
     Uygulama: 500 connection görür.
     PgBouncer: aslında 50 PostgreSQL connection → multiplexing.
     Transaction mode: her tx sonrası connection geri havuza → çok daha verimli.

  c) Async reservation queue:
     Rezervasyon isteği → Kafka'ya → consumer (kontrollü, 100/sn işle).
     Kullanıcı: hemen "Talebiniz alındı, işleniyor" → sonuç push notification.
     Dezavantaj: anlık "başarılı" bilgisi yok → kullanıcı bekleme toleransı.
     Konser bileti için: kabul edilebilir.

  d) Auto-scaling:
     CloudWatch: connection pool doluluk > %80 → yeni DB instance.
     Read replica: arama sorguları replica'ya → primary sadece write.
     Booking Service: HPA → pod sayısı artar → ama DB hâlâ bottleneck.
     Çözüm: şarding veya CQRS (rezervasyon write → ayrı DB).
```

---

### 7. Waitlist Bildirimi Kaçırıldı — İptal Olan Oda Boş Kaldı

```
Sorun:
  Hotel X, 25 Aralık: 5 kişi waitlist'te.
  Rezervasyon iptal edildi → oda boş.
  Waitlist Worker bildirim gönderdi → ama 3 kullanıcı bildirimi görmedi (push kapalı, e-posta spam).
  10 dk içinde kimse rezerve etmedi → bir sonraki waitlist kullanıcısına geçildi.
  Döngü: her seferinde 10 dk geçiyor → 5 saat sonra oda hâlâ boş.
  Hotel: dolu olması gereken gece boş geçti.

Çözüm:
  a) Multi-channel bildirim:
     Aynı anda: push notification + e-posta + SMS.
     Biri kaçırsa diğeri ulaşır.
     Öncelik: push (anlık) → e-posta (backup) → SMS (kritik).

  b) Kısa bildirim penceresi (10 dk → 5 dk):
     5 dk içinde rezervasyon → ok.
     5 dk geçince → bir sonraki kişiye → daha hızlı dönüşüm.

  c) Tüm waitlist'e aynı anda bildirim:
     Sırayla değil: hepine aynı anda bildir.
     Kim önce rezerve ederse alır (first come first served).
     5 waitlist × aynı bildirim → birincisi alır → diğerine "üzgünüz."
     Daha hızlı dolduruluyor.

  d) In-app waitlist dashboard:
     Kullanıcı: "Waitlist'im" sayfasında gördü → öncelikli bilgi.
     E-posta kaçmış olsa bile → açtığında görür.

  e) Otomatik fiyat düşürme (fallback):
     24 saat öncesine kadar kimse waitlist'ten almadıysa → son dakika fiyatı (%20 indirim).
     Yeni kullanıcılara duyuru: "Bugün için indirimli oda çıktı."
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Availability model | Booking-based (COUNT) | Pre-computed table | COUNT: 10K QPS'de ağır JOIN; Pre-computed: O(1) okuma |
| Arama DB | PostgreSQL LIKE | Elasticsearch | Geo, full-text, facet, sıralama; PG: küçük ölçekte yeterli |
| Kilit stratejisi (normal) | Optimistic (version) | Pessimistic (FOR UPDATE) | Otel: düşük contention, garanti şart; optimistic: retry storm |
| Kilit stratejisi (flash sale) | FOR UPDATE | Redis Distributed Lock | 100K req/sn → DB lock yetersiz; Redis: sub-ms ön eleme |
| Availability sync | Dual-write | CDC (Debezium) | Dual-write: atomik değil; CDC: WAL, güvenilir, replay |
| Pending timeout | DB scheduler (polling) | Redis ZSET + Kafka delayed | Polling: 1 dk hassasiyet; ZSET: sn hassasiyet |
| ES tutarsızlık | Senkron güncelleme | Async CDC (1-2sn lag) | Senkron: dağıtık transaction; Async: eventual, rezervasyonda DB doğrular |
| Fiyat değişikliği | Anlık fiyat uygula | 15 dk price lock | Anlık: kötü UX; Lock: kullanıcı arama fiyatını güvende hisseder |
| Multi-night lock sırası | Rastgele | Tarih sırasına göre | Rastgele: deadlock riski; Tarih sırası: deterministik, güvenli |
| Waitlist bildirim | Sıralı (birer birer) | Tüm waitlist'e aynı anda | Sıralı: yavaş dolduruluyor; Toplu: ilk gelenin alır, hız |
