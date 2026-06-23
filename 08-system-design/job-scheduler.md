# 08o — System Design: Dağıtık Job Scheduler (Cron)

## Gereksinimler

```
Functional:
  ✓ Cron expression ile zamanlanmış job tanımlama
  ✓ Tek seferlik delayed job
  ✓ Job execution (HTTP webhook veya mesaj kuyruğu)
  ✓ Retry on failure
  ✓ Job history ve status takibi
  ✓ Exactly-once execution (aynı job iki kez çalışmasın)

Non-Functional:
  1M tanımlı job
  100,000 job/gün execution
  Zamanlama hassasiyeti: ±1 saniye
  Availability: 99.99% (job'lar kaçırılmamalı)
```

---

## Capacity Estimation

```
100,000 job/gün:
  Ortalama: 1.15 job/sn
  Peak (gece yarısı cron): 10,000 job/dakika = 167/sn (tüm cron'lar aynı anda)

Metadata:
  1M job × 1 KB = 1 GB (job tanımları)
  100,000 execution/gün × 500B = 50 MB/gün (execution log)
  1 yıl log: 18 GB
```

---

## High-Level Tasarım

```
   API Layer
  (CRUD job tanımları)
       │
       ▼
┌─────────────────────────────────────────┐
│         Job Store (PostgreSQL)          │
│  jobs: {id, schedule, type, payload,    │
│          enabled, nextRunAt}            │
└──────────────────┬──────────────────────┘
                   │
       ┌───────────▼────────────┐
       │    Scheduler Service   │
       │  (Leader election ile) │
       │  nextRunAt < now → fire│
       └───────────┬────────────┘
                   │
        ┌──────────▼──────────┐
        │   Execution Queue   │
        │   (Kafka/SQS)       │
        └──────────┬──────────┘
                   │
         ┌─────────┼─────────┐
         ▼         ▼         ▼
     [Executor] [Executor] [Executor]
         │
    HTTP call / Kafka event / Lambda invoke
         │
    Result → Job Store (status, output, duration)
```

---

## Scheduler Service (Leader Election)

```
Problem: Birden fazla Scheduler node varsa — aynı job'ı hepsi tetikler!

Çözüm: Tek leader scheduler

Leader Election:
  ZooKeeper / etcd / Redis ile leader seçimi
  Leader → "scheduler-lock" alır
  Diğerleri standby (leader çöküşünde devreye girer)

Leader görevi:
  Her 1 saniyede:
    SELECT * FROM jobs
    WHERE enabled = true
      AND nextRunAt <= NOW() + INTERVAL '5 seconds'
    ORDER BY nextRunAt ASC
    LIMIT 1000
    FOR UPDATE SKIP LOCKED;  ← başka transaction görmez

  → Her job → Kafka'ya execution message push et
  → nextRunAt güncelle (bir sonraki scheduled time)
  → release lock

SKIP LOCKED (PostgreSQL):
  Aynı anda birden fazla thread çalışırsa → farklı job'ları alır
  Kilitli satırı atlayarak devam eder → deadlock yok
```

---

## Exactly-Once Execution

```
Problem: Scheduler job'ı kuyruğa attı, ama:
  Kuyruğa 2 kez eklendi (network retry)
  Executor 2 kez işledi (at-least-once queue)
  → Job 2 kez çalıştı!

Çözüm: Execution ID + Idempotency

Her job için execution oluşturulurken:
  execution_id = UUID (unique per scheduled time)
  örneğin: {jobId}_{scheduledAt} → hash

DB'de execution kaydı:
  INSERT INTO job_executions (execution_id, job_id, status)
  ON CONFLICT (execution_id) DO NOTHING;
  → Affected rows = 0 → duplicate → skip

Executor tarafında:
  SELECT status FROM job_executions WHERE execution_id = ?
  FOR UPDATE;  ← lock
  
  IF status = 'PENDING':
    status = 'RUNNING'
    → Execute
  ELSE:
    → Skip (already processed)
```

---

## Cron Expression Parser

```
"0 9 * * MON-FRI"  → Her hafta içi sabah 9:00
"*/15 * * * *"      → Her 15 dakikada
"0 0 1 * *"         → Her ayın 1'i gece yarısı
"0 2 * * 0"         → Her Pazar sabah 2:00

Standart kütüphaneler:
  Java: Quartz CronExpression veya cron-utils
  
  // Bir sonraki çalışma zamanı
  CronExpression expr = CronExpression.parse("0 9 * * MON-FRI");
  LocalDateTime nextRun = expr.next(LocalDateTime.now());

nextRunAt hesaplama:
  Job tamamlandı → schedule'a göre nextRunAt hesapla
  nextRunAt = cronExpr.next(lastRunAt)
  UPDATE jobs SET nextRunAt = ? WHERE id = ?
```

---

## Job Tipleri

```java
enum JobType {
    HTTP_WEBHOOK,    // POST belirtilen URL'e
    KAFKA_MESSAGE,   // Belirtilen topic'e mesaj
    LAMBDA,          // AWS Lambda invoke
    SHELL_COMMAND    // Shell komutu çalıştır (tehlikeli!)
}

// HTTP Webhook executor
class HttpJobExecutor {

    ExecutionResult execute(JobExecution execution) {
        Job job = execution.getJob();
        HttpWebhookPayload payload = deserialize(job.getPayload());

        try {
            HttpResponse<String> response = httpClient.post(
                payload.getUrl(),
                job.getPayloadBody(),
                Map.of("Authorization", "Bearer " + payload.getToken())
            );

            if (response.statusCode() >= 200 && response.statusCode() < 300) {
                return ExecutionResult.success(response.body());
            } else {
                return ExecutionResult.failure("HTTP " + response.statusCode());
            }

        } catch (IOException e) {
            return ExecutionResult.failure(e.getMessage());
        }
    }
}
```

---

## Retry Stratejisi

```
Retry policy (job bazında konfigüre edilebilir):
  maxRetries: 3
  backoffType: EXPONENTIAL
  initialDelay: 30s
  maxDelay: 1h

Job execution akışı:
  1. Attempt 1 → FAILED → retry_count=1
  2. 30s sonra → Attempt 2 → FAILED → retry_count=2
  3. 60s sonra → Attempt 3 → FAILED → retry_count=3
  4. maxRetries aşıldı → status=FAILED_PERMANENTLY → alert gönder

Retry için ayrı execution kaydı:
  execution_id = {jobId}_{scheduledAt}_{retryCount}
  → Her retry unique → idempotency korunur
```

---

## Veri Modeli

```sql
CREATE TABLE jobs (
    job_id          UUID PRIMARY KEY,
    name            VARCHAR(200),
    description     TEXT,
    schedule        VARCHAR(100),     -- cron expression veya NULL (tek seferlik)
    scheduled_at    TIMESTAMP,        -- tek seferlik için
    job_type        VARCHAR(50),      -- HTTP_WEBHOOK, KAFKA_MESSAGE
    payload         JSONB,            -- endpoint, topic, body vs.
    enabled         BOOLEAN DEFAULT TRUE,
    next_run_at     TIMESTAMP,
    timezone        VARCHAR(50) DEFAULT 'UTC',
    max_retries     INT DEFAULT 3,
    timeout_seconds INT DEFAULT 30,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_next_run (enabled, next_run_at)  -- Scheduler sorgusu için
);

CREATE TABLE job_executions (
    execution_id    VARCHAR(100) PRIMARY KEY,  -- {jobId}_{scheduledAt}_{retry}
    job_id          UUID NOT NULL,
    scheduled_at    TIMESTAMP,
    started_at      TIMESTAMP,
    finished_at     TIMESTAMP,
    status          VARCHAR(20),   -- PENDING, RUNNING, SUCCESS, FAILED, SKIPPED
    retry_count     INT DEFAULT 0,
    output          TEXT,
    error_message   TEXT,
    duration_ms     BIGINT,
    INDEX idx_job_executions (job_id, scheduled_at DESC)
);
```

---

## Özel Durumlar

```
Uzun süren job (grace period):
  Job timeout = 30s → 30s'de tamamlanmadıysa → TIMEOUT
  Executor → SIGTERM → graceful shutdown
  Timeout sonrası → retry (eğer idempotent)

Timezone desteği:
  "0 9 * * *" → hangi timezone'da sabah 9?
  job.timezone = "Europe/Istanbul"
  nextRunAt hesaplarken timezone'u dönüştür
  UTC'ye çevirerek sakla

Göç (migration) sırasında:
  Scheduler down → bazı job'lar kaçırıldı
  Catch-up execution: nextRunAt < 1 saat önce → çalıştır
  Daha eski → skip (çok geç kaldı)

Distributed lock (Redis) ile:
  Birden fazla scheduler → çakışma
  scheduler:lock → tek leader → Redis TTL 30s → her 10s yenile
  Redis down → leader belirlenemiyor → executorlar çalışmaya devam eder
  (Executor idempotent → job iki kez çalışsa bile OK)
```

---

## Trade-off Özeti

| Karar | Seçenek | Gerekçe |
|-------|---------|---------|
| Leader election | Redis lock | Basit, hızlı |
| Queue | Kafka | Retry, partition, ordering |
| DB | PostgreSQL | SKIP LOCKED, ACID |
| Exactly-once | Idempotency key + DB | Güvenilir |
| Job timeout | HTTP timeout | Basit, konfigüre edilebilir |
