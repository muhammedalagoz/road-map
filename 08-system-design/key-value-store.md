# 08s — System Design: Distributed Key-Value Store (Redis/DynamoDB Tasarımı)

## Gereksinimler

```
Functional:
  ✓ put(key, value, ttl?)
  ✓ get(key) → value
  ✓ delete(key)
  ✓ TTL (expiration)
  ✓ Conditional put (putIfAbsent, compareAndSwap)
  ✓ Batch operasyonlar (multiGet, multiPut)
  ✗ SQL-like query, join (out of scope)
  ✗ Full ACID transaction (out of scope)

Non-Functional:
  Ölçek: 100 TB veri
  Latency: < 10ms P99 (hedef < 1ms cache tier'ı için)
  Availability: 99.99%
  Consistency: Tunable (strong veya eventual, iş gereksinimi)
  Partition tolerance: zorunlu
  Durability: 11 nines (ödeme verisi için)
```

---

## Capacity Estimation

```
Veri:
  100 TB toplam veri.
  Ortalama value boyutu: 1 KB → 100B key-value çifti.
  Replication factor 3 → gerçek depolama: 300 TB.
  Node başına 10 TB SSD → 300 TB / 10 TB = 30 node minimum.
  Güvenlik payı 2x → 60 node önerilir.

QPS:
  1M kullanıcı × 100 istek/gün / 86400 ≈ 1,160 QPS ortalama.
  Peak (3x): 3,500 QPS.
  Read:Write = 10:1 → 3,200 read / 320 write QPS.

  1 node: ~10,000-50,000 QPS (in-memory).
  3,500 QPS → 1-2 node yeterli (ama HA için minimum 3 node).

Bandwidth:
  Read: 3,200 QPS × 1 KB = 3.2 MB/sn → küçük.
  Write: 320 QPS × 1 KB × RF 3 = 1 MB/sn → küçük.
  Büyük value (1 MB): 3,200 QPS → 3.2 GB/sn → NIC limiti → value boyutu sınırla.

Memory (Redis gibi in-memory):
  100 TB → tümü memory'de tutmak imkansız (çok pahalı).
  Hybrid: hot data (son 30 gün) memory, cold data disk.
  Hot data %20 = 20 TB memory → çok büyük → multi-tier cache.
```

---

## CAP Teoremi & PACELC

```
CAP: Consistency, Availability, Partition Tolerance — üçünü aynı anda alamazsın.
Network partition kaçınılmaz (P zorunlu) → C veya A'dan birini seç.

CP (Consistency + Partition):
  Partition sırasında: bazı node'lar yazma reddeder → consistency korunur.
  Sistem geçici olarak unavailable.
  Kullanım: banka bakiyesi, stok sayısı, distributed lock.
  Örnekler: etcd, ZooKeeper, HBase.

AP (Availability + Partition):
  Partition sırasında: tüm node'lar çalışmaya devam → consistency bozulabilir.
  Eventual consistency: partition iyileşince sync edilir.
  Kullanım: social feed, öneri, analitik.
  Örnekler: Cassandra, DynamoDB (eventual), CouchDB.

Tunable Consistency (gerçek dünya):
  Hem CP hem AP istemek → configürasyon ile seç.
  DynamoDB, Cassandra: okuma/yazma başına consistency level seç.
  Ödeme: strong consistent okuma → banka.
  Profil resmi: eventual → sosyal medya.

PACELC (CAP'ın uzantısı):
  Partition olmasa bile: Latency (L) vs Consistency (C) trade-off var.
  "Partition varsa: A veya C. Else: L veya C."
  Cassandra: PA/EL → partition→availability, else→latency.
  etcd: PC/EC → partition→consistency, else→consistency.
  DynamoDB: PA/EL (default) ama strong consistent okuma → PC/EC.

  Sistem tasarımcısı sorusu: "Her zaman doğru veri mi (latency↑) yoksa hızlı veri mi (stale↑)?"
```

---

## Tek Node KV Store

```
Temel yapı:
  HashMap<String, byte[]> store   // in-memory

Sorun: tek node → sınırlı kapasite, SPOF

Disk persistence (durability için):

  WAL (Write-Ahead Log):
    Her write → önce WAL'a sequential append → sonra memory güncelle.
    Crash: WAL'ı baştan replay et → state geri gelir.
    Segment dosyaları: her 64 MB → yeni segment, eski segment kompakt.

    [WAL segment-001] → [WAL segment-002] → [WAL segment-003 (aktif)]
    Kompakt: segment-001 + 002 → snapshot → segment-001,002 sil.

  Checkpointing:
    Periyodik: memory state → disk snapshot.
    Checkpoint sonrası: o ana kadar WAL silinebilir.
    Recovery: son checkpoint + sonraki WAL = güncel state.

LSM-Tree vs B-Tree seçimi:
  Write-heavy workload → LSM-Tree (RocksDB, Cassandra).
  Read-heavy workload → B-Tree (PostgreSQL, etcd).
  Çoğu KV store → LSM (write path optimize).
```

---

## Veriyi Dağıtma: Consistent Hashing

```
Naive sharding (mod N):
  shard = hash(key) % N
  N = 3 → node-0, node-1, node-2.
  Node eklendi (N=4): hemen hemen tüm key'lerin shardı değişir → tüm veri taşınır.
  Catastrophic!

Consistent Hashing (ring):
  Hash space: 0 → 2^32 (sanal ring).
  Her node: hash(node_id) → ring üzerinde 1 nokta.
  Key routing: hash(key) → ring üzerinde saat yönünde ilk node.

  Node eklendi: sadece komşu bölgedeki key'ler yeni node'a taşınır.
  Node silindi: sadece o node'un key'leri bir sonraki node'a taşınır.
  Taşınan veri = 1/N (N = node sayısı) → minimum disruption.

Virtual Nodes (vnodes):
  Problem: az node → düzensiz dağılım (bir node çok fazla veri).
  Çözüm: her fiziksel node → M sanal node (genellikle 150-200).

  hash("node-A-1") → ring'de nokta 1
  hash("node-A-2") → ring'de nokta 2
  ...
  hash("node-A-150") → ring'de nokta 150

  Avantaj:
    Uniform dağılım: 150 vnode → ring'e eşit dağılmış → her node ~eşit veri.
    Node çöküşünde: o node'un 150 vnode → yük 150 farklı node'a dağılır → hot spot yok.
    Heterojen cluster: güçlü node → daha fazla vnode.

Hash skew (hot partition) önleme:
  "celebrity problem": hash("user:celebrity") → tek node.
  Çözüm: key sonuna random suffix: "user:celebrity:0", "user:celebrity:1" → farklı node.
```

---

## Replikasyon

```
Replication Factor N=3:
  Coordinator node: key → ring → kendisi + saat yönünde N-1 sonraki node.
  3 kopya → 3 farklı fiziksel server (rack-aware: farklı rack).

Senkron vs Asenkron:
  Senkron (W=N): tüm replica'lar yazmadan client'a OK deme → güvenli, yavaş.
  Asenkron (W=1): ilk replica yazınca OK → hızlı, veri kaybı riski (crash).
  Quorum (W=2): ikisi yazınca OK → orta yol.

Quorum formülü (W + R > N):
  N=3, W=2, R=2 → 2+2 > 3 → güçlü tutarlılık (overlap garantisi).
  N=3, W=1, R=1 → eventual consistency → maksimum throughput.
  N=3, W=3, R=1 → hızlı okuma, yavaş yazma.
  N=3, W=1, R=3 → yavaş okuma, hızlı yazma.

  Seçim örneği:
    Write-heavy (log): W=1, R=1 → maximum write throughput.
    Read-heavy (cache): W=2, R=1 → okuma hızlı, yazma güvenli.
    Kritik (ödeme): W=3, R=3 → her şeyi feda et güvenlik için.

Rack-aware replikasyon:
  3 replica → 3 farklı rack.
  Rack arıza → diğer 2 rack'te kopyalar → sistem çalışmaya devam.
  Cassandra: NetworkTopologyStrategy → rack + DC aware.
```

---

## Sloppy Quorum & Hinted Handoff

```
Strict Quorum sorunu:
  N=3, W=2: Node-B ve C down → yazma başarısız → availability↓.
  CAP: consistency seçildi.

Sloppy Quorum (DynamoDB, Cassandra):
  Node-B down → ring'de sonraki Node-D yazıyı alır ("hint" ile).
  W=2 hâlâ sağlanıyor ama farklı node ile.
  "hint": "Bu veri aslında Node-B'ye ait, o geri gelince ver."

Hinted Handoff:
  Node-B çevrimiçi oldu → Node-D: "Sana ait birikmiş yazılar var."
  Transfer tamamlanır → Node-D hint siler.

  Avantaj: geçici arıza → availability korunur.
  Dezavantaj: hint sahibi node de çöküşe uğrarsa → veri kaybı.
  Çözüm: anti-entropy (Merkle tree) + hinted handoff → ikili savunma.
```

---

## Gossip Protokolü (Failure Detection)

```
Problem: 1,000 node → her node diğer 999'u periyodik ping → O(N²) yük → imkansız.

Gossip (Epidemic Protocol):
  Her node: her 1 sn → rastgele 3 node'a "health state" gönderir.
  Alıcı node: kendi bilgisini günceller → kendi rastgele 3 node'una iletir.
  Sonuç: O(log N) adımda tüm cluster'a yayılır.

State (heartbeat):
  Her node: (nodeId, heartbeat_counter, timestamp).
  Heartbeat sayacı her sn artar.
  counter artmıyorsa → node şüpheli.

Phi Accrual Failure Detector (Cassandra kullanır):
  Basit timeout (X sn gelmezse ölü): hatalı → yavaş ağda yanlış alarm.
  Phi detector: heartbeat aralıklarının istatistiksel dağılımını öğrenir.
  phi değeri: "Bu node'un ölmüş olma olasılığı".
  phi > 8 → şüpheli. phi > 12 → dead olarak işaretle.
  Avantaj: ağ gecikmesi değişkense → yanlış alarm yok.

Gossip convergence hızı:
  1,000 node, her node 3 node'a gossip → log₃(1000) ≈ 6 tur → ~6 sn'de tüm cluster bilir.
  Güçlü yayılma: küme büyüdükçe log(N) → lineer değil.
```

---

## Anti-Entropy & Merkle Tree

```
Problem: Node-B 2 saat down oldu, o sürede 1,000 write geldi.
Node-B geri geldi → nasıl eksikleri tamamlayacak?

Naif çözüm: tüm key'leri karşılaştır → O(N) → çok yavaş.

Merkle Tree (Hash Tree):
  Veri aralığını ikiye böl → her yarının hash'ini al → üst node.
  Ağaç şeklinde: leaf → veri blokları, root → tüm verinin hash'i.

  Node-A root hash: "abc123"
  Node-B root hash: "xyz789"
  Root farklı → sol alt ağaca in → sol aynı, sağ farklı → sağa in → ...
  → Sadece farklı olan veri aralığı bulunur.

  Repair: sadece farklı key'ler transfer edilir → O(diff) değil O(log N).

Anti-entropy süreci (arka plan):
  Her N saat: iki replica Merkle tree karşılaştırır.
  Farklılık → eksik/tutarsız veri → sync.
  Cassandra: "nodetool repair" komutu anti-entropy başlatır.

Read Repair (anlık):
  R=2 okuma: Node-A v2, Node-B v1 döndürdü.
  Coordinator: client'a v2 döndürür + async Node-B'yi v2 ile günceller.
  Avantaj: okuma yolu üzerinde, ekstra iş yok.
  Dezavantaj: okulanmayan key'ler hiç repair görmez.

İkili savunma:
  Read repair: sık okunan key'ler → anlık.
  Anti-entropy: tüm key'ler → periyodik arka plan.
```

---

## Okuma / Yazma Koordinasyonu

```
Write akışı:
  1. Client → herhangi node (Coordinator olur).
  2. Coordinator: hash(key) → hangi N node sorumlu?
  3. N node'a paralel write (async).
  4. W quorum yanıt → Client'a OK.
  5. Kalan N-W node → async güncellenir.

  Write path optimize:
    WAL'a append (sequential) → MemTable'a ekle → client'a dön.
    SSTable'a flush: arka planda (client beklemez).
    Sonuç: write latency < 1ms (WAL + memory).

Read akışı:
  1. Client → Coordinator.
  2. R node'dan paralel oku.
  3. Version karşılaştır (vector clock veya timestamp) → en güncel seç.
  4. Stale replica → read repair (async).
  5. Client'a en güncel değer.

  Hedged requests (tail latency azaltma):
    R=1: tek node'a istek → yavaşsa bekle.
    Hedged: 1. node'a istek gönder → 10ms içinde cevap gelmezse → 2. node'a da gönder.
    Hangisi önce cevap verirse → o kullanılır, diğeri iptal.
    P99 latency → P50'ye yaklaşır (yavaş node etkisi azalır).
```

---

## Conflict Resolution

```
Concurrent writes (network partition):
  Node-A: SET user:123 → "Ali" (t=100)
  Node-B: SET user:123 → "Ayşe" (t=101)
  Partition iyileşince: hangisi kazandı?

Last-Write-Wins (LWW):
  En yüksek timestamp kazanır.
  Avantaj: basit.
  Dezavantaj: clock skew → NTP ile bile ±ms fark → yanlış sonuç.
              Veri kaybı garantili: birisinin yazması kaybolur.
  Ne zaman kabul: cache, session (veri kaybı tolere edilebilir).

Vector Clock:
  Her node kendi sayacını tutar.
  Write: {Ali: [A:1]}, tekrar write: {Ali: [A:2]}.
  A ve B aynı anda: {Ali: [A:1, B:0]} ve {Ayşe: [A:0, B:1]} → conflict.
  Conflict: causally concurrent → uygulama katmanı karar verir.
  
  DynamoDB: vector clock → conflict varsa → client'a iki versiyonu döndür → client merge et.

  Sorun: zamanla vector clock büyür (N node → N entry).
  Pruning: eski entry'leri temizle → küçük vector clock.

CRDT (Conflict-free Replicated Data Types):
  Matematiksel garanti: her merge sonucu aynı (commutative, associative).

  G-Counter (Grow-only):
    Her node kendi sayacını artırır: A=5, B=3, C=7.
    Merge: max(A, B, C) per node → toplam = 15.
    Use case: toplam görüntülenme sayısı, beğeni.

  PN-Counter (increment/decrement):
    Pozitif counter + negatif counter.
    Net = P_total - N_total.

  OR-Set (Observed-Remove Set):
    Ekle: {Ali, tag-uuid-123}. Sil: tag-uuid-123 sil.
    Concurrent add+remove: add tag farklı uuid → add kazanır.
    Shopping cart: Cassandra bunu kullanır.

  LWW-Register:
    Value + timestamp. Merge: timestamp yüksek kazanır.
    Clock skew: HLC (Hybrid Logical Clock) ile çözülür.

Hangi strateji ne zaman:
  LWW:         cache, session, non-critical veri.
  Vector Clock: müşteri verisi, sipariş (kayıp kabul edilemez).
  CRDT:        counter, set, collaborative editing (conflict-free).
```

---

## Storage Engine

### LSM-Tree (Log-Structured Merge Tree)

```
RocksDB, Cassandra, LevelDB, TiKV, ScyllaDB kullanır.

Yazma yolu (sequential write → hız):
  1. WAL append (crash recovery).
  2. MemTable (in-memory red-black tree, sorted).
  3. MemTable dolar (64 MB) → SSTable olarak disk'e flush (immutable).

SSTable (Sorted String Table):
  Anahtarlar sıralı → binary search ile O(log N) erişim.
  Bloom filter: "Bu key bu SSTable'da var mı?" O(1) kontrol.
  Index block: key → offset mapping (sparse index).

Okuma yolu:
  1. MemTable bak (en güncel).
  2. L0 SSTable → L1 → L2 ... (en yeni → en eski).
  Bloom filter: her level'de check → yoksa skip → I/O azaltır.

Compaction (arka plan):
  L0 → L1: L0 SSTable'lar merge → L1'e.
  L1 → L2: boyutlar büyür (L1: 10 MB, L2: 100 MB, L3: 1 GB...).
  Merge sort: silinen/güncellenen key'ler temizlenir.

Write Amplification:
  1 write → WAL + MemTable + birden fazla compaction yazması.
  WA = toplam disk yazma / kullanıcı yazma.
  RocksDB ortalama: WA ≈ 10-30 (yazma 10-30x amplified).

Read Amplification:
  1 read → bloom filter × level sayısı kontrol.
  L0: 4 SSTable + L1 + L2 + L3 = 7 I/O worst case.
  Bloom filter: yanlış pozitif %1 → %99 miss → 1 I/O.

Space Amplification:
  Compaction sırasında: eski + yeni SSTable aynı anda var → 2x disk.
  Compaction bitince: eski silindi → normal boyuta döner.

Avantaj:
  ✓ Write very fast (sequential WAL + MemTable → sub-ms).
  ✓ Scan: SSTable sıralı → range scan verimli.
  ✓ Compression: SSTable block compression (Snappy/LZ4/ZSTD).

Dezavantaj:
  ✗ Read biraz yavaş (birden fazla level + bloom filter).
  ✗ Compaction I/O yükü (background).
  ✗ Write + Space amplification.
```

### B-Tree

```
PostgreSQL, MySQL, etcd, LMDB kullanır.

Yapı:
  Dengeli ağaç: her node B adet key, B+1 alt.
  Derinlik: O(log N) → her okuma maksimum O(log N) disk I/O.

In-place update:
  Key bul → sayfa oku → değeri güncelle → sayfayı geri yaz.
  WAL (redo log): crash recovery için.

Avantaj:
  ✓ Read very fast: O(log N), tek path, tahmin edilebilir.
  ✓ Range query: leaf node'lar linked list → scan verimli.
  ✓ Write amplification: 1x (sadece değişen sayfa + WAL).

Dezavantaj:
  ✗ Random write (in-place → SSD için kötü → write amplification hardware'de).
  ✗ Page split/merge: rebalancing → latency spike.
  ✗ Fragmentation: zamanla boşluk → VACUUM / REINDEX gerekli.

LSM vs B-Tree seçimi:
  Write > Read: LSM (Cassandra, RocksDB).
  Read > Write: B-Tree (PostgreSQL, etcd).
  Çoğu KV store workload: write-heavy → LSM.
```

### Bloom Filter

```
"Bu key bu set/SSTable'da var mı?" sorusunu O(1) cevapla.
False positive: "var" der ama yok → I/O bir tur boşa.
False negative: ASLA — "yok" derse kesinlikle yok.

Yapı: M bitlik dizi, K hash fonksiyonu.

Insert("ali"):
  h1("ali") % M = 3 → bit[3] = 1.
  h2("ali") % M = 7 → bit[7] = 1.
  h3("ali") % M = 14 → bit[14] = 1.

Lookup("ayşe"):
  h1("ayşe") % M = 3 → bit[3] = 1 ✓
  h2("ayşe") % M = 2 → bit[2] = 0 → KESİNLİKLE YOK. (I/O'ya gitme)

Lookup("mehmet"):
  h1 → bit[3]=1, h2 → bit[7]=1, h3 → bit[9]=1 (hepsi 1 ama "mehmet" eklenmedi)
  → False positive! Disk'e git → orada değil → boş I/O.

False positive oranı:
  p ≈ (1 - e^(-kn/m))^k
  n = eleman sayısı, m = bit sayısı, k = hash sayısı.
  n=1M, m=10M bit (1.25 MB), k=7 → p ≈ %0.8.
  %1 false positive = 1 okumada 100'de 1 ekstra disk I/O → tolere edilebilir.

KV Store'da kullanım:
  Her SSTable → kendi bloom filter → memory'de tutulur.
  Get("key"): bloom filter check → "yok" → bu SSTable'ı okuma.
  Büyük LSM: 100 SSTable × bloom filter → 100 check → 0-1 disk I/O.
```

---

## TTL (Time to Live) Yönetimi

```
Lazy expiration:
  Get key → expired mı? → sil, None döndür.
  Avantaj: overhead yok, extra cycle yok.
  Dezavantaj: expired key'ler memory/disk tutar.
  Kullanım: disk-based store (disk ucuz), erişim az.

Active expiration (Redis benzeri):
  Background thread → her 100ms → 20 rastgele key → expired mı? → sil.
  Expired key oranı > %25 → hemen tekrar çalış (döngü).
  Ortalama gecikme: birkaç saniye (tüm expired key'ler anında silinmez).

MinHeap TTL:
  Insert: (expiresAt, key) → heap'e ekle.
  Background: heap.peek() → now >= expiresAt → sil → bir sonrakine bak.
  Avantaj: kesin süreyle silme (O(log N) per deletion).
  Dezavantaj: heap bellekte → büyük TTL seti → RAM kullanımı.

Redis ZSET ile TTL (distributed):
  ZADD ttl_index {expiresAt_unix} {key}
  Expiry worker: ZRANGEBYSCORE ttl_index 0 {now_unix} LIMIT 100 → sil → ZREM.
  Avantaj: TTL state cluster genelinde paylaşılabilir.

Storage-level TTL (RocksDB):
  CompactionFilter: SSTable compaction sırasında expired key'leri atla.
  Avantaj: disk I/O'da zaten okunuyor → ücretsiz temizlik.
  Dezavantaj: compaction çalışana kadar disk'te var.

Tombstone:
  Delete(key) → key'i silme, yerine tombstone (delete marker) koy.
  Neden: replication — diğer node'lara "bu key silindi" bilgisi gerek.
  Compaction: tombstone + gerçek key aynı anda görünce → her ikisini sil.
  Tombstone birikimi: compaction seyrekse → "tombstone storm" → read yavaşlar.
```

---

## Hot Key Problemi

```
Problem: hash("celebrity") → tek node → o node'a milyonlarca istek → darboğaz.
Diğer node'lar boşta, bir node %100 CPU → sistem dengesiz.

Çözüm 1 — Key splitting:
  "celebrity:profile" → "celebrity:profile:0", ":1", ..., ":9" (10 shard).
  Write: tümüne yaz (fan-out) veya tek birine yaz, diğerleri read-replica.
  Read: rastgele suffix seç → 10 node'a dağıtılmış.
  Dezavantaj: key'in "gerçek" değerini bulmak için merge gerekebilir.

Çözüm 2 — Local cache (client-side):
  Client: hot key'i kendi memory'sine cache'le (30 sn TTL).
  Her istek DB'ye gitmiyor → node yükü dramatik düşer.
  Dezavantaj: stale data (30 sn) → write sonrası hemen görünmez.
  Ne zaman: immutable veya az değişen hot data (CDN URL, config).

Çözüm 3 — Read replica (dedicated):
  Hot key → özel read replica kümesi.
  Celebrity profili: 10 replica → 10x read throughput.
  Yük dengeli: LB round-robin → her replica eşit istek.

Çözüm 4 — Consistent hashing'de load-aware yönlendirme:
  Node CPU %80+ → o node'un bazı vnodes'ini başka node'a geçici taşı.
  "Load aware consistent hashing" — Google Maglev benzeri.

Çözüm 5 — Throttling + circuit breaker:
  Hot key'e saniyede max X istek → fazlası → rate limit → 429.
  Kötüye kullanım: bot, scraper → IP bazlı ban.
```

---

## Önbellek Geçersizleştirme (Cache Eviction)

```
LRU (Least Recently Used):
  Doubly linked list + HashMap → O(1) get/put/delete.
  Get("key"): HashMap → node → linked list'in head'ine taşı.
  Kapasite dolunca: tail'den sil (en az kullanılan).
  Dezavantaj: "scan" pattern: büyük dosyayı bir kez okurken tüm cache bozulur.

LFU (Least Frequently Used):
  Her key: frekans sayacı.
  Silinecek: en düşük frekans.
  Scan resistance: bir kez okunan key → düşük frekans → erken silinir.
  Dezavantaj: yeni eklenen key frekansı 1 → hemen silinebilir.
  Counter aging: her T sürede counter/2 → eski popülerlik unutulur.

TinyLFU (Redis 4.0+ default):
  LFU + frekans tahmini (Count-Min Sketch).
  Çok az memory ile frekans bilgisi → "%1 memory ile %99 doğruluk."
  Redis: allkeys-lfu → TinyLFU kullanır.

ARC (Adaptive Replacement Cache):
  T1: son 1 kez erişilen (LRU gibi).
  T2: birden fazla kez erişilen (LFU gibi).
  p: T1/T2 dinamik dengesi → workload'a göre kendini ayarlar.
  Oracle DB, NetApp WAFL kullanır.
  Avantaj: hem scan hemde frequency pattern'leri iyi yakalar.

Redis eviction tipleri:
  allkeys-lru:     tüm key'lerden LRU sil.
  allkeys-lfu:     tüm key'lerden LFU sil (Redis 4.0+).
  volatile-lru:    sadece TTL'li key'lerden LRU.
  volatile-ttl:    süresi en yakın key silinir.
  noeviction:      hata fırlat (cache doldu, write reddedildi).
  allkeys-random:  rastgele key sil (test dışında önerilmez).
```

---

## Multi-Region Replikasyon

```
Aktif-Pasif (master-slave):
  Bölge 1 (TR): yazma kabul eder → master.
  Bölge 2 (EU): okuma → slave → async replicated.
  Failover: TR down → EU master'a terfi → DNS güncelle.
  RPO: replication lag (saniyeler) → küçük veri kaybı riski.
  RTO: DNS propagation + warmup → 1-5 dk.

Aktif-Aktif:
  Her bölge hem okuma hem yazma kabul eder.
  Conflict: aynı key'e iki bölgeden concurrent write → CRDT veya LWW.
  DynamoDB Global Tables: aktif-aktif, LWW conflict resolution.
  Cassandra: multi-DC, LOCAL_QUORUM per region.

Cross-DC replication maliyeti:
  Senkron: her write → cross-DC round trip → 50-150 ms → latency katlanır.
  Asenkron: local quorum → OK → background cross-DC → düşük latency.
  Seçim: kritik veri → senkron. Performans kritik → asenkron.

Geo-aware routing:
  Client → GeoDNS → en yakın region.
  Read: local region (düşük latency, eventual).
  Write: critical → global region (güçlü tutarlılık, yüksek latency).
```

---

## API Tasarımı

```java
interface KeyValueStore {
    // Temel operasyonlar
    void put(String key, byte[] value, Duration ttl);
    Optional<byte[]> get(String key);
    void delete(String key);
    boolean exists(String key);

    // Batch operasyonlar (network round-trip azalt)
    void multiPut(Map<String, byte[]> entries);
    Map<String, Optional<byte[]>> multiGet(List<String> keys);
    void multiDelete(List<String> keys);

    // Conditional (optimistic locking)
    boolean putIfAbsent(String key, byte[] value, Duration ttl);
    boolean compareAndSwap(String key, byte[] expected, byte[] newValue);

    // Range scan (LSM-tree üzerinde verimli)
    List<Entry<String, byte[]>> scan(String startKey, String endKey, int limit);

    // TTL
    Optional<Duration> getTtl(String key);
    void expire(String key, Duration ttl);
}

// Distributed counter (CRDT)
interface DistributedCounter {
    long increment(String key);
    long incrementBy(String key, long delta);
    long get(String key);
    void reset(String key);
}

// Pub/Sub (Redis benzeri)
interface PubSub {
    void publish(String channel, byte[] message);
    Subscription subscribe(String channel, MessageHandler handler);
    Subscription pSubscribe(String pattern, MessageHandler handler); // glob pattern
}
```

---

## Monitoring & Operasyon

```
Temel metrikler:
  get_latency_ms{p50, p99, p999}  → latency SLA
  put_latency_ms{p50, p99, p999}  → write performansı
  cache_hit_rate                   → hit/(hit+miss) → %95+ hedef
  eviction_rate                    → yüksekse → memory yetersiz
  memory_usage_bytes               → eviction politikası devreye girme noktası
  replication_lag_ms               → replica ne kadar geride?
  compaction_pending_bytes         → LSM: bekleyen compaction
  key_count                        → toplam key sayısı (büyüme trendi)

Alarmlar:
  p99_latency > 10ms     → "KV store yavaş, kapasiteyi incele"
  hit_rate < 80%         → "Cache miss yüksek, boyutu artır"
  memory_usage > 85%     → "Eviction başlamak üzere"
  replication_lag > 5sn  → "Replica geride, failover riski"
  node_down              → PagerDuty alert (N-1 replica kaldı)

Operasyonel komutlar:
  Redis: INFO all → tüm metrikler
  Redis: DEBUG SLEEP 5 → bir node'u 5sn uyut (failover test)
  Cassandra: nodetool status → ring durumu
  Cassandra: nodetool repair → anti-entropy tetikle
  RocksDB: db_stats → compaction, write amp, level sizes
```

---

## Olası Sorunlar ve Çözümleri

### 1. Hot Key — Tek Node %100 CPU, Diğerleri Boşta

```
Sorun:
  Yeni ürün lansmanı: "iphone16:stock" key'i 500,000 req/sn alıyor.
  Consistent hashing: hash("iphone16:stock") → Node-3.
  Node-3: %100 CPU → P99 latency 500ms → timeout.
  Diğer 9 node: %5 CPU → boşta.
  Sistem "yavaş" görünüyor ama aslında tek node doldu.

Çözüm:
  a) Key sharding (anlık):
     "iphone16:stock" → "iphone16:stock:0" ... ":9" (10 shard).
     Write: tek bir shard'a yaz (atomic, önce "master" shard seç).
     Read: rastgele suffix → 10 node'a yük dağıtımı.
     Uyarı: "stok var mı?" için tüm shard'ları topla → fan-in.

  b) Client-side local cache:
     Hot key → client memory'sine 5 sn cache.
     500,000 req → sadece 200 req/5sn = 40,000 req/dk DB'ye.
     Stale: 5 sn → "stok az gösterebilir ama oversell DB'de önlenir."

  c) Read replica (hot key için):
     "iphone16:stock" → 5 dedicated read replica.
     LB: round-robin → her replica 100,000 req/sn.
     Write: master → async tüm replica'ya.
     Replication lag: 10ms → stale tolere edilebilir.

  d) Monitoring + otomatik tespit:
     Prometheus: key başına istek sayısı izle.
     Key > 10,000 req/sn → "hot key" alert → otomatik sharding devreye.
```

---

### 2. Network Partition — Split Brain

```
Sorun:
  3 node: A, B, C.
  Network split: A ↔ (B, C) ayrıldı.
  A: "Ben leader'ım" → yazmaya devam ediyor.
  B, C: "A ölü, ben leader'ım" → B yazmaya devam ediyor.
  Partition düzeldi: A ve B aynı key'e farklı değer yazmış → split brain.

CP (etcd/ZooKeeper) davranışı:
  Raft: quorum gerektir → 3 node, 2 gerekli.
  A: tek başına → quorum yok → yazma reddeder.
  B, C: 2 node → quorum var → yalnız B çalışmaya devam eder.
  Partition bitince: A → B'nin state'ini alır → sync.
  Sonuç: A'da kayıp (CP seçildi) ama tutarlılık korundu.

AP (Cassandra) davranışı:
  Sloppy quorum: A'da W=1 kabul → yazıyor.
  B, C de yazıyor.
  Partition bitince: conflict → LWW veya vector clock ile resolve.
  Veri kaybı yok ama çakışma → merge gerekli.

Önlem (üçlü strateji):
  a) Odd node sayısı (3, 5, 7): quorum her zaman oluşabilir.
  b) Rack/AZ aware: A → AZ1, B → AZ2, C → AZ3.
     Tek AZ arızası → diğer 2 AZ quorum sağlar.
  c) Fencing token: leader değişince → eski leader için oluşturulan tokenlar geçersiz.
     Eski leader yazma girişimde → "token expired" → yazma reddedilir.
```

---

### 3. Tombstone Storm — Silinen Key'ler Read'i Yavaşlattı

```
Sorun:
  Uygulama: milyonlarca key sildi (session temizliği, cache flush).
  LSM-Tree: silme = tombstone ekleme (gerçek silme değil).
  Compaction seyrek çalıştı → 100M tombstone birikti.
  Get("any_key"): her SSTable'ı tara → tombstone'ları atla → süreç yavaş.
  P99 latency: 2ms → 50ms → kullanıcı şikayeti.

Çözüm:
  a) Compaction sıklaştır:
     RocksDB: compaction_trigger'ı düşür (L0'da 4 SSTable → 2).
     Daha sık compaction → tombstone'lar erken temizlenir.
     Dezavantaj: daha fazla background I/O → write throughput↓.
     Trade-off: burst delete sonrası geçici compaction agresifleştir.

  b) TTL kullan (delete yerine):
     Delete(key) yerine: SET key value EX 1 (1 sn TTL).
     Storage-level expiry: compaction sırasında expired = zaten silinmiş.
     Tombstone yok → daha temiz okuma.

  c) Compaction filter (RocksDB):
     Özel filter: tombstone yaşı > 7 gün → compaction'da fiziksel sil.
     Normal: tombstone → tüm replica'lara yayıldığından sonra silinmeli.
     Distributed tombstone: GC koordinasyonu gerekli.

  d) Ayrı "delete" namespace:
     Büyük silme operasyonu → "cold" node'da çalıştır.
     Tombstone'lar cold node'da birikir → hot path etkilenmez.
     Periyodik: cold node compaction → clean state.
```

---

### 4. Replication Lag — Replica Çok Geride, Stale Read

```
Sorun:
  Node-A (primary): ödeme onaylandı → balance güncellendi.
  Node-B (replica): 5 sn geride → kullanıcı Node-B'den okudu → eski bakiye.
  Kullanıcı: "Ödeme yaptım ama bakiyem değişmedi" → destek şikayeti.
  Replikasyon lag sebebi: Node-B disk I/O yoğun → compaction → replikasyonu yavaşlattı.

Çözüm:
  a) Read-your-writes (session consistency):
     Ödeme yapan kullanıcı: bir sonraki okumayı primary'den yap.
     "Sticky read": write sonrası 5 sn → primary okuma.
     5 sn sonra: lag kapandı → replica'dan okuma tamam.
     Uygulama: write token {userId, timestamp} → cookie'ye yaz → read sırasında kontrol.

  b) Monotonic reads:
     Kullanıcı replica-B'den v5 gördü → sonraki istek replica-A'ya gitti → v3 görüyor.
     Timestamp: kullanıcının gördüğü en yüksek versiyon → session'da sakla.
     Read: "en az bu versiyon" → lag'da replica → primary'ye yönlendir.

  c) Bounded staleness:
     Max lag = 500ms. Lag > 500ms → o replica'yı okuma dışı bırak.
     Monitoring: replication_lag metric → alert.
     Otomatik: lag > threshold → replica degrade → primary'ye yönlendir.

  d) Compaction sırasında replikasyon önceliği:
     RocksDB: compaction priority düşür → replikasyon daha fazla CPU alır.
     Replikasyon lag → önce kapat → compaction ikincil.
```

---

### 5. Memory Doldu — Eviction Kritik Veriyi Sildi

```
Sorun:
  Redis memory: %95 doldu → allkeys-lru aktif.
  LRU: en uzun süredir kullanılmayanı sil.
  Session token: 30 dk önce set edildi, henüz kullanılmadı → LRU'nun en altında.
  Büyük scan (analytics): çok key okundu → session token'lar LRU'dan düştü.
  Kullanıcılar: oturumları kapandı → "neden çıktım?" şikayeti.

Çözüm:
  a) TTL bazlı eviction (volatile-lru):
     volatile-lru: sadece TTL'li key'lerden sil → TTL'siz kritik key'ler güvende.
     Session token: TTL set et.
     Config data: TTL yok → eviction'dan korunur.

  b) Memory partition (namespace):
     Redis: ayrı DB numaraları (0-15) → DB 0: session, DB 1: cache, DB 2: config.
     Her DB için ayrı maxmemory → birbirini etkilemez.
     Daha iyi: ayrı Redis instance → tam izolasyon.

  c) Memory alarm + proaktif scale:
     Prometheus: memory_usage > 70% → Slack alert.
     >80% → auto scale (Redis Cluster yeni shard ekle).
     >90% → acil → on-call uyandır.

  d) Değer boyutu kontrolü:
     Large value: büyük JSON → memory'nin büyük kısmını tek key tutuyor.
     Alarm: key size > 1 MB → log + flag.
     Uygulama: büyük veriyi S3'e → Redis'te sadece S3 URL.
```

---

### 6. Kompaksiyon I/O Spike — Yazma Durdu

```
Sorun:
  Production gece yarısı: RocksDB L0 → L1 kompaksiyon tetiklendi.
  100 GB SSTable merge → 30 dk sürdü.
  Bu 30 dk: disk I/O %100 → write stall (RocksDB yeni write'ı bekletiyor).
  Uygulama: put() çağrısı → 10 sn timeout → exception.
  Alarm: "KV store yazma hatası" → yanlış alarm gibi göründü ama gerçek.

Neden RocksDB write stall:
  L0'da 20 SSTable birikti (eşik: 20) → compaction tetiklendi.
  L0 doluyken yeni write → stall → "compaction bitsin, sonra yaz."

Çözüm:
  a) Compaction ayarları:
     level0_slowdown_writes_trigger: 20 → 40 (daha geç yavaşlama).
     level0_stop_writes_trigger: 36 → 60 (daha geç durma).
     max_background_compactions: 2 → 4 (daha paralel compaction).
     Dezavantaj: disk I/O artar, SSD ömrü.

  b) Write throttling:
     L0 > 12 SSTable → 10% write throttle (yavaşlat ama durma).
     Uygulama: retry + backpressure → compaction bitimine kadar.

  c) Off-peak compaction scheduling:
     Manual compaction: gece 02:00-04:00 tetikle (trafik az).
     Gündüz: compaction priority düşür → write throughput öncelikli.

  d) SSD seçimi:
     NVMe SSD: random I/O → write stall daha kısa.
     SATA SSD: daha yavaş → write stall daha uzun.
     LSM-Tree → sequential write → SSD dostu ama compaction random I/O.
```

---

### 7. Consistent Hashing Node Eklendi — Veri Transferi Traffic'i Çökertti

```
Sorun:
  Cluster: 10 node, her biri 10 TB (toplam 100 TB).
  11. node eklendi → 1/11 veri transfer edilmeli ≈ 9 TB.
  Transfer: 9 TB × hız 100 MB/sn = 90,000 sn = 25 saat.
  Bu 25 saat: network %80 dolu → normal istek latency 5ms → 50ms.
  Kullanıcı: yavaş deneyim.

Çözüm:
  a) Throttled data migration:
     Transfer: max 20 MB/sn (%20 network) → 25 saat → 5 gün (yavaş ama sağlıklı).
     Gündüz: 10 MB/sn. Gece: 40 MB/sn → adaptif.

  b) Kademeli node ekleme:
     Haftada 1 node → 10 hafta → 20 node.
     Her seferinde az veri taşınır → network yükü minimal.

  c) Vnode yeniden dengeleme (önce soğuk):
     Yeni node başta boş → önce cold data (eski, az erişilen) taşı.
     Hot data: en son taşı → kısa window.
     Uygulama etki: sadece hot data transfer süresinde risk.

  d) Ön hazırlık (pre-warming):
     Yeni node → önce cache warm-up (boş node cold start sorunu).
     Read-through: ilk okuma miss → source node'dan al → yeni node'a cache.
     Birkaç saat sonra: yeni node %80 hit → transfer yoğunluğu düşer.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| CAP | CP (strong) | AP (eventual) tunable | Business gereksinim: ödeme→CP, cache→AP |
| Sharding | Mod N | Consistent Hashing | Mod N: node ekleme/çıkarmada tüm veri taşınır |
| Conflict | LWW | Vector Clock / CRDT | LWW: clock skew riski; VC: causal correctness |
| Storage engine | B-Tree (read opt) | LSM-Tree (write opt) | KV store: write-heavy workload → LSM |
| Replikasyon | Senkron (W=N) | Quorum (W=N/2+1) | Senkron: yavaş; Quorum: balance |
| Failure detection | Ping-timeout | Gossip + Phi detector | Ping: O(N²); Gossip: O(log N) |
| Partition repair | Full scan | Merkle Tree | Full scan: O(N); Merkle: O(diff) |
| Hot key | Tek node | Key sharding + local cache | Tek node: darboğaz; Sharding: dağıtılmış yük |
| Eviction | LRU | TinyLFU (Redis 4+) | LRU: scan sensitivity; LFU: frequency-aware |
| Tombstone | Immediate delete | Lazy (compaction) | Immediate: distributed coord. zor; Lazy: LSM-native |
| Multi-region | Aktif-pasif | Aktif-aktif (CRDT) | Aktif-pasif: failover yavaş; Aktif-aktif: global write |
