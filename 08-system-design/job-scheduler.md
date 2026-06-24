# 08o — System Design: Dağıtık Job Scheduler (Cron / Temporal)

## Gereksinimler

```
Functional:
  ✓ Cron expression ile zamanlanmış job tanımlama
  ✓ Tek seferlik delayed job (fire-and-forget)
  ✓ Job execution (HTTP webhook, Kafka event, Lambda invoke)
  ✓ Retry on failure (exponential backoff, dead letter)
  ✓ Job history ve status takibi
  ✓ Exactly-once execution (aynı job iki kez çalışmasın)
  ✓ Job önceliklendirme (HIGH / MEDIUM / LOW)
  ✓ Job bağımlılıkları (job B, job A bittikten sonra çalışsın)
  ✓ Timezone desteği
  ✓ Job iptal / duraklatma / yeniden başlatma
  ✓ Uzun süren job'lar için heartbeat + cancel

Non-Functional:
  1M tanımlı job
  100,000 job/gün execution
  Zamanlama hassasiyeti: ±1 saniye
  Availability: 99.99% (job'lar kaçırılmamalı)
  Missed job recovery: scheduler down → toparlandıktan sonra yakala
  Multi-tenant: farklı takımların job'ları izole
```

---

## Capacity Estimation

```
Job execution:
  100,000 job/gün ortalama → 1.15 job/sn
  Gece yarısı spike: tüm "0 0 * * *" cron'lar → 10,000 job/dakika = 167/sn
  Her dakika spike: "*/1 * * * *" → 1,000 job/dakika olabilir

Scheduler polling:
  Her sn: SELECT önümüzdeki 10sn'deki job'lar → 1M job × index tarama.
  LIMIT 1000 → verimli (index üzerinden).

Executor QPS:
  167 job/sn × ortalama HTTP çağrısı 500ms = 84 eş zamanlı HTTP bağlantısı.
  Executor pod: 50 eş zamanlı iş → 2 pod peak'te yeterli, 5 pod güvenli.

Storage:
  Job tanımları: 1M × 2 KB = 2 GB (PostgreSQL)
  Execution log: 100,000/gün × 1 KB = 100 MB/gün → 36 GB/yıl
  Partitioning: execution log → aylık partition → eski log S3'e arşiv

Kafka (execution queue):
  167 msg/sn peak → trivial (Kafka: milyonlar/sn kapasiteli)
  Partition sayısı: 10 (executor pod başına 2 partition)
  Retention: 7 gün (retry + replay için)
```

---

## High-Level Mimari

```
         Client / API (REST)
         CRUD: job oluştur, güncelle, iptal et, geçmiş gör
                     │
             ┌───────▼────────┐
             │   API Service  │  ← auth, validation, rate limit
             └───────┬────────┘
                     │
         ┌───────────▼──────────────┐
         │    Job Store (PostgreSQL) │
         │  jobs: tanım, schedule,   │
         │        nextRunAt, priority│
         └───────────┬──────────────┘
                     │
         ┌───────────▼──────────────┐
         │   Scheduler Service      │
         │  (Partitioned Leader)    │  ← birden fazla node, farklı partition
         │  nextRunAt < now → fire  │
         └───────────┬──────────────┘
                     │
              ┌──────▼──────┐
              │ Kafka Queue  │  ← priority topic'leri: high, medium, low
              └──────┬───────┘
                     │
         ┌───────────┼──────────────┐
         ▼           ▼              ▼
    [Executor]  [Executor]    [Executor]
    HTTP call / Kafka event / Lambda
         │
    Result → job_executions (status, output, duration)
         │
    Kafka: JobCompleted → Dependency Resolver → downstream job tetikle

Yan servisler:
  Missed Job Detector:   scheduler gecikmelerini tespit et
  SLA Monitor:           zamanında çalışmayan job → alert
  Dead Letter Handler:   maxRetry aşan job → DLQ → insan müdahalesi
  Tenant Rate Limiter:   tenant başına execution limiti
```

---

## Scheduler Service: Partition Bazlı Ölçekleme

```
Tek Leader problemi:
  1M job, 167/sn peak → tek node → CPU darboğazı.
  Leader çöküşünde: Sentinel/ZK yeni leader seçer (~30sn) → boşluk.

Partition-based scheduler (daha iyi):
  Job'lar N partition'a atanır: partition_id = hash(job_id) % N
  Her scheduler node: belirli partition'ları yönetir.
  N=4: Node-0 → {0,1}, Node-1 → {2,3}, Node-2 → standby...

  Avantaj:
    Yatay ölçeklenebilir: node ekle → partition yeniden dağıt.
    Tek node çöküşü → sadece o partition'lar etkilenir → diğerleri çalışır.
    Tek leader: tüm node'lar scheduler (lider değil, her biri kendi partition'ından sorumlu).

  Partition atama (ZooKeeper / etcd):
    Node'lar register olur → ZK partition'ları atar.
    Node çöküşü → ZK event → diğer node partition'ı devralır.
    Rebalance: yeni node → partition'lar yeniden dağıtılır.

  PostgreSQL: her node kendi partition'ını sorgular:
    SELECT * FROM jobs
    WHERE partition_id IN (0, 1)   -- bu node'un partition'ları
      AND enabled = true
      AND next_run_at <= NOW() + INTERVAL '5 seconds'
    ORDER BY priority DESC, next_run_at ASC
    LIMIT 500
    FOR UPDATE SKIP LOCKED;
```

### Klasik Leader Election (Basit Kurulum İçin)

```java
// Redis ile leader election
@Scheduled(fixedDelay = 10_000) // her 10 sn yenile
void renewLeaderLock() {
    String nodeId = System.getenv("POD_NAME");
    // SETNX: varsa set etme, yoksa set et → atomic
    Boolean isLeader = redis.opsForValue()
        .setIfAbsent("scheduler:leader", nodeId, Duration.ofSeconds(30));

    if (Boolean.TRUE.equals(isLeader)) {
        this.isLeader = true;
    } else {
        // Mevcut leader hâlâ bu node mı?
        String current = redis.opsForValue().get("scheduler:leader");
        this.isLeader = nodeId.equals(current);
    }
}

@Scheduled(fixedRate = 1_000) // her sn
void schedulerLoop() {
    if (!isLeader) return; // Lider değilsem hiçbir şey yapma

    List<Job> dueJobs = jobRepository.findDueJobs(Instant.now().plusSeconds(5));
    for (Job job : dueJobs) {
        enqueueExecution(job);
        updateNextRunAt(job);
    }
}
```

---

## Exactly-Once Execution

```
Problem: Scheduler job'ı kuyruğa attı, ama:
  Network retry → Kafka'ya 2 kez eklendi.
  Executor at-least-once → 2 kez işledi.
  → Job 2 kez çalıştı!

Katman 1 — Idempotency key (DB):
  execution_id = SHA-256(jobId + scheduledAt.truncate(1min))
  -- aynı job için aynı dakikada her zaman aynı ID

  INSERT INTO job_executions (execution_id, job_id, status)
  VALUES (:execId, :jobId, 'PENDING')
  ON CONFLICT (execution_id) DO NOTHING;
  -- affected_rows = 0 → duplicate → Kafka'ya gönderme

Katman 2 — Optimistic state transition (Executor):
  UPDATE job_executions
  SET status = 'RUNNING', started_at = NOW()
  WHERE execution_id = :execId AND status = 'PENDING';
  -- affected_rows = 0 → zaten başka executor aldı → skip

  Bu sayede iki executor aynı job'ı aldıysa → biri kazanır, diğeri skip.

Katman 3 — Kafka consumer group idempotency:
  Kafka: consumer group → her partition tek consumer.
  Partition key = execution_id → aynı execution hep aynı consumer.
  Consumer crash → tekrar okur → Katman 2 kurtarır.

Retry'da farklı execution_id:
  execution_id = SHA-256(jobId + scheduledAt + retryCount)
  Retry 1: execution_id = "abc...01"
  Retry 2: execution_id = "abc...02"
  → Her retry bağımsız idempotency birimi.
```

---

## Cron Expression Parser

```
Standart 5-alan cron:
  DK   SAAT  GÜN  AY  HAFTA
  0    9     *    *   MON-FRI   → Her hafta içi sabah 9:00
  */15 *     *    *   *         → Her 15 dakikada bir
  0    0     1    *   *         → Her ayın 1'i gece yarısı
  0    2     *    *   0         → Her Pazar sabah 2:00
  0    0     *    *   *         → Her gece yarısı (spike!)

Genişletilmiş cron (6-alan, saniye desteği):
  SN  DK  SAAT  GÜN  AY  HAFTA
  0   0   9     *    *   MON-FRI  → Saniye sıfırda, sabah 9

Özel ifadeler:
  @yearly   = "0 0 1 1 *"   → yılda bir kez
  @monthly  = "0 0 1 * *"   → ayda bir kez
  @weekly   = "0 0 * * 0"   → haftada bir
  @daily    = "0 0 * * *"   → günde bir
  @hourly   = "0 * * * *"   → saatte bir

nextRunAt hesaplama:
  Java: CronExpression (Spring) veya cron-utils kütüphanesi
  CronExpression expr = CronExpression.parse("0 9 * * MON-FRI");
  ZonedDateTime next = expr.next(ZonedDateTime.now(ZoneId.of("Europe/Istanbul")));

Timezone zorunluluğu:
  "0 9 * * *" → İstanbul'da sabah 9 = UTC 06:00 (yaz saatinde 05:00).
  nextRunAt UTC'ye çevirerek DB'de saklanır.
  Yaz/kış saati geçişi: kütüphane otomatik halleder.
  DST hatası olmayı önle: Europe/Istanbul gibi IANA timezone kullan, UTC+3 değil.

İş günü (business day) desteği:
  "Her iş günü saat 18:00" → standart cron yetmez.
  Özel schedule tipi: BUSINESS_DAY_CRON.
  Holiday calendar: tatil günleri DB'de → o günlere skip.
  Örn: "Cumartesi-Pazar + resmi tatiller → çalıştırma."
```

---

## Job Önceliklendirme (Priority Queue)

```
Problem: 10,000 job aynı anda sıraya girdi (gece yarısı spike).
Kritik job (ödeme hatırlatıcı) 5 dk bekleyemez.
Düşük öncelikli job (haftalık rapor) 30 dk beklemesi kabul edilebilir.

Kafka Priority Topics:
  kafka-jobs-high     → partition 4, consumer 4
  kafka-jobs-medium   → partition 4, consumer 2
  kafka-jobs-low      → partition 2, consumer 1

Job'a öncelik atama:
  HIGH:   Kullanıcıya doğrudan etki, SLA var (ödeme bildirimi, alarm)
  MEDIUM: İş süreci (rapor, email kampanyası, veri senkron)
  LOW:    Bakım, temizlik, analitik (log rotate, dead record GC)

DB'den Kafka'ya gönderirken:
  IF job.priority == HIGH   → kafka.send("kafka-jobs-high", message)
  IF job.priority == MEDIUM → kafka.send("kafka-jobs-medium", message)
  IF job.priority == LOW    → kafka.send("kafka-jobs-low", message)

Executor thread pool ayrımı:
  HIGH   executor: 20 thread (her zaman boş thread var)
  MEDIUM executor: 10 thread
  LOW    executor: 5 thread (yoğun olsa bile HIGH etkilenmiyor)

Starvation önleme:
  LOW job 30+ dk beklediyse → MEDIUM'a terfi (priority escalation).
  Escalation scheduler: her 5 dk çalışır, eski LOW job'ları kontrol eder.
```

---

## Job Bağımlılıkları (DAG)

```
Kullanım: ETL pipeline, veri işleme zinciri.
  job_A (veri çek) → job_B (temizle) → job_C (raporla)
  job_B ve job_C: job_A başarıyla tamamlanmadan çalışmamalı.

Veri modeli:
  CREATE TABLE job_dependencies (
    job_id        UUID REFERENCES jobs(job_id),      -- downstream job
    depends_on_id UUID REFERENCES jobs(job_id),      -- upstream job (önce bu bitmeli)
    PRIMARY KEY (job_id, depends_on_id)
  );

Execution akışı:
  1. job_A tamamlandı (status=SUCCESS) → Kafka: JobCompleted {jobId: A, execId: ...}
  2. Dependency Resolver (consumer):
     SELECT job_id FROM job_dependencies WHERE depends_on_id = A
     → [B, C] (B ve C, A'ya bağımlı)
  3. Her downstream için: tüm upstream'ler tamamlandı mı?
     SELECT COUNT(*) FROM job_dependencies d
     JOIN job_executions e ON e.job_id = d.depends_on_id
     WHERE d.job_id = B AND e.status != 'SUCCESS'
     → 0 ise B tetiklenebilir.
  4. B ve C'yi sıraya ekle.

Döngüsel bağımlılık önleme:
  Job oluşturulurken: DAG cycle detection (DFS/topological sort).
  Döngü varsa → 400 Bad Request: "Döngüsel bağımlılık tespit edildi."

Kısmi başarı (fan-in):
  job_D: [A, B, C]'nin hepsine bağımlı.
  A başarılı, B başarılı, C başarısız → D çalışmaz.
  Alternatif: "herhangi biri başarılı olursa çalış" (ANY vs ALL mode).

Hata yayılımı:
  job_A başarısız → B ve C otomatik SKIPPED olarak işaretlenir.
  Alert: "Pipeline A→B→C: A başarısız → 2 job atlandı."
```

---

## Retry Stratejisi

```
Retry policy (job bazında konfigüre edilebilir):
  max_retries:   3 (varsayılan)
  backoff_type:  EXPONENTIAL
  initial_delay: 30s
  multiplier:    2.0
  max_delay:     1h
  jitter:        ±%20 (tüm retry'lar aynı anda spike oluşturmasın)

Execution sırası:
  Deneme 1: başarısız → retry_count=1 → 30s × 1.2 (jitter) = 36s bekle
  Deneme 2: başarısız → retry_count=2 → 60s × 0.9 (jitter) = 54s bekle
  Deneme 3: başarısız → retry_count=3 → 120s bekle
  maxRetries aşıldı → status=DEAD → Dead Letter Queue

Dead Letter Queue (DLQ):
  DLQ topic: kafka-jobs-dead
  Her DEAD execution → DLQ'ya event → alert gönder.
  Dashboard: "Son 24 saatte 15 job DLQ'ya düştü."
  İnsan müdahalesi: DLQ'dan job'ı replay et veya CANCELLED yap.

Retry'ın idempotent olması şart:
  HTTP POST isteği: idempotency-key header gönder → server dedupe.
  Veritabanı işlemi: ON CONFLICT DO NOTHING / UPSERT.
  İdempotent olmayan job: max_retries=0 yap veya DLQ'ya düşünce manuel karar ver.

Farklı hata tipine göre retry:
  5xx (sunucu hatası): retry (geçici olabilir).
  4xx (istemci hatası): retry etme (payload yanlış → retry işe yaramaz).
  Timeout: retry (network geçici).
  Auth error: retry etme (credential yanlış → düzeltilmeli).

  IF response.status == 401 OR response.status == 403:
    status = 'FAILED_PERMANENTLY'  -- retry yok
    alert("Credential sorunu: " + job.name)
```

---

## Uzun Süren Job Yönetimi

```
Problem: Bazı job'lar sn değil dakika/saat sürer (batch işleme, rapor).
  Heartbeat yoksa: executor "ölü" mü çalışıyor mu? bilinmez.
  Timeout çok kısa: iş yarıda kesilir.
  Timeout çok uzun: takılı job tespit edilmez.

Heartbeat mekanizması:
  Executor: her 30 sn → DB güncelle:
    UPDATE job_executions
    SET last_heartbeat = NOW(), progress_pct = :progress
    WHERE execution_id = :id AND status = 'RUNNING';

  Watchdog (ayrı servis, her 1 dk çalışır):
    SELECT * FROM job_executions
    WHERE status = 'RUNNING'
      AND last_heartbeat < NOW() - INTERVAL '2 minutes';
    -- Son 2 dk heartbeat yok → executor muhtemelen çöktü → STALLED olarak işaretle.
    → Retry queue'ya gönder.

Checkpoint (büyük batch job'larda):
  Job: 1M kayıt işleyecek.
  Her 10,000 kayıtta bir: checkpoint kaydet (son işlenen ID).
  Crash → restart: checkpoint'ten devam, baştan değil.
  
  job_checkpoints tablosu:
    (execution_id, checkpoint_key, checkpoint_value, saved_at)
    UPSERT her checkpoint'te.

İptal (cancel):
  Kullanıcı: DELETE /executions/{id}
  API: job_executions SET status='CANCEL_REQUESTED'
  Executor: heartbeat update sırasında status kontrol:
    IF status == 'CANCEL_REQUESTED' → işi durdur → status='CANCELLED'
  Graceful shutdown: SIGTERM → maksimum 30sn tamamla → yoksa force kill.

Timeout konfigürasyonu (job tipine göre):
  HTTP webhook:   30sn (varsayılan)
  Batch report:   1 saat
  Data migration: 6 saat
  timeout_seconds: job tanımında belirtilir.
```

---

## Gece Yarısı Spike Yönetimi

```
Problem: Tüm "0 0 * * *" cron job'ları gece yarısı aynı anda patlar.
  1M job × %1 gece yarısı cron = 10,000 job/dk = 167/sn.
  Executor: normal kapasitesi 30/sn → kuyruk birikir.
  Sonuç: bazı job'lar 10-20 dk geç çalışır.

Çözüm 1 — Jitter (random offset):
  Job oluşturulurken: random_offset = random(0, 300) sn (0-5 dk).
  nextRunAt = midnight + random_offset.
  10,000 job → 300 sn'ye yayılır → 33/sn → yönetilebilir.
  Kullanıcı bilgilendirilir: "Gece 00:00-00:05 arası çalışır."

Çözüm 2 — Gece yarısı için ek executor kapasitesi:
  Kubernetes Scheduled HPA:
    23:55-00:15 → executor pod sayısı 5 → 20.
    00:15 sonra → 5'e dön.
  Gece yarısı ek kapasite → spike absorbe edilir.

Çözüm 3 — Batch enqueue (tek tek değil grup):
  Scheduler: 10,000 job'ı tek seferde Kafka'ya atar (batch produce).
  Kafka: 10 partition → her partition 1,000 job → paralel consume.
  DB: UPDATE nextRunAt tek bir batch UPDATE (10,000 UPDATE vs 10,000 ayrı UPDATE).

Çözüm 4 — Pre-enqueue (önceden kuyruğa al):
  Scheduler: 1 saat öncesinden (23:00) nextRunAt < 01:00 olan job'ları enqueue et.
  Kafka'da: mesajlar var ama delivery time = gece yarısı (Kafka scheduled message).
  Gece yarısı: Kafka mesajları yayınlanır → executor kademeli alır.
  Avantaj: DB spike yok (23:00'da sakin dönemde yapıldı).
```

---

## Missed Job Tespiti ve Kurtarma

```
Senaryo: Scheduler 5 dk down oldu (deployment, crash).
  Bu 5 dk içinde çalışması gereken job'lar kaçırıldı.
  Scheduler yeniden başladı → ne yapmalı?

Catch-up stratejisi (konfigüre edilebilir, job bazında):
  catch_up: true  → kaçırılan job'ları çalıştır (idempotent job için).
  catch_up: false → sadece sonraki zamanlamada çalışsın (non-idempotent).

Kurtarma akışı (startup'ta):
  1. Scheduler başladı.
  2. Missed job detection:
     SELECT * FROM jobs
     WHERE enabled = true
       AND next_run_at < NOW() - INTERVAL '5 minutes'
       AND catch_up = true;
  3. Kaçırılan her job için:
     IF now - next_run_at < catch_up_window (varsayılan: 1 saat):
       → Enqueue (geç de olsa çalıştır).
     ELSE:
       → SKIPPED olarak kaydet, nextRunAt güncelle (bir sonraki zamanlama).

Catch-up window:
  Küçük pencere (5 dk): sadece az geçmişi kurtarır → güvenli ama kaçırma riski.
  Büyük pencere (24 saat): tüm kaçırılanları çalıştırır → idempotent değilse tehlikeli.
  Öneri: HIGH priority job → 1 saat, LOW priority → 15 dk.

Kaçırılan job bildirimi:
  SKIPPED execution kaydı → alert: "5 kritik job atlandı: [listesi]."
  Dashboard: kırmızı gösterge → ops ekibi görür.
```

---

## Multi-Tenant Rate Limiting

```
Problem: Bir tenant 1,000 job oluşturdu, hepsi aynı anda çalıştı → diğer tenant'ların job'ları gecikiyor.

Tenant bazlı limitler:
  free_tier:       max 10 eş zamanlı execution, 100 job/gün
  pro_tier:        max 50 eş zamanlı, 1,000 job/gün
  enterprise_tier: max 200 eş zamanlı, sınırsız/gün

Redis'te sayaç:
  INCR tenant:{tenantId}:concurrent_jobs  → limit aşıldıysa → kuyruğa al (beklet)
  INCRBY tenant:{tenantId}:daily_jobs 1   → günlük limit aşıldıysa → reddedip hata
  EXPIRE otomatik sıfırlama: gün bazlı key, günlük reset.

Executor'da kontrol:
  Kafka'dan mesaj alındı → tenant limit kontrol → geçtiyse execute, geçmediyse delay.
  Delay: exponential backoff → tekrar dene (tenant kapasitesi açılınca).

Tenant izolasyon (kaynak):
  Dedicated executor pool: enterprise tenant → kendi executor thread pool.
  Diğer tenant'ların yavaşlaması enterprise'ı etkilemez.
  Ortak pool: free/pro → ortak ama rate limited.

Fairness (starvation önleme):
  Round-robin scheduling: her tenant'tan sırayla 1 job al.
  A tenant 1000 job kuyruğunda, B tenant 5 job → B önce tükenir → A'dan devam.
  Weighted fair queuing: tenant tier'ına göre ağırlık.
```

---

## Job Tipleri

```java
enum JobType {
    HTTP_WEBHOOK,   // POST belirtilen URL'e
    KAFKA_MESSAGE,  // Belirtilen topic'e mesaj
    LAMBDA,         // AWS Lambda invoke
    GRPC,           // gRPC servis çağrısı
    SHELL_COMMAND   // Shell komutu (sandbox gerekli — tehlikeli!)
}

// HTTP Webhook executor (gelişmiş)
class HttpJobExecutor {

    ExecutionResult execute(JobExecution execution) {
        Job job = execution.getJob();
        HttpWebhookPayload cfg = deserialize(job.getPayload());

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(cfg.getUrl()))
            .method(cfg.getMethod(), HttpRequest.BodyPublishers.ofString(cfg.getBody()))
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer " + cfg.getToken())
            // Idempotency için scheduler tarafından verilen execution ID
            .header("X-Idempotency-Key", execution.getExecutionId())
            .header("X-Job-Id", job.getJobId().toString())
            .timeout(Duration.ofSeconds(job.getTimeoutSeconds()))
            .build();

        try {
            HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

            int status = response.statusCode();

            // 4xx → retry etme (client hatası)
            if (status >= 400 && status < 500) {
                return ExecutionResult.permanentFailure(
                    "HTTP " + status + " (client error, no retry): " + response.body()
                );
            }

            // 5xx → retry
            if (status >= 500) {
                return ExecutionResult.failure("HTTP " + status + ": " + response.body());
            }

            return ExecutionResult.success(response.body());

        } catch (HttpTimeoutException e) {
            return ExecutionResult.failure("Timeout after " + job.getTimeoutSeconds() + "s");
        } catch (IOException e) {
            return ExecutionResult.failure("Network error: " + e.getMessage());
        }
    }
}
```

---

## Takvim Bazlı Zamanlama

```
Kullanım: "Her iş günü saat 08:00'de çalış" (hafta sonu + resmi tatil hariç).

Holiday Calendar tablosu:
  CREATE TABLE holiday_calendars (
    calendar_id UUID PRIMARY KEY,
    name        VARCHAR(100),    -- "Turkey Public Holidays 2025"
    country     CHAR(2),
    year        INT
  );

  CREATE TABLE holidays (
    calendar_id UUID REFERENCES holiday_calendars(calendar_id),
    holiday_date DATE,
    name         VARCHAR(100),
    PRIMARY KEY (calendar_id, holiday_date)
  );

Job'a takvim bağlama:
  jobs.calendar_id → hangi takvim kullanılsın.
  jobs.skip_weekends BOOLEAN DEFAULT FALSE.

nextRunAt hesaplama (takvim aware):
  ZonedDateTime candidate = cronExpr.next(lastRunAt, timezone);
  WHILE isHoliday(candidate, calendarId) OR isWeekend(candidate, skipWeekends):
    candidate = cronExpr.next(candidate, timezone);
  RETURN candidate;

Örnek:
  Cron: "0 8 * * *" (her gün sabah 8).
  calendar: Turkey Public Holidays.
  23 Nisan Ulusal Egemenlik Günü: bu günü atla → 24 Nisan'da çalış.
  Cumartesi-Pazar: skip_weekends=true → Pazartesi'de çalış.
```

---

## Monitoring & Alerting

```
Temel metrikler (Prometheus):
  job_execution_total{status="success|failed|skipped"}  → counter
  job_execution_duration_seconds{job_type="HTTP"}       → histogram (P50, P99)
  scheduler_lag_seconds                                  → şimdiki - nextRunAt fark
  executor_queue_depth{priority="high|medium|low"}      → gauge
  retry_count_total                                      → counter
  dead_letter_total                                      → kritik counter

Alarmlar:
  scheduler_lag > 60s → "Scheduler geride kalıyor, kapasite artır."
  dead_letter_total rate > 5/sn → "Çok fazla job DLQ'ya düşüyor."
  executor_queue_depth{priority="high"} > 100 → "Kritik iş kuyruğu birikti!"
  job başarısız (maxRetries aşıldı) → PagerDuty / Slack alert.
  execution_duration_seconds P99 > timeout * 0.9 → "Job'lar timeout sınırına yaklaşıyor."

SLA monitoring:
  Her job için: max_delay_seconds (zamanında başlamak için tolerans).
  SLA ihlali: started_at > scheduled_at + max_delay_seconds.
  Alert: "Ödeme hatırlatıcı job 3 dk geç başladı (SLA: 1 dk)."

Dashboard (Grafana):
  Execution success/failure rate (saatlik).
  Ortalama gecikme (schedule → actual start).
  Job tiplerine göre çalışma süresi.
  DLQ doluluk trendi.
  Tenant bazlı execution hızı (rate limiti görsel).
```

---

## Veri Modeli (Genişletilmiş)

```sql
CREATE TABLE jobs (
    job_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    schedule        VARCHAR(100),       -- cron expression
    scheduled_at    TIMESTAMPTZ,        -- tek seferlik için
    job_type        VARCHAR(30) NOT NULL CHECK (job_type IN
                    ('HTTP_WEBHOOK','KAFKA_MESSAGE','LAMBDA','GRPC')),
    payload         JSONB NOT NULL,     -- endpoint, topic, body, credentials
    priority        VARCHAR(10) DEFAULT 'MEDIUM'
                    CHECK (priority IN ('HIGH','MEDIUM','LOW')),
    enabled         BOOLEAN DEFAULT TRUE,
    catch_up        BOOLEAN DEFAULT FALSE, -- kaçırılanları çalıştır
    next_run_at     TIMESTAMPTZ,
    timezone        VARCHAR(50) DEFAULT 'UTC',
    calendar_id     UUID REFERENCES holiday_calendars(calendar_id),
    skip_weekends   BOOLEAN DEFAULT FALSE,
    max_retries     INT DEFAULT 3,
    timeout_seconds INT DEFAULT 30,
    max_delay_secs  INT DEFAULT 60,     -- SLA: bu kadar geç başlarsa alarm
    partition_id    INT,                -- scheduler partition assignment
    created_by      UUID,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_jobs_scheduler ON jobs (partition_id, enabled, next_run_at)
  WHERE enabled = true;  -- partial index → sadece aktif job'lar

CREATE TABLE job_executions (
    execution_id    VARCHAR(150) PRIMARY KEY, -- {jobId}_{scheduledAt}_{retryCount}
    job_id          UUID NOT NULL REFERENCES jobs(job_id),
    tenant_id       UUID NOT NULL,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    enqueued_at     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    last_heartbeat  TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','RUNNING','SUCCESS','FAILED',
                                      'DEAD','SKIPPED','CANCELLED','STALLED',
                                      'CANCEL_REQUESTED')),
    retry_count     INT DEFAULT 0,
    output          TEXT,
    error_message   TEXT,
    error_type      VARCHAR(30),  -- TIMEOUT, HTTP_4XX, HTTP_5XX, NETWORK
    duration_ms     BIGINT,
    executor_node   VARCHAR(100), -- hangi pod çalıştırdı
    progress_pct    INT DEFAULT 0 CHECK (progress_pct BETWEEN 0 AND 100)
) PARTITION BY RANGE (scheduled_at);  -- aylık partition

-- Aylık partition'lar
CREATE TABLE job_executions_2025_01 PARTITION OF job_executions
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE job_executions_2025_02 PARTITION OF job_executions
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- ...

CREATE INDEX idx_exec_job     ON job_executions (job_id, scheduled_at DESC);
CREATE INDEX idx_exec_running ON job_executions (status, last_heartbeat)
  WHERE status = 'RUNNING';
CREATE INDEX idx_exec_pending ON job_executions (status, execution_id)
  WHERE status = 'PENDING';

CREATE TABLE job_dependencies (
    job_id        UUID REFERENCES jobs(job_id),
    depends_on_id UUID REFERENCES jobs(job_id),
    require_mode  VARCHAR(10) DEFAULT 'ALL' CHECK (require_mode IN ('ALL', 'ANY')),
    PRIMARY KEY (job_id, depends_on_id)
);

CREATE TABLE job_checkpoints (
    execution_id     VARCHAR(150) REFERENCES job_executions(execution_id),
    checkpoint_key   VARCHAR(100),
    checkpoint_value JSONB,
    saved_at         TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (execution_id, checkpoint_key)
);

CREATE TABLE holiday_calendars (
    calendar_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL,
    country     CHAR(2),
    year        INT
);

CREATE TABLE holidays (
    calendar_id  UUID REFERENCES holiday_calendars(calendar_id),
    holiday_date DATE NOT NULL,
    name         VARCHAR(100),
    PRIMARY KEY (calendar_id, holiday_date)
);
```

---

## Özel Durumlar

```
Timezone & DST:
  "0 9 * * *" → Europe/Istanbul.
  Yaz saati geçişinde (26 Ekim, saat 04:00 → 03:00):
    03:00-04:00 saatleri iki kez yaşanır.
    Cron "0 3 26 10 *" → iki kez mi tetiklensin?
    Çözüm: UTC'de hesapla, yerel sadece gösterim.
    Execution_id: UTC timestamp → yineleme yok.

Uzun kesinti sonrası catch-up (cascade problem):
  24 saat down → 100 job/sn × 86,400 = 8.6M birikmiş job.
  Hepsini aynı anda çalıştır → sistem çöker.
  Throttled catch-up: max 50 job/sn × 24 saat = yavaş toparlanma.
  Seçici catch-up: priority=HIGH → hepsini çalıştır; LOW → son olanı çalıştır, öncekiler skip.

İdempotent olmayan job sorunu:
  E-posta gönder → iki kez gönderilemez.
  Tasarım: her execution için idempotency_key → e-posta servisi dedupe.
  Alternatif: max_retries=0 → sadece bir kez dene.
  DLQ'ya düşen e-posta job'ı → insan karar verir: gönder mi, vazgeç mi.

Executor node affinity:
  Büyük file işleyen job → lokal disk gerekli → belirli node'larda çalışmalı.
  node_selector: "job.nodeSelector=disk-heavy" → executor bu node'lara ata.
  Kubernetes: pod anti-affinity / node label → executor pod seçimi.

Job versiyon yönetimi:
  Job payload değişti → eski execution'lar eski payload'la mı çalışsın?
  Versiyon snapshot: her execution oluşturulurken payload kopyalanır.
  job_executions.payload_snapshot JSONB → o anki payload.
  Güvenli: payload değişse bile çalışmakta olan execution etkilenmez.
```

---

## Olası Sorunlar ve Çözümleri

### 1. Gece Yarısı Spike — Tüm Cron'lar Aynı Anda Patladı

```
Sorun:
  00:00:00.000: 10,000 job aynı anda Kafka'ya eklendi.
  Executor kapasitesi: 30 job/sn → 10,000 / 30 = 333 sn ≈ 5.5 dk gecikme.
  Son job saat 00:05:30'da çalıştı → SLA: 1 dk → ihlal.
  Alert yağmuru: "10,000 job SLA ihlali!"

Çözüm:
  a) Jitter (öneri):
     Job oluşturulurken: random_offset = random(0, 300) sn.
     next_run_at = midnight + offset.
     10,000 job → 300 sn'ye dağılır → 33/sn → kapasitede.
     Kullanıcıya: "Gece 00:00-05:00 arası çalışır" olarak göster.

  b) Scheduled HPA:
     Kubernetes CronJob: 23:55 → executor replicas: 5 → 20.
     00:15 sonra → 5'e dön.
     Maliyet: 20 dk fazladan 15 pod.

  c) Pre-enqueue (23:00'da sıraya al):
     23:00: nextRunAt 00:00-01:00 arası job'ları tespit et.
     Kafka'ya ekle ama delivery_time = midnight (Kafka scheduled message).
     DB update: pre-enqueued = true.
     00:00: Kafka delivery başlar → executor kademeli alır.
     DB spike yok (23:00'da sakin).
```

---

### 2. Leader Crash — RUNNING Job'lar Orphan Kaldı

```
Sorun:
  Leader scheduler 50 job'ı enqueue etti → status=PENDING.
  Executor 30'unu RUNNING yaptı.
  Bu sırada executor pod'un biri çöktü.
  20 job: status=RUNNING, ama hiçbir executor çalıştırmıyor → orphan.
  Sonraki heartbeat gelmiyor → watchdog yok → job sonsuza kadar RUNNING.

Çözüm:
  a) Watchdog servisi (her 1 dk):
     SELECT * FROM job_executions
     WHERE status = 'RUNNING'
       AND last_heartbeat < NOW() - INTERVAL '2 minutes';
     → STALLED olarak işaretle.
     → Retry queue'ya gönder (execution_id yeni retry ile).

  b) Executor heartbeat:
     Executor: her 30 sn → UPDATE last_heartbeat = NOW().
     Executor çöktüyse: heartbeat kesilir → 2 dk'da watchdog yakalar.
     Tolerans: 2 × heartbeat_interval (30sn × 2 = 60sn + buffer 60sn = 2 dk).

  c) K8s terminationGracePeriodSeconds:
     Pod ölmeden: SIGTERM → executor 30sn tamamlamaya çalışır.
     Tamamlanmayan → status = STALLED → 30sn → SIGKILL.
     STALLED: watchdog yakalar → retry.

  d) Heartbeat tabanlı lock (executor sahipliği):
     Executor: her execution için Redis lock: "exec-lock:{execId}" TTL 60sn.
     Her 30sn yenile.
     Executor çöktü → 60sn sonra lock expire → başka executor alabilir.
     Katman: watchdog + Redis lock birlikte (defence in depth).
```

---

### 3. Retry Storm — Tüm Başarısız Job'lar Aynı Anda Retry Yaptı

```
Sorun:
  Downstream servis 10 dk down oldu → 500 job başarısız.
  Servis geri geldi → 500 job aynı anda retry.
  Downstream: ani spike → tekrar down → 500 yeni başarısız → döngü.

Çözüm:
  a) Exponential backoff + jitter (temel):
     Retry 1: base_delay × 1 = 30sn × random(0.8, 1.2) = 24-36sn.
     Retry 2: 30 × 2 × random = 48-72sn.
     Retry 3: 30 × 4 × random = 96-144sn.
     Jitter: farklı job'lar farklı zamanlarda retry → spike yok.

  b) Circuit breaker (tenant / endpoint bazlı):
     Bir endpoint'e son 5 dk'da 10+ başarısız → Circuit OPEN.
     OPEN: yeni retry → anında fail (endpointe gitme, boşuna deneme).
     30 sn sonra: HALF-OPEN → 1 test isteği → başarılı → CLOSED.
     Başarısız → OPEN'a dön.
     Kütüphane: Resilience4j.

  c) DLQ + manual replay:
     3 retry'dan sonra: DLQ → ops alert.
     Servis geri geldi → operator: DLQ'yu replay et (kontrollü hızda).
     Throttled replay: 10 job/sn (downstream koruma).

  d) Bulkhead (endpoint başına paralel limit):
     Her HTTP endpoint: max 5 eş zamanlı retry.
     Yüzlerce retry → sıralanır, 5'er 5'er gider.
     Downstream: kontrollü yük.
```

---

### 4. Scheduler Polling DB'yi Bunalttı — 1M Job × Her Saniye Sorgu

```
Sorun:
  Scheduler: SELECT * FROM jobs WHERE next_run_at < NOW() + 5sn.
  1M job → index taraması: küçük ama her sn × çok node = sorun.
  4 scheduler node × her sn = 4 sorgu/sn → DB'ye yük.
  Execution log: 100,000/gün insert → hot table → yavaşlama.

Çözüm:
  a) Partial index (en önemli):
     CREATE INDEX idx_jobs_scheduler ON jobs (next_run_at)
       WHERE enabled = true;
     Sadece aktif job'lar index'e girer.
     1M job × %80 aktif = 800K → hâlâ büyük.
     + partition_id filtresi → her node sadece kendi partitionını okur → 200K.

  b) Polling aralığını artır + lookahead:
     Her 5 sn → önümüzdeki 30 sn'lik job'ları al.
     Kafka delayed message: 30 sn boyunca delivery time ile.
     DB: her 5 sn sorgula (1/sn yerine) → DB yükü 5x azaldı.

  c) Execution log partitioning:
     job_executions → PARTITION BY RANGE (scheduled_at).
     Aylık partition: INSERT → sadece bu ayın tablosuna → hot table değil.
     Index: her partition kendi indeksini yönetir → küçük, hızlı.

  d) Read replica:
     Scheduler polling: read replica'dan (biraz stale, ama tolere edilebilir).
     Write (nextRunAt update, execution insert): primary.
     Primary yükü: sadece write → çok daha az.
```

---

### 5. Job Bağımlılığı Zinciri Kilitlendi — Döngüsel Bekleme

```
Sorun:
  job_A → job_B → job_C → job_A (yanlışlıkla döngü oluşturuldu).
  A tamamlandı → B bekleniyor. B tamamlandı → C bekleniyor.
  C tamamlandı → A bekleniyor ama A zaten DONE.
  Dependency Resolver: "A'nın DONE'u bekliyorum ama A zaten DONE" → karışık.
  Sonuç: A tekrar çalışmaz, beklenti boşa → job'lar sonsuza askıda.

Çözüm:
  a) Oluşturma anında cycle detection:
     Job dependency eklenirken: DFS ile döngü kontrolü.
     A → B, B → C, C → A eklenmeye çalışıldı → DFS: A → B → C → A (ziyaret edildi) → hata.
     HTTP 400: "Döngüsel bağımlılık oluşturulmak isteniyor."

  b) Execution dependency state machine:
     Her execution başlamadan: tüm upstream'lerin SUCCESS durumu kontrol edilsin.
     Upstream FAILED/DEAD → downstream SKIPPED (bekleme yok, direkt atla).
     Timeout: upstream 24 saat içinde tamamlanmadı → downstream SKIPPED + alert.

  c) Dependency graph versioning:
     Bağımlılık değişti → tüm PENDING downstream execution'ları yeniden değerlendir.
     Yeni bağımlılık eklenince: etkilenen execution'ları bul → re-check.

  d) Orphan dependency detection (haftalık job):
     job_dependencies: depends_on_id referansı → bu job silinmiş mi?
     Silinmiş upstream → orphan dependency → alert + temizle.
```

---

### 6. Executor Sonucu Kaydederken Çöktü — Job Başarılı mı Değil mi?

```
Sorun:
  Executor: HTTP POST → 200 OK aldı (job başarılı).
  DB'ye "SUCCESS" yazarken executor pod çöktü.
  status hâlâ PENDING.
  Watchdog: PENDING + timeout → retry olarak gönderdi.
  Job ikinci kez çalıştı.
  İdempotent değilse: iki kez e-posta gitti, iki kez ödeme çekildi.

Çözüm:
  a) Sonuç yazımı atomik + idempotent:
     UPDATE job_executions
     SET status='SUCCESS', finished_at=NOW(), output=:output
     WHERE execution_id = :id AND status = 'RUNNING';
     Crash sonrası retry → UPDATE tekrar → idempotent (aynı sonuç).

  b) İki adımlı durum: COMPLETING ara durumu:
     Executor: iş bitti → status = 'COMPLETING' (DB commit).
     Sonra: status = 'SUCCESS' (ikinci commit).
     Watchdog: COMPLETING + heartbeat > 30sn → SUCCESS olarak tamamla.
     Böylece "iş bitti ama kayıt olmadı" senaryosu kapanır.

  c) Downstream servis idempotency zorunluluğu:
     HTTP webhook receiver: X-Idempotency-Key header ile dedupe.
     İkinci çağrı: aynı key → "zaten işlendi" → 200 OK (ama işlem tekrar yok).
     Scheduler: X-Idempotency-Key = execution_id header ekler.

  d) Outbox pattern (executor için):
     Executor: iş bitti → local DB'ye "SUCCESS event" kaydet.
     Ayrı outbox publisher: bu event → job store API'sine gönder.
     Crash → restart → outbox tamamlanmamış event → tekrar gönderir.
     Job store: idempotent update → sorun yok.
```

---

### 7. Büyük Tenant Job Flood — Diğer Tenant'lar Mağdur Oldu

```
Sorun:
  Tenant A: 10,000 job oluşturdu, aynı anda hepsini tetikledi.
  Executor queue: 10,000 Tenant A job ile doldu.
  Tenant B ve C: job'ları bekliyor, hiç çalışmıyor.
  SLA ihlali: Tenant B "5 dk içinde çalışmaz" alert.

Çözüm:
  a) Tenant bazlı eş zamanlı execution limiti:
     Tenant A: max 50 eş zamanlı.
     10,000 job → 50'si çalışır, 9,950'si bekler.
     Queue: tenant bazlı alt kuyruklar → Tenant B'nin kuyruğu boş kalmıyor.

  b) Weighted Fair Queuing (WFQ):
     Executor: her turda her tenant'tan sırayla 1 job al.
     Tenant A'nın 10,000'i var, B'nin 5'i → B önce biter, A devam eder.
     Weight: enterprise tenant → 3 token/tur, free → 1 token/tur.

  c) Tenant queue depth limiti:
     Kafka: tenant bazlı partition → max lag limiti.
     Tenant A 10,000 job ekledi → partition lag > 1,000 → yeni job kabul etme.
     HTTP 429: "Kuyruk dolu, lütfen bekleyin."

  d) Burst vs sustained rate:
     Burst: kısa süre yüksek rate (10,000 job 1 sn) → buffer.
     Sustained: saatlik ortalama > limit → throttle.
     Token bucket: 100 token/dk → 100 job/dk steady state.
     Burst: 1,000 token biriktirilebilir (10 dk için).
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Scheduler mimarisi | Tek leader (Redis lock) | Partition-based | Tek leader: SPOF, ölçek yok; Partition: yatay ölçek, partial failure |
| Exactly-once | Kafka exactly-once (transaction) | Idempotency key + DB | Kafka tx: karmaşık, cross-service zor; ID key: basit, güvenilir |
| DB | MySQL | PostgreSQL | SKIP LOCKED, partial index, partitioning, JSONB |
| Öncelik | Tek kuyruk | Priority topic'leri (high/mid/low) | Tek kuyruk: kritik iş haftalık raporla yarışır |
| Retry | Fixed interval | Exponential backoff + jitter | Fixed: retry storm; Backoff+jitter: kademeli, eşit dağıtım |
| Uzun job izleme | Timeout only | Heartbeat + watchdog | Timeout: çöken executor → orphan; Heartbeat: 2 dk'da tespit |
| Gece yarısı spike | Anlık enqueue | Jitter + pre-enqueue | Anlık: DB + executor spike; Jitter: düzgün dağıtım |
| Tenant izolasyon | Shared pool | Weighted fair queuing + rate limit | Shared: bir tenant herkesi bloklar; WFQ: adil, izole |
| Execution log | Tek büyük tablo | Aylık partitioning | Tek tablo: yıl sonunda yavaşlar; Partition: her ay küçük tablo |
| Job bağımlılığı | Manuel sıralama (caller) | DAG + Dependency Resolver | Manuel: insan hatası; DAG: otomatik, cycle detection |
