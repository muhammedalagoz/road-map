# 07 — Altyapı & DevOps

## 1. Docker Internals

### Neden Docker?

"Bende çalışıyor" problemini çözer. Container = process + izolasyon.

### Linux Kernel Özellikleri (Docker'ın temeli)

**Namespaces (izolasyon):**
```
Container içinde görülen dünya:
├── pid namespace   → sadece kendi process'lerini görür
├── net namespace   → kendi network interface'i
├── mnt namespace   → kendi filesystem mount'ları
├── uts namespace   → kendi hostname
├── ipc namespace   → kendi IPC kaynakları
└── user namespace  → user ID mapping (root in container ≠ root on host)
```

**cgroups (resource limiting):**
```
Container'a kaynak limiti koy:
memory.limit_in_bytes = 512MB  → fazla kullanırsa OOMKilled
cpu.shares = 512               → CPU oranı
blkio.weight = 100             → disk I/O önceliği
```

**Overlay Filesystem (image layers):**
```
Image katmanları (read-only):
Layer 4: COPY app.jar /app/app.jar    ← en üst
Layer 3: RUN apt-get install java
Layer 2: FROM ubuntu:22.04
Layer 1: Base layer

Container başladığında:
Tüm read-only katmanlar +
Container layer (read-write) ← değişiklikler buraya
```

Bu sayede aynı base image birden fazla container paylaşır (disk tasarrufu).

### Dockerfile Best Practices

```dockerfile
# Multi-stage build — final image küçülür
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline  # bağımlılıkları önce çek (cache layer)
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine  # sadece JRE, JDK değil
WORKDIR /app
COPY --from=builder /app/target/app.jar .

# Non-root user (güvenlik)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Layer caching:** `COPY pom.xml` + `mvn dependency:go-offline` önce → kaynak kodu değişse bile bağımlılıklar tekrar indirilmez.

---

## 2. Kubernetes Nasıl Çalışır

### Mimari

```
Control Plane (Master):
├── kube-apiserver    ← tek giriş noktası, RESTful API
├── etcd              ← cluster state'in key-value store'u
├── kube-scheduler    ← Pod'u hangi node'a koyalım?
├── cloud-controller  ← cloud provider entegrasyonu
└── kube-controller-manager
    ├── ReplicaSet controller
    ├── Deployment controller
    ├── Node controller
    └── Service controller

Worker Nodes:
├── kubelet           ← Pod'ları yönetir, API server ile konuşur
├── kube-proxy        ← network rules (iptables/IPVS)
└── Container Runtime ← containerd / CRI-O (Docker deprecated)
```

### Pod Yaşam Döngüsü

```
kubectl apply -f deployment.yaml
        ↓
API Server → etcd'ye yaz
        ↓
Deployment Controller → ReplicaSet oluştur
        ↓
ReplicaSet Controller → Pod oluştur
        ↓
Scheduler → Node seç (resource fit, affinity, taints/tolerations)
        ↓
kubelet (seçilen node) → container runtime'a image pull + container oluştur
        ↓
Pod Running
        ↓
Liveness Probe → başarısız → container restart
Readiness Probe → başarısız → Service'ten çıkar (traffic kesme)
```

### Kubernetes Networking

**CNI (Container Network Interface):** Her Pod IP alır, Pod'lar birbirini direkt IP ile bulur.

```
Sorun: Container'lar node'a bağlı → node değişirse IP değişir
Çözüm: Service → stable virtual IP (ClusterIP) + DNS

kube-proxy her node'da iptables kuralları yazar:
ClusterIP:80 → {Pod1:8080, Pod2:8080, Pod3:8080} (random selection)

DNS:
orderservice.default.svc.cluster.local → ClusterIP
```

**Service Türleri:**
```
ClusterIP (default):
  Sadece cluster içi erişim
  orderservice:8080 → Pod'lara yönlendir

NodePort:
  Her node'un belirli portuna dışarıdan erişim
  Node IP:30080 → Service → Pod
  Port range: 30000-32767
  Production'da nadiren kullanılır

LoadBalancer:
  Cloud provider'dan external LB ister (AWS ELB, GCP Load Balancer)
  External IP → LB → Node → Service → Pod

Ingress:
  HTTP/HTTPS routing, host/path bazlı
  L7 load balancing
  TLS termination
  
  example.com/orders → OrderService
  example.com/users  → UserService
```

### Resource Management

```yaml
resources:
  requests:          # Scheduler bu değere göre node seçer
    memory: "256Mi"
    cpu: "250m"      # 250 millicore = 0.25 CPU
  limits:            # Aşınca: CPU throttle edilir, memory OOMKilled
    memory: "512Mi"
    cpu: "500m"

# QoS Classes:
# Guaranteed: requests == limits (en öncelikli, son evict)
# Burstable: requests < limits
# BestEffort: hiç request/limit yok (ilk evict edilir)
```

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU %70'e gelince scale out
```

---

## 3. CI/CD Pipeline

```
Developer → Push → Git Repository
                         ↓
                    CI Pipeline (GitHub Actions / Jenkins / GitLab CI)
                    ├── 1. Code checkout
                    ├── 2. Dependency install (cache)
                    ├── 3. Static analysis (SonarQube, SpotBugs)
                    ├── 4. Unit tests
                    ├── 5. Integration tests
                    ├── 6. Docker build
                    ├── 7. Security scan (Trivy, Snyk)
                    ├── 8. Push to registry (ECR, GCR, Docker Hub)
                    └── 9. Deploy (Staging)
                         ↓
                    CD Pipeline
                    ├── E2E tests
                    ├── Performance tests (k6, Gatling)
                    ├── Manual approval (production için)
                    └── Deploy to Production
```

**GitHub Actions örneği:**
```yaml
name: CI/CD
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21' }
      - name: Cache Maven
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
      - run: mvn test
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS ...
          docker push myapp:${{ github.sha }}
```

### Deployment Stratejileri

**Rolling Update (varsayılan K8s):**
```
v1 v1 v1 v1
→ v2 v1 v1 v1
→ v2 v2 v1 v1
→ v2 v2 v2 v1
→ v2 v2 v2 v2
Sıfır downtime, ama anlık iki versiyon çalışır
```

**Blue-Green:**
```
Blue (v1) aktif → Green (v2) deploy et → test et → traffic switch → Blue silinir
Anlık rollback: trafiği Blue'ya geri al
Dezavantaj: 2x kaynak
```

**Canary:**
```
v1: %90 traffic
v2: %10 traffic → metrikler iyi mi? → %50 → %100
Hata varsa: %10'a rollback et
```

---

## 4. Observability (Gözlemlenebilirlik)

Üç sütun:

### Metrics (Prometheus + Grafana)

```
Prometheus: pull-based metrics collector
Uygulama /actuator/prometheus endpoint'i açar
Prometheus her N saniyede pull yapar

Metric tipleri:
Counter:   artan değer (request sayısı, hata sayısı)
Gauge:     anlık değer (aktif connection, memory)
Histogram: dağılım (request duration, response size)
Summary:   percentile (p95, p99 latency)
```

```yaml
# Spring Boot actuator + micrometer:
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

### Distributed Tracing (Jaeger / Zipkin)

```
User Request → OrderService → PaymentService → InventoryService

TraceId: abc123 (tüm servislerde aynı)
Spans:
├── OrderService   [0ms - 150ms]    spanId: span1
│   ├── PaymentService [10ms - 80ms] spanId: span2, parentId: span1
│   └── InventoryService [90ms - 140ms] spanId: span3, parentId: span1

Distributed tracing ile:
- Hangi servis yavaş? → span sürelerine bak
- Nerede hata? → error span'i bul
- Servisler arası sıra → trace görselleştir
```

```xml
<!-- Spring Boot Micrometer Tracing -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Logging (ELK Stack)

```
Application → Logstash (collect, parse) → Elasticsearch (store) → Kibana (visualize)

Yapılandırılmış log (JSON):
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "traceId": "abc123",
  "spanId": "span1",
  "service": "order-service",
  "message": "Payment failed",
  "orderId": "ORD-456",
  "customerId": "CUST-789",
  "error": "Connection timeout"
}

TraceId log'da olunca → Jaeger'da trace'i bul → tam akış görülür
```

**Golden Signals (Google SRE):**
```
1. Latency   — başarılı/başarısız request süresi
2. Traffic   — RPS (request per second)
3. Errors    — hata oranı (%)
4. Saturation — kapasite ne kadar dolu (CPU, memory, queue)
```

---

## 5. Infrastructure as Code (Terraform)

```hcl
# main.tf
provider "aws" {
  region = "eu-west-1"
}

resource "aws_db_instance" "orders" {
  identifier        = "orders-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  
  username = var.db_username
  password = var.db_password  # secrets manager kullan, hardcode etme!
  
  multi_az               = true  # HA
  backup_retention_period = 7
}

# terraform init → terraform plan → terraform apply
```

**Neden IaC?**
- Altyapı versiyonlanır (git history)
- Reproducible environment (dev/staging/prod aynı yapı)
- Code review ile altyapı değişikliği
- Disaster recovery: her şeyi yeniden yaratabilirsin
