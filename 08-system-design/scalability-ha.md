# 08b — Scalability Patterns & High Availability

## 1. Horizontal vs Vertical Scaling

```
Vertical (Scale Up):
  4 core, 16GB RAM → 32 core, 256GB RAM
  Avantaj: Basit, uygulama değişmez, DB için ideal (ACID)
  Dezavantaj: Fiziksel limit var, pahalı, SPOF

Horizontal (Scale Out):
  1 sunucu → 100 sunucu
  Avantaj: Teorik sınırsız, HA, maliyet etkin
  Dezavantaj: Stateless gerektirir, koordinasyon karmaşıklığı

Hybrid (gerçek dünya):
  DB: Vertical (güçlü tek node) + Read Replicas (horizontal)
  API: Horizontal (stateless, load balanced)
  Cache: Horizontal cluster (Redis Cluster)
```

---

## 2. Database Sharding Stratejileri

### Range-Based Sharding

```
Shard 0: user_id 1 – 1,000,000
Shard 1: user_id 1,000,001 – 2,000,000
Shard 2: user_id 2,000,001 – ...

✓ Range query kolay (user_id BETWEEN 1 AND 1000000 → tek shard)
✗ Hot spot: Yeni kullanıcılar son shard'da birikir
✗ Shard yeniden dengeleme zor

Kullanım: Zaman serisi (tarih aralığına göre shard), log verileri
```

### Hash-Based Sharding

```
shard = hash(user_id) % shard_count

✓ Uniform dağılım, hot spot yok
✗ Range query imkansız (hepsini tara)
✗ Shard ekleme → rehashing (consistent hashing çözer)

Consistent Hashing ile:
  Shard ekleme → sadece komşu key'ler taşınır (N/shard_count taşınır)
  Normal hash: shard ekleme → hepsini taşı

Kullanım: User data, session, cache
```

### Directory-Based Sharding

```
Lookup service (shard map):
  user_id → shard_id tablosu

✓ En esnek: herhangi key'i herhangi shard'a taşı
✓ Yeniden dengeleme kolay
✗ Lookup service = SPOF (HA yapılmalı)
✗ Her sorgu +1 hop (lookup overhead)

Kullanım: Veri dağılımı öngörülemeyen senaryolar
```

### Shard Key Seçimi

```
İyi shard key:
  ✓ Yüksek kardinalite (100 değil, milyonlarca farklı değer)
  ✓ Uniform dağılım (hot key yok)
  ✓ Query pattern'le uyumlu (sıkça birlikte sorgulanıyorsa aynı shard)

Kötü örnekler:
  ✗ status (ACTIVE/INACTIVE) → sadece 2 değer, hot partition
  ✗ created_at günlük → yeni kayıtlar hep aynı shard
  ✗ country (türkiye) → tek ülke çok büyük → hot

İyi örnekler:
  ✓ user_id (UUID veya high cardinality)
  ✓ tenant_id (multi-tenant sistemlerde)
  ✓ hash(user_id) → uniform
```

---

## 3. Replication Stratejileri

### Primary-Replica (Leader-Follower)

```
Primary: Tüm write'lar buraya
Replica: Read trafiği buraya yönlendir

Senkron replikasyon:
  Primary → Replica'ya yaz → COMMIT → client'a OK
  ✓ Hiç veri kaybı yok
  ✗ Yavaş (replica cevap bekle)

Asenkron replikasyon (default):
  Primary → COMMIT → client'a OK → Arka planda Replica güncelle
  ✓ Hızlı write
  ✗ Primary çökerse replike edilmemiş son data kaybolabilir (RPO > 0)

Yarı-senkron (MySQL semi-sync):
  En az 1 replica yazdıysa COMMIT
  ✓ En az 1 kopya garantili
  ✗ Tüm replica'lardan garanti değil
```

### Multi-Primary (Multi-Master)

```
Her node yazabilir → conflict resolution gerekli

Last-Write-Wins (LWW): timestamp büyük olan kazanır
  ✗ Saat senkronizasyonu şart (NTP + logical clock)

CRDT (Conflict-free Replicated Data Type):
  Counter, Set gibi matematik olarak birleşebilen struct
  CouchDB, Riak, Redis Cluster kullanır
  ✓ Conflict yok (matematiksel garanti)
  ✗ Sadece belirli veri yapıları

Kullanım: Geo-distributed sistemler, offline-first uygulamalar
```

---

## 4. High Availability (HA)

### SLA Hesabı

```
Availability = Uptime / (Uptime + Downtime)

99%     → 3.65 gün/yıl   downtime (basit web)
99.9%   → 8.76 saat/yıl  (e-ticaret)
99.99%  → 52.6 dakika/yıl (fintech, kritik)
99.999% → 5.26 dakika/yıl (telecom, payment)

Seri bileşenler (tüm bileşen çalışmalı):
  A × B → 99.9% × 99.9% = 99.8%
  Her bileşen availability azaltır

Paralel bileşenler (biri çalışıyorsa yeterli):
  1 - (1-A) × (1-B) → 1 - 0.001 × 0.001 = 99.9999%
  Redundancy toplam availability artırır
```

### SPOF Elimination

```
Her katmanda redundancy:

DNS:
  Birden fazla NS kaydı
  GeoDNS: lokasyona göre farklı IP
  TTL düşük tut (failover hızlı): 60-300s

Load Balancer:
  Active-Active: iki LB aynı anda trafiği paylaşır
  Active-Passive: biri devre dışı, diğeri hazır (VRRP/keepalived)
  AWS: ALB zaten managed HA

Application:
  N stateless instance, load balanced
  Health check + auto-restart (Kubernetes liveness probe)
  Graceful shutdown (inflight istekler tamamlansın)

Database:
  Primary + 2 Replica minimum
  Automatic failover: Patroni (PostgreSQL), MySQL InnoDB Cluster
  RDS Multi-AZ: synchronous standby, <30s failover
  Point-in-time recovery: WAL/binlog + hourly snapshot

Cache:
  Redis Sentinel: 1 primary + 2 sentinel (failover yöneticisi)
  Redis Cluster: 6 node (3 primary, 3 replica), otomatik failover

Message Queue:
  Kafka: replication.factor=3, min.insync.replicas=2
  RabbitMQ: Quorum Queue (Raft, 3 node)

Storage:
  S3: 99.999999999% durability (11 nine) — multi-AZ erasure coding
  EBS: Multi-AZ snapshot, cross-region replication

Cross-Region:
  Active-Active multi-region: en yüksek HA, en pahalı
  Active-Passive (pilot light): ana bölge çalışıyor, dr bölge minimal
  Warm standby: dr bölgede küçük kapasitede çalışıyor
```

### Graceful Degradation

```
Normal: Recommendation service çalışıyor → kişisel öneriler
Degraded: Servis yavaş → cached popular ürünler (stale ama çalışıyor)
Offline: Circuit open → "Şu an öneriler yüklenemiyor" (static mesaj)

Fallback hiyerarşisi:
  1. Canlı veri (ideal)
  2. Cache'ten stale veri
  3. DB'den yavaş sorgu
  4. Default değer / empty state
  5. Hata mesajı (son çare)

Feature flag ile:
  ProductService down → featureflag.recommendations=OFF
  UI'da öneri bölümü gizlenir
  Diğer özellikler çalışmaya devam eder
```

### Health Check Patterns

```
Liveness Probe (Kubernetes):
  "Uygulama hayatta mı?" — hayır → restart
  GET /actuator/health/liveness → 200 / 503

Readiness Probe:
  "Uygulama trafik almaya hazır mı?" — hayır → load balancer'dan çıkar
  GET /actuator/health/readiness → 200 / 503
  DB bağlantısı OK? Redis bağlantısı OK? → ready

Startup Probe:
  "Uygulama başlangıç aşamasında mı?"
  Uzun başlangıç süresi olan uygulamalar için (slow start)
  Startup probe OK olmadan liveness/readiness kontrol edilmez

# Spring Boot
management:
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  endpoint:
    health:
      probes:
        enabled: true
```

---

## 5. Deployment Stratejileri

```
Rolling Deploy:
  Instance'ları tek tek güncelle
  ✓ Downtime yok, basit
  ✗ İki versiyon aynı anda çalışıyor (backward compatible olmalı)

Blue-Green:
  Blue (v1 prod) + Green (v2 hazır) → traffic switch → green prod
  ✓ Anlık switch, instant rollback
  ✗ Çift kaynak (2x instance)

Canary:
  %5 traffic v2'ye → metrik izle → sorunsa geri al → %100'e çıkar
  ✓ Risk minimized, gerçek kullanıcıda test
  ✗ Yavaş rollout, iki versiyon aynı anda

Feature Flag:
  Kod deploy edildi ama feature kapalı → aç/kapa runtime'da
  ✓ Deploy ve release bağımsız
  ✓ A/B test
  ✗ Flag yönetimi (LaunchDarkly, Unleash)
```
