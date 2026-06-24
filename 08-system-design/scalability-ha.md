# 08b — Scalability Patterns & High Availability

## 1. Horizontal vs Vertical Scaling

```
Vertical (Scale Up):
  4 core, 16GB RAM → 32 core, 256GB RAM
  Avantaj: Basit, uygulama değişmez, tek makine ACID kolaylaşır
  Dezavantaj: Fiziksel sınır, pahalı, SPOF (tek node)
  Kullanım: DB primary, ML inference node (GPU)

Horizontal (Scale Out):
  1 sunucu → 100 sunucu
  Avantaj: Teorik sınırsız, HA, maliyet etkin (commodity hw)
  Dezavantaj: Stateless gerektirir, koordinasyon karmaşıklığı, dağıtık sistemin tüm zorlukları
  Kullanım: API katmanı, cache cluster, Kafka broker

Hybrid (gerçek dünya):
  DB: Vertical primary (güçlü tek node) + Horizontal read replica
  API: Horizontal stateless, load balanced
  Cache: Horizontal cluster (Redis Cluster)
  Search: Elasticsearch horizontal (shard + replica)

Ölçek kararı matrisi:
  Durum               → Tercih
  Yeni ürün, düşük trafik → Vertical (basitlik önce)
  Trafik 10x arttı       → Horizontal API + Read replica
  DB bottleneck          → Connection pool → Read replica → Sharding
  Global kullanıcı       → Multi-region horizontal
```

---

## 2. Load Balancing

### Algoritmaları

```
Round Robin:
  İstek 1 → Server A, İstek 2 → Server B, İstek 3 → Server C, İstek 4 → Server A...
  ✓ Basit, uniform dağılım.
  ✗ Server kapasiteleri farklıysa adil değil (2 CPU → 16 CPU, aynı yük).

Weighted Round Robin:
  Server A (2 CPU): weight=1, Server B (8 CPU): weight=4.
  Her 5 istekten: 1'i A'ya, 4'ü B'ye.
  ✓ Kapasite farkını dengeler.
  ✗ Anlık yük durumunu görmez.

Least Connections:
  Anlık en az aktif bağlantıya sahip server'a yönlendir.
  ✓ Yavaş işleyen server'a daha az yük → dinamik denge.
  ✗ Bağlantı sayısı ≠ CPU yükü (bazı bağlantılar ağır).
  Kullanım: long-lived connection (WebSocket, gRPC streaming).

IP Hash (Sticky Session):
  hash(client_ip) % server_count → her zaman aynı server.
  ✓ Durumsuz olmayan servisler için (session).
  ✗ Server eklenince tüm yönlendirmeler değişir → consistent hashing çözer.
  ✗ Bir IP'den çok trafik → hot server.

Random:
  Rastgele server seç.
  Yeterli sayıda istek → pratikte round robin'e yakınsar.
  Basit, çok sık kullanılmaz.

Layer 4 vs Layer 7 LB:
  L4 (TCP/UDP): IP + Port bazında.
    Avantaj: çok hızlı, düşük overhead.
    Dezavantaj: HTTP header göremez, cookie-based sticky yok.
    Araç: HAProxy (stream mode), AWS NLB.
  
  L7 (HTTP/HTTPS): URL path, header, cookie bazında.
    Avantaj: path-based routing (/api → API pod, /static → CDN).
    Avantaj: SSL termination, gzip, WebSocket upgrade.
    Dezavantaj: biraz daha yavaş (HTTP parse etmesi gerekiyor).
    Araç: Nginx, HAProxy (HTTP mode), AWS ALB, Envoy.

Sağlık kontrolü + otomatik çıkarma:
  LB: her 5sn → GET /health → 200 değilse → pool'dan çıkar.
  Thresholds: 3 ardışık başarısız → çıkar, 2 ardışık başarılı → geri al.
```

### Health Check Patterns

```
Liveness Probe (Kubernetes):
  "Uygulama hayatta mı?" → hayır → restart.
  Sadece uygulama donmuş mu kontrol et.
  DB bağlantısı başarısız → liveness FALSE yapma! (DB sorununda tüm podlar restart).
  GET /actuator/health/liveness → 200 (her zaman, sadece JVM hayattaysa).

Readiness Probe:
  "Trafik almaya hazır mı?" → hayır → LB'den çıkar (restart değil).
  DB bağlantısı OK + Redis OK + dependency'ler OK → ready.
  GET /actuator/health/readiness → 200 / 503.
  Kullanım: startup, maintenance, graceful shutdown.

Startup Probe:
  Yavaş başlayan uygulama (Spring Boot: 30sn) → liveness başlamadan önce.
  Startup OK olmadan liveness/readiness çalışmaz.
  failureThreshold: 30, periodSeconds: 10 → 5dk timeout.

Deep Health Check (dependency'ler):
  /health/deep: DB ping + Redis ping + Kafka connectivity.
  Kullanım: monitoring dashboard, not for LB (bir dependency bozuksa tüm instance çıkma).
  LB: sadece shallow health check.

# Spring Boot Actuator
management:
  health:
    livenessstate.enabled: true
    readinessstate.enabled: true
    db.enabled: true        # Readiness'a dahil (DB check)
    redis.enabled: true     # Readiness'a dahil
  endpoint.health.probes.enabled: true
```

---

## 3. Database Sharding Stratejileri

### Range-Based Sharding

```
Shard 0: user_id 1 – 1,000,000
Shard 1: user_id 1,000,001 – 2,000,000
Shard 2: user_id 2,000,001 – ...

✓ Range query kolay → tek shard.
✓ Belirli ID aralığına shard ekleme kolayca yapılabilir.
✗ Hot spot: yeni kullanıcılar son shard'da birikir → tüm write yükü oraya.
✗ Shard yeniden dengeleme zor (range'i böl → veri taşı).

Kullanım: Zaman serisi verisi (tarih aralığına göre), arşiv, log.
```

### Hash-Based Sharding

```
shard_id = hash(user_id) % shard_count

✓ Uniform dağılım, hot spot yok.
✗ Range query → tüm shard'ları tara.
✗ Shard ekleme → rehashing → büyük veri taşıma.

Consistent Hashing ile çözüm:
  Hash ring: 0 - 2^32 arası sanal ring.
  Her shard: ring üzerinde birkaç nokta (virtual node).
  Key → ring üzerinde konumu → saat yönünde ilk shard.
  Shard ekleme: sadece komşu key'ler taşınır (N/shard_count).
  Normal hash: shard ekleme → hepsini yeniden dağıt.

Kullanım: User data, session, cache.
```

### Directory-Based Sharding

```
Lookup service (shard map):
  user_id → shard_id tablosu.

✓ En esnek: herhangi key'i herhangi shard'a taşı.
✓ Yeniden dengeleme kolay (sadece map güncelle).
✗ Lookup service = SPOF (HA yapılmalı, Redis ile cache).
✗ Her sorgu +1 hop (lookup overhead → Redis: 1ms).

Kullanım: Veri dağılımı öngörülemeyen, büyük müşteri varlığı.
```

### Shard Key Seçimi

```
İyi shard key:
  ✓ Yüksek kardinalite (milyonlarca farklı değer).
  ✓ Uniform dağılım (hot key yok).
  ✓ Query pattern'le uyumlu (birlikte sorgulanıyorsa aynı shard).
  ✓ Değişmez (shard key değişince veri taşıma gerekir — kaçın).

Kötü örnekler:
  ✗ status (ACTIVE/INACTIVE) → 2 değer → hot partition.
  ✗ created_at (günlük) → yeni kayıtlar hep aynı shard.
  ✗ country (Türkiye) → tek ülke çok büyük → hot.

İyi örnekler:
  ✓ user_id (UUID veya yüksek kardinalite int).
  ✓ tenant_id (multi-tenant: her müşteri kendi shard'ında).
  ✓ hash(user_id) → uniform.

Cross-shard query sorunu:
  "Son 30 günde tüm siparişleri bul" → user_id shard → tek shard.
  "Tüm kullanıcıların toplam harcaması" → tüm shard → scatter-gather.
  Scatter-gather: N shard'a paralel sorgu → merge → yavaş, pahalı.
  Çözüm: aggregation için Data Warehouse / ClickHouse (shard'ları kopyala).
```

---

## 4. Replication Stratejileri

### Primary-Replica (Leader-Follower)

```
Primary: tüm write'lar.
Replica: read trafiği (70-80% okuma ağırlıklı sistemlerde etkili).

Senkron replikasyon:
  Primary → Replica'ya yaz → Replica ACK → COMMIT → client OK.
  ✓ Sıfır veri kaybı (RPO=0).
  ✗ Yavaş (replica cevabını bekle) → write latency artar.
  ✗ Replica yavaşsa → primary yavaşlar.

Asenkron replikasyon (default):
  Primary → COMMIT → client OK → arka planda replica güncelle.
  ✓ Hızlı write.
  ✗ Primary çökerse: replike edilmemiş son data kaybolabilir (RPO > 0).
  ✗ Replication lag → okuma sayfasında stale data.

Yarı-senkron (MySQL semi-sync):
  En az 1 replica yazdıysa COMMIT.
  ✓ En az 1 kopya garantili.
  ✗ Tüm replica'lardan garanti yok.
  Kullanım: kritik ama tam sync'in yavaşlığını kaldıramayan sistem.

Replication lag monitoring:
  PostgreSQL: SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
  Alarm: > 30sn → okumalar stale → uygulama bunu biliyor mu?
  
Okuma yönlendirme:
  Write: her zaman primary.
  "En yeni oku" (read-your-writes): primary'dan oku.
  "Stale OK" (sosyal feed, katalog): replica'dan oku.
  Spring: @Transactional(readOnly=true) → replica, readOnly=false → primary.
```

### Multi-Primary (Multi-Master)

```
Her node yazabilir → conflict resolution gerekli.

Last-Write-Wins (LWW): timestamp büyük kazanır.
  ✗ Saat senkronizasyonu şart → NTP drift → yanlış kazanan.
  ✗ Aynı saniyede iki write → belirsiz sonuç.

CRDT (Conflict-free Replicated Data Type):
  G-Counter: sadece artış → merge: max al.
  PN-Counter: artış/azalış → P set + N set, merge: union.
  OR-Set: ekle/sil → tombstone ile → merge: union.
  LWW-Register: son write kazanır (timestamp ile).
  ✓ Conflict yok (matematiksel garanti).
  ✗ Sadece belirli veri yapıları.
  Kullanım: shopping cart, real-time collab, presence.

Kullanım: Geo-distributed sistemler, offline-first uygulamalar.
```

---

## 5. High Availability (HA)

### SLA / Availability Hesabı

```
Availability = Uptime / (Uptime + Downtime)

99%     → 3.65 gün/yıl   (blog, basit web)
99.9%   → 8.76 saat/yıl  (e-ticaret, SaaS)
99.99%  → 52.6 dk/yıl    (fintech, kritik)
99.999% → 5.26 dk/yıl    (telekom, ödeme, sağlık)

Seri bileşenler (zincir): tümü çalışmalı.
  A (99.9%) → B (99.9%) → C (99.9%) → toplam = 99.9%³ = 99.7%
  Her bileşen ekledikçe availability düşer!
  Çözüm: kritik path'teki bileşen sayısını azalt.

Paralel bileşenler (redundancy): biri çalışırsa yeterli.
  2 node (99.9%): 1 - (0.001 × 0.001) = 99.9999%
  3 node (99.9%): 1 - (0.001³) = 99.9999999%
  Redundancy → availability katlar.

Composite örnek (gerçek sistem):
  DNS: 99.99% × LB: 99.99% × App: 99.99% × DB: 99.99% = 99.96%
  Her katman 2 node: app = 99.9999% → toplam çok artar.
  Hedefe göre: her katmanda kaç node gerektiğini hesapla.
```

### SPOF Elimination

```
Her katmanda redundancy:

DNS:
  Birden fazla NS kaydı (farklı provider: Cloudflare + Route53).
  GeoDNS: lokasyona göre farklı IP → yakın region'a yönlendir.
  TTL düşük tut (failover için): 60-300sn.
  DNS tek provider → DNS sağlayıcısı down → site erişilemiyor (SPOF!).

Load Balancer:
  Active-Active: iki LB trafiği paylaşır → herhangi biri down → diğeri devam.
  Active-Passive: biri devre dışı, diğeri hazır (VRRP/keepalived, sanal IP).
  Cloud managed: AWS ALB, GCP Load Balancer → zaten HA (provider garantisi).
  Kendi LB'n: 2 HAProxy + keepalived → VRRP ile yüzer IP.

Application:
  N stateless instance (minimum 3 → biri down → 2 kalan yük alır).
  Health check + auto-restart (Kubernetes liveness probe).
  Graceful shutdown: SIGTERM → inflight istekler bitsin → çık.
    preStop: sleep 5 → LB'den çıkma süresi.
    terminationGracePeriodSeconds: 30.
  Kubernetes PodDisruptionBudget: deploy sırasında min 2 pod zorunlu.

Database:
  Primary + 2 Replica minimum (tek replica → replica down = asenkron, veri kaybı riski).
  Automatic failover:
    PostgreSQL: Patroni (etcd ile leader election, otomatik promote).
    MySQL: InnoDB Cluster veya Orchestrator.
    RDS Multi-AZ: synchronous standby, < 30sn failover.
  Connection string: PgBouncer → primary değişince uygulama fark etmez.
  PITR: WAL streaming + hourly snapshot → herhangi ana dön.

Cache:
  Redis Sentinel: 1 primary + 2 sentinel (failover yöneticisi, quorum 2).
  Redis Cluster: 6 node (3 primary, 3 replica) → otomatik failover.
  Cache down toleransı: cache-aside → DB fallback (yavaş ama çalışır).

Message Queue:
  Kafka: replication.factor=3, min.insync.replicas=2.
    Bir broker down → 2 replica → mesaj kaybı yok.
  RabbitMQ: Quorum Queue (Raft consensus, 3 node).

Storage:
  S3: 99.999999999% durability (11 nine) — multi-AZ erasure coding.
  EBS: Multi-AZ snapshot, cross-region replication.
  NFS/EFS: multi-AZ, ölçeklenebilir.
```

### Graceful Degradation

```
Fallback hiyerarşisi (azalan tercih):
  1. Canlı veri (ideal) → recommendation service OK.
  2. Cache'ten stale data → 5dk önce hesaplanan öneri.
  3. DB'den yavaş sorgu → DB recommendation tablosu.
  4. Default değer → "Popüler ürünler" (statik).
  5. Hata mesajı → "Öneriler yüklenemedi" (son çare).

Circuit Breaker + Graceful Degradation:
  RecommendationService → timeout/error → circuit OPEN.
  Fallback: cachedRecommendations.get(userId) → varsa cache.
  Cache yok: popularItemsService.getTopN(10) → statik popüler.
  Kullanıcı: farklı öneri görüyor ama sistem çalışıyor.

Feature Flag:
  ProductService down → flag "recommendations=OFF".
  UI: öneri bölümü gizlenir → boş kalmaz.
  Diğer özellikler çalışır.
  Araç: LaunchDarkly, Unleash, Flipt (open source).

Partial response:
  API: 5 bağımsız microservice'ten veri topla.
  1 servis timeout → diğer 4'ün verisini döndür, başarısız alanı null.
  Kullanıcı: kısmi veri görür → tam hata yerine.
  Header: X-Partial-Response: true → client biliyor.
```

---

## 6. Circuit Breaker Pattern

```
Amaç: başarısız servise tekrar tekrar çağrı yapmayı önle → hızlı fail.

Durumlar:
  CLOSED (normal): istekler geçiyor.
    N ardışık hata veya %X hata oranı → OPEN'a geç.
  OPEN (arıza): istekleri hemen reddet (timeout beklemez).
    Avantaj: caller servis thread'leri serbest kalır → downstream çöküşü cascade etmez.
    Belirli süre (örn 30sn) → HALF-OPEN'a geç.
  HALF-OPEN (deneme): 1 istek geç → başarılıysa CLOSED, başarısızsa OPEN.

Resilience4j (Spring Boot):
```
```java
@Bean
public CircuitBreaker paymentCircuitBreaker(CircuitBreakerRegistry registry) {
    CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50)           // %50 hata → OPEN
        .slowCallRateThreshold(50)          // %50 yavaş çağrı da OPEN tetikler
        .slowCallDurationThreshold(Duration.ofSeconds(3))
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .permittedNumberOfCallsInHalfOpenState(5)
        .slidingWindowSize(10)              // son 10 çağrıya bak
        .build();
    return registry.circuitBreaker("payment", config);
}

// Kullanım
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
public PaymentResponse charge(PaymentRequest req) {
    return paymentServiceClient.charge(req);
}

public PaymentResponse paymentFallback(PaymentRequest req, Exception ex) {
    // Queue'ya at, async retry
    pendingPaymentQueue.offer(req);
    return PaymentResponse.queued("Ödeme işleme alındı, kısa sürede tamamlanacak.");
}
```
```

---

## 7. Bulkhead Pattern (İzolasyon)

```
Amaç: bir servisin yavaşlaması diğer servisleri etkilemesin.
Gemi gövdesi metaforu: bir bölme su aldı → diğerleri dayanıklı.

Thread Pool Bulkhead:
  OrderService: max 20 thread.
  PaymentService: max 10 thread.
  RecommendationService: max 5 thread.
  
  Recommendation yavaşladı → 5 thread dolu → Recommendation kuyrukta bekliyor.
  Order ve Payment: kendi thread'leri → etkilenmez.

Resilience4j Thread Pool Bulkhead:
  @Bulkhead(name = "recommendation", type = Bulkhead.Type.THREADPOOL)
  CoreThreadPoolSize: 5, MaxThreadPoolSize: 10, QueueCapacity: 20.
  20'yi aştı → BulkheadFullException → fast fail.

Semaphore Bulkhead:
  Max 10 eş zamanlı çağrı → 11.'si → hemen REJECT.
  Thread pool gibi değil: aynı thread, sadece sayaç.
  Avantaj: düşük overhead.
  Dezavantaj: thread'i serbest bırakmaz (thread block olunca bekleme artar).

Connection Pool Bulkhead:
  PgBouncer: her uygulama için ayrı pool.
  DB bağlantıları: OrderService → max 20, PaymentService → max 10.
  OrderService bağlantıları tükendi → Payment etkilenmez.

Microservice düzeyinde bulkhead:
  Kubernetes: her servis → kendi namespace, kendi resource limit.
  CPU/Memory limit: bir servis OOM → diğer servisleri etkilemez (evicted).
```

---

## 8. Timeout ve Retry Stratejisi

```
Timeout tipleri:
  Connection timeout: sunucuya TCP bağlantı kurma süresi → 3-5sn.
  Read timeout: bağlantı kuruldu, response bekleme → servis başına 1-30sn.
  Request timeout: tüm isteğin toplam süresi (connection + read) → end-to-end.

Timeout ayarı:
  Çok kısa → gereksiz hata (P99 latency'den yüksek tut).
  Çok uzun → thread'ler meşgul bekleyerek → cascading failure.
  Öneri: P99 latency × 1.5 → timeout.
  Örn: payment P99 = 2sn → timeout = 3sn.

Deadline propagation:
  API Gateway: istek timeout = 10sn.
  Auth Service: 2sn, Order Service: 3sn, Payment: 5sn → toplam = 10sn.
  gRPC: context deadline → downstream'e otomatik propagate.
  REST: X-Request-Deadline header → manuel propagate.
  Kalan süre kalmadıysa → downstream çağrısı yapma → zaten geç.

Retry stratejisi:
  Geçici hata (5xx, timeout): retry et.
  Kalıcı hata (4xx, validation): retry etme.
  Idempotent endpoint: güvenle retry (GET, DELETE, PUT).
  Non-idempotent (POST): idempotency key ile retry.

Exponential Backoff + Jitter:
  1. deneme: hemen.
  2. deneme: 2^1 × base (1sn) = 2sn + jitter.
  3. deneme: 2^2 × base = 4sn + jitter.
  4. deneme: 2^3 × base = 8sn + jitter.
  max retry: 3-5.
  max delay: cap at 30-60sn.
  
  Jitter neden? Tüm istemciler aynı anda retry → thundering herd.
  Full jitter: sleep = random(0, min(cap, base * 2^attempt)).
  
  Resilience4j:
  @Retry(name = "paymentRetry")
  RetryConfig: maxAttempts=3, waitDuration=1s, enableExponentialBackoff=true,
               exponentialBackoffMultiplier=2, retryExceptions={IOException.class, TimeoutException.class},
               ignoreExceptions={ValidationException.class}.
```

---

## 9. Rate Limiting

### Token Bucket

```
Kova: max N token kapasitesi.
Her istek: 1 token tüket.
Kova dolma hızı: R token/sn.
Kova boşaldı → istek reddedildi (429 Too Many Requests).

Avantaj: burst izin verir (kova doluysa anlık çok istek).
Kullanım: API rate limit, burst traffic absorb.

Redis implementasyonu:
  MULTI/EXEC:
    GET tokens:{userId}
    SET tokens:{userId} {newCount} EX {ttl}
  Lua script (atomik):
    local current = redis.call('GET', key)
    if current == false then current = capacity end
    if current > 0 then
      redis.call('SET', key, current - 1, 'EX', ttl)
      return 1  -- istek kabul
    else
      return 0  -- istek reddedildi
    end
```

### Sliding Window Log

```
Her istek: timestamp → sorted set'e ekle.
Window dışındaki: temizle.
Set boyutu > limit → reddet.

ZADD requests:{userId} {now} {requestId}
ZREMRANGEBYSCORE requests:{userId} 0 {now - window}
count = ZCARD requests:{userId}
if count > limit: REJECT

✓ Tam hassasiyet (milisaniye bazında).
✗ Her istek için fazla bellek (her timestamp saklanıyor).
✗ Yüksek QPS'de Redis yükü.
```

### Fixed Window Counter

```
Her dakika / saat: sayaç sıfırla.
INCR counter:{userId}:{minute}
EXPIRE counter:{userId}:{minute} 60

✗ Window sınırında burst: 00:59 → 100 istek, 01:00 → 100 istek → 1sn'de 200.
Kullanım: hassasiyet gerekmiyorsa (marketing kampanya limiti).
```

### Distributed Rate Limiting

```
Sorun: 10 API instance → her biri kendi sayacı → toplam N×limit istek.

Çözüm 1: Redis merkezi sayaç.
  Tüm instance'lar aynı Redis key'i günceller → global limit.
  Dezavantaj: her istek → Redis round trip (+1ms).

Çözüm 2: Token Bucket + Redis Lua.
  Atomik Lua script → race condition yok.
  Pipeline ile batch istekler → round trip azalt.

Çözüm 3: Local + global sync.
  Her instance: local quota (N/instance_count).
  Periyodik sync: global Redis → redistribute.
  Avantaj: Redis'e her istekte gitme.
  Dezavantaj: bir instance quota'sını harcadı → diğerlerinde kalan var.

Rate limit tipleri:
  User bazlı: ratelimit:user:{userId} → kişisel limit.
  IP bazlı: ratelimit:ip:{ip} → DDoS önleme.
  API key bazlı: ratelimit:key:{apiKey} → partner limiti.
  Endpoint bazlı: ratelimit:{endpoint}:{userId} → /search daha kısıtlı.
  Global: ratelimit:global → tüm sisteme gelen yük sınırı.

Response header:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1705316400  (unix timestamp)
  Retry-After: 60  (429 durumunda)
```

---

## 10. Auto-Scaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # CPU %70 → scale out
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # 30sn boyunca yüksekse → scale
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60            # dakikada max 5 pod ekle
    scaleDown:
      stabilizationWindowSeconds: 300  # 5dk boyunca düşükse → scale down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60            # dakikada max %10 azalt
```

### KEDA (Event-Driven Auto-Scaling)

```yaml
# Kafka consumer lag'ına göre scale
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: notification-worker
  minReplicaCount: 1
  maxReplicaCount: 100
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: notification-workers
        topic: notifications
        lagThreshold: "1000"   # 1000 mesaj geride → 1 pod ekle
        offsetResetPolicy: latest

# Prometheus metriğine göre scale
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: active_trips
        threshold: "500"         # 500 aktif yolculuk/pod
        query: sum(active_trips_gauge)
```

### Predictive Scaling

```
Sorun: HPA reaktif → trafik spike geldi → scale out başladı → pod start 2dk → geç!
Çözüm: önceden tahmin et → scale et.

AWS Auto Scaling Predictive:
  ML model: geçmiş trafik pattern → tahmin.
  "Salı 09:00 → hep spike" → 08:45'te scale et.
  
Scheduled Scaling:
  Bilinen event: konser, indirim, Ramazan → manuel schedule.
  Kubernetes: CronJob ile replica count güncelle.
  Örn: Cuma 18:00 → minReplicas=20, Cuma 22:00 → minReplicas=5.

Warm pool (AWS):
  Pre-warmed instance: başlatılmış ama trafik almıyor.
  Spike → warm pool'dan al → anında hazır (2dk yerine 30sn).
  Maliyet: ek instance → tradeoff: hız vs maliyet.

Ölçek kararı hiyerarşisi:
  1. Scheduled (bilinen event) → en doğru.
  2. Predictive (ML) → geçmiş pattern.
  3. Reactive (HPA/KEDA) → anlık metrik.
  Gerçek dünya: 3'ünü birlikte kullan.
```

---

## 11. Connection Pool Yönetimi

```
DB bağlantısı pahalı:
  TCP handshake + TLS + authentication + session init = 50-100ms.
  Her istek yeni bağlantı → 100ms overhead → kabul edilemez.
  Connection pool: önceden aç, yeniden kullan.

HikariCP (Java default):
  maximumPoolSize: 10 (her uygulama instance'ı için).
  minimumIdle: 5 (her zaman açık).
  connectionTimeout: 30,000ms (pool'da bağlantı yoksa bekle).
  idleTimeout: 600,000ms (10dk boş → kapat).
  maxLifetime: 1,800,000ms (30dk → yeni bağlantı aç, eskisi kaç).

Boyut belirleme:
  Threads = (core_count * 2) + effective_spindle_count
  Pratik: DB sunucusu max 200 bağlantı, 10 app instance → 200/10 = 20/instance.
  Çok büyük pool: DB'ye çok bağlantı → context switch → performans düşer!

PgBouncer (PostgreSQL connection pooler):
  Uygulama → PgBouncer → PostgreSQL (az bağlantı).
  Transaction mode: her transaction sonrası bağlantı havuza döner.
  500 uygulama bağlantısı → PgBouncer → 20 PostgreSQL bağlantısı.
  Session mode: bağlantı transaction değil, session boyunca.

Connection pool exhaustion:
  Tüm bağlantılar meşgul → yeni istek bekler → timeout → 503.
  Tespiti: HikariCP metric → pool.pending (bekleyen istek sayısı).
  Çözüm: pool büyüt → DB yüküne dikkat.
  Kök neden: yavaş sorgu → uzun süre bağlantı tutuyor.
  Fix: sorgu optimize et → bağlantılar serbest kalır.

Read/Write ayrımı:
  Write pool: primary → max 10.
  Read pool: replica → max 50 (okuma ağırlıklı).
  @Transactional(readOnly=true) → replica pool kullan.
  Spring: AbstractRoutingDataSource → readOnly flag → replica datasource.
```

---

## 12. CDN Stratejileri

```
CDN neden?
  Statik varlıklar (JS, CSS, resim, video): her kullanıcıya her seferinde origin'den vermek → yavaş + pahalı.
  CDN: edge server → kullanıcıya yakın → hızlı.

Cache-Control header:
  public, max-age=31536000, immutable → 1 yıl cache (hash'li URL: main.abc123.js).
  public, max-age=3600 → 1 saat cache (API response).
  private, no-store → kullanıcıya özel, cache'leme (bank bilgisi).
  
  Immutable hash URL:
    main.abc123.js → içerik değişmez → sonsuza kadar cache.
    Yeni deploy: main.def456.js → browser yeni dosya istiyor.
    CDN evict gerekmez (farklı URL).

Origin Shield (CDN → CDN → Origin):
  Çok sayıda edge node → hepsi cache miss'te origin'e gidiyor.
  Origin shield: edge → regional CDN (shield) → origin.
  Shield: tek yerden origin'e gidiyor → origin yükü azaldı.
  Cache fill: shield bir kez dolursa → diğer edge'ler shield'dan alıyor.

Cache invalidation (purge):
  Deploy: yeni JS → CDN'deki eski JS → kullanıcılar eski görüyor.
  Purge API: Cloudflare, Fastly → belirli URL'yi invalidate.
  Selective purge: /static/main.js purge.
  Wildcard purge: /api/* → dikkatli (tüm API cache'i sil).
  Purge hızı: Cloudflare: saniyeler içinde global propagation.

Dynamic content CDN:
  API response cache: Cache-Control: public, max-age=60.
  Ürün katalogu (az değişen): 5dk cache → %90 request CDN'den.
  Kullanıcıya özel: private → CDN cache'leme.
  Vary header: Vary: Accept-Encoding, Accept-Language → her dil ayrı cache entry.

Multi-CDN:
  Cloudflare + Fastly + Akamai → farklı bölgelerde farklı CDN.
  Failover: Cloudflare down → DNS → Fastly'ye yönlendir.
  Cost optimization: pahalı bölge → ucuz CDN provider seç.
```

---

## 13. Deployment Stratejileri

```
Rolling Deploy:
  Instance'ları tek tek güncelle (1 güncelle → sağlıklı → 1 daha...).
  ✓ Downtime yok, kaynak israfı yok.
  ✗ İki versiyon aynı anda çalışıyor → backward compatible API/DB şart.
  ✗ Rollback: yavaş (geri almak da rolling).

Blue-Green:
  Blue (v1, prod) + Green (v2, hazır) → traffic switch → Green prod.
  ✓ Anlık switch (LB veya DNS).
  ✓ Instant rollback (Blue hâlâ hazır).
  ✗ Çift kaynak (2x instance maliyeti).
  ✗ DB migration: her iki versiyon aynı DB kullanıyorsa → schema uyumlu olmalı.

Canary Deploy:
  %5 trafik v2'ye → metrik izle → sorunsa geri al → kademeli %100'e çıkar.
  ✓ Risk minimized (sadece %5 kullanıcı etkileniyor).
  ✓ Gerçek trafik ile test.
  ✗ Yavaş rollout.
  ✗ İki versiyon aynı anda uzun süre çalışıyor.
  Araç: Argo Rollouts, Flagger (otomatik metrik analiz + rollback).

Shadow Deploy (Mirroring):
  Tüm trafik v1'e gidiyor, aynı zamanda v2'ye kopyalanıyor (response kullanılmıyor).
  v2: gerçek yük altında test, kullanıcı etkilenmiyor.
  Kullanım: kritik servis (ödeme) değişikliği öncesi.
  Dezavantaj: çift yük, idempotent olmayan operasyonlar (gerçek ödeme yollama).

Feature Flag:
  Kod deploy edildi ama feature kapalı → aç/kapa runtime'da.
  ✓ Deploy ve release bağımsız.
  ✓ A/B test, kill switch.
  ✗ Flag yönetimi karmaşıklığı (LaunchDarkly, Unleash, Flipt).
  ✗ Eski flag'lerin temizlenmesi teknik borç.
```

---

## 14. Zero-Downtime Database Migration

```
Sorun: 500M satırlık tabloya yeni kolon ekle → downtime olmadan.

Naif: ALTER TABLE ADD COLUMN → tablo lock → downtime!

Adım adım yaklaşım (3 deploy):

Deploy 1 — Ekleme (backward compatible):
  DB: yeni kolon nullable ekle (NOT NULL olmadan).
  Uygulama: yeni kolonu yaz + eski kolonu da yaz (dual-write).
  ALTER TABLE users ADD COLUMN phone VARCHAR(20);  -- nullable, hızlı
  Eski kod: phone kolonu yok → sorun yok.

Deploy 2 — Backfill:
  Batch: eski kayıtları güncelle (phone doldurmayan satırlar).
  UPDATE users SET phone = '...' WHERE phone IS NULL LIMIT 10000;
  -- Batch + sleep → table lock yok, yavaş ama güvenli.
  pg_repack: live table reorg (lock-free).
  Online schema change: gh-ost (MySQL), pg_repack (PostgreSQL).

Deploy 3 — Kullan + temizle:
  Uygulama: sadece yeni kolonu oku.
  DB: NOT NULL constraint ekle (backfill tamam → güvenli).
  ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
  Eski dual-write kodu: kaldır.

Expand/Contract pattern:
  Expand: yeni kolon/tablo ekle (backward compatible).
  Migrate: data taşı.
  Contract: eski kolonu/tabloyu kaldır.
  Her adım: ayrı deploy → rollback her an mümkün.

Index eklemek (büyük tablo):
  CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
  CONCURRENTLY: tablo lock yok, arka planda → yavaş ama production güvenli.
  Dikkat: CONCURRENTLY transaction içinde çalışmaz.
```

---

## 15. Disaster Recovery (DR)

### RTO / RPO

```
RTO (Recovery Time Objective): Outage'dan ne kadar sürede kurtulabiliriz?
  "Sistem 2 saat içinde ayağa kalkmalı" → RTO = 2 saat.
  
RPO (Recovery Point Objective): Ne kadar veri kaybını kabul edebiliriz?
  "Son 5 dakikadaki veri kaybı kabul edilebilir" → RPO = 5 dk.

İlişki:
  RPO düşük → sık backup/sync → pahalı.
  RTO düşük → hazır standby → pahalı.
  İkisi de sıfır → çok pahalı (active-active multi-region).

Maliyet - Kurtarma İlişkisi:
  RTO yüksek (günler) → ucuz (cold backup)
  RTO düşük (dakikalar) → pahalı (hot standby)
```

### DR Stratejileri

```
Backup & Restore (en ucuz):
  Günlük snapshot → S3 → Outage → restore → çalıştır.
  RTO: saatler/günler (restore süresi).
  RPO: günlük (son backup'tan bu yana kayıp).
  Maliyet: düşük (sadece storage).
  Kullanım: non-kritik, uzun RTO tolere.

Pilot Light:
  DR bölgesinde: sadece kritik infrastructure çalışıyor (DB sync).
  Uygulama: kapalı (scale-out hızlı yapılabilecek image).
  Outage: uygulama instance'larını başlat → DNS switch.
  RTO: 10-30 dk (instance başlatma + DNS).
  RPO: dakikalar (DB async replication lag).
  Maliyet: orta (DB + temel infra).

Warm Standby:
  DR bölgesinde: tam uygulama ama küçük kapasitede (1/4 prod kapasitesi).
  Sürekli çalışıyor → trafiği anında alabilir.
  Outage: DNS switch → DR bölge scale-out → tam kapasite.
  RTO: dakikalar (DNS TTL + scale-out).
  RPO: saniyeler (senkron veya yarı-senkron DB replikasyon).
  Maliyet: yüksek (%25 prod kapasitesi sürekli çalışıyor).

Active-Active Multi-Region (en pahalı):
  Her bölge aktif trafik alıyor.
  GeoDNS: kullanıcı → en yakın region.
  Outage: DNS diğer region'a → trafik otomatik geçiş.
  RTO: saniyeler (DNS).
  RPO: sıfır (senkron) veya saniyeler (asenkron).
  Maliyet: 2x infra (her bölge full capacity).
  Zorluk: cross-region yazma çakışması → conflict resolution.

DR Drill (tatbikat):
  6 ayda bir: gerçek failover testi.
  "Production'ı kasıtlı kapat → DR'ye geç → ne kadar sürdü?"
  Belgelenmemiş DR → tatbikat yapılmamış → gerçek outage'da çalışmıyor.
  Chaos Engineering: DR drill'in planlı versiyonu.
```

---

## 16. Chaos Engineering

```
Amaç: sistemin gerçek arızalara dayanıklılığını proaktif test et.
"Güvenli ortamda arıza üret → zayıflıkları bul → düzelt → sistem daha sağlam."

Netflix Chaos Monkey: random olarak prod service'leri sonlandırır.
Principle: "Eğer bir servisi istediğimiz zaman kapatabilirsek, sistem gerçek arızalara hazır."

Experiment tipleri:
  Pod kill: rastgele Kubernetes pod'u sil.
  CPU spike: bir instance'da %100 CPU kullan.
  Network partition: iki service arası network'ü kes.
  Latency injection: tüm yanıtlara 500ms ekle.
  Memory pressure: belirli bir pod'a memory leak simüle et.
  DNS failure: bir servisin DNS çözümlemesini boz.
  Disk full: bir node'un diskini doldur.

Litmus Chaos (Kubernetes için):
  ChaosExperiment CR: ne yapılacak.
  ChaosEngine CR: hangi target, hangi süre.

Minimal uygulamaya başla:
  Test ortamı → staging → sonra production.
  Production'da: düşük trafik saatinde (gece 02:00).
  Rollback planı: eğer gerçek outage tetiklenirse ne yapacaksın?
  Blast radius: küçük başla → etkiyi gözlemle → büyüt.

Game Day:
  Planlı kaos tatbikatı: tüm ekip hazır.
  Senaryo: "X region düşüyor → nasıl tepki veririz?"
  Öğrenim: runbook güncelle, otomasyon iyileştir.

Steady state hypothesis:
  Önce: "Normal durumda sistemin P99 latency < 200ms, error rate < %1."
  Chaos uygula.
  Sonra: hâlâ P99 < 200ms ve error < %1? → dayanıklı.
  Değilse → zayıflık bulundu → düzelt.
```

---

## Olası Sorunlar ve Çözümleri

### 1. Thundering Herd — Cache Miss Sonrası DB Çöktü

```
Sorun:
  Popüler ürün sayfası: Redis cache → her istek cache hit → hızlı.
  Deploys sırasında: cache flush edildi.
  10,000 kullanıcı: aynı anda aynı sayfa → hepsi cache miss.
  DB: 10,000 eş zamanlı sorgu → CPU %100 → timeout → site çöktü.

Çözüm:
  a) Cache stampede lock (mutex):
     Cache miss → Redis SETNX "lock:product:123" 1 EX 10 → kazandım → DB sor → cache'e yaz → lock kaldır.
     Lock alamayanlar: kısa süre bekle → tekrar dene → cache doldu → direkt oku.
     Dezavantaj: bekleyen istemciler (lock süresi boyunca).

  b) Early expiration (stale-while-revalidate):
     Cache: gerçek TTL = 10dk, yazılan TTL = 8dk.
     8. dakikada: cache hâlâ veriyor ama "expire yaklaşıyor" flag.
     Arka planda: tek istek → DB → cache güncelle.
     Diğer istekler: stale ama çalışıyor (2dk stale tolerable).

  c) Request coalescing:
     Singleflight pattern (Go): aynı key için bir tane DB sorgusu → tüm bekleyenler aynı sonucu alıyor.
     Java: Caffeine cache.get(key, k -> db.query(k)) → tek sorgu, çok waiter.

  d) Gradual cache warm-up:
     Deploy: cache'i flush'lamadan önce warm-up yap.
     CDN pre-warming: deployment sonrası → kritik URL listesi → bot olarak hit → cache dolsun.
     Redis: deploy öncesi yeni instance'ı warm-up et, sonra geçiş yap.
```

---

### 2. Cascading Failure — Tek Yavaş Servis Tüm Sistemi Durdurdu

```
Sorun:
  RecommendationService: ağır ML inference → P99 5sn.
  API Gateway: her kullanıcı isteğinde recommendation çağrısı.
  Thread pool: 200 thread → hepsi recommendation bekliyorsa meşgul.
  Yeni istek: thread bulamadı → queue → timeout → 503.
  Recommendation sorunu → tüm API 503.

Çözüm:
  a) Timeout:
     RecommendationService: max 500ms timeout → geçmezse fallback.
     Düşük timeout: thread serbest kalır → cascade yok.

  b) Circuit Breaker:
     Recommendation: %50 hata → circuit OPEN.
     OPEN: recommendation çağrısı yapma → anında fallback.
     Thread: hiç meşgul olmuyor → API çalışıyor.

  c) Bulkhead:
     Recommendation: ayrı thread pool (max 10 thread).
     Recommendation yavaşlayınca: sadece 10 thread meşgul.
     Ana API pool: 190 thread → etkilenmiyor.

  d) Async non-blocking:
     WebFlux / reactive: thread park etme → event loop.
     1,000 concurrent recommendation isteği: 10 thread (async I/O).
     Dezavantaj: reactive programming complexity.

  e) Shedding:
     Sistem yük altında → non-critical istekleri reddet.
     Recommendation: non-critical → load > %80 → skip → fallback.
     Payment: critical → her zaman işle.
```

---

### 3. Split Brain — İki Primary Aynı Anda Yazma Kabul Etti

```
Sorun:
  Network partition: Datacenter A ↔ Datacenter B bağlantısı koptu.
  Her DC kendi primary'sini seçti → iki primary.
  Her ikisi de write kabul ediyor → çelişen veri.
  Partition çözüldü → hangi primary'nin verisi doğru?

Çözüm:
  a) Quorum (çoğunluk) gerektir:
     3 node (A, B, C) → partition: A | B+C.
     Write: quorum (2 node) onayı gerekli.
     A: tek başına → quorum yok → write kabul etme.
     B+C: iki node → quorum var → sadece bu taraf write kabul eder.
     Split brain: mümkün değil (tek taraf her zaman quorum altında).

  b) Fencing token:
     Leader: her epoch'ta fencing token alır.
     Write: token ile birlikte → storage: eski token → reddet.
     Eski leader: eski token → yeni primary varken write → reddedilir.

  c) STONITH (Shoot The Other Node In The Head):
     Network partition → belirsiz: karşı taraf gerçekten down mu?
     STONITH: power cut API → "karşı taraf" fiziksel kapat.
     Kesin: artık iki primary yok.
     Kullanım: yüksek güvenilirlik gerektiren ortam (HA Proxy gibi).

  d) Etcd/ZooKeeper ile distributed lock:
     Primary seçimi: etcd → distributed consensus.
     Primary kesinlikle bir tane → split brain imkansız.
     PostgreSQL Patroni: etcd + STONITH → endüstri standardı.
```

---

### 4. Hot Partition — Tüm Yük Tek Shard'a Gitti

```
Sorun:
  E-ticaret: shard key = category.
  Büyük kampanya: "Elektronik" kategorisi → tüm satışlar → tek shard.
  Elektronik shard: CPU %100 → query timeout → ürün sayfaları 503.
  Diğer shard'lar: boş.

Çözüm:
  a) Shard key değiştir:
     category → user_id veya product_id (yüksek kardinalite).
     Elektronik shard yok → load uniform dağıtıldı.

  b) Key compound etme:
     shard_key = hash(product_id) + suffix (0-9).
     "Elektronik" → 10 farklı shard key → 10 shard'a dağıt.
     Okuma: 10 shard'ı sorgula → merge.

  c) Read replica (hot shard için):
     Hot shard: okumalar için 3 replica → yük 4'e bölün.
     Write: hâlâ primary, ama read yükü %75 azaldı.

  d) Cache önce:
     Popüler ürünler → Redis cache → DB'ye gidilmeden önce.
     Cache hit: shard'a hiç gitme.
     Cache miss: sadece o an shard'a → çok az istek.

  e) CQRS + materialized view:
     Kategori bazlı sorgular → ayrı okuma modeli (DynamoDB veya Elasticsearch).
     Write: shard'a → Event → okuma modeli güncelle.
     Okuma: shard değil, okuma modeli → hot shard bypass.
```

---

### 5. Connection Pool Exhaustion — DB Bağlantısı Tükendi

```
Sorun:
  Normal: HikariCP pool 10 bağlantı, her sorgu 50ms.
  Yavaş sorgu: bir endpoint → 10sn süren sorgu.
  10 eş zamanlı bu endpoint isteği: 10 bağlantı × 10sn = 10sn meşgul.
  Yeni istek: pool boş → 30sn bekle → connectionTimeout → 503.
  Tüm API etkilendi (payment, user, order — hepsi DB kullanıyor).

Neden:
  Yavaş sorgu → bağlantıları uzun süre meşgul → pool tükeniyor.

Çözüm:
  a) Yavaş sorguyu bul ve düzelt:
     EXPLAIN ANALYZE → missing index → CREATE INDEX.
     PostgreSQL: pg_stat_statements → en yavaş sorgular.
     Uzun süren sorgu kısaldı → bağlantılar serbest kalır.

  b) Query timeout:
     Statement timeout: SET statement_timeout = '5s'.
     5sn'de bitmezse → PostgreSQL: sorguyu iptal et → bağlantı serbest.
     Calling code: SQLException → handle → 503 (pool tüketme yerine).

  c) Bulkhead per endpoint:
     Yavaş endpoint: ayrı DB datasource → max 3 bağlantı.
     Tüm havuzu tüketemez → diğer endpoint'ler etkilenmez.

  d) Pool monitoring + alarm:
     HikariCP JMX: hikaricp.connections.pending → bekleyen istek sayısı.
     Alarm: pending > 5 → "pool tükenme riski."
     Grafana: pool kullanımı trend → önceden gör.

  e) PgBouncer:
     1,000 uygulama bağlantısı → PgBouncer → 20 PostgreSQL bağlantısı.
     Transaction mode: her transaction sonrası bağlantı havuza döner.
     Etkin bağlantı sayısı düşer → pool exhaustion riski azalır.
```

---

### 6. OOM Kill — Bellek Sızıntısı Pod'u Çökertiyor

```
Sorun:
  API pod: başlangıçta 512MB → her saat %2 büyüme.
  24 saat sonra: 1.2GB → Kubernetes memory limit 1GB aşıldı → OOM Kill.
  Pod restart → tüm in-flight istekler 503 → kullanıcı hatası.
  Her gün yeniden oluşuyor → gece 02:00'de.

Çözüm:
  a) Bellek sızıntısını bul:
     Java: heap dump analizi → Eclipse MAT, JVisualVM.
     Büyüyen nesne: örn. static list'e sürekli ekleme, kapatılmayan stream.
     Heapster / Pyroscope: continuous profiling → kod satırı bazında.

  b) Monitoring + alarm:
     Prometheus: container_memory_usage_bytes → trend grafik.
     Alarm: 80% limit → "Bellek artışı var, araştır."
     OOM yaşanmadan önce fark et.

  c) Kubernetes resource limit (güvenli):
     Limit: 1GB. Request: 512MB.
     Limit aşılırsa OOM → restart (zaten oluyor).
     Çözüm: limit yükselt → geçici band-aid, sızıntıyı bulmak gerekiyor.

  d) Scheduled restart (geçici çözüm):
     CronJob: haftada 1 kez pod restart (memory akümülasyonunu sıfırla).
     Graceful restart: zero-downtime (rolling).
     Not: sızıntıyı çözmez, semptom yönetimi.

  e) Off-heap bellek:
     Java: DirectByteBuffer, Netty — heap dışı bellek → GC tarafından yönetilmez.
     Off-heap sızıntısı: heap dump'ta görünmez → NMT (Native Memory Tracking) ile bul.
```

---

### 7. Zero-Downtime Migration'da Backward Incompatible Schema

```
Sorun:
  users tablosunda: name VARCHAR(100) → first_name + last_name olarak böl.
  Rolling deploy: yeni pod + eski pod aynı anda çalışıyor.
  Yeni pod: first_name kolonu yazıyor.
  Eski pod: name kolonu okumak istiyor → kolon silinmişse → hata.

Çözüm:
  Expand-Migrate-Contract:

  Adım 1 — Expand (Deploy 1):
    ALTER TABLE users ADD COLUMN first_name VARCHAR(50);
    ALTER TABLE users ADD COLUMN last_name VARCHAR(50);
    -- name kolonu hâlâ var
    Uygulama: hem name hem first_name+last_name yaz (dual-write).
    Eski pod: name okur → OK.
    Yeni pod: first_name+last_name yazar → OK.

  Adım 2 — Backfill (batch job):
    UPDATE users SET
      first_name = split_part(name, ' ', 1),
      last_name = split_part(name, ' ', 2)
    WHERE first_name IS NULL
    LIMIT 10000;
    -- Tekrarlı çalıştır → tüm kayıtlar dolu.

  Adım 3 — Contract (Deploy 2):
    Uygulama: sadece first_name+last_name oku/yaz.
    Eski pod kalmadı (deploy tamam).
    name kolonu: kullanılmıyor ama hâlâ var → kaldırmak güvenli.

  Adım 4 — Cleanup (Deploy 3 veya migration):
    ALTER TABLE users DROP COLUMN name;
    -- Artık güvenli: hiçbir kod okumuyou.

  Bu pattern: her ölçekte çalışır, downtime yok, rollback her an mümkün.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Scaling | Vertical (scale-up) | Hybrid (vertical DB + horizontal API) | Saf vertical: limit var; Saf horizontal: DB karmaşıklığı |
| Shard stratejisi | Range | Hash + consistent hashing | Range: hot spot; Hash: uniform dağılım |
| LB algoritması | Round Robin | Least Connections (WebSocket) | Round: equal weight; LC: uzun bağlantılarda denge |
| DB replikasyon | Asenkron | Yarı-senkron (kritik data) | Async: hızlı, veri kaybı; Semi-sync: orta yol |
| Circuit Breaker | Yok | Resilience4j (sliding window) | Yok: cascading failure; CB: hızlı fail, izolasyon |
| Rate limiting | Fixed window | Sliding window + Token bucket | Fixed: boundary burst; Sliding: hassas |
| Cache invalidation | TTL only | TTL + early revalidation | Sadece TTL: thundering herd; Early: smooth |
| DR strateji | Backup & restore | Warm standby | B&R: RTO günler; Warm: dakikalar |
| Deployment | Rolling | Canary (kritik servis) | Rolling: basit; Canary: risk minimal, gerçek test |
| Connection pool | Tek pool | Ayrı pool per servis (bulkhead) | Tek: bir servis tükenir → hepsi etkilenir |
| Schema migration | Big bang | Expand-Migrate-Contract | Big bang: downtime; EMC: zero-downtime, rollback |
| Auto-scaling | CPU bazlı (HPA) | Event-driven (KEDA) + Scheduled | CPU: reaktif geç; KEDA+Schedule: proaktif |
