# 03 — Veritabanları: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: SQL vs NoSQL & ACID vs BASE

### Gerçek Hayat Sorunları

---

**Sorun 1: Yanlış NoSQL seçimi — sipariş verisi MongoDB'ye yazıldı, tutarsızlık oldu**

```
Senaryo:
  E-ticaret: sipariş + stok güncelleme aynı işlemde yapılmalı.
  Hız için MongoDB seçildi.
  
  İşlem:
    1. orders koleksiyonuna sipariş yaz → başarılı
    2. inventory koleksiyonunu güncelle → network hatası, başarısız
  
  Sonuç: Sipariş kayıtlı ama stok düşmedi → negatif stok, mali zarar.

Problem: MongoDB 4.0+ multi-document transaction var ama:
  - Tek replica set'te çalışır, sharded cluster'da kısıtlı.
  - Performans maliyeti yüksek (RDBMS transaction'ına kıyasla ağır).
  - Distributed transaction = saga pattern gerektiriyor.

Doğru karar matrisi:
  "Para veya stok gibi konsistensi kritik mi?" → PostgreSQL
  "Yazma hızı > tutarlılık mı?" → Cassandra / DynamoDB (eventual consistent)
  "Esnek şema + okuma ağırlıklı katalog mu?" → MongoDB

Düzeltme:
  Sipariş + stok → PostgreSQL (ACID, foreign key, transaction).
  MongoDB: ürün katalogu (esnek özellikler, hiyerarşik veri).
  Polyglot: her veri tipi kendi doğal DB'sinde.
```

---

**Sorun 2: Transaction isolation seviyesi yanlış — dirty read ile yanlış stok görüldü**

```sql
-- Senaryo: Aynı anda iki kullanıcı aynı ürünün son stokunu satın aldı.
-- Stok: 1 adet.

-- T1 başladı, henüz commit etmedi:
UPDATE inventory SET stock = 0 WHERE product_id = 42;
-- stock = 0, ama commit yok

-- T2 aynı anda çalışıyor (READ UNCOMMITTED isolation):
SELECT stock FROM inventory WHERE product_id = 42;
-- → 0 döndü (dirty read!) → "stok yok" dedi → sipariş reddedildi
-- T1 rollback yaptı → stok tekrar 1 oldu → ama T2 zaten reddetmişti

-- Tersine senaryo: T2 dirty read ile 1 gördü, sipariş oluşturdu
-- T1 commit etti (stok 0) → iki sipariş var, stok 1 → oversell

-- Isolation seviyeleri ve korudukları:
-- READ UNCOMMITTED → dirty read, non-repeatable read, phantom read
-- READ COMMITTED   → dirty read korunur [PostgreSQL default]
-- REPEATABLE READ  → dirty + non-repeatable korunur [MySQL InnoDB default]
-- SERIALIZABLE     → hepsi korunur, ama yavaş

-- Stok yönetimi için doğru yaklaşım:
BEGIN;
SELECT stock FROM inventory WHERE product_id = 42 FOR UPDATE;  -- satırı kilitle
-- stock = 1 → sipariş oluştur
UPDATE inventory SET stock = stock - 1 WHERE product_id = 42;
COMMIT;
-- FOR UPDATE: diğer transaction'lar bu satırı okuyamaz → sıralı işlem garantisi
```

---

**Sorun 3: Deadlock — iki transaction birbirini bekledi**

```sql
-- Senaryo: Para transferi — iki kullanıcı birbirlerine aynı anda para transfer ediyor.

-- T1: Ali'den Ayşe'ye transfer
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Ali kilitlendi
-- ...küçük gecikme...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Ayşe'yi bekle → WAIT

-- T2: Ayşe'den Ali'ye transfer (aynı anda)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Ayşe kilitlendi
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Ali'yi bekle → WAIT

-- Sonuç: T1 Ayşe'yi bekler, T2 Ali'yi bekler → DEADLOCK
-- PostgreSQL: "ERROR: deadlock detected" → bir transaction rollback edilir

-- ÇÖZÜM: Kilitleri her zaman aynı sırada al (küçük ID önce)
void transfer(long fromId, long toId, BigDecimal amount) {
    long first = Math.min(fromId, toId);
    long second = Math.max(fromId, toId);
    // Her zaman küçük ID'yi önce kilitle → deadlock imkânsız
    lockAndUpdate(first, second, fromId, toId, amount);
}

-- Diğer çözüm: SELECT ... FOR UPDATE NOWAIT
-- Kilit alamıyorsa hemen hata fırlat → retry → deadlock bekleme yok
UPDATE accounts SET balance = balance - 100 WHERE id = 1 FOR UPDATE NOWAIT;
```

---

### Mülakat Soruları — SQL vs NoSQL & ACID

**Junior / Mid:**

1. ACID nedir? Her harfini örnekle açıkla.

   > **Beklenen:** Atomicity: bankada para transferi — iki hesap güncelleme ya ikisi de olur ya hiçbiri. Yarıda kesilirse rollback. Consistency: hesap bakiyesi negatife düşemez — constraint ihlali → transaction red. Isolation: T1 commit olmadan T2 T1'in değişikliklerini görmez (isolation seviyesine bağlı). Durability: commit sonrası sunucu çöktü → veri WAL'dan kurtarılır. Pratik: PostgreSQL default READ COMMITTED → dirty read yok ama non-repeatable read var.

2. Ne zaman MongoDB, ne zaman PostgreSQL kullanırsın?

   > **Beklenen:** PostgreSQL: para, stok, sipariş, ilişkisel veri, ACID kritik, karmaşık sorgu, reporting. MongoDB: ürün katalogu (farklı kategoride farklı özellik), CMS içerik, kullanıcı profili (esnek alanlar), log, IoT event. Karar sorusu: "Birden fazla dokümanı aynı anda atomik güncellemem gerekiyor mu?" → evet → PostgreSQL. "Her kaydın şeması farklı ve JOIN yok mu?" → MongoDB. Yanlış seçim: sipariş + stok MongoDB → distributed transaction problemi.

3. `READ COMMITTED` ve `REPEATABLE READ` arasındaki fark ne zaman önemlidir?

   > **Beklenen:** READ COMMITTED: her sorgu o anki commit'lenmiş veriyi görür. Aynı transaction'da aynı sorguyu iki kez çalıştırsan farklı sonuç alabilirsin (non-repeatable read). REPEATABLE READ: transaction başında snapshot alınır, aynı sorgu hep aynı sonuç. Ne zaman önemli: rapor, hesaplama, birden fazla sorgu sonuçlarının tutarlı olması gerektiğinde. Örnek: inventory raporunda toplam stok = her ürün ayrı sorgu → READ COMMITTED → paralel update varsa toplamlar tutarsız. REPEATABLE READ → snapshot tutarlı.

---

**Senior / Architect:**

4. Finansal bir sistemde hangi isolation seviyesi kullanırsın? Neden SERIALIZABLE her zaman doğru değil?

   > **Beklenen:** Finansal: çoğu senaryo READ COMMITTED + `SELECT FOR UPDATE` ile yönetilir. Neden SERIALIZABLE değil: MVCC serializability kontrolü ek overhead — "serialization failure" hataları → retry mekanizması zorunlu. Throughput düşer. Doğru yaklaşım: kritik satır kilitleri `FOR UPDATE`, application-level concurrency control, idempotency key. SERIALIZABLE: karmaşık bütünlük kuralı (A+B+C = 100 olmalı gibi) gerektiren sorgularda. Pratik: Stripe, PayPal → READ COMMITTED + explicit lock + idempotency key.

---

## Bölüm 2: PostgreSQL Internals

### Gerçek Hayat Sorunları

---

**Sorun 4: Table bloat — AUTOVACUUM yetişemedi, disk doldu**

```
Senaryo:
  Yüksek yazma trafiği: saniyede 5000 UPDATE.
  AUTOVACUUM default ayarlar → yetişemiyor.
  Dead tuple birikimi → tablo boyutu 50GB → gerçek veri 5GB.
  Disk doldu → PostgreSQL yeni yazma yapamadı → downtime.

MVCC'nin getirisi:
  Her UPDATE → eski satır dead tuple olarak kalır → VACUUM temizler.
  UPDATE-heavy tablo → çok dead tuple → AUTOVACUUM sık çalışmalı.

Teşhis:
  SELECT relname, n_dead_tup, n_live_tup,
         round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
  -- dead_pct > %20 → acil VACUUM

Düzeltme:
  -- Acil: manuel VACUUM
  VACUUM ANALYZE orders;
  VACUUM FULL orders;  -- tablo tamamen yeniden yaz, disk geri al (LOCK alır!)

  -- Kalıcı: AUTOVACUUM tuning
  ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- %1 dead tuple → vacuum tetikle (default %20)
    autovacuum_vacuum_cost_delay = 2,       -- daha agresif (default 20ms)
    autovacuum_vacuum_threshold = 50        -- 50 dead tuple eşik (default 50 zaten)
  );

  -- pg_wal dolmasın: wal_keep_size, archive_cleanup doğru ayarla
  -- Monitoring: dead_tup / live_tup oranını alert sisteme ekle (>%20 warn)
```

---

**Sorun 5: Yanlış composite index sırası — sorgu index'i kullanamadı**

```sql
-- Senaryo: orders tablosunda müşteri bazlı tarih sorgusu yavaş.
-- Index var ama kullanılmıyor.

-- Mevcut index:
CREATE INDEX idx_orders_date_customer ON orders(created_at, customer_id);

-- Sorgu:
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC;
-- EXPLAIN ANALYZE: Seq Scan on orders → 500ms

-- Neden? Sol-prefix kuralı:
-- (created_at, customer_id) index'i:
--   WHERE created_at = X → kullanılır (sol prefix)
--   WHERE customer_id = X → KULLANILMAZ (sol prefix değil!)
--   WHERE created_at = X AND customer_id = Y → kullanılır

-- Düzeltme: Sorgu pattern'ına göre sıra belirle
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at DESC);
-- Şimdi:
--   WHERE customer_id = 42 → kullanılır
--   WHERE customer_id = 42 ORDER BY created_at DESC → index-only scan mümkün

-- EXPLAIN ANALYZE sonrası:
-- Index Scan using idx_orders_customer_date on orders → 0.8ms

-- Genel kural: Eşitlik sütunları önce, range/sort sütunları sonra
-- (status, customer_id, created_at) → WHERE status='ACTIVE' AND customer_id=42 ORDER BY created_at
```

---

**Sorun 6: WAL disk doldu — PostgreSQL tamamen durdu**

```
Senaryo:
  pg_wal dizini 100GB'a ulaştı → disk %100 → PostgreSQL write yapamadı → crash.
  
Neden WAL büyüdü?
  1. Replica bağlantısı kesildi ama WAL streaming duraksatılmadı
     → WAL dosyaları replica catch-up edene kadar saklandı.
  2. Long-running transaction → WAL kesilemedi.
  3. wal_keep_size çok yüksek → gereksiz WAL saklama.

Teşhis:
  SELECT pg_walfile_name(pg_current_wal_lsn());
  SELECT sent_lsn, replay_lsn, (sent_lsn - replay_lsn) AS lag
  FROM pg_stat_replication;
  -- lag büyükse replica geride kalmış

Düzeltme:
  Acil: pg_wal'daki gereksiz segment'leri temizle
  SELECT pg_switch_wal();  -- mevcut WAL segment'ini tamamla, yenisine geç
  Replica yeniden bağlan → WAL streaming devam et.
  
  Kalıcı:
  wal_keep_size = 512MB    -- default: replica geride kalınca tutulacak max WAL
  max_wal_size = 2GB       -- checkpoint'ler arası max WAL (küçük → sık checkpoint)
  
  Monitoring: pg_stat_replication.replay_lag → alert (lag > 10dk → sorun)
  WAL archiving: S3'e archive → pg_wal dizini küçük kalır.
```

---

### Mülakat Soruları — PostgreSQL Internals

**Junior / Mid:**

5. MVCC nedir? Okuma ve yazma işlemleri birbirini neden bloke etmez?

   > **Beklenen:** Multi-Version Concurrency Control: her UPDATE yeni satır versiyonu üretir, eski silinmez — sadece xmax doldurulur. Okuyucu: kendi transaction başlangıcında var olan versiyonu görür. Yazıcı: yeni versiyon yazar, eskisi hâlâ var. Sonuç: okuyucu yazıcıyı beklemez, yazıcı okuyucuyu beklemez. Ödeme: dead tuple birikimi → VACUUM zorunlu. Karşı örnek: MySQL MyISAM → tablo seviyesi lock → okuma yazma birbirini bloke eder.

6. `EXPLAIN ANALYZE` çıktısında nelere bakarsın?

   > **Beklenen:** Seq Scan vs Index Scan: Seq Scan büyük tabloda → index eksik veya kullanılmıyor. Rows estimate vs actual rows: büyük fark → istatistikler güncel değil → ANALYZE çalıştır. Actual time: node başına gerçek süre. Loops: NestLoop'da iç döngü kaç kez çalıştı. Filter: "Rows Removed by Filter" yüksekse → index o filtreden geçiremiyor. Hash Join vs Nested Loop vs Merge Join: work_mem düşükse disk spill. Önemli: EXPLAIN ANALYZE gerçek sorgu çalıştırır, production'da dikkatli kullan.

7. B-tree, GIN ve BRIN index'lerini hangi durumlarda kullanırsın?

   > **Beklenen:** B-tree: genel amaçlı, eşitlik ve range sorguları (=, <, >, BETWEEN, LIKE 'prefix%'). Neredeyse her durum için. GIN: array, JSONB, full-text search — çoklu değer içeren sütun. `tags @> ARRAY['java']` gibi "içerir" sorguları. BRIN: fiziksel olarak artan sütunlar (timestamp, serial ID). Çok küçük index (blok bazlı min/max), büyük append-only tablolarda etkili. Hash: sadece eşitlik (=), büyük string'lerde B-tree'den hızlı ama range yok.

---

**Senior / Architect:**

8. Yüksek yazma trafiğinde AUTOVACUUM nasıl tune edilir? Neden kritik?

   > **Beklened:** AUTOVACUUM kritik: MVCC → dead tuple → disk şişme, transaction ID wraparound (XID wraparound → veri kaybı riski!). Tune: `autovacuum_vacuum_scale_factor` küçült (0.01) → %1 dead'de tetikle. `autovacuum_vacuum_cost_delay` düşür → daha agresif çalış. `autovacuum_max_workers` artır → paralel tablo. Per-table override: çok aktif tablolar için ALTER TABLE SET. Monitoring: `pg_stat_user_tables.n_dead_tup`, `age(datfrozenxid)` — bu 2 milyar'a yaklaşırsa kritik. XID wraparound: önlemek için `VACUUM FREEZE`.

9. `SELECT FOR UPDATE`, `FOR SHARE`, `SKIP LOCKED` ne zaman kullanılır?

   > **Beklened:** FOR UPDATE: satırı hem oku hem kilitle → kimse güncelleyemez/silemez. Stok rezervasyonu, sıralı işlem. FOR SHARE: satırı oku ve paylaşımlı kilitle → başkası da okuyabilir ama güncelleyemez. FOR UPDATE NOWAIT: kilit alamıyorsa hemen hata → retry (deadlock bekleme yok). SKIP LOCKED: kilitli satırları atla → iş kuyruğu (job queue). Örnek: N worker, M görev → her worker `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` → aynı görevi iki worker almaz, birbirini beklemez.

---

## Bölüm 3: Connection Pooling (HikariCP)

### Gerçek Hayat Sorunları

---

**Sorun 7: Pool exhaustion — tüm thread'ler bağlantı bekliyor**

```
Senaryo:
  Yüksek trafik: HikariCP pool size = 10, request sayısı artı.
  Tüm 10 bağlantı dolu → yeni request'ler bekliyor.
  connectionTimeout: 30sn (default) → 30sn sonra exception.
  Kullanıcılar "503 Service Unavailable" alıyor.

Teşhis:
  HikariCP metric'ler (Micrometer):
  hikaricp.connections.active   → şu an kullanılan
  hikaricp.connections.pending  → bekleyen thread sayısı (bu artıyorsa kötü)
  hikaricp.connections.timeout  → zaman aşımı sayısı

Sebepler:
  1. Long-running query → bağlantıyı uzun tutuyor → pool doldu.
  2. Transaction leak: bağlantı alındı, commit/rollback yok → asla geri dönmüyor.
  3. Pool size çok küçük.

Düzeltme:
  spring:
    datasource:
      hikari:
        maximum-pool-size: 20          # artır (ama dikkat: DB limiti)
        minimum-idle: 5
        connection-timeout: 5000       # 5sn (30sn'de kullanıcı zaten gitti)
        idle-timeout: 600000           # 10dk idle → bağlantıyı kapat
        max-lifetime: 1800000          # 30dk → DB tarafı timeout öncesi yenile
        leak-detection-threshold: 2000 # 2sn'den uzun tutulan bağlantı → log

  Long query: pg_stat_activity izle, sorgu timeout ekle:
  statement_timeout: 10s → 10sn üzerinde sorgu kill edilir.
```

---

**Sorun 8: Çok instance × pool size = DB bağlantı limitini aştı**

```
Senaryo:
  PostgreSQL max_connections = 200.
  Uygulama: pool size = 20.
  K8s: 15 replica.
  Toplam bağlantı: 15 × 20 = 300 → max_connections aşıldı.
  Yeni bağlantı: "FATAL: sorry, too many clients already"

Neden önemli:
  Her PostgreSQL bağlantısı ~5-10MB bellek → 300 × 10MB = 3GB sadece bağlantı.
  PostgreSQL: bağlantı başına process (worker process) → yüksek context switch.

Çözüm 1: Pool size'ı küçült
  15 replika × 10 pool = 150 < 200. Ama iş yükü buna izin veriyor mu?

Çözüm 2: PgBouncer (önerilen)
  Uygulama → PgBouncer → PostgreSQL
  Transaction-mode pooling:
    150 uygulama bağlantısı → PgBouncer → 20 PostgreSQL bağlantısı
    Transaction bitince bağlantı pool'a döner (session bitmeden!)
  
  Avantaj: PostgreSQL'e çok az bağlantı, uygulama tarafı bol bağlantı.
  Dezavantaj: prepared statement, SET komutları, LISTEN/NOTIFY → dikkat.
  
  Session-mode: session boyunca aynı bağlantı → az tasarruf ama tam uyumluluk.

Çözüm 3: max_connections artır
  postgresql.conf: max_connections = 500
  Ama: daha fazla bellek ve context switch overhead → dikkatli artır.
```

---

### Mülakat Soruları — Connection Pooling

**Junior / Mid:**

10. Connection pool neden gereklidir? HikariCP pool size nasıl belirlenir?

    > **Beklened:** Her DB bağlantısı TCP + auth → 50-100ms overhead. Pool: N bağlantıyı hazır tutar, thread alır-kullanır-bırakır. Olmadan: her request yeni bağlantı → yavaş ve DB aşırı yük. Pool size formülü (PostgreSQL wiki): `(core_count × 2) + disk_spindle_sayısı`. SSD 4-core: (4×2)+1 = 9. Mantık: thread IO beklerken başka thread çalışır → CPU overclock. Çok büyük pool: DB'de bağlantı yarışması → context switch → daha yavaş. Kural: küçük pool + fast query > büyük pool + slow query.

11. PgBouncer nedir? Transaction mode ile session mode farkı?

    > **Beklened:** PgBouncer: PostgreSQL önünde connection proxy. Uygulama 1000 bağlantı açar, PostgreSQL 20 görür. Session mode: her uygulama bağlantısı → bir PostgreSQL bağlantısı, oturum boyunca. Transaction mode: her transaction → bir PostgreSQL bağlantısı, transaction bitince bağlantı pool'a döner. Transaction mode çok daha verimli ama kısıtlar: prepared statement, advisory lock, `SET LOCAL`, `LISTEN/NOTIFY` transaction bazında çalışmaz. Spring + JPA: transaction mode genellikle uyumlu, ama dikkat edilmesi gereken edge case'ler var.

---

## Bölüm 4: Redis Internals

### Gerçek Hayat Sorunları

---

**Sorun 9: Persistence yok — Redis restart'ta tüm veri kayboldu**

```
Senaryo:
  Redis: session store olarak kullanılıyor.
  Persistence: kapalı (varsayılan ayar, sadece in-memory).
  Redis server patched + restart → tüm session'lar silindi.
  Tüm kullanıcılar "oturumunuz sonlandı" aldı → destek hattı patladı.

Redis default: persistence YOK.
  Restart → boş Redis.

Düzeltme:
  redis.conf:
    # RDB (snapshot):
    save 900 1       # 900sn içinde 1 değişiklik varsa kaydet
    save 300 10      # 300sn içinde 10 değişiklik varsa kaydet
    save 60 10000    # 60sn içinde 10000 değişiklik varsa kaydet
    
    # AOF (append-only file, daha güvenli):
    appendonly yes
    appendfsync everysec   # her saniye sync (önerilen denge)
    
    # Hybrid (en iyi):
    aof-use-rdb-preamble yes  # hızlı restart (RDB) + az veri kaybı (AOF)

Seçim:
  Session/cache: veri kaybı tolere edilebilir → RDB yeterli
  Rate limiting, lock: kısa TTL → RDB yeterli
  Financial record: AOF everysec (max 1sn veri kaybı)
  Sıfır veri kaybı: AOF always (her write fsync → yavaş, SLA gerekirse)
```

---

**Sorun 10: Redis Sentinel failover — uygulama eski master'a bağlı kaldı**

```
Senaryo:
  Redis Sentinel: 1 master, 2 slave, 3 sentinel.
  Master çöktü → Sentinel slave'i promote etti (30sn sürdü).
  Uygulama: eski master IP'yi hardcode → yeni master'a bağlanamadı.
  30 + X dakika: cache çalışmıyor → DB'ye tüm yük → yavaşlama.

Sorun: Client, Sentinel'dan yeni master adresini almıyor.

Düzeltme (Spring/Lettuce):
  spring:
    data:
      redis:
        sentinel:
          master: mymaster
          nodes:
            - sentinel1:26379
            - sentinel2:26379
            - sentinel3:26379
  
  # Uygulama Sentinel'e bağlanır, "mymaster" için gerçek host/port sorar.
  # Failover olunca Sentinel yeni adresi bildirir → Lettuce otomatik reconnect.
  # Hardcode IP → asla!

Süreç:
  Master down → Sentinels quorum (2/3) → promote → CLIENT FAILOVER bildir
  → Spring Boot yeni master'a bağlanır (birkaç sn kesinti kabul edilebilir)
```

---

**Sorun 11: Redis Cluster cross-slot MGET hatası**

```java
// Senaryo: Kullanıcı sepetindeki tüm ürünleri Redis'ten tek seferde çek.
List<String> keys = cartItems.stream()
    .map(item -> "product:" + item.getProductId())
    .collect(toList());

List<Object> products = redisTemplate.opsForValue().multiGet(keys);
// HATA: CROSSSLOT Keys in request don't hash to the same slot

// Neden: Redis Cluster → 16384 hash slot
// "product:1" → slot 3422 → Master 1
// "product:2" → slot 9188 → Master 2
// MGET: tüm key aynı shard'da olmalı → olmadı → hata

// ÇÖZÜM 1: Hash tag — {} içindeki kısım hash'lenir
keys = cartItems.stream()
    .map(item -> "{product}:" + item.getProductId())  // hepsi aynı slot
    .collect(toList());
// "{product}" → aynı slot → MGET çalışır
// Dezavantaj: tüm product key'leri tek shard → hotspot riski

// ÇÖZÜM 2: Pipeline ile ayrı GET'ler (cross-slot yok)
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) conn -> {
    for (String key : keys) {
        conn.get(key.getBytes());
    }
    return null;
});
// Pipeline: N komutu tek round-trip → cross-slot sorunu yok, ama atomic değil

// ÇÖZÜM 3: Cluster client smart routing
// Lettuce/Jedis Cluster: her key'i doğru shard'a yönlendirir
// ama MGET atomic değil — birden fazla shard'a paralel gönderir
```

---

### Mülakat Soruları — Redis

**Junior / Mid:**

12. Redis neden hızlıdır? Tek thread olması dezavantaj değil midir?

    > **Beklened:** Hızlı nedenleri: (1) In-memory — disk I/O yok. (2) Basit veri yapıları — B-tree yok, hash table/linked list. (3) Single-threaded command loop — lock yok, context switch yok. (4) Non-blocking I/O (epoll) — binlerce client, tek thread. Single thread: CPU bottleneck mu? Komutlar çok hızlı (microsecond) → throughput yüksek. CPU sınırına ulaşınca: Redis 6+ I/O threading (komutlar hâlâ single thread, I/O paralel). Bottleneck genellikle CPU değil, network bandwidth.

13. Redis RDB ve AOF arasında nasıl karar verirsin?

    > **Beklened:** RDB: snapshot, belirli anlarda tüm veri yazılır. Hızlı restart, compact dosya. Dezavantaj: son snapshot'tan bu yana veri kaybı (1-15 dakika). AOF: her komut log'a yazılır. always: sıfır veri kaybı ama yavaş. everysec: max 1sn kayıp, dengeli. no: OS'e bırak, tehlikeli. Hybrid: AOF + RDB preamble → hızlı restart + az kayıp. Kullanım: cache/session → RDB yeterli. Birincil veri store (session persistent) → AOF everysec. Kritik finansal → AOF always + multi-node replica.

---

**Senior / Architect:**

14. Redis Cluster vs Sentinel: hangisini ne zaman seçersin?

    > **Beklened:** Sentinel: single shard HA. Tek master'ın fail-over'ı otomatik. Basit, küçük veri seti. Sınır: tek node'un RAM'i kadar veri. Cluster: horizontal sharding — 3+ master, veriler bölünür. Sınırsız ölçekleme (teorik). Cross-slot kısıtı. Karar: Veri < RAM ama HA lazım → Sentinel. Veri > RAM veya write throughput → Cluster. K8s'de: Redis Operator ile her ikisi yönetilebilir. Dikkat: Cluster'da Lua script, transaction, MGET → cross-slot olmayan key'lerle çalışır.

15. Cache stampede (thundering herd) nedir? Redis ile nasıl önlenir?

    > **Beklened:** Stampede: popüler key expire oldu → aynı anda N istek cache miss → N istek DB'ye gidiyor → DB'yi ezdi. Önlem: (1) Mutex lock: ilk thread DB'ye gider, diğerleri bekler → Redis SET NX ile "regenerating" flag. (2) Cache-aside + jitter: TTL = base + random(0, jitter) → tüm key'ler aynı anda expire olmuyor. (3) Probabilistic early expiration: TTL dolmadan, kalan süreye bağlı ihtimalle yenile (beta algoritması). (4) Caffeine LoadingCache: loader mutex built-in. (5) Read-through cache: cache miss → otomatik DB'den yükler, diğerleri bekler.

---

## Bölüm 5: Elasticsearch

### Gerçek Hayat Sorunları

---

**Sorun 12: Çok fazla shard — arama yavaş, bellek tükendi**

```
Senaryo:
  Her gün yeni index: logs-2024-01-01, logs-2024-01-02...
  Her index: 5 shard (varsayılan).
  1 yıl: 365 × 5 = 1825 shard.
  Elasticsearch: her shard = JVM heap tüketimi.
  Heap doldu → GC storm → node out of memory → cluster kırmızı.

Neden çok shard kötü:
  Arama: coordinator → tüm shard'lara fan-out → sonuçları merge.
  1825 shard: 1825 thread / network call → overhead.
  "Shard = mini Lucene index" → her biri bellek kullanır.

Kural: 1 shard ~= 20-50GB veri (Elastic önerisi).
  Shard sayısı = toplam veri / 30GB

Düzeltme:
  ILM (Index Lifecycle Management):
    Hot: son 7 gün → 1 shard, 1 replica
    Warm: 7-30 gün → force-merge (segment azalt), replica kaldır
    Cold: 30-365 gün → frozen (disk, bellek az)
    Delete: 365+ gün → sil

  Rollover + ILM:
    PUT /_ilm/policy/logs-policy → phase tanımları
    Index alias: logs-current → güncel index'e işaret
    Rollover: 50GB veya 7 gün → yeni index aç
```

---

**Sorun 13: Dynamic mapping explosion — beklenmedik alan → heap doldu**

```
Senaryo:
  Log indexleme: kullanıcı log objesi gönderdi.
  Kötü log: `{"error": {"code": 500, "details": {...nested 50 level...}}}`
  Dynamic mapping: her alan otomatik mapping'e eklendi.
  Yüzlerce unique alan → mapping büyüdü → heap'e yüklendi.
  "Limit of total fields [1000] has been exceeded" hatası.

Elasticsearch dynamic mapping tehlikesi:
  Her yeni alan → master node'da cluster state güncellenir.
  Çok alan → cluster state büyür → tüm node'lara dağıtılır → bellek.

Düzeltme:
  # Strict mapping: bilinmeyen alan → hata
  PUT /logs { "mappings": { "dynamic": "strict", "properties": { ... } } }
  
  # Sadece bilinen alanları index'le, bilinmeyenleri yoksay:
  { "dynamic": false }
  
  # Total field limit artır (dikkatli):
  PUT /logs/_settings { "index.mapping.total_fields.limit": 2000 }
  
  # Nested JSON'u string olarak sakla, sadece önemli alanları index'le:
  "message": { "type": "text" }
  "raw_json": { "type": "object", "enabled": false }  # index'lenmiyor, sadece saklıyor
```

---

**Sorun 14: Disk doldu — index read-only oldu, yazma durdu**

```
Senaryo:
  Elasticsearch disk %85 doldu → flood watermark tetiklendi → index read-only.
  Yeni log yazılamıyor, uygulama "ClusterBlockException" alıyor.

Elasticsearch disk watermark:
  cluster.routing.allocation.disk.watermark.low: 85%  → uyarı, shard taşıma
  cluster.routing.allocation.disk.watermark.high: 90% → yeni shard atama durdur
  cluster.routing.allocation.disk.watermark.flood_stage: 95% → READ ONLY

Acil düzeltme:
  # Disk temizle → sonra block'u kaldır:
  curl -X PUT "localhost:9200/_all/_settings" \
    -H 'Content-Type: application/json' \
    -d '{"index.blocks.read_only_allow_delete": null}'

Kalıcı çözüm:
  1. ILM ile eski index'leri sil/archive.
  2. Curator: belirli günden eski index sil.
  3. Hot-warm-cold: cold tier → object storage (S3).
  4. Monitoring: disk %75'te alert (flood'dan önce).
  5. Elasticsearch 8.x: autoscaling → disk dolunca otomatik node ekle.
```

---

### Mülakat Soruları — Elasticsearch

**Junior / Mid:**

16. Inverted index nasıl çalışır? Neden full-text search için idealdir?

    > **Beklened:** Klasik index: doküman → kelimeler. Inverted: kelime → dokümanlar. "quick brown fox" → quick:[doc1], brown:[doc1,doc2], fox:[doc1]. Arama "brown fox": brown seti ∩ fox seti → {doc1}. Hızlı çünkü: intersection → küçük set, disk'te sıralı → binary search. PostgreSQL full-text: GIN index = aynı mantık ama Elasticsearch: relevance scoring, fuzzy, aggregation, distributed. ne zaman: basit search + mevcut PostgreSQL → pg_trgm + GIN. Kompleks search + büyük veri → Elasticsearch.

17. Elasticsearch'ü primary datastore olarak kullanmak neden tehlikeli?

    > **Beklened:** (1) Eventual consistent: yazma → primary shard → async replication → replica. Okuyucun farklı replica'ya gitmesi → stale data. (2) Veri kaybı riski: single shard + disk arızası → segment kaybolabilir (replica yoksa). (3) ACID yok: multi-document atomic işlem desteklenmez. (4) Mapping değişikliği: reindex gerektirir (downtime veya alias trick). (5) Disk dolunca read-only. Doğru kullanım: PostgreSQL primary + Elasticsearch search index (Debezium CDC ile sync). İkili ama güvenli.

---

**Senior / Architect:**

18. Elasticsearch shard sayısını nasıl belirlersin? Overcounting ve undercounting etkileri neler?

    > **Beklened:** Kural: 1 shard ~20-50GB veri, günlük/haftalık veri tahminiyle hesapla. Too many shards: fan-out overhead — coordinator N shard'a sorgu → N cevap merge → latency. Her shard JVM heap tüketir (~50MB/shard minimum). Too few shards: tek shard → CPU bottleneck, shard büyüyünce reindex gerekir (shard sayısı sonradan değiştirilemez). Strateji: data stream + ILM → rollover + shrink. Yüksek QPS index: daha fazla shard (arama paralel). Log analiz (bulk write, nadir search): az shard. Replica: read scaling için artır, write performansı için azalt.

---

## Karma — Architect Seviyesi

19. **"Bir e-ticaret sisteminde PostgreSQL yavaş sorgular artmaya başladı. Nasıl yaklaşırsın?"**

    > **Beklened:** Teşhis adımları: (1) `pg_stat_statements`: en çok süre alan top-N sorgu. `SELECT query, total_exec_time, calls, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;` (2) Slow query log: `log_min_duration_statement = 1000` (1sn+ logla). (3) `EXPLAIN ANALYZE` problematik sorgu → Seq Scan mı? row estimate vs actual büyük fark mı? (4) `pg_stat_user_tables`: dead tuple oranı → VACUUM gerekli mi? (5) `pg_stat_activity`: uzun süren transaction var mı, lock bekleme var mı? (6) `pg_locks + pg_stat_activity` join → deadlock, lock queue. Çözüm sıralı: index ekle → sorgu optimize et → VACUUM tune → connection pool → read replica → partitioning.

20. **"Redis'i hem cache hem de session store olarak kullanıyorsun. Birinin performans sorunu diğerini nasıl etkiler? Nasıl izole edersin?"**

    > **Beklened:** Redis single-threaded: büyük key okuma/yazma, KEYS * gibi pahalı komut → tüm operasyonlar bloke. Cache büyük value → session okuması gecikir. Çözüm: (1) Ayrı Redis instance: session Redis + cache Redis. Bellek, CPU bağımsız. Failover politikası farklı (session: persistence, cache: yok). (2) `redis-namespace`: aynı Redis ama key prefix ile mantıksal ayrım — izolasyon sağlamaz. (3) Cluster: farklı shard — kısmi izolasyon. Monitoring: `slowlog get` → yavaş komut tespiti. OBJECT ENCODING: büyük list/hash → ziplist → listpack arası otomatik geçiş dikkat et. En iyi: ayrı instance, ayrı bellek limiti, ayrı eviction policy (session: noeviction, cache: allkeys-lru).

21. **"Bir microservice mimarisinde hangi servis hangi DB'yi kullanmalı? Örnek bir senaryo üzerinden anlat."**

    > **Beklened:** Örnek e-ticaret: Order Service → PostgreSQL (ACID, transaction kritik). Inventory Service → PostgreSQL (stok consistency). Product Catalog Service → MongoDB (esnek özellik, category bazlı şema). Search Service → Elasticsearch (full-text, facet, relevance). Session/Auth → Redis (low latency, TTL). Analytics/Reporting → ClickHouse veya BigQuery (OLAP, columnar). Genel prensipler: (1) DB per service: her servis kendi DB'si, doğrudan erişim yok. (2) Consistency kritikse → SQL. (3) Scale > consistency → NoSQL. (4) Sorgu patternı belirler: aggregation → columnar, document → MongoDB. Tuzak: her şeyi PostgreSQL'e koyma → servisler arası coupling (paylaşılan tablo). Tuzak: her şeyi MongoDB'ye koyma → ACID sorunları.

22. **"Elasticsearch arama latency'si artmaya başladı. Debug planın ne?"**

    > **Beklened:** (1) Cluster health: `GET /_cluster/health` → yellow/red mu? Atanmamış shard var mı? (2) Node stats: `GET /_nodes/stats` → heap kullanımı, GC süresi. Heap %85+ → GC pause → latency spike. (3) Index stats: `GET /index/_stats` → segment sayısı, merge. Çok segment → search yavaş. Force merge (`_forcemerge`) → segment birleştir. (4) Slow log: `index.search.slowlog.threshold.query.warn: 1s` → 1sn+ sorgular. (5) Shard büyüklüğü: çok büyük shard → tek shard'da yavaş. (6) Query analizi: `_profile` API → her query/aggregation adımının süresi. (7) Caching: `fielddata_cache`, `request_cache` → hit rate düşükse. Çözüm: GC → heap artır / JVM tuning. Segment → force merge. Sorgu → aggregation filter önce, fetch az alan.
