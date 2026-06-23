# 08k — System Design: Ride-Sharing (Uber/Lyft)

## Gereksinimler

```
Functional:
  ✓ Yolcu → sürücü eşleştirme
  ✓ Gerçek zamanlı konum takibi (GPS)
  ✓ Ücret hesaplama (surge pricing dahil)
  ✓ Yolculuk state machine (REQUESTED→ACCEPTED→PICKED_UP→COMPLETED)
  ✓ Sürücü uygunluk yönetimi
  ✗ Ödeme, harita rendering, chat (out of scope)

Non-Functional:
  DAU: 5M yolcu, 1M sürücü
  Peak: 100,000 aktif yolculuk aynı anda
  Konum güncellemesi: Her 4 saniyede bir (sürücüler)
  Eşleştirme latency: < 1 dakika
  Availability: 99.99%
```

---

## Capacity Estimation

```
Konum güncellemeleri:
  1M sürücü, her 4 saniyede bir güncelleme
  QPS: 1M / 4 = 250,000 location update/sn

Eşleştirme:
  100,000 yolculuk/saat peak → 28 istek/sn

Storage:
  Konum: {driverId, lat, lng, timestamp} = 50 bytes
  250,000 × 50 bytes = 12.5 MB/sn (streaming, saklamak zorunda değiliz)
  Son konum: 1M sürücü × 50 bytes = 50 MB → Redis'te

Yolculuk kaydı:
  100,000 yolculuk/saat × 24 = 2.4M/gün × 1KB = 2.4 GB/gün
```

---

## High-Level Tasarım

```
   Sürücü App          Yolcu App
       │                   │
  (Konum GPS)         (İstek + konum)
       │                   │
   ┌───┴──────────────────┴────┐
   │        API Gateway        │
   └───┬──────────────────┬────┘
       │                  │
  ┌────┴────┐        ┌────┴──────┐
  │Location │        │  Trip     │
  │Service  │        │  Service  │
  └────┬────┘        └────┬──────┘
       │                  │
   ┌───┴───┐         ┌────┴─────┐
   │ Redis │         │ Matching │
   │Geosearch│       │  Engine  │
   └───────┘         └────┬─────┘
                          │
                    ┌─────┴───────┐
                    │  Trip DB    │
                    │ (Cassandra) │
                    └─────────────┘
```

---

## Konum Servisi (Location Service)

```
Sürücü her 4 saniyede konum gönderir:
  POST /driver/location
  {driverId: "d-123", lat: 41.0082, lng: 28.9784, timestamp: ...}

Location Service:
  1. Redis GeoSET'e yaz:
     GEOADD available_drivers 28.9784 41.0082 "d-123"
     (sadece müsait sürücüler burada)
  
  2. EXPIRE driver:d-123:lastSeen 30 (30s güncelleme gelmezse müsait değil)

Redis GeoSET:
  Dahili: WGS84 koordinat → 52-bit geohash → sorted set score
  GEOPOS available_drivers d-123 → lat/lng
  GEOSEARCH available_drivers FROMMEMBER yolcu_konumu BYRADIUS 5 km
    SORT ASC COUNT 10
```

---

## Eşleştirme Motoru (Matching Engine)

```
Yolcu istek oluşturdu:
  pickup: {lat: 41.01, lng: 28.98}
  dropoff: {lat: 41.05, lng: 29.01}

Matching Engine:
  1. pickup çevresindeki müsait sürücüleri bul:
     GEOSEARCH available_drivers FROMLONLAT 28.98 41.01 BYRADIUS 5 km ASC COUNT 10

  2. Her sürücü için tahmini varış süresi hesapla (ETA):
     Mapping Service API'si veya in-house routing engine

  3. Sürücülere teklif gönder (en yakından başla):
     Push notification → "Yolculuk teklifi (30s yanıt süresi)"

  4. İlk kabul eden sürücü → eşleşme tamamlandı
     İkinci gelen → "Başkası aldı" mesajı

  5. Sürücü 30s içinde yanıtlamazsa → sıradaki sürücüye geç

Redis Pub/Sub ile teklif:
  PUBLISH driver:d-123:offers {tripId, pickupLat, pickupLng, fare}
  Sürücü app WebSocket/MQTT ile subscribe
```

---

## Yolculuk State Machine

```
REQUESTED → (sürücü kabul) → ACCEPTED
ACCEPTED  → (sürücü pickup noktasında) → ARRIVED
ARRIVED   → (yolcu bindi) → IN_PROGRESS
IN_PROGRESS → (yolcu indi) → COMPLETING
COMPLETING → (ödeme tamamlandı) → COMPLETED

Hata durumları:
REQUESTED → (5 dk sürücü bulunamadı) → NO_DRIVER_FOUND
ACCEPTED  → (sürücü iptal) → CANCELLED_BY_DRIVER → yeniden eşleştir
ACCEPTED  → (yolcu iptal) → CANCELLED_BY_PASSENGER

Her state değişikliği:
  → DB'ye kaydet
  → Kafka event yayınla
  → Yolcu/sürücü app'e push notification
```

---

## Surge Pricing (Dinamik Fiyatlandırma)

```
Talep / Arz oranına göre çarpan:

Şehir bölgelere (hex grid: H3 index) bölünür
Her bölge için:
  active_requests = son 5 dakikada gelen istek sayısı
  active_drivers  = o bölgedeki müsait sürücü sayısı
  ratio = active_requests / active_drivers

  ratio < 1.0 → multiplier = 1.0x
  ratio 1.0-1.5 → multiplier = 1.2x
  ratio 1.5-2.0 → multiplier = 1.5x
  ratio > 2.0   → multiplier = 2.0x (max)

H3 Hexagonal Grid (Uber kullanır):
  Şehri altıgen hücreler → her hücre için istatistik
  GEOADD + ZRANGEBYSCORE ile bölge bazlı analiz
  Resolution 9: ~500m çaplı hücreler

Surge hesaplama servisi:
  Her 30 saniyede tüm aktif bölgeleri hesapla
  Redis'e yaz: SET surge:h3:{cellIndex} 1.5 EX 60
  Yolcu istek yaparken → kendi bölgesinin surge değerini çek
```

---

## Gerçek Zamanlı Konum Yayını (Yolcuya)

```
Yolculuk başladı → Yolcu sürücüyü canlı izler

WebSocket / Server-Sent Events:
  Yolcu app → WebSocket bağlantısı
  Location Service: sürücü konum güncelledi
  → Redis Pub/Sub: PUBLISH trip:t-123:location {lat, lng}
  → WebSocket server subscribe eder → yolcu app'e push

Frekans optimizasyonu:
  Takip ekranı açıkken: 4s güncelleme
  Arka planda: 15s (pil tasarrufu)
  Sürücü yolcuya 500m yaklaştığında: 2s (hassas takip)
```

---

## Veri Modeli

```sql
-- Yolculuk
CREATE TABLE trips (
    trip_id       UUID PRIMARY KEY,
    passenger_id  UUID NOT NULL,
    driver_id     UUID,
    status        VARCHAR(20),         -- REQUESTED, ACCEPTED, ...
    pickup_lat    DECIMAL(10,7),
    pickup_lng    DECIMAL(10,7),
    dropoff_lat   DECIMAL(10,7),
    dropoff_lng   DECIMAL(10,7),
    fare_estimate DECIMAL(10,2),
    fare_actual   DECIMAL(10,2),
    surge_multiplier DECIMAL(4,2) DEFAULT 1.0,
    requested_at  TIMESTAMP,
    started_at    TIMESTAMP,
    completed_at  TIMESTAMP
);

-- Konum geçmişi (yolculuk süresince)
CREATE TABLE trip_locations (
    trip_id    UUID,
    recorded_at TIMESTAMP,
    lat        DECIMAL(10,7),
    lng        DECIMAL(10,7),
    PRIMARY KEY (trip_id, recorded_at)
) -- Cassandra: trip_id by partition
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Konum depolama | PostgreSQL | Redis GeoSET | Redis (hızlı geo query) |
| Matching | Merkezi | Dağıtık bölge bazlı | Bölge bazlı (ölçeklenebilir) |
| Sürücü bildirim | Polling | WebSocket/MQTT | WebSocket (gerçek zamanlı) |
| Surge hesaplama | Per-request | Pre-computed | Pre-computed (hız) |
