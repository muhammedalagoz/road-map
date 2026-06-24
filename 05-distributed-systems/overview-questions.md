# 05 — Dağıtık Sistemler Genel Bakış: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: CAP Teoremi & Consistency

### Gerçek Hayat Sorunları

---

**Sorun 1: AP sistem seçildi ama iş kritik işlem tutarsız veri döndürdü**

```
Senaryo:
  E-ticaret: stok servisi için Cassandra (AP) seçildi.
  "Eventual consistency kabul edilebilir" varsayıldı.

  Black Friday:
    T+0:  Stok: 1 adet kaldı.
    T+1:  User A ve User B aynı anda satın aldı.
    T+1:  Node A: stok=0 yazdı (User A).
    T+1:  Node B: stok=0 yazdı (User B) — replikasyon henüz gelmedi.
    T+2:  Her iki sipariş onaylandı.
    T+3:  Replikasyon tamamlandı: LWW (Last Write Wins) → bir yazma kazandı.
    T+3:  İki sipariş var, bir ürün var → overselling.

Kök neden:
  AP sistem: ağ bölünmesinde her iki partition yazmaya devam etti.
  LWW conflict resolution: sadece birisini tutar, öbürü kaybolmaz ama
  uygulama her ikisini de "başarılı" olarak işledi.

Düzeltme seçenekleri:

  1. Cassandra QUORUM consistency:
     ConsistencyLevel.QUORUM → yazma/okuma quorum (2/3 node onayı).
     Ağ bölünmesinde minority partition reddeder → CP davranışı.
     Tradeoff: latency artar, availability azalır.

  2. DB unique constraint (son savunma):
     RDBMS + SELECT FOR UPDATE: stok azaltmayı DB transaction ile yap.
     Cassandra lightweight transaction (LWT):
       UPDATE stock SET quantity = 0
       WHERE product_id = 'X' IF quantity > 0
       -- IF koşulu: compare-and-set (Paxos tabanlı) → daha yavaş

  3. İş mantığı kararı:
     Stok kritikse → CP sistem (PostgreSQL, etcd) kullan.
     Stale stok tolere ediliyorsa → AP + idempotency key + sonradan düzeltme.

CAP öğrenimi:
  "AP sistem seçtik" ≠ "her şey yolunda."
  AP: partition anında availability → consistency feda.
  İş gereksinimi: "yanlış satmak" mı "satamamak" mı daha kötü?
  → Buna göre CP / AP seç.
```

---

**Sorun 2: Strong consistency + yüksek latency → kullanıcı deneyimi bozuldu**

```
Senaryo:
  Sosyal medya: "beğeni sayısı" için strong consistency seçildi.
  Her beğenide: tüm replica'lara sync yaz (quorum).

  Sonuç:
    Beğeni işlemi: 150ms (tek DC) → 800ms (çok bölgeli, quorum bekle).
    Kullanıcı: butona bastı, 800ms dondu.
    Black Friday: 10.000 eş zamanlı beğeni → DB bottleneck.

Kök neden:
  "Beğeni sayısı" için strong consistency gerekli mi?
  Kullanıcı 1234 yerine 1235 görse ne olur? Hiçbir şey.
  Ama ödeme için yanlış bakiye görülmesi: felaket.

Consistency seçimi iş gereksinimi bazlı:

  Strong Consistency:
    Banka bakiyesi, stok azaltma, ödeme işlemi.
    "Yanlış veri görmek" ciddi sonuç doğurursa.

  Eventual Consistency:
    Beğeni/yorum sayacı, öneri sistemi, kullanıcı profil cache.
    "Biraz stale" kabul edilebilirse.

  Causal Consistency:
    Yorum → yanıt ilişkisi: yanıtı gören yorumu da görmeli.
    "B, A'ya bağlı" ise A önce görülmeli.

Düzeltme:
  Beğeni sayacı: Redis INCR (eventual) → periyodik DB sync.
  Okuma: Redis'ten hızlı dön, DB sync arkaplanda.
  Sonuç: 150ms → 5ms, strong consistency yok ama kullanıcı fark etmez.
```

---

**Sorun 3: 2PC coordinator çöktü — tüm servisler kilitlendi**

```
Senaryo:
  Sipariş sistemi: OrderService, PaymentService, InventoryService.
  Two-Phase Commit ile distributed transaction.

  Timeline:
    T+0:  Coordinator → Prepare → [Payment: OK, Inventory: OK]
    T+1:  Coordinator → Commit gönderiyor...
    T+1:  Coordinator çöktü (Commit mesajı gönderilmedi).

  Sonuç:
    PaymentService: "Prepare aldım, commit beklemeliyim" → lock tutuyor.
    InventoryService: "Prepare aldım, commit beklemeliyim" → lock tutuyor.
    Coordinator geri gelene kadar (recovery): her iki servis kilitli.
    Recovery süre: dakikalar, saatler olabilir.

2PC'nin "blocking protocol" sorunu:
  Coordinator çöküşünde participant'lar ne yapacağını bilemez.
  Commit mi rollback mı? → Coordinator olmadan karar veremez.
  Çözüm için Coordinator'ın log'undan recovery → karmaşık, yavaş.

Saga Pattern ile çözüm:
  Her adım: local transaction → event yayınla.
  Hata: compensating transaction (geri alma işlemi).

  OrderService → order.created event
    PaymentService: ödeme al → payment.completed event
    InventoryService: stok azalt → inventory.reserved event
  Hata (InventoryService):
    inventory.failed event →
    PaymentService: ödeme iade et (compensating)
    OrderService: siparişi iptal et (compensating)

  Tradeoff:
    Saga: lock yok, throughput yüksek, eventual consistency.
    2PC: strong consistency, lock tabanlı, düşük throughput.
    Microservices → Saga tercih edilir.
```

---

**Sorun 4: Saga choreography — hangi event nerede hata verdi? Debug kabusu**

```
Senaryo:
  10 mikroservis, choreography-based Saga.
  Sipariş akışı: Order → Payment → Inventory → Shipping → Notification → ...
  Hata: bazı siparişler yarıda kaldı ama hangi adımda bilinmiyor.

  Log inceleme: her serviste ayrı log → hangisinde failure?
  Distributed trace yok → siparişi takip etmek imkânsız.
  Event loop: bir servis event yayınladı ama consumer çökmüştü → event kayboldu.

Choreography sorunları:
  Business logic: 10 servise dağıldı → anlık durum görüntüsü yok.
  Hata ayıklama: hangi event nerede kayboldu?
  Circular event: A → B → C → A (döngü riski).
  Test: tüm servisleri ayağa kaldır, event sırasını simüle et.

Orchestration ile karşılaştırma:
  SagaOrchestrator: her adımı merkezi olarak yönetir.
  Durum makinesi: "şu an hangi adımdayım, ne beklenildi, ne geldi?"
  Log: tek yerde → debug kolay.
  
  Hangi akış için ne:
    Choreography: 2-3 servis, basit lineer akış, event-driven kültür varsa.
    Orchestration: 5+ servis, karmaşık iş mantığı, rollback kritikse.

Operasyonel çözüm:
  Distributed tracing: OpenTelemetry + Jaeger → trace_id ile tüm akışı takip.
  Correlation ID: her event'te ortak ID → log aggregation (ELK) ile takip.
  Dead Letter Queue: işlenemeyen event'ler DLQ'ya → monitoring alert.
  Event store: tüm event'leri persist et → replay ve audit mümkün.
```

---

**Sorun 5: Consistent hashing — node çıkışı, beklentinin ötesinde cache miss**

```
Senaryo:
  Redis Cluster: consistent hashing ring, 5 node.
  1 node kaldırıldı → beklenti: %20 key etkilenir.

  Gerçek: sanal node (virtual node) yoksa dengesiz dağılım:
    Node A: 0-72° arası → key'lerin %20'si
    Node B: 72-200° arası → key'lerin %35'i
    ...dağılım eşit değil.

  1 node çıktı → komşu node'a devredilen key sayısı farklı farklı.

Virtual node çözümü:
  Her fiziksel node → ring'de 150 sanal node.
  Dağılım çok daha homojen → node ekle/çıkar → tutarlı %1/N etki.

  Redis Cluster: 16384 hash slot → sanal node yerine slot tabanlı.
  Fiziksel node = belirli slot aralığı.
  Node ekle/çıkar: sadece ilgili slot'lar taşınır.

Thundering herd (cache miss burst) sonrası:
  Büyük cache miss → tüm istek DB'ye gider → DB çöker.
  Çözüm: mutex/lock ile "tek request DB'ye gider, diğerleri bekler."
  Spring Cache: @Cacheable yeterli değil, ayrıca "cache stampede" koruması ekle.
  
  // Redisson ile cache stampede koruması:
  RLock lock = redisson.getLock("cache-load:" + key);
  if (lock.tryLock(100, 5000, TimeUnit.MILLISECONDS)) {
      try {
          value = cache.get(key);        // tekrar kontrol (başkası yüklemiş olabilir)
          if (value == null) {
              value = db.load(key);
              cache.put(key, value);
          }
      } finally {
          lock.unlock();
      }
  }
```

---

### Mülakat Soruları — CAP & Consistency

**Junior / Mid:**

1. CAP teoremi nedir? Neden P (partition tolerance) vazgeçilemez?

   > **Beklened:** CAP: dağıtık sistemde Consistency, Availability ve Partition Tolerance üçü aynı anda sağlanamaz. C: her okuma en güncel yazıyı görür. A: her istek yanıt alır. P: ağ bölünmesinde sistem çalışmaya devam eder. Neden P vazgeçilemez: gerçek dünyada ağ bölünmesi kaçınılmaz (kablo kopuşu, veri merkezi arızası, ağ gecikmesi). P olmadan: ağ bölünmesinde sistem tamamen durur → pratik değil. Bu yüzden seçim: CP (consistency feda etmezsin, availability azalır — HBase, ZooKeeper) veya AP (availability feda etmezsin, consistency azalır — Cassandra, DynamoDB). CAP binary değil spectrum: Cassandra tunable consistency ile ikisi arasında kayar.

2. Strong consistency, eventual consistency ve causal consistency farkı nedir?

   > **Beklened:** Strong: okuma her zaman en son yazılan veriyi görür. Distributed lock veya single leader gerektirir. Yavaş. Kullanım: banka bakiyesi, stok azaltma. Eventual: sistem sonunda tutarlı olur, anlık farklılıklar mümkün. Replikasyon gecikmesi → kısa süre eski veri dönebilir. Kullanım: beğeni sayacı, sosyal feed. Causal: nedensel ilişki korunur. A → B sırasıyla oluştuysa, B'yi gören A'yı da görür. Kullanım: yorum ve yanıt sistemi — yanıtı gören yorumu da görür. Fark: causal, eventual'dan güçlü ama strong'dan zayıf. Her biri için örnek: strong = "bakiye yanlış görünmemeli," eventual = "beğeni sayısı 1 eksik görünse de olur," causal = "yanıt yorumdan önce görünmemeli."

3. Two-Phase Commit nedir? Neden microserviceste kullanılmaz?

   > **Beklened:** 2PC: distributed transaction protokolü. Phase 1 (Prepare): coordinator → tüm participant'lara "hazır mısın?" Phase 2 (Commit/Rollback): hepsi "evet" → commit, biri "hayır" → rollback. Sorunlar: (1) Blocking protocol: coordinator çökerse participant'lar kilitli bekler — kaynak tüketimi, deadlock. (2) Lock süresi: tüm participant'lar cevap verene kadar lock tutulur → düşük throughput. (3) Microservices: servisler bağımsız deploy edilir, farklı DB'ler kullanır → koordinasyon karmaşık, single point of failure. Alternatif: Saga Pattern — local transaction + compensating transaction. 2PC uygulaması: tek DB, aynı cluster içi (same JTA transaction manager).

---

**Senior / Architect:**

4. Saga Pattern: choreography vs orchestration. Hangisini ne zaman seçersin?

   > **Beklened:** Choreography: servisler birbirinin event'lerini dinler, merkezi koordinatör yok. Avantaj: loose coupling, bağımsız deploy. Dezavantaj: iş mantığı dağılır, debug zor, circular event riski. Orchestration: Saga Orchestrator her adımı yönetir, merkezi durum makinesi. Avantaj: iş mantığı tek yerde, debug kolay, rollback net. Dezavantaj: orchestrator SPOF, servisler orchestratora bağımlı. Seçim: 2-3 servis, basit lineer akış → choreography. 5+ servis, karmaşık iş mantığı, rollback kritik → orchestration. Her durumda: distributed tracing (OpenTelemetry), correlation ID, DLQ şart. Kompansasyon işlemleri idempotent olmalı (aynı iade iki kez yapılsa sorun olmamalı).

5. CAP teoremi gerçek bir sistemde nasıl uygulanır? "Tunable consistency" ne demektir?

   > **Beklened:** CAP soyut — gerçek sistemlerde spectrum. Cassandra "tunable consistency": ConsistencyLevel.ONE → AP davranışı, hızlı ama stale olabilir. ConsistencyLevel.QUORUM → CP'ye yakın, çoğunluk onayı. ConsistencyLevel.ALL → strong, tüm replica onayı, en yavaş. Seçim: okuma ve yazma consistency'si ayrı ayarlanır. W + R > N formülü: N=3, W=2, R=2 → W+R=4 > 3 → strong consistency garantisi. Gerçek karar: sorgu başına iş gereksinimi → stok azaltma: QUORUM. Kullanıcı feed: ONE. Ödeme: ALL veya QUORUM + uygulama katmanı kontrolü. "Tunable consistency": tek sistemde her iki davranış → ekibin iş gereksinimi bazlı seçim yapması.

---

## Bölüm 2: Rate Limiting & Circuit Breaker

### Gerçek Hayat Sorunları

---

**Sorun 6: Distributed rate limiting yok — her pod ayrı saydı, limit 3× aşıldı**

```
Senaryo:
  API Gateway: kullanıcı başına 100 istek/dakika limiti.
  3 uygulama instance, her biri bellekte sayıyor.

  Kullanıcı: 3 instance'a round-robin → her birine 99 istek.
  Toplam: 297 istek → limit 100, ama hiçbir instance 100'ü geçmedi.
  429 Too Many Requests dönmedi → limit bypass edildi.

Düzeltme (Redis atomic counter):
  String key = "rate:" + userId + ":" + currentMinute();
  Long count = redis.execute(script,
      Collections.singletonList(key),
      "1",          // INCRBY değeri
      "60"          // EXPIRE saniye
  );
  // Lua script:
  // local count = redis.call('INCR', KEYS[1])
  // if count == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
  // return count

  if (count > 100) {
      throw new RateLimitExceededException();
  }

  Neden Lua: INCR + EXPIRE iki ayrı komut → araya başka istek girebilir.
  Lua script: Redis'te atomik → race condition yok.

Sliding window (daha doğru):
  Fixed window sorunu: 0:59 → 99 istek, 1:00 → 99 istek.
  2 saniyede 198 istek → limit 100 ama aşıldı.
  
  Redis ZSET sliding window:
  key = "rate:user:123"
  ZADD key timestamp request_id
  ZREMRANGEBYSCORE key 0 (now - 60s)  // eski istekleri temizle
  count = ZCARD key
  if count > 100: reject
  
  Her istek: O(log N) → daha pahalı ama doğru.
```

---

**Sorun 7: Circuit Breaker threshold yanlış — sağlıklı servisi OPEN'a geçirdi**

```
Senaryo:
  Payment servisi, Circuit Breaker:
    slidingWindowSize = 5
    failureRateThreshold = 50%

  Senaryo:
    İlk 5 istek: 2 başarısız, 3 başarılı.
    %40 hata → CB CLOSED (tamam).
    Sonraki 5 istek: 3 başarısız (geçici DB yavaşlığı).
    %60 hata → CB OPEN.
    Payment servisi aslında sağlıklı, sadece 5 istek penceresi kötüydü.
    
    30 saniye: tüm ödeme istekleri reddedildi (CB OPEN).

Sorun: küçük window boyutu → gürültüye duyarlı.

Düzeltme:
  slidingWindowSize = 100  // 100 istek penceresi → istatistiksel güvenilir
  failureRateThreshold = 50
  minimumNumberOfCalls = 20  // En az 20 çağrı olmadan OPEN'a geçme
  waitDurationInOpenState = 60s
  permittedNumberOfCallsInHalfOpenState = 5

  Ayrıca: slowCallRateThreshold → yavaş çağrıları da say.
  slowCallDurationThreshold = 2s → 2s'den uzun → "başarısız" say.

CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .minimumNumberOfCalls(20)          // <20 çağrıda OPEN olmaz
    .slidingWindowSize(100)
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .permittedNumberOfCallsInHalfOpenState(5)
    .slowCallRateThreshold(80)
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .build();

Fallback stratejisi:
  CB OPEN → fallback mutlaka olmalı:
  - Cache'ten eski veri dön
  - "Şu an kullanılamıyor" mesajı
  - Async queue'ya al, sonra işle
  Fallback olmadan: OPEN → hata → kullanıcıya exception.
```

---

### Mülakat Soruları — Rate Limiting & Circuit Breaker

**Junior / Mid:**

6. Token bucket ve sliding window rate limiting farkı nedir?

   > **Beklened:** Token bucket: bucket kapasitesi (100 token), doldurma hızı (10/s). Gelen istek → token al. Token yoksa → 429. Burst'a izin verir: bucket doluysa anlık 100 istek. AWS API Gateway, Nginx kullanır. Sliding window: son X saniyedeki istek sayısını say. Fixed window sorunu: 0:59 → 99, 1:00 → 99 → 2s'de 198 istek, limit 100. Sliding window: herhangi 60s penceresinde max 100. Redis ZSET: timestamp ile istek logla, eski kayıtları temizle, say. Distributed: Redis atomic counter → tüm instance'lar tek sayaç. Token bucket: burst-friendly, hafif implementasyon. Sliding window: kesin limit, daha pahalı (O(log N) per request).

7. Circuit Breaker state machine'ini açıkla. HALF_OPEN ne işe yarar?

   > **Beklened:** 3 state: CLOSED (normal çalışma), OPEN (tüm çağrılar reddedilir), HALF_OPEN (test modu). CLOSED → OPEN: hata oranı threshold'u aşınca (örn. %50). OPEN → HALF_OPEN: waitDuration geçince (örn. 30s). Servis geri döndü mü? Test et. HALF_OPEN → CLOSED: test çağrıları başarılı → servis iyileşti, normale dön. HALF_OPEN → OPEN: test çağrıları başarısız → servis hâlâ problem var, tekrar bekle. HALF_OPEN amacı: OPEN'dan direkt CLOSED'a geçmek tehlikeli — servis hâlâ bozuk olabilir. HALF_OPEN: sınırlı istek geçir, test et, sonra karar ver. Resilience4j: permittedNumberOfCallsInHalfOpenState → kaç test isteği.

---

**Senior / Architect:**

8. Circuit Breaker fallback stratejilerini karşılaştır. Ne zaman hangisi?

   > **Beklened:** (1) Cache'ten eski veri: kullanıcıya biraz stale bilgi göster. Örnek: ürün listesi, döviz kuru. Ne zaman: eventual consistency tolere, hiç yanıt vermemek daha kötü. (2) Default değer: "servis şu an yavaş, varsayılan değer: ücretsiz kargo" gibi. Ne zaman: iş mantığı safe default'u kabul ediyorsa. (3) Queue'ya al: ödeme işlemini "pending" olarak kaydet, servis açılınca işle. Ne zaman: işlem ertelenebilirse, idempotency garantisi varsa. (4) Kullanıcıya hata: "şu an kullanılamıyor, daha sonra deneyin." Ne zaman: safe default yok, yanlış bilgi göstermek daha tehlikeli. Seçim: iş etkisi × veri doğruluk gereksinimi. Ödeme: queue → sonradan işle. Ürün fiyatı: cache → stale tolere. Kritik stok: hata → "şu an satın alınamıyor."

---

## Bölüm 3: Service Discovery & Load Balancing

### Mülakat Soruları

**Junior / Mid:**

9. Client-side ve server-side service discovery farkı nedir?

   > **Beklened:** Client-side (Eureka + Ribbon): servis → Eureka'ya kaydolur. İstemci: Eureka'dan adres listesini çeker, kendi load balance eder. Avantaj: altyapı basit (ekstra hop yok). Dezavantaj: her client LB mantığını bilmeli → dil/framework bağımlı. Server-side (Consul + LB): servis → Consul'e kaydolur. İstemci → Load Balancer'a gider. LB → Consul → doğru servise yönlendirir. Avantaj: client basit, dil bağımsız. Dezavantaj: ekstra hop (latency), LB SPOF. Kubernetes: server-side (kube-proxy + Service resource). Spring Boot: Eureka (client-side). Seçim: Kubernetes ortamı → server-side zaten sağlanmış. Eureka: Spring Boot ekosistemi, client-side tercih.

10. Consistent hashing nedir? Neden normal mod-N hashing'den üstündür?

    > **Beklened:** Normal mod-N: hash(key) % N = node. Sorun: N değişirse (node ekle/çıkar) → hemen hemen tüm key'ler farklı node'a gider → büyük cache miss. Consistent hashing: sanal ring (0-2^32). Her node ring'de birkaç noktada. Key → hash → ring'de saat yönünde ilk node. Node ekle: sadece o node ile önceki komşu arasındaki key'ler etkilenir. N=5 → 1 ekle → %1/6 key etkilenir (teorik). Mod-N: 1 ekle → %83 key etkilenir. Virtual node: her fiziksel node ring'de 150 sanal nokta → homojen dağılım. Redis Cluster: 16384 hash slot → consistent hashing variant. Kullanım: distributed cache (Redis Cluster, Memcached), stateful load balancer, Cassandra token ring.

---

**Senior / Architect:**

11. Idempotency key implementasyonu nasıl yapılır? Hangi edge case'leri var?

    > **Beklened:** Temel akış: client → Idempotency-Key: UUID header ile istek. Server: (1) Redis/DB'de key var mı? (2) Evet → önceki yanıtı döndür (işlemi tekrar yapma). (3) Hayır → işle, sonucu key ile birlikte kaydet. Edge case'ler: (a) İlk istek işlenirken ikinci aynı key geldi → "processing" state → ikincisini beklet veya 409 Conflict dön. Race condition için Redis SET NX (atomic). (b) TTL: key sonsuza kadar saklanmamalı. 24h yeterli genellikle. (c) Yanıt ne kadar süre saklanmalı? Retry window'u geç → key'i temizle. (d) Partial failure: işlem başladı, sonucu kaydedemeden çöktü → retry gelir → yeniden işler. Çözüm: işlemi ve sonucu aynı transaction'da kaydet. (e) Client farklı payload ile aynı key gönderirse: 422 Unprocessable (key = önceki payload). HTTP kaynaklar: Stripe, Braintree idempotency key implementasyonu.

---

## Karma — Architect Seviyesi

12. **"Yeni bir e-ticaret sistemi tasarlıyorsun. Sipariş, ödeme ve stok servislerinde hangi consistency stratejisini seçersin?"**

    > **Beklened:** Servis başına iş gereksinimi analizi: Stok servisi: overselling felakettir → strong consistency. PostgreSQL + SELECT FOR UPDATE veya Cassandra QUORUM. Ödeme servisi: çift ödeme = itibar ve yasal sorun → strong consistency + idempotency key. DB transaction ile. Sipariş servisi: "sipariş listesi 1 sn stale görünse" tolere → eventual consistency. CQRS: yazma strong, okuma eventually consistent. Distributed transaction (2PC): microservicesta kullanma. Saga pattern: Order → Payment → Inventory. Compensating: ödeme iade et, siparişi iptal et. Tradeoff sunumu: Strong consistency: doğru ama yavaş. Eventual: hızlı ama business risk. Her alan için doğru seviyeyi seç. Monitoring: idempotency key başarı oranı, saga compensation sayısı, CB open sayısı.

13. **"Sisteminde rate limiting uygulamak istiyorsun. Nerede ve nasıl implement edersin?"**

    > **Beklened:** Nerede: API Gateway katmanı (en iyi yer): tüm trafik buradan → merkezi kontrol. Servis katmanında da yapılabilir (per-service quota). Nasıl: distributed → Redis atomic counter. Tek instance → in-memory token bucket. Algoritma seçimi: Token bucket: burst kabul edilebilirse (API genel kullanım). Sliding window: kesin limit gerekirse (ödeme API). Granülasyon: user bazlı, IP bazlı, endpoint bazlı, tenant bazlı. Farklı endpoint farklı limit: POST /payments → 10/dakika, GET /products → 1000/dakika. Edge case'ler: Redis down → fail-open (isteğe izin ver) veya fail-closed (reddet). İş gereksinimi: fail-open → rate limiting bypass riski ama availability korunur. Yanıt: 429 Too Many Requests + Retry-After header. Monitoring: rate limit hit sayısı, hangi kullanıcı/IP en çok hit, anomali tespiti (DDoS).

14. **"Microservis mimarisinde distributed transaction yerine ne kullanırsın? Tradeoff'ları neler?"**

    > **Beklened:** Alternatifler: (1) Saga Pattern: local transaction + event + compensating transaction. Choreography veya orchestration. Tradeoff: eventual consistency, komplex hata senaryoları, idempotent compensating gerekir. (2) Outbox Pattern: DB transaction ile event'i aynı anda kaydet (outbox table). CDC (Debezium) veya poller event'i Kafka'ya iletir. Tradeoff: at-least-once delivery → consumer idempotent olmalı. (3) Idempotency key: retry güvenli → duplicate işlem değil. (4) DB unique constraint: race condition'ı DB'nin ACID'ine bırak. Seçim kılavuzu: 2 servis, basit: Saga choreography. 5+ servis, karmaşık: Saga orchestration. Veri bütünlüğü kritik: Outbox + Kafka. Çok yüksek throughput: eventual + compensation + monitoring. Her durumda: idempotency key, DLQ, distributed tracing şart.
