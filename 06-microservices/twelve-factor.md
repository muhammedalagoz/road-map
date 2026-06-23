# 06e — The Twelve-Factor App

## Ne?

**Twelve-Factor App:** Heroku'nun 2012'de yayımladığı, cloud-native, ölçeklenebilir ve deploy edilebilir uygulama geliştirmek için 12 prensip. Microservices dünyasının temel manifestosu.

```
12 faktör → modern cloud-native uygulama tasarımının checklist'i:
  I.   Codebase        → tek repo, birden fazla deploy
  II.  Dependencies    → bağımlılıkları explicit tanımla
  III. Config          → yapılandırmayı environment'tan al
  IV.  Backing Services→ DB/cache bağlı servis gibi davran
  V.   Build/Release   → build, release, run ayrı aşamalar
  VI.  Processes       → stateless process'ler
  VII. Port Binding    → servis kendi portunu bağlar
  VIII.Concurrency     → process modeli ile ölçeklen
  IX.  Disposability   → hızlı başla, graceful kapat
  X.   Dev/Prod Parity → geliştirme-üretim ortamı benzer
  XI.  Logs            → log'u event stream olarak yönet
  XII. Admin Processes → tek seferlik admin görevleri ayrı çalıştır
```

---

## Neden?

**Çözdüğü problem:** "Bende çalışıyor" sorunu, config yönetimi kabusu, ölçekleme zorluğu, environment farklılıkları.

---

## Nasıl? — Her Faktör

### I. Codebase — Tek Repo, Birden Fazla Deploy

```
✓ Bir uygulama = bir Git repo
✓ Aynı repo → dev, staging, prod'a deploy edilir
✗ Birden fazla app aynı repo'yu paylaşmamalı
✗ Bir app birden fazla repo'da olmamalı

Monorepo:
  Birden fazla uygulama → her biri kendi codebase faktörüne uyar
  (shared-libs repo ≠ app repo)

Microservices:
  OrderService → kendi repo
  PaymentService → kendi repo
  Her servis bağımsız deploy → ✓ twelve-factor uyumlu
```

---

### II. Dependencies — Bağımlılıkları Açıkça Tanımla

```
✓ Maven/Gradle → tüm bağımlılıklar pom.xml / build.gradle'da
✓ Docker → OS-level dependency de declare edilmiş
✗ "Sunucuda yüklü" gibi implicit dependency yok

# pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
# → tüm transitive dep. lock edilmiş (maven wrapper ile)

# Yanlış:
apt-get install imagemagick  # sunucuda kurulu farz etme!
# Doğru: Dockerfile'a ekle
RUN apt-get install -y imagemagick
```

---

### III. Config — Environment'tan Al

```
✓ Config = ortama göre değişen şey
✓ Kod değişmez, environment variable değişir
✗ Config'i kod içine gömme (hardcode)
✗ .properties dosyasını repo'ya commit etme (secret ise)

Twelve-factor test:
  "Repo'yu şu an açık kaynak yapsam güvenli mi?"
  Hayır → secret/config kodda demektir → yanlış

# Yanlış
datasource.url=jdbc:postgresql://prod-db:5432/orders  # hardcoded prod

# Doğru (environment variable)
datasource.url=${DATABASE_URL}

# Spring ile
@Value("${DATABASE_URL}")
String dbUrl;

# Kubernetes Secret
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: url

# Spring Cloud Vault / AWS Secrets Manager
spring:
  cloud:
    vault:
      uri: https://vault.internal
      authentication: KUBERNETES
```

---

### IV. Backing Services — Bağlı Servisi Kaynak Gibi Davran

```
✓ DB, cache, mesaj kuyruğu → hepsi "attached resource"
✓ URL/credential değişince kod değişmemeli
✓ Local PostgreSQL ↔ RDS PostgreSQL → sadece URL değişir

# application.yml
spring:
  datasource:
    url: ${DATABASE_URL}          # local veya prod — kod aynı
  redis:
    host: ${REDIS_HOST}
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS}

Swap edilebilir olmalı:
  Dev:  localhost:5432
  CI:   postgres-ci:5432
  Prod: rds.amazonaws.com:5432
  → Kod değişmez, sadece env var değişir
```

---

### V. Build/Release/Run — Ayrı Aşamalar

```
Build stage:
  git commit → Dockerfile build → Docker image (immutable artifact)
  mvn package → fat jar

Release stage:
  Docker image + Config (env vars, secrets) → Release artifact
  Her release = belirli bir build + belirli bir config
  Release immutable: bir kez oluşturuldu, değiştirilemez

Run stage:
  Release → Kubernetes pod olarak çalıştırılır

✓ Production'da kodu değiştirme → yeni build → yeni release → deploy
✗ SSH ile prod sunucuya girip kod değiştirme → anti-pattern

CI/CD pipeline:
  git push → build → test → Docker push → deploy (GitOps)
```

---

### VI. Processes — Stateless Olun

```
✓ Her process stateless: işlemi bitince hiçbir şey hafızada bırakılmaz
✓ Shared nothing: process'ler birbirinin state'ine erişemez
✓ State = backing service (DB, Redis, S3)
✗ Sticky session → anti-pattern
✗ Local cache (başka instance bilmez)

# Yanlış: In-memory session
HttpSession session = request.getSession();
session.setAttribute("cartItems", cartItems);
// Başka pod → farklı session → cart kayboldu!

# Doğru: Redis'te session
spring:
  session:
    store-type: redis    # tüm pod'lar aynı Redis'i okur

# Yanlış: Local file cache
Files.write(Path.of("/tmp/product-cache.json"), data);
// Başka pod → /tmp yok!

# Doğru: Redis veya S3
redisTemplate.opsForValue().set("product:" + id, product, 5, MINUTES);
```

---

### VII. Port Binding — Kendi Portunu Bağla

```
✓ Uygulama kendi web server'ını içerir (Tomcat, Netty)
✓ HTTP_PORT environment variable ile portu alır
✗ Harici Apache/Nginx web server'ına bağımlı değil
✗ War deploy değil, jar ile çalıştır (Spring Boot fat jar)

# Spring Boot — kendi Tomcat içerir
java -jar order-service.jar  → kendi portunu bağlar (default 8080)

server:
  port: ${PORT:8080}

# Bir servis, başka bir servisin API'sini backing service olarak kullanır:
# OrderService → port 8080
# PaymentService → OrderService'e http://order-service:8080 ile bağlanır
```

---

### VIII. Concurrency — Process Modeli ile Ölçeklen

```
✓ Yatay ölçekleme: aynı process'ten daha fazla başlat
✓ Thread pool yerine process'leri artır (Kubernetes replicas)
✓ Her process tipi ayrı ölçeklenir

Web process:    3 replica → HTTP isteklerini karşılar
Worker process: 10 replica → async job'ları işler
Scheduler:      1 replica → cron trigger

# Kubernetes
kubectl scale deployment order-service --replicas=10

# HPA (Horizontal Pod Autoscaler)
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

### IX. Disposability — Hızlı Başla, Graceful Kapat

```
✓ Fast startup: saniyeler içinde başla (Kubernetes rolling deploy için)
✓ Graceful shutdown: SIGTERM → mevcut istekleri bitir → kapat
✗ Uzun startup süresi → deploy yavaş, ölçekleme yavaş

# Spring Boot graceful shutdown
server:
  shutdown: graceful         # SIGTERM → mevcut istekleri bitir
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # max 30s bekle

# Kubernetes termination
spec:
  terminationGracePeriodSeconds: 40   # SIGTERM + 40s bekle + SIGKILL

# GraalVM native image → <100ms startup (JVM yerine)

# Worker graceful shutdown — Kafka consumer
@PreDestroy
void shutdown() {
    consumer.wakeup();  // poll()'u interrupt et
    // consumer bekleyen job'ları bitirsin → CommitSync → exit
}
```

---

### X. Dev/Prod Parity — Ortamları Benzer Tut

```
Üç boşluk:
  Time gap:    Dev commit → prod deploy → haftalar (CI/CD ile kısalt)
  Personnel:   Dev yazdı, Ops deploy etti (DevOps ile kaldır)
  Tools gap:   Dev: H2 in-memory, Prod: PostgreSQL (aynı DB kullan!)

# Yanlış
# application-dev.properties
spring.datasource.url=jdbc:h2:mem:test  # in-memory, prod'dan farklı davranış!

# Doğru — Docker Compose ile prod-like ortam
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: orders_dev
  redis:
    image: redis:7
  kafka:
    image: confluentinc/cp-kafka:7.5.0

# Dev = docker-compose up → prod ile aynı stack
```

---

### XI. Logs — Event Stream Olarak Yönet

```
✓ Log = event stream, stdout'a yaz
✓ Uygulama log dosyası yönetmez, log toplama sistemi halleder
✗ app.log gibi local file'a yazma
✗ Uygulama kendi log rotation'ını yapmamalı

# application.yml
logging:
  level:
    root: INFO
  pattern:
    console: "%d{ISO8601} %5p [%X{traceId},%X{spanId}] %40.40c: %m%n"
  # file: ile log dosyası açma → stdout yeterli

# Docker → stdout → Fluentd/Filebeat → Elasticsearch → Kibana
# Kubernetes → pod stdout → node log agent → central log system

# Structured logging (JSON)
logging:
  structured:
    format:
      console: logstash   # JSON çıktı → makine tarafından okunur
```

---

### XII. Admin Processes — Tek Seferlik Görevleri Ayrı Çalıştır

```
✓ DB migration, console REPL, tek seferlik script → ayrı process
✓ Aynı release artifact, ayrı command
✗ Uygulama startup'ında migration çalıştırma (prod'da risk)

# Flyway migration — ayrı kubernetes job olarak çalıştır
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migration
          image: order-service:v1.2.3  # aynı image
          command: ["java", "-jar", "app.jar", "--spring.batch.job.enabled=true"]
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: migration
      restartPolicy: Never

# Admin script
kubectl run admin-task \
  --image=order-service:v1.2.3 \
  --restart=Never \
  -- java -jar app.jar --admin.task=fix-orphan-orders
```

---

## Twelve-Factor Checklist

```
□ I.   Codebase    — Tek repo, Git kullanılıyor, her branch deploy edilebilir
□ II.  Deps        — pom.xml/build.gradle tam, Docker image OS dep içerir
□ III. Config      — Secret/URL kod içinde değil, env var veya Vault'ta
□ IV.  Backing     — DB URL env var'dan geliyor, swap edilebilir
□ V.   Build/Run   — CI build → Docker image → Kubernetes deploy, SSH yok
□ VI.  Stateless   — Sticky session yok, in-memory state yok, Redis'te
□ VII. Port        — Uygulama fat jar, kendi Tomcat'ini içeriyor
□ VIII.Concurrency — HPA ile yatay ölçekleme, replicas ayarlanabilir
□ IX.  Disposable  — Graceful shutdown, startup < 30s
□ X.   Parity      — Docker Compose ile prod benzeri local ortam
□ XI.  Logs        — stdout JSON logging, ELK/Loki stack
□ XII. Admin       — Migration Kubernetes Job, admin script ayrı
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Cloud-native deployment uyumlu | Mevcut legacy uygulamayı dönüştürmek emek ister |
| Ortamlar arası tutarlılık | Stateless dönüşüm → Redis gibi state store yatırımı |
| Horizontal scaling kolaylığı | Config management (Vault, K8s Secrets) operasyonel yük |
| Ops/Dev sürtüşmesi azalır | Küçük projeler için fazla overhead |
| 12 faktör → Kubernetes uyumlu | Tüm prensipleri aynı anda uygulamak zor |
