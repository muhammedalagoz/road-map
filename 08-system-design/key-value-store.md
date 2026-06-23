# 08s — System Design: Distributed Key-Value Store (Redis/DynamoDB Tasarımı)

## Gereksinimler

```
Functional:
  ✓ put(key, value)
  ✓ get(key) → value
  ✓ delete(key)
  ✓ TTL (expiration)
  ✗ Transaction, secondary index (out of scope)

Non-Functional:
  Ölçek: 100 TB veri, 1M node
  Latency: < 10ms (P99)
  Availability: 99.99%
  Consistency: Tunable (strong veya eventual)
  Partition tolerance: zorunlu
```

---

## Tek Node KV Store

```
Temel yapı:
  HashMap<String, byte[]> store
  Sorun: tek node → sınırlı kapasite, SPOF

Disk persistence:
  WAL (Write-Ahead Log): her write → log'a yaz → memory güncelle
  Crash recovery: WAL'ı replay et → state'i geri yükle

Compaction:
  WAL büyüdükçe → snapshot al → eski WAL sil
  LSM-Tree (Log-Structured Merge Tree): RocksDB kullanır
  B-Tree: PostgreSQL, MySQL kullanır
```

---

## Distributed KV Store

### Veriyi Dağıtma: Consistent Hashing

```
Hash Ring:
  Tüm hash space: 0 → 2^32 (sanal ring)
  Her node: birden fazla "virtual node" (vnodes) üzerinde

  hash("node-A") → 30°, 90°, 210° (3 vnode)
  hash("node-B") → 60°, 150°, 270°
  hash("node-C") → 120°, 180°, 330°

Key routing:
  hash("user:123") → 145° → saat yönünde ilk node → node-B

Node ekleme:
  hash("node-D") → 75°, 160°, 320°
  → Sadece komşu node'lardan veri taşı (minimum disruption)

VNode avantajı:
  Her node birden fazla segment → uniform dağılım
  Node çöküşünde yük birden fazla node'a yayılır (hot spot yok)

Sorun: Hash skew (çok fazla key aynı yere giderse)
  Çözüm: Daha fazla vnode (150-200 vnode per node)
```

---

## Replication

```
Replication Factor = N (genellikle 3):
  key → hash → coordinator node
  Coordinator → N node'a kopyala (ring'de ardışık N node)

Senkron vs Asenkron:
  Senkron (N=3, W=3): 3 node da yazmadan OK deme → yavaş
  Asenkron (N=3, W=1): 1 node yazınca OK → hızlı, veri kaybı riski

Quorum:
  N=3, W=2, R=2 → W+R>N → strong consistency
  N=3, W=1, R=1 → eventual consistency → max performans

Hinted Handoff:
  Node-B down → Node-D "hint" olarak alır
  Node-B geri gelince → Node-D'den transfer
  Geçici down tolerans
```

---

## Okuma/Yazma Koordinasyonu

```
Write akışı:
  Client → Coordinator (hash ring'den key'in yeri)
  Coordinator → N replica'ya async yaz
  W quorum yanıt verince → Client'a OK

Read akışı:
  Client → Coordinator
  Coordinator → R replica'dan oku
  En yüksek version (vector clock) değeri → Client'a döndür
  Stale replica → read repair ile güncelle

Read Repair:
  R=2, 2 node'dan oku:
    Node-A: value="v2" (güncel)
    Node-B: value="v1" (eski)
  Coordinator: "v2" döndür + Node-B'yi async güncelle (read repair)
```

---

## Conflict Resolution

```
Concurrent writes (network partition sırasında):
  Node-A: SET user:123 → "Ali" (timestamp 100)
  Node-B: SET user:123 → "Ayşe" (timestamp 101)
  Partition iyileşince: hangisi kazandı?

Last-Write-Wins (LWW):
  Timestamp yüksek olan kazanır → "Ayşe"
  Sorun: Clock skew (node saatleri eşit değil) → yanlış sonuç

Vector Clock (DynamoDB benzeri):
  Her yazma → {nodeId, counter} ile işaretlenir
  [Node-A:1] → [Node-A:2] → kausal sıra
  Concurrent: [Node-A:1, Node-B:1] vs [Node-A:1, Node-C:1] → conflict!
  Conflict resolution → application'a bırak veya merge

CRDT (Conflict-free Replicated Data Types):
  Counter: her node kendi sayacını artırır → toplam SUM
  Set: union (eklemeler birleşir, silme = "tombstone" ekle)
  Register: LWW-Register (timestamp ile)
```

---

## Storage Engine

### LSM-Tree (Log-Structured Merge Tree)

```
RocksDB, Cassandra, LevelDB kullanır

Yazma yolu:
  Write → WAL (crash recovery) → MemTable (in-memory, sorted)
  MemTable dolar → SSTable (Sorted String Table, immutable, disk'e flush)

Okuma yolu:
  MemTable'a bak → SSTable L0 → L1 → L2 → ...
  Bloom Filter: "Bu SSTable'da bu key var mı?" → O(1) yanlış negatif yok

Compaction:
  Zamanla SSTable'lar birleştirilir (merge sort)
  Silinen/güncellenen key'ler temizlenir
  L0 → L1 → L2 (boyutları büyür, tekrar azalır)

Avantaj:
  ✓ Write çok hızlı (sequential write, WAL + MemTable)
  ✓ Zaman içinde okuma hızlanır (compaction)

Dezavantaj:
  ✗ Read biraz yavaş (birden fazla level)
  ✗ Compaction I/O yükü (write amplification)
  ✗ Space amplification (compaction sırasında 2x disk)
```

### B-Tree

```
PostgreSQL, MySQL, etcd kullanır

In-place update: key → node'u bul → value update

Avantaj:
  ✓ Read çok hızlı (O(log n))
  ✓ Range query iyi
  
Dezavantaj:
  ✗ Random write (SSD için daha az sorun)
  ✗ Fragmentation (page split)
```

---

## TTL (Time to Live)

```
Lazy expiration:
  Get key → expired mı? → sil, None döndür
  Avantaj: overhead yok
  Dezavantaj: Expired key'ler disk/memory tutar

Active expiration:
  Background thread → expired key'leri tarar → sil
  Redis: her 100ms → 20 random key → expired mı? → %25'den fazlası expired → tekrar

Priority queue ile TTL:
  MinHeap: {expiresAt, key}
  Background: heap.peek().expiresAt <= now → sil → tekrar

Storage level TTL (RocksDB):
  Bloom filter + TTL aware compaction
  Compaction sırasında expired key'ler otomatik silinir
```

---

## Cache Eviction Politikaları

```
LRU (Least Recently Used) — en yaygın:
  Doubly linked list + HashMap
  Get → head'e taşı (most recently used)
  Kapasite dolunca → tail'den sil (least recently used)
  O(1) get/put

LFU (Least Frequently Used):
  En az kullanılan silinir
  Avantaj: gerçekten "hot" data korunur
  Dezavantaj: Yeni eklenen (henüz kullanılmamış) hemen silinebilir
  Counter aging: zamanla count azaltılır (scan resistance)

ARC (Adaptive Replacement Cache):
  LRU + LFU kombinasyonu
  T1: yakın zamanda erişilen, T2: tekrar erişilen
  Dinamik denge → Oracle DB kullanır

Redis eviction:
  allkeys-lru   → tüm keylerden LRU sil
  allkeys-lfu   → tüm keylerden LFU sil
  volatile-lru  → TTL'li keylerden LRU sil
  noeviction    → hata fırlat (cache doldu)
```

---

## API Tasarımı

```java
interface KeyValueStore {
    void put(String key, byte[] value, Duration ttl);
    Optional<byte[]> get(String key);
    void delete(String key);
    boolean exists(String key);
    
    // Batch operations (network round-trip azalt)
    void multiPut(Map<String, byte[]> entries);
    Map<String, byte[]> multiGet(List<String> keys);
    
    // Conditional operations (optimistic locking)
    boolean putIfAbsent(String key, byte[] value);
    boolean compareAndSwap(String key, byte[] expected, byte[] newValue);
}

// Distributed coordination için:
interface DistributedCounter {
    long increment(String key);
    long incrementBy(String key, long delta);
    long get(String key);
}
```

---

## Trade-off Özeti

| Karar | CP (Strong) | AP (Eventual) | Seçim |
|-------|-------------|---------------|-------|
| Consistency | W+R>N quorum | W=1 R=1 | Business gereksinim |
| Conflict | LWW | Vector Clock | Accuracy vs simplicity |
| Storage | B-Tree (read) | LSM (write) | Workload (read/write ratio) |
| Partition | Shard | Ring | Ring (dynamic membership) |
| Replication | Synchronous | Asynchronous | Latency vs durability |

| Sistem | Consistency | Latency | Use Case |
|--------|-------------|---------|----------|
| Redis | Eventual (default) | <1ms | Cache, session |
| DynamoDB | Tunable | 1-10ms | General KV |
| etcd | Strong (Raft) | 5-20ms | Coordination, config |
| Cassandra | Tunable | 1-10ms | Time-series, IoT |
