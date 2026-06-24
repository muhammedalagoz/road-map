# 09d — Zero Trust & Yetkilendirme Modelleri: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Zero Trust Mimarisi

### Gerçek Hayat Sorunları

---

**Sorun 1: Perimeter güvenliği — VPN içine giren saldırgan aylarca lateral movement yaptı**

```
Senaryo (SolarWinds 2020 benzeri):
  Şirket: "Firewall + VPN = güvenliyiz" yaklaşımı.
  İç ağda: tüm servisler birbirini güveniyor, auth yok.
  "İç ağdaysan zaten güvenilirsin."

  Saldırı zinciri:
    1. Phishing email → çalışanın VPN credential'ı çalındı.
    2. Saldırgan VPN'e bağlandı → iç ağa girdi.
    3. İç ağda: servisler arası auth yok → tüm API'ler açık.
       curl http://payment-service:8080/admin/export → 200 OK.
       curl http://user-service:8080/users?size=all → 200 OK.
    4. DB server: iç ağdan bağlantı → şifre bile yok (IP whitelist).
    5. 4 ay boyunca: veri sızdırma, keşif, persistence.
    6. Tespit: yavaş ağ trafiği anomalisi → SIEM alert.

  Zarar: 18,000+ şirketin sistem erişimi, ABD devlet verileri.

Zero Trust ile fark:
  VPN girse bile:
    Servisler: her isteği → JWT / mTLS doğruluyor.
    Ağ seviyesi: NetworkPolicy → payment-service sadece api-gateway'den istek kabul ediyor.
    DB: VPN IP'den bağlantı değil → vault credential + service account.
    Davranış: SIEM → "bu kullanıcı 03:00'da bulk export yaptı" → alert.

  Assume Breach prensibi:
    "İhlal oldu varsay" → lateral movement sınırlanır.
    Bir servis/kullanıcı compromise → tüm sistem değil.
    Mikro segmentasyon: her servis izole → yatay hareket zor.

  Zero Trust bileşenleri uygulaması:
    1. Kimlik: VPN değil, her kullanıcı/servis için ayrı kimlik.
       Kullanıcı: MFA + device health check.
       Servis: SPIFFE SVID sertifikası.
    2. mTLS: servisler arası her çağrı → karşılıklı sertifika.
    3. OPA policy: "kim, neye, ne zaman erişebilir?" → merkezi kontrol.
    4. Network Policy: sadece izinli servis çiftleri iletişim kurar.
    5. Audit: her erişim loglanır → SIEM anomaly detection.
```

---

**Sorun 2: SPIFFE/SPIRE eksikliği — servisler arası kimlik doğrulama yok, shared secret sızdı**

```
Senaryo:
  Microservice mimarisi: 30+ servis.
  Servisler arası auth: paylaşılan API key.

  application.properties:
    internal.api.secret=Shared-Secret-2024

  Tüm servislerde aynı secret → tüm servislere paylaşıldı.

  Sorunlar:
    1. Secret rotation: 30 serviste aynı anda güncelleme → koordinasyon cehennemi.
       Bir servis güncellenmeden önce deploy edilirse → iletişim kesilir.
    2. Granülarite: hangi servis hangi servise erişebilir? Hepsi herkese.
       payment-service → analytics-service'e erişmemeli ama aynı secret var.
    3. Secret sızıntısı: log'a düştü → herkesin erişimi.
       Saldırgan: log sunucusuna erişti → shared secret → tüm servislere erişim.
    4. Audit: "bu isteği hangi servis attı?" → bilinmiyor (hepsi aynı secret).

SPIFFE/SPIRE ile çözüm:
  Her pod'a unique kimlik: spiffe://mycompany.com/ns/prod/sa/payment-service

  SPIRE mimarisi:
    SPIRE Server (merkezi): sertifika imzalar.
    SPIRE Agent (her node'da DaemonSet): workload attestation.
    Attestation: K8s service account + pod bilgisi → kimlik kanıtlama.

  Pod başlangıcı:
    payment-service pod → SPIRE Agent'a: "Ben payment-service'im."
    SPIRE Agent: K8s API ile doğrula → gerçekten payment-service pod'u mu?
    Doğrulandı → SVID (X.509 sertifikası) verildi:
    Subject: spiffe://mycompany.com/ns/prod/sa/payment-service
    TTL: 1 saat (kısa ömürlü → sızdırılsa bile kısa süre geçerli).

  mTLS bağlantısı:
    payment-service → order-service:
    TLS handshake: payment SVID sunar, order SVID sunar → karşılıklı doğrulama.
    Sertifika expire → SPIRE otomatik yeniler (uygulama bilmez).

  Avantajlar:
    Secret yok → rotation sorunu yok.
    Her servis farklı kimlik → granüler erişim kontrolü.
    "Bu isteği kim attı?" → sertifikaya bak → payment-service.
    OPA policy: spiffe://*/payment-service → sadece order-service çağırabilir.
```

---

**Sorun 3: NetworkPolicy yok + "iç ağ güvenli" varsayımı — compromised pod tüm cluster'ı taradı**

```
Senaryo:
  Kubernetes cluster: NetworkPolicy tanımlı değil.
  Varsayılan K8s davranışı: tüm pod'lar birbirine erişebilir.

  frontend-pod compromise edildi (XSS + RCE zinciri).
  Saldırgan pod içinden:
    kubectl exec değil, curl:
    curl http://payment-service.prod.svc:8080/health → 200 ✓ (erişilebilir!)
    curl http://payment-service.prod.svc:8080/admin/transactions → 200 ✓
    curl http://postgres.prod.svc:5432 → bağlandı!
    curl http://redis.prod.svc:6379 → session store → tüm session'lar.
    curl http://vault.internal:8200/v1/secret → Vault'a erişim!

  "frontend sadece BFF'e çağrı atmalı" → kod seviyesinde, ağ seviyesinde değil.
  Çalışma zamanı saldırı → kod değişmiyor → ağ erişimi mümkün.

  Lateral movement: frontend → payment → DB → Vault → tüm sistem.

Düzeltme (Zero Trust Network Policy):
  # Adım 1: Tüm trafiği reddet (deny-all)
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: deny-all
    namespace: production
  spec:
    podSelector: {}          # tüm pod'lar
    policyTypes: [Ingress, Egress]
    # ingress/egress kuralı yok → her şey reddedildi

  # Adım 2: Sadece izinli trafik whitelist
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: order-service-policy
  spec:
    podSelector:
      matchLabels:
        app: order-service
    policyTypes: [Ingress, Egress]
    ingress:
      - from:
          - podSelector:
              matchLabels:
                app: api-gateway   # sadece API GW'den
        ports:
          - port: 8080
    egress:
      - to:
          - podSelector:
              matchLabels:
                app: payment-service  # sadece payment'a
        ports:
          - port: 8081
      - to: []           # DNS
        ports:
          - port: 53
            protocol: UDP

  Sonuç: frontend compromise → payment-service'e erişim engellendi.
  Assume Breach: bir pod compromise → diğerleri izole.
  Istio ile ek katman: mTLS + AuthorizationPolicy → "kim neyi çağırabilir?"
```

---

## Bölüm 2: Yetkilendirme Modelleri

### Gerçek Hayat Sorunları

---

**Sorun 4: Role explosion — RBAC 500+ role büyüdü, yönetilemez hale geldi**

```
Senaryo:
  Kurumsal SaaS: başlangıçta basit roller.
  ADMIN, USER, VIEWER → işe yarıyordu.

  Zaman içinde ihtiyaçlar:
    "Muhasebe sadece kendi raporlarını görsün."
    "İstanbul ofis yöneticisi sadece İstanbul çalışanlarını yönetsin."
    "Proje A geliştiricisi sadece Proje A kaynaklarına erişsin."
    "Finans direktörü tüm raporları görsün ama sadece okusun."
    "Müşteri destek: ticket'ları yönetsin ama kullanıcı verisini değiştirmesin."

  Çözüm: yeni rol oluştur!
    ACCOUNTING_VIEWER_ISTANBUL
    ACCOUNTING_VIEWER_ANKARA
    ACCOUNTING_EDITOR_ISTANBUL
    ...
    PROJECT_A_DEV_READ
    PROJECT_A_DEV_WRITE
    ...
    CUSTOMER_SUPPORT_TICKET_ONLY
    CUSTOMER_SUPPORT_FULL
    ...
    500+ rol → kimse ne anlamına geldiğini bilmiyor.
    "Bu kullanıcı hangi rolde?" → 12 rol atanmış.
    Yeni çalışan: hangi roller lazım? → kimse bilmiyor → "hepsini ver."
    Audit: "Kim ne yapabilir?" → sorulamaz hale geldi.
    Güvenlik açığı: eski roller temizlenmiyor → ayrılan çalışan erişimde.

ABAC ile çözüm:
  Roller yerine attribute'lar:
    user.department = "accounting"
    user.location   = "istanbul"
    user.level      = "viewer"

  Tek policy:
    ALLOW IF user.department == resource.department
         AND user.location   == resource.location
         AND action IN user.allowedActions[resource.type]

  Yeni ihtiyaç (ankara muhasebe): attribute değişir → yeni rol oluşturmak yok.
  Role sayısı: 500 → 10 (temel roller + attribute).

  Hibrit yaklaşım (gerçekçi):
    Kaba taneli: RBAC (ADMIN, EMPLOYEE, VIEWER).
    İnce taneli: ABAC (hangi departman, hangi lokasyon, hangi proje).
    OPA: RBAC + ABAC birlikte → tek policy engine.
```

---

**Sorun 5: ABAC context eksikliği — mesai dışı erişim ve şüpheli konumdan erişim engellenmedi**

```
Senaryo:
  Çalışan hesabı credential'ı çalındı.
  Saldırgan: Ukrayna'dan (şirket Türkiye merkezli), gece 03:00'da erişti.
  Sistem: kimlik doğrulandı → erişim verildi.

  RBAC'ta: rol check → "EMPLOYEE rolü var → izin ver."
  Context yok: ne zaman, nereden, hangi cihazdan → fark etmiyor.

  Çalışan: her zaman 09:00-18:00 arasında, Türkiye'den, şirket cihazından erişiyor.
  Bu anomali → tespit edilmedi → bulk data export → veri sızdı.

ABAC + context-aware policy:
  # OPA Rego policy
  package myapp.authz

  default allow := false

  allow if {
      input.user.roles[_] == "employee"
      
      # Zaman kontrolü (mesai saatleri: 09:00-18:00 UTC+3)
      hour := time.clock([input.now, "Europe/Istanbul"])[0]
      hour >= 9
      hour < 18
      
      # Konum kontrolü (Türkiye veya kayıtlı VPN)
      input.context.country in {"TR", "VPNTR"}
      
      # Cihaz kontrolü (yönetilen cihaz)
      input.context.deviceManaged == true
      
      # Hassas kaynak için ek kontrol
      input.resource.sensitivity != "critical"
  }

  # Admin: her yerden ama MFA zorunlu
  allow if {
      input.user.roles[_] == "admin"
      input.context.mfaVerified == true
  }

  Spring entegrasyonu:
    boolean allowed = opaClient.evaluate("myapp/authz/allow", OpaQuery.builder()
        .user(currentUser())
        .resource(resource)
        .context(Map.of(
            "country", geoIpService.getCountry(request.getRemoteAddr()),
            "deviceManaged", deviceService.isManaged(request),
            "mfaVerified", session.getMfaStatus(),
            "now", Instant.now().toString()
        ))
        .build()).isAllowed();

  Adaptive auth:
    Anormal: Ukrayna + gece 03:00 → MFA challenge zorla.
    Bilinmeyen cihaz → ek doğrulama.
    Velocity: son 1 saatte 5 farklı ülkeden → hesap geçici kilitle.
    SIEM alert: güvenlik ekibine bildir.
```

---

**Sorun 6: ReBAC eksikliği — klasör izni dosyalara yayılmadı, paylaşım tutarsız**

```
Senaryo:
  Döküman yönetim sistemi: klasör/dosya hiyerarşisi.
  RBAC ile: USER, EDITOR, VIEWER rolleri.

  Kullanıcı Ali: "Engineering" klasörünü paylaştı (VIEWER yetkisiyle).
  "Engineering" klasörü içinde:
    ├── Designs/
    │   ├── Architecture.pdf
    │   └── Database.pdf
    └── Specs/
        └── API-Spec.docx

  Beklenen: Ali Engineering'i VIEWER ile paylaştı → alt klasör ve dosyalar da VIEWER.
  Gerçek (RBAC ile): sadece "Engineering" klasörüne VIEWER kaydı. Alt öğeler: kontrol edilmiyor.
    Architecture.pdf → kimde hangi izin? → DB'de kayıt yok → erişim engellendi (404).
    Kullanıcı: "Ali paylaştı ama erişemiyorum" → destek talebi.

  Ters sorun: Designs klasörü ayrı sahipten → Engineering'e dahil edildi.
    Designs viewer'ları → Engineering viewer'larına görünüyor olmamalıydı.
    İzin sızması: A grubunun izni B grubuna geçti.

ReBAC (OpenFGA / Zanzibar) ile çözüm:
  Relationship tuple'ları:
    folder:engineering#viewer@user:mehmet  (Mehmet, engineering'i görebilir)
    folder:designs#parent@folder:engineering  (designs, engineering'in alt klasörü)
    document:arch#parent@folder:designs  (arch.pdf, designs'ın içinde)

  Sorgu: "Mehmet, arch.pdf'i okuyabilir mi?"
    → arch.pdf parent: designs.
    → designs parent: engineering.
    → Mehmet, engineering#viewer → evet!
    → İzin hiyerarşik olarak yayıldı.

  Doğrudan override:
    document:arch#viewer@user:ali  (Ali direkt görüntüleyici)
    → Ali, engineering'de VIEWER olmasa bile arch.pdf'i görebilir.

  OpenFGA implementasyonu:
    fga.check({
      user: "user:mehmet",
      relation: "viewer",
      object: "document:arch-pdf"
    }) → { allowed: true }
    // Hiyerarşiyi traverse etti, gerçek zamanlı.

  Avantajlar:
    Hiyerarşik izin yayılımı: doğal ve tutarlı.
    "Neden erişebiliyor?" → ilişki zinciri: mehmet → engineering viewer → designs parent → arch.pdf.
    Google Drive, GitHub, Discord izin modeli → ReBAC.
    RBAC ile: N×M izin satırı. ReBAC: ilişki tuple'ları → daha az satır.
```

---

**Sorun 7: Least Privilege ihlali — servis hesabı tüm DB yetkisine sahipti**

```
Senaryo:
  order-service: kendi tablosuna CRUD yapıyor.
  DB kullanıcısı: uygulama kolaylığı için → tam yetkili (SUPERUSER veya tüm DB'ye).

  DB kullanıcısı: app_user → GRANT ALL ON *.* → tüm DB'ye tam yetki.

  order-service compromise edildi (RCE):
    Saldırgan DB'ye bağlandı → SELECT * FROM users → tüm kullanıcı verisi.
    SELECT * FROM payments → tüm ödeme geçmişi.
    DROP TABLE orders → veri yok edildi.
    CREATE USER backdoor IDENTIFIED BY '...' → kalıcı erişim.

  Gerçek zararı büyüten: "sadece order-service compromise, sadece orders tablosu risk altında" değil.
  Tüm DB: kullanıcılar, ödemeler, adminler → hepsi erişilebilir.

Least Privilege DB kurulumu:
  # Her servis için ayrı DB kullanıcısı ve minimum yetki:

  -- order-service: sadece kendi tablosunda CRUD
  CREATE USER 'order_svc'@'%' IDENTIFIED BY '...';
  GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.orders TO 'order_svc'@'%';
  GRANT SELECT ON app_db.products TO 'order_svc'@'%';  -- ürün okuma
  -- users, payments, admin tablolarına: YETKİ YOK

  -- payment-service: sadece payments tablosu
  CREATE USER 'payment_svc'@'%' IDENTIFIED BY '...';
  GRANT SELECT, INSERT, UPDATE ON app_db.payments TO 'payment_svc'@'%';
  -- orders, users, admin: YETKİ YOK

  Vault Dynamic Secret ile birleştir:
    app_user statik değil → her deploy için Vault otomatik oluşturur.
    TTL: 2 saat → expire → otomatik revoke.
    Compromise: sadece bu instance'ın 2 saatlik credential → zarar sınırlı.

  JIT (Just-in-Time) admin erişimi:
    DBA: normalde DB'ye erişimi yok.
    İhtiyaç: Vault'a istek → "10 dakika için admin erişimi" → onay → geçici credential.
    10 dakika sonra: otomatik revoke → log: kim, ne zaman, ne yaptı.
    Standing privilege yok → insider threat azalır.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Zero Trust mimarisinin 3 temel prensibi nedir? "Never trust, always verify" ne anlama gelir?

   > **Beklened:** Üç prensip: (1) Verify Explicitly: her isteği doğrula — konuma (iç ağ) değil, kimliğe güven. Bağlam: cihaz sağlığı, konum, zaman, davranış. "Bu kişi bu saatte bu cihazdan bu kaynağa erişmeli mi?" (2) Use Least Privilege: minimum yetki. JIT (Just-in-Time): ihtiyaç anında, geçici. JEA (Just-Enough-Access): sadece gereken kaynak. Standing privilege → kaldır. (3) Assume Breach: ihlal oldu varsay. Lateral movement'ı sınırla. Mikro segmentasyon, şifreli iç trafik (mTLS), SIEM. "Never trust, always verify" vs geleneksel: Geleneksel perimeter — firewall içindeysen güvenilirsin. Zero Trust — iç ağ = dış ağ kadar tehlikeli. VPN içine giren saldırgan (SolarWinds 2020) → 4 ay fark edilmedi. Zero Trust: VPN içine girse bile her servis kimlik doğruluyor, her erişim loglanıyor, NetworkPolicy ile lateral movement engellendi.

2. RBAC, ABAC ve ReBAC arasındaki fark nedir? Hangi senaryo hangisini gerektirir?

   > **Beklened:** RBAC (Role-Based): kullanıcı → rol → izin. Basit, anlaşılır. "Admin tüm kullanıcıları silebilir." Dezavantaj: role explosion, context-duyarsız. Küçük-orta kurumsal uygulama için ideal. ABAC (Attribute-Based): subject + resource + environment attribute'larına göre policy. "Kullanıcı departmanı == kaynak departmanı AND mesai saatlerinde AND ofisten." Context-aware. Çok ince taneli. OPA/Casbin ile. Büyük enterprise, multi-tenant SaaS. ReBAC (Relationship-Based): Google Zanzibar. "Bu kullanıcı bu kaynak ile hangi ilişkide?" Hiyerarşik. document:abc#viewer@user:ali. Klasör izni → alt dosyalara otomatik yayılım. Google Drive, GitHub, Discord. OpenFGA. Senaryo seçimi: "Admin paneli" → RBAC yeterli. "Kullanıcı sadece kendi departman raporlarını görür, ofis saatlerinde" → ABAC. "Klasör paylaştım, alt dosyalar da görünsün" → ReBAC. Hibrit: kaba taneli RBAC + ince taneli ABAC sık kullanılır.

3. Open Policy Agent (OPA) nedir? Neden "policy as code" önemlidir?

   > **Beklened:** OPA: bağımsız policy engine. Rego dili ile policy yaz. HTTP/gRPC API üzerinden "bu izin verilmeli mi?" sorgusu. Uygulama kodu değil, policy kodu → Rego. Policy as code önemi: (1) Git'te sakla: policy değişikliği → PR → review → history. "Kim, ne zaman, neden bu izni değiştirdi?" (2) Test edilebilir: `opa test policies/` → unit test. "Bu policy bu input için doğru karar mı?" (3) Bağımsız: uygulama dilinden bağımsız. Java, Python, Go — hepsi aynı OPA'yı sorgular. Tutarlı policy. (4) Hot reload: uygulama restart olmadan policy güncelle. (5) Merkezi: 30 microservice aynı OPA cluster'ı → tek policy, consistent. Örnek: `allow if { input.user.roles[_] == "admin" }` → tüm servisler aynı admin kuralını kullanıyor. Değişince tek noktadan → tüm servislere yayılır. Kubernetes admission control: OPA Gatekeeper → Pod spec'leri policy'ye aykırıysa → reddedilir.

4. Least Privilege prensibi pratikte nasıl uygulanır?

   > **Beklened:** DB: her servis için ayrı DB kullanıcısı, sadece kendi tablosuna yetki. `GRANT SELECT, INSERT ON orders TO order_svc` — tüm DB'ye GRANT ALL değil. Servis hesabı: K8s Service Account → sadece kendi namespace kaynakları. `serviceAccountName: order-service` → ClusterAdmin değil. API yetki: JWT scopes → `orders:read orders:write` — admin scope değil. Uygulama kullanıcı: non-root (UID 10001) — container root değil. K8s: `runAsNonRoot: true`. Admin erişimi: JIT (Just-in-Time) → Vault PAM. Normalde erişim yok. İhtiyaç: 10 dakika geçici erişim → onay → log. Vault dynamic secret: servis DB credential'ı statik değil, TTL'li → expire → revoke. "En az ayrıcalık" ihlalinin bedeli: order-service compromise → tüm DB erişimi (sadece orders değil). Least privilege: hasar sınırlı — sadece orders tablosu. Saldırganın lateral movement kapasitesi kısıtlandı.

---

**Senior / Architect:**

5. SPIFFE/SPIRE nedir? Neden microservice ortamında shared secret'tan üstündür?

   > **Beklened:** Problem: 30 microservice → shared API key → rotation zor, granülarite yok, audit zayıf. SPIFFE: standart — her workload'a unique kimlik. `spiffe://trust-domain/ns/prod/sa/payment-service`. X.509 SVID (SPIFFE Verifiable Identity Document). SPIRE: implementasyon. SPIRE Agent (DaemonSet): pod'un kimliğini K8s SA üzerinden attestation. SPIRE Server: sertifika imzalar. Avantajlar: (1) Secret yok → rotation sorunu yok. Sertifika TTL: 1 saat → expire → otomatik yenileme. (2) Her servis farklı kimlik → granüler: payment-service sertifikası ile sadece payment izinli servislere bağlan. (3) Audit: "bu isteği kim attı?" → sertifikaya bak → tam servis kimliği. (4) mTLS: karşılıklı doğrulama → "sen gerçekten order-service misin?" (5) Zero touch: uygulama kodu değişmez. Shared secret: rotation → 30 serviste koordinasyon. Sızdıysa → herkese erişim. "Kim kullandı?" → bilinmiyor. SPIFFE: her biri izole, kısa ömürlü, audit'able.

6. Zero Trust mimarisinin hangi seviyelerini ne zaman uygularken başlamak gerekir? Tüm L4'ü birden yapmak neden risklidir?

   > **Beklened:** L1 (Temel, düşük maliyet): MFA her yerde. HTTPS everywhere (HTTP → HTTPS redirect). Strong şifre politikası. Hemen uygulanabilir, büyük etki. L2 (Kimlik bazlı, orta maliyet): Merkezi IdP (Keycloak, Okta). JWT + kısa TTL. mTLS servisler arası. NetworkPolicy (deny-all + whitelist). Orta ekip, orta maliyet. L3 (İleri, yüksek maliyet): SPIFFE/SPIRE (servis kimliği). OPA policy engine (policy as code). Continuous monitoring (SIEM, anomaly). Vault dynamic secret. Büyük ekip, altyapı gerektirir. L4 (Tam ZT, çok yüksek): Continuous verification (her 15 dakikada re-auth). Behavioural analytics (ML tabanlı anomaly). Device posture continuous check. Neden kademeli: L4'ü birden uygulamak → operasyonel karmaşıklık → outage. HSTS yanlış yapılandırma → site erişilemez. mTLS yanlış → tüm servisler iletişim kesiyor. NetworkPolicy yanlış → DNS bile çalışmayabilir (port 53 unutulursa). Recommendation: startup → L1. Büyüyen ürün → L2. Enterprise/fintech → L3. Kritik altyapı/devlet → L4.

---

## Karma — Architect Seviyesi

7. **"Şirkette Zero Trust'a geçiş yapılıyor. Mevcut VPN tabanlı perimeter security'den nasıl migration yaparsın?"**

   > **Beklened:** Discovery (1. ay): Mevcut durum: hangi servisler, hangi iletişim, hangi auth? Network traffic analizi: tüm servis-to-servis çağrıları haritala. "Kimler kiminle konuşuyor?" — beklenmedik iletişimler? Risk assessment: en yüksek değerli asset'lar → ilk ZT katmanı buraya. L1 — hemen (1-2 ay): MFA tüm kullanıcılara zorunlu (Okta/Azure AD). HTTPS everywhere — HTTP endpoint'leri kapat. Audit logging — tüm erişimler loglanıyor mu? Bu adımlar VPN'i kaldırmıyor, ek katman ekliyor. L2 — paralel altyapı (3-6 ay): Identity Provider entegrasyonu. JWT servisler arası (shared secret kademeli kaldır). NetworkPolicy — deny-all + whitelist (staging'de test). Istio service mesh kurulum — permissive mTLS mode. Permissive: mTLS olmayan trafik de geçiyor (eski servisler kırılmıyor). L3 — sıkılaştırma (6-12 ay): Istio STRICT mTLS mode → mTLS olmayan trafik reddedildi. SPIFFE/SPIRE — servis kimliği. OPA policy engine — centralized authz. Vault dynamic secret — static credential kaldır. VPN kademeli kullanım dışı: VPN üzerinden erişen servis sayısı → düşüyor. Kritik dönem: mTLS strict moduna geçiş → staging'de 2 hafta test. Her servis SVID alıyor mu? → canary deploy. Geri alma planı: permissive mod hızlıca dönülebilir. Metrikler: VPN üzerinden erişim oranı (hedef: 0). mTLS coverage (hedef: %100). Policy violation sayısı. Lateral movement tespit süresi (MTTD).

8. **"Bir fintech şirketinde yetkilendirme modeli seçiyorsun. RBAC mi, ABAC mi, hibrit mi? Neden?"**

   > **Beklened:** Fintech gereksinimleri: compliance (PCI-DSS, BDDK). Çok tenant (farklı şirketler). Context: mesai, konum, cihaz önemli. Fine-grained: "analist sadece kendi müşteri portföyü". Audit: her erişim loglanabilir, raporlanabilir. Hibrit öneririm — neden: RBAC tek başına: role explosion (50+ müşteri × 10 işlem tipi × 3 seviye = 1500 rol). ABAC tek başına: policy karmaşıklığı → debug zor → audit zor → compliance kanıtlama zor. Hibrit (RBAC + ABAC): Kaba taneli: RBAC → ANALYST, TRADER, ADMIN, COMPLIANCE, AUDITOR. İnce taneli: ABAC → "ANALYST rolündeyse AND kendi portfolio'su AND mesai saatlerinde AND yönetilen cihazdan → izin ver." OPA ile implement: Rego → RBAC check + ABAC context check → tek policy engine. Audit: OPA decision log → her karar loglu → PCI-DSS uyumu. ReBAC ne zaman: portföy/hesap hiyerarşisi → "Müşteri A → Alt Hesap A1 → İşlem A1a" ilişkisi varsa. OpenFGA + OPA hibrit: ilişki sorgusu (ReBAC) + context policy (ABAC). Servis kimliği: mTLS + SPIFFE → "bu isteği hangi servis attı?" → OPA policy'de servis kimliği de girdi.
