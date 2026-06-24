# 08a — System Design Framework & Capacity Estimation

## 1. System Design Interview Framework

Her system design sorusuna bu sırayla yaklaş:

```
1. Gereksinimleri netleştir (5 dk)
   - Functional requirements: ne yapmalı?
   - Non-functional: QPS, latency, availability, consistency
   - Scale: kaç kullanıcı? kaç istek/saniye? ne kadar veri?
   - Out of scope: neyi tasarlamıyoruz?

   Örnek sorular:
     "Kaç aktif kullanıcı bekliyoruz?"
     "Read-heavy mi, write-heavy mi?"
     "Global mi, tek bölge mi?"
     "Consistency mi, availability mi öncelikli?"
     "Mobil, web veya her ikisi mi?"

2. Capacity estimation (5 dk)
   - Günlük aktif kullanıcı × ortalama istek = QPS
   - Depolama: veri boyutu × günlük kayıt × yıl
   - Bandwidth: QPS × mesaj boyutu
   - Cache RAM: hot data × mesaj boyutu

3. High-level tasarım (10 dk)
   - Temel bileşenler: Client → API Gateway → Service → DB/Cache → CDN
   - Veri akışını çiz (read path ve write path ayrı)
   - DB seçimi: SQL mu NoSQL mu?
   - API tipi: REST, GraphQL, gRPC?
   - İlk taslak: basit tut, karmaşıklaştırma

4. Deep dive (20 dk)
   - Darboğazları tespit et (bottleneck)
   - Trade-off'ları tartış: "Şunu seçtim çünkü..." + "Alternatif X, ama..."
   - DB schema, API tasarım, veri akışı detaylandır
   - Ölçeklendirme: sharding, partitioning, read replica
   - Hata senaryoları: bir node düşerse? network split?

5. İyileştirme & Özet (5 dk)
   - SPOF (Single Point of Failure) var mı? nasıl giderilir?
   - Nasıl scale edilir? (horizontal, CDN, cache)
   - Monitoring: hangi metrikler izlenir?
   - Güvenlik: auth, rate limiting, encryption
```

### Mülakat Tuzakları (Yaygın Hatalar)

```
✗ Soru anlamadan tasarıma atlamak
  → Her zaman 5 dk gereksinim topla. "Bir dakika, önce şunu netleştirelim."

✗ Tek doğru cevap var sanmak
  → Her karar trade-off'tur. "X seçtim çünkü Y, ancak Z dezavantajı var."

✗ Ayrıntıya boğulmak (DB schema ilk 5 dk'da)
  → Önce high-level, sonra deep dive. Mülakatta yol haritasını söyle.

✗ Sessiz çalışmak
  → Düşüncelerini sürekli sesli ifade et. Mülakat bir monolog değil, diyalog.

✗ "En iyi" DB'yi sormak
  → Doğru soru: "Bu use case için doğru DB hangisi?"
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

Latency referansları (Jeff Dean Numbers):
  L1 cache:           0.5 ns
  L2 cache:           7 ns
  RAM read:           100 ns       (L1'den 200x yavaş)
  SSD random read:    100 μs       (RAM'den 1,000x yavaş)
  Aynı DC network:    500 μs (RT)
  SSD sequential:     1 GB/sn okuma
  HDD seek:           10 ms        (SSD'den 100x yavaş)
  Cross-DC (örn. İst→Frankfurt): 20-50 ms
  İstanbul → ABD:     ~150 ms
  Mutex lock/unlock:  25 ns

  Temel kural:
    RAM < SSD < Network < HDD
    Cache miss = 100x yavaş

Zaman:
  1 gün  = 86,400 saniye ≈ 100,000 (hesap kolaylığı)
  1 ay   ≈ 2,500,000 saniye ≈ 3,000,000
  1 yıl  ≈ 31,500,000 saniye ≈ 30,000,000

QPS hızlı hesap:
  1M kullanıcı × 1 istek/gün   →  ~12 QPS
  1M kullanıcı × 10 istek/gün  →  ~120 QPS
  10M kullanıcı × 10 istek/gün →  ~1,200 QPS
  100M DAU × 10 istek/gün      →  ~12,000 QPS
  
  Peak = avg × 2-3 (diurnal pattern)
  Black Friday / büyük event = avg × 10+

Bandwidth:
  1 GB/sn = 8 Gbps
  Tipik CDN edge:     100 Gbps
  Tipik DC uplink:    10-40 Gbps
  Tek sunucu NIC:     1-10 Gbps
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
  Text:  864GB × 365 × 10 ≈ 3.2 PB
  Medya: 43TB × 365 × 10 ≈ 157 PB
```

#### Instagram

```
DAU: 500M
Her kullanıcı: 2 fotoğraf upload/ay, 100 görüntüleme/gün

Upload QPS:
  500M × 2 / 30 / 86400 ≈ 400 fotoğraf/sn

Read QPS:
  500M × 100 / 86400 ≈ 580,000/sn (CDN şart)

Storage:
  Fotoğraf: ortalama 3 MB (compressed + thumbnails: 5 MB)
  Günlük: 400 × 86400 × 5MB ≈ 172 TB/gün
  10 yıl: 172TB × 365 × 10 ≈ 628 PB
```

#### YouTube

```
DAU: 50M
Upload: 500 saat video/dakika → 8.3 saat/sn upload
İzleme: 1B saat izleme/gün

Upload QPS:
  500 × 60 / 60 = 500 dakikalık video/sn
  (Her dakika video = 3 boyut × 300 MB → 900 MB/dakika-video)
  Bandwidth: 500 × 900 MB/dk / 60 sn ≈ 7.5 GB/sn upload

İzleme QPS:
  1B saat/gün = 1B × 3600 sn / 86400 ≈ 41M saniye video/sn izleme
  1 video stream: ~4 Mbps (1080p)
  Toplam bandwidth: 41M × 4Mbps = 164 Tbps (CDN olmadan imkansız)

Storage:
  Orijinal: 500 × 60 × 60 × 10 GB/saat ≈ 300 TB/gün (raw)
  Transcoding: 3 kalite × 50% sıkıştırma = orijinalin 1.5x
  CDN cache: son 30 gün, en popüler %10 → ~petabyte
```

#### WhatsApp

```
DAU: 1B
Her kullanıcı: 30 mesaj/gün

Write QPS (mesaj gönder):
  1B × 30 / 86400 ≈ 350,000 mesaj/sn
  Peak (sabah 9, akşam 6): 700,000/sn

Read QPS (mesaj al):
  Grup mesajı ort. 5 kişiye → delivery: 350K × 5 = 1.75M/sn

Storage:
  Mesaj boyutu: 1 KB (metin) + medya (ayrı)
  Günlük: 350,000 × 86400 × 1KB ≈ 30 TB/gün (metin)
  Medya: %20 mesaj × 50 KB = 350K × 0.2 × 86400 × 50KB ≈ 300 TB/gün
  WhatsApp politikası: mesajlar end-to-end → server'da geçici (iletilince sil)
```

#### Uber

```
DAU: 20M sürücü + 50M yolcu
Peak: sabah 8-9, akşam 17-19

Konum güncellemesi (sürücü):
  20M sürücü × her 5 sn güncelleme = 4M/sn write (yüksek!)

Yolcu eşleştirme (matching):
  50M yolcu × 2 yolculuk/gün / 86400 ≈ 1,200 eşleştirme/sn
  Peak: 5,000/sn

Geospatial sorgu:
  Her eşleştirme: "5 km içindeki sürücüler" → spatial index (QuadTree/S2)
  1,200 sorgu/sn × 20 ms latency → 24 sn toplam işlem → paralel gerekli

Storage:
  Konum: 20M × 24 saat × 12 güncelleme = 5.7B satır/gün
  Yolculuk: 1,200/sn × 86400 = 103M yolculuk/gün × 1KB = 103 GB/gün
  Geçmiş (5 yıl): 103GB × 365 × 5 ≈ 188 TB
```

---

### Estimation Şablonu

```
1. Users
   MAU (Monthly Active Users):  __M
   DAU (Daily Active Users):    __M  (MAU × 0.3 tipik)
   Peak:                        DAU × 2-3

2. QPS hesabı
   Write: DAU × write/day / 86400 = __ QPS
   Read:  DAU × read/day  / 86400 = __ QPS
   Peak:  avg × 2-3 (× 10 büyük event)

3. Bandwidth
   Write bandwidth: write_QPS × message_size = __ MB/sn
   Read bandwidth:  read_QPS  × message_size = __ MB/sn
   CDN gerekli?: read bandwidth > 1 Gbps → CDN zorunlu

4. Storage
   Günlük:  write_QPS × 86400 × message_size = __ GB/gün
   5 yıl:   günlük × 365 × 5 = __ TB
   Medya:   ayrı hesapla (S3 + CDN)

5. Cache
   Hot data: günlük erişimin %20'si → toplam verinin %80'ini kapsar (80/20 rule)
   Cache boyutu: daily_read × hit_rate × message_size / 86400
   Örn: 1M obj × 1KB = 1 GB RAM (Redis)

6. Sunucu sayısı
   1 sunucu ≈ 1,000-10,000 QPS (işlem ağırlığına göre)
   QPS / sunucu_kapasitesi = sunucu sayısı
   × 2 güvenlik faktörü
```

---

## 3. Non-Functional Gereksinim Şablonu

```
Availability (SLA):
  99.9%   → 8.76 saat/yıl downtime  (e-ticaret, blog)
  99.99%  → 52 dakika/yıl           (fintech, kritik sistemler)
  99.999% → 5 dakika/yıl            (telecom, payment, hayati)

  Formül: Downtime = (1 - availability) × yıldaki dakika
  99.99% = 0.0001 × 525,600 dk = 52.56 dk/yıl

Consistency (CAP seçimi):
  Strong:   banka, ödeme, inventory, rezervasyon → doğru veri > hız
  Eventual: sosyal medya, öneri, arama, analytics → hız > anlık doğruluk
  Karar: "Birkaç saniyelik tutarsızlık iş sürecini etkiler mi?"

Latency (percentile):
  P50:  tipik kullanıcı deneyimi (medyan)
  P99:  en kötü %1 → yavaş kullanıcılar ne görür?
  P999: %0.1 outlier → inceleme gerekir mi?

  Hedefler:
    API response:    P99 < 100ms
    Search:          P99 < 200ms
    DB query:        P99 < 50ms (index'li)
    Video start:     P95 < 3s (buffer başlayana kadar)
    Payment:         P99 < 5s (user tolerance yüksek)

Durability:
  Veri kaybı sıfır:  RF=3, multi-region, WAL
  Cache kaybolabilir: source of truth DB
  RPO (Recovery Point Objective): ne kadar veri kayıplarız?
  RTO (Recovery Time Objective):  ne kadar sürede toparlarız?

Scalability:
  Horizontal: node ekle → kapasite artar (stateless servisler)
  Vertical:   daha güçlü sunucu → tek node limiti var
  Hedef: peak × 3 kapasitede sorunsuz çalış

Throughput:
  Peak QPS × güvenlik faktörü (2-3x) kapasite planla
  Burst absorb: queue, cache, circuit breaker
```

---

## 4. API Tasarım Seçimi

### REST vs GraphQL vs gRPC

```
REST:
  Güçlü yanlar:
    ✓ Standart, herkes bilir (HTTP)
    ✓ Browser native, curl ile test edilebilir
    ✓ Stateless, cache dostu (GET → CDN)
    ✓ Ekosistem: Swagger, Postman, API Gateway

  Zayıf yanlar:
    ✗ Over-fetching: ihtiyaç olmayan field'lar geliyor
    ✗ Under-fetching: birden fazla endpoint çağırmak gerekiyor
    ✗ Versioning sorunu: /v1, /v2, /v3 proliferasyonu

  Ne zaman kullan:
    Genel amaçlı public API
    CRUD tabanlı servisler
    Browser tabanlı client

GraphQL:
  Güçlü yanlar:
    ✓ Client istediği field'ı ister → over-fetch yok
    ✓ Tek endpoint: /graphql
    ✓ Schema = documentation + validation (otomatik)
    ✓ Nested query: tek istekte ilişkili veri

  Zayıf yanlar:
    ✗ Complexity: N+1 query sorunu (DataLoader gerekli)
    ✗ Cache zor: GET değil POST → CDN cache yok
    ✗ Rate limiting: farklı maliyetli query'ler → complexity limit
    ✗ File upload: özel çözüm gerekli

  Ne zaman kullan:
    BFF (Backend for Frontend) katmanı
    Mobile app (bandwidth kritik)
    Karmaşık, iç içe veri modeli

gRPC:
  Güçlü yanlar:
    ✓ Protocol Buffers: binary → 10x daha hızlı/küçük
    ✓ Bidirectional streaming (client stream, server stream)
    ✓ Strongly typed: .proto → kod generate
    ✓ HTTP/2: multiplexing, header compression

  Zayıf yanlar:
    ✗ Browser desteği kısıtlı (gRPC-Web gerekli)
    ✗ Human-readable değil (binary)
    ✗ Debug zor (Wireshark/gRPC tooling)

  Ne zaman kullan:
    Servisler arası iletişim (internal microservices)
    Yüksek throughput (streaming, real-time)
    Çok dilli ekip (proto dosyası → tüm dillerde client/server)

Karar Matrisi:
  Public API        → REST
  Mobile BFF        → GraphQL
  Internal servis   → gRPC
  Real-time/stream  → gRPC (bidirectional) veya WebSocket
  Simple CRUD       → REST
```

```
REST API Tasarım Kuralları:
  URL: kaynak ismi, fiil değil
    ✓ GET /users/{id}/orders
    ✗ GET /getUserOrders

  HTTP methodlar:
    GET:    okuma, idempotent, cache'lenebilir
    POST:   oluşturma, non-idempotent
    PUT:    tam güncelleme, idempotent
    PATCH:  kısmi güncelleme
    DELETE: silme, idempotent

  Status code:
    200 OK          → başarılı GET/PUT/PATCH
    201 Created     → başarılı POST (Location header)
    204 No Content  → başarılı DELETE
    400 Bad Request → invalid input
    401 Unauthorized → auth gerekli (token yok/geçersiz)
    403 Forbidden   → auth var ama yetki yok
    404 Not Found   → kaynak yok
    409 Conflict    → tutarsızlık (unique constraint)
    422 Unprocessable Entity → validation hatası
    429 Too Many Requests → rate limit
    500 Internal Server Error → sunucu hatası

  Pagination:
    Cursor-based: ?after=eyJpZCI6MTIzfQ (büyük veri seti, social feed)
    Offset-based: ?page=2&size=20 (admin dashboard, small set)
    Keyset:       ?after_id=123 (monotonic ID → hızlı)
```

---

## 5. Veritabanı Seçimi

### Karar Ağacı

```
Soru 1: ACID transaction gerekli mi?
  Evet → SQL (PostgreSQL, MySQL)
    - Banka, sipariş, ödeme
    - Complex JOIN gerekiyorsa
    - Foreign key integrity şart

Soru 2: Çok büyük veri + yatay ölçek gerekli mi?
  Evet → NoSQL
    Document:  MongoDB, DynamoDB
    Column:    Cassandra, HBase
    Key-Value: Redis, DynamoDB
    Search:    Elasticsearch
    Graph:     Neo4j, Amazon Neptune
    Time-series: InfluxDB, TimescaleDB

Soru 3: Okuma/yazma oranı?
  Read-heavy  → Redis cache + DB (primary/replica)
  Write-heavy → Cassandra / DynamoDB (LSM tree, optimized write)
  Mixed       → PostgreSQL + Redis (hybrid)

Soru 4: Query pattern sabit mi?
  Sabit (by userId, by timestamp) → Cassandra (partition key)
  Esnek (complex filter, full-text) → PostgreSQL + Elasticsearch

Soru 5: Ne tür veri?
  İlişkisel → PostgreSQL
  Doküman (JSON, nested) → MongoDB, DynamoDB
  Zaman serisi → InfluxDB, TimescaleDB
  Graf ilişkileri → Neo4j
  Önbellek, session → Redis
  Analitik (OLAP) → ClickHouse, BigQuery, Redshift
  Full-text arama → Elasticsearch
```

### Veritabanı Hızlı Referans

| DB | Ne Zaman | Güçlü | Zayıf |
|-------|----------|-------|-------|
| PostgreSQL | ACID, complex query, genel amaç | ACID, JSONB, extension zengin, matüre | Yatay write scale zor |
| MySQL | Read-heavy web app | Read replica, geniş ekosistem | Complex query'de PG geride kalır |
| MongoDB | Esnek şema, doküman | Horizontal scale, BSON, atlas | JOIN yok, eventual consistency |
| Cassandra | Time-series, yüksek write | Masterless, linear scale, 99.99% up | Query esnekliği az, eventual |
| Redis | Cache, session, pub-sub, rate limit | Sub-ms latency, veri yapısı zengin | RAM sınırı, persistence opsiyonel |
| Elasticsearch | Full-text search, log, analitik | Relevance scoring, aggregation | Write ağır, eventual, kaynak tüketimi |
| ClickHouse | OLAP, analytics, time-series | Columnar, 100x sorgu hızı | OLTP uygun değil, JOIN kısıtlı |
| DynamoDB | AWS ekosistemi, serverless | Managed, autoscale, single-digit ms | Vendor lock-in, query kısıtlı |
| Neo4j | Graf ilişkileri, öneri | Cypher query, ilişki traversal | Büyük ölçekte yavaşlar |
| InfluxDB | Metrik, IoT, monitoring | Time-series optimize, compaction | Time-series dışı sorgu yetersiz |

### Sharding Stratejileri

```
Hash Sharding:
  shard_id = hash(user_id) % N
  Veri: uniform dağılım
  Sorun: N değişirse → tüm veri yeniden dağıtılır
  Çözüm: consistent hashing (N değişince az veri taşınır)

Range Sharding:
  Shard 1: user_id 1-1M
  Shard 2: user_id 1M-2M
  Avantaj: range query efisyan
  Sorun: hotspot (yeni kullanıcılar hep son shard'a)

Directory-Based Sharding:
  Lookup tablosu: user_id → shard_id
  Avantaj: esnek, rebalance kolay
  Sorun: lookup tablosu SPOF

Geo Sharding:
  Kullanıcı lokasyonuna göre: TR→shard1, EU→shard2, US→shard3
  Avantaj: GDPR uyumu, latency
  Sorun: cross-region query zor
```

---

## 6. Cache Stratejileri

```
Cache-Aside (Lazy Loading) — En yaygın:
  Read: önce cache kontrol → miss ise DB'den al → cache'e yaz.
  Write: DB güncelle → cache'i sil (invalidate).
  Kullanım: kullanıcı profili, ürün detayı.
  Dezavantaj: cold start (ilk istek yavaş), stale data riski.

Write-Through:
  Write: hem cache hem DB'ye eşzamanlı yaz.
  Read: cache her zaman güncel → miss oranı düşük.
  Kullanım: sık okunan, sık güncellenen (stok sayısı).
  Dezavantaj: her write 2 operasyon → write latency artar.

Write-Behind (Write-Back):
  Write: önce cache'e yaz → async batch olarak DB'ye.
  Avantaj: write latency çok düşük.
  Dezavantaj: cache crash → DB'ye gitmeyen veri kaybolur.
  Kullanım: analytics event, log toplama.

Read-Through:
  Cache katmanı DB'yi doğrudan bilir.
  Miss olunca cache kendi DB'den yükler (uygulama değil).
  Kullanım: ORM level cache (Hibernate 2nd level).

Refresh-Ahead:
  TTL dolmadan önce background'da yenile.
  Avantaj: kullanıcı asla cache miss yaşamaz.
  Dezavantaj: bazı veri gereksiz yere yenilenir.
  Kullanım: hot data (ana sayfa banner, trend hashtag).
```

```
Cache Eviction Policy:
  LRU (Least Recently Used):  en uzun süredir kullanılmayanı at.
    Kullanım: genel amaç, user profile cache.
  LFU (Least Frequently Used): en az erişileni at.
    Kullanım: sık değişen popularite (trending topic).
  TTL (Time To Live):  süre dolunca at.
    Kullanım: session, JWT blacklist, rate limit counter.
  FIFO:  ilk gireni at.
    Kullanım: basit kuyruk senaryoları.

Cache boyutlandırma:
  Hot data genellikle toplam verinin %20'si → trafiğin %80'ini kapsar.
  Hedef cache hit rate: %95+
  Hit rate < %80 → cache boyutu yetersiz veya key dağılımı kötü.

Redis vs Memcached:
  Redis:      veri yapısı zengin (Hash, List, ZSet, Stream), persistence, Lua script, cluster.
  Memcached:  basit key-value, çok thread (CPU yoğun workload'da bazen hızlı), daha az feature.
  Seç:        Redis (neredeyse her durumda artık).
```

---

## 7. Ölçeklendirme Pattern'ları

### Horizontal vs Vertical

```
Vertical (Scale Up):
  Daha büyük sunucu: 8 CPU → 32 CPU, 16 GB → 128 GB RAM.
  Avantaj: basit, code değişikliği yok.
  Dezavantaj: tek nokta arıza, hardware limiti, pahalı.
  Ne zaman: database (sharding öncesi), prototip.

Horizontal (Scale Out):
  Daha fazla sunucu: 2 instance → 20 instance.
  Avantaj: teorik sonsuz ölçek, HA, maliyet esnek.
  Dezavantaj: stateless olmalı, load balancer gerekli, veri tutarlılığı zor.
  Ne zaman: API servisi, uygulama katmanı, cache cluster.

Genel kural:
  Uygulama katmanı → horizontal (stateless, istenen kadar).
  DB write → vertical (önce) → sonra sharding.
  DB read → read replica (horizontal read).
```

### Load Balancer Stratejileri

```
Round Robin:    eşit dağıt. Sunucular eşit kapasitedeyse ideal.
Weighted RR:    güçlü sunucuya daha fazla trafik.
Least Conn:     en az aktif bağlantıya gönder. DB connection ağırlıklı işlemde iyi.
IP Hash:        kullanıcı her zaman aynı sunucuya → sticky session (stateful için).
Random:         basit, yük eşit dağılımdaysa iyi.
Health Check:   sağlıksız node'a trafik gönderme (L4 ve L7 check).
```

### Read Replica

```
Write: primary (master).
Read:  replica(lar) → okuma yükü dağıtılır.
Gecikme: async replication → replica birkaç ms geride (replication lag).

Okuma nereye gidecek?
  Stale okuma kabul edilirse → replica.
  Read-your-write (az önce yazdım, okuyorum) → primary.
  Kritik veri (stok, bakiye) → primary.
  Raporlama, analytics → replica (hatta ayrı analytic DB).
```

---

## 8. High Availability Pattern'ları

### SPOF Tespiti ve Giderme

```
Her bileşeni tek tek sorgula: "Bunu kaldırırsam sistem çalışır mı?"

Yaygın SPOF'lar ve çözümleri:

  Load Balancer:
    SPOF: tek LB → L1/L2 LB pair (active-passive) veya anycast.
    Çözüm: HAProxy/Nginx pair, cloud LB (AWS ALB → managed HA).

  Database:
    SPOF: tek DB node.
    Çözüm: primary + replica + automatic failover (Patroni/RDS Multi-AZ).
    RPO: replication lag (~sn), RTO: failover ~30sn.

  Cache (Redis):
    SPOF: tek Redis.
    Çözüm: Redis Sentinel (3 node, otomatik failover) veya Redis Cluster.

  Message Queue:
    SPOF: tek Kafka broker.
    Çözüm: RF=3 (3 farklı broker'a replika), min.insync.replicas=2.

  Single Region:
    SPOF: veri merkezi arızası.
    Çözüm: multi-region active-passive veya active-active.
    RTO: active-passive: 15-30 dk, active-active: sn.

Circuit Breaker Pattern:
  Downstream servis yavaşladı → sınırsız istek gönderme → cascade failure.
  Circuit Breaker: başarısızlık eşiği aşılırsa → OPEN → istek kesme → fallback.
  Durum: CLOSED (normal) → OPEN (yavaş, kesme) → HALF-OPEN (test et).
  Kütüphane: Resilience4j (Java), Hystrix (deprecated), Polly (.NET).

Bulkhead Pattern:
  Bir özelliğin kaynak tüketimi diğerini boğmasın.
  Thread pool isolation: ödeme servisi için 20 thread, öneri için 5 thread.
  Sorun: öneri servisi çöktüğünde ödeme etkilenmiyor.
```

---

## 9. Yaygın Tasarım Pattern'ları

### CQRS (Command Query Responsibility Segregation)

```
Sorun: Aynı model hem okuma hem yazma için kullanılıyor → her ikisi için suboptimal.

Çözüm:
  Command (Write):  OrderService.placeOrder() → PostgreSQL (normalized, ACID).
  Query (Read):     OrderQueryService.getOrders() → Elasticsearch/Redis (denormalized, hızlı).

Avantaj:
  Write tarafı: consistency için optimize.
  Read tarafı: query için optimize (flat, indexed).
  Ölçek: read ve write bağımsız scale edilir.

Dezavantaj:
  Eventual consistency: write → event → read model güncelle → kısa gecikme.
  Karmaşıklık: iki model, iki katman, senkronizasyon.

Ne zaman kullan:
  Read QPS >> Write QPS (10:1+).
  Read ve write gereksinimleri çok farklı.
  Raporlama ve OLTP aynı DB'de çakışıyor.
```

### Event Sourcing

```
Sorun: "Sipariş bakiyesi şu anda 1500₺. Neden?" → geçmiş bilgi yok.

Çözüm:
  State saklamak yerine eventleri sakla.
  events tablosu: [OrderCreated, ItemAdded, CouponApplied, OrderConfirmed, ...]
  Mevcut state: eventleri baştan uygula (fold/reduce).

Avantaj:
  Tam denetim izi (audit log ücretsiz).
  Zaman yolculuğu: "Dün saat 14'te ne durumdaydı?"
  Replay: yeni read model → eventleri yeniden işle.
  Event-driven integration: event'ler diğer servislere gönderilir.

Dezavantaj:
  Snapshot gerekir (10,000 event'i her seferinde replay etme).
  Query zor: "Tüm aktif siparişleri getir" → read model şart (CQRS ile birlikte).
  Event schema değişikliği → migration zor.

Ne zaman kullan:
  Audit log kritik (fintech, sağlık, e-devlet).
  Complex iş akışı geçmişi.
  Event-driven microservice entegrasyonu.
```

### Saga Pattern

```
Sorun: Birden fazla servisi kapsayan distributed transaction nasıl yönetilir?

Choreography Saga:
  Her servis event yayar → diğer servis dinler → kendi işini yapar.
  ✓ Loosely coupled.
  ✗ Akış görünürlüğü zor, debug zor.

Orchestration Saga:
  Merkezi orchestrator → her adımı sıraya koyar.
  ✓ Akış açık, merkezi monitoring.
  ✗ Orchestrator SPOF, bağımlılık artar.

Compensating Transaction:
  Adım başarısız → önceki adımları geri al.
  Geri alınamaz işlem (e-posta gönder) → idempotent / "saga already completed" guard.

Ne zaman kullan:
  Birden fazla servisin atomik görünmesi gerektiğinde.
  Distributed transaction (2PC) kullanmak istemiyorsak.
```

### Outbox Pattern

```
Sorun: DB güncelle + event yayımla → atomik değil.
  DB güncellendi → uygulama çöktü → event gitmedi → tutarsızlık.

Çözüm:
  Aynı DB transaction'ında:
    1. Business tablosuna yaz.
    2. Outbox tablosuna event yaz.
  Ayrı outbox publisher process:
    Outbox'ı polling → Kafka'ya publish → başarılıysa işaretle.

Outbox tablosu:
  (event_id, event_type, payload, created_at, published_at)

Garantisi: at-least-once delivery (idempotency karşıya bırakılır).
Ne zaman: event yayımı + DB güncellemesi aynı anda şarttır.
```

### BFF (Backend for Frontend)

```
Sorun: Tek API → hem web hem mobile → over-fetch (mobile çok veri alıyor) + under-fetch (web az).

Çözüm:
  Web BFF:    web'e özel response şekli, tam veri, desktop için.
  Mobile BFF: hafifletilmiş response, bandwidth düşük, pagination agresif.
  Her BFF:    downstream servisleri compose eder.

Avantaj:
  Her client kendi ihtiyacına göre API alır.
  Backend servisleri client'tan izole → değişiklik kolaylaşır.
  Authentication, rate limiting, auth BFF'de → servislerde tekrar yok.

Ne zaman:
  Birden fazla client türü (web + mobile + TV + IoT).
  API Gateway yetmiyorsa (composition logic gerekiyorsa).
```

---

## 10. Deep Dive Checklist

Her system design sunumunda şunları ele al:

```
✓ Data Model:
  Hangi tablolar/collection'lar?
  Primary key, index stratejisi?
  Partitioning/sharding key?

✓ API Tasarım:
  Endpoint listesi (method + URL + request/response)
  Auth: JWT, OAuth, API key?
  Versioning: /v1/, header, query param?

✓ Scalability:
  Hangi bileşen en çok yük alıyor?
  Nasıl horizontal scale edilir?
  DB: sharding mı, read replica mı?
  Stateless mu? Load balancer arkasına girebilir mi?

✓ Availability:
  SPOF nerede?
  Failover mekanizması: otomatik mi, manuel mi?
  RTO ve RPO hedefleri nedir?

✓ Consistency:
  Strong mı eventual mı? Neden?
  Write sonrası read: stale okuma olabilir mi? Kabul edilebilir mi?
  Conflict resolution (multi-master): last-write-wins? version vector?

✓ Caching:
  Ne cache'lenir? (hot objects)
  Cache invalidation: TTL? Event-based?
  Stampede koruması var mı?

✓ Messaging:
  Async event mi, sync call mı?
  Event format: Avro/Protobuf (şema evrim) vs JSON?
  Guaranteed delivery? Exactly-once?

✓ Monitoring:
  Hangi metrikler? (QPS, error rate, P99 latency)
  Alarm thresholds?
  Tracing (OpenTelemetry) ve logging?

✓ Security:
  Auth/Authz: kim erişebilir?
  Data encryption: at-rest, in-transit?
  Rate limiting: DDoS koruması?
  PII: hangi veri maskelenir / silinir?
```

---

## 11. Hızlı Karar Tablosu

| Gereksinim | Çözüm |
|------------|-------|
| Çok hızlı okuma | Redis cache + CDN |
| Çok fazla write | Cassandra / DynamoDB + LSM tree |
| Full-text search | Elasticsearch |
| ACID transaction | PostgreSQL |
| Büyük dosya depolama | S3 + CDN |
| Event streaming | Kafka |
| Async task | RabbitMQ / SQS / Kafka |
| Real-time bidirectional | WebSocket |
| Server push (tek yön) | SSE (Server-Sent Events) |
| Rate limiting | Redis token bucket (INCR + TTL) |
| Session | Redis (TTL, key = sessionId) |
| Distributed lock | Redis SETNX / Redlock |
| Geospatial sorgu | PostGIS / Elasticsearch geo / QuadTree |
| Recommendation | Spark ALS + Redis precomputed |
| Analytics/OLAP | ClickHouse / BigQuery |
| Config/feature flag | ZooKeeper / etcd / LaunchDarkly |
| Secret yönetimi | HashiCorp Vault / AWS Secrets Manager |
| Service discovery | Consul / Kubernetes DNS |
| API Gateway | Kong / AWS API Gateway / Nginx |
| Circuit breaker | Resilience4j / Istio (service mesh) |
