# 08k — System Design: Ride-Sharing (Uber/Lyft)

## Gereksinimler

```
Functional:
  ✓ Yolcu → sürücü eşleştirme (yakın, hızlı, adil)
  ✓ Gerçek zamanlı konum takibi (GPS, 4sn interval)
  ✓ ETA hesaplama (rota + trafik)
  ✓ Ücret hesaplama (surge pricing dahil)
  ✓ Yolculuk state machine
  ✓ Sürücü uygunluk yönetimi
  ✓ Surge pricing (H3 hex grid, 30sn güncelleme)
  ✓ Rating sistemi (sürücü ↔ yolcu)
  ✓ Geofencing (havalimanı, yasak bölge)
  ✓ Güvenlik: SOS butonu, yolculuk paylaşma
  ✗ Ödeme, harita rendering, chat (out of scope)

Non-Functional:
  DAU: 5M yolcu, 1M sürücü
  Peak: 100,000 aktif yolculuk aynı anda
  Konum güncellemesi: her 4sn (aktif sürücü), her 10sn (bekleme)
  Eşleştirme latency: < 30sn (ilk sürücü teklifi)
  ETA hesaplama: < 200ms
  Availability: 99.99% (yılda 52 dk downtime)
  Konum doğruluğu: ±10m (şehir merkezi), ±30m (kenar)
```

---

## Capacity Estimation

```
Konum güncellemeleri:
  1M aktif sürücü × (1/4sn) = 250,000 location update/sn
  Her update: {driverId(16B) + lat(8B) + lng(8B) + ts(8B) + speed(4B)} ≈ 50 byte
  Bandwidth: 250,000 × 50B = 12.5 MB/sn inbound
  
  Kafka throughput: 12.5 MB/sn → 3 partition, 3 broker → trivial

Redis Geo storage:
  Müsait sürücü: 1M × 50 byte = 50 MB → tek Redis node yeter
  Geosearch: GEOSEARCH 5km radius → P99: ~2ms

Matching QPS:
  100,000 yolculuk/saat peak = 28 eşleştirme isteği/sn
  Her eşleştirme: ~10 sürücüye teklif → 280 push/sn → trivial

ETA hesaplama:
  28 eşleştirme × 10 sürücü = 280 ETA hesaplama/sn
  Routing engine: 280 req/sn → 3-5 node ile karşılanır

Yolculuk storage:
  2.4M yolculuk/gün × 2 KB = 4.8 GB/gün (Cassandra)
  Konum geçmişi: 100,000 aktif yolculuk × 4sn → 25,000 point/sn × 50B = 1.25 MB/sn

WebSocket bağlantıları:
  5M yolcu + 1M sürücü = 6M eş zamanlı WebSocket
  Her sunucu: ~100K bağlantı → 60 WebSocket sunucu node
```

---

## High-Level Mimari

```
  Sürücü App                  Yolcu App
   │ GPS update (4sn)           │ Trip request + GPS
   │ WebSocket (teklif/bildirim) │ WebSocket (konum takip, durum)
   ▼                            ▼
┌──────────────────────────────────────────────┐
│              API Gateway                      │
│  (Auth, rate limit, WebSocket upgrade)        │
└──┬──────────────┬──────────────┬─────────────┘
   │              │              │
   ▼              ▼              ▼
[Location    [Trip Service]  [WebSocket
 Service]    (state machine)  Manager]
   │              │              │
   ▼              │         ┌────┴──────────────┐
[Kafka        ┌──▼──┐       │  Redis Pub/Sub    │
 "locations"] │Trip │       │  (real-time push) │
   │           │ DB  │       └───────────────────┘
   ▼           │(PG) │
[Location    └──┬──┘
 Processor]     │
   │            ▼
   ▼         [Kafka "trip-events"]
[Redis       ┌──────┬──────────┬──────────────┐
 Geo]        ▼      ▼          ▼              ▼
          [Notif] [Analytics] [Billing]   [Driver
             Svc]              Svc        Earnings]
   │
   ▼
[Matching
 Engine]
   │
   ▼
[ETA Service ← Routing Engine (OSRM)]
   │
   ▼
[Surge Pricing Service (H3 grid)]
```

---

## Konum Servisi (Location Service)

### GPS Güncelleme Pipeline

```
Sürücü App → WebSocket (tek bağlantı, her 4sn location frame):
  {driverId: "d-123", lat: 41.0082, lng: 28.9784, speed: 45, heading: 180, ts: 1705312800}

Location Service (stateless, horizontal scale):
  1. Validasyon: koordinat sınır kontrolü, timestamp tazeliği (> 30sn eskiyse reddet).
  2. Kafka "driver-locations" topic'e publish (async, fire-and-forget).
  3. Client'a 200 OK → sürücü beklemez.

Location Processor (Kafka consumer, state yönetimi):
  1. Driver status belirle: müsait (AVAILABLE) mı, yolculukta (ON_TRIP) mı?
  2. AVAILABLE → Redis GeoSET güncelle: GEOADD available_drivers lng lat driverId
  3. ON_TRIP → Redis: SET driver:{driverId}:pos {lat,lng} EX 30
              → Kafka "trip-locations" → yolcu WebSocket'e push
  4. Heartbeat: SET driver:{driverId}:alive 1 EX 20
     → 20sn güncelleme gelmezse: key expire → driver OFFLINE → GEOREM

Delta compression (bant genişliği tasarrufu):
  Sürücü durdu (kırmızı ışık): konum değişmedi → yine de 4sn'de gönder? Hayır.
  Client: son konuma göre delta > 10m veya heading değişti > 5° → gönder.
  Aksi halde: "still alive" tiny heartbeat (sadece driverId + ts, 20 byte).
  Tasarruf: şehir trafiğinde ~%60 (çok kırmızı ışık).

GeoSET yapısı (Redis):
  available_drivers: ZADD — dahili olarak WGS84 → 52-bit geohash → sorted set score.
  GEOPOS available_drivers d-123 → {lat, lng}
  GEOSEARCH available_drivers FROMLONLAT 28.98 41.01 BYRADIUS 5 km ASC COUNT 20
```

### Cell-Based Sharding (Büyük Şehirler İçin)

```
Problem: İstanbul'da 200,000 aktif sürücü → tek Redis GeoSET → 1 node darboğaz.

Çözüm: H3 hex cell bazlı sharding.
  Resolution 6 hücre: ~36 km² → İstanbul ~1,400 km² → ~39 hücre.
  Her hücre: kendi Redis Cluster node'unda.
  
  Şema:
    available_drivers:{h3CellR6} → o hücredeki sürücüler.
    Örnek: available_drivers:86269ca4fffffff → Kadıköy bölgesi.

Matching sırasında:
  Yolcu konumu → H3 resolution 6 hücresi → o hücre + komşu hücreler (ring-1).
  7 hücre × kendi node → paralel GEOSEARCH → sonuçları merge et.
  
  Ring expansion: 5km yarıçapında sürücü yok → ring-2 → 10km → vs.

  k = H3.kRing(passengerCell, 1)  → merkez + 6 komşu = 7 hücre
  paralel: 7 × GEOSEARCH → union → ETA hesapla → rank et

Avantaj: her cell bağımsız → bir node down → sadece o bölge etkilenir.
```

---

## Eşleştirme Motoru (Matching Engine)

### Sürücü Seçim Algoritması

```
Sürücü havuzu: 5km içindeki 20 müsait sürücü.

Naif yaklaşım: sadece mesafe → en yakın sürücü.
Sorun: aynı sürücüler sürekli seçilir → yüksek gelirli bölge monopoly.
Gerçek Uber yaklaşımı: çoklu faktör skoru.

Sürücü skoru = w1 × ETA_score
             + w2 × acceptance_rate_bonus
             + w3 × rating_score
             + w4 × earnings_fairness
             - w5 × cancellation_penalty

ETA_score:
  Tahmini varış süresi (dakika) → ters orantılı.
  ETA 2dk → 1.0, ETA 5dk → 0.6, ETA 10dk → 0.3.

acceptance_rate_bonus:
  Son 7 günde teklif kabul oranı > %90 → bonus.
  Sürekli reddeden sürücü → ceza.

rating_score:
  Sürücü ortalama puanı (son 100 yolculuk).
  > 4.8 → bonus, < 4.5 → ceza.

earnings_fairness:
  Bugün az kazanan sürücü → bonus (adil dağılım).
  Günde 8 saatte herkes benzer gelir → sistem sağlıklı.
  Bu LinkedIn / Uber'ın gerçekten yaptığı.

cancellation_penalty:
  Son 24 saatte yolculuğu kabul edip iptal → ağır ceza.
```

### Teklif Sırası ve Timeout Cascade

```
Top 3 sürücüye aynı anda teklif gönder (paralel):
  Neden 3: tek sürücüye → reddederse 30sn kayıp → kötü UX.
  Neden 3 (max): hepsine birden → tümü kabul → hangisi gerçek eşleşme?

Paralel teklif yönetimi:
  TripId → Redis SET: offered_to:{tripId} {d1, d2, d3} EX 30

  D1 ACCEPT:
    Redis: SETNX trip_accept:{tripId} "d1" EX 5 → success.
    D2 sonra ACCEPT: SETNX → fail → "Başkası aldı" mesajı.
    D3 reddetti → zaten geç.

  30sn içinde kimse kabul etmedi:
    Sonraki 3 sürücü → tekrar teklif (ring-2'ye genişle).
    Bu süreç 5 dakika devam eder → sürücü bulunamazsa NO_DRIVER_FOUND.

  Teklif geri alma:
    Sürücü: "30sn içinde yanıtlamazsa" → otomatik TIMEOUT sayılır.
    acceptance_rate düşer → bu birikirse ceza.

Fairness (gelir adaleti):
  Merkez sürücüler: sürekli kısa yolculuk, yüksek başlangıç ücreti.
  Kenar sürücüler: uzun yolculuk, bekler ama km ücreti.
  Sistem: periyodik olarak merkez sürücülere kenar yolculukları yönlendir.
  Sürücü kazancı: saatlik garantili minimum → Uber'ın driver retention stratejisi.
```

---

## ETA Hesaplama

```
ETA = pickup ETA + dropoff ETA (tahmini yolculuk süresi).

Routing engine seçenekleri:
  Google Maps/Directions API:
    Avantaj: gerçek zamanlı trafik, çok hassas.
    Dezavantaj: pahalı (her çağrı ücretli), rate limit.
    Kullanım: müşteriye gösterilen final ETA.

  OSRM (Open Source Routing Machine):
    Avantaj: kendi sunucunda, çok hızlı (<10ms), sınırsız.
    Dezavantaj: gerçek zamanlı trafik yok (statik OpenStreetMap).
    Kullanım: matching sırasında (hızlı, çok sayıda sürücü için).

  HERE / TomTom:
    Google Maps alternatifi, trafik dahil, daha ucuz.

Hibrit yaklaşım (Uber):
  Matching: OSRM (hızlı, 280 hesaplama/sn).
  Kullanıcıya gösterilen ETA: traffic-aware routing (Google/HERE).
  Yolculuk başladı: gerçek zamanlı rota yenileme (her 30sn).

Trafik faktörü:
  Geçmiş veri: "Salı 08:30'da Kadıköy-Taksim 25 dk sürer" → ML prediction.
  Anlık GPS: aktif yolculukların hızı → o andaki trafik durumu → ETA güncelle.
  Örnek: 1000 araç Boğaz Köprüsü'nü geçiyor → ortalama hız 15km/s → ETA artar.

ETA cache:
  Aynı pickup-dropoff çifti son 1 dakikada hesaplandıysa → cache'ten dön.
  Cache key: H3(pickup, res9) + H3(dropoff, res9) + floor(epoch/60).
  Resolution 9 hücre: ~150m → komşu pickup noktaları aynı cache key.
```

---

## Yolculuk State Machine

```
              ┌─────────────┐
              │  REQUESTED  │ ← yolcu istek oluşturdu
              └──────┬──────┘
                     │ sürücü ACCEPT
              ┌──────▼──────┐
              │  ACCEPTED   │ ← sürücü yolda
              └──────┬──────┘
                     │ sürücü pickup'a geldi
              ┌──────▼──────┐
              │   ARRIVED   │ ← "Sürücünüz kapıda"
              └──────┬──────┘
                     │ yolcu bindi (sürücü onayladı)
              ┌──────▼──────┐
              │ IN_PROGRESS │ ← yolculuk devam ediyor
              └──────┬──────┘
                     │ yolcu indi (sürücü onayladı)
              ┌──────▼──────┐
              │ COMPLETING  │ ← ödeme işleniyor
              └──────┬──────┘
                     │ ödeme OK
              ┌──────▼──────┐
              │  COMPLETED  │
              └─────────────┘

Hata / iptal durumları:
  REQUESTED  → (5dk sürücü bulunamadı) → NO_DRIVER_FOUND
  REQUESTED  → (yolcu iptal) → CANCELLED_BY_PASSENGER (ücretsiz)
  ACCEPTED   → (yolcu iptal) → CANCELLED_BY_PASSENGER (iptal ücreti var)
  ACCEPTED   → (sürücü iptal) → CANCELLED_BY_DRIVER → yeniden eşleştir
  ARRIVED    → (yolcu 5dk gelmedi) → CANCELLED_BY_DRIVER (sürücü ücreti alır)
  IN_PROGRESS → (app crash) → ORPHANED → recovery: GPS takibi ile devam

Her state değişimi:
  1. PostgreSQL: UPDATE trips SET status=? WHERE trip_id=? AND status=? (optimistic lock)
  2. Redis: SET trip:{tripId}:status {newStatus} EX 3600
  3. Kafka "trip-events" → notification, billing, analytics consumer'ları
  4. WebSocket: yolcu + sürücü uygulamalarına push
```

```java
@Service
@Transactional
class TripStateMachine {

    private static final Map<TripStatus, Set<TripStatus>> TRANSITIONS = Map.of(
        REQUESTED,   Set.of(ACCEPTED, CANCELLED_BY_PASSENGER, NO_DRIVER_FOUND),
        ACCEPTED,    Set.of(ARRIVED, CANCELLED_BY_DRIVER, CANCELLED_BY_PASSENGER),
        ARRIVED,     Set.of(IN_PROGRESS, CANCELLED_BY_DRIVER),
        IN_PROGRESS, Set.of(COMPLETING),
        COMPLETING,  Set.of(COMPLETED)
    );

    void transition(String tripId, TripStatus newStatus, String actor) {
        // Optimistic lock: sadece beklenen status'taysa güncelle
        int updated = tripRepo.transitionStatus(tripId,
            TRANSITIONS.keySet().stream()
                .filter(s -> TRANSITIONS.get(s).contains(newStatus))
                .collect(Collectors.toSet()),
            newStatus);

        if (updated == 0) {
            throw new InvalidStateTransitionException("Geçersiz geçiş: " + newStatus);
        }

        tripEventRepo.save(TripEvent.of(tripId, newStatus, actor));

        kafkaTemplate.send("trip-events", tripId,
            new TripStatusChangedEvent(tripId, newStatus, actor, Instant.now()));
    }
}
```

---

## Surge Pricing (Dinamik Fiyatlandırma)

```
Talep / Arz dengesine göre çarpan belirleme.

H3 Hexagonal Grid (Uber'ın kullandığı):
  Şehir → altıgen hücreler.
  Resolution 7: ~5.16 km² → ~750m çaplı → mahalle seviyesi.
  Resolution 9: ~0.10 km² → ~150m çaplı → sokak bloğu.
  Surge: Resolution 7 (çok ince bölümleme → karmaşık).

Her hücre için her 30sn:
  active_requests  = son 5 dakikada o hücreden gelen istek sayısı
  available_drivers = o hücre + komşu ring-1'deki müsait sürücü sayısı
  ratio = active_requests / max(available_drivers, 1)

  Çarpan tablosu:
  ratio < 0.5  → 1.0x (arz > talep, düşük yoğunluk)
  ratio 0.5-1.0 → 1.0x (normal)
  ratio 1.0-1.5 → 1.2x
  ratio 1.5-2.0 → 1.5x
  ratio 2.0-3.0 → 2.0x
  ratio > 3.0  → 2.5x (tavan — Türkiye'de yasal sınır var)

Surge Service (Flink Streaming + Redis):
  Kafka "driver-locations" + "trip-requests" → Flink → her 30sn pencere.
  Çıkış: SET surge:h3:{cellId} 1.5 EX 90 (90sn geçerliliği var).

Yolcuya gösterim:
  İstek → kendi H3 hücresi → surge_multiplier çek.
  "Yoğunluk nedeniyle ücret 1.5x" → onay pop-up → kullanıcı ONAYLAMADAN ödeme yok.
  Kabul etmezse: 5dk bekle → surge düşebilir → bildirim gönder.

Surge manipülasyon önleme:
  Sürücüler bölgeden ayrılarak yapay surge yaratabilir.
  Tespit: ani sürücü çekimi + request artışı olmadan → anomali → manuel inceleme.
  Ceza: kasıtlı manipülasyon → hesap askıya alma.

Yapay zeka ile surge tahmini:
  Geçmiş veri: "Cumartesi 23:00 Taksim → her zaman 2.0x"
  Önceden bildir: "Yarın akşam bu bölgede yoğunluk bekleniyor" → sürücüler gelir.
  Proaktif surge → reaktif surge'den daha iyi (sürücü çekme).
```

---

## Ücret Hesaplama

```
fare = base_fare
     + (distance_km × per_km_rate)
     + (duration_min × per_min_rate)
     + tolls
     + airport_fee (varsa)
     + surge_multiplier

Örnek (İstanbul, standart):
  base_fare    = 30 TL
  per_km_rate  = 12 TL/km
  per_min_rate = 1.5 TL/dk
  
  5 km, 15 dk, 1.5x surge:
  = (30 + 5×12 + 15×1.5) × 1.5
  = (30 + 60 + 22.5) × 1.5
  = 112.5 × 1.5 = 168.75 TL

Minimum ücret: 50 TL (kısa yolculuk koruması).
Maksimum ücret: önceden tahmin verilen üst sınır × 1.2 (sürücü sapma koruması).

Fare estimation (yolculuk başlamadan):
  OSRM: pickup → dropoff → tahmini mesafe + süre.
  Fare estimate: yukarıdaki formül + anlık surge.
  "Tahmini ücret: 150-180 TL" → aralık göster (trafik belirsizliği).

Gerçek fare (yolculuk bitti):
  Gerçek GPS treki: Polyline → toplam mesafe (Haversine).
  Gerçek süre: started_at → completed_at.
  Surge: yolculuk başlangıcındaki surge (değişmez → kullanıcı onayladı).

Route deviation koruması:
  Sürücü kasıtlı uzun yol giderse → maksimum ücret = önerilen rota × 1.3.
  Kullanıcı: daha fazla ödemez → sürücü fark ödemiyor (negatif davranışı azaltır).

Toll otomasyonu:
  HGS/OGS verisi: hangi köprü, ne zaman geçildi → fiyatı biliniyor.
  Otomatik: toll belirlenen miktarda ayrıca eklenir.
```

---

## Gerçek Zamanlı Konum Yayını

### WebSocket Mimarisi ve Ölçekleme

```
6M eş zamanlı WebSocket bağlantısı (5M yolcu + 1M sürücü).

WebSocket Gateway (Nginx / Envoy):
  Sticky session: kullanıcı → her zaman aynı backend node (WebSocket stateful).
  Sticky mechanism: user_id hash → consistent hashing → node.
  Avantaj: aynı connection her zaman aynı yerde.

WebSocket Node (Node.js / Netty):
  Her node: 100K bağlantı → 60 node.
  Bellek: her bağlantı ~10KB → 100K × 10KB = 1 GB/node.
  Node kapasitesi: 16 GB RAM → 1,500K bağlantı/node mümkün.

Redis Pub/Sub (node'lar arası mesajlaşma):
  Sürücü konum güncelledi → Location Processor → Redis PUBLISH trip:{tripId}:loc {lat,lng}
  Yolcunun WebSocket node'u → ilgili trip channel'larına SUBSCRIBE.
  Yolcu bağlantısı: WebSocket Node A → Redis'e subscribe.
  Sürücü mesajı: Location Processor → Redis PUBLISH → Node A → yolcu uygulamasına.

Alternatif (büyük ölçek): Apache Kafka (daha güvenilir pub/sub):
  Redis Pub/Sub: at-most-once (WebSocket node down → mesaj kayıp).
  Kafka + Consumer Group: at-least-once → yeniden bağlanınca catch-up.

Frekans optimizasyonu:
  Takip ekranı açık (foreground): 4sn.
  Arka planda (background): 15sn.
  Sürücü 500m yakınında: 2sn (kritik an).
  Pil modu (low battery): 30sn.
  
  Client: state bazlı interval → sunucuya bildir → gereksiz update azalt.
```

### Yolcu Canlı Takip Akışı

```
Yolculuk başladı:
  Yolcu app: WebSocket bağlantısı kurdu → "SUBSCRIBE trip:{tripId}".
  
  Sürücü GPS → Location Service → Kafka "trip-locations".
  Location Processor (Kafka consumer):
    msg = {driverId, lat, lng, ts}.
    activeTrip = redis.get("driver:{driverId}:active_trip") → tripId.
    redis.publish("trip:{tripId}:loc", {lat, lng, ts, eta_to_dest}).
  
  WebSocket Node: "trip:{tripId}:loc" subscribe → yolcu app'e push.
  
  Yolcu app: haritada sürücü konumu güncellendi.
  ETA: routing engine → güncellenen konum → yeni ETA → push.

Kopuk bağlantı (yolcu tünel geçiyor):
  WebSocket: koptu → yolcu app: reconnect (exponential backoff).
  Yeniden bağlandı: "GET /trip/{tripId}/driver-location" → son bilinen konum.
  WebSocket: yeniden subscribe → normal akışa dön.
```

---

## Geofencing

```
Tanımlı özel bölgeler:
  Havalimanı: özel ücret + kalkış noktası kısıtlaması.
  Okul bölgesi: belirli saatlerde sürücü yavaşlaması.
  Yasak bölge (pedestrian): araç giremez.
  Surge zone: manuel olarak surge artırılan bölge (konser, maç).
  No-pickup zone: bir yerden yolcu alınamaz (doğrudan gelmeli).

Geofence tespiti (Polygon İçinde mi?):
  Ray casting algoritması: nokta → sonsuz yönde ray at → kaç kez polygon kenarını kesti?
  Tek sayıda kesim: içeride.
  Çift sayıda: dışarıda.
  H3: polygon → H3 hücreleri → SET → O(1) "içinde mi?" sorgusu.

Havalimanı özel mantığı:
  Pickup: terminal girişi, belirlenen hat.
  Dropoff: herhangi yerinden gelebilir.
  Ücret: "Havalimanı Ücreti" fixed fee + normal fare.
  Kuyruk sistemi: çok sürücü bekliyorsa → first-in-first-out kuyruk.
    FIFO queue: Redis LIST → RPUSH (arkaya ekleme) → LPOP (öne alma).

Uygulama:
  Location Service: her driver update → geofence check.
  Geofence cache: tüm aktif geofence'leri Redis → polygon geometrisi.
  Her konum güncellemesi: hangi geofence içinde? → driver status güncelle.
```

---

## Sürücü Supply/Demand Forecasting

```
Sorun: Saat 08:00, Kadıköy metrosu önünde 200 kişi → 5 sürücü var.
Sonuç: uzun bekleme → kullanıcı şikayeti → churn.

Çözüm: önceden tahmin et, sürücüleri yönlendir.

Demand forecasting (ML):
  Features:
    Gün, saat, hava durumu, tatil mi, yakında etkinlik var mı?
    Geçmiş talep: "Pazartesi 08:00 Kadıköy → 150 istek/saat tipik."
    Anlık veriler: public transit gecikmesi (metro arızası → Uber talebi artar).
  Model: LightGBM veya LSTM (zaman serisi).
  Çıkış: her H3 hücresi × saat için tahmini istek sayısı.
  Güncelleme: saatlik model çalışması + gerçek zamanlı düzeltme.

Sürücü yönlendirme (incentive):
  "Kadıköy'e git → bonus 20 TL ekstra" → push notification.
  Sürücü: zaten yakındaysa → gitme olasılığı yüksek.
  Çok sürücü gitmeye başladı → incentive azalt (real-time koordinasyon).

Havalimanı özel:
  Uçuş iniyor → 20 dk sonra yolcu talebi → sürücüleri terminal'e çek.
  Uçuş verileri API (FlightAware): gerçek zamanlı iniş bilgisi → demand spike tahmini.

Heat map:
  Sürücü app: şehir haritasında ısı haritası.
  Kırmızı: yoğun talep, az sürücü → "buraya gel, kazanırsın."
  Yeşil: çok sürücü, az talep → "buradan uzaklaş."
```

---

## Rating Sistemi

```
Yolculuk tamamlandı → yolcu + sürücü birbirini puanlar (1-5 yıldız).

Sürücü puanı:
  Son 500 yolculuğun ortalaması (eski puanlar ağırlığı azalır).
  < 4.5 → uyarı e-postası.
  < 4.2 → hesap deaktive (3 uyarı sonrası).
  Neden önemli: eşleştirme skoru → düşük puan → daha az yolculuk.

Yolcu puanı:
  Sürücüler "rahatsız edici yolcu" bildirmesi → negatif sinyal.
  Düşük puanlı yolcu → bazı premium sürücüler eşleşmez.
  < 4.0 → bazı özellikler kısıtlı.

Puan manipülasyon önleme:
  "5 yıldız veya tehdit" → anomali tespiti.
  Korelasyon: sürücü belirli yolcuları hep 1 yıldız veriyorsa → anlaşma şüphesi.
  Kötü puan sonrası şikayet → destek incelemesi.

Rating'in eşleştirmeye etkisi:
  Sürücü: puanı × rating_weight → matching score.
  Yolcu: premium sürücü → yüksek puanlı yolcuları tercih edebilir.
  Ama: bu ayrımcılığa kapı açar → dikkatli uygulama (ırk, cinsiyet körü).
```

---

## Güvenlik Özellikleri

```
SOS Butonu:
  Yolcu (veya sürücü) app'te kırmızı butona basar.
  Emergency Service: konum + kullanıcı bilgisi + trip detayı → 112'ye gönder.
  Aynı zamanda: acil kişi listesine SMS + konum paylaş.
  Kayıt: SOS sırasında ses kaydı başlar (hukuki gereklilik → bilgilendirme var).
  Trip: EMERGENCY statüsüne geçer → 7/24 destek ekibi bildirilir.

Yolculuk paylaşma:
  "Güvenli ulaştım" özelliği: bir telefon numarasına canlı konum linki.
  Link: "https://uber.com/share/{shareToken}" → gerçek zamanlı konum.
  TTL: yolculuk + 30 dk.
  Yolcu bitmedi mi? → "Yolculuk beklenmenden uzun sürdü" → acil kişiye bildirim.

Araç bilgisi doğrulama:
  Yolcu bindi → app: "Sürücünün plakası XXXXX mi?" → onay.
  Eşleşmiyor → "Binme, destek ekibini ara" uyarısı.
  Plaka tanıma: kamera → OCR → karşılaştır (gelişmiş).

Route izleme:
  Beklenen rota: OSRM önerisi.
  Gerçek rota: GPS track.
  Büyük sapma (>500m, planlanmamış yön): otomatik bildirim → "Beklenen rotadan saptı."
  ML: "Bu sapma normal mı?" (kısa yol mu, kasıtlı sapma mı?).

Sürücü kimlik doğrulama:
  Her shift başında: selfie → yüz tanıma → kayıtlı fotoğrafla karşılaştır.
  Başkasının hesabını kullanan sürücü → tespit → hesap askıya alma.
```

---

## Veri Modeli

```sql
CREATE TABLE trips (
    trip_id           UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    passenger_id      UUID          NOT NULL,
    driver_id         UUID,
    status            VARCHAR(30)   NOT NULL DEFAULT 'REQUESTED',
    pickup_lat        DECIMAL(10,7) NOT NULL,
    pickup_lng        DECIMAL(10,7) NOT NULL,
    pickup_address    TEXT,
    dropoff_lat       DECIMAL(10,7) NOT NULL,
    dropoff_lng       DECIMAL(10,7) NOT NULL,
    dropoff_address   TEXT,
    vehicle_type      VARCHAR(20)   DEFAULT 'STANDARD',  -- STANDARD, COMFORT, XL
    fare_estimate     DECIMAL(10,2),
    fare_actual       DECIMAL(10,2),
    distance_km       DECIMAL(8,3),
    duration_min      INT,
    surge_multiplier  DECIMAL(4,2)  DEFAULT 1.0,
    toll_amount       DECIMAL(10,2) DEFAULT 0,
    passenger_rating  SMALLINT,        -- sürücünün yolcuya verdiği puan
    driver_rating     SMALLINT,        -- yolcunun sürücüye verdiği puan
    cancellation_reason VARCHAR(50),
    h3_pickup_r7      VARCHAR(20),     -- surge hesabı için
    requested_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    accepted_at       TIMESTAMPTZ,
    arrived_at        TIMESTAMPTZ,
    started_at        TIMESTAMPTZ,
    completed_at      TIMESTAMPTZ,
    cancelled_at      TIMESTAMPTZ
) PARTITION BY RANGE (requested_at);   -- aylık partition

-- Yolculuk konum geçmişi (Cassandra — yazma ağırlıklı)
-- Partition key: trip_id
-- Clustering key: recorded_at DESC
CREATE TABLE trip_locations (
    trip_id      UUID,
    recorded_at  TIMESTAMP,
    lat          DECIMAL(10,7),
    lng          DECIMAL(10,7),
    speed_kmh    DECIMAL(5,1),
    heading      SMALLINT,          -- 0-359 derece
    accuracy_m   SMALLINT,          -- GPS doğruluğu (metre)
    PRIMARY KEY ((trip_id), recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC)
  AND default_time_to_live = 2592000;  -- 30 gün TTL

-- Sürücü durumu (PostgreSQL)
CREATE TABLE drivers (
    driver_id        UUID          PRIMARY KEY,
    user_id          UUID          NOT NULL UNIQUE,
    vehicle_plate    VARCHAR(20)   NOT NULL,
    vehicle_model    VARCHAR(100),
    vehicle_type     VARCHAR(20),
    rating           DECIMAL(3,2)  DEFAULT 5.0,
    rating_count     INT           DEFAULT 0,
    status           VARCHAR(20)   DEFAULT 'OFFLINE',  -- OFFLINE, AVAILABLE, ON_TRIP
    acceptance_rate  DECIMAL(5,4),
    cancellation_rate DECIMAL(5,4),
    total_trips      INT           DEFAULT 0,
    total_earnings   DECIMAL(12,2) DEFAULT 0,
    last_known_lat   DECIMAL(10,7),
    last_known_lng   DECIMAL(10,7),
    last_seen_at     TIMESTAMPTZ
);

-- Sürücü kazanç tablosu
CREATE TABLE driver_earnings (
    earning_id       UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id        UUID         NOT NULL,
    trip_id          UUID,
    amount           DECIMAL(10,2) NOT NULL,
    commission_rate  DECIMAL(4,3), -- platform komisyonu (örn: 0.25 = %25)
    net_amount       DECIMAL(10,2) NOT NULL,
    earning_type     VARCHAR(20),  -- TRIP, BONUS, INCENTIVE, ADJUSTMENT
    earned_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Redis key'leri (referans):
-- available_drivers:{h3CellR6}  → GeoSET (müsait sürücüler)
-- driver:{driverId}:alive       → TTL 20sn (heartbeat)
-- driver:{driverId}:pos         → {lat,lng} (ON_TRIP sürücüsü)
-- driver:{driverId}:active_trip → tripId
-- trip:{tripId}:status          → hızlı okuma cache
-- trip_accept:{tripId}          → hangi sürücü kabul etti (race condition önleme)
-- offered_to:{tripId}           → {d1,d2,d3} SET (kimler teklif aldı)
-- surge:h3:{cellId}             → 1.5 (90sn TTL)
```

---

## Olası Sorunlar ve Çözümleri

### 1. Bölge Sınırı Sorunu — Yolcu Bölgeler Arasında, Sürücü Bulunamadı

```
Sorun:
  Cell-based matching: şehir H3 hücrelerine bölündü.
  Yolcu: iki hücrenin tam sınırında → hangi hücrede?
  Matching: sadece yolcunun hücresi + ring-1 → sürücü bir sonraki ring-2'de.
  Sonuç: 500m ötede müsait sürücü var ama eşleştirme bulamıyor.

Çözüm:
  a) Ring expansion (adaptif):
     Önce ring-1 → 15sn içinde sürücü yok → ring-2 → 30sn → ring-3.
     Her expansion: daha büyük alan → daha fazla sürücü aday.

  b) H3 kompaktlaştırma:
     Yolcu konumunu birden fazla H3 hücresinde göster (pickup accuracy belirsiz).
     Eşleştirme: birden fazla H3 hücresi → union → daha geniş arama.

  c) Soft boundary:
     Hücre sınırını %20 örtüştür: sınır boyunca sürücüler → her iki hücre matching'ine dahil.
     Veri çoğalması: kabul edilebilir (sınır sürücüleri her iki hücrede görünür).

  d) Geofence bazlı hata yok:
     Arama: merkezi Euclidean/Haversine, hücre sadece sharding için.
     Redis GEOSEARCH: zaten gerçek mesafeye göre → hücre sadece hangi Redis node seçileceğini belirler.
```

---

### 2. Hayalet Sürücü — Kabul Etti, Gelmiyor

```
Sorun:
  Sürücü teklifi kabul etti → yolcu "5 dk içinde gelecek" mesajı aldı.
  Sürücü: başka bir konumda, hareket etmiyor.
  GPS: 3 dk sonra hâlâ 2km uzakta.
  Yolcu: 10 dk bekliyor → "nerede bu sürücü?" → destek ekibine şikâyet.

Neden olur:
  Sürücü yanlışlıkla kabul etti → fark etmedi.
  Sürücü kask/kulaklık → bildirim sesi duymadı.
  Sistem hatası → sürücü GPS güncelleme göndermiyor.

Çözüm:
  a) Hareket beklentisi (3 dakika kuralı):
     Sürücü ACCEPTED → 3 dk → pickup yönünde 100m+ hareket etmedi → uyarı push.
     "Yolculuğa doğru hareket etmiyor musunuz? İptal etmek ister misiniz?"
     5 dk → hâlâ hareket yok → otomatik CANCELLED_BY_SYSTEM → yeniden eşleştir.

  b) ETA aşımı:
     Kabul anındaki ETA: 5 dk.
     Şu an ETA: 15 dk (trafik veya hareket etmedi).
     Yolcuya: "Tahmini süre güncellendi: +10 dk" → yolcu iptal edebilir (ücretsiz).

  c) Sürücü ceza mekanizması:
     "Kabul edip gelmeyen" → cancellation_rate artar.
     %10 üzeri → eşleştirme skoru düşer → daha az yolculuk.
     %20 üzeri → hesap inceleme.

  d) Yolcuya ikinci şans önerisi:
     Phantom detection → "Sürücünüz gecikmesi → Şimdi yeni sürücü bul?" buton.
     Eski sürücü: hâlâ ACCEPTED → sistem: eski eşleşmeyi CANCELLED → yeni arama.
```

---

### 3. GPS Stale — Sürücü Konumu Güncel Değil

```
Sorun:
  Sürücü: tünel içinde → GPS sinyali yok → 2 dk konum güncellemesi gelmiyor.
  Redis: driver:{driverId}:alive key → expire → sürücü OFFLINE sayıldı.
  Matching: bu sürücü müsait değil → yolcu başka sürücüye yönlendirildi.
  Sürücü tünelden çıktı → yolculuğunu kaçırdı.

Neden kritik:
  Sürücü gelir kaybı → memnuniyetsizlik → platfor terk.

Çözüm:
  a) Toleranslı TTL:
     Hayatta kalma: 30sn (20sn yerine).
     Eşleştirme: 30sn güncelleme yok → "stale" ama hâlâ aday.
     60sn yok → OFFLINE.

  b) Dead reckoning (ölçüm tahmini):
     Son bilinen konum + hız + heading → 30sn sonra tahmini konum.
     Sürücü kaybolunca: tahmini konumla devam et.
     ETA: tahmini konum kullan → yaklaşık ama kullanılabilir.

  c) Tünel tespit:
     Hızlanma sensörü + gyro → "araç hareket ediyor ama GPS yok" → tünel.
     Client: tünel modu → inertial navigation → GPS bekleme.
     WiFi positioning: tünel WiFi → kaba konum tahmini.

  d) Graceful degradation:
     GPS yok → Network cell tower triangulation → ±100m doğruluk.
     Hâlâ yok → son bilinen konum + timestamp → "Sinyal zayıf" göster.
```

---

### 4. Peak Spike — Konser Sonrası 50,000 Eş Zamanlı İstek

```
Sorun:
  Büyük konser bitti → 50,000 kişi aynı anda Uber açtı.
  Matching Engine: 50,000 istek/sn → normalde 28/sn.
  Redis GEOSEARCH: 50,000 × paralel sorgu → Redis CPU %100 → timeout.
  Sonuç: "Sürücü bulunamadı" herkes için → app çöküyor gibi görünüyor.

Çözüm:
  a) Pre-built matching için H3 önbelleği:
     Her 30sn: her H3 hücresi için müsait sürücü listesi → önbelleğe al.
     Konser spike: GEOSEARCH değil → önbellekten oku.
     Stale: 30sn → kabul edilebilir.

  b) Queue ve rate limiting:
     Matching isteği → queue → 1,000 matching/sn kapasitesi ile işle.
     Yolcu: "Sürücü aranıyor..." spinner → aslında kuyrukta.
     Bekleme süresi: 60-90sn → kabul edilebilir (bant genişliği aşımından iyidir).

  c) Proaktif sürücü çekimi:
     Etkinlik takvimi → 2 saat önce tahmin → sürücülere incentive.
     "XYZ Arena çevresine gel → bonus 50 TL."
     Konser bitince: 500 sürücü hazır → spike absorbe edilir.

  d) Horizontal auto-scaling:
     Matching Engine: KEDA → Kafka consumer lag → pod sayısı artır.
     Lag arttı → 10 pod → 50 pod → 5 dk içinde ölçeklenir.
     Redis: read replica → GEOSEARCH yükü → okuma replikalara dağıt.
```

---

### 5. Çift Kabul — İki Yolcu Aynı Sürücüye Eşlendi

```
Sorun:
  İki yolcu aynı anda istek oluşturdu → her ikisi de D-123'ü en uygun buldu.
  Matching Engine: paralel iş parçacığı → her ikisi de "D-123 müsait" gördü.
  Her ikisi de teklif gönderdi → sürücü her ikisini de kabul etti.
  Sonuç: sürücü iki yolcuya da "geliyor" → ikisi de bekliyor → biri hayal kırıklığı.

Çözüm:
  a) Redis SETNX ile atomik kilit:
     Sürücü ilk teklifi kabul ettiğinde:
     SETNX driver_locked:{driverId} {tripId1} EX 60 → başarılı.
     İkinci tripId2: SETNX → fail → "sürücü zaten alındı" → bu yolcuya başka sürücü.
     Atomik: Redis single-threaded → iki SETNX'ten biri kazanır.

  b) Driver status çift kontrol:
     Matching Engine: GEOSEARCH → sürücü listesi.
     Her sürücüye teklif öncesi: GET driver:{driverId}:status → AVAILABLE mı?
     Değilse → skip → sonraki sürücü.

  c) Optimistic concurrency:
     Trip DB: UPDATE trips SET driver_id=?, status='ACCEPTED'
       WHERE trip_id=? AND driver_id IS NULL AND status='REQUESTED'
     Sürücü iki trip için bu komutu çalıştırıyorsa → ilki günceller, ikincisi 0 satır → fail.

  d) Sürücü onayında dedupe:
     Sürücü "KABUL" tuşuna bastı → POST /trips/{tripId}/accept.
     Server: driver atomik al → sadece ilk başarılı.
     İkinci kabul: "Bu yolculuk başkasına gitti" → sürücüye bildir.
```

---

### 6. Surge Anomalisi — Sürücüler Koordineli Çekildi

```
Sorun:
  Whatsapp grubu: "Hepimiz Taksim'den çekilelim → surge artar → geri gel → daha fazla kazan."
  100 sürücü: aynı anda GeoSET'ten çıktı (telefonu kapattı).
  Surge: 3.0x → sürücüler geri döndü → her biri 2.0x ile çalıştı.
  Yolcular: %200 artış → 10 dk'da fiyat 3x → şikâyet.

Çözüm:
  a) Anomali tespiti (Flink):
     Bölgede sürücü sayısı → 5 dk içinde %30+ ani düşüş → talep artmadan.
     Bu pattern: koordineli çekilme göstergesi → alert.
     Surge: "talep artmadı ama supply düştü" → surge bastır veya sınırla.

  b) Geçmiş supply modeli:
     "Normal Salı 20:00 Taksim: 80 sürücü."
     Şu an 20 sürücü → anomali.
     Surge hesabı: min(actual_supply, historical_expected_supply) kullan.
     Yapay supply düşüşü → surge etkilenmiyor.

  c) Ceza mekanizması:
     Koordineli çekilme tespiti → bu sürücü grubuna bonus iptal.
     Tekrar yapanlar → hesap askıya alma.

  d) Supply garantisi:
     Şehir merkezi için minimum sürücü sayısı SLA.
     Yetersiz → rezerv sürücülere acil incentive ("şimdi gel, garantili ücret").
```

---

### 7. Rota Sapması — Sürücü Kasıtlı Uzun Yol Gidiyor

```
Sorun:
  Sürücü: GPS navigasyon yerine kendi bildiği yolu kullanıyor.
  Meşru neden: trafik, yol çalışması.
  Kötü neden: km başına ücret → kasıtlı uzun rota → daha fazla kazanç.
  Yolcu: "Neden bu kadar sürdü?" → anlamıyor → ama daha fazla ödedi.

Çözüm:
  a) Maksimum ücret koruma:
     Önerilen rota: OSRM → X km, Y TL.
     Maksimum ücret: Y × 1.3 (trafik toleransı %30).
     Sürücü daha uzun gitse bile → yolcu X × 1.3 TL'den fazla ödemez.
     Fark: platform karşılar → sürücüye fazlası ödenmiyor.
     Etki: kasıtlı sapmanın anlamı yok → sürücü uzadıkça kendi zamanını harcıyor.

  b) Rota izleme (anomali):
     Gerçek GPS treki vs önerilen rota → fark.
     Sürekli sapma > 500m → "Önerilen rotadan ayrıldınız" uyarısı → sürücüye.
     Yolcuya: anlık bildirim → "Sürücünüz alternatif rota kullanıyor."

  c) Post-trip analizi:
     Trip tamamlandı → gerçek rota vs optimal rota → verimlilik skoru.
     Düşük verimlilik + yüksek ücret → şüpheli → inceleme.
     Tekrar eden pattern → sürücü uyarısı.

  d) Şikayet mekanizması:
     "Uzun yol kullandı" şikayet butonu → fare adjustment → otomatik iade.
     Rota logu: kanıt → yolcu haklıysa → iade onayı.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Konum depolama | PostgreSQL | Redis GeoSET | PG: yavaş geo query; Redis: 2ms GEOSEARCH |
| Matching | Merkezi tek node | Cell-based sharding (H3) | Tek: darboğaz; H3: bağımsız ölçek |
| Matching algoritma | Sadece mesafe | Çoklu faktör skoru | Mesafe: tek sürücüye yüklenme; Çoklu: adalet |
| Teklif stratejisi | Seri (birer birer) | Paralel (top-3 aynı anda) | Seri: her timeout 30sn kayıp; Paralel: hızlı |
| ETA | Google Maps | OSRM (matching) + HERE (gösterim) | Google: pahalı yüksek QPS; OSRM: ücretsiz, hızlı |
| Sürücü bildirim | HTTP polling | WebSocket + MQTT | Polling: gecikme, yük; WS: anında, verimli |
| Surge hesap | Per-request | Pre-computed (30sn) | Per-req: her ödeme yüklü; Pre-comp: hızlı |
| Konum yayını | HTTP SSE | Redis Pub/Sub + WebSocket | SSE: tek yön; Pub/Sub: çok node dağıt |
| Trip storage | PostgreSQL tek tablo | PG (metadata) + Cassandra (GPS log) | PG: GPS yazma yükü dayanamaz; Cassandra: write-heavy |
| Rota sapma kontrolü | Yok | Max fare cap + post-trip analiz | Yok: fraud riski; Cap: sürücü motivasyonu kalkar |
| Supply forecasting | Reaktif (spike gelince) | Proaktif (ML tahmin + incentive) | Reaktif: peak'te kötü UX; Proaktif: hazır sürücü |
