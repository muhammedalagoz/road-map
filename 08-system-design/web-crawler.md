# 08m — System Design: Dağıtık Web Crawler

## Gereksinimler

```
Functional:
  ✓ Seed URL listesinden başla, web'i tara
  ✓ Yinelenen URL'leri atla (deduplication)
  ✓ robots.txt kurallarına uy (politeness)
  ✓ HTML içeriği indir ve sakla
  ✓ Yeni URL'leri keşfet ve kuyruğa ekle
  ✗ İçerik parse / index (out of scope — search engine ayrı)

Non-Functional:
  Hedef: 1 milyar web sayfası/ay
  QPS: 1B / 30gün / 86400 ≈ 400 sayfa/sn
  Storage: Her sayfa ~500KB → 1B × 500KB = 500 TB/ay
  Politeness: Aynı domain'e > 1 req/sn gönderme
```

---

## Capacity Estimation

```
400 sayfa/sn hedef
Her sayfa ortalama: 500 KB HTML + kaynaklar
Net veri: 400 × 500KB = 200 MB/sn = 17 TB/gün

URL deduplication:
  1B URL × 64 bytes (hash) = 64 GB → Bloom filter ile 1 GB'a düşür

Bant genişliği:
  400 req/sn × 500 KB = 200 MB/sn download
  Tipik internet bağlantısı başına: 10 MB/sn
  20 crawler sunucu gerekli
```

---

## High-Level Tasarım

```
Seed URLs
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                  URL Frontier                        │
│  Öncelik Kuyruğu (Priority Queue + Politeness)      │
└──────────────────────┬──────────────────────────────┘
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
        [Worker]   [Worker]   [Worker]   ← 20 crawler sunucu
            │
   ┌────────┴────────┐
   ▼                 ▼
HTML Fetcher    robots.txt Checker
   │
   ▼
Content Parser
(Yeni URL'ler çıkar)
   │
   ▼
URL Deduplication
(Bloom Filter + Redis)
   │ Yeni URL
   ▼
URL Frontier (kuyruğa ekle)
   │ İçerik
   ▼
Content Storage (S3)
   │
   ▼
Downstream (Indexer, ML pipeline)
```

---

## URL Frontier (Öncelik Kuyruğu)

```
İki katmanlı öncelik:

Katman 1 — Öncelik Sıralama:
  Yüksek öncelik:
    - Popüler domain (google.com, wikipedia.org)
    - Yeni keşfedilen URL (fresh content)
    - Belirli kategoriler (haber, e-ticaret)
  Düşük öncelik:
    - Nadir güncellenen sayfa
    - Derin sayfa (link derinliği > 5)

Katman 2 — Politeness (Domain Throttling):
  Her domain için ayrı kuyruk
  Domain başına: max 1 request/saniye

  domain_queue:
    example.com → [url1, url2, url3]
    news.com    → [url4, url5]
    blog.co     → [url6]

  Scheduler:
    Her domain'in son request zamanını kontrol et
    Son req + 1s geçtiyse → sıraki URL'yi al
    Geçmediyse → bekle

Veri yapısı:
  Kafka: domain bazlı partition (aynı domain aynı partition)
  → Her partition'da tek consumer → politeness garantisi
```

---

## URL Deduplication (Bloom Filter)

```
Problem: 1B URL → hepsini DB'de saklamak pahalı
          "Bu URL daha önce ziyaret edildi mi?" → hızlı cevap gerek

Bloom Filter:
  Bit array: M bit (örneğin 1 milyar bit = 128 MB)
  K hash fonksiyonu
  
  URL ekle: K hash hesapla → K bit'i 1 yap
  URL sorgula: K bit'in hepsi 1 mi? → "muhtemelen var" (false positive olabilir)
                Herhangi biri 0 → kesinlikle yok

  False positive rate (FPR):
    1B URL, M=10B bit, K=7 → FPR ≈ 1%
    Kabul edilebilir: %1 URL tekrar ziyaret edilebilir (büyük sorun değil)

  False negative: yok (kesinlikle yeni URL → her zaman ekle)

Implementasyon:
  Java: Guava BloomFilter veya Redis'te bitset
  Distributed: RedisBloom module (multi-node bloom)

Bloom filter dolduğunda:
  Periyodik olarak temizle ve DB'den rebuild (aylık)
  Veya counting bloom filter (silme desteği)
```

---

## HTML Fetcher

```java
class HTMLFetcher {

    private final HttpClient httpClient;
    private final RobotsTxtCache robotsCache;

    FetchResult fetch(String url) {
        // 1. robots.txt kontrolü
        String domain = extractDomain(url);
        RobotsTxt robots = robotsCache.get(domain);
        if (robots.isDisallowed(url, USER_AGENT)) {
            return FetchResult.disallowed(url);
        }

        // 2. Domain throttle kontrolü (son request zamanı)
        throttler.waitIfNeeded(domain);

        // 3. HTTP isteği
        try {
            HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .header("User-Agent", "MyBot/1.0 (+http://mybot.com/bot)")
                    .timeout(Duration.ofSeconds(10))
                    .build(),
                HttpResponse.BodyHandlers.ofString()
            );

            // 4. Redirect takip et
            if (response.statusCode() == 301 || response.statusCode() == 302) {
                String redirectUrl = response.headers().firstValue("Location").orElse(null);
                return redirectUrl != null ? fetch(redirectUrl) : FetchResult.error(url, "No redirect location");
            }

            if (response.statusCode() != 200) {
                return FetchResult.error(url, "HTTP " + response.statusCode());
            }

            return FetchResult.success(url, response.body(), response.headers());

        } catch (IOException | InterruptedException e) {
            return FetchResult.error(url, e.getMessage());
        }
    }
}
```

---

## Content Parser (URL Çıkarma)

```java
class ContentParser {

    List<String> extractUrls(String html, String baseUrl) {
        Document doc = Jsoup.parse(html, baseUrl);

        return doc.select("a[href]").stream()
            .map(el -> el.absUrl("href"))  // relative → absolute
            .filter(url -> isValid(url))
            .filter(url -> isSameDomain(url, baseUrl) || isAllowedDomain(url))
            .distinct()
            .toList();
    }

    boolean isValid(String url) {
        if (url == null || url.isEmpty()) return false;
        if (!url.startsWith("http")) return false;
        if (url.contains("?") && url.length() > 2000) return false;  // çok uzun URL
        if (BLACKLISTED_EXTENSIONS.stream().anyMatch(url::endsWith)) return false;
        // .jpg, .pdf, .mp4, .zip → skip
        return true;
    }
}
```

---

## robots.txt Uyumu

```
robots.txt örneği:
  User-agent: *
  Disallow: /admin/
  Disallow: /private/
  Crawl-delay: 2

  User-agent: MyBot
  Allow: /public/
  Disallow: /

Implementasyon:
  Her domain için robots.txt cache'le (1 gün TTL)
  İlk istekten önce: GET https://example.com/robots.txt
  Parse → "Disallowed paths" listesi

Sitemap:
  robots.txt'de: Sitemap: https://example.com/sitemap.xml
  Sitemap → tüm URL'lerin listesi → Frontier'e ekle (yüksek öncelik)
```

---

## Storage

```
İçerik depolama: S3
  Key: s3://crawl-data/{year}/{month}/{day}/{url_hash}.html.gz
  Sıkıştırılmış: ~500KB → ~50KB (gzip)
  500TB → 50TB etkili

Metadata: Cassandra
  (url_hash, url, fetched_at, status_code, content_type, outlinks_count)

Crawl state: Redis
  visited_bloom: bloom filter
  domain_last_fetched: {domain → timestamp}
  priority_queue: sorted set (score=priority)
```

---

## Dağıtık Koordinasyon

```
20 worker sunucu — nasıl koordine edilir?

Kafka partitioned by domain:
  domain hash → partition → tek worker
  Aynı domain'in URL'leri hep aynı worker → politeness garantisi
  Worker çöküşünde → Kafka rebalance → başka worker devam eder

Consistent hashing:
  Worker ekleme/çıkarma → minimum URL redistribution

Monitoring:
  Crawl rate (sayfa/sn) → hedef 400
  Error rate (5xx, timeout)
  Bloom filter false positive rate
  URL queue depth (büyüyor mu → worker sayısı artır)
```

---

## Trade-off Özeti

| Karar | Seçenek | Gerekçe |
|-------|---------|---------|
| Dedup | HashSet | Bloom Filter | Bloom (128MB vs 64GB) |
| Kuyruk | RabbitMQ | Kafka | Kafka (partitioned by domain) |
| Politeness | Sleep | Throttler per domain | Per-domain (paralel domain) |
| Storage | DB | S3 | S3 (petabyte ölçeği) |
