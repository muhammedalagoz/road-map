# 08n — System Design: Metrics & Monitoring Sistemi (Prometheus/Datadog)

## Gereksinimler

```
Functional:
  ✓ Metrik toplama (CPU, memory, custom metrics)
  ✓ Time-series depolama
  ✓ Sorgulama ve aggregation
  ✓ Alert tanımlama ve bildirim
  ✓ Dashboard (Grafana entegrasyonu)
  ✗ Log yönetimi, tracing (out of scope)

Non-Functional:
  1000 servis, her biri 100 metrik → 100,000 metrik
  Her metrik 15s scrape → 6,667 data point/sn
  Retention: 15 gün full, 1 yıl aggregated
  Sorgu latency: < 1s
```

---

## Capacity Estimation

```
Data points:
  100,000 metrik × (1/15s) = 6,667 data point/sn
  6,667 × 86400 = 576M data point/gün

Storage:
  Her data point: {timestamp(8B), value(8B), labels(~50B)} ≈ 70 bytes
  576M × 70B = 40 GB/gün (sıkıştırma öncesi)
  Sıkıştırma (delta encoding + XOR): %90 tasarruf → 4 GB/gün
  15 gün full: 60 GB
  1 yıl aggregated (1m resolüsyon): 576M × 4 × 70B / 15 = ~10 GB

Query:
  QPS: 100 (dashboard refresh'ler)
  Her query: 1M data point'i scan edebilir → aggregation engine kritik
```

---

## High-Level Tasarım

```
  Services / Infrastructure
  (CPU, memory, HTTP rate, custom metrics)
       │ /metrics HTTP endpoint (Prometheus format)
       │ veya push (StatsD/Telegraf)
       ▼
┌──────────────────────────────────┐
│        Collection Layer          │
│   Prometheus Scraper             │
│   (pull model, 15s interval)     │
│   veya Push Gateway              │
└──────────────┬───────────────────┘
               │
        ┌──────▼──────┐
        │   Kafka     │ ← yüksek throughput için buffer
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
[Writer]   [Writer]   [Writer]  ← paralel yazma
    │
    ▼
┌─────────────────────────────┐
│  Time-Series DB             │
│  (Prometheus TSDB /         │
│   VictoriaMetrics /         │
│   InfluxDB / Thanos)        │
└──────────────┬──────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
[Query API] [Alert Manager] [Grafana]
```

---

## Veri Toplama: Pull vs Push

```
Pull Model (Prometheus):
  Prometheus → her 15s → servisler /metrics endpoint'ini scrape eder
  
  Avantaj:
    ✓ Servis "durumu" anlaşılır (pull başarısız → servis down)
    ✓ Servisler birbirinden bağımsız
    ✓ Scrape interval servis başına ayarlanabilir
  
  Dezavantaj:
    ✗ Firewall arkasındaki servisler scrape edilemez
    ✗ Short-lived jobs (batch, cron) → çalışıp bitebilir
    Push Gateway → short-lived jobs buraya push eder → Prometheus pull

Push Model (StatsD, Telegraf):
  Servis → metrik değeri → koleksiyon ajanına gönderir
  
  Avantaj:
    ✓ Firewall sorunu yok
    ✓ Short-lived job desteği
  
  Dezavantaj:
    ✗ Servis down → hâlâ "push" yok → anlamak zor
    ✗ Agent konfigürasyonu her serviste
```

---

## Time-Series Veri Yapısı

```
Prometheus formatı:
  # TYPE http_requests_total counter
  http_requests_total{method="POST",status="200",service="order-service"} 1234 1705312800000
  http_requests_total{method="GET",status="404",service="order-service"} 56 1705312800000

Data model:
  metric_name + labels (key:value map) = unique time series
  
  http_requests_total{method="POST",status="200"} → time series 1
  http_requests_total{method="GET",status="404"}  → time series 2

Yüksek kardinalite sorunu:
  http_requests_total{userId="user-12345"} → 1M kullanıcı = 1M time series
  → Cardinality explosion → memory/disk tükenir
  Çözüm: userId gibi high-cardinality label KULLANMA
```

---

## TSDB Depolama (Time-Series Database)

### Prometheus TSDB İç Yapısı

```
Chunk-based storage:
  Her time series → chunks (2 saatlik bloklar)
  Chunk: delta-encoded timestamps + XOR-encoded float values
  
  Timestamp encoding (delta of deltas):
  1705312800, 1705312815, 1705312830, 1705312845
  → delta: 15, 15, 15 → delta of delta: 0, 0 → neredeyse sıfır bit

  Value encoding (XOR floating point):
  CPU: 0.67, 0.68, 0.67, 0.69
  → XOR: çok az bit değişiyor → %90 sıkıştırma

  2 saatlik block:
    {metric_name, labels} → head (WAL + in-memory)
    Dolunca → disk'e yaz (immutable block)
    Compaction: küçük bloklar → büyük bloklar (1 gün, 1 hafta)
```

### Thanos / VictoriaMetrics (Ölçekleme)

```
Prometheus sınırı:
  Tek node → ~10M active time series
  Data retention sınırlı (RAM/disk)

Thanos:
  Her Prometheus node → Thanos Sidecar
  Sidecar → blokları S3'e upload (uzun retention)
  Thanos Querier → birden fazla Prometheus'u federe eder
  Thanos Compactor → S3'te blokları compact eder

VictoriaMetrics:
  Prometheus drop-in replacement
  Daha iyi sıkıştırma, daha az RAM
  Cluster mode: storage + query ayrı ölçeklenir
```

---

## Sorgulama (PromQL)

```promql
# Son 5 dakikanın ortalama CPU kullanımı
avg(rate(process_cpu_seconds_total[5m])) by (service)

# P99 HTTP response time
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Error rate (son 5 dk)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# 1 saatten fazla müsait olmayan servis
up{job="order-service"} == 0 for 1h
```

---

## Alert Manager

```yaml
# Alert tanımı
groups:
  - name: service_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 2m          # 2 dk sürekli aşarsa
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} error rate > 5%"
          description: "Current: {{ $value | humanizePercentage }}"

      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.90
        for: 5m
        labels:
          severity: warning
```

```
Alert routing:
  Critical → PagerDuty (acil nöbet)
  Warning → Slack #alerts kanalı
  Business metric → Email

Alert fatigue önleme:
  Grouping: 5 dakika içinde aynı service'ten gelen alertleri grupla
  Silencing: Deploy sırasında 30 dk sustur
  Inhibition: Düğüm down → o düğümdeki service alertlerini bastır
```

---

## Downsampling (Veri Sıkıştırma)

```
Retention stratejisi (storage tasarrufu):
  
  0-48 saat:   15s resolüsyon (raw data)
  2-30 gün:    1m resolüsyon  (5m avg/min/max)
  30-90 gün:   5m resolüsyon
  90-365 gün:  1h resolüsyon
  1 yıl+:      1d resolüsyon

Downsampling nasıl:
  Thanos Compactor / VictoriaMetrics:
  Raw: 15s → 1m aggregate (avg, min, max, sum, count)
  1m → 5m aggregate
  → Raw silinir, aggregate saklanır

Storage tasarrufu:
  40 GB/gün × 15 gün = 600 GB
  vs aggregated 1 yıl = 10 GB
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Toplama | Push (StatsD) | Pull (Prometheus) | Pull (health check dahil) |
| TSDB | InfluxDB | Prometheus TSDB | Prometheus (ekosistem) |
| Ölçekleme | Federation | Thanos | Thanos (S3 unlimited retention) |
| Alert | Custom | Alert Manager | Alert Manager (PromQL native) |
| Dashboard | Custom | Grafana | Grafana (hazır panel) |
