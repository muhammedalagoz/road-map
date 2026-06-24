# 08n — System Design: Metrics & Monitoring Sistemi (Prometheus/Datadog)

## Gereksinimler

```
Functional:
  ✓ Metrik toplama (CPU, memory, custom business metrics)
  ✓ Time-series depolama (ham veri + downsampled)
  ✓ Sorgulama ve aggregation (PromQL)
  ✓ Alert tanımlama, değerlendirme, bildirim
  ✓ Dashboard (Grafana)
  ✓ SLO/SLI takibi ve error budget
  ✓ Servis keşfi (Kubernetes SD, Consul SD)
  ✓ Multi-cluster / multi-region aggregation
  ✗ Log yönetimi (out of scope → ELK/Loki)
  ✗ Distributed tracing (out of scope → Jaeger/Tempo)

Non-Functional:
  1,000 servis × 100 metrik = 100,000 aktif time series
  Her metrik 15s scrape → 6,667 data point/sn
  Retention: 15 gün full, 1 yıl downsampled
  Sorgu latency: < 1s (dashboard), < 10s (heavy range query)
  Alert evaluation: her 15-30sn
  Availability: 99.9% (monitoring sistemi down → kör uçuş)
```

---

## Capacity Estimation

```
Data points:
  100,000 metrik × (1/15s) = 6,667 data point/sn
  6,667 × 86,400 = 576M data point/gün

Storage:
  Ham: {timestamp(8B) + value(8B) + labels(~50B)} ≈ 70 byte/point
  576M × 70B = 40 GB/gün sıkıştırma öncesi
  Delta + XOR sıkıştırma: %90 tasarruf → 4 GB/gün
  15 gün tam: 60 GB
  1 yıl downsampled (1m resolüsyon): ~10 GB

Query yükü:
  100 dashboard × 20 panel × her 30sn yenile = 66 QPS
  Her sorgu: 1-10M data point scan → TSDB aggregation engine kritik

Bant genişliği:
  100,000 metrik × 70B × her 15sn = ~450 KB/scrape cycle
  1,000 servis → paralel scrape → 450 KB / 15sn = 30 KB/sn → trivial

Ölçek faktörü:
  10,000 servis, 1,000 metrik/servis = 10M time series → cluster gerekli
  Prometheus tek node: ~10M active series sınırı → bu noktada Thanos/VM Cluster
```

---

## Observability'nin Üç Sütunu

```
Metrics (bu dosya):
  "Sistem şu anda nasıl?" → sayısal, zaman serisi, aggregation.
  CPU %80, error rate %2, P99 latency 450ms.
  Güçlü: eğilim, karşılaştırma, alert, dashboard.
  Zayıf: "Neden yavaş?" sorusunu cevaplayamaz.

Logs:
  "Ne oldu?" → olay bazlı, yapılandırılmış metin.
  ERROR: payment failed, orderId=123, reason="card declined".
  Güçlü: root cause analysis, debug.
  Zayıf: yüksek hacim → depolama pahalı; arama yavaş.
  Araçlar: ELK Stack (Elasticsearch + Logstash + Kibana), Grafana Loki, Fluentd.

Traces:
  "Nerede vakit geçiyor?" → istek boyunca latency dağılımı.
  request-123: API GW 2ms → Auth 5ms → Order Svc 180ms → DB 160ms.
  Güçlü: darboğaz tespiti, servisler arası bağımlılık.
  Zayıf: her istek için overhead, sampling gerekli.
  Araçlar: Jaeger, Zipkin, Grafana Tempo, AWS X-Ray.

Correlation (bağlantı):
  Trace ID: her log satırında → trace ile eşleştir.
  Exemplar: Prometheus histogram → yavaş request'in trace ID'si.
  Dashboard: "P99 spike" → exemplar'a tıkla → ilgili trace → ilgili log.
  Bu bağlantı olmadan: üç araç ayrı ada → "hangisi doğru?"
```

---

## High-Level Mimari

```
Services / Infrastructure
(CPU, memory, HTTP rate, custom business metrics)
     │  /metrics HTTP endpoint (OpenMetrics / Prometheus format)
     │  veya OTLP (OpenTelemetry Protocol)
     ▼
┌──────────────────────────────────────────────────┐
│              Collection Layer                     │
│  Prometheus (pull, 15s) + OTEL Collector (push)  │
│  Service Discovery: Kubernetes SD, Consul SD      │
│  Push Gateway: short-lived job'lar için           │
└─────────────────┬────────────────────────────────┘
                  │ high volume data points
                  ▼
           ┌──────────────┐
           │    Kafka     │  ← tampon (burst absorb, fan-out)
           └──────┬───────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
  [Writer]    [Writer]   [Writer]   ← paralel, sharded
       │
       ▼
┌──────────────────────────────────────────────────┐
│          Time-Series DB Cluster                   │
│  Prometheus TSDB (lokal, kısa retention)          │
│  + Thanos / VictoriaMetrics (uzun retention, S3)  │
└─────────────────┬────────────────────────────────┘
                  │
    ┌─────────────┼──────────────┐
    ▼             ▼              ▼
[Query API]  [Alert Manager]  [Grafana]
(PromQL)     (PagerDuty/      (Dashboard,
             Slack/Email)      SLO panel)
                  │
                  ▼
         [Runbook Automation]
         (on-call handbook, auto-remediation)
```

---

## Veri Toplama: Pull vs Push

```
Pull Model (Prometheus):
  Prometheus → her 15sn → servislerin /metrics endpoint'ini scrape eder.

  Avantaj:
    ✓ Servis durumu anlaşılır: scrape başarısız → servis down.
    ✓ Servisler birbirinden bağımsız: sadece endpoint açık olsun.
    ✓ Scrape interval servis başına ayarlanabilir.
    ✓ Replay: eski veri tekrar scrape edilebilir (yoksa Kafka).

  Dezavantaj:
    ✗ Firewall arkasındaki servisler scrape edilemez.
    ✗ Short-lived job (batch, Lambda): tamamlanıp gidebilir → push gateway.
    ✗ Çok sayıda servis → Prometheus'un tüm servisleri bilmesi gerekiyor.

Push Model (StatsD, OpenTelemetry Collector, Telegraf):
  Servis → metriği koleksiyon ajanına gönderir.

  Avantaj:
    ✓ Firewall sorunu yok.
    ✓ Short-lived job desteği.
    ✓ İnce istemci: sadece UDP ile gönder, ajan halleder.

  Dezavantaj:
    ✗ Servis down → hâlâ push görünmüyor → tespit zor.
    ✗ Her servis ajan konfigürasyonu gerektirir.

Hibrit (Gerçek Dünya):
  Kubernetes servisleri → Prometheus pull (otomatik SD).
  AWS Lambda, Batch job → Push Gateway veya OTEL Collector.
  External servis (3rd party API) → Blackbox Exporter (probe-based pull).
```

---

## Servis Keşfi (Service Discovery)

```
Problem: 1,000 servis → Prometheus'a her birini elle eklemek imkansız.
Çözüm: Prometheus SD → dinamik servis listesi.

Kubernetes Service Discovery:
  Prometheus: Kubernetes API'yi izler.
  Yeni pod: annotation "prometheus.io/scrape: true" → otomatik eklenir.
  Pod silinince: otomatik çıkar.
  
  Pod annotation'ları:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"

  scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: "true"
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: (.+)
          replacement: ${1}:${2}

Consul Service Discovery:
  Servis → Consul'a register.
  Prometheus: Consul → servis listesini çeker.
  Health check: Consul'un kendi check'i → unhealthy → Prometheus'tan çıkar.

Blackbox Exporter (dış probe):
  Prometheus → Blackbox → hedef URL'ye HTTP/ICMP probe.
  "Bu URL cevap veriyor mu?" → availability monitoring.
  Kullanım: 3rd party API, customer-facing endpoint.

  probe_http:
    targets:
      - https://api.myservice.com/health
      - https://cdn.myservice.com/status
```

---

## Metrik Tipleri

```
Counter (sayaç):
  Sadece artır, hiç azalmaz (reset dışında).
  Kullanım: toplam istek sayısı, hata sayısı, byte sayısı.
  http_requests_total{method="POST", status="200"} = 12345
  
  Sorgulama: rate() → "son 5 dk'da kaç/sn?"
  rate(http_requests_total[5m]) → 45 req/sn

Gauge (gösterge):
  Artabilir veya azalabilir (anlık değer).
  Kullanım: CPU kullanımı, memory, aktif bağlantı sayısı, kuyruk derinliği.
  node_memory_MemAvailable_bytes = 4294967296
  
  Sorgulama: doğrudan kullan.
  node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes → %72 boş

Histogram (dağılım):
  Değerleri önceden tanımlı bucket'lara say.
  Kullanım: istek süresi, yanıt boyutu.
  
  http_request_duration_seconds_bucket{le="0.1"}  = 9000  → 0-100ms arası 9000 istek
  http_request_duration_seconds_bucket{le="0.5"}  = 9800  → 0-500ms arası 9800 istek
  http_request_duration_seconds_bucket{le="1.0"}  = 9950
  http_request_duration_seconds_bucket{le="+Inf"} = 10000 → toplam istek
  http_request_duration_seconds_sum = 1234.5      → toplam süre
  http_request_duration_seconds_count = 10000     → istek sayısı
  
  P99 sorgusu:
  histogram_quantile(0.99,
    rate(http_request_duration_seconds_bucket[5m]))
  
  Bucket seçimi: tahmin edilen değerlerin etrafında:
  [0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0] → tipik HTTP latency.

Summary (özet):
  Client-side quantile hesaplama.
  http_request_duration_seconds{quantile="0.99"} = 0.43
  Avantaj: sunucu tarafı hesaplama yok.
  Dezavantaj: quantile'lar aggregation yapılamaz (farklı instance'lardan toplama imkansız).
  Öneri: Histogram kullan (aggregable), Summary kaçın.

Naming convention:
  {namespace}_{subsystem}_{name}_{unit}
  Örnek: http_server_request_duration_seconds
         db_connection_pool_active_connections_total
         kafka_consumer_lag_messages
  Unit daima suffix'te: _seconds, _bytes, _total (counter için).
```

---

## USE / RED / Golden Signals Metodolojileri

```
USE Method (Brendan Gregg — Infrastructure):
  U - Utilization:  Kaynak ne kadar kullanılıyor? (CPU %70, disk %80)
  S - Saturation:   Kapasite aşıldı mı? (queue depth, run queue)
  E - Errors:       Hata var mı? (disk I/O error, packet drop)
  
  Her kaynak için (CPU, memory, disk, network, GPU):
    CPU: utilization = cpu_percent, saturation = load_avg, errors = cpu_errors
    Memory: utilization = mem_used%, saturation = swap_usage, errors = oom_kills
    Disk: utilization = iops%, saturation = io_wait, errors = disk_errors

RED Method (Tom Wilkie — Microservices):
  R - Rate:     Kaç istek/sn geliyor?
  E - Errors:   Hata oranı nedir? (500, timeout)
  D - Duration: İstek ne kadar sürüyor? (P99, P50)
  
  Her servis için:
    rate(http_requests_total[5m])
    rate(http_requests_total{status=~"5.."}[5m])
    histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  
  Öneri: Her microservice için standart RED dashboard.

Google's Four Golden Signals (SRE Book):
  1. Latency:     Başarılı ve başarısız isteklerin süresi (ayrı izle!)
  2. Traffic:     Sistem ne kadar yük görüyor? (QPS, B/sn)
  3. Errors:      Başarısız istek oranı (HTTP 500, timeout, explicit error)
  4. Saturation:  Sistemin "ne kadar dolu?" (CPU queue, memory pressure)
  
  Kural: Bu 4 sinyal alert edilirse → olayların %99'u yakalanır.

Hangi metodoloji ne zaman:
  Infrastructure monitoring → USE.
  API/Microservice monitoring → RED.
  Üst seviye servis → Golden Signals.
  Pratikte: üçü birlikte kullan → farklı katmanları kapsar.
```

---

## TSDB Depolama

### Prometheus TSDB İç Yapısı

```
Chunk-based storage:
  Her time series → 2 saatlik chunk blokları.
  Chunk: delta-encoded timestamp + XOR float values.
  
  Timestamp delta of deltas (çok verimli):
    t0=1705312800, t1=1705312815, t2=1705312830
    delta: 15, 15 → delta of delta: 0 → ~0 bit
  
  XOR float encoding (Gorilla):
    CPU: 0.672, 0.673, 0.671 → XOR: sadece değişen bit'ler.
    Monoton değerler: neredeyse sıfır bit.
  
  Sonuç: 70 byte/point → ~1.7 byte/point sıkıştırma sonrası.

Block lifecycle:
  0-2 saat: WAL (Write-Ahead Log) + MemTable (in-memory).
  2 saat: disk'e immutable block olarak flush.
  Compaction: küçük bloklar → büyük bloklar (2h → 6h → 12h → 24h).
  Avantaj: compaction sırasında eski blok tamamen silinir → tombstone yok.

Index yapısı:
  Label → posting list (bu label'ı içeren time series ID'leri).
  Sorgu: status="200" AND service="order" → iki posting list intersection → O(N).
  TSDB: bitmap index (roaring bitmap) → set intersection çok hızlı.
```

### Thanos / VictoriaMetrics

```
Prometheus sınırları:
  Tek node: ~10M aktif time series, RAM kritik.
  Retention: disk dolunca → eski veri silinir.
  HA: tek node SPOF.

Thanos mimarisi:
  Prometheus (kısa, lokal) + Thanos Sidecar → her Prometheus yanında.
  Sidecar: her 2h block → S3'e upload.
  Thanos Querier: birden fazla Prometheus + S3'ten global view.
  Thanos Compactor: S3'te blokları merge + downsampling (1m, 5m, 1h).
  Thanos Ruler: uzun retention AlertManager.
  
  Sorgu akışı:
    Grafana → Thanos Querier → Prometheus'lar (son 2h) + S3 (eski)
    Deduplicate: 2 Prometheus aynı veriyi topladıysa → bir tane kullan.

VictoriaMetrics (VM):
  Prometheus drop-in replacement: aynı PromQL, aynı format.
  Daha iyi: %3-4 disk kullanımı (Prometheus %8-10), daha az RAM.
  vminsert + vmstorage + vmselect → bağımsız ölçeklenebilir.
  vmselect: okuma scale; vminsert: yazma scale; vmstorage: storage scale.
  
  Tercih:
    Küçük kurulum → Prometheus + Thanos.
    Büyük kurulum, maliyet kritik → VictoriaMetrics.
```

---

## PromQL Sorgulama

```promql
# Temel sorgular

# Son 5 dk ortalama CPU (servis başına)
avg(rate(process_cpu_seconds_total[5m])) by (service)

# P50/P90/P99 HTTP latency
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# Error rate (5xx / total)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service) * 100

# Memory kullanım yüzdesi
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Kafka consumer lag (maksimum topic başına)
max(kafka_consumer_group_lag) by (topic, consumer_group)

# DB bağlantı pool doluluk oranı
db_pool_active_connections / db_pool_max_connections * 100

# JVM GC pause zamanı (son 5 dk toplam)
rate(jvm_gc_pause_seconds_sum[5m])

# 30 gün öncesiyle karşılaştırma (WoW — Week over Week)
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 7d)

# Anlık throughput baseline altında mı?
rate(http_requests_total[5m]) < 0.5 * avg_over_time(rate(http_requests_total[5m])[1h:5m] offset 1d)
```

---

## Alert Tasarımı

### Alert Kuralları

```yaml
groups:
  - name: slo_alerts
    rules:
      # --- RED Metodolojisi ---
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) > 0.05
        for: 2m
        labels:
          severity: critical
          team: "{{ $labels.service }}"
        annotations:
          summary: "{{ $labels.service }}: Error rate %{{ $value | humanizePercentage }}"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
          description: "Son 2 dk boyunca error rate %5 üzerinde."

      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }}: P99 latency {{ $value | humanizeDuration }}"
          runbook: "https://wiki.company.com/runbooks/high-latency"

      # --- USE Metodolojisi ---
      - alert: HighCPUSaturation
        expr: |
          100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (node) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.node }}: CPU %{{ $value | humanize }}"

      - alert: OOMKillDetected
        expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
        for: 0m   # anında alert (OOM = kritik)
        labels:
          severity: critical
        annotations:
          summary: "OOM Kill: {{ $labels.namespace }}/{{ $labels.pod }}"

      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_group_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka lag: {{ $labels.topic }} → {{ $value }} mesaj geride"
```

### Alert Routing ve Fatigue Önleme

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'service']    # aynı service'ten alertleri grupla
  group_wait: 30s                        # grup oluşmadan bekle
  group_interval: 5m                     # aynı gruba yeni ekleme aralığı
  repeat_interval: 4h                    # tekrar bildirim
  receiver: 'slack-warnings'

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
      continue: true   # aynı zamanda slack'e de gönder

    - match:
        severity: warning
      receiver: 'slack-warnings'

    - match:
        alertname: OOMKillDetected
      receiver: 'pagerduty-oncall'
      repeat_interval: 30m  # kritik → daha sık hatırlatma

receivers:
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - routing_key: 'xxx'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/xxx'
        channel: '#alerts'
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

inhibit_rules:
  # Node down → o node'daki tüm servis alert'leri bastır
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      service: '.+'
    equal: ['node']

  # Cluster down → bireysel servis alert'leri bastır
  - source_match:
      alertname: 'ClusterDown'
    target_match:
      severity: warning
```

```
Alert Fatigue (Uyarı Yorgunluğu) Önleme:

  Sorun: 500 alert/gün → ekip alertleri görmezden gelir → gerçek kriz kaçırılır.
  
  İyi alert özellikleri:
    ✓ Actionable: on-call'ın yapabileceği bir şey var.
    ✓ Urgent: hemen müdahale edilmeli (yoksa SLA ihlali).
    ✓ Novel: zaten bilinen/beklenen durum değil.
    ✗ Kötü: "CPU %70" → zaten beklenen → gereksiz.

  Flapping önleme:
    for: 2m → 2 dk boyunca sürekli ihlal → sonra alert.
    Tek anlık spike: 1 data point → alert değil.

  Silencing:
    Deploy sırasında: 30 dk sustur → bilinen reboot/restart alertleri gelmez.
    Bakım penceresi: Grafana'da "silence" oluştur.

  Alert sayısı hedefi:
    Critical: < 5/gün (PagerDuty uyanmaları).
    Warning: < 30/gün (Slack mesajları).
    Daha fazlası → alert seti gözden geçir, thresholdları yükselt.
```

---

## SLO / SLI / Error Budget

```
Tanımlar:
  SLA (Service Level Agreement): müşteriye verilen taahhüt (ihlal → tazminat).
    Örn: "Servisimiz aylık %99.9 uptime sağlar."
  
  SLO (Service Level Objective): iç hedef (SLA'dan biraz sıkı).
    Örn: "Dahili hedef: %99.5 uptime" (SLA %99.9'dan önce alarm).
  
  SLI (Service Level Indicator): gerçek ölçüm.
    Availability SLI: başarılı istek / toplam istek.
    Latency SLI: P99 < 500ms olan istekler / toplam.

Error Budget:
  SLO %99.9 uptime → aylık 0.1% hata payı = 43.8 dk downtime.
  Error budget bitti → yeni feature deploy değil, reliability'ye odaklan.

  Error budget tüketimi:
    consumed = (1 - current_availability) / (1 - SLO)
    %99.0 SLI, %99.9 SLO → consumed = 0.01 / 0.001 = 10 → %1000 tüketildi!
  
  PromQL:
    # 30 günlük availability
    1 - (
      sum(rate(http_requests_total{status=~"5.."}[30d]))
      /
      sum(rate(http_requests_total[30d]))
    )
    
    # Error budget kalan (%)
    (
      (1 - 0.999) - (1 - sli_30d)
    ) / (1 - 0.999) * 100

Burn Rate Alert:
  "Error budget'ı bu hızda tüketiyorsak 1 saat içinde biter!"
  
  - alert: ErrorBudgetBurnRate
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[1h]))
        / sum(rate(http_requests_total[1h]))
      ) > (1 - 0.999) * 14.4   # 1 saatte 1 aylık budget'ın 1/2'si
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Error budget çok hızlı tükeniyor (14.4x burn rate)"
  
  Burn rate çarpanları:
    1x: normal tüketim (SLO tam sınırında).
    14.4x: 1 saatte aylık budget'ın 1/12'si → kritik.
    6x: 6 saatte budget'ın 1/4'ü → uyarı.
```

---

## Downsampling ve Retention

```
Retention katmanları:
  0-2 saat:    RAM (MemTable, WAL) → en hızlı sorgu.
  0-48 saat:   15s raw → lokal Prometheus disk.
  2-30 gün:    1m aggregated (avg, min, max, sum, count).
  30-90 gün:   5m aggregated → S3 (Thanos).
  90-365 gün:  1h aggregated → S3 Glacier.
  1+ yıl:      1d aggregated → long-term trend (kapasite planlaması).

Downsampling nasıl çalışır:
  Thanos Compactor / VictoriaMetrics:
  Raw (15s) → 1m: her dakika için 4 raw point → {avg, min, max, count} 4 float.
  Sıkıştırılmış 1m: ~10x küçük (4 point → 4 float, farklı sıkıştırma).

Storage tasarrufu örneği:
  Raw (15s, 15 gün): 4 GB/gün × 15 = 60 GB.
  1m (30-90 gün):    ~400 MB/gün × 60 = 24 GB.
  5m (90-365 gün):   ~80 MB/gün × 275 = 22 GB.
  1h (1+ yıl):       ~14 MB/gün × 365 = 5 GB.
  Toplam 1 yıl: ~111 GB (raw sadece 15 gün olmasaydı 1.4 TB).

Sorgu routing:
  Son 2 gün → lokal Prometheus (hızlı).
  Son 30 gün → Thanos + 1m aggregated.
  Son 1 yıl → S3 + 1h aggregated.
  Grafana: zaman aralığı → otomatik doğru kaynak.
```

---

## Kapasite Planlaması

```
Metriklerle gelecek tahmini:

  Trend analizi:
    disk_usage_bytes → predict_linear(disk_usage_bytes[7d], 30*24*3600)
    "Mevcut trend ile diskimiz 30 gün sonra dolacak."
    
    predict_linear(node_filesystem_avail_bytes[6h], 4 * 3600) < 0
    → "4 saat içinde disk dolacak" → alarm!

  Büyüme modeli:
    Kullanıcı büyümesi: %20/ay → istek %20/ay → kaynak %20/ay.
    Şu an: 100 QPS, 10 pod → 6 ay sonra: 300 QPS → 30 pod.
    Maliyet tahmini: 30 pod × $0.10/saat = $2,160/ay.

  Yük testi ile doğrulama:
    k6 / Gatling: senaryo → kapasiteyi bul.
    Metrikler: QPS arttıkça P99 nasıl değişiyor? Kırılma noktası nerede?

  Anomali tespiti:
    "Bu servis normalde 100 QPS, şu an 500 QPS → anomali!"
    Basit: fixed threshold.
    İleri: ML (Prophet, SARIMA) → mevsimsellik + trend → daha akıllı.
    Datadog: ML-based anomaly detection (Watchdog).

Saturation öncesi alarm:
  CPU: %80 olunca alarm (saturation = %100'e varmadan).
  Memory: %85 olunca alarm.
  Kafka partition: lag > X → consumer ekleme zamanı.
  DB bağlantı: %75 → havuz büyütme zamanı.
  
  "Alarm geldiğinde hâlâ vaktin var" → proaktif yönetim.
```

---

## OpenTelemetry Enstrümantasyonu

```
Neden OpenTelemetry:
  Her araç (Prometheus, Datadog, Jaeger) için ayrı enstrümantasyon → vendor lock-in.
  OTEL: standart API → tek kodla Prometheus, Datadog, Jaeger'e gönder.
  Traces + Metrics + Logs → tek SDK ile.

Java Spring Boot örneği:
  implementation 'io.micrometer:micrometer-registry-prometheus'
  // Spring Actuator otomatik /actuator/prometheus endpoint oluşturur.
  
  Custom metric:
  @Autowired MeterRegistry registry;
  
  Counter orderCreated = Counter.builder("orders.created")
    .description("Toplam oluşturulan sipariş")
    .tag("payment_method", "credit_card")
    .register(registry);
  
  orderCreated.increment();
  
  // Timer (histogram)
  Timer timer = Timer.builder("db.query.duration")
    .description("DB sorgu süresi")
    .tag("query_type", "select")
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(registry);
  
  timer.record(() -> dbRepository.findById(id));

OTEL Collector mimarisi:
  Servis → OTEL SDK → OTEL Collector (exporter).
  Collector: Prometheus format → Prometheus / VictoriaMetrics.
  Collector: OTLP → Jaeger (traces).
  Collector: JSON → Loki (logs).
  
  Avantaj: servis sadece OTEL ile konuşur → backend değişse bile kod değişmez.

Exemplar (Metrics → Traces köprüsü):
  Histogram: yavaş bir request → trace_id ekle.
  http_request_duration_seconds_bucket{le="1.0"} = 100 # {traceID="abc123"} 0.987
  Grafana: P99 spike → exemplar → trace'e git → root cause bul.
```

---

## Prometheus HA (Yüksek Erişilebilirlik)

```
Problem: Tek Prometheus → SPOF → down olduğunda kör kalıyoruz.

HA Pattern 1: Dual Prometheus (aktif-aktif):
  İki Prometheus aynı servisleri scrape eder.
  Her ikisi de aynı alert kurallarını değerlendirir.
  Alert Manager: her ikisinden de aynı alert gelir → deduplicate eder.
  Avantaj: biri down → diğeri çalışmaya devam.
  Dezavantaj: veri iki kez yazılıyor → 2x storage.

HA Pattern 2: Thanos HA:
  Her Prometheus → Thanos Sidecar.
  Thanos Querier: her iki Prometheus'tan sorgula → deduplicate.
  Replica label: prometheus_replica="A" ve "B" → Querier birini seçer.
  Avantaj: global view, long-term storage, HA birlikte.

Recording Rules (sorgu öncesi compute):
  Pahalı sorgu: her dashboard refresh → tekrar hesaplama → DB yük.
  Recording rule: her 1m önceden hesapla → TSDB'e yaz.
  
  groups:
    - name: precomputed
      interval: 1m
      rules:
        - record: job:http_requests_total:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)
        - record: job:http_error_rate:ratio5m
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
            / sum(rate(http_requests_total[5m])) by (job)
  
  Dashboard: job:http_requests_total:rate5m → çok hızlı (precomputed).

Federation (büyük kurulum):
  Lokal Prometheus: servis metriklerini scrape.
  Global Prometheus: lokal'lerden sadece kritik metrikleri çeker.
  Multi-cluster: her cluster'da lokal → merkezi global.
  Thanos: federation yerine daha modern çözüm.
```

---

## Veri Modeli

```sql
-- Monitoring config tabloları (TSDB değil, metadata)

CREATE TABLE alert_rules (
    rule_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name         VARCHAR(200) NOT NULL,
    expr         TEXT NOT NULL,          -- PromQL expression
    for_duration VARCHAR(20),            -- "2m", "5m"
    severity     VARCHAR(20),            -- critical, warning, info
    labels       JSONB,
    annotations  JSONB,                  -- summary, description, runbook
    team         VARCHAR(100),
    enabled      BOOLEAN DEFAULT TRUE,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE slo_definitions (
    slo_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service        VARCHAR(100) NOT NULL,
    slo_name       VARCHAR(200),
    sli_expr       TEXT NOT NULL,        -- SLI hesaplayan PromQL
    target         DECIMAL(5,4),         -- 0.999 = %99.9
    window_days    INT DEFAULT 30,
    error_budget_pct DECIMAL(5,2),       -- başlangıçta %100
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE error_budget_events (
    event_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slo_id       UUID REFERENCES slo_definitions(slo_id),
    event_type   VARCHAR(20),            -- BUDGET_DEPLETED, BUDGET_RESTORED
    budget_pct   DECIMAL(5,2),
    occurred_at  TIMESTAMPTZ DEFAULT NOW(),
    incident_url TEXT
);

CREATE TABLE on_call_schedules (
    team         VARCHAR(100),
    user_id      UUID NOT NULL,
    start_at     TIMESTAMPTZ NOT NULL,
    end_at       TIMESTAMPTZ NOT NULL,
    timezone     VARCHAR(50) DEFAULT 'UTC'
);
```

---

## Olası Sorunlar ve Çözümleri

### 1. Cardinality Explosion — Prometheus OOM Oldu

```
Sorun:
  Yeni developer: HTTP latency metriğine userId label ekledi.
  http_request_duration_seconds{userId="user-12345", ...}
  1M kullanıcı → 1M time series bu metrik için.
  100 bin → 1M time series → Prometheus RAM: 10 GB → 100 GB → OOM kill.
  Prometheus pod çöktü → monitoring tamamen durdu → kör kaldık.

Neden kritik:
  Prometheus: her aktif time series için ~5KB RAM.
  1M series × 5KB = 5 GB sadece bu metrik için.
  + 99 başka metrik → OOM.

Çözüm:
  a) High-cardinality label'ları yasakla:
     CI/CD: metric değişikliği → otomatik cardinality check.
     "userId, sessionId, requestId, traceId" → label olarak YASAK.
     Bu değerler: log'a yaz veya exemplar olarak traces'e gönder.

  b) Cardinality limit (Prometheus):
     --storage.tsdb.max-block-duration → memory limit.
     VictoriaMetrics: -maxLabelsPerTimeseries=30 → limit.
     Prometheus 2.45+: per-rule cardinality limit.

  c) Recording rule ile aggregation:
     userId'yi label'dan çıkar → servis başına aggregate:
     http_request_duration_seconds → histogram_quantile → servis bazında.
     Detay: userId → ClickHouse (yüksek kardinalite için uygun).

  d) Cardinality monitoring:
     prometheus_tsdb_head_series → aktif series sayısı → alarm:
     alert: prometheus_tsdb_head_series > 5_000_000 → "Cardinality yüksek!"
     Günlük rapor: en çok series tüketen metrikler → geliştirici bildirimi.
```

---

### 2. Alert Fatigue — Ekip Alert'leri Görmezden Geliyor

```
Sorun:
  Monitoring sistemi: 200 alert/gün üretiyor.
  PagerDuty: 50 gece uyanışı/ay → 5 ingeniör → 10/kişi/ay.
  Ekip: "Zaten yanlış alarm oluyor" → alert'i ses kısıp uyuyor.
  Gerçek kriz: büyük outage → alert geldi → kimse bakmıyor → 4 saat sonra fark edildi.

Çözüm:
  a) Alert audit (haftalık):
     Son 30 gün: her alert → "actionable miydi?" değerlendir.
     Actionable değilse: kaldır veya threshold'u yükselt.
     Hedef: < 5 critical alert/gün, < 20 warning/gün.

  b) Severity seviyeleri (katı):
     Critical (PagerDuty, uyan): SLO ihlali, veri kaybı, complete outage.
     Warning (Slack, iş saatinde bak): degraded performance, kapasiteye yaklaşma.
     Info (dashboard): bilgilendirme, takip et, harekete geçme.

  c) Runbook zorunluluğu:
     Alert olmadan runbook → CI/CD reddeder.
     Runbook: "Bu alert için adım adım ne yap?"
     On-call: alert → runbook → 5 dk içinde aksiyon alabilir.

  d) Flapping önleme:
     Alert: for: 2m → 2 dk boyunca sürekli ihlal → tetiklensin.
     Tek spike: for süresi geçmeden düzeldi → alert yok.
     Resolve cooldown: alert çözüldü → 15 dk → tekrar aynı alert gelirse → yeni incident.

  e) Alert mesajları kalitesi:
     Kötü: "CPU yüksek" → ne yapacaksın?
     İyi: "order-service CPU %90 (10dk), P99 latency 2sn'ye çıktı.
           Muhtemel sebep: DB yavaşlaması. Kontrol: /runbooks/high-cpu"
```

---

### 3. Prometheus Disk Doldu — Incident Sırasında Metrik Kaybı

```
Sorun:
  Büyük incident: ağır load → her servis normalden 10x metrik üretiyor.
  Prometheus disk: normalde 4 GB/gün → incident'ta 40 GB/gün.
  15 günlük retention × 40 GB = 600 GB → disk doldu.
  Prometheus: yeni veri yazamıyor → en eski blokları sil → eski metrikler kayboluyor.
  Post-mortem: "Incident öncesi ne durumundaydık?" → veri yok.

Çözüm:
  a) Disk doluluk alarmı:
     disk_free_bytes / disk_total_bytes < 0.20 → "Disk %80 dolu!"
     Önceden müdahale et → incident zamanı değil.

  b) Remote write (sürekli S3/VictoriaMetrics'e):
     Prometheus → her yazma → aynı zamanda VictoriaMetrics'e remote write.
     Prometheus disk dolsa bile → VictoriaMetrics'te tam kopyası var.
     Sorgu: Thanos Querier → hem lokal hem VictoriaMetrics.

  c) Otomatik downsampling tetikle:
     Disk %70 → compaction'ı hızlandır → eski blokları 1m'e downsample → sil.
     1h'lık data: raw 15s (400MB) → 1m aggregated (20MB) → 20x kazanç.

  d) Incident için kalıcı storage:
     Tüm raw data: remote write → S3 (Thanos).
     Prometheus local: sadece son 48 saat (fast query için buffer).
     Post-mortem: S3'ten tüm geçmiş sorgulanabilir.
```

---

### 4. Scrape Timeout — Yavaş Servis, Metrikler Eksik

```
Sorun:
  payment-service: anlık yük → /metrics endpoint'i 30sn sürede cevap verdi.
  Prometheus scrape timeout: 10sn → timeout → bu scrape interval metrikler eksik.
  Alert: "payment-service down görünüyor" → aslında sadece yavaş.
  On-call: servis down değil, uğraşıyor → yanlış müdahale.

Çözüm:
  a) Scrape timeout servise göre ayarla:
     payment-service: scrape_timeout: 30s (ağır /metrics için).
     Normal servis: 10s.
     Tradeoff: uzun timeout → Prometheus scrape döngüsünü geciktirebilir.

  b) /metrics endpoint'i hafiflet:
     Problem: /metrics hesaplanırken heavy computation → yavaş.
     Çözüm: background thread → her 15sn precompute → /metrics cache'ten serve.
     /metrics: < 100ms her zaman (scrape timeout sorunu ortadan kalkar).

  c) up metric ile ayır:
     up{job="payment-service"} = 0 → timeout veya bağlantı hatası.
     up = 1 ama metrikler eksik → partial scrape.
     Alert: up == 0 for 2m → gerçekten down.
     Uyarı: kısa timeout → up = 0 ama aslında yavaş.

  d) Scrape başarı monitoring:
     scrape_duration_seconds → scrape ne kadar sürdü.
     scrape_samples_scraped → kaç sample geldi (eksikse azaldı).
     Alert: avg(scrape_duration_seconds) > 8s → "Scrape yavaş, timeout yakın."
```

---

### 5. False Positive Alert — Deployment Sırasında Alarm Yağdı

```
Sorun:
  Yeni versiyon deployment: rolling update → eski pod'lar ölüyor, yeni pod'lar başlıyor.
  30 sn: bazı pod'lar yeniden başlıyor → error rate geçici %10 → 2 dk threshold → alarm.
  PagerDuty: 03:00'da on-call uyandırıldı.
  Araştırma: 20 dk → "Ah, normal deployment'tı."
  Sonuç: gereksiz uyanış → moral bozukluğu → alert yorgunluğu.

Çözüm:
  a) Deployment sırasında silence:
     CI/CD: deployment başladı → Alert Manager'a 30 dk silence ekle.
     Kubernetes: annotation "deployment.in-progress: true" → Prometheus silence.
     Otomatik: yeni pod başladı event → Alertmanager API → silence.

  b) Deployment alert'i ayrı yönet:
     Normal alert: SLO ihlali → PagerDuty.
     Deployment alert: error spike < 5 dk → Slack'e log, PagerDuty değil.
     Deployment validation: yeni versiyon error rate > 5% → 5 dk sonra → rollback.

  c) for: süresini artır:
     HighErrorRate: for: 5m (2m yerine).
     Rolling update: 2 dk içinde biter → 5 dk threshold → alarm yok.
     Gerçek kriz: 5 dk boyunca devam eder → alarm verir.

  d) Canary deployment + alert:
     %5 trafik yeni versiyona → 10 dk izle → sorun varsa alert → rollback.
     Tüm trafik geçmeden alarm → çok daha küçük etki.
```

---

### 6. Yüksek Kardinalite Sorgusu — Grafana Dashboard 30sn Dondu

```
Sorun:
  Grafana panel: "Tüm kullanıcıların P99 latency'si, user bazında."
  PromQL: histogram_quantile(0.99, sum(rate(...[5m])) by (le, userId))
  1M userId × 100 bucket = 100M time series scan → 30sn sürdü.
  Dashboard: "Loading..." → kullanıcı yeniledi → daha da yük → Prometheus OOM riski.

Çözüm:
  a) Query'yi değiştir (service bazında, user değil):
     histogram_quantile(0.99, sum(rate(...[5m])) by (le, service))
     Service sayısı: 50 → 50 × 100 bucket = 5,000 series → anında.
     Kullanıcı bazlı P99: ClickHouse'a sor (yüksek kardinalite için tasarlandı).

  b) Recording rule:
     Her 1m: service bazında P99 precompute → TSDB'e kaydet.
     Dashboard: precomputed metric → < 1sn.
     Pahalı sorgu: sadece recording rule zamanında (1m başına 1 kez).

  c) Grafana query caching:
     Sonuç: 30sn'de hesaplandı → 1m cache → sonraki 60sn aynı sonuç.
     Stale: kabul edilebilir (dashboard zaten 1m yenileniyor).
     Prometheus: query_cache → built-in (son sorguyu cache).

  d) Step (resolüsyon) artır:
     Dashboard 7 gün aralık: step=15s → 7 × 86400 / 15 = 40,320 point.
     step=5m → 7 × 288 = 2,016 point → 20x daha az veri → hızlı.
     Grafana: Min Step → "auto" yerine "5m" → uzun aralıkta.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Toplama modeli | Push (StatsD) | Pull (Prometheus) | Pull: health check dahil, servis bağımsız |
| TSDB | InfluxDB | Prometheus TSDB + Thanos | Prometheus: ekosistem, PromQL, Grafana enteg. |
| Uzun retention | Prometheus lokal | S3 + Thanos Compactor | Lokal: disk dolar; S3: sınırsız, ucuz |
| HA | Tek Prometheus | Dual + Thanos Querier dedupe | Tek: SPOF; Dual: kesintisiz alert eval. |
| Yüksek ölçek | Prometheus Federation | VictoriaMetrics Cluster | Federation: karmaşık; VM: daha verimli |
| Alert kanal | Tek kanal | Severity bazlı routing | Tek: fatigue; Routing: doğru kişiye doğru öncelik |
| Enstrümantasyon | Vendor SDK | OpenTelemetry | Vendor: lock-in; OTEL: standart, taşınabilir |
| Quantile | Summary | Histogram | Summary: aggregation yok; Histogram: cross-instance P99 |
| Cardinality kontrol | Runtime | CI/CD gate | Runtime: geç tespit; CI/CD: önceden önle |
| Sorgu performansı | Raw query | Recording Rules | Raw: her refresh hesapla; Record: precomputed |
