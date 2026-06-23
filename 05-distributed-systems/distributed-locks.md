# 05b — Distributed Locks

## Ne?

**Distributed Lock:** Birden fazla node/process/thread'in aynı anda paylaşılan bir kaynağa erişmesini engelleyen koordinasyon mekanizması. Tek JVM içindeki `synchronized` veya `ReentrantLock`'un dağıtık karşılığı.

```
Tek JVM:
  Thread A → Lock → kritik bölge → Unlock
  Thread B → Lock bekler → A bitince alır

Distributed (birden fazla uygulama sunucusu):
  Server-1 → Redis/ZooKeeper Lock → kritik bölge
  Server-2 → aynı lock'u almaya çalışır → bekler
```

---

## Neden?

**Çözdüğü problem:** Birden fazla uygulama instance'ının aynı işi çift yapması, çakışan yazmaları ve race condition'ları önlemek.

```
Senaryo: Scheduled job (cron) — 3 uygulama sunucusu
  Her gece 02:00'da fatura oluştur
  Uygulama-1: çalışıyor → fatura 1234 oluştur
  Uygulama-2: çalışıyor → fatura 1234 oluştur (TEKRAR!)
  Uygulama-3: çalışıyor → fatura 1234 oluştur (ÜÇÜNCÜ KEZ!)

Çözüm: Distributed lock
  Uygulama-1: lock'u aldı → fatura oluştur
  Uygulama-2: lock'u alamadı → atla
  Uygulama-3: lock'u alamadı → atla

Diğer kullanım alanları:
  - Inventory azaltma (overselling önleme)
  - Idempotent API işlemleri
  - Leader election (sadece bir node koordinatör olsun)
  - Sequence number üretimi
```

---

## Nasıl?

### Redis ile Distributed Lock (Redlock)

#### Temel Redis Lock (Basit)

```java
// Tek Redis node — production için yeterli olmayabilir
@Service
class SimpleRedisLock {

    private final StringRedisTemplate redis;

    boolean tryLock(String lockKey, String requestId, Duration ttl) {
        // SET key value NX PX ttl — atomik
        // NX: sadece key yoksa set et
        // PX: millisecond TTL (lock süresi)
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(lockKey, requestId, ttl);
        return Boolean.TRUE.equals(acquired);
    }

    void unlock(String lockKey, String requestId) {
        // ÖNEMLI: sadece kendi koyduğun lock'u sil (requestId kontrolü)
        // Lua script — atomik check-and-delete
        String luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(lockKey),
            requestId
        );
    }
}

// Kullanım
@Service
class InvoiceService {

    @Autowired SimpleRedisLock lock;

    void generateInvoice(String invoiceId) {
        String lockKey = "lock:invoice:" + invoiceId;
        String requestId = UUID.randomUUID().toString();

        if (!lock.tryLock(lockKey, requestId, Duration.ofSeconds(30))) {
            log.info("Lock alınamadı, başka instance işliyor");
            return;
        }
        try {
            doGenerateInvoice(invoiceId);
        } finally {
            lock.unlock(lockKey, requestId);
        }
    }
}
```

**Neden `requestId`?**
```
Sorun: Lock TTL doldu → başka process aldı → eski process'in unlock'u yanlış lock'u siler

Timeline:
  Process A → lock alır (TTL: 30s)
  Process A → yavaş çalışır (GC pause, 35 saniye geçti)
  TTL doldu → Redis lock'u sildi
  Process B → lock alır
  Process A → "unlock" der → Process B'nin lock'unu sildi!

Çözüm: Unlock ederken "bu benim lock'um mu?" kontrol et (requestId ile)
```

---

#### Redlock Algoritması (Yüksek Güvenilirlik)

Tek Redis node çökerse lock kaybolur. Redlock 5 bağımsız Redis node kullanır.

```
5 Redis node (bağımsız, replica yok — her biri standalone)
Quorum = 3 (5/2 + 1)

Algoritma:
1. Mevcut zamanı kaydet: t1
2. 5 Redis node'a sırayla tryLock(key, requestId, TTL)
3. 3+ node'dan "OK" aldıysa VE süre < TTL/2 ise → lock başarılı
4. Kalan geçerlilik süresi: TTL - (şimdiki_zaman - t1)

Eğer quorum sağlanamadıysa:
  → Aldığın lock'ları hepsinden sil → retry (backoff ile)

Release:
  → 5 node'un hepsine unlock (hangileri başarılı olmuş bilmiyoruz)
```

```java
// Redisson kütüphanesi (Redlock implementasyonu):
@Bean
RedissonClient redissonClient() {
    Config config = new Config();
    // 5 bağımsız Redis node
    config.useClusterServers()  // veya sentinel
        .addNodeAddress(
            "redis://node1:6379",
            "redis://node2:6379",
            "redis://node3:6379",
            "redis://node4:6379",
            "redis://node5:6379"
        );
    return Redisson.create(config);
}

// Kullanım — Redisson RLock
@Service
class DistributedJobService {

    @Autowired RedissonClient redisson;

    void runExclusiveJob(String jobId) {
        RLock lock = redisson.getLock("job:" + jobId);
        boolean locked = false;

        try {
            // tryLock(waitTime, leaseTime, unit)
            // waitTime=0 → bekleme, alamadıysa false dön
            locked = lock.tryLock(0, 30, TimeUnit.SECONDS);

            if (!locked) {
                log.info("Job zaten çalışıyor");
                return;
            }
            doJob(jobId);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (locked && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

---

### ZooKeeper ile Distributed Lock

ZooKeeper'ın ephemeral + sequential node'ları lock için idealdir.

```
ZooKeeper node yapısı:
  /locks/invoice-generator/
    lock-0000000001  (Process A — ephemeral sequential)
    lock-0000000002  (Process B — ephemeral sequential)
    lock-0000000003  (Process C — ephemeral sequential)

Algoritma:
  1. Process A → /locks/invoice-generator/lock- (ephemeral sequential node yarat)
     → lock-0000000001 oluştu
  2. Tüm children listele, sıralamaya bak
  3. Benim node en küçük mü?
     → Evet → lock bende! (Process A)
     → Hayır → bir önceki node'u watch et (Process B, lock-0000000001'i izler)

Lock serbest bırakma:
  Process A → işini bitirdi → node'u sil
  → ZooKeeper event → Process B uyandı → "ben en küçüğüm" → lock bende!

Ephemeral node:
  Process A çökerse → ZooKeeper session timeout → node otomatik silinir
  → Process B lock'u alır (deadlock yok!)
```

```java
// Curator Framework (Apache) — ZooKeeper lock
@Bean
CuratorFramework curatorFramework() {
    return CuratorFrameworkFactory.builder()
        .connectString("zookeeper:2181")
        .retryPolicy(new ExponentialBackoffRetry(1000, 3))
        .build();
}

@Service
class ZooKeeperLockService {

    @Autowired CuratorFramework curator;

    void runWithLock(String resource, Runnable task) throws Exception {
        InterProcessMutex lock = new InterProcessMutex(
            curator, "/locks/" + resource);

        if (lock.acquire(10, TimeUnit.SECONDS)) {
            try {
                task.run();
            } finally {
                lock.release();
            }
        } else {
            throw new RuntimeException("Lock alınamadı: " + resource);
        }
    }
}
```

---

### Fencing Token (Güvenli Unlock)

```
Problem:
  Process A → lock alır → yavaşlar (GC, network)
  Lock TTL dolar → Process B lock alır
  Process A → devam eder → DB'ye yazar → Process B'nin çalışmasını bozar

Çözüm: Fencing Token
  Lock alırken monoton artan token döner:
    Process A → token=33
    Process B → token=34 (A'nın lock'u dolunca)

  DB'ye yazarken token gönder:
    Process A → WRITE (token=33) → DB: "34'ten küçük, reddet!"
    Process B → WRITE (token=34) → DB: "tamam"

  ZooKeeper'da: zxid (ZooKeeper transaction ID)
  Redis: Lua script ile counter
```

---

## Ne zaman?

```
Distributed lock kullan:
✓ Scheduled job'lar (bir instance çalışsın)
✓ Inventory azaltma (aynı ürünü çift satma önleme)
✓ Kritik section — aynı kaynak üzerinde tek işlem
✓ Leader election (basit senaryolarda)
✓ Rate limiting (token bucket state'i paylaşma)

Distributed lock kullanma:
✗ Yüksek throughput — her işlem için lock → bottleneck
✗ Eventual consistency kabul edilebiliyorsa → optimistic locking (DB version)
✗ Lock TTL belirlemek zor → büyük olursa yavaş, küçük olursa veri bozulur
✗ Uzun süren işlemler → lock renewed etmek gerekir (kompleks)

Alternatifleri düşün:
  - Optimistic locking: DB version/timestamp kontrolü (lock yok, conflict retry)
  - Idempotency key: aynı işlemi iki kez yapmaktan sakın (lock yerine)
  - DB unique constraint: race condition'ı DB'ye bırak
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Race condition önler | Lock TTL belirlemek zor |
| Distributed ortamda senkronizasyon | Lock sahibi çöküşünde TTL beklenmeli |
| Basit Redis implementasyonu | Redis tek node çökerse lock kaybolur |
| ZooKeeper: crash'te otomatik release | Redlock: 5 Redis node yönetimi |
| Throughput korunur (tek işlem) | Deadlock riski (TTL çözse de gecikme) |
| | Fencing token olmadan veri bozulabilir |
