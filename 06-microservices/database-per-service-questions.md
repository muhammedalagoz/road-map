# 06a — Database per Service: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Paylaşımlı DB Anti-Pattern

### Gerçek Hayat Sorunları

---

**Sorun 1: Paylaşımlı DB — bir servisin migrasyonu diğerlerini çökertti**

```
Senaryo:
  3 microservice, tek PostgreSQL: OrderService, PaymentService, UserService.
  OrderService ekibi: "orders tablosuna yeni sütun ekleyelim + büyük index."

  Migration:
    ALTER TABLE orders ADD COLUMN estimated_delivery DATE;
    CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id, created_at);

  Sorun:
    CREATE INDEX CONCURRENTLY: tablo üzerinde ShareUpdateExclusiveLock tutar.
    Yüksek trafik altında: index oluşturma 20 dakika sürdü.
    Bu 20 dakika boyunca: orders tablosuna yazma yavaşladı.
    PaymentService: sipariş durumunu orders tablosundan okuyor (join!).
    PaymentService: her sorgu beklemede → timeout → ödeme akışı durdu.
    UserService: kullanıcı sipariş geçmişi → aynı tablo → timeout.

  Kök neden:
    Paylaşımlı DB: bir servisin migration'ı diğer servislerin tablolarını etkiledi.
    Cross-service JOIN: servisler aynı tabloya bağımlı.

Database per Service ile:
  OrderService → kendi order_db'si.
  Migration: sadece OrderService'i etkiler.
  PaymentService: kendi payment_db'si → OrderService migration'ından haberi yok.
  UserService: kendi user_db'si → izole.

  Cross-service veri: API çağrısı veya event replikasyonu.
  Migration bağımsızlığı: takımlar birbirini bloke etmez.
```

---

**Sorun 2: Paylaşımlı DB — bir servisin connection pool tükenmesi tüm servisleri çökertti**

```
Senaryo:
  Paylaşımlı PostgreSQL: max_connections = 200.
  3 servis, her biri HikariCP pool = 80 bağlantı → toplam potansiyel: 240.
  Normalde: hepsi 80'i kullanmıyor → toplam ~120 aktif.

  Black Friday: AnalyticsService heavy query çalıştırdı.
    SELECT * FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON oi.product_id = p.id
    WHERE o.created_at > '2024-01-01'   -- 10 milyon satır
    GROUP BY p.category                  -- 30 saniye sürdü

  30 saniye × 80 bağlantı = AnalyticsService 80 bağlantıyı doldurdu.
  OrderService, PaymentService: bağlantı almak için sırada bekledi.
  HikariCP timeout (30s): bağlantı alınamadı → exception.
  OrderService: sipariş oluşturulamıyor. PaymentService: ödeme alınamıyor.

  Kayıp: 45 dakika boyunca sipariş + ödeme akışı durdu.

Database per Service ile:
  AnalyticsService → analytics_db (ayrı, optimize edilmiş read replica).
  OrderService → order_db: kendi bağlantı havuzu, analitik sorgular etkilemez.
  Hata izolasyonu: AnalyticsService çöküşü → sadece AnalyticsService etkili.

Ayrıca: AnalyticsService için OLAP DB (ClickHouse, Redshift) → aggregate sorgular için optimize.
```

---

**Sorun 3: Cross-service JOIN — schema coupling, bağımsız deployment imkânsız**

```
Senaryo:
  Paylaşımlı DB'de OrderService ve UserService aynı şemada:

  -- OrderService'in sorgusu:
  SELECT o.id, o.total, u.name, u.email
  FROM orders o
  JOIN users u ON o.customer_id = u.id   -- UserService'in tablosu!
  WHERE o.status = 'PENDING';

  Sorun 1: Schema coupling.
    UserService: "users tablosuna sütun ekleyeceğiz" → OrderService'i bozabilir.
    UserService: "users.name → users.full_name" rename → OrderService sorgusu patlar.
    İki servis bağımsız değil: birini değiştirmek diğerini kırar.

  Sorun 2: Bağımsız deployment imkânsız.
    UserService yeni şema → OrderService da güncellenmeli → aynı anda deploy.
    "Bağımsız deployment" microservice'in temel vaadi → ihlal edildi.

  Sorun 3: Farklı teknoloji seçimi yok.
    UserService → Redis'e (hız için) taşımak istedi.
    Ama OrderService JOIN kullanıyor → Redis'te JOIN yok → taşıma imkânsız.

Database per Service çözümü:

  Seçenek A — API Çağrısı:
    OrderService: userId listesini UserService'e gönderir.
    UserService: kullanıcı detaylarını döndürür.
    JOIN → iki ayrı çağrı + uygulama seviyesinde merge.
    Dezavantaj: network overhead, UserService down → OrderService etkilenir.

  Seçenek B — Event replikasyonu:
    UserService: kullanıcı oluştu/güncellendi → event yayınlar.
    OrderService: kendi DB'sinde customer_snapshot tablosu tutar (id, name, email).
    JOIN → kendi DB'si içinde (snapshot ile).
    Avantaj: UserService down olsa bile OrderService çalışır.
    Dezavantaj: eventual consistency (snapshot biraz stale olabilir).
```

---

## Bölüm 2: Veri Tutarlılığı & Saga

### Gerçek Hayat Sorunları

---

**Sorun 4: Saga compensating transaction eksik — stok düşüldü ama ödeme başarısız**

```
Senaryo:
  Sipariş akışı:
    1. OrderService: sipariş oluştur → order_db: PENDING
    2. InventoryService: stok azalt → inventory_db: quantity - 1
    3. PaymentService: ödeme al → HATA (kart reddedildi)

  Compensating transaction YAZILMADI:
    PaymentService: ödeme başarısız → inventory.failed event yayınlandı.
    InventoryService: "ne yapayım?" → handler yok → stok geri verilmedi.
    OrderService: sipariş PENDING'de kaldı.

  Sonuç:
    Stok: 1 azaltıldı ama ürün satılmadı → phantom stok kaybı.
    Sipariş: PENDING → kullanıcı "sipariş ne oldu?" diye merak ediyor.

Compensating transaction ile düzeltme:

  Choreography tabanlı Saga:

  // InventoryService: PaymentFailed event'ini dinle
  @KafkaListener(topics = "payment.payment-failed")
  void onPaymentFailed(PaymentFailedEvent event) {
      // Compensating: stoku geri ver
      inventoryRepository.increaseStock(
          event.getProductId(),
          event.getQuantity()
      );
      // Compensating event yayınla
      eventBus.publish(new InventoryReleasedEvent(event.getOrderId()));
  }

  // OrderService: InventoryReleased event'ini dinle
  @KafkaListener(topics = "inventory.inventory-released")
  void onInventoryReleased(InventoryReleasedEvent event) {
      // Compensating: siparişi iptal et
      orderRepository.updateStatus(event.getOrderId(), OrderStatus.CANCELLED);
  }

  Kural: her Saga adımı için mutlaka compensating transaction yaz.
  Test: hata senaryolarını da test et (happy path yeterli değil).
  Idempotency: compensating transaction iki kez çağrılırsa sorun olmamalı.
```

---

**Sorun 5: Event replikasyonu — stale fiyat ile sipariş oluşturuldu**

```
Senaryo:
  CatalogService: ürün fiyatlarını event ile yayınlıyor.
  OrderService: kendi DB'sinde product_prices tablosu (read model).

  T+0:   CatalogService: fiyat güncellendi (100 TL → 150 TL) → event yayınlandı.
  T+0:   Kafka: event partition lag = 5 saniye (anlık yüksek trafik).
  T+3s:  Kullanıcı sipariş verdi.
  T+3s:  OrderService: product_prices tablosundan okudu → 100 TL (eski).
  T+5s:  Event geldi → product_prices güncellendi → 150 TL.
  T+5s:  Sipariş zaten oluşturuldu: 100 TL üzerinden.

  İş problemi: 50 TL daha ucuza sipariş kabul edildi.

Çözüm stratejileri:

  1. Kabul et + iş kararı:
     "5 saniyelik stale fiyat tolere edilir."
     Kampanya veya fiyat güncelleme sonrası: yeni siparişe süre koy.
     "Fiyat değişiminden 30s sonra geçerli" → event gecikmesini absorbe eder.

  2. Fiyat doğrulama API çağrısı (kritik sipariş anında):
     Sipariş oluşturulurken: CatalogService'ten güncel fiyatı al (senkron).
     Read model: hızlı arama için → fiyat validation: API çağrısı.
     Hybrid: iki aşamalı → hız + doğruluk dengesi.

  3. Event ordering garantisi:
     Kafka: aynı ürün fiyat event'leri aynı partition'a gönder.
     OrderService: consumer lag'ını izle.
     Alert: "fiyat event lag > 2s" → uyarı.

  4. Price lock at checkout:
     Kullanıcı sepete eklediğinde fiyatı "lock" et (Redis, TTL=10dk).
     Ödeme sırasında: lock edilen fiyatı kullan.
     Fiyat değişiminden etkilenmez (lock süresi içinde).
```

---

**Sorun 6: Strangler Fig migration — dual-write sırasında veri tutarsızlığı**

```
Senaryo:
  Monolith'ten ayrılma: PaymentService için ayrı DB'ye geçiş.
  Dual-write stratejisi: hem eski DB hem yeni DB'ye yaz.

  Adım 3 (fiziksel ayrım):
    PaymentService: eski DB + yeni DB'ye aynı anda yazıyor.
    Eski DB: payments tablosu (monolith paylaşımlı).
    Yeni DB: payment_db (ayrı PostgreSQL instance).

  Sorun:
    T+0:   Ödeme yazıldı → eski DB: OK.
    T+1ms: Yeni DB yazma: network timeout → başarısız.
    T+1ms: PaymentService: "ödeme başarılı" döndürdü (eski DB OK'dı).
    Sonuç: eski DB'de kayıt var, yeni DB'de yok.
    Cutover: yeni DB'ye geçildi → bu ödeme kayıp!

Dual-write doğru yapma:

  Seçenek A — Outbox pattern:
    PaymentService: eski DB'ye yaz + outbox tablosuna event yaz (aynı transaction).
    CDC (Debezium): outbox'ı izler → yeni DB'ye iletir.
    Atomik: eski DB transaction başarılıysa event de var → yeni DB'ye gider.
    Rollback: eski DB rollback → event de yok → yeni DB'ye gitmez.

  Seçenek B — Event-driven migration:
    1. Eski DB'ye yaz (tek kaynak of truth).
    2. Event yayınla → yeni DB'ye async yaz.
    3. Yeni DB'ye geçiş: event replay → eski tüm veriyi yeni DB'ye çek.
    4. Doğrula: iki taraf tutarlı mı?
    5. Cutover: yeni DB primary, eski readonly → sonra kapat.

  Doğrulama önemi:
    Her N satırda bir: eski DB vs yeni DB checksum karşılaştır.
    Tutarsız kayıt → alarm → düzelt → sonra cutover.
    "Aşağı yukarı tutarlı" yetmez → tam tutarlılık doğrula.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Database per Service nedir? Neden gereklidir?

   > **Beklened:** Her microservice kendi veritabanını yönetir, diğer servisler doğrudan erişemez — sadece API üzerinden. Neden: (1) Bağımsız ölçekleme: CatalogService yoğun okuma → Elasticsearch. OrderService ACID → PostgreSQL. Tek DB: hepsine aynı çözüm → uygun olmayabilir. (2) Bağımsız deployment: paylaşımlı DB'de migration tüm servisleri etkiler. Ayrı DB: migration sadece o servisi etkiler. (3) Hata izolasyonu: bir servis bağlantı havuzunu tüketirse → sadece o servis etkilenir, diğerleri değil. (4) Teknoloji özgürlüğü: her servis en uygun DB'yi seçer (polyglot persistence). Zorluklar: cross-service JOIN yok → API çağrısı veya event replikasyonu. Distributed transaction yok → Saga. Eventual consistency → iş mantığının kabul etmesi gerekir.

2. Database per Service'te cross-service veri paylaşımı nasıl yapılır? İki yaklaşımı karşılaştır.

   > **Beklened:** Yaklaşım 1 — Senkron API çağrısı: OrderService → CatalogService'e HTTP/gRPC çağrısı yaparak fiyat alır. Avantaj: her zaman güncel veri. Dezavantaj: CatalogService down → OrderService de çalışmaz (tight coupling). Latency eklenir, cascade failure riski. Yaklaşım 2 — Event-driven replikasyon: CatalogService fiyat değişince event yayınlar. OrderService event'i dinler, kendi DB'sinde product_prices tablosu (read model) tutar. Sipariş oluştururken kendi yerel kopyasını okur. Avantaj: CatalogService down olsa bile çalışır (loose coupling). Dezavantaj: eventual consistency → fiyat birkaç saniye stale olabilir. Seçim: coupling tolere edilemiyorsa (kritik bağımlılık) → event. Güncel veri şart, stale tolere edilemiyorsa → API çağrısı (ama fallback + circuit breaker ekle).

3. "Shared Database" anti-pattern nedir? Ne zaman geçici olarak kabul edilebilir?

   > **Beklened:** Shared Database: birden fazla microservice aynı DB'yi (veya şemayı) paylaşır. Anti-pattern çünkü: migration coupling (bir servis şema değiştirir → diğeri etkilenir), bağımsız deployment imkânsız, teknoloji değiştirme imkânsız, hata izolasyonu yok. Ne zaman geçici kabul edilir: monolith → microservices geçiş dönemi (Strangler Fig başlangıcı). Takım küçük, operasyonel yük çok. Domain henüz netleşmemiş (sık değişiyor). Süre: geçici → plan yap, zaman çizelgesi koy. "Geçici" sonsuza uzamasın. Çıkış stratejisi: ayrı şema → ayrı fiziksel DB → event-driven replikasyon. Long-term paylaşımlı DB: microservice'in temel faydalarını yok eder.

---

**Senior / Architect:**

4. Polyglot persistence nedir? E-ticaret sisteminde hangi servis hangi DB'yi seçmeli?

   > **Beklened:** Polyglot persistence: her servis ihtiyacına en uygun DB teknolojisini seçer — tek tip DB kullanmak zorunda değil. E-ticaret örneği: UserService → PostgreSQL (ACID, relational, auth, güvenlik). CatalogService → Elasticsearch (full-text search, filter, faceted search). OrderService → PostgreSQL (transaction, ACID, audit log). CartService → Redis (hız, TTL, ephemeral — sepet birkaç saat geçerli). SessionService → Redis (token, TTL, çok hızlı okuma). AnalyticsService → ClickHouse veya Redshift (OLAP, aggregate sorgular). RecommendationService → Neo4j (graph DB, kullanıcı-ürün ilişkisi). NotificationService → MongoDB (flexible schema, log, event history). PaymentService → PostgreSQL (ACID, double-entry, audit). Avantaj: her servis optimal DB → performans, maliyet, özellik uyumu. Dezavantaj: operasyonel yük artar (farklı DB teknolojileri → farklı expertise, monitoring, backup).

5. Saga pattern ile 2PC farkını açıkla. Microservices'te neden Saga tercih edilir?

   > **Beklened:** 2PC (Two-Phase Commit): Prepare + Commit. Coordinator tüm katılımcıları koordine eder. Sorunlar: blocking protocol (coordinator çökerse tüm servisler askıda), lock süresince tüm katılımcılarda kaynak tutulur, microservicesta farklı DB teknolojileri 2PC desteklemeyebilir, tight coupling. Saga: her adım local transaction + event. Hata → compensating transaction (geri alma). Choreography: servisler birbirinin event'lerini dinler (loose coupling, debug zor). Orchestration: merkezi Saga koordinatörü (debug kolay, orchestrator SPOF). Neden Saga tercih: lock yok → yüksek throughput. Farklı DB teknolojileri → heterogen sistem. Microservice bağımsızlığı korunur. Dezavantajlar: eventual consistency, compensating transaction yazma zorluğu, debug karmaşık (distributed tracing şart). Her Saga adımı idempotent olmalı.

6. Monolith'ten Database per Service'e geçişi nasıl yönetirsin? Dual-write risklerini nasıl azaltırsın?

   > **Beklened:** Strangler Fig migration adımları: (1) Şema ayrımı: tek DB'de ayrı şema/kullanıcı (hâlâ aynı server). Servis sadece kendi şemasına erişir. Cross-schema JOIN'leri API çağrısına dönüştür. (2) Servis izolasyonu: servis sadece kendi şemasına yazar. Diğer şemalardan okuma → event replikasyonuna taşı. (3) Fiziksel ayrım (dual-write): Outbox pattern → eski DB'ye yaz + outbox event → CDC (Debezium) → yeni DB. Atomik: eski DB transaction ile outbox aynı anda → tutarlı. (4) Doğrulama: eski DB vs yeni DB checksum. Tutarsız kayıt → düzelt → tekrar kontrol. (5) Cutover: yeni DB primary, eski DB read-only. Trafik yeni DB'ye yönlendir. (6) Temizlik: eski şema/DB kapat. Riskler: dual-write tutarsızlığı → Outbox ile azalt. Eksik migration → checksum ile tespit. Rollback planı: eski DB hâlâ ayakta, sorun olursa geri dön.

---

## Karma — Architect Seviyesi

7. **"OrderService, UserService'in müşteri verilerine ihtiyaç duyuyor. Nasıl tasarlarsın?"**

   > **Beklened:** İlk soru: ihtiyaç ne? Sipariş oluşturma anında mı? Sipariş listesi gösterirken mi? Sıklık ne? Tasarım kararı — hibrit yaklaşım önerim: Read model (event replikasyonu): UserService → user.created, user.updated event'leri yayınlar. OrderService: customer_snapshot tablosu (customer_id, name, email, phone). Sipariş listesi için: yerel snapshot → UserService'e bağımlılık yok, hızlı. Sipariş oluşturma (kritik an): API çağrısı → anlık doğrulama (hesap aktif mi? ban'lı mı?). Sonuç: okuma → snapshot (hızlı, resilient). Yazma/kritik kontrol → API (güncel, doğru). Cache: UserService API çağrısı için Redis (TTL=5dk) → sık tekrar eden kullanıcılar için. Circuit Breaker: UserService down → cache'ten dön veya "kullanıcı bilgisi geçici olarak alınamıyor" — siparişi bloke etme. Monitoring: snapshot lag, UserService API error rate, cache hit rate.

8. **"5 yıllık monolith, tek dev DB. Ekip 'database per service'e geçelim' diyor. Ne söylersin?"**

   > **Beklened:** Hemen "evet, yapalım" demem. Önce analiz: (1) Neden? Gerçek problem nedir? Deployment bağımlılığı mı? Ölçekleme mi? Ekip bağımsızlığı mı? Farklı DB ihtiyacı mı? Problemi net tanımla. (2) Maliyet: kaç servis? Kaç tablo? Domain sınırları net mi? Belirsiz domain → migration sırasında sürekli yeniden çizme. (3) Ekip kapasitesi: Saga pattern biliniyor mu? Distributed tracing var mı? Operasyonel yük artacak. (4) Alternatif değerlendir: Ayrı şema (aynı DB) → deployment bağımsızlığı için yeterli olabilir. Read replica → ölçekleme için yeterli olabilir. Tüm servisleri ayırmak zorunda değilsin. (5) Eğer karar "geç": Strangler Fig — adım adım. En iyi ayrışan servisten başla (az bağımlılık, net domain). Dual-write + Outbox + doğrulama. Acele etme: 1 servisi doğru yap, sonra genişlet. Anti-pattern: tüm monolith'i 1 ayda microservice'e çevir → "Distributed Monolith" riski.
