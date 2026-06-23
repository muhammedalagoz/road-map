# 05e — Split-Brain, Quorum & PACELC

---

## 1. Split-Brain Problemi

### Ne?

**Split-Brain:** Ağ bölünmesi (network partition) nedeniyle bir dağıtık sistemin iki veya daha fazla bağımsız parçaya ayrılması ve her parçanın kendini "lider" veya "ana kopya" sanması.

```
Normal:
  Node A ←── network ──→ Node B ←── network ──→ Node C
  [Leader: A]

Network partition:
  Node A  ✗──── (bölünme) ────✗  Node B ←──→ Node C
  
  A: "B ve C'ye ulaşamıyorum, onlar çökmüş" → liderlik iddiası
  B+C: "A'ya ulaşamıyoruz, A çökmüş" → yeni lider seç: B

  Şimdi:
  A: "Ben liderim, yazmaları kabul ediyorum"
  B: "Ben liderim, yazmaları kabul ediyorum"
  → Aynı veri iki yerden farklı güncellendi → veri çakışması
```

### Neden Ciddi?

```
Banka örneği (split-brain):
  Hesap bakiyesi: 1000 TL
  
  Partition:
  Node A: DEBIT 500 → bakiye 500 TL → commit (A kendi kendine lider)
  Node B: DEBIT 700 → bakiye 300 TL → commit (B kendi kendine lider)
  
  Partition iyileşince:
  A: 500 TL
  B: 300 TL
  Hangisi doğru? — belirsiz!
  Toplam 1200 TL çekildi, ama hesapta 1000 TL vardı.
```

### Nasıl?

#### Çözüm 1: Quorum (Çoğunluk Kararı)

```
N node, quorum = N/2 + 1

3 node:
  Quorum = 2
  Partition: A | B + C
  A: sadece 1 oy → quorum yok → yazma reddet
  B+C: 2 oy → quorum → yazma kabul

5 node:
  Quorum = 3
  Partition: A + B | C + D + E
  A+B: 2 oy → quorum yok → yazma reddet
  C+D+E: 3 oy → quorum → yeni lider seç

Garanti: Asla iki taraf aynı anda quorum sağlayamaz (N/2+1 + N/2+1 > N)
```

```
Etcd / Raft:
  3 node cluster
  1 node çöküp geri gelirse: 2/3 quorum → çalışır
  2 node çöküp geri gelirse: 1/3 quorum → yazma durdurulur, okuma da

Cassandra tunable quorum:
  N=3 replikasyon
  QUORUM write: 2/3 node yazmalı
  QUORUM read: 2/3 node okumalı
  R + W > N garantisi → her okuma en az bir node'da yazılmış veri görür
```

#### Çözüm 2: Fencing (Eskimiş Lider Engelleme)

```
Problem: Eski lider "zombie" gibi çalışmaya devam eder
  Eski leader (token=1) → sisteme yazma dener
  Yeni leader (token=2) → seçildi

Fencing token:
  Her lider seçiminde monoton artan token
  Depolama katmanı (DB, lock service) → düşük token → reddeder

Leader epoch (Kafka):
  Her controller seçiminde epoch artar
  Eski controller'dan gelen mesajlar: epoch=5
  Yeni controller epoch=6
  Broker: "epoch 5 < 6, bu mesaj eski liderden, reddet"
```

#### Çözüm 3: Witness Node (3. Taraf Hakem)

```
2 node: Her partition kendini lider sayabilir (quorum yok)
3. node (witness/arbitrator):
  Sadece oy verir, veri tutmaz
  Maliyetsiz: zayıf sunucu olabilir
  2 node partition'da witness hangi tarafla → o taraf quorum

AWS RDS Multi-AZ:
  Primary (AZ-a) + Standby (AZ-b) + witness (3. AZ)
  a-b bağlantısı kopunca: witness quorum verir → failover
```

---

## 2. Quorum Reads & Writes

### Ne?

**Quorum:** N kopyadan en az W'ye yazma ve R'den okuma garantisi. W + R > N olduğunda strong consistency sağlanır.

```
Cassandra örneği (RF=3, yani 3 replikasyon):

Senaryolar:
  W=3, R=1:  Strong write, fast read (tüm replica güncellenmeli)
  W=1, R=3:  Fast write, strong read
  W=2, R=2:  QUORUM — dengeli (W+R=4 > N=3 → overlap garantili)
  W=1, R=1:  Eventual consistency, en hızlı ama stale okuma riski
```

### Nasıl?

```
W + R > N garantisi (her okuma en az bir yazılmış kopyayı görür):

N=3, W=2, R=2:
  Write:
    Replica 1: YAZILDI ✓
    Replica 2: YAZILDI ✓
    Replica 3: henüz yazılmadı

  Read:
    Replica 2: OKUNDU ✓ (güncel)
    Replica 3: OKUNDU (eski) 
    → En az 1 overlap (Replica 2) → güncel veri okunur

  Matematiksel kanıt:
    W + R > N → W + R ≥ N + 1
    → Read ve Write setleri mutlaka en az 1 ortak node içerir
    → O node her ikisini de gördü → güncel veri

N=3, W=1, R=1:
  W+R = 2, N = 3 → overlap garantisi yok → eventual consistency
```

```java
// Cassandra Java Driver — Consistency Level
CqlSession session = CqlSession.builder()
    .withKeyspace("shop")
    .build();

// Quorum okuma
ResultSet rs = session.execute(
    SimpleStatement.builder("SELECT * FROM products WHERE id = ?")
        .addPositionalValue(productId)
        .setConsistencyLevel(ConsistencyLevel.QUORUM) // 2/3 node'dan oku
        .build()
);

// Hızlı yazma (ONE) + quorum okuma → eventual consistency
session.execute(
    SimpleStatement.builder("UPDATE products SET stock = ? WHERE id = ?")
        .addPositionalValues(newStock, productId)
        .setConsistencyLevel(ConsistencyLevel.ONE) // sadece 1 node'a yaz
        .build()
);

// LOCAL_QUORUM — çok bölgeli cluster'da sadece yerel quorum
// (cross-datacenter latency'den kaçın)
.setConsistencyLevel(ConsistencyLevel.LOCAL_QUORUM)
```

#### Quorum + Sloppy Quorum (DynamoDB/Cassandra)

```
Sloppy quorum:
  "Gerçek" N node'a yazılamıyorsa → "hinted handoff" ile başkasına yaz
  Sonra asıl node geri gelince → gönder

DynamoDB:
  Ana node'a ulaşılamıyor → "hint" alır, düzelince transfer
  Avantaj: daha fazla availability
  Dezavantaj: W+R>N garantisi bozulabilir (sloppy quorum → strict quorum değil)
```

---

## 3. PACELC Teoremi

### Ne?

**PACELC (Daniel Abadi, 2012):** CAP teoremine ek olarak, **partition olmadığında** da **latency vs consistency** arasında seçim yapılması gerektiğini söyleyen genişletilmiş teorem.

```
CAP → "Partition olduğunda: Consistency mi Availability mı?"
PACELC → "Partition olduğunda C vs A, Else (normal çalışma) L vs C"

P: Partition varsa:
  A: Availability (AP sistem) → stale veri dönebilir
  C: Consistency (CP sistem) → bazı istekler reddedilebilir

E: Else (normal çalışma, partition yok):
  L: Low Latency → hızlı yanıt, eventual consistency
  C: Consistency → yavaş yanıt (quorum bekle, replication tamamlansın)
```

### Sistemlerin PACELC Konumları

```
PA/EL — Partition→Availability, Normal→Low Latency:
  DynamoDB, Cassandra (default), Riak
  Normal çalışmada da hız tercih edilir → eventual consistency

PC/EC — Partition→Consistency, Normal→Consistency:
  HBase, BigTable, VoltDB
  Her zaman strong consistency → yavaş ama doğru

PA/EC (nadir):
  MongoDB (yazma onayı = 1 ise)

PC/EL:
  Pratik sistemde zor — partition olunca consistency ama normal çalışmada hız
  Bazı sistemler configurable: Cassandra CL=ALL vs CL=ONE
```

### Gerçek Dünya Seçimleri

```
E-ticaret ürün kataloğu:
  PACELC: PA/EL
  Neden: Catalog nadiren değişir, birkaç saniye stale kabul edilebilir
  Seçim: Cassandra CL=ONE veya DynamoDB eventual consistency

Banka transferi:
  PACELC: PC/EC
  Neden: Tutarsız bakiye felaket
  Seçim: CockroachDB, Spanner, PostgreSQL (single node veya synchronous replication)

Shopping cart:
  PACELC: PA/EL
  Neden: Sepete ürün ekleme gecikmesi tolere edilemez, stale tolere edilebilir
  Seçim: DynamoDB eventual consistency

Oturum/Token:
  PACELC: PA/EL
  Neden: Redis cache — saniyeler içinde tüm node'lara yayılır
  Bir kullanıcı "logged out" görünse bile 1-2 saniye fark etmez

Inventory (stok):
  PACELC: PC/EC veya PA/EL (iş modeline göre)
  Overselling kabul edilebiliyorsa → PA/EL (hız + sonradan compensate)
  Overselling kabul edilemiyorsa → PC/EC (yavaş ama doğru)
```

---

## 4. Vector Clock (Nedensellik Takibi)

### Ne?

**Vector Clock:** Her node'un kendi saat sayacını tuttuğu, olaylar arasındaki nedensel ilişkiyi (causality) belirlemek için kullanılan dağıtık saat mekanizması.

```
Lamport Timestamp (daha basit):
  Her event'te clock artar, mesaj gönderimde clock iletilir
  T(receive) = max(local, received) + 1
  Sınır: hangi olayın hangisinden önce olduğunu gösterir ama
         "concurrent" (bağımsız) olayları ayırt edemez

Vector Clock:
  N node için N boyutlu vektör
  [VC_A, VC_B, VC_C] → her elemanı ilgili node'un clock'u
```

```
3 node: A, B, C
Başlangıç: A=[0,0,0], B=[0,0,0], C=[0,0,0]

A event yapar:
  A=[1,0,0]

A → B mesaj gönderir (A=[1,0,0]):
  B: max([0,0,0], [1,0,0]) + B'nin sayacı = [1,1,0]

B event yapar:
  B=[1,2,0]

B → C gönderir:
  C: max([0,0,0], [1,2,0]) + C = [1,2,1]

C event yapar:
  C=[1,2,2]

B ve C bağımsız event yaparsa:
  B: [1,3,0]
  C: [1,2,3]
  → [1,3,0] vs [1,2,3] — karşılaştırılamaz → concurrent!
  → Conflict resolution gerekli (last-write-wins, CRDT, vb.)

Kullanım:
  Amazon DynamoDB (internal)
  Amazon Dynamo (orijinal paper)
  Riak
```

---

## Trade-off Özeti

| Sorun | Çözüm | Maliyeti |
|-------|-------|---------|
| Split-brain | Quorum | Partition'da availability kaybı |
| Split-brain | Witness node | Ekstra bileşen |
| Split-brain | Fencing token | Yazma latency artar |
| Stale read | W+R > N quorum | Yüksek latency (daha fazla node bekle) |
| Yüksek latency | Eventual consistency | Geçici tutarsızlık |
| Concurrent write conflict | Vector clock | Conflict resolution karmaşıklığı |
| PACELC PA/EL | Hız | Stale data |
| PACELC PC/EC | Tutarlılık | Latency, düşük availability |
