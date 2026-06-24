# 08p — System Design: Reklam Tıklama Takip Sistemi

## Gereksinimler

```
Functional:
  ✓ Reklam tıklamalarını kayıt altına al
  ✓ Gerçek zamanlı tıklama sayısı (son 1 dk, 1 saat)
  ✓ Günlük/aylık aggregated rapor
  ✓ Tıklama başına ödeme (CPC) hesabı
  ✓ Click fraud tespiti
  ✓ Duplicate tıklama önleme (exactly-once counting)
  ✗ Reklam serving, targeting (out of scope)

Non-Functional:
  1B tıklama/gün
  Latency: tıklama kaydı < 10ms (kullanıcıyı bloke etmemeli)
  Aggregation sorgu: < 1s
  Exactly-once counting (fraud ve duplicate önleme)
  Hot advertiser (büyük kampanya) → spike trafiği
  Availability: %99.99 (reklam kaydı — gelir kaybı riski)
```

---

## Capacity Estimation

```
Tıklama QPS:
  1B / 86400 ≈ 11,600/sn ortalama
  Peak: 11,600 × 5 = 58,000/sn (Black Friday, büyük kampanya)

Storage:
  Raw click: {clickId, adId, userId, timestamp, ip, userAgent, country, device} = 200 bytes
  1B × 200B = 200 GB/gün
  90 gün retention: 18 TB (cold storage → S3 + ClickHouse)

Aggregated storage (much smaller):
  1M ad × 24 saat × 100B = 2.4 GB/gün (raw'un 1/80'i)

Kafka topic boyutu:
  click.raw: 58,000 msg/sn × 200B = 11.6 MB/sn
  3 günlük retention: 11.6 MB/sn × 259,200s = ~3 TB
  Partition sayısı: 58,000 / 1,000 (partition throughput) = 58 → 64 partition

Redis memory (fraud detection):
  Active ad sayısı: 1M
  Per-ad Redis key: 100 bytes × 2 key/ad = 200 bytes
  1M ad: 200 MB → Redis Fleet'te rahatlıkla sığar

Kafka consumer lag:
  Aggregation consumer: 64 partition, 10ms/msg → 64 × 10ms = 640ms gecikme
  Hedef: < 1 sn real-time lag → yeterli
```

---

## High-Level Tasarım

```
User clicks ad
      │
      ▼
  ┌──────────────────────────────────────────┐
  │         API Gateway / Load Balancer      │
  │         (Rate limiting per IP)           │
  └──────────────────┬───────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────┐
  │          Click API (ultra-low latency)   │
  │  1. Dedup check (Redis Bloom Filter)     │
  │  2. Async Kafka publish (fire-and-forget)│
  │  3. 302 redirect → destination           │
  └──────────────────┬───────────────────────┘
                     │
                     ▼
          ┌──────────────────┐
          │      Kafka       │  ← click.raw topic (64 partition)
          └────────┬─────────┘
                   │
    ┌──────────────┼──────────────────┐
    ▼              ▼                  ▼
[Fraud         [Real-time         [Batch
 Detector]      Aggregator]        Writer]
 (Flink)        (Kafka Streams)    (Spark/Flink)
    │              │                  │
    ▼              ▼                  ▼
[click.valid   [Redis             [ClickHouse
 topic]         live counts]       OLAP reports]
                                      │
                                      ▼
                                 [Report API]
                                 [CPC Engine]
                                 [Billing]
```

---

## Click API — Ultra-Low Latency & Deduplication

```
Kullanıcı tıkladı → 10ms'den fazla beklememeli

Akış:
  GET /click?adId=ad-123&redirectUrl=https://advertiser.com/product

  Click API:
  1. Bloom Filter dedup check (Redis) — 1ms
  2. clickId = UUID oluştur — 5μs
  3. Kafka'ya async publish (non-blocking) — <1ms
  4. 302 redirect → kullanıcı gidiyor, işlem arka planda

  Kritik: Kafka publish başarısız olsa bile redirect yap.
  "Fire and forget" — kullanıcı deneyimi > veri kaybı riski.
  Kafka güvenilirliği: %99.99 → kayıp ihmal edilebilir.
```

```java
@GetMapping("/click")
void trackClick(@RequestParam String adId,
                @RequestParam String redirectUrl,
                HttpServletRequest req,
                HttpServletResponse res) throws IOException {

    String clickId = UUID.randomUUID().toString();

    // 1. Bloom Filter ile hızlı dedup (false positive olabilir, false negative yok)
    String deduKey = buildDeduKey(extractUserId(req), adId);
    if (bloomFilter.mightContain(deduKey)) {
        // Muhtemelen duplicate → Kafka'ya işaretle, count etme
        res.sendRedirect(redirectUrl);
        return;
    }
    bloomFilter.put(deduKey);

    // 2. Async Kafka publish (non-blocking)
    kafkaTemplate.send("click.raw",
        adId,   // partition key = adId (aynı ad → aynı partition → sıralı)
        ClickEvent.builder()
            .clickId(clickId)
            .adId(adId)
            .userId(extractUserId(req))
            .sessionId(extractSessionId(req))
            .ip(getClientIp(req))
            .userAgent(req.getHeader("User-Agent"))
            .country(geoIpService.getCountry(getClientIp(req)))
            .deviceType(deviceDetector.detect(req.getHeader("User-Agent")))
            .timestamp(Instant.now())
            .build()
    );

    // 3. Hemen redirect (Kafka'yı bekleme)
    res.sendRedirect(redirectUrl);
}
```

---

## Exactly-Once Counting & Deduplication

```
Problem: Kullanıcı yanlışlıkla çift tıkladı, ağ retry, bot — aynı click birden sayılmasın.

Katmanlı deduplication stratejisi:

Katman 1 — Click API (Bloom Filter, <1ms):
  Redis Bloom Filter: probabilistic veri yapısı.
  key = hash(userId + adId + 5dk_pencere)
  False positive olabilir (bazı gerçek tıklamalar atlanır — kabul edilebilir).
  False negative yok (kesinlikle duplicate olan → geçmez).
  Memory: 1M element, %1 FP oranı → ~1.2 MB (çok küçük).

Katman 2 — Kafka Consumer (Exact dedup):
  Flink stateful processing:
  clickId'yi state store'da sakla (1 saat TTL).
  Aynı clickId tekrar gelirse → drop.
  Tam dedup — kesin sonuç.

  // Flink KeyedProcessFunction
  public void processElement(ClickEvent click, Context ctx, Collector<ClickEvent> out) {
      ValueState<Boolean> seen = getRuntimeContext()
          .getState(new ValueStateDescriptor<>("seen", Boolean.class));
      
      if (seen.value() != null) {
          // Duplicate → drop
          return;
      }
      seen.update(true);
      ctx.timerService().registerProcessingTimeTimer(
          ctx.timerService().currentProcessingTime() + 3_600_000); // 1 saat TTL
      out.collect(click);
  }

  public void onTimer(long timestamp, OnTimerContext ctx, Collector<ClickEvent> out) {
      getRuntimeContext().getState(...).clear(); // TTL doldu → state temizle
  }

Katman 3 — ClickHouse (Idempotent upsert):
  ClickHouse ReplacingMergeTree:
  INSERT INTO clicks VALUES (...) — aynı clickId birden insert edilse bile.
  Background merge: duplicate'ları birleştirir (eventual dedup).
  Sorgu: FINAL keyword → anlık dedup sonucu.

  CREATE TABLE clicks (
      click_id  String,
      ...
  ) ENGINE = ReplacingMergeTree()
  ORDER BY (click_id);

  -- Duplicate-free sorgu:
  SELECT count() FROM clicks FINAL WHERE ad_id = 'ad-123';
```

---

## Hot Ad Problemi (Hotspot)

```
Senaryo: Apple yeni iPhone lansmanı — tek bir adId milyonlarca tıklama.
Tüm tıklamalar aynı Kafka partition'a gidiyor (partition key = adId).
O partition'ın consumer'ı → bottleneck → lag birikir.

Çözüm 1 — Partition key salting:
  Normal: partition key = adId → tek partition
  Salted:  partition key = adId + "_" + (clickId.hashCode() % 8)
  8 partition'a dağılım → 8x throughput.

  Dezavantaj: Aynı adId farklı partition'larda → sıralı okuma yok.
  Aggregation: tüm partition'ları merge et.
  Bu senaryo için kabul edilebilir (sıra önemli değil, count önemli).

Çözüm 2 — Hot ad tespiti + ayrı topic:
  Önceki gün top 1000 reklam → "hot" olarak işaretle.
  Hot ad → click.hot topic (daha fazla partition).
  Normal ad → click.normal topic.
  Click API: Redis'te hot ad listesi → hangi topic'e göndereceğini bilir.

Çözüm 3 — Local pre-aggregation:
  Click API'de memory buffer: her adId için local counter.
  100ms'de bir → batch olarak Kafka'ya gönder.
  1 event yerine: {"adId": "ad-123", "count": 47, "window": "..."}
  Kafka'ya giden mesaj sayısı 47x azaldı.

  Dezavantaj: API node crash → 100ms buffer kaybı.
  Trade-off: veri kaybı riski vs throughput (kabul edilebilir).
```

---

## Click Fraud Tespiti

```
Fraud türleri:
  1. Same user/IP çok fazla tıklama (competitor, bot farm)
  2. Bot traffic (headless browser, script)
  3. Data center / proxy / VPN IP'leri
  4. Click farms (gerçek insanlar, çok düşük engagement)
  5. Invalid traffic (tarayıcı açık değil, spider, scraper)

Real-time fraud rules (Kafka Streams / Flink, <1s karar):

  Rule 1: Aynı kullanıcı + aynı ad, 5 dk içinde 3+ tıklama
    Redis: INCR "u:{userId}:a:{adId}:clicks" EX 300
    count > 3 → fraud flag

  Rule 2: Aynı IP, 1 dakikada 100+ tıklama
    Redis: INCR "ip:{ip}:clicks:{minute}" EX 60
    count > 100 → IP geçici blacklist (1 saat)

  Rule 3: User Agent kara liste
    python-requests, curl/7, wget, scrapy → anında block
    Headless browser imzaları:
      navigator.webdriver == true (Puppeteer/Playwright leak)
      User-Agent'ta "HeadlessChrome" → şüpheli

  Rule 4: IP Reputation (3rd party)
    MaxMind GeoIP2: data center IP? → fraud score +50
    IPQualityScore: Tor exit node, known proxy → block
    API call: 5ms timeout, Redis cache 1 saat

  Rule 5: Conversion rate anomalisi
    CTR (click-through rate) çok yüksek → şüpheli
    Normal CTR: %0.1 - %3. %30+ → fraud

ML tabanlı fraud (offline, batch):
  Feature vector:
    - Click velocity (son 1/5/60 dk tıklama sayısı)
    - Session duration
    - Mouse movement pattern (varsa)
    - IP diversity (aynı advertiser, farklı IP'ler)
    - Time pattern (gece 03:00 burst?)
    - Conversion rate (reklam → satın alma oranı)
  
  Model: Gradient Boosting (LightGBM) → fraud_score 0.0-1.0
  score > 0.8 → otomatik fraud
  score 0.5-0.8 → manuel review queue
  Günlük model yenileme → Spark ML pipeline

  False positive sorunu:
    Gerçek tıklama fraud diye işaretlendi → advertiser şikayet.
    Çözüm: fraud kararı reversible — audit queue → human review.
    Advertiser portal: "Bu tıklamayı itiraz et" → inceleme.
```

```java
// Kafka Streams — real-time fraud check
StreamsBuilder builder = new StreamsBuilder();

KStream<String, ClickEvent> clicks = builder.stream("click.raw");

clicks
    .filter((adId, click) -> !isBot(click.getUserAgent()))
    .filter((adId, click) -> !isBlacklistedIp(click.getIp()))
    .mapValues(click -> {
        // Redis fraud check
        String userAdKey = "u:" + click.getUserId() + ":a:" + click.getAdId();
        Long count = redis.opsForValue().increment(userAdKey);
        redis.expire(userAdKey, 5, TimeUnit.MINUTES);

        if (count > 3) {
            click.setFraud(true);
            click.setFraudReason("DUPLICATE_USER_AD");
        }

        String ipKey = "ip:" + click.getIp() + ":m:" + currentMinute();
        Long ipCount = redis.opsForValue().increment(ipKey);
        redis.expire(ipKey, 60, TimeUnit.SECONDS);

        if (ipCount > 100) {
            click.setFraud(true);
            click.setFraudReason("IP_VELOCITY");
        }
        return click;
    })
    .split()
    .branch((k, v) -> !v.isFraud(), Branched.withConsumer(s -> s.to("click.valid")))
    .branch((k, v) ->  v.isFraud(), Branched.withConsumer(s -> s.to("click.fraud")));
```

---

## Real-Time Aggregation (Kafka Streams)

```java
// 1 dakikalık tumbling window ile tıklama sayısı
StreamsBuilder builder = new StreamsBuilder();

builder.stream("click.valid", Consumed.with(Serdes.String(), clickSerde))
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count(Materialized.as("click-counts-1m"))
    .toStream()
    .foreach((windowedKey, count) ->
        redis.opsForValue().set(
            "ad:" + windowedKey.key() + ":1m:" + windowedKey.window().start(),
            count.toString(),
            Duration.ofMinutes(10)  // 10 dk TTL
        )
    );

// Saatlik tumbling window
builder.stream("click.valid")
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
    .count(Materialized.as("click-counts-1h"))
    .toStream()
    .foreach((windowedKey, count) ->
        redis.opsForValue().set(
            "ad:" + windowedKey.key() + ":1h:" + windowedKey.window().start(),
            count.toString(),
            Duration.ofHours(3)
        )
    );

// Sorgu API:
// GET /api/ads/{adId}/stats?window=1m  → Redis'ten
// GET /api/ads/{adId}/stats?window=1h  → Redis'ten
// GET /api/ads/{adId}/stats?from=2026-01-01&to=2026-01-31 → ClickHouse'dan
```

---

## Late-Arriving Events & Watermark

```
Problem: Network gecikmesi, mobil offline → tıklama 2 dakika sonra geldi.
Aggregation: 1 dakika window zaten kapandı → bu tıklama sayılmaz mı?

Watermark stratejisi:
  Watermark = event_time - allowed_lateness
  allowed_lateness = 2 dakika (2 dk geç gelen → hâlâ kabul et)

  Event time: tıklama gerçekleşme zamanı (client timestamp)
  Processing time: Kafka'ya gelme zamanı

  Flink watermark:
    WatermarkStrategy
        .forBoundedOutOfOrderness(Duration.ofMinutes(2))
        .withTimestampAssigner((click, ts) -> click.getTimestamp().toEpochMilli())

  Allowed lateness penceresi:
    Window [12:00 - 12:01] → watermark 12:03'e ulaşana kadar açık.
    12:02:30'da gelen tıklama → window'a dahil edilir.
    12:03:01'de gelen tıklama → "side output" (geç veri) → ayrı işlem.

Side output (çok geç gelen veriler):
  Late click → click.late topic → batch job ile ClickHouse'a corrected olarak yaz.
  Raporlar: "corrected" flag ile güncellenebilir (T+1 günün raporu düzeltilebilir).

Event time vs Processing time seçimi:
  Fraud detection: processing time (anında karar gerekli).
  Billing: event time (gerçek tıklama zamanı önemli).
  Real-time dashboard: processing time (yaklaşık ama anlık).
  Resmi rapor: event time (doğru, delayed olabilir).
```

---

## Batch Aggregation (ClickHouse)

```sql
-- ClickHouse: OLAP, column-store, aggregation odaklı
-- 1B row/sn insert, milisaniye aggregation

CREATE TABLE clicks (
    click_id     String,
    ad_id        String,
    user_id      String,
    campaign_id  String,
    country      LowCardinality(String),  -- az unique değer → optimize
    device_type  LowCardinality(String),
    timestamp    DateTime,
    is_fraud     UInt8 DEFAULT 0,
    fraud_reason LowCardinality(String) DEFAULT ''
) ENGINE = ReplacingMergeTree()   -- duplicate clickId'leri merge
PARTITION BY toYYYYMM(timestamp)  -- aylık partition → eski veri prune kolay
ORDER BY (ad_id, timestamp)       -- sık kullanılan sort key
SETTINGS index_granularity = 8192;

-- Materialized View: saatlik pre-aggregation (sorgu hızlandır)
CREATE MATERIALIZED VIEW clicks_hourly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (ad_id, campaign_id, country, device_type, hour)
AS SELECT
    ad_id,
    campaign_id,
    country,
    device_type,
    toStartOfHour(timestamp) AS hour,
    countIf(is_fraud = 0)                       AS valid_clicks,
    countDistinctIf(user_id, is_fraud = 0)      AS unique_users,
    countIf(is_fraud = 1)                       AS fraud_clicks
FROM clicks
GROUP BY ad_id, campaign_id, country, device_type, hour;

-- Günlük rapor: MaterializedView'dan → çok hızlı
SELECT
    ad_id,
    country,
    device_type,
    sum(valid_clicks)   AS valid_clicks,
    sum(unique_users)   AS unique_users,
    sum(fraud_clicks)   AS fraud_clicks
FROM clicks_hourly_mv
WHERE hour >= toStartOfDay(today() - 1)
  AND hour <  toStartOfDay(today())
GROUP BY ad_id, country, device_type;

-- Saatlik trend (advertiser dashboard)
SELECT
    toHour(hour)      AS hour_of_day,
    sum(valid_clicks) AS clicks
FROM clicks_hourly_mv
WHERE ad_id = 'ad-123'
  AND hour >= toStartOfDay(today())
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

---

## CPC Hesabı & Budget Pacing

```
Günlük bütçe tükenmesi:
  Advertiser: $100 bütçe, $0.50 CPC → max 200 geçerli tıklama

Naif yaklaşım: sabah tüm bütçeyi harcama (click farm burst).
Budget pacing: bütçeyi güne yay (her saate $100/24 ≈ $4.17).

Redis ile atomic budget check:
  ad:ad-123:daily_spend    → float, günlük toplam harcama
  ad:ad-123:hourly_budget  → saatlik limit
  ad:ad-123:active         → 0/1

  // Lua script: atomic check + increment
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local cost = tonumber(ARGV[2])
  local current = tonumber(redis.call('GET', key) or "0")
  if current + cost > limit then
      return 0  -- bütçe doldu
  end
  redis.call('INCRBYFLOAT', key, cost)
  return 1  -- geçerli

  Lua atomic → race condition yok (distributed lock gereksiz).

Pacing stratejisi:
  Saatlik hedef: daily_budget / 24
  Her saatin başında: hourly_remaining = hourly_budget
  Saat dolduğunda: kalan → bir sonraki saate taşıma (configurable).

  Throttling: bütçe hızlı tükeniyorsa → tıklama kabul oranını azalt.
  80% bütçe → %100 kabul
  90% bütçe → %80 kabul (random %20 düşür → pacing)
  100% bütçe → kampanya durdur

// Click API'de budget check
boolean canCharge = luaScript.execute(
    "ad:" + adId + ":daily_spend",
    dailyBudget,
    cpcAmount
);
if (!canCharge) {
    // Kampanya bütçesi doldu → redirect yap ama kaydetme
    redis.opsForValue().set("ad:" + adId + ":active", "0", 1, TimeUnit.DAYS);
    res.sendRedirect(redirectUrl);
    return;
}

Bildirimler:
  %80 harcama → email: "Bütçenizin %80'i harcandı"
  %95 harcama → email + Slack: "Bütçe dolmak üzere"
  %100 → kampanya durduruldu bildirimi
  Scheduler (midnight): daily_spend sıfırla → kampanya yeniden aktif
```

---

## Raporlama API

```
GET /api/v1/campaigns/{campaignId}/stats
  ?granularity=hour|day|week|month
  &from=2026-06-01T00:00:00Z
  &to=2026-06-30T23:59:59Z
  &breakdown=country,device_type

Response:
  {
    "campaignId": "camp-456",
    "totalValidClicks": 1482930,
    "totalFraudClicks": 12847,
    "totalUniqueUsers": 947382,
    "totalSpend": 741465.00,
    "data": [
      {
        "timestamp": "2026-06-01T00:00:00Z",
        "country": "TR",
        "device_type": "MOBILE",
        "validClicks": 8472,
        "uniqueUsers": 7103,
        "spend": 4236.00
      },
      ...
    ]
  }

Query routing:
  Son 1 saat → Redis (live counts, <1ms)
  Son 7 gün  → ClickHouse materialized view (<100ms)
  30+ gün    → ClickHouse raw table + FINAL (<2s)

Cache stratejisi:
  Dün raporu: değişmez → 24 saat cache.
  Bu gün raporu: değişiyor → 5 dk cache.
  Canlı (son 1 dk): 10 sn cache.

  Cache key: "report:{campaignId}:{from}:{to}:{granularity}:{breakdown}"
  Cache store: Redis ile response cache.
```

---

## Veri Modeli

```sql
-- Kampanya tanımları (PostgreSQL — OLTP)
CREATE TABLE campaigns (
    campaign_id   UUID PRIMARY KEY,
    advertiser_id UUID NOT NULL,
    name          VARCHAR(200),
    budget_daily  DECIMAL(10,2),
    cpc           DECIMAL(10,4),
    status        VARCHAR(20) CHECK (status IN ('ACTIVE','PAUSED','EXHAUSTED','ENDED')),
    start_date    DATE,
    end_date      DATE,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ads (
    ad_id        UUID PRIMARY KEY,
    campaign_id  UUID REFERENCES campaigns(campaign_id),
    title        VARCHAR(200),
    landing_url  TEXT,
    status       VARCHAR(20)
);

-- Günlük aggregated rapor (PostgreSQL — billing için)
CREATE TABLE daily_click_stats (
    ad_id        UUID,
    stat_date    DATE,
    valid_clicks BIGINT      DEFAULT 0,
    fraud_clicks BIGINT      DEFAULT 0,
    unique_users BIGINT      DEFAULT 0,
    spend        DECIMAL(12,4) DEFAULT 0,
    PRIMARY KEY (ad_id, stat_date)
);

-- Upsert (idempotent):
INSERT INTO daily_click_stats (ad_id, stat_date, valid_clicks, spend)
VALUES ($1, $2, $3, $4)
ON CONFLICT (ad_id, stat_date)
DO UPDATE SET
    valid_clicks = daily_click_stats.valid_clicks + EXCLUDED.valid_clicks,
    spend        = daily_click_stats.spend + EXCLUDED.spend;
```

---

## Multi-Region & Yüksek Erişilebilirlik

```
Senaryo: US, EU, APAC reklamcıları → farklı region'larda servis.

Active-Active: her region kendi Click API cluster'ına sahip.
  Kullanıcı → GeoDNS → en yakın region Click API.
  Her region: kendi Kafka cluster'ı → local işleme.
  Cross-region aggregation: saatlik Kafka MirrorMaker2 veya Flink replikasyonu.
  ClickHouse: global replica (eventual consistency, T+5 dk).

Kafka yüksek erişilebilirlik:
  Replication factor: 3 (3 broker'da kopya)
  min.insync.replicas: 2 (2 broker onaylamadan write başarısız)
  acks=all: producer → tüm in-sync replica onayını bekle

  Click API → Kafka publish başarısız?
    Async → callback'te log (veri kaybı acceptable: <0.01%)
    Retry buffer: local in-memory queue → Kafka tekrar dene
    Kafka tamamen down → local buffer dolana kadar → flush

ClickHouse HA:
  ReplicatedMergeTree: ZooKeeper ile replikasyon.
  2 shard × 2 replica = 4 node.
  Bir shard down → diğeri yanıtlar (degraded mode).

Felaket kurtarma (DR):
  S3: raw click event'ler 1 saatlik batch → Parquet format.
  Kafka down → S3'ten replay → ClickHouse yeniden doldur.
  RPO: 1 saat (son 1 saatlik veri kaybı kabul edilebilir).
  RTO: 4 saat (cluster yeniden başlatma + 3 saatlik replay).
```

---

## Olası Sorunlar ve Çözümleri

### 1. Click API Kafka'ya Yazamadı — Tıklama Kaybı

```
Sorun:
  Click API → Kafka publish → network timeout veya broker down.
  Fire-and-forget: hata fark edilmedi → tıklama kaybedildi → gelir kaybı.
  Büyük kampanyada: 58,000 tıklama/sn × %0.1 kayıp = 58 tıklama/sn kaybı.
  Günde: 5M tıklama → advertiser faturası eksik → anlaşmazlık.

Çözüm:
  a) Local fallback buffer (in-memory):
     KafkaProducer callback → başarısız → ConcurrentLinkedQueue'ya ekle.
     Arka plan thread: her 100ms → kuyruğu Kafka'ya yeniden dene.
     Kuyruk limiti: 10,000 event → dolunca log + alert (Kafka ciddi sorun).

  b) Kafka producer retry konfigürasyonu:
     retries=3, retry.backoff.ms=100
     delivery.timeout.ms=5000 (5 sn max retry süresi)
     acks=1 (leader onayı yeterli — fire-and-forget için denge)

  c) Dead Letter Queue (DLQ):
     3 retry sonrası başarısız → click.dlq topic'ine yaz.
     Ops ekibi: DLQ'yu monitor eder → Kafka sorun çözününce replay.

  d) Metrik + Alert:
     kafka.producer.record-error-rate > 0.001 → PagerDuty.
     Click API latency p99 > 50ms → Kafka yavaşlıyor → alert.
```

---

### 2. Redis Çöktü — Budget Check Durdu, Fraud Tespiti Devre Dışı

```
Sorun:
  Budget check: Redis'te → Redis down → check yapılamıyor.
  Seçenek A: "Redis yoksa tıklamayı reddet" → kampanya durur → gelir kaybı.
  Seçenek B: "Redis yoksa tıklamayı onayla" → bütçe aşılabilir → advertiser zarara uğrar.

  Fraud tespiti: Redis counter'ları → Redis down → tüm tıklamalar geçiyor.
  Bot saldırısı tam bu anda → fraud tespitsiz → sahte tıklamalar sayıldı.

Çözüm:
  a) Redis Sentinel / Cluster (HA):
     3 sentinel + 1 master + 2 replica.
     Master down → sentinel → replica otomatik promote (~30sn).
     Bu süre: degraded mode (aşağıdaki b).

  b) Degraded mode (circuit breaker pattern):
     Redis connection fail → circuit breaker OPEN.
     Budget check: Redis yerine yerel approximate counter (AtomicLong).
     Her Click API instance bağımsız counter → distributed değil ama %100 durma yok.
     Bütçe aşımı riski: node sayısı × CPC kadar (sınırlı aşım, kabul edilebilir).
     Fraud check: Redis yokken conservative threshold uygula (50 tıklama/dk).

  c) Fraud için local Bloom Filter (Redis'ten bağımsız):
     In-process Bloom Filter → Redis önünde → %90 duplicate'ı yakalar.
     Redis sadece kesin dedup için → down olunca local Bloom yeterli.

  d) Budget aşımı sonrası reconciliation:
     Günlük batch job: gerçek harcama vs bütçe karşılaştır.
     Aşım varsa: advertiser hesabına not; politikaya göre iade veya fatura.
```

---

### 3. Kafka Consumer Lag Birikti — Real-Time Dashboard Geriden Geliyor

```
Sorun:
  Black Friday: 58,000 tıklama/sn → Kafka consumer yetişemiyor.
  Consumer lag: 5 dakika → advertiser dashboard "son 1 dk" için 5 dk eski veri.
  Reklam durup durmayacağı belli değil → advertiser panikliyor.

Neden:
  Consumer sayısı < partition sayısı → bazı partition'lar atıl.
  Consumer işleme süresi arttı (Redis yavaşladı, fraud check gecikiyor).
  GC pause: JVM → consumer durdu → lag birikmesi.

Çözüm:
  a) Consumer'ı partition sayısına çıkar:
     64 partition → max 64 consumer (Kafka kısıtı).
     K8s HPA: consumer lag > 10,000 → yeni consumer pod.
     KEDA (Kubernetes Event-Driven Autoscaler): Kafka lag metriği → otomatik scale.

  b) İşleme süresini azalt:
     Redis pipeline: tek tek INCR yerine → batch pipeline (10 command → 1 RTT).
     Fraud check async: click'i queue'ya bırak → ayrı thread havuzu işlesin.
     Batch consume: 100 mesajı birden oku, toplu Redis pipeline.

  c) Backpressure göstergesi:
     Dashboard: lag > 30sn → "veriler gecikebilir" uyarısı göster.
     Kullanıcı: yanıltıcı "0 tıklama" yerine "gecikme var" bilgisi alır.

  d) Tiered consumer:
     Kritik (bütçe/fraud): ayrı consumer group, öncelikli işleme.
     Aggregation: ayrı consumer group, lag toleranslı.
     Kritik consumer lag'ı → 0. Aggregation lag → tolere edilebilir.
```

---

### 4. Hot Partition — Tek Büyük Advertiser Tüm Yükü Bir Partition'a Yığdı

```
Sorun:
  Google, Apple gibi büyük advertiser → tek adId → tek Kafka partition.
  O partition consumer'ı: 58,000/sn'nin %30'u = 17,400 msg/sn → bottleneck.
  Diğer 63 partition: boş bekliyor.
  Consumer lag: sadece hot partition → genel lag spike.

Nasıl tespit edilir:
  Kafka consumer group lag per-partition metriği.
  Partition 7: lag=500,000. Diğerleri: lag<1,000.
  → Hot partition.

Çözüm:
  a) Partition salting (anlık müdahale):
     partition key = adId + "_" + (clickId.hashCode() % 8)
     1 partition → 8 partition'a dağıtım.
     Dezavantaj: aynı adId farklı partition → sıralı window zor.
     Aggregation: 8 partition'dan merge et → final count.

  b) Hot ad tespiti + ayrı topic (proaktif):
     Cron job: önceki gün top 100 reklam → "hot_ads" Redis Set.
     Click API: adId hot_ads'ta mı? → click.hot topic (128 partition).
     Normal ad → click.normal topic (64 partition).
     Hot ad consumer: daha fazla instance → izole scale.

  c) Adaptive partition count:
     Kafka topic partition sayısını artır (64 → 128).
     Not: partition artırma kolay, azaltma zor (immutable mesajlar).
     Yeni partition → mevcut consumer'lar yeniden assign (rebalance) → anlık lag spike.
     Maintenance window'da yap.
```

---

### 5. Duplicate Tıklama Sayıldı — Advertiser Fazla Faturalandı

```
Sorun:
  Kullanıcı yanlışlıkla çift tıkladı → 2 click event Kafka'ya gitti.
  Bloom Filter: %1 FP → gerçek duplicate'ı kaçırdı.
  Her ikisi de click.valid'e geçti → 2 kere faturalandı.
  Advertiser: "Aynı kullanıcı 2 saniyede 2 kere tıklayamaz, bu hata" → şikayet.

Neden kaçtı:
  Bloom Filter: probabilistic, %99 doğru ama %1 FP ve FN riski.
  Flink stateful dedupe: clickId farklıysa → farklı event sayar.
  (Her click ayrı clickId → UUID → Flink ikisini farklı görür)

Çözüm:
  a) Flink'te session bazlı dedup:
     State key: userId + adId (clickId değil).
     5 dakika window içinde aynı userId+adId → duplicate.
     // Flink
     KeyedStream<ClickEvent, String> keyed = clicks
         .keyBy(c -> c.getUserId() + ":" + c.getAdId());
     keyed.process(new SessionDedupeFunction(Duration.ofMinutes(5)));

  b) ClickHouse'da final dedup kontrolü:
     Billing job: daily_click_stats hesaplamadan önce
     countDistinct(userId, adId, toStartOfFiveMinutes(timestamp)) kullan.
     Aynı (userId, adId) 5 dk içinde → 1 tıklama say.

  c) Advertiser anlaşmazlık süreci:
     Advertiser: tıklama itirazı → audit log → clickId zinciri göster.
     "Bu 2 tıklama: farklı session, farklı IP → gerçek 2 tıklama" veya
     "Aynı session, 2 saniye arayla → sistem hatası → iade edildi."
     SLA: itiraz 5 iş günü içinde çözülür.
```

---

### 6. Click Farm — Gerçek İnsan Ama Sahte Tıklama, ML Yakalayamadı

```
Sorun:
  Click farm: gerçek insanlar, gerçek cihazlar, gerçek IP'ler.
  User agent: normal browser. IP: residential. Zaman: rastgele.
  Kural tabanlı fraud: hiçbir kural tetiklenmedi → geçerli sayıldı.
  ML modeli: normal kullanıcıya benziyor → fraud score < 0.3 → geçti.
  Advertiser: tıklamalar var ama conversion yok → ROAS sıfır.

Nasıl tespit edilir:
  Conversion rate (click → satın alma): normal %2-5. Click farm: %0.001.
  Session depth: gerçek kullanıcı 3+ sayfa. Bot: 0-1 sayfa (hemen çıkış).
  Time on page: gerçek: 30-120sn. Click farm: 2-5sn.
  Geographic clustering: Bangladeş'ten 10,000 tıklama → ürün TR'ye özgüyse şüpheli.

Çözüm:
  a) Post-click behaviour analizi (conversion funnel):
     Advertiser pixel: landing page'de JS → session bilgisi gönder.
     Feature: bounce rate, time on page, scroll depth, form interaction.
     ML: bu özellikler → fraud score → gecikmeli invalidation.
     T+24 saat: "Bu tıklamalar fraud" → retroactive billing adjustment.

  b) Advertiser ROAS bazlı anomali:
     ROAS (Return on Ad Spend) < threshold (advertiser tanımlı) → alert.
     "500 tıklama, 0 conversion → click farm şüphesi → manuel inceleme."

  c) Proactive advertiser bildirimi:
     "Kampanyanızda olağandışı tıklama pattern'i tespit ettik → inceleniyor."
     Şeffaflık: advertiser güveni → platform değeri.
```

---

### 7. ClickHouse Sorgusu Yavaşladı — Büyük Tarih Aralığı Sorgusu

```
Sorun:
  Yıllık rapor: SELECT ... WHERE timestamp BETWEEN '2025-01-01' AND '2025-12-31'
  ClickHouse: 365 × 200GB = 73 TB raw data tara → sorgu 3 dakika.
  Report API timeout (30sn) → advertiser "rapor gelmiyor" şikayeti.

Neden:
  Raw table: her şeyi tarar.
  Partition pruning: PARTITION BY toYYYYMM → aylık, ama hâlâ 12 ay taranıyor.
  Materialized View: saatlik — ama 1 yıl için 8,760 satır × 1M ad = çok satır.

Çözüm:
  a) Kademeli pre-aggregation (Tiered MV):
     clicks_hourly_mv   → saatlik (son 7 gün sorgusu)
     clicks_daily_mv    → günlük (son 90 gün sorgusu)
     clicks_monthly_mv  → aylık (90+ gün sorgusu)

     Query router:
     7 gün içi  → clicks_hourly_mv
     90 gün içi → clicks_daily_mv
     90+ gün    → clicks_monthly_mv

  b) Async rapor (uzun sorgular için):
     POST /api/reports/generate → {"status": "pending", "reportId": "r-123"}
     Arka planda: ClickHouse sorgusu → S3'e Parquet.
     GET /api/reports/r-123 → {"status": "ready", "downloadUrl": "..."}
     Kullanıcı: hazır olunca email + download linki.

  c) Sorgu cache (değişmeyen geçmiş):
     Geçen ay raporu → değişmez → Redis'te 7 gün cache.
     Cache hit: <1ms.
     Cache key: "report:{adId}:monthly:2025-06".
```

---

### 8. Gece Yarısı Budget Reset — Race Condition ile Çift Harcama

```
Sorun:
  Gece 00:00:00: scheduler → "daily_spend sıfırla" job çalışır.
  Aynı anda: tıklama geldi → INCRBYFLOAT daily_spend.
  Race condition:
    T=23:59:59.999: spend=$99.80, tıklama → $100.30 (bütçe doldu → kampanya durduruldu)
    T=00:00:00.001: scheduler → daily_spend = 0 (sıfırlandı)
    T=00:00:00.002: tıklama → spend=$0.50 (kampanya aktif sayıldı, ama durdurulmuştu)
    T=00:00:00.003: bir önceki gün "durduruldu" flag'i hâlâ var → tıklama reddedildi.
  Tutarsız durum: spend=0 ama active=0 → advertiser 00:00-00:05 arası kaybetti.

Çözüm:
  a) Atomic reset + activate:
     Lua script: daily_spend sıfırla + active=1 → tek işlemde.
     local keys = KEYS
     redis.call('SET', keys[1], '0', 'EX', '86400')  -- spend
     redis.call('SET', keys[2], '1', 'EX', '86400')  -- active
     return 1

  b) Reset zamanı yayma:
     Tüm kampanyalar gece tam 00:00'da değil → advertiser timezone'a göre.
     TR advertiser: UTC+3 → 21:00 UTC reset.
     US advertiser: UTC-5 → 05:00 UTC reset.
     Yük dağılımı: 1M kampanya → tüm gün reset → spike yok.

  c) Graceful transition:
     23:59:50 - 00:00:10 arası: "transition window."
     Bu pencerede: spend kontrolü → hem eski hem yeni bütçe.
     max(yesterday_remaining, today_budget × 0.1) → geçiş yumuşak.
```

---

## Trade-off Özeti

| Karar | Alternatif A | Alternatif B (Seçilen) | Gerekçe |
|-------|-------------|------------------------|---------|
| Click kayıt | Sync DB write | Kafka fire-and-forget | <10ms latency zorunlu; Kafka async → kullanıcı bloke olmaz |
| Dedup Katman 1 | Redis Set (exact) | Bloom Filter | Memory: 1M entry Set=50MB, Bloom=1.2MB; %1 FP kabul edilebilir |
| Dedup Katman 2 | Kafka idempotent consumer | Flink stateful dedupe | Flink: event time pencereli dedup, daha esnek |
| Fraud | Sadece batch ML | Real-time rules + batch ML | Gerçek zamanlı kural: anlık karar; ML: pattern tespiti — ikisi tamamlayıcı |
| Aggregation store | PostgreSQL OLAP | ClickHouse | ClickHouse: 1B row sorgusunda 100x hızlı, columnar, compressed |
| Real-time count | Polling DB | Redis + Kafka Streams | Redis: sub-ms okuma; Kafka Streams: fraud filtrelenmiş count |
| Budget check | DB transaction | Redis Lua script (atomic) | Lua: atomik, sub-ms; DB: 50ms+ → tıklama latency'sine ekler |
| Hot ad sorunu | Tek partition | Partition salting + ayrı topic | Salting: 8x throughput; ayrı topic: consumer izolasyonu |
| Partition key | Round-robin | adId hash | adId: aynı ad sıralı → windowed aggregation doğru; round-robin: sıra yok |
| Late data | Grace period yok | 2 dk watermark + side output | Mobil/offline click kaybetme → gelir kaybı; side output: audit |
| Raporlama | Tek sorgu katmanı | Redis + ClickHouse MV + raw table | <1 saat: Redis; 1-7 gün: MV (hızlı); 30+ gün: raw (esnek) |
