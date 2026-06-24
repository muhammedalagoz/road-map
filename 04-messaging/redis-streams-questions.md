# 04c — Redis Streams: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Temel Kavramlar & XADD / XREAD

### Gerçek Hayat Sorunları

---

**Sorun 1: Redis Pub/Sub kullandık — consumer offline'dayken tüm event'ler kayboldu**

```
Senaryo:
  Bildirim servisi Redis Pub/Sub ile event dinliyor.
  Deployment sırasında servis 2 dakika down oldu.
  Bu 2 dakikada 5.000 "order.created" event'i yayınlandı.
  Servis ayağa kalkınca: tüm event'ler kayıp — bildirimler hiç gönderilmedi.

Redis Pub/Sub davranışı:
  SUBSCRIBE → o an bağlı subscriber'lara iletilir.
  Bağlı değilsen → mesaj KAYBOLUR. Persistence yok, buffer yok.
  Kullanım: cache invalidation sinyali, gerçek zamanlı dashboard (kayıp tolere edilir).

Sorun: "Kesinlikle işlenmeli" senaryolarda Pub/Sub yanlış seçim.

Düzeltme: Redis Streams
  XADD orders * event order.created orderId 123  → stream'e kalıcı kayıt
  
  Consumer group:
  XGROUP CREATE orders notification-service $ MKSTREAM
  XREADGROUP GROUP notification-service worker-1 COUNT 10 STREAMS orders >
  
  Servis 2 dakika down oldu → ayağa kalkınca:
  XREADGROUP ... > → pending (verilmemiş) entry'leri okur
  2 dakikadaki 5.000 entry → hepsi işlenir, hiçbiri kaybolmaz.
  
Seçim tablosu:
  "Consumer yokken mesaj kaybolsun" → Pub/Sub (realtime dashboard, game score)
  "Mutlaka işlenmeli, replay lazım" → Redis Streams
  "Karmaşık routing, öncelik"       → RabbitMQ
  "100K+ msg/sn, haftalık retention"→ Kafka
```

---

**Sorun 2: `MAXLEN` ayarlanmadı — Redis belleği doldu, servis çöktü**

```bash
# Senaryo: Stream'e MAXLEN koymadan sürekli XADD.
# Audit log: günde 2M kayıt → hafta sonunda 14M entry → Redis OOM.
# Redis: "OOM command not allowed when used memory > maxmemory"
# Tüm Redis operasyonları fail → servisler çöktü.

# YANLIŞ: Sınırsız stream
XADD audit-log * userId 42 action LOGIN ip 192.168.1.1
# 14M entry × 100 byte = ~1.4GB sadece audit log

# DOĞRU A: Yaklaşık trim (performanslı)
XADD audit-log MAXLEN ~ 100000 * userId 42 action LOGIN ip 192.168.1.1
# ~ = segment bazlı trim (kesin 100K değil ama yakın, çok daha hızlı)

# DOĞRU B: Kesin trim (yavaş, büyük stream'de dikkat)
XADD audit-log MAXLEN 100000 * userId 42 action LOGIN ip 192.168.1.1

# DOĞRU C: Zaman bazlı trim (Redis 6.2+)
XADD audit-log MINID ~ 1705312800000 * userId 42 action LOGIN ip 192.168.1.1
# MINID: bu timestamp'ten eski entry'leri sil

# Monitoring:
XLEN audit-log  # entry sayısı
XINFO STREAM audit-log  # radix-tree boyutu, bellek kullanımı

# İlke: Stream'ler RAM'de yaşar — MAXLEN her XADD'de ver
# veya periyodik XTRIM ile temizle:
XTRIM audit-log MAXLEN ~ 100000  # cron job ile
```

---

**Sorun 3: `XGROUP CREATE` ile `$` vs `0` karıştırıldı — eski event'ler işlenmedi**

```bash
# Senaryo: Yeni bir consumer group oluşturuldu.
# Stream'de 50.000 eski entry var.
# Beklenti: tüm geçmişi işlesin.
# Gerçek: hiçbirini görmedi.

# YANLIŞ: $ (şu andan itibaren → geçmiş yok sayılır)
XGROUP CREATE orders inventory-service $ MKSTREAM
# $ = "benden sonra gelen yeni mesajları ver"
# 50.000 eski entry → hiç işlenmedi

# DOĞRU: 0 (baştan okumak için)
XGROUP CREATE orders inventory-service 0 MKSTREAM
# 0 = stream'in başından → tüm geçmiş entry'ler dahil

# MKSTREAM: stream yoksa oluştur (opsiyonel)

# Kullanım rehberi:
# $ → "Sadece bugünden itibaren işle, geçmiş önemli değil" (monitoring, bildirim)
# 0 → "Geçmişi de işle" (migration, yeni servis onboarding, replay)
# Belirli ID → "Bu noktadan itibaren işle" (kısmi replay)
XGROUP CREATE orders inventory-service 1705312800000-0
# Bu ID'den sonraki entry'leri işle

# Pratik: Yeni servis deploy → 0 ile başla → geçmişi tüket → sonra $ ile normal.
```

---

### Mülakat Soruları — Temel Kavramlar

**Junior / Mid:**

1. Redis Streams, Redis Pub/Sub ve Redis List'ten nasıl farklıdır?

   > **Beklened:** Pub/Sub: fire-and-forget — subscriber yoksa mesaj kaybolur. Persistence yok, replay yok. Kullanım: cache invalidation, anlık broadcast. List (LPUSH/RPOP): basit FIFO queue. Persistence var (AOF/RDB). Consumer group yok, tek consumer. Replay yok, okuyunca silinir. Streams: persistent append-only log. Consumer group → paralel işlem. ACK mekanizması (at-least-once). ID bazlı replay. Zaman serisi desteği. Pub/Sub → "anlık, kayıp olursa olsun." List → "basit sıra, tek consumer." Streams → "güvenilir, çok consumer, replay gerekli."

2. Stream entry ID formatı nasıldır? `*` yerine manuel ID ne zaman verilir?

   > **Beklened:** Format: `<milliseconds>-<sequence>`. Örnek: `1705312800000-0`. Aynı ms'de birden fazla entry → sequence artar (0, 1, 2...). `*` (yıldız): Redis otomatik üretir — önerilen, monoton artış garantili. Manuel ID: (1) Dışarıdan zaman damgası dayatmak istiyorsan (IoT cihaz zamanı). (2) Belirli bir sıra garantisi. (3) Idempotency: aynı ID tekrar XADD → hata (duplicate önleme). Dikkat: manuel ID geçmişe gidemez — son entry'den büyük olmalı, yoksa XADD reddedilir.

3. `XPENDING` ne işe yarar?

   > **Beklened:** Consumer group'ta teslim edilmiş ama henüz XACK edilmemiş (pending/unacknowledged) entry'leri listeler. `XPENDING orders inventory-processors - + 10` → hangi consumer hangi entry'yi ne kadar süredir tuttuğunu gösterir: entry ID, consumer adı, idle time (ms), delivery count. Kullanım: crash tespiti — consumer uzun süredir ACK etmiyorsa muhtemelen crash etti. XCLAIM ile o entry başka consumer'a devredilir. Monitoring: pending count yükseliyorsa consumer'lar işleyemiyor demektir → alert.

---

**Senior / Architect:**

4. MAXLEN için `~` (tilde) ile kesin trim arasındaki fark nedir? Neden `~` önerilir?

   > **Beklened:** Kesin MAXLEN: her XADD'de stream boyutunu tam olarak N'e düşürür. Radix tree'nin ortasından silme gerekebilir → pahalı. Büyük stream + yüksek yazma hızı → latency spike. `~` (yaklaşık): en eski tam segment silinir — radix tree segment sınırında doğal kırılma noktası. Kesin N değil ama N'e yakın kalır. Çok daha hızlı — O(1)'e yakın. Production: MAXLEN ~ her zaman tercih et. Kesin MAXLEN: sadece boyut tam kritikse (nadir). Alternatif: XTRIM'i periyodik çalıştır (cron) → XADD overhead'ı sıfır, trim ayrı.

---

## Bölüm 2: Consumer Group & Crash Recovery

### Gerçek Hayat Sorunları

---

**Sorun 4: XACK unutuldu — pending listesi şişti, entry'ler tekrar teslim edildi**

```bash
# Senaryo: Consumer entry'leri okuyor, işliyor ama XACK çağırmıyor.
# Pending list giderek büyüdü → yeniden başlatmada tüm entry'ler tekrar işlendi.
# 100K işlem çift çalıştı → duplicate veri.

# YANLIŞ: ACK yok
XREADGROUP GROUP inventory-service worker-1 COUNT 10 STREAMS orders >
# → 10 entry alındı, işlendi ama ACK edilmedi
# → Redis: "Bu entry'ler hâlâ pending" → bir sonraki okumada tekrar gelebilir

# Neden tekrar gelir?
# XREADGROUP ... > → sadece YENİ (henüz verilmemiş) entry'leri alır
# XREADGROUP ... 0 → PENDİNG (verilmiş ama ACK edilmemiş) entry'leri alır
# Consumer restart → 0 ile pending'leri kontrol etmeli

# DOĞRU: Her işlem sonrası ACK
entries = XREADGROUP GROUP inventory-service worker-1 COUNT 10 STREAMS orders >

for entry in entries:
    processOrder(entry)                              # işle
    XACK orders inventory-service entry.id           # onay

# İdempotency şart: at-least-once delivery → duplicate olabilir
# DB'de idempotency key: INSERT ... ON CONFLICT DO NOTHING
# veya entry ID'yi işlem kaydına ekle → tekrar gelirse tespit et

# Java - Spring Data Redis:
for (MapRecord<String, Object, Object> record : records) {
    try {
        processOrder(record.getValue());
        redis.opsForStream().acknowledge("orders", "inventory-service", record.getId());
    } catch (Exception e) {
        // ACK etme → pending'de kal → retry (XCLAIM ile başka consumer alır)
        log.error("Failed to process: {}", record.getId(), e);
    }
}
```

---

**Sorun 5: Consumer crash → pending entry'ler sonsuza dek bekledi**

```bash
# Senaryo:
# consumer-1, entry'yi XREADGROUP ile aldı → işlemeye başladı → crash etti.
# ACK gelmedi → entry pending listesinde kaldı.
# consumer-2 ve consumer-3 ">" ile sadece yeni entry'leri okuyor.
# O entry sonsuza kadar işlenmeden bekliyor.

# Teşhis:
XPENDING orders inventory-service - + 10
# 1705312800000-5  consumer-1  3600000  1
# → 1 saattir idle, consumer-1'de bekliyor

# Çözüm: XCLAIM ile başka consumer'a devret
XCLAIM orders inventory-service consumer-2 60000 1705312800000-5
#                                            ↑ 60sn'den fazla idle → devret

# XAUTOCLAIM (Redis 6.2+): otomatik sahipliği devret
XAUTOCLAIM orders inventory-service consumer-2 60000 0-0 COUNT 10
# 60sn+ idle olan pending entry'leri otomatik consumer-2'ye ata

# Tam crash recovery döngüsü (her consumer başlangıçta çalıştırmalı):
def recover_pending(consumer_name):
    # 1. Kendi pending entry'lerini işle (önceki session'dan kalanlar)
    pending = XREADGROUP GROUP inventory-service {consumer_name} COUNT 10 STREAMS orders 0
    for entry in pending:
        process_and_ack(entry)
    
    # 2. Uzun süredir sahipsiz entry'leri devral
    XAUTOCLAIM orders inventory-service {consumer_name} 300000 0-0  # 5dk+ idle
    
    # 3. Normal okumaya başla
    while True:
        entries = XREADGROUP GROUP inventory-service {consumer_name} 
                  COUNT 10 BLOCK 5000 STREAMS orders >
        for entry in entries:
            process_and_ack(entry)

# delivery-count izle: çok teslim edilmişse → poison entry
XPENDING orders inventory-service - + 10
# delivery_count > 3 → işlenemiyor → DLQ'ya taşı, sil
```

---

**Sorun 6: Redis Cluster'da stream key tek shard'a düştü — hotspot**

```bash
# Senaryo:
# Redis Cluster: 3 master, 3 slave.
# Stream key: "orders" → hash slot 9189 → sadece Master 2.
# Tüm XADD ve XREADGROUP işlemleri Master 2'ye gidiyor.
# Master 2: CPU %90, diğerleri %10 → hotspot.

# Redis Streams ve Cluster:
# Bir stream key → bir hash slot → bir shard.
# Sharding yok! Yüksek yazma → tek node'u eziyor.

# Çözüm 1: Uygulama seviyesinde sharding
# Birden fazla stream key kullan, producerde yük dağıt:
shard = hash(orderId) % 4
XADD orders:{shard} * orderId {orderId} ...
# orders:0, orders:1, orders:2, orders:3 → 4 farklı slot

# Consumer group her shard için ayrı:
XGROUP CREATE orders:0 inventory-service 0
XGROUP CREATE orders:1 inventory-service 0
# Consumer her shard'ı ayrı thread'de oku

# Çözüm 2: Hash tag ile aynı slot'a koy (sharding değil, co-location)
# {orders}:0, {orders}:1 → hepsi aynı slot → hotspot devam eder
# Bu çözüm DEĞİL, sadece multi-key komutlar için kullanılır

# Çözüm 3: Standalone Redis (cluster değil) → tek büyük node
# < 100K msg/sn: standalone Redis yeterli, cluster overhead yok
# Sharding kritikse → Kafka (partition = doğal sharding)

# Monitoring:
redis-cli --cluster info → shard başına key/memory dağılımı
# Tek shard'da aşırı yük → hotspot tespiti
```

---

### Mülakat Soruları — Consumer Group

**Junior / Mid:**

5. Consumer group ile `>` ve `0` arasındaki fark nedir?

   > **Beklened:** `>` (greater-than işareti): gruba daha önce verilmemiş YENİ entry'leri al. Normal çalışma modu — sürekli yeni entry tüketmek için. `0` (veya herhangi bir ID): pending (verilmiş ama ACK edilmemiş) entry'leri al. Consumer restart'ta ilk iş: `0` ile kendi pending entry'lerini tüket → sonra `>` ile devam et. Örnek akış: consumer crash → yeniden başlat → `0` ile "daha önce aldım ama ACK edemedim" olanları işle → tükenince `>` ile normal moda geç.

6. `XCLAIM` ve `XAUTOCLAIM` ne zaman kullanılır?

   > **Beklened:** XCLAIM: belirli bir entry'nin sahipliğini başka consumer'a devreder. Belirtilen idle süreyi aşmış entry için geçerli. Manuel crash recovery. XAUTOCLAIM (Redis 6.2+): tek komutla birden fazla idle entry'yi toplu devret. Periyodik çalıştırılır (health check döngüsü). `XAUTOCLAIM orders group consumer-2 60000 0-0 COUNT 10` → 60sn+ idle en fazla 10 entry'yi consumer-2'ye ata. Hangi consumer çağırmalı: sağlıklı consumer'lardan biri (liveness check yoksa herhangi biri). Delivery count takibi: çok teslim edilmiş → poison entry → DLQ veya sil.

---

**Senior / Architect:**

7. Redis Streams'te exactly-once nasıl sağlanır? Mümkün müdür?

   > **Beklened:** Redis Streams natively at-least-once: ACK edilmemiş entry yeniden teslim edilebilir. Exactly-once yok — consumer crash → pending → XCLAIM → tekrar işlenir. Yaklaşım: idempotency. Entry ID'si her kayıt için benzersiz — işlem kaydına yaz. Tekrar gelince: "Bu ID'yi zaten gördüm → atla." Redis transaction (MULTI/EXEC): XACK + DB yazma atomik değil. Redisson veya Lua script: XACK + Redis'e idempotency key SET atomik. Gerçek exactly-once için: Kafka transactional API + Kafka Streams. Redis Streams'te "effectively exactly-once" = at-least-once + idempotency key.

8. Redis Streams'i ne zaman Kafka'ya tercih edersin?

   > **Beklened:** Redis Streams tercih: (1) Zaten Redis var, ek bileşen istemiyorsun. (2) <100K msg/sn throughput yeterli. (3) Hızlı prototip, MVP. (4) Mesaj boyutu küçük, kısa TTL (RAM sınırı). (5) Servis-içi modüller arası event. (6) IoT sensör verisi + zaman serisi birlikte (Redis TimeSeries entegrasyonu). Kafka tercih: (1) >100K msg/sn. (2) Haftalık/aylık retention (RAM değil disk). (3) Partition ile doğal sharding. (4) Kafka Streams / Flink ile stateful stream processing. (5) Multi-datacenter (MirrorMaker). (6) Schema Registry ekosistemi. Karar sorusu: "Redis zaten var mı ve ölçek < 100K msg/sn?" → Streams. Değilse Kafka.

---

## Bölüm 3: Persistence & Güvenilirlik

### Gerçek Hayat Sorunları

---

**Sorun 7: Redis restart → stream verisi kayboldu**

```
Senaryo:
  Redis: sadece RDB (snapshot) ile persistence.
  save 900 1 ayarı: 15 dakikada 1 değişiklik varsa kaydet.
  Gece 02:00'da Redis çöktü → son snapshot 01:47'de alınmış.
  13 dakikalık stream entry'leri (yaklaşık 50K event) kayboldu.
  Sabah audit log'larında boşluk → compliance problemi.

Redis Streams persistence = Redis'in genel persistence'ına bağlı:
  Persistence kapalı (default) → restart = tüm stream silinir.
  RDB: snapshot anlar arası kayıp mümkün.
  AOF: her komut log'lanır → neredeyse sıfır kayıp.
  Hybrid (AOF + RDB preamble): hızlı restart + az kayıp.

Düzeltme:
  appendonly yes
  appendfsync everysec   # max 1sn kayıp, dengeli performans
  aof-use-rdb-preamble yes  # hybrid mod

  # Audit log gibi kritik veriler için:
  appendfsync always     # her write → fsync (yavaş ama sıfır kayıp)
  
  # Replikasyon ekle:
  replica-of master-host 6379
  # Master crash → replica promote → stream verisi replica'da da var

  # Seçim:
  # Audit/compliance → AOF always + replica
  # Cache + stream hybrid → AOF everysec
  # Pure cache (kayıp tolere) → RDB veya persistence kapalı

Monitoring:
  INFO persistence → aof_enabled, rdb_last_save_time
  aof_last_rewrite_status: ok/err → AOF sağlığı
```

---

**Sorun 8: Poison entry — tüm consumer'lar loop'a girdi, delivery count patladı**

```bash
# Senaryo:
# Bir entry bozuk JSON içeriyor → parse exception.
# XACK edilmiyor → pending → XCLAIM ile başka consumer → o da fail → XCLAIM → döngü.
# XPENDING: delivery_count = 500 → bu entry 500 kez teslim edildi.
# Tüm consumer'lar zamanlarının %90'ını bu entry ile geçiriyor.

# Teşhis:
XPENDING orders inventory-service - + 5
# Entry ID                    Consumer    Idle(ms)   DeliveryCount
# 1705312800000-3              worker-1    1200       487
# → delivery_count 487 → poison entry

# Çözüm: delivery-count eşiğinde DLQ'ya taşı ve sil
def process_with_dlq(entry):
    try:
        process(entry)
        XACK orders inventory-service entry.id
    except Exception as e:
        pending_info = XPENDING orders inventory-service entry.id entry.id
        delivery_count = pending_info[0][3]
        
        if delivery_count >= 3:
            # DLQ stream'e kopyala
            XADD orders-dlq * original_id entry.id error str(e) payload entry.value
            # Orijinalden ACK et (artık işlenmiş sayılır)
            XACK orders inventory-service entry.id
            log.error("Poison entry moved to DLQ: %s", entry.id)
        # else: pending'de kal, XCLAIM ile başkası tekrar dener

# Quorum Queue'nun built-in poison message özelliği burada yok
# Redis Streams: manuel delivery count takibi gerekir

# Spring Data Redis: acknowledge + delivery count:
long deliveryCount = pendingMessage.getTotalDeliveryCount();
if (deliveryCount > 3) {
    redis.opsForStream().add(MapRecord.create("orders-dlq", dlqData));
    redis.opsForStream().acknowledge("orders", "inventory-service", record.getId());
}
```

---

### Mülakat Soruları — Persistence & Güvenilirlik

**Junior / Mid:**

9. Redis Streams mesaj güvenliği için hangi persistence modu seçilmeli?

   > **Beklened:** Redis Streams persistence = Redis persistence. RDB: snapshot → snapshot anlar arası kayıp. Kritik stream için yetersiz. AOF everysec: max 1sn kayıp, dengeli. Çoğu production senaryosu için yeterli. AOF always: her write fsync, sıfır kayıp ama yavaş. Compliance/audit → bu mod. Hybrid (AOF + RDB preamble): hızlı restart + az kayıp. Ek güvenlik: replikasyon — master crash → replica promote → veri kaybolmaz. Stream kritikliğine göre: "kayıp olursa olsun" → RDB. "Max 1sn" → AOF everysec. "Sıfır kayıp" → AOF always + replica.

10. Consumer grubunda bir entry çok fazla kez teslim edildi. Ne yaparsın?

    > **Beklened:** XPENDING ile delivery count'u kontrol et. Yüksek delivery count → poison entry (işlenemeyen, bad data, schema uyumsuzluğu). Çözüm: eşik belirle (örn. 3 deneme). Eşik aşıldı → DLQ stream'e kopyala (XADD orders-dlq), sonra XACK ile ana stream'den kaldır. DLQ monitör: ops ekibi veya ayrı servis inceler, düzeltilmiş veriyle retry. Redis Streams'te Quorum Queue gibi built-in delivery limit yok — uygulamada takip etmeli. `x-delivery-limit` analog: manuel delivery_count kontrol + DLQ mantığı.

---

**Senior / Architect:**

11. Redis Streams ile bir microservice event bus tasarlıyorsun. Ölçeklenebilirlik sınırını nasıl aşarsın?

    > **Beklened:** Redis Streams sınırları: tek key → tek shard → max ~100K msg/sn. RAM sınırlı → uzun retention imkânsız. Aşma stratejileri: (1) Uygulama sharding: `orders:0...orders:N` → N stream key → farklı Redis shard'ları. Consumer her shard için ayrı thread. (2) Partition by entity: `orders:customer:{id % N}` → müşteri bazlı. (3) MAXLEN ile kısa TTL → sadece son X entry RAM'de. (4) Tiered: Redis Streams (hot, son 1 saat) + Kafka/S3 (cold, arşiv). (5) Sınır aşıldıysa → Kafka'ya migrate. Karar: Kafka'ya geçiş noktası: throughput > 50K msg/sn veya retention > 24 saat veya consumer group > 20.

---

## Karma — Architect Seviyesi

12. **"Audit log sistemini Redis Streams ile kuruyorsun. Entry'lerin kaybolmaması, replay edilebilmesi ve eski entry'lerin temizlenmesi nasıl sağlanır?"**

    > **Beklened:** Kaybolmaması: AOF always + replica. Producer: XADD sonrasında okuma ile verify (veya Lua script ile atomic write+confirm). Replay: XRANGE audit-log `<start_id>` `<end_id>` → belirli zaman aralığını yeniden oku. Consumer group 0 offset → baştan başla. Eski entry temizleme: XADD ile MINID (zaman bazlı). `XADD audit-log MINID ~ <90gün öncesi timestamp> * ...` → 90 günden eski entry'leri temizle. Arşivleme: cron job ile eski entry'leri XRANGE ile çek → S3/PostgreSQL'e yaz → sonra XTRIM. Consumer group monitoring: XPENDING → unacked entry'ler alert. DLQ: parse edilemeyen entry'ler `audit-log-dlq`'ya. Pratik stack: Redis 7 + AOF everysec + replica + MAXLEN ~ 1M + 90 günlük MINID trim.

13. **"Redis Streams, RabbitMQ ve Kafka arasında seçim yapman gerekiyor. Hangi kriterlere bakarsın?"**

    > **Beklened:** Redis Streams: zaten Redis varsa, <50K msg/sn, kısa retention (saatler/günler), basit consumer group, hızlı prototip, servis-içi event. Ek bileşen yok. RabbitMQ: kompleks routing (topic exchange, headers), öncelikli mesaj, TTL/delayed message, task queue (bir worker alsın), request-reply, orta ölçek. Mesaj consume edilince gitsin. Kafka: >100K msg/sn, haftalık/aylık retention, birden fazla consumer group bağımsız replay, event sourcing, stream processing (Kafka Streams/Flink), schema registry, multi-DC. Sorular: "Replay lazım mı?" → Kafka veya Streams. "Karmaşık routing?" → RabbitMQ. "Zaten Redis altyapısı var, <50K msg/sn?" → Streams. "Büyük ölçek, long retention?" → Kafka. Hiçbir zaman sadece bir araç değil — mimari ihtiyaç belirler.
