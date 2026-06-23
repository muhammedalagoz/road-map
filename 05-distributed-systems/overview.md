# 05 — Dağıtık Sistemler

## 1. CAP Teoremi

Dağıtık bir sistemde aynı anda üçünü birden sağlayamazsın:

- **C**onsistency — Her okuma en güncel veriyi döner
- **A**vailability — Her istek yanıt alır (hata olmayabilir)
- **P**artition Tolerance — Ağ bölünmesi olsa bile sistem çalışır

**Ağ bölünmesi (partition) gerçek dünyada kaçınılmazdır.** Bu yüzden seçim CP vs AP'dir:

```
CP sistemler (tutarlılık > erişilebilirlik):
- HBase, Zookeeper, Etcd
- Ağ bölünmesinde bazı node'lar yanıt vermez
- Örnek: Bank transfer — yanlış veri vermektense hata ver

AP sistemler (erişilebilirlik > tutarlılık):
- Cassandra, DynamoDB, CouchDB
- Ağ bölünmesinde eski veri dönebilir
- Örnek: Shopping cart — biraz stale veri kabul edilebilir

Not: CAP binary değil, bir spectrum. "Tunable consistency" (Cassandra) ikisi arasında seçim yapmanı sağlar.
```

---

## 2. Consistency Patterns

### Strong Consistency
Okuma her zaman en son yazılan veriyi gösterir. Distributed lock veya single leader gerekir. Yavaştır.

### Eventual Consistency
Sistem sonunda tutarlı olur, ama anlık farklılıklar olabilir.

```
Write → Node A (güncellendi)
        Node B (henüz güncellenmedi) ← okuma buradan → eski veri
        Node C (henüz güncellenmedi)
... birkaç ms sonra ...
        Node B (replikasyon tamamlandı)
        Node C (replikasyon tamamlandı)
```

### Causal Consistency
Nedensel ilişki korunur: A→B sırasıyla oluştuysa, A'yı gören B'yi de görür.

```
User posts a comment (A)
User replies to comment (B) → A'ya bağlı

Causal consistency: B'yi gören A'yı da görür
Eventual consistency: B görünür A görünmeyebilir (tutarsız UI)
```

---

## 3. Distributed Transactions

Tek veritabanında transaction kolay. Dağıtık sistemde nasıl?

### Two-Phase Commit (2PC)

```
Coordinator → Prepare? → [Service A, Service B, Service C]
                              ↓
Coordinator ← OK / Abort ← [Service A, Service B, Service C]
                              ↓
Coordinator → Commit / Rollback → [Service A, Service B, Service C]
```

**Sorunlar:**
- Coordinator çökerse → tüm servisler askıda kalır (blocking protocol)
- Tüm servisler cevap verene kadar lock tutulur → düşük throughput
- Microservices'te kullanımı zor

### Saga Pattern

Her işlem local transaction'dır; başarısız olursa compensating transaction çalışır.

**Choreography (Event-driven):**
```
OrderService → order.created event
    ↓
PaymentService → payment.completed event
    ↓
InventoryService → inventory.reserved event
    ↓
ShippingService → shipment.created event

Hata: InventoryService "inventory.failed" event → 
  PaymentService → payment.refunded
  OrderService → order.cancelled
```

**Orchestration (Central coordinator):**
```
SagaOrchestrator:
1. → PaymentService.charge()    ✓
2. → InventoryService.reserve()  ✗ HATA
3. → PaymentService.refund()    (compensating)
4. → OrderService.cancel()      (compensating)
```

| | Choreography | Orchestration |
|-|-------------|---------------|
| **Coupling** | Düşük | Yüksek (orchestrator bağımlı) |
| **Complexity** | Dağınık logic | Merkezi, anlaşılır |
| **Debugging** | Zor (event tracing) | Kolay (tek yer) |
| **Kullanım** | Basit akışlar | Karmaşık iş mantığı |

---

## 4. Service Discovery

Servisler dinamik IP/port'larla ayağa kalkar. Nasıl bulunur?

### Client-Side Discovery (Eureka)
```
Service Register:  OrderService → Eureka → {ip:port, health}
Service Discover: PaymentService → Eureka → OrderService adresi
Load Balance: PaymentService kendi LB algoritmasını uygular (Ribbon)

Avantaj: Basit altyapı
Dezavantaj: Her client LB bilmeli (dil/framework bağımlı)
```

### Server-Side Discovery (Consul + Load Balancer)
```
OrderService → Consul (register)
PaymentService → Load Balancer → Consul → OrderService
                      ↑
               LB burada routing yapar

Avantaj: Client basit, dil bağımsız
Dezavantaj: Ekstra hop, LB single point of failure
```

---

## 5. Load Balancing Algoritmaları

**Round Robin:**
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (başa dön)
Sorun: Server kapasiteleri farklıysa adil değil
```

**Weighted Round Robin:**
```
Server A (weight 3): 3 istek
Server B (weight 1): 1 istek
Güçlü sunucuya daha fazla yük
```

**Least Connections:**
```
Mevcut aktif bağlantı sayısı en az olan sunucuya yönlendir
Uzun süren requestler için ideal (WebSocket, file upload)
```

**Consistent Hashing:**
```
Hash ring: 0 ─────────── 360° (veya 2^32)
Server A: 60°
Server B: 180°
Server C: 300°

key = "user:123" → hash → 150° → saat yönünde ilk server → Server B

Avantaj: Server ekle/çıkar → sadece komşu key'ler etkilenir (cache miss minimized)
Kullanım: Distributed cache (Redis Cluster, Memcached), stateful LB
```

**IP Hash:**
```
hash(client_ip) % server_count → aynı client hep aynı sunucuya
Kullanım: Session affinity (sticky sessions)
Sorun: NAT arkasındaki clientlar hep aynı sunucuya gider
```

---

## 6. Circuit Breaker (Resilience4j)

Başarısız servise sürekli çağrı yapmayı engelle.

```
State Machine:
CLOSED → (failure threshold aşılır) → OPEN
OPEN   → (wait duration geçer) → HALF_OPEN
HALF_OPEN → (test çağrı başarılı) → CLOSED
HALF_OPEN → (test çağrı başarısız) → OPEN
```

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // %50 hata → OPEN
    .waitDurationInOpenState(Duration.ofSeconds(30)) // 30s bekle
    .slidingWindowSize(10)              // son 10 çağrıya bak
    .permittedNumberOfCallsInHalfOpenState(3) // HALF_OPEN'da 3 test
    .build();

CircuitBreaker cb = CircuitBreaker.of("payment-service", config);

Supplier<String> supplier = CircuitBreaker.decorateSupplier(cb, 
    () -> paymentService.charge(amount));

Try.ofSupplier(supplier)
   .recover(CallNotPermittedException.class, ex -> "fallback response");
```

**Fallback stratejileri:**
- Cache'ten eski veri dön
- Default değer dön
- Kuyrukta beklet, sonra işle
- Kullanıcıya "şu an kullanılamıyor" mesajı göster

---

## 7. Rate Limiting Algoritmaları

### Token Bucket (yaygın)
```
Bucket kapasitesi: 100 token
Doldurma hızı: 10 token/saniye

Request gelir → token var mı?
  Evet → token al, işle
  Hayır → 429 Too Many Requests

Özellik: Burst'a izin verir (bucket dolu ise 100 istek anlık)
Kullanım: API Gateway rate limiting (AWS API Gateway, Nginx)
```

### Leaky Bucket
```
Sabit hızda işle, fazlası queue'da bekler
Queue dolarsa → drop

Özellik: Çıkış hızı sabit, no burst
Kullanım: Network traffic shaping
```

### Sliding Window Counter
```
Log tabanlı:
Son 60 saniyede kaç istek geldi? → O(1) değil, O(request) memory
Redis ZSET ile: timestamp score, member=request_id

Fixed window sorunu:
0:59 → 100 istek   |
1:00 → 100 istek   | → 2 saniyede 200 istek! (eşik 100 ama aşıldı)

Sliding window bunu çözer: herhangi 60 saniye penceresinde max 100
```

### Distributed Rate Limiting

```
Problem: 3 uygulama instance, her biri ayrı sayar → 3x limit
Çözüm: Redis atomic operations

Redis INCR + EXPIRE:
key = "rate:user:123:minute:2024010112" (dakika bazlı)
count = INCR key
IF count == 1: EXPIRE key 60
IF count > limit: reject
```

---

## 8. Idempotency

Aynı işlemi birden fazla yapmak sonucu değiştirmemeli.

```
HTTP metodları:
GET, HEAD, DELETE → idempotent (doğal)
PUT → idempotent (aynı kaynak aynı state)
POST → idempotent DEĞİL (her çağrı yeni kaynak oluşturur)

Ödeme örneği:
POST /payments → iki kez gönderilirse → iki ödeme?

Çözüm: Idempotency key
Client: POST /payments
        Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Server:
1. Bu key daha önce görüldü mü? (Redis/DB kontrolü)
2. Evet → önceki sonucu döndür (işlemi tekrar yapma)
3. Hayır → işle, key+sonucu kaydet, yanıt dön
```

---

## 9. Distributed Caching Strategies

**Cache-Aside (Lazy Loading):**
```
Read:
1. Cache'e bak
2. Miss → DB'den oku → Cache'e yaz → döndür
3. Hit → döndür

Write:
DB'ye yaz → cache'i invalidate et (sil)

Sorun: Cache miss burst (thundering herd) → mutex/lock ile çöz
```

**Write-Through:**
```
Write:
1. Cache'e yaz
2. DB'ye yaz (sync)

Read:
Cache'e bak → her zaman güncel

Sorun: Yazma yavaş, cache'te hiç okunmayacak data var (waste)
```

**Write-Behind (Write-Back):**
```
Write:
1. Cache'e yaz (hızlı)
2. Async olarak DB'ye yaz

Risk: Cache çökerse veri kaybı
Kullanım: Çok yoğun write, eventual consistency kabul edilebiliyorsa
```

**Read-Through:**
```
Cache read miss → Cache kendisi DB'den çeker
Uygulama sadece cache ile konuşur
Kullanım: Ehcache, NCache gibi framework'ler
```
