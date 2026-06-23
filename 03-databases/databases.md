# 03 — Veritabanları & Depolama

## 1. SQL vs NoSQL

### Ne Zaman SQL?

- **ACID garantisi** kritikse (finans, sipariş, stok)
- **İlişkisel veri** — foreign key, join gerekiyorsa
- **Şema tutarlılığı** önemliyse
- **Karmaşık sorgular** — aggregate, window functions, subquery

### Ne Zaman NoSQL?

| NoSQL Türü | Örnek | Ne zaman? |
|------------|-------|-----------|
| Document | MongoDB, Couchbase | Esnek şema, hiyerarşik veri, katalog |
| Key-Value | Redis, DynamoDB | Cache, session, sıralı hesap |
| Wide-Column | Cassandra, HBase | Zaman serisi, IoT, yüksek yazma |
| Graph | Neo4j, Amazon Neptune | Sosyal ağ, öneri motoru, fraud |
| Search | Elasticsearch | Full-text search, log analizi |

---

## 2. ACID vs BASE

**ACID** (SQL, güçlü tutarlılık):
- **Atomicity** — ya hepsi ya hiçbiri
- **Consistency** — geçerli durumdan geçerli duruma
- **Isolation** — eş zamanlı işlemler birbirini etkilemez
- **Durability** — commit sonrası kalıcıdır

**BASE** (NoSQL, yüksek erişilebilirlik):
- **Basically Available** — her zaman yanıt verir
- **Soft state** — zaman içinde değişebilir
- **Eventually consistent** — sonunda tutarlı olur

```
ACID: Bank transfer — tutarlılık kritik, kısa gecikme kabul edilebilir
BASE: Facebook like count — kısa süre yanlış göstermek kabul edilebilir, high availability kritik
```

---

## 3. PostgreSQL Internals

### MVCC (Multi-Version Concurrency Control)

**Problem:** Read ve Write işlemleri birbirini block etmesin.

**Çözüm:** Her satırın birden fazla versiyonu saklanır.

```
Row versioning:
xmin = bu versiyonu yaratan transaction ID
xmax = bu versiyonu silen/güncelleyen transaction ID (0 ise aktif)

UPDATE users SET name='Ali' WHERE id=1:
1. Eski satır: xmin=100, xmax=200 (artık "silinmiş")
2. Yeni satır: xmin=200, xmax=0  (aktif)

Eş zamanlı okuma:
Transaction 150 → xmin <= 150 ve xmax = 0 veya xmax > 150 → eski satırı görür
Transaction 250 → xmin <= 250 ve xmax = 0 → yeni satırı görür
```

**Dead tuple problemi:** Eski versiyonlar disk'te kalır → `VACUUM` temizler.  
**AUTOVACUUM:** PostgreSQL otomatik çalıştırır. Çok yazma olan tablolarda tuning gerekir.

### WAL (Write-Ahead Log)

```
Crash recovery için:
1. Değişiklik önce WAL (sequential write) → sonra data file
2. Crash sonrası: WAL'den replay et → tutarlı duruma gel

Replikasyon için:
Primary → WAL stream → Standby (WAL replay)
```

**WAL avantajı:** Sequential write (hızlı) + crash recovery  
**pg_wal dosyaları:** `/var/lib/postgresql/data/pg_wal/` — dolmasın dikkat

### Index Türleri

**B-tree (varsayılan):**
```
Yapı: Balanced tree, O(log n) arama
WHERE koşullar: =, <, >, BETWEEN, LIKE 'prefix%'
Neredeyse her durumda: ✓

CREATE INDEX idx_users_email ON users(email);
```

```
B-tree görsel:
             [50]
           /       \
       [25]          [75]
      /    \        /    \
  [10,20] [30,40] [60,70] [80,90]
```

**Hash Index:**
```
O(1) lookup, sadece = için
Büyük veri setinde B-tree'den hızlı olabilir
WHERE user_id = 12345 → Hash index ideal
Dezavantaj: range query yok, WAL desteği eksikti (düzeldi Pg 10+)
```

**GIN (Generalized Inverted Index):**
```
Composite değerler için: array, jsonb, full-text search
@> (contains), && (overlap), @@ (text search)

CREATE INDEX idx_tags ON articles USING GIN(tags);
WHERE tags @> ARRAY['postgresql', 'index']
```

**BRIN (Block Range Index):**
```
Tablonun fiziksel sırasıyla korelasyonu yüksek veriler için
(timestamp, serial id gibi insert sırasına göre artan kolonlar)
Çok küçük index, büyük tablolarda etkili
Tablonun "blokları" hakkında min/max tutar

WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
→ Sadece ilgili blok aralığını tarar
```

**GiST (Generalized Search Tree):**
```
Geometrik veri, IP adresi, tam metin
PostGIS lokasyon sorguları için
```

### Index Seçim Kılavuzu

```
Eşitlik sorgusu (=):
  Low cardinality (cinsiyet, durum) → Index faydasız, full scan daha iyi
  High cardinality (email, UUID)    → B-tree

Range sorgusu (<, >, BETWEEN):
  → B-tree

Full-text search:
  → GIN with to_tsvector

JSON/Array içerik:
  → GIN

Büyük tablo, tarih kolonu, insert sırasında artan:
  → BRIN

Composite index sırası önemli:
  (a, b, c) index → a ile başlayan sorgularda kullanılır
  WHERE b=1 → bu index KULLANILMAZ (sol prefix kuralı)
  WHERE a=1 AND b=2 → kullanılır
```

### Query Planner

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- Çıktı:
Seq Scan on orders  (cost=0.00..2500.00 rows=1 width=100) actual time=0.042..15.234
  Filter: (customer_id = 123)
  Rows Removed by Filter: 99999
Planning Time: 0.5 ms
Execution Time: 15.3 ms
```

**Planner nasıl karar verir?**
- Tablo istatistiklerini kullanır (`pg_statistic`, `ANALYZE` ile güncellenir)
- Satır sayısı, data distribution, index seçicilik (selectivity)
- Sequential scan cost vs Index scan cost hesaplar

**`Seq Scan` görünce ne yapmalı?**
```
1. Index var mı? → EXPLAIN çıktısında "Index Scan" görmüyorsan
2. Selectivity düşük mü? → WHERE status='active' ve %90 active ise index yok sayılır
3. İstatistikler güncel mi? → ANALYZE tabloname
4. enable_seqscan=off → zorla index kullan (test için, production'da yapma)
```

---

## 4. Connection Pooling (HikariCP)

**Neden connection pool?**  
Her DB connection → TCP handshake + authentication → 50-100ms maliyeti var.  
Pool: N bağlantıyı hazır tutar, thread alır kullanır bırakır.

```
Thread 1 ──┐
Thread 2 ──┤ → HikariCP Pool [conn1, conn2, conn3] → PostgreSQL
Thread 3 ──┘
Thread 4 → bekler (pool dolu)
```

**Pool boyutu formülü (PostgreSQL wiki):**
```
pool_size = (core_count * 2) + number_of_spindles

SSD sunucu, 4 core → (4*2) + 1 = 9
```

**Dikkat:** Uygulama pool size × uygulama instance sayısı = toplam DB connection.  
100 instance × 10 pool = 1000 connection → PostgreSQL için çok fazla olabilir.  
Çözüm: **PgBouncer** (connection proxy, transaction-mode pooling).

---

## 5. Redis Internals

### Veri Yapıları ve Kullanım Alanları

```
String:  SET key value / GET key
→ Cache, counter, distributed lock, rate limiting

List:    RPUSH key value / LPOP key
→ Queue, recent items, timeline

Hash:    HSET key field value / HGET key field
→ Nesne cache (user:123 → {name: Ali, email: ...})

Set:     SADD key member / SMEMBERS key
→ Unique visitor, tag, friend list

Sorted Set: ZADD key score member / ZRANGEBYSCORE
→ Leaderboard, rate limiting (sliding window), job scheduling

Stream:  XADD stream-key * field value
→ Kafka-lite, event log

Bitmap:  SETBIT / GETBIT
→ Feature flags, user activity tracking (304 gün × 100M user = ~3.5GB)

HyperLogLog: PFADD / PFCOUNT
→ Unique count estimation (hata payı ~0.81%, çok az bellek)
```

### Persistence

**RDB (Redis Database Snapshot):**
```
Belirli aralıklarla tüm veriyi disk'e yazar
save 900 1     # 900 saniye içinde 1+ değişiklik varsa
save 300 10    # 300 saniyede 10+ değişiklik varsa

Avantaj: Compact, hızlı recovery
Dezavantaj: Son snapshot'tan bu yana veri kaybı
```

**AOF (Append-Only File):**
```
Her write komutu log'a append edilir
appendfsync always    # Her write → fsync (yavaş, güvenli)
appendfsync everysec  # Her saniye → fsync (orta, önerilen)
appendfsync no        # OS'e bırak (hızlı, tehlikeli)

Avantaj: Neredeyse sıfır veri kaybı
Dezavantaj: Büyük dosya, yavaş restart
AOF rewrite: Dosya büyüyünce compaction (BGREWRITEAOF)
```

**Hybrid:** RDB + AOF birlikte → hızlı restart + güvenli recovery.

### Cluster vs Sentinel

**Sentinel (High Availability):**
```
Master → Slave (replika)
Sentinel izler → Master çökerse → Slave'i promote → Client'lara bildirir

Kullanım: HA, automatic failover
Sınır: Tek shard, tek node'un memory'si kadar veri
```

**Cluster (Sharding):**
```
16384 hash slot → N master'a bölünür
key → CRC16(key) % 16384 → hangi slot/master?

3 master, 3 slave:
Master 1: slot 0-5460    + Slave 1
Master 2: slot 5461-10922 + Slave 2
Master 3: slot 10923-16383 + Slave 3

Avantaj: Yatay ölçekleme
Dezavantaj: MGET/transaction cross-slot çalışmaz (hashtag ile çöz: {user}.123)
```

---

## 6. Elasticsearch

### Inverted Index

```
Klasik index: document → words
  Doc 1: "quick brown fox"
  Doc 2: "lazy brown dog"

Inverted index: word → documents
  brown → [Doc 1, Doc 2]
  fox   → [Doc 1]
  quick → [Doc 1]
  lazy  → [Doc 2]
  dog   → [Doc 2]

"brown fox" araması:
  brown → {Doc1, Doc2}
  fox   → {Doc1}
  intersection → {Doc1} → hızlı!
```

### Sharding ve Relevance

```
Index: products
├── Shard 0 (primary) + Shard 0 (replica)
├── Shard 1 (primary) + Shard 1 (replica)
└── Shard 2 (primary) + Shard 2 (replica)

Shard sayısı sabit, baştan doğru seç (değiştirmek = reindex)
Replica = HA + read scaling
```

**TF-IDF Relevance Scoring:**
```
TF (Term Frequency): Terim belgede ne kadar sık geçiyor?
IDF (Inverse Document Frequency): Terim kaç belgede geçiyor?
  → Çok belgede geçen kelimeler (the, a) düşük IDF

Score = TF × IDF

Elasticsearch BM25 kullanır (TF-IDF'nin geliştirilmiş versiyonu):
- Uzun belgeleri penalize eder (term sıklığının getirisi azalır)
```

### Architect Kararları

```
Ne zaman Elasticsearch?
✓ Full-text search (product catalog, article search)
✓ Log analizi (ELK stack)
✓ Aggregation & analytics
✓ Fuzzy/typo tolerant search

Ne zaman kullanma?
✗ Primary datastore (eventual consistent, veri kaybı riski)
✗ ACID gerektiren işlemler
✗ Karmaşık join sorguları
✗ Çok az veri (PostgreSQL full-text yeter)
```

---

## 7. Architect Checklist: Veritabanı Kararları

```
□ Primary datastore seçimi: SQL mi NoSQL mi? Neden?
□ Sharding stratejisi gerekli mi? (>TB data, >10K write/s)
□ Read replica var mı? (read yükü dağıtımı)
□ Connection pooling konfigürasyonu doğru mu?
□ Index'ler query pattern'larını karşılıyor mu?
□ VACUUM / ANALYZE scheduled mi?
□ Backup stratejisi: RTO ve RPO tanımlandı mı?
□ Slow query monitoring (pg_stat_statements, slow_query_log)
□ N+1 sorunu için ORM audit
□ Database migration stratejisi (Flyway/Liquibase)
```
