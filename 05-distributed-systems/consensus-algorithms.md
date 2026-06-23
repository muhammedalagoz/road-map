# 05a — Consensus Algoritmaları (Raft & Paxos)

## Ne?

**Consensus (Uzlaşı):** Dağıtık bir sistemdeki birden fazla node'un, aralarından biri veya birkaçı çökse bile bir değer üzerinde anlaşmasını sağlayan algoritma ailesi.

Consensus, distributed sistemlerin temel taşıdır:
- **etcd** (Kubernetes config store) → Raft kullanır
- **ZooKeeper** (Kafka coordination) → ZAB (Raft benzeri) kullanır
- **Kafka KRaft** (ZooKeeper'sız mod) → Raft kullanır
- **Consul** (service discovery) → Raft kullanır
- **CockroachDB, TiKV** → Raft kullanır

---

## Neden?

**Çözdüğü problem:** Birden fazla node'un aynı anda "lider" olması (split-brain) veya node'ların farklı değerler üzerinde hemfikir olması.

```
Senaryo: 3 sunucu, value "X" kabul edilmeli mi?

Naive yaklaşım:
  Node A: "evet" → kaydet
  Node B: "evet" → kaydet
  Node A çöktü → Node B & C var
  Soru: A kaydetti mi? C ne yapmalı?

Consensus olmadan: veri tutarsızlığı, "split brain", veri kaybı
Consensus ile: çoğunluk (quorum) karar verir → tek doğru değer
```

---

## Nasıl?

### Paxos (Orijinal — Teorik Temel)

Leslie Lamport tarafından 1989'da tanımlandı. Anlaşılması zordur ama modern algoritmaların temelidir.

```
3 rol:
  Proposer  → değer önerir
  Acceptor  → önerileri kabul/reddeder
  Learner   → kabul edilen değeri öğrenir

2 aşama:
  Phase 1 (Prepare):
    Proposer → Acceptor'lara "Prepare(n)" (n = proposal number)
    Acceptor → "Promise(n, önceki_değer)" veya reddeder

  Phase 2 (Accept):
    Proposer → çoğunluktan promise aldıysa → "Accept(n, value)"
    Acceptor → kabul eder → Learner'lara bildirir

Garantiler:
  - Çoğunluk (quorum = N/2 + 1) gerekli
  - Bir kez seçilen değer değişmez
  - Birden fazla proposer → liveness sorunu (durum: hiçbir şey seçilmedi)
```

**Neden Paxos pratik değil?**
- Multi-Paxos karmaşık (lider seçimi, log replication ayrı problemler)
- İmplementasyon zor, hata eğilimli
- Bu yüzden Raft çıktı

---

### Raft (Modern — Anlaşılır Consensus)

Diego Ongaro & John Ousterhout, 2014. "Understandable consensus" hedefiyle tasarlandı.

```
3 node state:
  Follower  → pasif, liderden komut bekler
  Candidate → lider adayı, oy toplar
  Leader    → tüm log yazmaları burada, follower'lara replike eder

Term (dönem):
  Her lider seçiminde artır. Monoton artan sayı.
  Eski termin mesajları reddedilir.
```

#### Lider Seçimi (Leader Election)

```
Normal durum:
  Leader → Heartbeat (empty AppendEntries) → Followers
  Follower heartbeat alır → "lider hayatta" → bekler

Lider çöktü:
  Follower: election timeout (150-300ms random) doldu → "Candidate" oldu
  Candidate → RequestVote(term, lastLogIndex, lastLogTerm) → diğerleri

Oy verme kuralı:
  1. Adayın term'i >= benim term'im
  2. Adayın log'u en az benim log'um kadar "up-to-date"
  3. Bu term'de daha önce oy vermedim
  → Tüm şartlar sağlanırsa → oy ver

Quorum = N/2 + 1 oy alan → Leader olur
  3 node: 2 oy yeterli
  5 node: 3 oy yeterli

Random timeout neden?
  Birden fazla candidate aynı anda → oy bölünür → kimse quorum alamaz
  Random timeout → genellikle bir candidate önce başlar → oy toplar
```

#### Log Replication

```
Client → Leader → "SET x = 5"

1. Leader log'a yazar (uncommitted):
   Log: [index:1, term:1, "SET x = 5"]

2. AppendEntries → Tüm Follower'lar
   Follower log'a yazar (uncommitted)

3. Çoğunluk (quorum) yazdıysa → Leader commit eder
   → "SET x = 5" uygulandı
   → Client'a "OK"

4. Sonraki heartbeat'te Follower'lar da commit eder

Follower çöküp geri gelirse:
  Leader → eksik entry'leri gönderir (AppendEntries ile)
  nextIndex[] → her follower için "nereye kadar göndereyim" pointer

Leader çöküp geri gelirse:
  Yeni lider seçilir
  Eski leader'ın uncommitted log entry'leri → gerekirse overwrite edilir
  (commit edilmemiş entry'ler kaybolabilir — bu normaldir)
```

#### Raft Güvenlik Garantileri

```
Election Safety:
  Bir term'de en fazla bir lider seçilir (quorum garantisi)

Log Matching:
  İki log'da aynı index/term'de aynı komut varsa → önceki tüm entry'ler aynı

Leader Completeness:
  Commit edilmiş entry → tüm gelecekteki liderların log'unda

State Machine Safety:
  Aynı index'te farklı iki node farklı komut uygulamaz
```

---

### Java/Spring ile Raft — etcd Client Örneği

```java
// Spring Cloud Kubernetes + etcd için Lease-based distributed lock
// (etcd Raft cluster'ın üstünde çalışır)

@Service
class EtcdLeaderElection {

    private final Client etcdClient;

    @PostConstruct
    void startElection() {
        Election election = Election.builder(etcdClient)
            .name("/my-service/leader")
            .build();

        // Campaign = lider olmaya aday ol
        // Block eder — lider olunca döner
        CompletableFuture.runAsync(() -> {
            try {
                LeaderKey leaderKey = election.campaign("node-1").get();
                log.info("Bu node lider oldu!");
                doLeaderWork();
                election.resign(leaderKey).get(); // liderliği bırak
            } catch (Exception e) {
                log.error("Election failed", e);
            }
        });
    }
}
```

---

### Raft vs Paxos Karşılaştırması

```
Raft özellikleri:
  - Strong leader: tüm yazma işlemi liderden geçer
  - Randomized election timeout: basit, anlaşılır
  - Log-centric: log = truth, state machine log'dan türetilir
  - Joint consensus: cluster üyeliği değişikliği güvenli

Paxos özellikleri:
  - Multi-Paxos: lider var ama Raft kadar strict değil
  - Teorik olarak minimum round-trip
  - Google Chubby, Spanner kullanır (Multi-Paxos)

Pratikte:
  - Yeni sistemler → Raft (etcd, CockroachDB, TiKV, Consul)
  - Google sistemleri → Paxos/ZAB (tarihsel)
  - ZooKeeper → ZAB (Raft benzeri ama önceden geliştirildi)
```

---

## Ne zaman?

**Consensus algoritmaları ne zaman devreye girer:**
```
✓ Lider seçimi gerektiğinde (Kafka broker seçimi → KRaft)
✓ Config/metadata dağıtımı (etcd → Kubernetes, Consul → service discovery)
✓ Distributed lock implementasyonu (ZooKeeper)
✓ Distributed transaction coordinator seçimi
✓ Replicated state machine (database replication, log replication)
```

**Doğrudan consensus implement etme:**
```
✗ Zaten etcd, ZooKeeper, Consul kullanıyorsan → bunların üstünde çalış
✗ Raft kütüphanesi: Apache Ratis (Java), etcd/raft (Go)
✗ Kendin yazmaya çalışma — edge case'ler inanılmaz zordur
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Strong consistency garantisi | Quorum yazması yavaş (N/2+1 node beklenir) |
| Partition tolerant (quorum sağlanırsa) | Leader bottleneck — tüm yazma tek node'dan |
| Otomatik failover (yeni lider seçilir) | Partition büyükse sistem yazamaz (CP seçimi) |
| Tutarlı sıralı log | Raft cluster'ı küçük tutulmalı (3, 5, 7 node) |
| Lider çöküşünde veri kaybı yok (commit edilmiş) | Network partition → minority taraf yanıt veremez |
