# 06e — Twelve-Factor App: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Config, Secrets & Stateless

### Gerçek Hayat Sorunları

---

**Sorun 1: Config hardcode — production DB credential'ı Git'e commit edildi**

```
Senaryo:
  Developer: "hızlıca test edeyim" dedi, application.properties'e prod URL yazdı.

  # application.properties
  spring.datasource.url=jdbc:postgresql://prod-db.company.com:5432/orders
  spring.datasource.username=app_user
  spring.datasource.password=Sup3rS3cr3t!

  git add . && git commit -m "fix: db connection"
  git push origin main

  Sonuç:
    GitHub public repo ise: credential anında internette.
    Private repo olsa bile: git log herkes görebilir.
    Secret scan (GitGuardian, GitHub Secret Scanning) alert verdi.
    Güvenlik ekibi: "prod DB şifresi değiştirilmeli" → acil incident.
    Şifre değişikliği: tüm bağlanan servisler güncellenmeli → koordinasyon.

Kök neden: III. faktör ihlali — config kod içinde.

Twelve-factor test:
  "Bu repo'yu şu an public yapsam güvenli mi?"
  Hayır → config/secret kodda demek → yanlış.

Düzeltme:

  1. Environment variable (temel):
     export DATABASE_URL="jdbc:postgresql://..."
     export DB_PASSWORD="..."
     
     # application.yml
     spring:
       datasource:
         url: ${DATABASE_URL}
         password: ${DB_PASSWORD}

  2. Kubernetes Secret:
     kubectl create secret generic db-secret \
       --from-literal=url="jdbc:postgresql://..." \
       --from-literal=password="..."
     
     # Pod spec:
     env:
       - name: DATABASE_URL
         valueFrom:
           secretKeyRef:
             name: db-secret
             key: url

  3. Vault / AWS Secrets Manager (production standartı):
     Uygulama başlangıcında Vault'tan çeker.
     Secret rotation: kod değişmez, Vault güncellenir.
     Audit: kim, ne zaman, hangi secret'a erişti?

  Git geçmişi temizleme:
    git filter-branch veya BFG Repo Cleaner ile geçmişten sil.
    Ama: "bir kez commit edilen secret = tehlikeli" → şifreyi değiştir.
```

---

**Sorun 2: Stateful process — sticky session, scale-out başarısız**

```
Senaryo:
  E-ticaret: sepet bilgisi HTTP Session'da (in-memory).

  // Yanlış
  @PostMapping("/cart/add")
  void addToCart(@RequestBody Product product, HttpSession session) {
      List<Product> cart = (List<Product>) session.getAttribute("cart");
      if (cart == null) cart = new ArrayList<>();
      cart.add(product);
      session.setAttribute("cart", cart);   // Sunucunun RAM'inde!
  }

  Black Friday: 1 pod → 10 pod'a scale-out yapıldı (Kubernetes HPA).
  
  Kullanıcı:
    T+0:  Pod-1'e istek → sepete ürün ekledi → Pod-1 RAM'inde cart.
    T+1:  Pod-3'e yönlendirildi (load balancer round-robin) → cart boş!
    Kullanıcı: "sepetim neden boş?"

  Sticky session çözümü (anti-pattern):
    Load balancer: aynı kullanıcı → hep aynı Pod.
    Sorun: Pod ölünce → session kaybı. Scale-out dengeli değil (bazı Pod'lar dolu).
    Anti-pattern: XII faktör ihlali, gerçek stateless değil.

Doğru çözüm: VI. faktör — stateless process:
  Session → Redis'e taşı:

  # pom.xml
  <dependency>
      <groupId>org.springframework.session</groupId>
      <artifactId>spring-session-data-redis</artifactId>
  </dependency>

  # application.yml
  spring:
    session:
      store-type: redis
      redis:
        namespace: "session:"
    data:
      redis:
        host: ${REDIS_HOST}

  Artık:
    Her Pod → Redis'ten session okur.
    Pod-1 veya Pod-3: fark etmez, cart Redis'te.
    Pod ölünce: session kaybolmaz (Redis'te).
    Scale-out: sorunsuz, herhangi Pod → herhangi kullanıcı.
    Sticky session: gereksiz.
```

---

**Sorun 3: Dev/Prod parity ihlali — H2 ile geliştirme, PostgreSQL ile production**

```
Senaryo:
  Geliştirici: "hızlı test için H2 in-memory kullanıyorum."

  # application-dev.properties
  spring.datasource.url=jdbc:h2:mem:testdb
  spring.datasource.driver-class-name=org.h2.Driver
  spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

  # application-prod.properties
  spring.datasource.url=${DATABASE_URL}
  spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

  Prodüksiyon bug'ları:
  
  Bug 1 — Case sensitivity:
    H2: SQL case-insensitive. PostgreSQL: case-sensitive.
    Dev'de: SELECT * FROM Users → çalışıyor.
    Prod'da: "users" table adı küçük → hata. Dev'de hiç fark edilmedi.

  Bug 2 — Boolean tipi:
    H2: BOOLEAN destekler. PostgreSQL: BOOLEAN var ama TINYINT farklı.
    JPA mapping: dev'de sorunsuz → prod'da tip uyumsuzluğu.

  Bug 3 — Constraint davranışı:
    H2: bazı constraint'leri farklı enforce eder.
    Unique constraint ihlali: H2'de silent, PostgreSQL'de exception.

  Bug 4 — Native query:
    PostgreSQL-specific sorgu: dev'de H2 syntax → prod'da hata.
    RETURNING clause: PostgreSQL'de var, H2'de yok.

  X. faktör çözümü — Docker Compose ile prod-like ortam:

  # docker-compose.yml
  services:
    postgres:
      image: postgres:16
      environment:
        POSTGRES_DB: orders_dev
        POSTGRES_USER: app
        POSTGRES_PASSWORD: dev_password
      ports:
        - "5432:5432"
    redis:
      image: redis:7
      ports:
        - "6379:6379"
    kafka:
      image: confluentinc/cp-kafka:7.5.0
      ports:
        - "9092:9092"

  # application-dev.properties
  spring.datasource.url=jdbc:postgresql://localhost:5432/orders_dev
  # Prod ile aynı dialect, aynı davranış!

  docker-compose up → dev = prod ile aynı stack.
  H2 ve prod farklılığından kaynaklanan bug: sıfır.
```

---

**Sorun 4: Graceful shutdown yok — deploy sırasında aktif istekler kesildi**

```
Senaryo:
  Kubernetes rolling deploy: yeni pod geldi, eski pod SIGTERM aldı.
  
  Eski pod: 50 aktif HTTP isteği işliyordu (10-500ms süren).
  SIGTERM → uygulama anında kapandı (graceful shutdown yok).
  50 istek: mid-flight kesildi → client 502 Bad Gateway aldı.
  
  Kullanıcı: ödeme formunu doldurdu → "submit" → 502.
  "Ödeme yapıldı mı yapılmadı mı?" → destek talebi.

  Ayrıca: Kafka consumer pod'u SIGTERM aldı.
    Consumer: offset commit yapılmamış mesajlar var.
    Anında kapandı → mesajlar tekrar işlendi → duplicate.

IX. faktör çözümü — graceful shutdown:

  Spring Boot:
  server:
    shutdown: graceful           # SIGTERM → yeni istek alma → mevcut bitir
  spring:
    lifecycle:
      timeout-per-shutdown-phase: 30s  # 30s bekle

  Kubernetes:
  spec:
    terminationGracePeriodSeconds: 40  # SIGTERM + 40s + SIGKILL
    # Spring 30s + buffer 10s = 40s

  Kubernetes lifecycle hook:
  lifecycle:
    preStop:
      exec:
        command: ["sleep", "5"]
    # Service → Pod'u endpoint listesinden çıkarması için 5s bekle
    # Sonra SIGTERM → aktif istekler bitti → graceful kapat

  Kafka consumer graceful shutdown:
  @PreDestroy
  void shutdown() {
      log.info("Shutting down Kafka consumer...");
      consumer.wakeup();        // poll() interrupt et
      // thread join: consumer son batch'i işlesin, commitSync yapsın
  }

  Startup hızı (disposability'nin diğer yüzü):
  - Spring Boot native image (GraalVM): <100ms başlangıç.
  - Lazy bean initialization: sadece gerekli bean'ler başlangıçta.
  - Kubernetes readinessProbe: hazır olmadan trafik gelmesin.
  
  readinessProbe:
    httpGet: {path: /actuator/health/readiness, port: 8080}
    initialDelaySeconds: 10
    periodSeconds: 5
```

---

**Sorun 5: DB migration startup'ta — zero-downtime deploy bozuldu**

```
Senaryo:
  // Yanlış: uygulama başlangıcında migration
  # application.yml
  spring:
    flyway:
      enabled: true   # her pod başlangıcında Flyway çalışır

  Rolling deploy: 3 pod → yeni image ile sırayla güncelleniyor.
  
  T+0:  Pod-1 (yeni): başladı → Flyway migration çalıştı.
        ALTER TABLE orders ADD COLUMN estimated_delivery DATE;
  T+1:  Pod-2 (eski): hâlâ çalışıyor → "estimated_delivery" kolonunu bilmiyor.
        INSERT INTO orders (...) VALUES (...) → hata! (yeni şema, eski kod)
  T+2:  Pod-3 (eski): aynı hata.

  Sorun:
    Eski ve yeni pod aynı anda çalışıyor (rolling deploy).
    Eski pod yeni şemayı bilmiyor → hata.
    Flyway: migration'ı birden fazla pod aynı anda çalıştırmaya çalışıyor → lock.

XII. faktör çözümü — migration Kubernetes Job olarak:
  
  # Migration job (deploy öncesi çalışır):
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: db-migration-v1-2-3
  spec:
    template:
      spec:
        containers:
          - name: migration
            image: order-service:v1.2.3     # aynı image
            command: ["java", "-jar", "app.jar"]
            env:
              - name: SPRING_PROFILES_ACTIVE
                value: migration            # sadece Flyway çalıştır
        restartPolicy: Never
  
  # application.yml
  spring:
    flyway:
      enabled: false   # uygulama başlarken ÇALIŞMA
  
  # application-migration.yml
  spring:
    flyway:
      enabled: true    # sadece migration profile'ında çalış
  
  CI/CD pipeline:
    1. Migration job → tamamlandı mı? (kubectl wait)
    2. Rolling deploy → eski pod'lar yeni şemayı biliyor (backward compat migration)
    3. Her pod: Flyway yok → hızlı başlangıç.

  Zero-downtime migration kuralı:
    Backward-compatible migration: yeni kolon nullable veya default → eski pod sorunsuz.
    İki aşamalı: yeni kolon ekle (backward compat) → deploy → eski kolonu kaldır (ayrı deploy).
```

---

**Sorun 6: Log dosyasına yazma — log rotation bozuldu, disk doldu**

```
Senaryo:
  # application.yml (yanlış)
  logging:
    file:
      name: /var/log/order-service/app.log   # dosyaya yaz
      max-size: 100MB
      max-history: 30

  Production sorunu:
    Pod restart: /var/log/ → emptyDir → sıfırlandı → log geçmişi kayboldu.
    Persistent volume: log için PVC → her pod'a ayrı disk → pahalı.
    Log rotation: JVM içinde → restart → rotation sıfırlandı → tam 100MB'de bıraktı.
    Disk dolu: /var/log/ dolunca uygulama log yazamadı → exception.
    Merkezi log: her pod'un /var/log/'unu ayrı ayrı incele → zahmetli.

XI. faktör çözümü — stdout'a yaz, log sistemi halleder:

  # application.yml (doğru)
  logging:
    level:
      root: INFO
      com.company: DEBUG
    pattern:
      console: "%d{ISO8601} %-5p [%X{traceId},%X{spanId}] %logger{36}: %m%n"
    # file: SATIRI YOK — sadece stdout

  Structured JSON logging (makine okunabilir):
  logging:
    structured:
      format:
        console: logstash    # JSON çıktı

  Kubernetes log pipeline:
    Pod stdout → Fluentd/Filebeat (node agent) → Elasticsearch → Kibana
    veya: → Loki → Grafana (daha hafif)
    
    Her pod aynı agent kullanır → merkezi log → tek arayüz.
    Log geçmişi: Elasticsearch'te → pod restart → log kaybolmaz.
    Log rotation: Elasticsearch index lifecycle → otomatik.

  TraceId ile log:
    MDC.put("traceId", span.getTraceId());
    log.info("Order created: {}", orderId);
    # Çıktı: {"traceId":"abc123","message":"Order created: 456"}
    # Kibana: traceId="abc123" → tüm loglar bir arada.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Twelve-Factor App nedir? Neden önemlidir?

   > **Beklened:** Heroku'nun 2012'de yayımladığı 12 prensip — cloud-native, ölçeklenebilir uygulama geliştirme manifestosu. Neden: "bende çalışıyor ama prod'da çalışmıyor" problemini çözer. Config yönetimi kabusu, ortamlar arası fark, ölçekleme zorluğu sorunlarına sistematik cevap. Kubernetes ile doğal uyum: stateless process → HPA, port binding → Service, config env var → ConfigMap/Secret, graceful shutdown → terminationGracePeriodSeconds. Modern microservices geliştirmenin temel checklist'i. Pratikte: her faktör ayrı düşünülmez, birlikte cloud-native deployment hazırlığı sağlar. Mülakatçının gözünden: "12 faktörü biliyor musun?" değil, "uygulamanı nasıl cloud-native yapıyorsun?" sorusunun arka planı.

2. III. faktör (Config) nedir? Secrets nasıl yönetilmeli?

   > **Beklened:** Config: ortama göre değişen her şey (DB URL, API key, feature flag). Kod değişmez, environment değişir. Test: "repo'yu şu an public yapsam güvenli mi?" — hayır → secret kodda. Yanlış: hardcoded URL/password, .properties dosyası commit edilmiş. Doğru — katmanlar: (1) Environment variable: temel, Docker/K8s ile iyi entegrasyon. (2) Kubernetes Secret: base64 encode, etcd'de şifreli. (3) Vault / AWS Secrets Manager: rotation, audit, fine-grained access. Spring ile: `@Value("${DATABASE_URL}")` veya `application.yml`'de `${ENV_VAR}`. Git'e commit edilmişse: şifreyi hemen değiştir, git geçmişini temizle (BFG). .gitignore: `*.properties`, `application-prod.*` ekle.

3. VI. faktör (Stateless Processes) nedir? Sticky session neden anti-pattern?

   > **Beklened:** Stateless: her process işlemi bitince hiçbir şey hafızada bırakmaz. State → backing service (Redis, DB, S3). Sticky session: aynı kullanıcı → hep aynı pod. Neden anti-pattern: pod ölünce session kaybı. Scale-out dengeli değil (bazı pod'lar aşırı yüklü). Kubernetes pod restart → yeni pod ID → sticky session bozuldu. Horizontal scaling → verimli değil. Doğru: Spring Session + Redis — tüm pod'lar aynı Redis'ten session okur. Pod fark etmez, herhangi pod herhangi kullanıcıya hizmet edebilir. Aynı prensip: local file cache (`/tmp`), in-memory singleton state → pod'lar arası paylaşılmaz → Redis, S3, DB'ye taşı.

4. IX. faktör (Disposability) nedir? Graceful shutdown nasıl sağlanır?

   > **Beklened:** Disposability: hızlı başla (fast startup), graceful kapat (mevcut işleri bitir). Neden: Kubernetes rolling deploy → SIGTERM → eski pod kapanıyor. Graceful shutdown yoksa: aktif HTTP istekleri kesilir (502), Kafka mesajları commit edilmez (duplicate). Spring Boot graceful shutdown: `server.shutdown: graceful`, `timeout-per-shutdown-phase: 30s`. Kubernetes: `terminationGracePeriodSeconds: 40` (Spring 30s + buffer). preStop hook: `sleep 5` — pod endpoint listesinden çıkmak için süre. readinessProbe: hazır olmadan trafik alma. Kafka consumer: `@PreDestroy` → `consumer.wakeup()` → son batch'i işle, commitSync → exit. Fast startup: GraalVM native image (<100ms), lazy init, minimal bean yükleme. Ölçekleme: hızlı başlayan pod → HPA tepkisi hızlı.

---

**Senior / Architect:**

5. X. faktör (Dev/Prod Parity) neden kritiktir? Hangi araçlarla sağlanır?

   > **Beklened:** Üç boşluk: Time gap (commit → deploy: haftalar → CI/CD ile günler/saatler). Personnel gap (developer yazar, ops deploy eder → DevOps kültürü). Tools gap (dev: H2/SQLite, prod: PostgreSQL → farklı davranış). Tools gap en tehlikeli: H2 ile test geçti, PostgreSQL'de bug → case sensitivity, constraint davranışı, native query syntax, boolean type farklılığı. Çözüm: Docker Compose → prod ile aynı stack. Dev: `docker-compose up` → PostgreSQL 16, Redis 7, Kafka 7.5. application-dev.yml: prod ile aynı dialect, aynı driver. Testcontainers: entegrasyon testleri için gerçek DB container. CI: aynı Docker Compose → her PR'da gerçek stack ile test. Avantaj: "bende çalışıyor ama prod'da çalışmıyor" neredeyse sıfır. Maliyeti: docker-compose öğrenme, yerel kaynak tüketimi (hafif).

6. XII. faktör (Admin Processes) — DB migration neden startup'ta değil, ayrı Job olarak çalışmalı?

   > **Beklened:** Startup'ta migration sorunu: Rolling deploy sırasında eski + yeni pod aynı anda çalışır. Yeni pod migration çalıştırdı → eski pod yeni şemayı bilmiyor → hata. Birden fazla pod aynı anda migration → Flyway lock (tek yazma garantisi var ama gecikme). Startup süresi uzar → readinessProbe timeout → pod restart loop. Kubernetes Job çözümü: deploy öncesi tek migration Job çalışır. Tamamlandı mı? → rolling deploy başlar. Aynı image → aynı artifact (V. faktör: build/release/run ayrı). `spring.flyway.enabled: false` uygulama profilinde, sadece migration profilinde true. Zero-downtime kuralı: backward-compatible migration (yeni kolon nullable) → iki aşamalı deploy. Uygulama kodu yeni kolonu kullan → sonraki deploy'da NOT NULL constraint ekle. Admin script: `kubectl run` ile one-off task → log, exit code → izlenebilir.

7. V. faktör (Build/Release/Run) CI/CD pipeline'a nasıl yansır? SSH ile prod'da kod değiştirmenin neden tehlikeli?

   > **Beklened:** Build: `git push` → CI → Docker image build → artifact (immutable). Release: Docker image + config (env var, K8s Secret) → release bundle. Her release: benzersiz tag (git SHA). Run: Kubernetes → image + config → pod. SSH ile prod değiştirme neden tehlikeli: (1) Reproducibility yok: "prod'daki kod neydi?" → bilinmiyor. (2) Git geçmişi ile prod arasında fark → "drift." (3) Rollback imkânsız: hangi dosyayı değiştirdim? (4) Audit yok: kim, ne değiştirdi? (5) Diğer pod'larda aynı değişiklik yok → pod'lar birbirinden farklı. CI/CD pipeline: GitOps → git commit = gerçeğin kaynağı. ArgoCD/Flux: git'teki Kubernetes manifest → cluster'a uygula. Rollback: git revert → otomatik deploy. Immutable artifact: image bir kez build → dev, staging, prod aynı image → ortam farkı sadece config'de.

---

## Karma — Architect Seviyesi

8. **"Legacy Java EE uygulamasını twelve-factor uyumlu hale getirmek istiyorsun. Hangisini önce yaparsın?"**

   > **Beklened:** Önce yüksek etki, düşük maliyet adımlar: (1) III. Config — env var: hardcoded config → environment variable. En hızlı kazanım, en az risk. Vault/K8s Secret entegrasyonu. (2) XI. Logs — stdout: dosyaya yazma → stdout. Log4j2/Logback config değişikliği, ELK entegrasyonu. (3) VI. Stateless — session: HTTP session Redis'e. Spring Session entegrasyonu. (4) IX. Disposability — graceful shutdown: SIGTERM handler. Spring Boot: tek config satırı. (5) X. Dev/Prod Parity — Docker Compose: H2'den PostgreSQL'e geçiş, local prod-like ortam. Daha büyük yatırım gerektiren: (6) V. Build/Release/Run: CI/CD pipeline kurulumu. (7) VIII. Concurrency: HPA, autoscaling. (8) XII. Admin: migration Job'a taşıma. Sıralama: güvenlik (config) → observability (log) → scalability (stateless) → reliability (graceful) → dev experience (parity). Hepsini aynı anda yapmaya çalışma — iteratif.

9. **"Ekip 'twelve-factor too much overhead, küçük projede gerekmez' diyor. Hangilerini savunursun, hangilerinden vazgeçersin?"**

   > **Beklened:** Savunulanlar (her boyutta şart): III. Config/Secrets: hardcoded secret → her zaman güvenlik riski. Küçük proje = düşük risk değil. `.env` dosyası bile yeterli, Vault gerekmez. VI. Stateless: sticky session → küçük projede de ölçekleme sorunu çıkar. Redis Session maliyeti düşük, faydası yüksek. XI. Logs stdout: dosyaya yazmak → her ortamda sorun. Stdout tek satır config. IX. Graceful shutdown: deploy sırasında kullanıcı etkisi. Spring Boot: iki satır config. Maliyet neredeyse sıfır. Ertelenebilir (proje büyüyünce): VIII. Concurrency/HPA: tek pod yetiyorsa erken optimizasyon. VII. Port Binding: Spring Boot zaten fat jar → bu "bedava" geliyor. XII. Admin Job: küçük projede startup migration yeterli. Flyway lock sorunu tek pod'da yok. Kısaltılmış 12 faktör → "core 5": Config, Stateless, Logs, Graceful Shutdown, Dev/Prod Parity. Bunlar her projede, geri kalanlar ölçek gerektikçe.
