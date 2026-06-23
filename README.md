# Software Architect Roadmap

Senior Software Architect olmak için kapsamlı öğrenme rehberi. Her dosya "nasıl çalışır" seviyesinde teknik derinliğe sahiptir.

## Öğrenme Sırası

```
1. Temeller (CS + OOP + Patterns)
        ↓
2. Java & Spring Ekosistemi
        ↓
3. Veritabanları & Depolama
        ↓
4. Mesajlaşma Sistemleri (Kafka, RabbitMQ)
        ↓
5. Dağıtık Sistemler
        ↓
6. Microservices & DDD
        ↓
7. Altyapı & DevOps
        ↓
8. System Design
        ↓
9. Güvenlik
        ↓
10. Öğrenme Yolu & Kaynaklar
```

## Dizin Yapısı

```
architecture-road-map/
├── 01-foundations/
│   └── foundations.md
├── 02-java-spring/
│   ├── deep-dive.md
│   ├── spring-framework.md
│   ├── spring-boot.md
│   └── spring-cloud.md
├── 03-databases/
│   └── databases.md
├── 04-messaging/
│   ├── kafka.md
│   ├── rabbitmq.md
│   ├── redis-streams.md
│   ├── aws-sqs-sns.md
│   ├── grpc.md
│   ├── debezium-cdc.md
│   ├── mqtt.md
│   └── patterns.md
├── 05-distributed-systems/
│   ├── overview.md
│   ├── consensus-algorithms.md
│   ├── distributed-locks.md
│   ├── gossip-failure-detection.md
│   ├── bulkhead-timeout-retry.md
│   └── split-brain-quorum.md
├── 06-microservices/
│   ├── overview.md
│   ├── database-per-service.md
│   ├── api-versioning.md
│   ├── bff-pattern.md
│   ├── contract-testing.md
│   └── twelve-factor.md
├── 07-infrastructure/
│   └── devops.md
├── 08-system-design/
│   ├── overview.md
│   ├── framework.md
│   ├── scalability-ha.md
│   ├── url-shortener.md
│   ├── twitter-feed.md
│   ├── chat-system.md
│   ├── notification-system.md
│   ├── video-streaming.md
│   ├── google-drive.md
│   ├── search-autocomplete.md
│   ├── payment-system.md
│   ├── ride-sharing.md
│   ├── hotel-booking.md
│   ├── web-crawler.md
│   ├── metrics-monitoring.md
│   ├── job-scheduler.md
│   ├── ad-click-tracking.md
│   ├── live-streaming.md
│   ├── e-commerce.md
│   ├── key-value-store.md
│   └── news-feed-ranking.md
├── 09-security/
│   ├── overview.md
│   ├── owasp-api-top10.md
│   ├── csrf-xss-cors.md
│   ├── cryptography.md
│   ├── zero-trust-rbac.md
│   ├── security-headers.md
│   └── container-supply-chain.md
└── 10-learning-path/
    └── learning-path.md
```

## Dosyalar

### 01 — Temeller

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [foundations.md](01-foundations/foundations.md) | CS Temelleri, OOP, Design Patterns | Yüksek |

### 02 — Java & Spring Ekosistemi

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [deep-dive.md](02-java-spring/deep-dive.md) | JVM internals, GC, Concurrency, Spring IoC/AOP/MVC/Security/Transaction | Yüksek |
| [spring-framework.md](02-java-spring/spring-framework.md) | Spring Core, AOP, SpEL, Events, MVC, WebFlux, WebSocket, Data JPA/Redis/MongoDB, Security | Yüksek |
| [spring-boot.md](02-java-spring/spring-boot.md) | Auto-Config, Actuator, DevTools, Test, Batch, Cache, Retry, Scheduling, AMQP, Kafka, Session | Yüksek |
| [spring-cloud.md](02-java-spring/spring-cloud.md) | Gateway, Config, OpenFeign, Stream, Vault, Eureka, Circuit Breaker | Orta |

### 03 — Veritabanları

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [databases.md](03-databases/databases.md) | SQL, NoSQL, PostgreSQL, Redis, Elasticsearch | Yüksek |

### 04 — Mesajlaşma Sistemleri

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [kafka.md](04-messaging/kafka.md) | Apache Kafka — producer/consumer/broker internals, exactly-once, KRaft | Yüksek |
| [rabbitmq.md](04-messaging/rabbitmq.md) | RabbitMQ — AMQP, exchange türleri, DLX, Quorum Queue, ACK | Yüksek |
| [redis-streams.md](04-messaging/redis-streams.md) | Redis Streams — consumer group, ACK, Kafka ile fark | Orta |
| [aws-sqs-sns.md](04-messaging/aws-sqs-sns.md) | AWS SQS & SNS — managed messaging, fanout, DLQ, FIFO | Orta |
| [grpc.md](04-messaging/grpc.md) | gRPC — protobuf, HTTP/2, 4 streaming modu, schema evolution | Yüksek |
| [debezium-cdc.md](04-messaging/debezium-cdc.md) | Debezium / CDC — WAL okuma, outbox pattern, cache invalidation | Orta |
| [mqtt.md](04-messaging/mqtt.md) | MQTT — IoT protokolü, QoS, LWT, retained message | Orta |
| [patterns.md](04-messaging/patterns.md) | Mesajlaşma Pattern'ları — Saga, DLQ, Backpressure, Idempotency, Schema Registry | Yüksek |

### 05 — Dağıtık Sistemler

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [overview.md](05-distributed-systems/overview.md) | CAP, Saga, Circuit Breaker, Rate Limiting, Idempotency, Caching | Orta |
| [consensus-algorithms.md](05-distributed-systems/consensus-algorithms.md) | Raft, Paxos — lider seçimi, log replication, etcd/ZooKeeper temeli | Yüksek |
| [distributed-locks.md](05-distributed-systems/distributed-locks.md) | Redis Redlock, ZooKeeper lock, fencing token | Orta |
| [gossip-failure-detection.md](05-distributed-systems/gossip-failure-detection.md) | Gossip protokolü, Phi Accrual, SWIM — Cassandra/Consul temeli | Orta |
| [bulkhead-timeout-retry.md](05-distributed-systems/bulkhead-timeout-retry.md) | Bulkhead pattern, timeout stratejileri, exponential backoff + jitter | Yüksek |
| [split-brain-quorum.md](05-distributed-systems/split-brain-quorum.md) | Split-brain problemi, quorum R/W, PACELC teoremi, vector clock | Yüksek |

### 06 — Microservices & DDD

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [overview.md](06-microservices/overview.md) | DDD, Bounded Context, CQRS, Event Sourcing, Service Mesh | Orta |
| [database-per-service.md](06-microservices/database-per-service.md) | DB per service, polyglot persistence, veri paylaşım stratejileri | Yüksek |
| [api-versioning.md](06-microservices/api-versioning.md) | URL/header/query versioning, breaking vs non-breaking, deprecation | Yüksek |
| [bff-pattern.md](06-microservices/bff-pattern.md) | Backend for Frontend — web/mobile/partner BFF, GraphQL | Orta |
| [contract-testing.md](06-microservices/contract-testing.md) | Consumer-Driven Contract Testing, Pact, Can-I-Deploy | Yüksek |
| [twelve-factor.md](06-microservices/twelve-factor.md) | 12 Factor App — cloud-native uygulama tasarım prensipleri | Yüksek |

### 07 — Altyapı & DevOps

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [devops.md](07-infrastructure/devops.md) | Docker, Kubernetes, CI/CD, Observability | Orta |

### 08 — System Design

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [overview.md](08-system-design/overview.md) | Scalability, Caching, System Design örnekleri (referans) | Referans |
| [framework.md](08-system-design/framework.md) | System design framework, capacity estimation, non-functional gereksinimler | Yüksek |
| [scalability-ha.md](08-system-design/scalability-ha.md) | Horizontal/vertical scaling, sharding, replication, HA, deployment stratejileri | Yüksek |
| [url-shortener.md](08-system-design/url-shortener.md) | URL Shortener (bit.ly) — KGS, redirect, cache, ölçekleme | Yüksek |
| [twitter-feed.md](08-system-design/twitter-feed.md) | Twitter/Social Feed — fan-out on write/read, hybrid, timeline cache | Yüksek |
| [chat-system.md](08-system-design/chat-system.md) | Chat Sistemi (WhatsApp) — WebSocket, offline msg, grup, E2EE | Yüksek |
| [notification-system.md](08-system-design/notification-system.md) | Notification Sistemi — Push/Email/SMS, template, rate limit, retry | Yüksek |
| [video-streaming.md](08-system-design/video-streaming.md) | Video Streaming (YouTube) — HLS, ABR, CDN, encoding pipeline | Yüksek |
| [google-drive.md](08-system-design/google-drive.md) | Google Drive/Dropbox — chunk upload, dedup, sync, conflict resolution | Yüksek |
| [search-autocomplete.md](08-system-design/search-autocomplete.md) | Search Autocomplete — Trie, Redis, batch update, kişiselleştirme | Orta |
| [payment-system.md](08-system-design/payment-system.md) | Ödeme Sistemi — idempotency, state machine, double-entry ledger, PCI DSS | Yüksek |
| [ride-sharing.md](08-system-design/ride-sharing.md) | Ride-Sharing (Uber) — GPS, Geosearch, eşleştirme, surge pricing | Yüksek |
| [hotel-booking.md](08-system-design/hotel-booking.md) | Otel/Rezervasyon (Airbnb) — müsaitlik, çift rezervasyon önleme, flash sale | Yüksek |
| [web-crawler.md](08-system-design/web-crawler.md) | Dağıtık Web Crawler — Bloom filter, politeness, URL frontier | Orta |
| [metrics-monitoring.md](08-system-design/metrics-monitoring.md) | Metrics & Monitoring (Prometheus) — TSDB, PromQL, Alert Manager, Thanos | Yüksek |
| [job-scheduler.md](08-system-design/job-scheduler.md) | Distributed Job Scheduler — cron, exactly-once, leader election | Orta |
| [ad-click-tracking.md](08-system-design/ad-click-tracking.md) | Reklam Tıklama Takibi — fire-and-forget, fraud, ClickHouse aggregation | Orta |
| [live-streaming.md](08-system-design/live-streaming.md) | Canlı Yayın (Twitch) — RTMP, HLS/LL-HLS, CDN, WebRTC | Orta |
| [e-commerce.md](08-system-design/e-commerce.md) | E-Ticaret (Amazon) — katalog, sepet, stok lock, flash sale, kampanya | Yüksek |
| [key-value-store.md](08-system-design/key-value-store.md) | KV Store Tasarımı (Redis/DynamoDB) — consistent hashing, LSM-Tree, quorum | Yüksek |
| [news-feed-ranking.md](08-system-design/news-feed-ranking.md) | News Feed Sıralama (Facebook) — ML ranking, affinity, fan-out hybrid | Yüksek |

### 09 — Güvenlik

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [overview.md](09-security/overview.md) | OAuth2, JWT, TLS/mTLS, Vault, Checklist | Orta |
| [owasp-api-top10.md](09-security/owasp-api-top10.md) | OWASP API Top 10 (2023) — BOLA, BFLA, SSRF, Mass Assignment, tüm 10 madde | Yüksek |
| [csrf-xss-cors.md](09-security/csrf-xss-cors.md) | CSRF saldırıları ve önleme, XSS türleri (Stored/Reflected/DOM), CORS konfigürasyonu | Yüksek |
| [cryptography.md](09-security/cryptography.md) | AES-GCM, RSA, ECC, SHA-256, bcrypt/Argon2 şifre saklama, HMAC imzalama | Yüksek |
| [zero-trust-rbac.md](09-security/zero-trust-rbac.md) | Zero Trust mimarisi, SPIFFE/SPIRE, RBAC, ABAC, OPA policy engine, ReBAC | Yüksek |
| [security-headers.md](09-security/security-headers.md) | CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy | Orta |
| [container-supply-chain.md](09-security/container-supply-chain.md) | Docker güvenliği, Trivy, K8s Pod Security, SBOM, Cosign, Dependabot | Orta |

### 10 — Öğrenme Yolu

| Dosya | Konu | Öncelik |
|-------|------|---------|
| [learning-path.md](10-learning-path/learning-path.md) | Öğrenme yolu, kitaplar, projeler | Referans |

---

## Architect Perspektifi

Bir Software Architect şunları yapabilmelidir:

- **Teknik kararları savunabilmek** — "Neden Kafka seçtik, RabbitMQ değil?" sorusuna teknik derinlikte cevap verebilmek
- **Trade-off'ları görebilmek** — Her kararın maliyeti ve faydası vardır, ikisini de görmek
- **Nasıl çalıştığını bilmek** — Kütüphane/framework kullanmak yetmez, altında ne olduğunu bilmek gerekir
- **Sistemi bütün görebilmek** — Tek bir servis değil, tüm sistemin davranışını öngörebilmek
- **İletişim kurabilmek** — Teknik olmayan paydaşlara mimariyi açıklayabilmek

## Nasıl Kullanılır

Her dosyayı okurken şu soruları sor:
1. **Ne?** — Bu teknoloji/pattern ne yapar?
2. **Neden?** — Neden ihtiyaç duyuldu, hangi problemi çözer?
3. **Nasıl?** — İçeride tam olarak nasıl çalışır?
4. **Ne zaman?** — Hangi durumda kullanmalıyım, hangisinde kullanmamalıyım?
5. **Trade-off?** — Avantajları ve dezavantajları neler?
