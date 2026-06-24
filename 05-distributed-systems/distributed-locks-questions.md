# 05b — Distributed Locks: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Lock TTL & Temel Hatalar

### Gerçek Hayat Sorunları

---

**Sorun 1: Lock TTL çok kısa ayarlandı — iki process aynı anda kritik bölgede**

```
Senaryo:
  Fatura oluşturma servisi, 3 pod üzerinde çalışıyor.
  Redis distributed lock, TTL = 5 saniye.
  İşlem bazen 10 saniye sürüyor (yoğun DB sorgusu).

  Timeline:
    T+0s:   Pod-1 lock aldı (TTL=5s). Fatura oluşturma başladı.
    T+5s:   TTL doldu → Redis lock'u sildi.
            Pod-1 hâlâ çalışıyor (işlem bitmedi).
    T+5s:   Pod-2 lock aldı → fatura oluşturmaya başladı.
    T+10s:  Pod-1 işlemi bitirdi → DB'ye yazdı.
    T+11s:  Pod-2 işlemi bitirdi → DB'ye yazdı.
    Sonuç:  Aynı fatura iki kez oluşturuldu. Çift ödeme riski.

Kök neden:
  TTL < işlem süresi → lock zamanından önce doldu.
  requestId ile unlock yapmıyorsa Pod-1 de Pod-2'nin lock'unu silecekti.

Çözümler:

  1. TTL'yi yeterince büyük ayarla:
     p99 işlem süresi + buffer:
     p99 = 8s → TTL = 30s (yeterli marj).
     Sorun: process çökerse 30s boyunca kimse lock alamaz.

  2. Lock renewal (watchdog):
     Redisson: tryLock(0, 30, SECONDS) → otomatik watchdog.
     Redisson, lock sahibi thread'i canlıyken lock'u otomatik uzatır.
     Thread çöküşünde watchdog durur → TTL dolar → lock serbest.

     RLock lock = redisson.getLock("invoice:" + id);
     // leaseTime=-1 → Redisson watchdog otomatik renewal
     lock.lock();  // watchdog aktif, manuel TTL yok
     try {
         doWork();
     } finally {
         lock.unlock();
     }

  3. Fencing token + DB idempotency kontrolü:
     İki process aynı anda çalışsa bile DB'de duplicate önle.
     INSERT INTO invoices (id, ...) ON CONFLICT (id) DO NOTHING

  Doğru yaklaşım: Redisson watchdog + DB idempotency birlikte.
```

---

**Sorun 2: unlock() finally bloğu dışında — lock asla serbest bırakılmadı**

```
Senaryo:
  Distributed lock ile korunan kritik section.
  Exception durumunda unlock unutuldu.

  // YANLIŞ
  boolean locked = lock.tryLock(lockKey, requestId, Duration.ofSeconds(30));
  if (!locked) return;

  processOrder(orderId);   // Exception fırlatabilir!
  lock.unlock(lockKey, requestId);  // Exception olursa buraya gelinmez

  Sonuç:
    processOrder() exception fırlatırsa unlock çalışmaz.
    Lock TTL dolana kadar (30s) hiçbir process bu lock'u alamaz.
    Eğer TTL uzunsa (5dk, 10dk): uzun süre blokaj.

  // DOĞRU
  boolean locked = lock.tryLock(lockKey, requestId, Duration.ofSeconds(30));
  if (!locked) {
      log.info("Lock alınamadı");
      return;
  }
  try {
      processOrder(orderId);
  } finally {
      lock.unlock(lockKey, requestId);  // Her durumda çalışır
  }

Önemli:
  TTL: "sigorta" — process çökerse lock eninde sonunda serbest kalır.
  finally: normal akışta lock'u zamanında serbest bırak.
  İkisi birlikte: hem normal hem crash senaryosunu kapsar.

  Redisson kontrolü:
  finally {
      if (lock.isHeldByCurrentThread()) {
          lock.unlock();
      }
  }
  // isHeldByCurrentThread: başka thread'in lock'unu silme
```

---

**Sorun 3: requestId olmadan unlock — başka process'in lock'u silindi**

```
Senaryo:
  Basit DEL komutu ile unlock:

  // YANLIŞ — sadece DEL
  void unlock(String lockKey) {
      redis.delete(lockKey);
  }

  Timeline:
    T+0s:   Process A lock aldı (TTL=10s). Lock value: herhangi bir şey.
    T+10s:  GC pause nedeniyle A yavaşladı, TTL doldu.
    T+10s:  Process B lock aldı.
    T+12s:  Process A GC bitti → unlock() çağırdı → DEL yaptı.
    T+12s:  Process B'nin lock'u silindi!
    T+12s:  Process C lock aldı.
    T+13s:  Process B hâlâ çalışıyor, Process C de çalışıyor → race condition.

Lua script çözümü (atomik check-and-delete):
  String lua = """
      if redis.call('get', KEYS[1]) == ARGV[1] then
          return redis.call('del', KEYS[1])
      else
          return 0
      end
      """;

  // Lock alırken: requestId = UUID.randomUUID().toString()
  // Lock value olarak requestId sakla:
  redis.opsForValue().setIfAbsent(lockKey, requestId, ttl);

  // Unlock ederken:
  // "get → değerim mi? → evet → del" — atomik (Lua)
  // "get → değerim mi? → hayır → 0 dön (silme)"

Neden Lua? GET + DEL iki ayrı komut → aralarında başka process müdahale edebilir.
Lua script: Redis'te tek atomik operasyon olarak çalışır.
```

---

**Sorun 4: Tek Redis node çöktü — lock kayboldu, çift işlem gerçekleşti**

```
Senaryo:
  Tek Redis node ile distributed lock (Redlock değil).
  Process A kritik section'da, lock tutarken Redis node çöktü.

  T+0s:   Process A lock aldı → Redis'te key var.
  T+5s:   Redis node çöktü (RAM'deki lock verisi kayboldu).
  T+6s:   Redis yeniden başladı (RDB/AOF yoksa: boş başlıyor).
  T+6s:   Process B tryLock → lock yok → ALIYOR.
  T+8s:   Process A hâlâ çalışıyor (lock'unun dolmadığını sanıyor).
  Sonuç:  Process A + Process B aynı anda kritik section'da.

Alternatiflere karar verme:

  1. Tek Redis node (kabul edilebilir risk):
     - Düşük kritiklikte (scheduled job yeniden çalışabilir)
     - Aynı işi çift yapmak tolere edilebilirse
     - Idempotency garantisi varsa (DB duplicate guard)
     - Avantaj: basit, hızlı, operasyonel yük az

  2. Redlock (5 bağımsız Redis node):
     - Ödeme, stok azaltma gibi kritik işlemler
     - Tek node çöküşünde quorum (3/5) hâlâ çalışıyor
     - Dezavantaj: 5 node yönetimi, ağ gecikmesi × 5

  3. ZooKeeper / etcd:
     - Güçlü consistency garantisi (Raft/ZAB)
     - Ephemeral node: process çökünce otomatik release
     - Fencing token doğal olarak üretilir (zxid)
     - Dezavantaj: daha yüksek latency, operasyonel karmaşıklık

  4. DB unique constraint (lock yerine):
     INSERT INTO job_executions (job_id, node_id, started_at)
     ON CONFLICT (job_id) DO NOTHING
     -- Unique constraint: DB'nin ACID garantisi ile lock
     -- Avantaj: distributed system yok, transaction içinde
     -- Dezavantaj: DB'ye ek yük, TTL mekanizması manuel

Karar: kritiklik × operasyonel maliyet dengesine göre seç.
```

---

**Sorun 5: GC pause + fencing token yok — stale process veriyi bozdu**

```
Senaryo:
  Process A lock aldı (token=33), hesaplama yapıyor.
  GC pause: 40 saniye (stop-the-world, büyük heap).
  Lock TTL: 30 saniye → TTL doldu.
  Process B lock aldı (token=34), işlemini tamamladı, DB'ye yazdı.
  Process A GC'den uyandı → "lock bende" sanıyor → DB'ye yazdı.
  Sonuç: Process B'nin doğru verisinin üstüne Process A'nın stale verisi yazıldı.

  Bu senaryo teorik değil: Martin Kleppmann (DDIA yazarı) Redlock için
  bu senaryoyu belgelimiştir. Fencing token olmadan hiçbir lock güvenli değildir.

Fencing token çözümü:
  Lock alırken monoton artan token döner.
  DB'ye yazarken token'ı da gönder.
  DB: "benden önce gelen token'ı reddediyorum."

  // Lock alırken token döner
  LockResult result = lock.acquire("resource:123");
  long fencingToken = result.getToken();  // monoton artan: 33, 34, 35...

  // DB'ye yazarken token gönder
  repository.updateWithFencing(data, fencingToken);

  // DB tarafı (SQL):
  UPDATE resources
  SET data = ?, last_token = ?
  WHERE id = ? AND last_token < ?
  -- Eski token gelirse güncelleme yapılmaz (etkilenen satır: 0)

ZooKeeper: zxid doğal fencing token.
Redis: Lua script ile atomic counter → token üret.

Sonuç: Fencing token = distributed lock'un "seatbelt"i.
Lock mekanizması başarısız olursa bile veri katmanı son savunma hattı.
```

---

**Sorun 6: Redlock — Redis replica kullanan yanlış kurulum**

```
Senaryo:
  Ekip: "5 Redis node" kurdu ama 1 primary + 4 replica (sentinel cluster).
  Redlock algoritması: 5 BAĞIMSIZ node varsayar.

  T+0:  Process A → Primary'e lock yazdı (SET NX).
        Primary → 4 Replica'ya asenkron replikasyon başladı.
  T+1:  Primary çöktü. Replikasyon tamamlanmadı.
  T+2:  Sentinel: yeni primary seçti (eski replica).
        Yeni Primary'de lock YOK (replikasyon gelmedi).
  T+3:  Process B → yeni Primary'e lock yazdı → ALDI.
  T+4:  Process A hâlâ lock'unu tutuyor (çökmedi, çalışıyor).
  Sonuç: İki process aynı anda lock sahibi → race condition.

Redlock'un doğru kurulumu:
  5 BAĞIMSIZ, replica'sız standalone Redis node.
  Quorum: 3 node'dan "SET NX OK" → lock başarılı.
  1 node çöküşü: 4 node hâlâ var → quorum (3) sağlanabilir.
  Neden standalone: replica asenkron → failover'da lock kaybı.

  Redisson konfigürasyonu:
  Config config = new Config();
  // NOT: useClusterServers değil!
  // 5 ayrı, bağımsız Redis instance
  config.useSentinelServers()...  // YANLIŞ (asenkron replikasyon)
  
  // DOĞRU: 5 standalone → MultiRedissonClient veya manuel
  // Redisson: MultiLock ile 5 ayrı client birleştir
  RLock lock1 = client1.getLock("resource");
  RLock lock2 = client2.getLock("resource");
  RLock lock3 = client3.getLock("resource");
  RLock lock4 = client4.getLock("resource");
  RLock lock5 = client5.getLock("resource");
  RLock multiLock = redisson.getMultiLock(lock1, lock2, lock3, lock4, lock5);
  multiLock.lock();

Martin Kleppmann notu:
  Redlock bile %100 güvenli değil (fencing token olmadan).
  Yüksek güvenlik gerektiren: ZooKeeper + fencing token (zxid).
```

---

**Sorun 7: Lock contention — yüksek throughput'ta bottleneck**

```
Senaryo:
  E-ticaret: her sipariş için stok azaltma → distributed lock.
  Black Friday: saniyede 10.000 sipariş aynı ürün için.

  Her sipariş:
    1. Lock almaya çalış (500ms bekle)
    2. Stok oku
    3. Stok azalt
    4. Lock bırak

  Sorun:
    10.000 sipariş → aynı lock key → sırada bekliyor.
    Lock alınma süresi: 10ms.
    Tüm siparişlerin bitmesi: 10.000 × 10ms = 100 saniye!
    Timeout: siparişlerin çoğu "lock alınamadı" hatası aldı.

Alternatif 1: DB Optimistic Locking (version/CAS)
  UPDATE stock SET quantity = quantity - 1, version = version + 1
  WHERE product_id = ? AND version = ? AND quantity > 0

  Conflict → retry (lock yok, sadece retry).
  Low conflict rate → distributed lock'tan çok daha hızlı.
  High conflict rate → çok retry → yine yavaş.

Alternatif 2: DB unique constraint + idempotency
  Stok azaltmayı serialized yapmak yerine:
  - DB transaction ile CAS (Compare-And-Swap)
  - SELECT FOR UPDATE (DB-level row lock)
  - Race condition'ı DB'ye devret, uygulama katmanında lock yok.

Alternatif 3: Redis atomic decrement (lock olmadan)
  DECRBY stock:product:123 1 → atomik azaltma
  Eğer sonuç < 0 → INCRBY ile geri al (rollback)
  Redis single-threaded → DECRBY atomik → race condition yok.
  Ama: DB ile sync ayrı problem.

Alternatif 4: Kafka ile sıralı işlem
  Tüm stok azaltma event'larını aynı partition'a gönder.
  Consumer: tek thread, sıralı işlem → race condition yok.

Sonuç: Distributed lock → low-throughput, kritik bölümler.
Yüksek throughput → DB CAS, Redis atomic ops, veya event sıralama.
```

---

### Mülakat Soruları — Temel Kavramlar

**Junior / Mid:**

1. Distributed lock nedir? Tek JVM `synchronized` ile farkı nedir?

   > **Beklened:** Distributed lock: birden fazla farklı JVM/sunucu üzerindeki process'lerin aynı kaynağa aynı anda erişmesini engelleyen mekanizma. `synchronized`: tek JVM içinde thread-level senkronizasyon — JVM dışına çıkmaz. Distributed sistemde: 3 pod aynı cron job'ı çalıştırabilir → aynı faturayı 3 kez oluşturabilir. Distributed lock: sadece birini devreye sokar. Uygulama: Redis SET NX (Set if Not eXists) veya ZooKeeper ephemeral node. Kullanım yerleri: cron job tekil çalıştırma, stok azaltma (overselling önleme), idempotent API işlemleri, basit lider seçimi.

2. Redis ile distributed lock nasıl alınır? `SET NX PX` ne anlama gelir?

   > **Beklened:** `SET key value NX PX ttl` tek atomik komut. NX: "Not eXists" — key zaten varsa set etme (başka biri lock'u tutuyorsa alma). PX: TTL millisecond olarak. Atomik olması kritik: GET + SET iki ayrı komut olsaydı araya başka process girebilirdi. Java örneği: `redis.opsForValue().setIfAbsent(lockKey, requestId, ttl)`. Dönen değer: true → lock alındı, false → başkası tutuyor. TTL: process çökünce lock otomatik silinir (deadlock önleme). Value olarak requestId: unlock'ta "bu benim lock'um mu?" kontrolü.

3. Unlock yaparken neden `DEL` değil Lua script kullanırız?

   > **Beklened:** Sorun: GET + DEL iki ayrı komut → aralarında başka process müdahale edebilir. Senaryo: Process A: GET → "değer benim" → Process B aynı anda GET → ikisi de "benim" der → A DEL eder → B DEL eder → B başka bir process'in lock'unu sildi. Lua script: Redis'te atomik çalışır — tek işlem gibi. İçerik: "eğer key'in değeri benim requestId'm ise → DEL, değilse → 0 dön." Bu sayede: başka process'in lock'unu silemezsin. Not: Lua script Redis single-threaded olduğu için gerçekten atomik — iki Lua script aynı anda çalışmaz.

4. Lock TTL ne olmalıdır? Çok kısa veya çok uzun olursa ne olur?

   > **Beklened:** Çok kısa TTL (işlem süresinden az): TTL dolar → başka process lock alır → iki process aynı anda kritik bölgede → race condition. Çok uzun TTL: process çökerse lock TTL dolana kadar beklenilir → iş duraksadar. Doğru hesap: `TTL = p99 işlem süresi × 2 + güvenlik marjı`. p99 = 8s → TTL ≈ 30s. Daha iyi yaklaşım: Redisson watchdog — process canlıyken lock'u otomatik yeniler, process çökünce watchdog durur → TTL dolar. Bu durumda TTL daha kısa ayarlanabilir (sadece crash senaryosu için).

---

**Senior / Architect:**

5. Fencing token nedir? Neden distributed lock'ların tek başına yeterli olmadığını gösterir?

   > **Beklened:** Problem: GC pause veya network partition → process lock'un dolduğunun farkında değil → stale veri yazar. Timeline: A lock'u alır (token=33) → GC pause 40s → TTL dolar → B lock alır (token=34) → B işini yapar → A uyanır, "lock bende" sanır → DB'ye yazar → B'nin verisini bozar. Fencing token çözümü: lock alınca monoton artan token verilir. DB'ye yazarken token gönderilir. DB: "eski token (33) geldi, yeni token (34) zaten uygulanmış → red." ZooKeeper: zxid doğal fencing token. Redis: Lua ile counter. Ders: lock mekanizması güvenilmez (process askıya alınabilir) → son savunma DB katmanında olmalı.

6. ZooKeeper ephemeral sequential node ile locking nasıl çalışır? Redis'e göre avantajı nedir?

   > **Beklened:** ZooKeeper algoritması: (1) Ephemeral sequential node yarat: `/locks/resource/lock-000001`. (2) Tüm children listele. (3) Ben en küçük node muyum → evet → lock bende. Hayır → bir önceki node'u watch et. (4) Bir önceki silinince → tekrar listele → en küçük miyim → devam. Release: node'umu sil → sonraki process uyandı. Ephemeral: process/session çökünce ZooKeeper otomatik siler → deadlock yok, TTL yok. Watch: polling yok, event-driven. Redis'e göre avantajları: (1) TTL yönetimi yok — crash'te otomatik. (2) zxid = doğal fencing token. (3) Strong consistency (ZAB) — Redis asenkron replikasyon riski yok. Dezavantaj: daha yüksek latency, ZooKeeper bağımlılığı.

---

## Bölüm 2: Redlock & İleri Düzey

### Mülakat Soruları

**Junior / Mid:**

7. Neden tek Redis node distributed lock için risklidir?

   > **Beklened:** Redis single node → SPOF (Single Point of Failure). Redis restart olursa: RDB snapshot'tan kurtarma → son birkaç saniyenin verisi kaybolabilir (lock dahil). Redis AOF her fsync: daha güvenli ama yavaş. Senaryo: Process A lock aldı → Redis çöktü → Redis yeniden başladı (lock yok) → Process B lock aldı → Process A + B aynı anda kritik bölgede. Çözüm seçenekleri: (1) Tek node + idempotency (düşük kritiklik). (2) Redlock (5 bağımsız node). (3) ZooKeeper/etcd (strong consistency). Risk kabulü: işin kritikliğine göre karar ver.

8. Distributed lock ne zaman KULLANILMAZ? Alternatifler nelerdir?

   > **Beklened:** Kullanılmaması gereken durumlar: Yüksek throughput — her işlem için lock → bottleneck, ölçeklenmez. Eventual consistency yeterliyse — kullanıcı bazlı counter gibi. Uzun süren işlemler — lock renewal karmaşık. Alternatifler: (1) DB optimistic locking: `UPDATE ... WHERE version = ?` — conflict durumunda retry, lock yok. (2) DB unique constraint: race condition'ı DB'nin ACID garantisine bırak. (3) Redis DECRBY: atomik decrement, stok azaltma için lock gereksiz. (4) Idempotency key: aynı işlemi iki kez yapmaktan sakın, lock yerine deduplicate. (5) Kafka/event sıralaması: aynı partition, tek consumer → sequential işlem.

---

**Senior / Architect:**

9. Redlock algoritmasını adım adım anlat. Hangi koşulda güvenlidir?

   > **Beklened:** Redlock: 5 bağımsız (standalone, replica'sız) Redis node. Adımlar: (1) `t1 = now()`. (2) 5 node'a sırayla `SET NX PX TTL` gönder. (3) ≥3 (quorum) node'dan "OK" aldı VE `now() - t1 < TTL/2` → lock başarılı. Geçerli kalan süre: `TTL - (now() - t1)`. (4) Quorum sağlanamadıysa: başarılı olan tüm node'larda `DEL` yap → retry (backoff). Release: 5 node'un hepsine `DEL` gönder (hangisi başarılı olmuş bilinmez). Güvenli koşul: 5 bağımsız node (replica asenkron → failover'da lock kaybolur). Kritik not: Martin Kleppmann — fencing token olmadan GC pause/network partition sonrası güvensiz. Redlock + fencing token birlikte kullanılmalı.

10. Distributed lock, DB transaction ile birlikte nasıl kullanılmalıdır? Hangisi hangisini kapsıyor?

    > **Beklened:** İki farklı garantisi: Distributed lock: birden fazla JVM/process'in aynı anda kritik bölgeye girmesini engeller. DB transaction: ACID, tek bir DB işleminin atomikliğini garantiler. İkisi farklı sorunları çözer. Örnek — yanlış: "DB transaction var, lock lazım değil." Senaryo: 3 pod aynı anda sipariş oluşturursa → 3 ayrı transaction başlar → 3 ayrı SELECT FOR UPDATE → deadlock veya race. Doğru kombinasyon: Distributed lock → sadece 1 process kritik bölgeye girer → o process DB transaction başlatır → ACID garantisi içinde. Lock: serileştirme. Transaction: atomiklik. SELECT FOR UPDATE: DB-level row lock (distributed lock yerine geçebilir, ama DB bağımlı).

---

## Bölüm 3: ZooKeeper Entegrasyonu & Pratik

### Mülakat Soruları

**Senior / Architect:**

11. ZooKeeper session expire olursa distributed lock ne olur?

    > **Beklened:** ZooKeeper session: her client ile ZooKeeper arasındaki bağlantı, timeout süresiyle. Session timeout: network kesintisi veya client process pause (GC) → ZooKeeper "client öldü" der. Ephemeral node: session'a bağlı → session expire → node otomatik silinir → lock serbest. Process hâlâ canlıysa ama GC pause'dan uyandıysa: "ben lock sahibiyim" sanıyor → ama ZooKeeper'da lock silinmiş → başka process lock almış. Fencing token (zxid) bu senaryoyu kurtarır: eski process DB'ye yazarken eski zxid ile gelir → reddedilir. Curator Framework: session expire sonrası reconnect ve lock re-acquire retry policy ile yönetilmeli.

12. Distributed lock senaryosunda "herd effect" (thundering herd) nedir? ZooKeeper nasıl önler?

    > **Beklened:** Herd effect: lock serbest bırakılınca tüm bekleyen process'ler aynı anda "lock al" denemesi yapıyor → sürü etkisi → ZooKeeper/Redis yük patlaması. Kötü implementasyon: tüm process'ler lock'a watcher koyarsa → lock silinince hepsi uyandı → hepsi getChildren yapıyor → hepsi lock almaya çalışıyor. ZooKeeper çözümü: sequential ephemeral node + bir öncekini watch et. Process B sadece Process A'nın node'unu izler. Lock serbest kalınca: sadece Process B uyanır (Process C, D... uyumaya devam). "Sıraya girmek" — herd yok, sadece sıradaki uyanır. Redis'te: Redisson FairLock — FIFO sırası, herd effect yok.

---

## Karma — Architect Seviyesi

13. **"Bir engineer 'stok azaltmak için her işleme distributed lock koyacağız' dedi. Ne söylersin?"**

    > **Beklened:** Yanlış yaklaşım — önce alternatifi değerlendir. Her işlem için lock: stok her satışta serialized → throughput bottleneck. Black Friday gibi yüksek yük: binlerce istek aynı ürün için aynı lock'ta bekler → timeout → hata. Redis DECRBY alternatifi: `DECRBY stock:product:123 1` → atomik, lock yok. Sonuç < 0 → stok yok → INCRBY ile geri al. DB CAS alternatifi: `UPDATE stock SET qty = qty-1 WHERE id=? AND qty>0` → etkilenen satır 0 ise stok yok. DB row lock (SELECT FOR UPDATE): uygulama lock değil, DB-level — daha doğal. Distributed lock gerçekten gerekirse: Redisson + watchdog + fencing. Sonuç: ilk önce ne kadar throughput gerekiyor? Lock gerçekten tek çözüm mü?

14. **"Redis distributed lock'u production'a alacaksın. Hangi kararları verirsin?"**

    > **Beklened:** (1) Kritiklik: ödeme/stok → Redlock (5 node) veya ZooKeeper. CRUD, cron → tek node yeter. (2) TTL: p99 işlem süresi × 2. Uzun işlem → Redisson watchdog (otomatik renewal). (3) Unlock: her zaman Lua script (atomik check-and-delete). requestId: UUID, her lock alma işleminde unique. (4) finally bloğu: unlock mutlaka finally içinde. (5) İdempotency: lock'a ek olarak DB'de duplicate guard (ON CONFLICT DO NOTHING). (6) Fencing token: yüksek kritiklikte — lock sürdükçe geçerli token, DB katmanında kontrol. (7) Monitoring: lock alma/bekleme süresi, timeout sayısı, lock contention (Micrometer metrik). (8) Circuit breaker: Redis down → lock alınamıyor → fallback (featureflag: lock'suz çalış veya reject).

15. **"Distributed lock olmadan race condition'ı önlemenin üç yolunu söyle ve hangisini ne zaman tercih edersin?"**

    > **Beklened:** (1) DB Optimistic Locking: `version` kolonu → `WHERE version = ?` → 0 satır etkilendiyse → conflict → retry. Ne zaman: okuma çok, yazma çakışması az (low contention). Avantaj: lock yok, throughput yüksek. Dezavantaj: yüksek çakışmada çok retry → yavaş. (2) DB unique constraint / SELECT FOR UPDATE: `INSERT ... ON CONFLICT DO NOTHING` veya `SELECT ... FOR UPDATE`. Ne zaman: tek DB, transaction içinde yeterli. Avantaj: DB garantisi, uygulama lock karmaşıklığı yok. Dezavantaj: DB'ye ek yük, row-level lock → satır bazlı darboğaz olabilir. (3) Idempotency key + deduplication: aynı işlemi tekrar yapmak yerine onceki sonucu döndür. `processed_requests` tablosu, request_id unique. Ne zaman: retry mümkün, aynı işlemi çift yapmak sorun. Avantaj: dağıtık senaryoda en güvenli. Dezavantaj: deduplication storage gerekir. Distributed lock: yukarıdakiler yetmediğinde, critical section gerçekten serileştirilmeli.
