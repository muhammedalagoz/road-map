# 09d — Zero Trust Mimarisi ve Yetkilendirme Modelleri

## Zero Trust Architecture

### Ne?
"Ağ içindekilere güven, dışarıdakilere güvenme" yaklaşımının tam tersi.
**"Never trust, always verify"** — konum (iç ağ) değil, kimlik esasıyla güven.

### Neden?

```
Geleneksel yaklaşım (Perimeter Security):
  Firewall → İçerideysen → Güvenilirsin
  
  Problem:
  VPN ile içeri giren saldırgan → tüm iç servislere erişir
  Ele geçirilmiş laptop → iç ağda → lateral movement
  Insider threat (çalışan ihlali)
  
  2020 SolarWinds: VPN üzerinden iç ağa girdi → aylarca fark edilmedi

Zero Trust:
  İç ağ = dış ağ kadar tehlikeli
  Her istek → kimlik doğrula
  Her erişim → yetki kontrolü
  Her şeyi logla
```

### Zero Trust Prensipleri

```
1. Verify Explicitly
   Kimliği her seferinde doğrula (token, sertifika, MFA)
   Context'e bak: cihaz sağlığı, konum, zaman, davranış
   "Bu kişi bu saatte, bu cihazdan bu kaynağa erişmeli mi?"

2. Use Least Privilege Access
   Minimum gerekli yetki (sadece işini yapabilecek kadar)
   JIT (Just-in-Time) access: ihtiyaç anında geçici yetki
   JEA (Just-Enough-Access): sadece gereken kaynak

3. Assume Breach
   İhlal oldu varsay → lateral movement'ı sınırla
   Mikro segmentasyon: her servis izole
   Şifreli iletişim: iç trafik de TLS (mTLS)
   Detect & respond: SIEM, anomaly detection
```

### Zero Trust Bileşenleri

```
Identity Provider (IdP):
  Tüm kimlik kararları merkezi → Okta, Azure AD, Keycloak
  MFA zorunlu
  Cihaz kaydı: yönetilen cihaz → daha fazla güven

Service-to-Service Authentication (mTLS):
  Her mikro servis → kendi sertifikası
  İstekler → karşılıklı sertifika doğrulama
  SPIFFE/SPIRE: servis kimliği standartı

Policy Engine:
  "Servis A, Servis B'ye bu action için erişebilir mi?"
  Open Policy Agent (OPA) → policy as code
  Istio + OPA → service mesh seviyesinde kararlar

Network Segmentation:
  Her servis kendi namespace'inde (Kubernetes)
  NetworkPolicy: sadece izinli servisler iletişim kurar
  Servis A → Servis C'ye direkt erişemez (sadece B üzerinden)
```

### SPIFFE/SPIRE (Service Identity)

```
SPIFFE: Secure Production Identity Framework for Everyone
  Her workload (pod, servis) → unique kimlik
  SVID (SPIFFE Verifiable Identity Document) → X.509 sertifikası
  Format: spiffe://trust-domain/path/to/workload

SPIRE: SPIFFE implementasyonu
  Agent (her node'da) → workload'un kimliğini attestation ile doğrular
  Server → sertifika imzalar ve dağıtır
  Kısa ömürlü sertifikalar (SVID: 1 saat) → sızdırılsa bile kısa süre geçerli

Akış:
  Pod başlatılır → SPIRE Agent attestation yapar (K8s SA, hostname, vb)
  SVID alır → mTLS bağlantılarında kullanır
  Sertifika expire → SPIRE otomatik yeniler
  Merkezi secret yönetimi gerekmez (secret'siz kimlik doğrulama)
```

```yaml
# Kubernetes NetworkPolicy (Zero Trust örneği)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway  # sadece API Gateway'den
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: payment-service  # sadece payment-service'e
      ports:
        - port: 8081
    - to:  # DNS
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
```

---

## Yetkilendirme Modelleri

### RBAC (Role-Based Access Control)

#### Ne?
Kullanıcı → Role → Permission. Rol grupları üzerinden yetki.

```
User → [Role: ADMIN, EDITOR] → [Permission: user:delete, content:write, content:read]

Avantaj:
  Basit, anlaşılır
  Rol değişince yetki değişir (tek noktadan)
  Audit kolay: "ADMIN rolü ne yapabilir?"

Dezavantaj:
  Role explosion: Çok fine-grained ihtiyaç → yüzlerce rol
  Context duyarsız: "Ali ADMIN'dir" → her durumda
  "Ali sadece kendi departmanının raporlarını okuyabilir" → RBAC ile zor
```

```java
// Spring Security RBAC
@PreAuthorize("hasRole('ADMIN')")
void deleteUser(Long userId) { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
void banUser(Long userId) { ... }

// URL bazlı RBAC
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/reports/**").hasAnyRole("ADMIN", "ANALYST")
    .requestMatchers("/api/users/**").authenticated()
    .anyRequest().permitAll()
);
```

### ABAC (Attribute-Based Access Control)

#### Ne?
Subject attributes + Resource attributes + Environment attributes → Policy kararı.
"Kim, neyi, ne zaman, nasıl yapabilir?"

```
Policy: 
  IF user.department == resource.department
  AND user.clearanceLevel >= resource.sensitivityLevel
  AND environment.time BETWEEN 09:00-18:00
  AND user.location == "OFFICE"
  THEN ALLOW

Avantaj:
  Çok ince taneli kontrol
  Context-aware (zaman, konum, cihaz)
  Esneklik: yeni attribute → policy güncelle (yeni rol gerekmiyor)

Dezavantaj:
  Karmaşık policy yönetimi
  Performans: her request'te policy evaluation
  Debug zor: "Neden erişim reddedildi?" — hangi attribute?
```

#### Open Policy Agent (OPA)

```
Policy as Code: Rego dili ile policy yaz, Git'te sakla
Sidecar veya standalone servis
gRPC/HTTP üzerinden policy evaluation
Kubernetes admission control, API authorization, Kafka ACL

Örnek Rego policy:
```

```rego
# OPA policy (Rego)
package myapp.authz

import future.keywords.if

default allow := false

# Admin her şeyi yapabilir
allow if {
    input.user.roles[_] == "admin"
}

# Kullanıcı sadece kendi profilini güncelleyebilir
allow if {
    input.method == "PUT"
    input.path == ["users", user_id]
    input.user.id == user_id
    input.user.roles[_] == "user"
}

# Business saatlerinde (09-18) sadece ofisten erişim
allow if {
    input.user.roles[_] == "employee"
    input.resource.sensitivity == "internal"
    time.clock(input.timestamp)[0] >= 9
    time.clock(input.timestamp)[0] < 18
    input.user.location == "office"
}
```

```java
// OPA'ya policy sorgusu (Spring/Java)
@Service
class AuthorizationService {

    @Autowired OpaClient opaClient;

    boolean isAllowed(String userId, String resource, String action) {
        OpaQuery query = OpaQuery.builder()
            .user(userService.getUser(userId))
            .resource(resource)
            .action(action)
            .environment(Map.of(
                "timestamp", Instant.now().toString(),
                "ipAddress", getRequestIp()
            ))
            .build();

        OpaResult result = opaClient.evaluate("myapp/authz/allow", query);
        return result.isAllowed();
    }
}
```

### ReBAC (Relationship-Based Access Control)

```
Google Zanzibar sistemi (Google Drive izinleri)
"Bu kullanıcı bu belge üzerinde hangi ilişkiye sahip?"

Relationship tuple:
  document:abc#owner@user:alice
  document:abc#viewer@group:engineering
  folder:xyz#parent@document:abc

Sorgu: "Alice, document:abc'yi okuyabilir mi?"
  → Alice, document'ın owner'ı mı?
  → Alice, viewer grubunun üyesi mi?
  → document, viewer izni olan bir klasörün içinde mi?

Uygulamalar:
  Google Drive, GitHub (repo izinleri), Discord
  OpenFGA (open source Zanzibar)
  Auth0 FGA

Avantaj: Hiyerarşik izinler doğal (klasör → dosya)
Dezavantaj: Karmaşık, çok büyük ölçek gerektirir
```

---

## Hangi Model Ne Zaman?

| Senaryo | Model | Neden |
|---------|-------|-------|
| Basit kurumsal uygulama | RBAC | Yönetimi kolay, yaygın anlayış |
| Çok tenant SaaS, fine-grained | ABAC + OPA | Tenant izolasyonu, context-aware |
| Belge/klasör hiyerarşisi | ReBAC (OpenFGA) | Google Drive benzeri |
| Mikro servis arası | RBAC + mTLS + SPIFFE | Servis kimliği + yetki |
| Regulated industry | RBAC + ABAC (hybrid) | Compliance + esneklik |
| DevOps / CI/CD | OPA (Admission Control) | Policy as Code |

---

## Trade-off Özeti

| Özellik | RBAC | ABAC | ReBAC |
|---------|------|------|-------|
| Karmaşıklık | Düşük | Yüksek | Orta |
| Esneklik | Orta | Çok yüksek | Yüksek (ilişki bazlı) |
| Performans | Hızlı | Yavaş (policy eval) | Orta (graph traversal) |
| Yönetim | Rol yönetimi | Policy yönetimi | İlişki yönetimi |
| Uygun ölçek | Küçük-orta | Büyük enterprise | Büyük, hiyerarşik |
| Araçlar | Spring Security, Keycloak | OPA, Casbin | OpenFGA, Zanzibar |

| Zero Trust Seviyesi | Ne Gerekiyor | Maliyet |
|--------------------|--------------|---------|
| L1 Temel | MFA, HTTPS everywhere | Düşük |
| L2 Kimlik bazlı | IdP, JWT, mTLS (servisler arası) | Orta |
| L3 İleri | SPIFFE, OPA, Network Policy | Yüksek |
| L4 Tam ZT | Continuous verification, behavioural | Çok yüksek |
