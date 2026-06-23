# 10 — Öğrenme Yolu & Kaynaklar

## Seviye Haritası

### Junior Developer (0-2 yıl)
Odak: Kod yazmayı öğren, bir dili iyi bil.
```
□ Java/Python/Go'dan birini iyi öğren
□ OOP prensipleri (SOLID)
□ SQL temelleri (JOIN, GROUP BY, index)
□ REST API tasarımı
□ Git (branch, merge, rebase)
□ Unit test yazma (JUnit, Mockito)
□ Spring Boot: basit CRUD uygulama
□ Docker: container oluştur, çalıştır
```

### Mid-Level Developer (2-5 yıl)
Odak: Sistemi anla, tasarım kararı ver.
```
□ Design patterns (GoF 23 pattern)
□ Concurrency (thread, async, reactive)
□ Database: index, query optimization, N+1
□ Cache: Redis, stratejiler
□ Message queue: Kafka veya RabbitMQ ile çalış
□ Microservices: en az 2-3 servisli sistem kur
□ REST API best practices, API versioning
□ Kubernetes: pod, service, deployment, ingress
□ CI/CD pipeline kur (GitHub Actions)
□ Monitoring: Prometheus + Grafana
□ Distributed tracing: Jaeger
□ Security: JWT, OAuth2, HTTPS
```

### Senior Developer (5+ yıl)
Odak: Sistem tasarla, ekibi yönlendir.
```
□ System Design: büyük ölçekli sistemleri tasarla
□ DDD: bounded context, aggregate, domain events
□ Event Sourcing & CQRS
□ Distributed systems: CAP, Saga, 2PC
□ Performance tuning: JVM GC, DB query plan, profiling
□ Capacity planning & cost optimization
□ Incident management: on-call, RCA (Root Cause Analysis)
□ Mentoring: junior developer'ları yönlendir
□ Architecture Decision Records (ADR) yaz
```

### Software Architect
Odak: Organizasyon geneli teknik yön, trade-off kararları.
```
□ Enterprise Architecture patterns
□ Multi-team coordination (API contracts, versioning)
□ Technology radar: ne adopt, ne hold, ne retire?
□ Non-functional requirements: SLA, RTO, RPO tanımla
□ Cloud architecture: multi-region, disaster recovery
□ FinOps: maliyet optimizasyonu
□ Security by design
□ Technical debt management
□ Stakeholder communication (teknik → iş dili)
```

---

## Öğrenme Sırası (Bu Roadmap için)

```
Ay 1-2: Foundations + Java/Spring
├── 01-foundations.md
├── 02-java-spring-deep-dive.md
└── Proje: Spring Boot + PostgreSQL CRUD API

Ay 3: Databases
├── 03-databases.md
└── Proje: Mevcut projeye Redis cache + Elasticsearch ekle

Ay 4: Messaging
├── 04-messaging-systems.md
└── Proje: Kafka ile order event streaming implementasyonu

Ay 5-6: Distributed Systems + Microservices
├── 05-distributed-systems.md
├── 06-microservices.md
└── Proje: Monolith'i 3 microservice'e böl

Ay 7: Infrastructure
├── 07-infrastructure-devops.md
└── Proje: Kubernetes'e deploy et, CI/CD kur

Ay 8: System Design
├── 08-system-design.md
└── Pratik: Her gün 1 system design sorusu çöz (30 gün)

Ay 9: Security
├── 09-security.md
└── Proje: OAuth2/JWT, security audit

Ay 10+: Derinleştir, contribute et, öğret
```

---

## Kitaplar (Öncelik Sırasıyla)

### Zorunlu Oku

| Kitap | Yazar | Neden? |
|-------|-------|--------|
| **Designing Data-Intensive Applications** | Martin Kleppmann | Dağıtık sistemlerin İncil'i. Her architect okumalı. |
| **Clean Code** | Robert C. Martin | Kod kalitesi temeli |
| **The Pragmatic Programmer** | Hunt & Thomas | Profesyonel yazılımcı zihniyeti |
| **System Design Interview Vol 1-2** | Alex Xu | Pratik sistem tasarımı, gerçek örnekler |

### Orta Seviye

| Kitap | Yazar | Neden? |
|-------|-------|--------|
| **Domain-Driven Design** | Eric Evans | DDD'nin kaynak kitabı, zor ama zorunlu |
| **Microservices Patterns** | Chris Richardson | Saga, CQRS, Outbox hepsi burada |
| **Building Microservices** | Sam Newman | Microservice mimarisi kapsamlı |
| **Release It!** | Michael Nygard | Production-grade sistem tasarımı |
| **Kubernetes in Action** | Marko Luksa | K8s en iyi kitabı |

### İleri Seviye

| Kitap | Yazar | Neden? |
|-------|-------|--------|
| **Site Reliability Engineering** | Google | SRE kültürü, error budget, SLA |
| **Enterprise Integration Patterns** | Hohpe & Woolf | Messaging patterns klasiği |
| **A Philosophy of Software Design** | John Ousterhout | Tasarım derinliği |
| **Accelerate** | Forsgren, Humble, Kim | DevOps metrikler (DORA), engineering culture |

---

## Pratik Projeler

### Proje 1: E-Commerce Backend (Monolith → Microservices)
```
Aşama 1: Spring Boot monolith
- User, Product, Order, Payment, Notification modülleri
- PostgreSQL + Redis cache
- JWT auth

Aşama 2: Microservices
- Her modülü ayrı servise böl (Strangler Fig)
- API Gateway (Spring Cloud Gateway)
- Kafka event streaming

Aşama 3: Production-ready
- Docker + Kubernetes deployment
- Prometheus + Grafana monitoring
- Distributed tracing (Jaeger)
- CI/CD pipeline
```

### Proje 2: Real-time Chat Sistemi
```
WebSocket ile gerçek zamanlı mesajlaşma
Redis pub/sub (birden fazla server instance)
Message history: PostgreSQL
File upload: S3
Push notification: FCM
```

### Proje 3: URL Shortener (System Design Pratiği)
```
08-system-design.md'deki tasarımı implement et:
- Key generation service
- Redirect service (read-heavy, cache kritik)
- Analytics: tıklama sayısı, coğrafi dağılım
- Rate limiting
```

---

## Günlük Pratik

### System Design
```
Her gün 30 dakika:
- Leetcode System Design soruları
- Grokking System Design (educative.io)
- ByteByteGo (Alex Xu'nun bülteni)
```

### Kod Kalitesi
```
Haftada 1 PR review: başkasının kodunu incele
Her PR'da: "Bu kod 1 yıl sonra anlaşılabilir mi?"
```

### Mimari Karar Günlüğü (ADR)
```
Her önemli teknik kararı yaz:
- Bağlam: Problem neydi?
- Karar: Ne seçtik?
- Gerekçe: Neden?
- Sonuç: Ne oldu?

Bir yıl sonra bak: Kararların ne kadarı doğruydu?
```

---

## Interview Hazırlık

### Coding Interview
```
LeetCode: Önce Easy (30 gün) → Medium (60 gün)
Kategoriler:
- Array/String (en sık)
- HashMap/HashSet
- Binary Search
- Tree/Graph (BFS/DFS)
- Dynamic Programming (zor ama çıkıyor)
```

### System Design Interview
```
Framework (tekrar et):
1. Gereksinimleri netleştir
2. Capacity estimate
3. High-level tasarım
4. Deep dive
5. Bottleneck & improvement

Pratik et (bu sistemleri tasarla):
- URL shortener
- Twitter timeline
- Netflix video streaming
- Uber / Google Maps
- WhatsApp / Slack
- Instagram
- Rate limiter
- Distributed cache
```

### Davranışsal Interview (STAR)
```
S - Situation: Bağlam neydi?
T - Task: Görevin neydi?
A - Action: Ne yaptın?
R - Result: Sonuç ne oldu?

Hazırlan:
- En zor teknik karar aldığın an
- Takım çatışmasını çözdüğün an
- Başarısız proje ve öğrendiğin ders
- Teknik borcu yönettiğin an
- Deadline baskısı altında çalıştığın an
```

---

## Kaynaklar

### Online Platformlar
```
ByteByteGo         → system design, Alex Xu'nun videoları
Martin Fowler Blog → enterprise patterns, microservices
High Scalability   → gerçek şirket mimarisi yazıları
InfoQ              → conference talks, architecture
The Morning Paper  → akademik paper özetleri
```

### YouTube Kanalları
```
Fireship       → hızlı teknoloji tanıtımları
Hussein Nasser → backend engineering derinlemesine
ByteByteGo     → system design animasyonları
TechWorld with Nana → DevOps, Kubernetes
```

### Podcast
```
Software Engineering Daily → derinlemesine teknik röportajlar
The Changelog             → open source, engineering culture
Distributed Systems Podcast
```

### Topluluk
```
GitHub: açık kaynak projelere contribute et
Stack Overflow: soru sor, cevapla
Tech blog: öğrendiklerini yaz (öğretmek en iyi öğrenme yöntemi)
Meetup/Konferans: KubeCon, JavaOne, SpringOne
```
