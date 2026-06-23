# 09f — Container ve Supply Chain Güvenliği

## Container (Docker) Güvenliği

### Ne?
Container imajları ve runtime'larındaki güvenlik açıklarını minimize etmek.
Container, güvenli olmayan bir imajla ayağa kalkarsa tüm ortam risk altında.

### İmaj Güvenliği

#### Distroless / Minimal Base Image

```dockerfile
# YANLIŞ — çok büyük, gereksiz araçlar (bash, curl, wget)
FROM ubuntu:latest
RUN apt-get install -y openjdk-17-jdk
COPY app.jar app.jar
CMD ["java", "-jar", "app.jar"]

# DOĞRU — minimal, sadece JRE
FROM eclipse-temurin:21-jre-alpine

# En iyi — distroless (shell bile yok)
FROM gcr.io/distroless/java21
COPY --chown=nonroot:nonroot app.jar /app/app.jar
USER nonroot
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```
Distroless avantajları:
  Shell yok → saldırgan bash ile komut çalıştıramaz (RCE'yi sınırlar)
  Package manager yok → ek yazılım yüklenemez
  CVE surface: Ubuntu JDK → 300+ CVE, Distroless Java → 5-10 CVE
```

#### Non-Root User

```dockerfile
# Root olarak çalışma!
FROM eclipse-temurin:21-jre-alpine

# Dedicated user oluştur
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Dosya sahipliği
COPY --chown=appuser:appgroup app.jar /app/app.jar
COPY --chown=appuser:appgroup config/ /app/config/

# User geç
USER appuser

WORKDIR /app
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```
Neden önemli?
  Container'dan çıkış (container escape) → host sisteme root erişim!
  Non-root: escape olsa bile kısıtlı yetki
  Kubernetes: runAsNonRoot: true ile enforce et
```

#### Multi-Stage Build

```dockerfile
# Stage 1: Build (büyük imaj, geçici)
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:resolve  # dependency cache için ayrı layer
COPY src/ src/
RUN mvn package -DskipTests

# Stage 2: Runtime (küçük imaj, production)
FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /build/target/app.jar /app/app.jar
USER nobody
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```
Avantajlar:
  Build araçları (Maven, JDK) production imajında yok
  .git, test dosyaları dahil olmuyor
  Kayda değer boyut farkı: 800MB → 150MB
```

---

### Vulnerability Scanning

#### Trivy

```bash
# İmaj tarama
trivy image myapp:latest

# Sonuç:
# myapp:latest (alpine 3.18.4)
# Total: 5 (HIGH: 2, CRITICAL: 0, MEDIUM: 3)
#
# Library     Vulnerability   Severity   Installed Ver  Fixed Ver
# openssl     CVE-2023-xxxx   HIGH       3.1.3          3.1.4
# libz        CVE-2023-xxxx   MEDIUM     1.2.11         1.2.13

# CI'da fail: critical varsa pipeline'ı durdur
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# SBOM üret (CycloneDX formatı)
trivy image --format cyclonedx --output sbom.json myapp:latest
```

#### CI/CD Pipeline Entegrasyonu

```yaml
# GitHub Actions
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1  # CI'ı durdur

      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: trivy-results.sarif
```

---

### Kubernetes Pod Security

#### Security Context

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      # Pod seviyesi
      securityContext:
        runAsNonRoot: true          # root olarak çalışmayı engelle
        runAsUser: 10001            # belirli UID
        runAsGroup: 10001
        fsGroup: 10001              # volume dosya sahipliği
        seccompProfile:
          type: RuntimeDefault      # syscall filtreleme

      containers:
        - name: myapp
          image: myapp:1.0.0
          # Container seviyesi
          securityContext:
            allowPrivilegeEscalation: false  # sudo/setuid yasak
            readOnlyRootFilesystem: true      # dosya sistemi read-only
            capabilities:
              drop: [ALL]           # tüm Linux capabilities kaldır
              add: [NET_BIND_SERVICE]  # sadece gereken (port 80 bind)
          
          # Resource limits (kaynak tüketim saldırısı önlemi)
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          # Secrets: environment değişken değil, volume mount
          volumeMounts:
            - name: db-secret
              mountPath: /secrets/db
              readOnly: true

      volumes:
        - name: db-secret
          secret:
            secretName: db-credentials
```

#### Network Policy ile İzolasyon

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-default
  namespace: production
spec:
  podSelector: {}  # tüm pod'lar
  policyTypes:
    - Ingress
    - Egress
  # ingress/egress yoksa → tümü reddet (whitelist yaklaşımı)
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
```

---

## Supply Chain Güvenliği

### Ne?
Uygulamanın bağımlılıkları (npm, Maven, PyPI paketleri) üzerinden gerçekleşen saldırılar.
SolarWinds, Log4Shell, event-stream (npm) gibi vakalar.

### Bağımlılık Güvenliği

#### SBOM (Software Bill of Materials)

```
Uygulamada kullanılan tüm bileşenlerin envanteri:
  - Bileşen adı ve sürümü
  - Lisans bilgisi
  - Kaynaklar (kayıt defteri URL)
  - Bilinen CVE'ler

Formatlar:
  CycloneDX: OWASP destekli, JSON/XML
  SPDX: Linux Foundation, geniş ekosistem desteği

Neden?
  Log4Shell: hangi servislerimde log4j var? SBOM varsa → dakikalar
  Lisans uyumu: copyleft (GPL) lisanslı kütüphane → ticari üründe sorun
  Tedarik zinciri: hangi bağımlılık hangi sürüm → güvenlik auditi
```

```xml
<!-- Maven — SBOM üretme (CycloneDX Maven Plugin) -->
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.7.11</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>makeAggregateBom</goal>
            </goals>
        </execution>
    </executions>
</plugin>
<!-- target/bom.json → SBOM dosyası -->
```

#### Bağımlılık Güvenlik Taraması

```yaml
# GitHub Dependabot — otomatik güvenlik güncellemeleri
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    # Security update PR'larını otomatik aç

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

```bash
# OWASP Dependency Check (Maven plugin)
mvn org.owasp:dependency-check-maven:check \
    -DfailBuildOnCVSS=7  # CVSS ≥ 7 → build fail

# Snyk
snyk test --severity-threshold=high

# npm audit (Node.js)
npm audit --audit-level=high
```

### Güvenli Bağımlılık Yönetimi

```
1. Sürüm pinning: exact version (1.2.3 değil, ~1.2 veya ^1 değil)
   pom.xml: <version>3.2.1</version>  (not: [3.2,) gibi range kullanma)
   package.json: "lodash": "4.17.21"  (not: "^4.0.0")

2. Bütünlük kontrolü:
   Maven: SHA-256 checksum (merkezi deposu kontrol eder)
   npm: package-lock.json (integrity alanı: sha512-...)

3. Private registry:
   JFrog Artifactory / Nexus: tüm bağımlılıklar kendi repo'ndan çekilir
   Upstream proxy: public → önce tarama → private repo'ya kopyala
   
4. Typosquatting önleme:
   "lodash" değil "l0dash" — yanlış yazım → zararlı paket
   npm namespacing: @mycompany/utils (scoped package)
   Internal packages → private registry

5. Transitive dependencies:
   mvn dependency:tree | grep log4j → tüm log4j bağımlılıkları
   Direkt bağımlılık değil, transitive'de güvenlik açığı çıkabilir
```

### Image Signing (Cosign / Sigstore)

```bash
# Sigstore Cosign ile imaj imzalama
# Build sonrası:
cosign sign --key cosign.key myregistry.io/myapp:1.0.0

# Deploy öncesi doğrula:
cosign verify --key cosign.pub myregistry.io/myapp:1.0.0

# Keyless signing (OIDC ile — Sigstore'un güçlü yönü)
cosign sign myregistry.io/myapp:1.0.0
# → GitHub Actions OIDC token ile imzalar → Fulcio CA sertifika verir → Rekor'a loglanır
```

```yaml
# Kubernetes Admission Controller (Kyverno) — imzasız imaj reddet
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
                      -----END PUBLIC KEY-----
```

---

## Secrets Yönetimi (Container Ortamında)

```
YANLIŞ: Dockerfile'da secret
  ENV DB_PASSWORD=mypassword   ← imaj layerına gömülür, imaj history'de görünür!
  
YANLIŞ: K8s Secret Base64 encode (şifreli değil!)
  kubectl create secret generic db-creds --from-literal=password=mypass
  kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d → şifre!
  → etcd encryption at rest aktif edilmeli

DOĞRU seçenekler:
  1. HashiCorp Vault + Vault Agent Injector
     Vault Agent sidecar → uygulama başlamadan secret çek → dosyaya yaz
     Uygulama /secrets/db'yi okur → env variable değil
  
  2. External Secrets Operator
     K8s → AWS Secrets Manager / GCP Secret Manager'dan çek
     K8s Secret'a sync → pod mount eder
  
  3. Sealed Secrets (Bitnami)
     SealedSecret (şifreli) → Git'e commit OK
     Controller → cluster key ile decrypt → K8s Secret oluşturur
```

---

## Trade-off Özeti

| Konu | Önerilen | Neden |
|------|----------|-------|
| Base image | Distroless / Alpine | CVE surface azaltır |
| Container user | Non-root (UID 10001) | Privilege escalation önler |
| Scanning | Trivy + CI entegrasyonu | CRITICAL → pipeline fail |
| K8s Pod | readOnlyRootFilesystem + capabilities drop ALL | Runtime saldırı yüzeyi azaltır |
| Secret | Vault / External Secrets | Env var veya dockerfile'a koyma |
| SBOM | CycloneDX — her build | Log4Shell gibi vakalarda hızlı yanıt |
| Image signing | Cosign keyless (Sigstore) | Tedarik zinciri doğrulama |
| Dependency | Dependabot + OWASP check | Otomatik güncelleme + CVE tarama |

| Araç | Amaç |
|------|------|
| Trivy | Image + IaC + SBOM tarama |
| Cosign/Sigstore | Image imzalama |
| OPA/Kyverno | Policy enforcement (K8s) |
| Vault | Secrets management |
| Dependabot | Otomatik bağımlılık güncellemesi |
| OWASP Dep Check | CVE bazlı bağımlılık taraması |
| Falco | Runtime container anomaly detection |
