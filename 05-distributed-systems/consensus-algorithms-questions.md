# 05a — Consensus Algoritmaları: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Quorum & Cluster Boyutu

### Gerçek Hayat Sorunları

---

**Sorun 1: Çift sayıda node — quorum sağlanamadı, cluster durdu**

```
Senaryo:
  Ekip: "4 node = 3 node'dan daha fazla redundancy" diye düşündü.
  etcd cluster: 4 node kuruldu.
  Network partition: 2-2 bölündü.

  Quorum hesabı:
    4 node → quorum = 4/2 + 1 = 3
    Her partition'da 2 node var.
    2 < 3 → hiçbir taraf quorum'u sağlayamıyor.
    Sonuç: Her iki taraf da yazma reddediyor → Kubernetes cluster yönetilemez hale geldi.

  3 node ile karşılaştırma:
    3 node → quorum = 3/2 + 1 = 2
    Network partition: 2-1 bölündü.
    2-node tarafı quorum var → çalışmaya devam eder.
    1-node tarafı quorum yok → yazma reddeder (split-brain önlendi).

Neden tek sayı:
  N node → quorum = N/2 + 1 (yuvarlak aşağı)
  3 node: 1 node kaybı tolere → quorum: 2/3
  4 node: 1 node kaybı tolere → quorum: 3/4 (4 ile aynı fault tolerance, ek maliyet)
  5 node: 2 node kaybı tolere → quorum: 3/5

  Formül: tolere edilen hata = (N-1)/2
  3 node: 1 hata. 4 node: 1 hata (4 yerine 3 koy, aynı fault tolerance).
  5 node: 2 hata. 6 node: 2 hata.
  Sonuç: Çift sayı → aynı fault tolerance, fazla maliyet + partition'da daha kötü.

Düzeltme:
  etcd cluster → 3, 5 veya 7 node.
  Production minimum: 3 node (1 node kaybı tolere).
  Yüksek kritiklik: 5 node (2 node kaybı tolere).
  7+ node: genellikle gereksiz (performans düşer, quorum latency artar).
```

---

**Sorun 2: etcd disk I/O yavaş — sık lider seçimi, Kubernetes kararsız**

```
Senaryo:
  Kubernetes: etcd 3-node cluster, HDD (SSD değil).
  Disk I/O yavaş: fsync 200ms sürüyor.
  Raft heartbeat default: 100ms.
  
  Sorun:
    Leader → log entry'yi disk'e yaz (fsync) → Followers'a AppendEntries gönder.
    Fsync 200ms + network → 300ms geciyor.
    Follower: heartbeat bekleme timeout = 500ms (5 × heartbeat).
    300ms yakın → zaman zaman timeout.
    Timeout → follower "lider çöktü" sanıyor → Candidate → lider seçimi başlıyor.
    Seçim devam ederken eski lider geri geliyor → yeni seçim.
    Kubernetes: lider sürekli değişiyor → API server hataları.

  etcd için SSD zorunluluğu:
    etcd dokümantasyonu: "SSD strongly recommended"
    WAL (Write-Ahead Log): her commit → fsync → SSD ile ~1ms, HDD ile ~10-200ms.
    Heartbeat interval: varsayılan 100ms.
    Disk gecikmesi heartbeat'in %30'unu aşmamalı.

  Düzeltme:
    SSD'ye geç (NVMe tercih).
    Heartbeat ve election timeout'u disk hızına göre ayarla:
    
    --heartbeat-interval=250   # 250ms (disk yavaşsa artır)
    --election-timeout=2500    # heartbeat × 10 (minimum)
    
    etcd benchmark çalıştır:
    etcd --data-dir=/var/lib/etcd benchmark --endpoints=...
    fio ile disk latency ölç: p99 fsync < heartbeat/3 olmalı.
    
    Ayrıca: etcd için dedicated disk (başka proseslerle paylaşma).
```

---

**Sorun 3: Split-brain — iki node kendini lider sandı**

```
Senaryo (consensus OLMAYAN sistem):
  3 primary-replica MySQL cluster, consensus yok, basit heartbeat ile failover.
  Network partition: Primary, Replica 1 ile bağlantısını kaybetti.
  
  T+0s:  Primary ↔ Replica1 bağlantısı kesildi.
  T+5s:  Replica1: "Primary öldü" → kendini Primary ilan etti (heartbeat timeout).
  T+5s:  Eski Primary: kendini hâlâ primary sanıyor (Replica2 ile bağlantısı var).
  
  Sonuç:
    İki Primary aynı anda yazar → split-brain.
    Client A: eski Primary'e yazar → x = 1.
    Client B: yeni Primary'e yazar → x = 2.
    Network düzeldi → tutarsız veri, reconciliation imkânsız.

Raft ile neden bu olmaz:
  Eski Primary ağdan kesildi:
    → Follower'lardan heartbeat ACK gelmiyor.
    → Ancak: artık quorum'u yok (1 node, 3'lü cluster).
    → Raft: "quorum alamıyorum → yazma reddediyorum" → leader step-down.
    → "Term expired" → eski leader follower'a düşer.
  
  Yeni election:
    Replica1 + Replica2: quorum var (2/3) → yeni lider seçilir.
    Eski lider geri geldiğinde: yeni term'i görür → follower olur.
    İki lider aynı anda olamaz → split-brain önlendi.

Sonuç:
  Consensus: quorum garantisi → minority partition yazamaz → split-brain imkânsız.
  Quorum tabanlı consensus olmayan HA çözümleri: split-brain riski taşır.
```

---

**Sorun 4: Raft log compaction yapılmadı — disk doldu, yeni node catch-up yapamadı**

```
Senaryo:
  etcd cluster: 1 yıldır çalışıyor, log compaction periyodik yapılmıyor.
  WAL dizini: 50GB → disk doldu → etcd yazma durdu → Kubernetes API Server hata.
  
  Ayrıca: Yeni etcd node eklendi, catch-up yapmak istedi.
  Log başından oynat → 1 yıllık log → saatler sürdü → bu sürede cluster kararsız.

Raft log compaction neden gerekli:
  Log: her yazma operasyonu append edilir → sonsuza büyür.
  Compaction (snapshot): belirli bir index'teki state machine durumunu snapshot al.
  Önceki log entry'leri silebilirsin — state snapshot yeterli.
  
  etcd snapshot:
  etcdctl snapshot save /backup/etcd-snapshot.db
  
  Otomatik compaction konfigürasyonu:
  --auto-compaction-mode=revision  # her N revision'da compaction
  --auto-compaction-retention=1000 # son 1000 revision tut, eskisini sil
  
  veya:
  --auto-compaction-mode=periodic  # zaman bazlı
  --auto-compaction-retention=1h   # 1 saatlik log tut
  
  Defragmentation (sonrası):
  etcdctl defrag  # compaction sonrası disk alanını geri al
  # Compaction: mantıksal silme. Defrag: fiziksel alan geri kazanımı.
  
  Yeni node catch-up:
  Snapshot transfer: log başından değil, son snapshot'tan başlar.
  etcd: snapshot threshold (--snapshot-count=10000) → 10000 commit'te snapshot al.
  Yeni node: snapshot indir → sonrasını log'dan oynat → çok daha hızlı.
```

---

### Mülakat Soruları — Quorum & Cluster

**Junior / Mid:**

1. Consensus nedir? Dağıtık sistemlerde neden gereklidir?

   > **Beklened:** Consensus: birden fazla node'un, aralarından bazıları çökmüş olsa bile tek bir değer üzerinde anlaşması. Gereksinim: dağıtık sistemde "doğru" cevap ne? Node'lar birbirinden bağımsız karar alırsa tutarsızlık (split-brain) oluşur. Kullanım yerleri: lider seçimi (Kafka broker, Kubernetes controller), config dağıtımı (etcd → K8s), distributed lock (ZooKeeper), replicated database (CockroachDB). Consensus olmadan: iki node aynı anda lider olabilir → veri tutarsızlığı. Consensus ile: quorum garantisi → sadece çoğunluk onaylayınca commit → tek doğru değer.

2. Quorum nedir? N node'lu cluster'da quorum formülü nedir?

   > **Beklened:** Quorum: bir kararın geçerli sayılması için gereken minimum node onayı. Formül: `quorum = N/2 + 1` (tam bölme, aşağı yuvarlama). 3 node: 2, 5 node: 3, 7 node: 4. Neden N/2+1: iki ayrı quorum kümesi asla oluşamaz → split-brain imkânsız. 3 node: quorum=2 → her iki grup aynı anda 2 alamaz (2+2=4 > 3). Fault tolerance: `(N-1)/2` node kaybı tolere edilir. 3 node: 1, 5 node: 2. Neden tek sayı: çift sayı fault tolerance artırmaz ama quorum eşiğini yükseltir.

3. etcd, ZooKeeper ve Consul Raft'ı nasıl kullanır?

   > **Beklened:** etcd: Kubernetes'in beyni. API Server, Scheduler, Controller Manager → etcd'de config/state saklar. Raft üzerine kurulu strong consistency. K8s: "etcd'ye git" = "cluster'ın gerçeği bu." ZooKeeper: Kafka'nın eski coordination layer'ı (broker seçimi, topic metadata). ZAB protokolü kullanır (Raft benzeri, daha eski). Kafka 3.x: KRaft ile ZooKeeper bağımlılığı kalktı. Consul: service discovery, health check, KV store. Raft kullanır. Ortak: hepsi Raft/ZAB ile strong consistency, leader-based, quorum commit. Uygulama geliştiricisi genellikle bu sistemlerin üstünde çalışır, Raft'ı direkt implemente etmez.

---

**Senior / Architect:**

4. Raft cluster'ını neden 3, 5 veya 7 node olarak kurarız? 4 veya 6 neden tercih edilmez?

   > **Beklened:** Fault tolerance: `(N-1)/2`. 3 → 1 hata. 4 → 1 hata (quorum=3). 5 → 2 hata. 6 → 2 hata (quorum=4). 4 ile 3 aynı fault tolerance; 4'ün quorum'u yüksek (3/4 vs 2/3). Network partition: 4 node, 2-2 bölünürlerse iki taraf da quorum alamaz → cluster tamamen durabilir. 3 node, 2-1 bölünürse 2'li taraf çalışır. Çift sayı: daha kötü partition davranışı + daha yüksek quorum maliyeti. Yorum: 4 node kurmak 3 node'dan daha iyi değil, daha kötü. Her zaman tek sayı.

5. Raft cluster boyutu 7'den fazla yapılır mı? Tradeoff nedir?

   > **Beklened:** Teorik olarak yapılabilir — pratik olarak nadiren mantıklı. 7 node: quorum=4, 3 hata tolere. 9 node: quorum=5, 4 hata tolere. Maliyet: quorum arttıkça her commit daha fazla node beklenmesi → latency artar. 7 node cluster: 4 node'un yazması beklenir → yazma latency 3 node'dan ~2× daha yüksek. Kullanım: çok kritik metadata store, coğrafi dağıtım (3 DC × 1 node + 2 tiebreaker). Genellikle: 3 node production minimum, 5 node yüksek kritiklik. 7+: nadir, özel gereksinim. etcd resmi önerisi: 5 node maksimum.

---

## Bölüm 2: Raft Lider Seçimi

### Gerçek Hayat Sorunları

---

**Sorun 5: Lider seçimi sırasında yazma kaybı yaşandı — commit vs uncommit**

```
Senaryo:
  3 node Raft cluster. Leader (Node A) bir log entry'yi sadece kendine yazdı
  ama quorum'a çoğaltmadan çöktü.

  Timeline:
    T+0: Client → Leader (Node A): "SET x = 5"
    T+1: Node A log'a yazdı (index 5, uncommitted)
    T+2: Node A → Followers'a AppendEntries gönderdi (ama ağ gecikmesi)
    T+3: Node A çöktü — AppendEntries henüz gelmedi
    T+4: Node B, Node C lider seçimi yaptı. Node B leader.
    T+5: Node B log'unda index 5 yok → Client "x = 5" yazdım sanıyor.

  Ne oldu?
    Index 5, "SET x = 5": commit EDİLMEDİ (quorum onaylamadı).
    Raft garantisi: commit edilmemiş entry kaybolabilir — bu normaldir.
    Client retry etmeli → Yeni lider'a gönder → quorum onayı → commit → güvenli.

  Raft garantisi NE SAĞLAR:
    COMMIT EDİLMİŞ entry asla kaybolmaz.
    "Committed" = quorum (2/3) onayladı → tüm gelecek liderların log'unda var.
    Uncommitted: kaybolabilir — client retry ile çözülür.

  At-least-once client retry:
    Client: timeout → "commit edildi mi?" bilinmiyor → idempotency key ile retry.
    Duplicate işlem değil: idempotency garantisi.
    "Read-your-writes" için: "committed" onayı gelene kadar client beklemeli.
```

---

**Sorun 6: Sık lider seçimi — election timeout çok kısa ayarlandı**

```
Senaryo:
  Kubernetes üzerinde etcd, aggressive timeout ayarları:
  --heartbeat-interval=50ms
  --election-timeout=250ms
  
  Yoğun yük altında:
    Network latency: 80ms (normal: 10ms).
    Heartbeat 50ms → 80ms latency → heartbeat zamanında gelmiyor.
    Follower: 250ms timeout doldu → "lider öldü" → candidate.
    Aslında lider sağlıklı, sadece yavaş.
    Gereksiz lider seçimleri → Kubernetes API instability.

Doğru timeout ayarı:
  Election timeout >> Heartbeat interval >> RTT (round-trip time)
  
  Kural:
    heartbeat-interval = max_RTT × 2  (network spike'ları absorbe et)
    election-timeout   = heartbeat-interval × 10 (yeterli buffer)
  
  Örnek (yüksek yük, cloud):
    Network RTT: 10ms (p99: 50ms)
    heartbeat-interval = 100ms (50ms × 2)
    election-timeout   = 1000ms (100ms × 10)
  
  Random election timeout neden?
    election-timeout=1000ms → her follower [1000ms, 2000ms] arası random timeout.
    Genellikle bir follower önce timeout → candidate → oy toplar → kazanır.
    Birden fazla candidate aynı anda olursa: oy bölünür → hiçbiri quorum alamaz.
    Yeni random timeout → biri daha önce timeout → seçim tamamlanır.
    Convergence garantili: random + retry.

  etcd benchmark ile validate et:
  etcd --data-dir=/tmp/bench benchmark --endpoints=...
  # p99 istek süresi < election-timeout/10 olmalı
```

---

### Mülakat Soruları — Lider Seçimi

**Junior / Mid:**

6. Raft'ta üç node state nedir? Aralarındaki geçişler nasıl olur?

   > **Beklened:** Follower: başlangıç state'i. Liderden heartbeat bekler. Election timeout dolunca → Candidate. Candidate: oy toplar. RequestVote gönderir. Quorum oy alırsa → Leader. Daha yüksek term'li mesaj gelirse → Follower. Timeout dolunca (oy bölündü) → yeni term, tekrar Candidate. Leader: heartbeat (empty AppendEntries) gönderir, yazma operasyonlarını yönetir. Yüksek term'li mesaj gelirse → Follower (eski lider olduğunu anlar). Tek yönlü değil: Leader → Follower olabilir (yüksek term geldiğinde). Follower → Leader direkt gidemez.

7. Raft'ta "term" nedir? Neden önemlidir?

   > **Beklened:** Term: monoton artan lojik saat / dönem numarası. Her lider seçiminde artar. Amacı: eski liderden gelen "stale" mesajları reddet. Örnek: Node A term=3'de lider, çöktü, term=4'de yeni lider seçildi. Node A geri geldi, hâlâ term=3 mesajları gönderiyor. Diğer node'lar: "term=3 < 4 → eski, reddediyorum." Node A: "term=4 görüyorum → ben artık eski liderim → follower oluyorum." Güvenlik garantisi: aynı term'de iki lider olamaz (quorum). Farklı term'deki eski lider → zaten term kontrolüyle reddedilir.

8. Raft lider seçiminde oy verme kuralları nelerdir? "En güncel log" ne demektir?

   > **Beklened:** Oy verme koşulları — tümü sağlanmalı: (1) Adayın term'i ≥ benim term'im. (2) Bu term'de henüz oy vermedim. (3) Adayın log'u en az benim kadar "up-to-date." Up-to-date karşılaştırma: önce son log entry'nin term'ini karşılaştır. Yüksek term → daha güncel. Aynı term → daha yüksek index (daha uzun log) → daha güncel. Neden: commit edilmiş entry'leri içeren node lider seçilmeli. Commit = quorum onayı = o node'lardan birinde mutlaka var. Güncel log şartı → commit edilmiş entry'ler kaybolmaz (Leader Completeness garantisi).

---

**Senior / Architect:**

9. Raft lider seçimi sırasında "yazma" ne olur? Client nasıl davranmalıdır?

   > **Beklened:** Seçim sırasında (lider çöktü, yeni lider henüz seçilmedi): hiçbir node leader olmadığı için yazma kabul edilemez. Client: timeout → retry. Yeni lider seçilince (genellikle 150-600ms): yazma kabul edilir. Uncommitted entry'ler: eski lider quorum'a çoğaltmadan çöktüyse kaybolabilir. Client retry → idempotency key → duplicate yazma değil. "Read-your-writes": client yazdı, hemen okursa → yeni lider'dan oku (eski follower stale okuyabilir). Linearizability: etcd ile → her okuma commit edilmiş son değeri garantiler. Dağıtık sistem ders: commit ≠ client "OK" aldı. "OK" gelmemişse → idempotency key ile retry.

---

## Bölüm 3: Log Replication & Güvenlik Garantileri

### Gerçek Hayat Sorunları

---

**Sorun 7: Follower'dan okuma — stale veri döndü**

```
Senaryo:
  3-node etcd cluster.
  Uygulama: read latency düşürmek için follower'lardan okuma yaptı.
  
  T+0: Leader'a "SET config.featureFlag = true" yazıldı.
  T+1: Leader commit etti, quorum (Node A, Node B) onayladı.
       Node C (follower) henüz sync olmadı.
  T+2: Uygulama Node C'den "GET config.featureFlag" okudu → false döndü!
       Node C hâlâ replikasyonu almadı.

Raft log replication:
  Commit: quorum (2/3) yazdı → "committed."
  Follower sync: heartbeat'te veya hemen sonra → ama küçük gecikme var.
  Follower'dan okuma = "eventually consistent" okuma.
  Stale window: genellikle ms seviyesinde ama kritik senaryolarda sorun.

Çözüm 1: Leader'dan oku (linearizable read)
  etcd: etcdctl get --endpoints=leader-endpoint key
  Leader: "ReadIndex" mekanizması — "benden daha güncel lider var mı?" kontrol.
  Okuma öncesi quorum'a "hâlâ lider miyim?" sorar → evet → güncel değeri döndür.
  Latency: +1 network round-trip ama doğruluk garantili.

Çözüm 2: Serializable read (etcd)
  --consistency=s → follower'dan okuyabilir, daha hızlı ama stale mümkün.
  --consistency=l (default) → linearizable, leader'dan veya ReadIndex.

Çözüm 3: Watch mekanizması
  etcdctl watch config.featureFlag
  → Config değişince anında bildirim → stale okuma yok.

Sonuç:
  "Follower'dan oku" = eventual consistency.
  Kritik config, distributed lock → leader'dan oku (linearizable).
  Read replica, analytics → follower okuma tolere edilebilir.
```

---

**Sorun 8: Eski lider geri döndü, geçersiz verileri yazmaya çalıştı**

```
Senaryo:
  3-node cluster. Node A lider (term=3).
  Node A ağdan kesildi → Node B yeni lider (term=4).
  Node A: ağ bölünmesinden habersiz, hâlâ lider sanıyor.
  Client: Node A'ya yazma isteği gönderdi (eski route).
  Node A: "SET x = 99" → log'a yazdı.

  Ne olur?
    Node A: quorum'a AppendEntries gönderiyor.
    Node B, Node C: term=4 mesajları alıyorlar, term=3 mesajı reddediyorlar.
    Node A: "AppendEntries reddedildi, daha yüksek term görüyorum → follower ol."
    Node A: follower oldu, "SET x = 99" uncommitted → overwrite edilecek.

  Raft'ın koruması:
    Term: Node A term=3 → Node B, Node C term=4 → eski mesaj reddedildi.
    Node A quorum alamadı → commit edemedi → "SET x = 99" kayboldu.
    Client: timeout aldı → retry → Node B'ye (yeni leader) gönderdi → başarı.

  "Stale leader" durumu:
    Ağdan kesilen lider: quorum'u yok → yazamaz → otomatik follower.
    Raft: minority partition asla yazamaz → split-brain imkânsız.
    
  Client tarafı:
    Timeout: hangi node'un leader olduğunu bilmiyor olabilir.
    etcd client: cluster endpoint listesi → aktif leader'ı keşfeder.
    Client redirect: follower → "ben leader değilim, şuraya git."
```

---

### Mülakat Soruları — Log Replication

**Junior / Mid:**

10. Raft'ta bir log entry ne zaman "commit edilmiş" sayılır?

    > **Beklened:** Commit: quorum (N/2 + 1) node log'a yazdığında ve leader bu durumu teyit ettiğinde. Adımlar: (1) Client → Leader: yazma isteği. (2) Leader log'a yazar (uncommitted). (3) AppendEntries → tüm Followers. (4) Quorum (en az 2/3 için 2) follower ACK → Leader commit. (5) Leader state machine'e uygular, client'a OK. (6) Sonraki heartbeat'te Followers da commit. Garantisi: commit edilmiş entry asla kaybolmaz — tüm gelecekteki liderların log'unda bulunacak (Leader Completeness). Commit edilmemiş: eski lider çökerse kaybolabilir — normaldir.

11. Raft log'unda "Log Matching" özelliği nedir?

    > **Beklened:** Log Matching: iki log'da aynı index ve term'e sahip entry varsa, bu index'e kadar tüm önceki entry'ler de aynıdır. Neden: AppendEntries'te leader önceki entry'nin (index, term) bilgisini gönderir. Follower: "Bu önceki entry bende var mı ve term eşleşiyor mu?" Eşleşmiyorsa: reddeder → leader daha eski entry'leri gönderir → follower'ı yakalar. Bu inductive garantinin sonucu: aynı (index, term) = öncesi garantili aynı. Kullanım: yeni lider follower'ları tutarlı hale getirirken bu özelliği kullanır.

---

**Senior / Architect:**

12. Yeni lider seçilince follower'lardaki tutarsız log entry'leri nasıl düzeltilir?

    > **Beklened:** Durum: Node A lider oldu, Node B ve C'de tutarsız (uncommitted) entry'ler var. Raft çözümü: Leader, her follower için `nextIndex[]` tutar. `nextIndex[B]` = "B'ye bir sonraki göndereceğim index." AppendEntries: önce `nextIndex-1`'deki (prevLogIndex, prevLogTerm) gönder. Follower "uyuşmuyor" → leader `nextIndex`'i azaltır, tekrar dener. Bulunca: follower o noktadan itibaren leader'ın log'u ile overwrite eder. Eski uncommitted entry silinir. Önemli: sadece uncommitted entry'ler overwrite edilir. Committed entry'ler zaten tüm gelecek liderlarda var (Leader Completeness) → overwrite edilmez.

13. Raft'ın 4 güvenlik garantisini açıkla. Hangisi en kritiktir?

    > **Beklened:** (1) Election Safety: aynı term'de en fazla bir lider (quorum ile garanti). (2) Log Matching: aynı (index, term) → önceki tümü aynı. (3) Leader Completeness: commit edilmiş her entry tüm gelecek liderların log'unda. (4) State Machine Safety: aynı index'e farklı iki node farklı komut uygulamaz. En kritik: State Machine Safety — bu ihlal edilirse farklı node'lar farklı state'e girer → tutarsızlık → sistem güvensiz. Bu garantiyi sağlayan: (1)+(3) birlikte. Lider seçiminde "güncel log" şartı → (3) → (4). Zincir: quorum oy → güncel lider → commit edilmiş entry'ler güvende → state machine tutarlı.

---

## Bölüm 4: Paxos & Karşılaştırma

### Mülakat Soruları

**Junior / Mid:**

14. Paxos ve Raft'ın temel farkı nedir? Neden modern sistemler Raft'ı tercih eder?

    > **Beklened:** Paxos (1989, Lamport): teorik olarak minimal — 2 aşama (Prepare/Accept), 3 rol (Proposer/Acceptor/Learner). Anlaşılması çok zor. Multi-Paxos: lider seçimi + log replication ayrı mekanizmalar → implementasyonu zor, hata eğilimli. Raft (2014, Ongaro): "Understandable consensus" hedefiyle tasarlandı. Lider seçimi, log replication, membership change tek çatı altında. Açık implementation spec → tutarlı implementasyon. Neden Raft: etcd, CockroachDB, TiKV, Consul, Kafka KRaft → hepsi Raft. Google: tarihsel Paxos (Chubby, Spanner) ama yeni sistemler Raft'a geçiyor. Pratik fark: Paxos daha az yazma round-trip ama implementasyon riski yüksek.

15. ZooKeeper hangi consensus protokolünü kullanır? Kafka ile ilişkisi nedir?

    > **Beklened:** ZooKeeper: ZAB (ZooKeeper Atomic Broadcast) — Raft öncesi, benzer fikir ama farklı implementasyon. Leader-based, quorum, log replication. Kafka eski mimari: ZooKeeper → broker leader election, topic metadata, controller seçimi. Sorunlar: ek bağımlılık, 200K partition limiti, controller election saniyeler. Kafka KRaft (3.3+ GA): ZooKeeper'ı kaldırdı, Raft'ı Kafka içine entegre etti. Kafka broker'ları hem Raft hem de normal broker rolü. Avantajlar: tek sistem, milyonlarca partition, ms'lik controller election, basit operasyon. Sonuç: ZooKeeper miras, KRaft modern — yeni Kafka kurulumu KRaft.

---

**Senior / Architect:**

16. CAP teoremi ile consensus algoritmaları ilişkisi nedir?

    > **Beklened:** Raft/Paxos → CP sistemleri (Consistency + Partition Tolerance). Network partition durumunda: minority partition yazma reddeder → Availability feda. Örnek: etcd 3 node, 2-1 bölünme → 1-node taraf "unavailable." Majority taraf hizmet verir. Neden CP: consensus'un amacı "tek doğru değer" → tutarsız okuma kabul edilemez. AP alternatifleri: Cassandra, DynamoDB → quorum yok, eventual consistency, partition'da her iki taraf yazar (reconciliation sonra). Seçim: distributed config, leader election, distributed lock → CP zorunlu (tutarsız config felakettir). High read throughput, eventual consistency tolere → AP. PACELC: partition yokken bile latency vs consistency tradeoff.

17. Consensus'u kendin implement etmeden nasıl kullanırsın? Hangi sistemi hangi senaryoda seçersin?

    > **Beklened:** Doğrudan implement etme: Raft edge case'leri (network partition + restart + log + membership) inanılmaz karmaşık. Yıllar sürer, yine de bug çıkar. Hazır sistemler: etcd: Kubernetes ekosistemi, config/metadata, distributed lock (etcd lease). ZooKeeper: Kafka (eski), Hadoop ekosistemi, matura. Consul: service discovery + health check + KV store + Raft. Apache Ratis (Java): kendi replicated state machine isteyenler için Raft library. CockroachDB/TiKV: embedded Raft, distributed database. Senaryo seçim: Kubernetes'te → etcd zaten var, distributed lock için etcd kullan. Kafka koordinasyonu → KRaft. Servis kaydı/discovery → Consul. Kendi replicated state machine → Ratis.

---

## Karma — Architect Seviyesi

18. **"5 node etcd cluster yönetiyorsun. Bir node'u maintenance'a almak istiyorsun. Ne kontrol edersin?"**

    > **Beklened:** (1) Cluster sağlığı: `etcdctl endpoint health --cluster` → tüm node'lar sağlıklı mı? (2) Lider kim: `etcdctl endpoint status --cluster` → kim leader? (3) Maintenance alınacak node lider mi? Evet → önce lider transferi: `etcdctl move-leader <follower-id>`. (4) Quorum kontrolü: 5 node, 1 alındı → 4 kaldı → quorum=3, 4 ≥ 3 → sağlam. (5) Node'u cluster'dan kaldır: `etcdctl member remove <id>` → etcd 4-node cluster olarak devam. (6) Maintenance yap. (7) Geri ekle: `etcdctl member add` → yeni node katılır, snapshot ile sync. (8) Sağlık kontrol: `etcdctl endpoint status --cluster` → yeni node sync'te mi? (9) Cluster member sayısı tekrar 5.

19. **"Dağıtık sisteminde lider seçimi gerekiyor. etcd mi, ZooKeeper mı, kendin mi implement etmelisin?"**

    > **Beklened:** Kendi implement etme: asla (production için). Edge case'ler çok karmaşık, yıllar sürer. etcd tercih et: Kubernetes ekosistemindeysen — zaten var. etcd Lease + Election API: basit, battle-tested. `Campaign()` → lider ol. `Resign()` → bırak. TTL tabanlı lease: lider çökünce lease expire → otomatik yeni seçim. ZooKeeper: Kafka/Hadoop ekosistemi, halihazırda var. Ephemeral node: lider node → session expire → sil → yeni seçim. Consul: multi-datacenter, service mesh ile birlikte. Kararlar: Kubernetes var → etcd. Kafka var → ZooKeeper (veya KRaft). Yeni sistem, cloud-native → etcd. Eski Java ekosistemi → ZooKeeper Curator (election abstraction). Hiçbiri uygun değil → Apache Ratis (Java Raft library).

20. **"Bir engineer 'etcd'yi 2 node'a düşüreceğiz, kaynak tasarrufu yapalım' dedi. Ne yanıt verirsin?"**

    > **Beklened:** Kesinlikle hayır. 2 node Raft cluster'ı pratik olarak işlevsizdir. Quorum: 2/2+1 = 2. Yani her iki node da ayakta olmalı. 1 node çökerse → quorum yok → cluster durur. 3-node cluster, 1 node çöküşte çalışır (quorum=2, 2≥2). 2-node: 0 fault tolerance — neredeyse single node gibi ama 2× kaynak kullanıyor. Önerilen: kaynak tasarrufu → 3 node → her birini küçült (küçük VM). 2 node'dan daha iyi: 1 node (en azından complexity yok). Production etcd: minimum 3 node. Kubernetes: etcd 2 node → controller manager, scheduler seçimi de bozulur → cluster tamamen yönetilemez. Matematiksel argüman + gerçek senaryo ile ekibe açıkla.
