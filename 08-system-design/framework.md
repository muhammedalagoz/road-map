# 08a — System Design Framework & Capacity Estimation

## 1. System Design Interview Framework

Her system design sorusuna bu sırayla yaklaş:

```
1. Gereksinimleri netleştir (5 dk)
   - Functional requirements: ne yapmalı?
   - Non-functional: QPS, latency, availability, consistency
   - Scale: kaç kullanıcı? kaç istek/saniye? ne kadar veri?
   - Out of scope: neyi tasarlamıyoruz?

2. Capacity estimation (5 dk)
   - Günlük aktif kullanıcı × ortalama istek = QPS
   - Depolama: veri boyutu × günlük kayıt × yıl

3. High-level tasarım (10 dk)
   - Temel bileşenler: client, API, servis, DB, cache
   - Veri akışını çiz

4. Deep dive (20 dk)
   - Darboğazları tespit et
   - Trade-off'ları tartış
   - Alternatifleri karşılaştır
   - DB schema, API design

5. İyileştirme & Özet (5 dk)
   - SPOF (Single Point of Failure) var mı?
   - Nasıl scale edilir?
   - Monitör nasıl yapılır?
```

---

## 2. Capacity Estimation

### Temel Sayılar (Ezberle)

```
Veri boyutu:
  1 KB  = 1,000 bytes
  1 MB  = 1,000 KB
  1 GB  = 1,000 MB
  1 TB  = 1,000 GB
  1 PB  = 1,000 TB

Latency referansları:
  L1 cache:        1 ns
  L2 cache:        4 ns
  RAM read:        100 ns
  SSD random read: 100 μs   (100,000 ns)
  Aynı DC network: 500 μs   (round-trip ~1ms)
  HDD seek:        10 ms
  Cross-DC:        50-150 ms
  İstanbul→ABD:    ~150 ms

Zaman:
  1 gün  = 86,400 saniye ≈ 100,000 (hesap kolaylığı)
  1 ay   ≈ 3,000,000 saniye
  1 yıl  ≈ 30,000,000 saniye

QPS:
  1 milyon kullanıcı, günde 10 istek → 10M/86400 ≈ 115 QPS
```

---

### Estimation Örnekleri

#### Twitter

```
Varsayım:
  DAU: 300M kullanıcı
  Her kullanıcı: 3 tweet/gün, 30 timeline okuma/gün

Write QPS:
  300M × 3 / 86400 ≈ 10,000 tweet/sn
  Peak (2x): 20,000 tweet/sn

Read QPS:
  300M × 30 / 86400 ≈ 100,000/sn
  (Write:Read = 1:10 — read-heavy sistem)

Depolama:
  Tweet boyutu: 280 char + metadata ≈ 1 KB
  Günlük: 10,000 × 86400 × 1KB = 864 GB/gün
  Medya (tweet'lerin %10'u, ortalama 500KB):
    1000 × 86400 × 500KB ≈ 43 TB/gün

10 yıllık storage:
  Text: 864GB × 365 × 10 ≈ 3.2 PB
  Medya: 43TB × 365 × 10 ≈ 157 PB
```

#### Instagram

```
DAU: 500M
Her kullanıcı: 2 fotoğraf upload/ay, 100 görüntüleme/gün

Upload QPS:
  500M × 2 / 30 / 86400 ≈ 400 fotoğraf/sn

Read QPS:
  500M × 100 / 86400 ≈ 580,000/sn (600K QPS → CDN şart)

Storage:
  Fotoğraf boyutu: ortalama 3 MB (compressed + thumbnail'lar: 5 MB)
  Günlük: 400 × 86400 × 5MB ≈ 172 TB/gün
  10 yıl: 172TB × 365 × 10 ≈ 628 PB
```

---

### Estimation Şablonu

```
1. Users
   MAU (Monthly Active Users): __
   DAU (Daily Active Users): __ (MAU × 0.3 typik)

2. QPS hesabı
   Write: DAU × write/day / 86400 = __ QPS
   Read:  DAU × read/day  / 86400 = __ QPS
   Peak:  avg × 2-3

3. Bandwidth
   Write bandwidth: write_QPS × message_size = __ MB/sn
   Read bandwidth:  read_QPS  × message_size = __ MB/sn

4. Storage
   Günlük: write_QPS × 86400 × message_size = __ GB/gün
   5 yıl:  günlük × 365 × 5 = __ TB

5. Cache
   Read QPS × latency × message_size = RAM gereksinimi
   Hot data: günlük erişimin %20'si → toplam verinin %80'ini kapsar (80/20 rule)
```

---

## 3. Non-Functional Gereksinim Şablonu

```
Availability:
  99.9%   → 8.76 saat/yıl downtime (e-ticaret için yeterli)
  99.99%  → 52 dakika/yıl (fintech, kritik sistemler)
  99.999% → 5 dakika/yıl (telecom, payment)

Consistency:
  Strong: banka, ödeme, inventory (doğru veri > hız)
  Eventual: sosyal medya, öneri, arama (hız > anlık doğruluk)

Latency:
  P50: tipik kullanıcı deneyimi
  P99: en kötü %1 (yavaş kullanıcılar ne görür?)
  P999: outlier (inceleme gerekir mi?)

  Hedefler:
    API response: <100ms P99
    Search:       <200ms P99
    Video start:  <3s (buffering başlayana kadar)

Durability:
  Veri kaybı kabul edilemez → replication factor 3, multi-region backup
  Cache → kaybolabilir (source of truth DB)

Throughput:
  Peak QPS × güvenlik faktörü (2-3x) kapasite planla
```
