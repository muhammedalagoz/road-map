# 05e — Split-Brain, Quorum & PACELC: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Split-Brain

### Gerçek Hayat Sorunları

---

**Sorun 1: Primary-replica MySQL, witness yok — network partition'da iki primary**

```
Senaryo:
  MySQL: 1 Primary (DC-A) + 1 Replica (DC-B). DC'ler arası link koptu.

  T+0s:  DC-A ↔ DC-B bağlantısı kesildi.
  T+5s:  Replica (DC-B): "Primary'e ulaşamıyorum → Primary çöktü" → kendini primary ilan etti.
  T+5s:  Primary (DC-A): "Replica çöktü" → yazmayı sürdürdü.

  Sonuç:
    DC-A Primary: kullanıcı A siparişi verdi → order_id=1001 oluştu.
    DC-B Primary: kullanıcı B siparişi verdi → order_id=1001 oluştu (aynı auto-increment!).
    T+20s: Bağlantı düzeldi → iki farklı kayıt aynı ID ile çakıştı → veri bozulması.

Neden oldu:
  2 node: her partition kendi başına quorum sayıyor (1/1).
  Witness yok → hangisinin "gerçek primary" olduğuna karar verecek 3. taraf yok.
  Consensus protokolü yok (Raft/Paxos değil, basit heartbeat failover).

Düzeltme seçenekleri:

  1. Witness node (3. AZ):
     Primary + Replica + Witness (oy verir, veri tutmaz).
     DC-A ↔ DC-B bağlantısı koparsa: Witness hangi tarafla bağlıysa → o taraf quorum.
     AWS RDS Multi-AZ: bu yaklaşımı kullanır.

  2. Orchestrated failover (konsensüs tabanlı):
     etcd/ZooKeeper: primary kilidini tutar.
     Eski primary quorum'u kaybedince kilit düşer → yeni seçim.
     Patroni (PostgreSQL HA): etcd/Consul üzerinde consensus → Patroni bunu yapar.

  3. STONITH (Shoot The Other Node In The Head):
     Network izolasyonu algılanınca → diğer node'u güç keserek durdur.
     Her iki taraf birbirini STONITH yaparsa → ikisi de kapanır (availability kaybı).
     Ama split-brain olmaz.

  4. Synchronous replication + Majority:
     PostgreSQL synchronous_commit = remote_apply:
     Primary: replica onaylamadan commit döndürmez.
     DC-B kopunca: Primary asılı kalır (availability feda, split-brain önlenir).
```

---

**Sorun 2: Fencing token yok — zombie lider veriyi bozdu**

```
Senaryo:
  Kafka controller seçimi (ZooKeeper döneminde).
  Controller-1 seçildi (epoch=5). Ağdan geçici koptu (GC pause, 30s).
  ZooKeeper: session expire → Controller-2 seçildi (epoch=6).

  T+30s: Controller-1 GC'den uyandı. Hâlâ "ben controller'ım" sanıyor.
  T+31s: Controller-1 → Broker-A'ya: "partition-X'in leader'ı sen ol" (epoch=5).
  T+31s: Controller-2 → Broker-A'ya: "partition-X'in leader'ı sen OLMA" (epoch=6).
  T+31s: Broker-A epoch kontrol: epoch=5 < epoch=6 → Controller-1'in mesajını reddetti.

Neden çalıştı? Epoch tabanlı fencing:
  Her controller epoch monoton artar.
  Broker: gelen epoch'u tutar, daha düşük epoch'lu mesajı reddeder.
  Controller-1 (zombie): tüm mesajları reddediliyor → etkisiz.

Fencing token olmayan senaryo (aynı sistemi Redis lock ile yapalım):
  Lock sahibi node TTL dolunca lock'u kaybetti ama bilmedi.
  Eski node → DB'ye yazdı → yeni node'un verisini ezdi.
  Çözüm: lock alınırken token al, DB'ye yazarken token gönder.
  DB: eski token gelirse reddet (WHERE last_token < new_token).

Genel kural:
  Fencing token = "gücüm var" kanıtı.
  Sistemin her katmanı token'ı doğrulamalı → lock mekanizması yetmez.
  "Process askıya alınabilir" varsayımı → fencing olmadan hiçbir lock güvenli değil.
```

---

**Sorun 3: Elasticsearch split-brain — two masters, index corruption**

```
Senaryo:
  Elasticsearch 4 node cluster. Yanlış ayar:
    discovery.zen.minimum_master_nodes = 1  (eski ES sürümü)
  
  Ağ bölünmesi: 2-2 partition.
  Her iki group: "minimum 1 master gerekli" → ikisi de master seçti.
  
  Sonuç:
    Group-A: index_A oluşturdu.
    Group-B: index_A oluşturdu (farklı shard dağılımı).
    Bağlantı düzeldi → iki farklı cluster state → cluster kilitlendi.
    Manuel müdahale: birinin state'ini sil, diğerini "doğru" kabul et.
    Veri kaybı: silinene ait indexlerdeki belgeler gitti.

Doğru ayar (eski ES):
  minimum_master_nodes = (N/2) + 1
  4 node → minimum_master_nodes = 3
  2-2 bölünmede: hiçbiri 3 master alamaz → ikisi de master seçemez.
  Availability kaybı (yazma durur) ama split-brain önlenir.

Modern Elasticsearch (7.x+):
  cluster.initial_master_nodes ile başlatılır.
  Quorum otomatik hesaplanır → manual minimum_master_nodes kaldırıldı.
  Raft tabanlı koordinasyon (voting configuration).

Ders:
  Quorum ayarını cluster boyutuna göre manuel yapıyorsan:
  Node ekle/çıkarırken ayarı güncelle (unutursan yanlış quorum).
  Otomatik quorum hesaplayan sistemler (Raft) daha güvenli.
```

---

**Sorun 4: Sloppy quorum — W+R > N garantisi bozuldu, stale veri döndü**

```
Senaryo:
  Cassandra RF=3, CL=QUORUM (W=2, R=2). W+R=4 > N=3 → strong consistency beklendi.

  T+0:  Replica-1, Replica-2, Replica-3 sağlıklı.
  T+1:  Replica-1 ve Replica-2 geçici ağ sorunuyla ulaşılamaz.
  T+2:  Yaz isteği: CL=QUORUM ama Replica-1, Replica-2 yok.
        Sloppy quorum devreye girdi: Replica-3 + Hinted Handoff Node-X'e yaz.
        Yazma başarılı döndü (2 node yazdı: Replica-3 + Node-X).
  T+3:  Oku isteği: CL=QUORUM → Replica-1 geri geldi + Replica-3.
        Replica-1: eski veri (yazma henüz gelmedi, hinted handoff bitmedi).
        Replica-3: güncel veri.
        "Newest timestamp wins" → Replica-3 kazandı → güncel döndü.
  T+4:  Ama başka senaryoda: Replica-3 + Replica-1 okundu, Node-X (yeni yazan) okunmadı.
        İkisi de eski veri → stale döndü!

Sloppy quorum neden W+R>N garantisini bozar:
  W+R>N: "asıl N node"dan W yaz, R oku → overlap garantili.
  Sloppy: "asıl" node yerine "başka" node'a yazıldı → overlap artık garantili değil.
  DynamoDB ve Cassandra: sloppy quorum ile availability artırır, consistency feda.

Ne zaman sloppy quorum kabul edilir:
  Shopping cart, kullanıcı session: kısa süre stale tolere.
  Banka, stok: sloppy quorum kullanma → CL=ALL veya RDBMS.

Cassandra'da strict quorum:
  Sloppy quorum davranışı: Cassandra her zaman hinted handoff yapar.
  Engel: hinted_handoff_enabled = false (dikkat: availability azalır).
  Ya da: CL=ALL → tüm replica onayı → sloppy quorum devreye giremez.
```

---

## Bölüm 2: Quorum Reads & Writes

### Gerçek Hayat Sorunları

---

**Sorun 5: CL=ONE okuma + CL=ONE yazma — replikasyon gecikmesinde stale veri**

```
Senaryo:
  Cassandra RF=3, CL=ONE her yerde.
  Kullanıcı profil güncelledi → CL=ONE → Replica-1'e yazıldı.
  Hemen oku → CL=ONE → Replica-2'den okundu.
  Replica-2: replikasyon henüz gelmedi (100ms gecikme).
  Kullanıcı: "az önce güncellediğim profil neden eski görünüyor?"

W+R hesabı:
  W=1, R=1 → W+R=2, N=3 → 2 < 3+1 → overlap garantisi yok.
  Stale read mümkün.

Seçenekler:

  1. CL=QUORUM (W=2, R=2):
     W+R=4 > 3 → her okuma en az bir yazan node'u görür.
     Latency artar (iki replica bekle).

  2. CL=LOCAL_QUORUM (çok bölgeli):
     Cross-DC latency'den kaçın, yerel quorum.
     Aynı DC'de W+R>N → güncel veri.

  3. Read-your-writes: Client-side sticky:
     Yazan node'dan oku (coordinator sticky).
     "Az önce yazdığım node'dan oku" → stale yok.
     Zor: birden fazla client koordinasyonu.

  4. Materialize + cache invalidation:
     Yaz → Redis'e de yaz (cache-aside).
     Oku → önce Redis → Redis'te yoksa Cassandra (CL=QUORUM).
     Kullanıcı kendi güncellemesini hemen görür.

Uygulama kararı:
  Tüm operasyon için tek CL seçme → endpoint başına karar ver.
  Profil güncelleme (kritik): QUORUM.
  Ürün kataloğu okuma (stale tolere): ONE.
```

---

### Mülakat Soruları — Split-Brain & Quorum

**Junior / Mid:**

1. Split-brain nedir? Neden dağıtık sistemlerde tehlikelidir?

   > **Beklened:** Split-brain: ağ bölünmesi nedeniyle cluster'ın birden fazla bağımsız parçaya ayrılması ve her parçanın kendini "birincil" (lider/primary) sanması. Tehlike: aynı veri iki ayrı yerden farklı güncellenir → bağlantı düzelince hangi verinin "doğru" olduğu belirsiz → çakışma. Banka örneği: bakiye 1000 TL, iki partition ayrı ayrı 500 TL + 700 TL çekerse → toplamda 1200 TL çekildi, hesapta 1000 TL. Quorum çözümü: N/2+1 oy olmadan yaz → minority partition yazamaz → split-brain imkânsız. Fencing token: eski lider sisteme yazmaya çalışırsa → düşük token → reddedilir.

2. Quorum neden split-brain'i önler? Matematiksel açıklama.

   > **Beklened:** Quorum = N/2 + 1 (tamsayı bölme, yukarı yuvarlama). Özellik: iki ayrı quorum kümesi oluşturulamaz. Kanıt: iki quorum kümesi boyutlarının toplamı: (N/2+1) + (N/2+1) = N+2 > N. N node'dan büyük bir toplam → iki küme en az 1 ortak node paylaşmalı. Ortak node: aynı anda iki farklı lidere oy veremez → iki lider aynı anda quorum alamaz. 5 node örneği: quorum=3. 2-3 bölünmesi: 2 taraf 3 oy alamaz, 3 taraf 3 oy alabilir. 3-2 tek quorum çalışır → split-brain imkânsız. Uygulama: etcd/Raft: commit için quorum şart. Cassandra QUORUM: aynı mantık.

3. Witness node (arbitrator) nedir? Ne zaman kullanılır?

   > **Beklened:** Witness: oy verir ama veri tutmaz — "tiebreaker" rolü. Neden gerekli: 2 node cluster → her biri quorum için diğerine ihtiyaç duyar → partition'da ikisi de çalışamaz (veya ikisi de çalışır → split-brain). 3. node (witness) quorum'u mümkün kılar: 2 node'dan biri + witness = 2/3 quorum. Maliyet: witness küçük/ucuz sunucu olabilir, veri replike etmez. Kullanım: AWS RDS Multi-AZ (Primary + Standby + witness 3. AZ'de). MongoDB 3-node replica set'te bir arbiter. 2 ana node + 1 arbitrator → 3 node quorum → 1 node kaybı tolere edilir, witness veri tutmaz. Dikkat: witness da çöküşe uğrayabilir → production'da witness'ı da monitörle.

---

**Senior / Architect:**

4. Fencing token olmadan distributed lock veya lider seçimi neden güvensiz?

   > **Beklened:** Senaryo: Process A lock aldı (token=33). GC pause 40s → TTL doldu → B lock aldı (token=34). A GC'den uyandı, "lock bende" sanıyor → DB'ye yazdı. B'nin verisini ezdi. Fencing token çözümü: lock alınca monoton artan token döner. DB'ye yazarken token gönder. DB: "WHERE last_token < 34 → UPDATE kabul et, 33 ile gelen UPDATE reddedilir." ZooKeeper: zxid doğal fencing token. Kafka: controller epoch → broker eski epoch'lu mesajı reddeder. Genel kural: dağıtık sistemde process GC pause, ağ gecikmesi veya OS scheduling nedeniyle beklenmedik süre "uyuyabilir." Bu sürede başka process devreye girebilir. Fencing: uyuyan process'in "gücünü" iptal eder. Lock mekanizması + fencing token birlikte = gerçek güvenlik.

5. Cassandra'da W+R>N formülü strong consistency sağlar mı? Sınırları nelerdir?

   > **Beklened:** W+R>N: yazma ve okuma setlerinin en az 1 ortak node içermesi garantisi → o node güncel veriyi biliyor. N=3, W=2, R=2: W+R=4>3 → overlap garantili → strong consistency. Sınırlar: (1) Sloppy quorum: asıl N node yerine "hinted handoff" node'a yazılırsa → overlap garantisi bozulur. Hinted handoff node okuma setine dahil olmayabilir → stale. (2) Aynı anda concurrent write: iki client aynı key'e yazdı → timestamp ile last-write-wins → "en son yazma" hangisi? Network skew nedeniyle belirsiz olabilir. (3) Node failure sonrası: yazma quorum'u aldı, replica down → okunan quorum farklı node'larsa eski veri dönebilir (replikasyon henüz yayılmadı). Çözüm: CL=ALL → tüm replica onayı → sloppy yok, daha güvenli ama daha yavaş ve availability düşük.

---

## Bölüm 3: PACELC

### Gerçek Hayat Sorunları

---

**Sorun 6: PA/EL seçildi ama iş gereksinimi PC/EC gerektiriyordu**

```
Senaryo:
  Flash sale sistemi: "stok 100 adet" kampanyası.
  Ekip: "hız önemli, Cassandra CL=ONE" kullandı (PA/EL davranışı).

  Flash sale başladı: 10.000 eş zamanlı istek.
  CL=ONE: sadece 1 replica yazdı, diğerleri henüz sync olmadı.
  Her istek "stok var" gördü (eski stok=100 değeri) → 500 sipariş onaylandı.
  Stok: 100 → 500 sipariş → 400 overselling.

  Gerçek gereksinim: stok azaltma → PC/EC → strong consistency.

PACELC kararı nasıl verilmeli:
  Soru 1: Partition durumunda ne olur?
    CP (PC): minority partition yazma reddeder → bazı kullanıcılar hata alır.
    AP (PA): tüm partition'lar yazar → tutarsızlık.
    Stok: overselling felakettir → CP seç.

  Soru 2: Normal çalışmada (partition yok) latency mi, consistency mi?
    EC (Consistency): her yazma quorum bekler → yüksek latency.
    EL (Low Latency): CL=ONE, hızlı dön → eventual.
    Stok: EC → doğru sayma zorunlu.

  Flash sale için doğru seçim:
    PC/EC: PostgreSQL (RDBMS, ACID) veya Cassandra CL=QUORUM.
    veya: Redis DECRBY (atomik, single-threaded → race condition yok) + DB sync.

  Ürün kataloğu (açıklama, resim):
    PA/EL kabul edilebilir: 1-2 sn stale tolere.
    Cassandra CL=ONE, DynamoDB eventual.
```

---

**Sorun 7: Vector clock conflict — DynamoDB'de "which version wins?"**

```
Senaryo:
  Mobil uygulama, offline desteği var. Kullanıcı uçakta not güncelledi.
  Sunucuda da başka bir cihazdan güncellendi (eş zamanlı, farklı DC).

  Node A (cihaz): not = "toplantı notları v2" → vector clock [2,0]
  Node B (sunucu): not = "toplantı notları v3" → vector clock [1,1]

  [2,0] vs [1,1] karşılaştırma:
    [2,0] → [1,1]: 2>1 ama 0<1 → karşılaştırılamaz → concurrent!
    Nedensel ilişki yok → hangisi "doğru" bilinmiyor.

  DynamoDB/Riak: "sibling" (kardeş versiyon) oluşturur.
  Uygulama conflict resolution yapmalı.

Conflict resolution stratejileri:

  1. Last Write Wins (LWW):
     Timestamp yüksek olan kazanır. Basit ama:
     Clock skew → yanlış "kazanan." Veri kaybı.

  2. Kullanıcıya sor (merge):
     "İki versiyon var, hangisini istersin?" UI merge ekranı.
     Doğru ama UX kötü.

  3. CRDT (Conflict-free Replicated Data Type):
     Veri yapısı merge'i otomatik yapar, conflict yok.
     Counter: her increment ayrı izlenir → toplam = tüm increment'ların toplamı.
     Set: add-wins set → iki taraf ekleme yaparsa ikisi de var.
     Shopping cart: CRDT set → iki cihazdan ekleme çakışmaz, ikisi de var.

  4. Operational Transformation:
     Google Docs: concurrent edit'leri op-level merge.
     Karmaşık ama seamless.

Mimari karar:
  Çakışmayı önlemek mümkünse: distributed lock, single writer pattern.
  Çakışma kaçınılmazsa (offline, multi-region): CRDT veya merge UI.
  LWW: sadece "son değer doğru" olan senaryolarda (log timestamp gibi).
```

---

### Mülakat Soruları — PACELC & Vector Clock

**Junior / Mid:**

6. PACELC teoremi nedir? CAP'ten farkı nedir?

   > **Beklened:** CAP: partition durumunda C (consistency) mi A (availability) mi seçilir sorusunu cevaplar. PACELC (Daniel Abadi, 2012): daha kapsamlı. P: partition varsa A mı C mi? E: partition yoksa (Else — normal çalışma) L (low latency) mı C mi? Fark: CAP sadece "kötü gün" senaryosunu (partition) ele alır. PACELC: normal günde de tradeoff var — quorum bekle → yavaş (EC), tek node'a yaz → hızlı ama stale (EL). Sistemler: DynamoDB, Cassandra default → PA/EL (partition'da available, normal'de hızlı). HBase, VoltDB → PC/EC (her zaman consistent). Cassandra tunable: CL=ONE → PA/EL, CL=QUORUM → PA/EC, CL=ALL → PC/EC. Hangi iş gereksinimi: stok → PC/EC. Sepet → PA/EL.

7. PA/EL ve PC/EC sistemlere birer gerçek dünya örneği ver.

   > **Beklened:** PA/EL — DynamoDB (Amazon Shopping Cart): Sepete ürün ekleme: milisaniye latency zorunlu. Partition'da: her AZ kendi başına çalışır → kullanıcı ürün ekleyebilir. Stale: kullanıcı A ve B aynı sepeti düzenlerse → geçici tutarsızlık → eventual merge. Kabul: alışveriş deneyimi > veri anlık tutarlılığı. PC/EC — CockroachDB (Banka transferi): Her yazma quorum onayı bekler → latency yüksek. Partition'da: quorum sağlanamıyorsa yazma reddedilir (availability feda). Tutarsız bakiye: kabul edilemez → her zaman strong consistency. Aradaki: Cassandra tunable → endpoint bazlı seçim. Profil okuma → EL. Ödeme → EC. Aynı sistem, farklı CL.

---

**Senior / Architect:**

8. Vector clock nedir? Lamport timestamp'ten farkı ve hangi sorunu çözdüğü?

   > **Beklened:** Lamport timestamp: global sıralama için saat. Her event'te clock artar. Sınır: iki event'in eş zamanlı (concurrent) mı yoksa nedensel mi olduğunu ayırt edemez. "A, B'den önce mi oldu yoksa bağımsız mı?" sorusuna cevap veremez. Vector clock: N node için N boyutlu vektör [VC_A, VC_B, VC_C]. Her node kendi sayacını tutar. Mesaj alınca: max(local, received) + kendi sayacı. Karşılaştırma: VC_A ≤ VC_B (tüm elemanlar) → A, B'den önce. Karşılaştırılamıyorsa (bazı eleman büyük bazı küçük) → concurrent. Concurrent → conflict resolution gerekli. Kullananlar: DynamoDB internal, Riak, CRDTs. Maliyet: N node → N boyutlu vektör → büyük cluster'da overhead. Dinamik node sayısı → vektör boyutu değişir.

9. Bir architect olarak "stok azaltma servisini" hangi PACELC pozisyonunda tasarlarsın?

   > **Beklened:** Stok azaltma: overselling = direkt gelir kaybı + müşteri güvensizliği → PC/EC. Partition durumunda: minority taraf yazma reddeder → bazı kullanıcılar "şu an satın alınamıyor" hatası alır. Availability feda, tutarsızlık kabul edilemez. Normal çalışmada: quorum onayı bekle → latency 20-50ms artar. Kabul edilebilir: kullanıcı "satın al" butonuna bastığında 50ms ek süre tolere edilir. Implementasyon seçenekleri: PostgreSQL + SELECT FOR UPDATE (satır kilidi, ACID). CockroachDB (dağıtık, PC/EC). Cassandra CL=QUORUM (sloppy quorum riski + dikkat). Redis DECRBY (atomik, tek node sorunu çözülüyorsa). Ek katman: idempotency key → çift sipariş önleme. DB unique constraint → son savunma. Monitoring: quorum hataları, transaction timeout, başarısız stok azaltma sayısı.

---

## Karma — Architect Seviyesi

10. **"3-node etcd cluster, 1 node'u network bölünmesine maruz kaldı. Ne olur?"**

    > **Beklened:** 3-node, quorum=2. Bölünme: 1-2 partition. 2-node taraf: quorum=2 sağlanıyor → çalışmaya devam. Yeni Raft lider seçimi: 2-node tarafta seçilir. 1-node taraf: quorum=1 < 2 → yazma reddedilir, okuma da reddedilir (strict read). Kubernetes API Server: 2-node etcd → API Server çalışır. 1-node tarafta olan etcd → unavailable. Network düzelince: 1-node, 2-node tarafının Raft liderinden güncel log'u alır. Diverge etmiş entry yok (1-node quorum alamadı → yazamadı). Resync: 1-node, Leader'dan AppendEntries alır, güncel duruma gelir. Operasyonel: etcdctl endpoint health → single node unhealthy → monitörü tetikledi → müdahale. 2-node çalışıyor → Kubernetes cluster devam.

11. **"Bir sistem PA/EL'den PC/EC'ye geçmek istiyor. Migrasyonu nasıl yönetirsin?"**

    > **Beklened:** (1) İş gereksinimi analizi: hangi endpoint'ler gerçekten PC/EC gerektiriyor? Tümü değil, kritik olanlar (ödeme, stok). (2) Mevcut stale okuma etkisini ölç: conflict ne sıklıkla oluyor? Stale window ortalama ne kadar? (3) Adım adım migrasyon: önce yazma CL yükselt (ONE → QUORUM) → latency ölç. Sonra okuma CL yükselt. Load test: quorum latency ile SLA uyuşuyor mu? (4) DB seçimi: Cassandra QUORUM yetmiyorsa (sloppy quorum riski) → PostgreSQL veya CockroachDB. Şema migrasyonu: veri taşıma stratejisi. (5) Availability etkisi: PC/EC: partition'da azınlık taraf yazma reddeder. SLA güncelle: "partition durumunda X% istek hata alabilir." (6) Monitoring: quorum hata oranı, latency p99, conflict sayısı. Rollback planı: feature flag ile CL'yi düşür. (7) Client tarafı: retry mantığı → quorum timeout → idempotency key ile güvenli retry.
