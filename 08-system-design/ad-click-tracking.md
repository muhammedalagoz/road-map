# 08p — System Design: Reklam Tıklama Takip Sistemi

## Gereksinimler

```
Functional:
  ✓ Reklam tıklamalarını kayıt altına al
  ✓ Gerçek zamanlı tıklama sayısı (son 1 dk, 1 saat)
  ✓ Günlük/aylık aggregated rapor
  ✓ Tıklama başına ödeme (CPC) hesabı
  ✓ Click fraud tespiti
  ✗ Reklam serving, targeting (out of scope)

Non-Functional:
  1B tıklama/gün
  Latency: tıklama kaydı < 10ms (kullanıcıyı bloke etmemeli)
  Aggregation sorgu: < 1s
  Exactly-once counting (fraud ve duplicate önleme)
  Hot advertiser (büyük kampanya) → spike trafiği
```

---

## Capacity Estimation

```
Tıklama QPS:
  1B / 86400 ≈ 11,600/sn ortalama
  Peak: 11,600 × 5 = 58,000/sn (büyük kampanya, Black Friday)

Storage:
  Raw click: {clickId, adId, userId, timestamp, ip, userAgent} = 200 bytes
  1B × 200B = 200 GB/gün
  90 gün: 18 TB

Aggregated storage (much smaller):
  1B click/gün, 1M ad → avg 1000 click/ad/gün
  Aggregated: {adId, date, hour, count, uniqueUsers} = 100 bytes
  1M ad × 24 hour × 100B = 2.4 GB/gün (1000x küçük!)

Bandwidth:
  58,000 × 200B = 11.6 MB/sn inbound
```

---

## High-Level Tasarım

```
User clicks ad
      │
      ▼
  ┌──────────────────────────────────┐
  │   Click API (ultra-low latency)  │
  │   "Fire and forget"              │
  │   → 302 redirect to destination  │
  └──────────────┬───────────────────┘
                 │ async
                 ▼
          ┌─────────────┐
          │    Kafka    │  ← click.raw topic
          └──────┬──────┘
                 │
    ┌────────────┼────────────────┐
    ▼            ▼                ▼
[Fraud      [Real-time        [Batch
 Detector]   Aggregator]       Writer]
    │            │                │
    ▼            ▼                ▼
[Filter     [Redis/ClickHouse] [ClickHouse
 invalid]    (live counts)     (OLAP reports)]
```

---

## Click API (Ultra-Low Latency)

```
Kullanıcı tıkladı → 10ms'den fazla beklememeli

Akış:
  GET /click?adId=ad-123&redirectUrl=https://advertiser.com/product

  Click API:
  1. clickId = UUID oluştur (5μs)
  2. Kafka'ya async publish (non-blocking, <1ms)
  3. 302 redirect → kullanıcı gidiyor, işlem arka planda
  
  Kritik: Kafka'ya publish başarısız olsa bile redirect yap
  → "fire and forget" → kullanıcı deneyimi > veri kaybı riski
  → Kafka güvenilirliği yüksek (%99.99)

Tracking pixel alternatifi:
  <img src="/track?adId=ad-123" width="1" height="1">
  → Sayfa yüklenince tetiklenir → GET isteği → log
```

```java
@GetMapping("/click")
void trackClick(@RequestParam String adId,
                @RequestParam String redirectUrl,
                HttpServletRequest req,
                HttpServletResponse res) throws IOException {

    // Async Kafka publish (non-blocking)
    kafkaTemplate.send("click.raw",
        adId,  // partition key = adId (aynı ad aynı partition)
        ClickEvent.builder()
            .clickId(UUID.randomUUID().toString())
            .adId(adId)
            .userId(extractUserId(req))
            .ip(getClientIp(req))
            .userAgent(req.getHeader("User-Agent"))
            .timestamp(Instant.now())
            .build()
    );  // fire-and-forget

    // Hemen redirect (Kafka bekleme)
    res.sendRedirect(redirectUrl);
}
```

---

## Click Fraud Tespiti

```
Fraud türleri:
  1. Same user multiple clicks (hızlı ardışık tıklama)
  2. Bot traffic (user agent analizi, headless browser)
  3. IP-based fraud (data center IP, proxy, VPN)
  4. Click farms (düşük kaliteli tıklama)

Gerçek zamanlı fraud detection (Kafka Streams):

Rule 1: Aynı kullanıcı, aynı ad, 5 dk içinde 3+ tıklama → fraud
  Redis: INCR user:{userId}:ad:{adId}:clicks EX 300
  count > 3 → bu tıklamayı invalid işaretle

Rule 2: Aynı IP'den 1 dakikada 100+ tıklama → bot
  Redis: INCR ip:{ip}:clicks:minute:{minute} EX 60
  count > 100 → IP blacklist

Rule 3: User Agent kontrolü
  "python-requests", "curl", "bot" içeriyorsa → skip
  Headless browser signatures (Puppeteer, Playwright) → şüpheli

Rule 4: IP reputation
  3rd party IP database (IPQualityScore, MaxMind)
  Data center IP, Tor exit node, known proxy → block

ML tabanlı fraud (offline):
  Click pattern → feature vector → model → fraud score
  Günlük model güncelleme → false positive → manuel inceleme
```

---

## Real-Time Aggregation (Flink/Kafka Streams)

```java
// Kafka Streams ile 1 dakikalık tıklama sayısı
StreamsBuilder builder = new StreamsBuilder();

builder.stream("click.raw", Consumed.with(Serdes.String(), clickSerde))
    .filter((key, click) -> !click.isFraud())  // fraud filtrele
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count(Materialized.as("click-counts-per-minute"))
    .toStream()
    .foreach((windowedKey, count) -> {
        // Redis'e yaz: INCR ad:{adId}:clicks:minute:{minute}
        redis.opsForValue().set(
            "ad:" + windowedKey.key() + ":minute:" + windowedKey.window().start(),
            count,
            Duration.ofMinutes(5)
        );
    });

// Sorgu: Son 1 dakika tıklama
Long count = redis.opsForValue().get("ad:ad-123:minute:" + currentMinute);
```

---

## Batch Aggregation (ClickHouse)

```sql
-- ClickHouse: OLAP, column-store, aggregation odaklı
-- 1B row/sn insert destekler, sorgular çok hızlı

CREATE TABLE clicks (
    click_id    String,
    ad_id       String,
    user_id     String,
    campaign_id String,
    country     String,
    device_type String,
    timestamp   DateTime,
    is_fraud    UInt8 DEFAULT 0
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (ad_id, timestamp);

-- Günlük rapor (saniyeler içinde)
SELECT
    ad_id,
    country,
    device_type,
    countIf(is_fraud = 0)        AS valid_clicks,
    countDistinctIf(user_id, is_fraud = 0) AS unique_users,
    count()                       AS total_clicks
FROM clicks
WHERE toDate(timestamp) = today() - 1
GROUP BY ad_id, country, device_type;

-- Saatlik trend
SELECT
    ad_id,
    toHour(timestamp) AS hour,
    count()           AS clicks
FROM clicks
WHERE ad_id = 'ad-123' AND toDate(timestamp) = today()
GROUP BY ad_id, hour
ORDER BY hour;
```

---

## CPC (Cost Per Click) Hesabı

```
Günlük bütçe tükenmesi:
  Advertiser: $100 bütçe, $0.50 CPC
  → Max 200 geçerli tıklama

Real-time budget check:
  Redis: ad:ad-123:daily_spend (float)
  INCRBYFLOAT ad:ad-123:daily_spend 0.50
  value > 100 → kampanyayı durdur

Durdurma:
  Redis: SET ad:ad-123:active 0
  Click API'de kontrol:
    active = redis.get("ad:ad-123:active")
    if not active → redirect ama kaydetme

Günlük sıfırlama:
  Gece yarısı scheduler → SETEX ad:{adId}:daily_spend 86400 0

Bütçe azalma bildirimi:
  %80 harcandı → email alert → advertiser
  %100 → kampanya otomatik durdurulur
```

---

## Veri Modeli

```sql
-- Kampanya tanımları (PostgreSQL)
CREATE TABLE campaigns (
    campaign_id  UUID PRIMARY KEY,
    advertiser_id UUID,
    name         VARCHAR(200),
    budget_daily DECIMAL(10,2),
    cpc          DECIMAL(10,4),  -- cost per click
    status       VARCHAR(20),    -- ACTIVE, PAUSED, EXHAUSTED
    start_date   DATE,
    end_date     DATE
);

-- Reklam tanımları
CREATE TABLE ads (
    ad_id        UUID PRIMARY KEY,
    campaign_id  UUID,
    title        VARCHAR(200),
    landing_url  TEXT,
    status       VARCHAR(20)
);

-- Günlük aggregated rapor
CREATE TABLE daily_click_stats (
    ad_id           UUID,
    stat_date       DATE,
    valid_clicks    BIGINT,
    fraud_clicks    BIGINT,
    unique_users    BIGINT,
    spend           DECIMAL(12,4),
    PRIMARY KEY (ad_id, stat_date)
);
```

---

## Trade-off Özeti

| Karar | Seçenek | Gerekçe |
|-------|---------|---------|
| Click kayıt | Sync DB | Kafka fire-and-forget | Kafka (10ms < latency) |
| Fraud | Batch | Real-time rules + batch ML | Hybrid |
| Aggregation | SQL | ClickHouse | ClickHouse (OLAP, 1B row hızlı) |
| Budget | DB | Redis INCR | Redis (atomic, sub-ms) |
| Partition | Round-robin | adId key | adId (aynı ad aynı partition) |
