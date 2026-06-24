# 09 — Güvenlik Genel Bakış: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Authentication & Authorization Hataları

### Gerçek Hayat Sorunları

---

**Sorun 1: BOLA (Broken Object Level Authorization) — başkasının siparişi görüntülendi**

```
Senaryo:
  E-ticaret uygulaması: kullanıcı kendi siparişini görüntülüyor.
  
  // YANLIŞ — nesne seviyesi yetki kontrolü yok
  @GetMapping("/api/orders/{id}")
  public Order getOrder(@PathVariable Long id) {
      return orderRepo.findById(id)
          .orElseThrow(() -> new NotFoundException("Order not found"));
  }

  Saldırı:
    Saldırgan kendi siparişini açtı: GET /api/orders/1042
    URL'deki ID'yi değiştirdi: GET /api/orders/1, 2, 3, ... 9999
    Herkesin siparişi: ad, adres, kart son 4 hane, ürünler.
    IDOR (Insecure Direct Object Reference) saldırısı.

  Gerçek zarar (2019, Parrot):
    Droni şirketi: sipariş ID sequential.
    300,000 kullanıcının verisi → tek bir komut dosyasıyla çekildi.
    GDPR cezası: 3 milyon €.

Düzeltme:
  // DOĞRU — authenticated user'ın ID'si ile çapraz kontrol
  @GetMapping("/api/orders/{id}")
  public Order getOrder(
          @PathVariable Long id,
          @AuthenticationPrincipal UserDetails user) {

      return orderRepo.findByIdAndCustomerEmail(id, user.getUsername())
          .orElseThrow(() -> new AccessDeniedException("Not your order"));
  }

  Ek önlemler:
    UUID kullan (sequential int değil): sipariş ID tahmin edilemez.
    ORDER_123 → c7f3a91b-8e2d-4c5f-a1b9-3d7e2f8c0a4e → brute force zor.
    Ama UUID tek başına yeterli değil — yetki kontrolü zorunlu.

  Her veri erişiminde: "bu kayıt bu kullanıcıya mı ait?"
    findByIdAndOwnerId → tek sorgu, güvenli.
    findById → sonra if check → TOCTOU riski (race condition).
```

---

**Sorun 2: JWT `alg: none` kabul edildi — imzasız token admin yetkisi aldı**

```
Senaryo:
  Eski jjwt kütüphanesi (< 0.10): "alg: none" → imzayı atla.

  Normal token:
    Header: {"alg": "RS256", "typ": "JWT"}
    Payload: {"sub": "user123", "roles": ["user"]}
    Signature: <gerçek imza>

  Saldırı:
    Header: {"alg": "none", "typ": "JWT"}
    Payload: {"sub": "admin", "roles": ["admin", "superuser"]}
    Signature: (boş)

    eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.
    eyJzdWIiOiJhZG1pbiIsInJvbGVzIjpbImFkbWluIl19.
    (nokta var ama imza yok)

  Kütüphane: "alg=none → imza doğrulaması gerekmiyor → kabul et!"
  Saldırgan admin oldu. Hiçbir şifre gerekmedi.

Düzeltme:
  // YANLIŞ — algoritma token'dan okunuyor
  Jwts.parser().parseSignedClaims(token); // hangi alg? token'a soruyor

  // DOĞRU — algoritma server-side sabit
  Jwts.parser()
      .verifyWith(publicKey)            // RS256 sabit, key ile belirtilmiş
      .build()
      .parseSignedClaims(token);
  // "alg: none" token → imza yok → exception → reddedildi

  // Sadece izin verilen algoritmaları kabul et:
  // Spring Security OAuth2:
  NimbusJwtDecoder.withPublicKey(publicKey).build();
  // RS256 hardcoded — none/HS256 reddedilir.

  Kontrol listesi:
    Kütüphane versiyonunu kontrol et (jjwt ≥ 0.10, nimbus-jose-jwt ≥ 6.x).
    "none" algoritmasını explicit olarak reddeden parser kullan.
    Algoritma whitelist: sadece ["RS256"] veya ["ES256"].
    Asla ["RS256", "HS256"] birlikte — alg confusion saldırısı riski.
```

---

**Sorun 3: Refresh token rotate edilmedi — çalınan token sonsuza kadar geçerli**

```
Senaryo:
  Access token: 1 saat, JWT (stateless).
  Refresh token: 30 gün, DB'de.
  Refresh token rotation yok.

  Saldırı senaryosu:
    Kullanıcının cihazı çalındı veya session fixation.
    Saldırgan refresh token'ı ele geçirdi.
    Access token expire oldu → saldırgan: POST /refresh → yeni access token.
    30 gün boyunca her saat yeni access token → hesaba tam erişim.
    Gerçek kullanıcı şifresini değiştirdi → access token süresi doldukça yeni refresh ile devam.
    Şifre değişikliği refresh token'ı geçersiz kılmadı!

  Ek sorun:
    Refresh token DB'de tek kayıt → saldırgan refresh etti → token güncellendi.
    Gerçek kullanıcı refresh yapmaya çalışıyor → geçersiz token → zorla logout.
    Saldırgan hâlâ içeride, gerçek kullanıcı dışarıda!

Düzeltme (Refresh Token Rotation):
  POST /auth/refresh:
    1. Gelen refresh_token'ı DB'de bul.
    2. Geçerli mi? (expire olmamış, revoked değil mi?)
    3. Evet → yeni access_token + yeni refresh_token üret.
    4. ESKİ refresh_token'ı geçersiz kıl (revoke).
    5. Yeni çifti dön.

  // DB şeması
  refresh_tokens:
    id, user_id, token_hash, created_at, expires_at, revoked_at, replaced_by

  Çalınan token tespiti:
    Revoke edilmiş token kullanıldı → "token reuse detected!"
    → Tüm kullanıcının refresh token'larını revoke et (tüm session'ları kapat).
    → Kullanıcıya: "şüpheli aktivite → tekrar giriş gerekli" bildirimi.

  Şifre değişikliği:
    → Tüm refresh token'ları revoke et (tüm cihazlardan çıkış).
    Opsiyonel: "Tüm cihazlarda oturumu kapat" seçeneği.
```

---

**Sorun 4: Excessive Data Exposure — API internal entity döndürdü**

```
Senaryo:
  Kullanıcı profil endpoint'i:

  // YANLIŞ — entity doğrudan dönüyor
  @GetMapping("/api/users/{id}")
  public User getUser(@PathVariable Long id) {
      return userRepo.findById(id).orElseThrow();
  }

  User entity:
  {
    "id": 123,
    "email": "ali@example.com",
    "passwordHash": "$2a$12$N9qo8...",  ← FELAKET
    "phoneVerificationCode": "847291",   ← FELAKET
    "internalScore": 847,                ← iş bilgisi sızdı
    "isAccountSuspended": false,         ← iç durum
    "createdAt": "2024-01-15",
    "lastLoginIp": "195.142.xxx.xxx",    ← GDPR sorunu
    "roles": ["user", "beta_tester"]     ← iç yapı
  }

  Zarar:
    passwordHash → offline brute force.
    phoneVerificationCode → 2FA bypass.
    roles → privilege escalation bilgisi.
    lastLoginIp → location tracking (GDPR ihlali).

Düzeltme:
  // DOĞRU — DTO pattern
  @GetMapping("/api/users/{id}")
  public UserProfileDTO getUser(@PathVariable Long id) {
      User user = userRepo.findById(id).orElseThrow();
      return new UserProfileDTO(user);
  }

  public record UserProfileDTO(
      String displayName,
      String email,         // gerekiyorsa
      String profilePicUrl,
      LocalDate memberSince
  ) {
      UserProfileDTO(User u) {
          this(u.getDisplayName(), u.getEmail(),
               u.getProfilePicUrl(), u.getCreatedAt().toLocalDate());
      }
  }

  Jackson @JsonIgnore (kötü pratik — entity'i değiştiriyor):
    @JsonIgnore private String passwordHash;
    // Refactor edilince unutulabilir, güvensiz.
    DTO daha açık, test edilebilir, bakımı kolay.

  Genel kural: API contract → sadece gerekli alanlar.
    "Neden bu alan lazım?" sorusuna cevap verilemeyen alan → çıkar.
```

---

**Sorun 5: Rate limiting yok — /login endpoint'i brute force ile kırıldı**

```
Senaryo:
  Login endpoint'te rate limiting yok.

  POST /api/auth/login
  Body: {"email": "admin@company.com", "password": "..."}

  Saldırı (credential stuffing / brute force):
    Araç: Hydra, Burp Suite Intruder.
    RockYou şifre listesi: 14 milyon şifre.
    İstek hızı: 500 req/sn (rate limit yok).
    "admin123" → 4 dakikada denendi → başarısız.
    "Company2024!" → 8 dakika → eşleşti! Admin hesabı ele geçirildi.

  Ayrıca: enumeration saldırısı.
    POST /login {"email": "ali@example.com"} → "Şifre yanlış" → email var.
    POST /login {"email": "mehmet@example.com"} → "Kullanıcı bulunamadı" → email yok.
    → Geçerli email listesi oluşturuldu → hedef listesi.

Düzeltme:
  1. Rate limiting (API Gateway veya uygulama seviyesi):
    Per IP: /login → 5 req/dakika.
    Per Account: 10 başarısız → 15 dakika lock.
    Kong/AWS API GW → rate limit plugin.

  2. Spring ile uygulama seviyesi:
    @Bean
    RateLimiter loginRateLimiter() {
        return RateLimiter.create(5.0); // 5 req/sn global (Guava)
    }
    // Bucket4j: per-IP, Redis destekli dağıtık rate limit.

  3. Hesap kilitleme (careful — DoS riski):
    5 başarısız → 15 dakika soft lock.
    CAPTCHA devreye gir.
    Progressive delay: 1s, 2s, 4s, 8s...

  4. Enumeration önleme:
    Hata mesajı: "Email veya şifre hatalı" (hangisi yanlış → bildirilmez).
    Başarısız/başarılı: aynı response süresi (timing-safe).

  5. MFA (Multi-Factor Authentication):
    Şifre kırılsa bile → TOTP kodu gerekli → saldırı başarısız.
```

---

**Sorun 6: Secrets git'e commit edildi — production DB credential'ı sızdı**

```
Senaryo:
  application-prod.properties:
    spring.datasource.password=Pr0d_DB_P@ss!
    stripe.api.key=sk_live_abc123...
    jwt.secret=myProductionSecret

  git add -A && git commit -m "add production config"
  git push origin main  → GitHub public repo!

  Sonuç:
    GitHub botu (truffleHog/GitGuardian): 3 dakika içinde secret'ı buldu.
    Stripe: "sk_live_" prefix → Stripe'ın kendi botu buldu → email uyarısı.
    Ama: 3 saat fark edilmedi.
    Bu sürede: Stripe key ile 47 sahte ödeme → $12,000 kayıp.
    DB: brute force değil, direkt credential → production DB'ye bağlandı.

  Git history'den silmek yetmez:
    git rm + commit: hash değişmez, clone yapanlar hâlâ görür.
    BFG Repo Cleaner / git-filter-repo → history rewrite → force push.
    Ama fork yapıldıysa → kopyası var.
    ÇÖZÜM: Secret rotation (credential'ı geçersiz kıl) birincil öncelik.

Düzeltme:
  Acil: secret rotation → eski key geçersiz.
  
  Doğru yapı:
    application.properties: ${DB_PASSWORD} → env var.
    CI/CD (GitHub Actions Secrets / GitLab CI Variables).
    K8s: Sealed Secrets veya External Secrets Operator.
    HashiCorp Vault: dynamic secret → her deploy yeni credential.

  Pre-commit hook (önleme):
    git-secrets (AWS): pattern tabanlı tarama.
    detect-secrets (Yelp): .secrets.baseline dosyası.
    gitleaks: pre-commit hook.
    // .pre-commit-config.yaml:
    repos:
      - repo: https://github.com/gitleaks/gitleaks
        hooks: [{id: gitleaks}]

  GITIGNOREDOSYALAR:
    .env → .gitignore.
    application-prod.properties → .gitignore.
    *.pem, *.key → .gitignore.
```

---

**Sorun 7: Spring Actuator endpoints açık — /actuator/env production'da**

```
Senaryo:
  Spring Boot uygulaması production'da: tüm actuator endpoint'ler açık.

  management.endpoints.web.exposure.include=*

  Saldırgan:
    GET /actuator/env →
    {
      "propertySources": [{
        "properties": {
          "spring.datasource.password": {"value": "Pr0d_DB_P@ss!"},
          "stripe.api.key": {"value": "sk_live_abc..."},
          "jwt.secret": {"value": "mySecret"}
        }
      }]
    }
    Tüm environment variable'lar → açık!

    GET /actuator/beans → tüm Spring bean'leri → iç mimari haritası.
    GET /actuator/mappings → tüm URL pattern'leri → saldırı yüzeyi.
    POST /actuator/shutdown → uygulamayı kapat (DoS!).
    GET /actuator/heapdump → heap dump → bellekte şifreler.

  Security Misconfiguration — OWASP Top 10 #5.

Düzeltme:
  # Sadece health ve info production'da:
  management.endpoints.web.exposure.include=health,info
  management.endpoint.health.show-details=when-authorized

  # Actuator endpoint'lerine auth zorunlu:
  @Bean
  SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
      http.authorizeHttpRequests(auth -> auth
          .requestMatchers("/actuator/health").permitAll()
          .requestMatchers("/actuator/**").hasRole("ADMIN")
      );
      return http.build();
  }

  # Farklı port (network seviyesinde izole):
  management.server.port=8081
  # 8081 → sadece internal network, dışarıya kapalı.

  # Stack trace production'da gizle:
  server.error.include-stacktrace=never
  server.error.include-message=never
```

---

### Mülakat Soruları

**Junior / Mid:**

1. Authentication ile Authorization arasındaki fark nedir? Örnekle açıkla.

   > **Beklened:** Authentication (Kimlik doğrulama): "Sen kimsin?" → kanıtlama. Kullanıcı adı/şifre, token, parmak izi. Sonuç: "Bu Ali." Authorization (Yetkilendirme): "Ne yapabilirsin?" → izin kontrolü. Sonuç: "Ali ödeme sayfasına girebilir ama admin paneline giremez." Sıra kritik: önce authn, sonra authz. Kim olduğunu bilmeden ne yapabileceğini bilemezsin. Örnek: ATM → kart + PIN (authn). Sonra: para çekme limiti, hesap görüntüleme (authz). Web: JWT ile giriş (authn). `@PreAuthorize("hasRole('ADMIN')")` (authz). Yaygın hata: BOLA — authentication var (kim olduğunu biliyoruz) ama authorization yok (bu kaydın bu kişiye ait olup olmadığını kontrol etmiyoruz). `findById(id)` → anyone's order. `findByIdAndOwnerId(id, userId)` → correct authz.

2. OAuth 2.0 Authorization Code Flow nasıl çalışır? PKCE ne işe yarar?

   > **Beklened:** Authorization Code Flow: (1) Kullanıcı "Google ile giriş yap" tıklar. (2) Client → Auth Server: GET /authorize?client_id&redirect_uri&scope&state. state = random string (CSRF koruması). (3) Kullanıcı Google'da giriş yapar, izin verir. (4) Auth Server → redirect: /callback?code=AUTH_CODE&state=xyz. (5) Client → Auth Server: POST /token (server-to-server, secret gizli): code + client_secret. (6) Auth Server → access_token + refresh_token. (7) Client → Resource API: Bearer access_token. Neden güvenli: access_token tarayıcıya hiç geçmiyor (URL'de değil). client_secret server-to-server → kullanıcı görmez. state → CSRF engeller. PKCE (mobile/SPA için): client_secret saklayamazsın (APK tersine mühendislik). code_verifier (random) → SHA256 → code_challenge. Auth server'a code_challenge gönder. Token isterken code_verifier gönder. Auth server: SHA256(code_verifier) == code_challenge → güvenli. code_challenge tek kullanımlık → intercept saldırısı işe yaramaz.

3. JWT nedir? Payload neden şifrelenmemiştir?

   > **Beklened:** JWT (JSON Web Token): Header.Payload.Signature. Header: algoritma ve tip. Payload (claims): sub, roles, exp, iat. Signature: header + payload imzalanmış. Neden şifrelenmemiş: JWT varsayılan olarak imzalanmış (JWS), şifrelenmiş değil (JWE farklı). Base64URL encoding: şifreleme değil, sadece encoding. Herkes decode edebilir: `atob(token.split('.')[1])` → payload görünür. İmzalama ne sağlar: payload değiştirilemez (private key olmadan imza geçersiz olur). Confidentiality yok: hassas veri koyma (TC kimlik, kart no). Sonuç: JWT içinde: sub (user ID), roles, exp — güvenli. JWT içinde: şifre, kredi kartı numarası — YANLIŞ. JWE (JSON Web Encryption) şifreleme içindir, ayrı standart.

4. OWASP API Top 10'dan en önemli 3 maddeyi açıkla.

   > **Beklened:** BOLA (#1 — en yaygın): GET /orders/123 → başkasının siparişi. Nesne seviyesi yetki kontrolü eksik. Düzeltme: findByIdAndOwnerId. UUID kullan. Broken Authentication (#2): Zayıf şifre, brute force koruması yok, JWT "none" alg kabul, refresh token rotate edilmiyor. Düzeltme: rate limit, MFA, RS256, token rotation. Excessive Data Exposure (#3): entity direkt return → passwordHash, internalScore, sızdı. Düzeltme: DTO pattern, sadece gerekli alanlar. Injection (#4): SQL, NoSQL, command injection. `"WHERE email = '" + email + "'"` → OR 1=1. Düzeltme: parameterized query, JPA, prepared statement. Security Misconfiguration (#5): actuator/env açık, default credentials, debug modda stack trace. Düzeltme: actuator sadece health, auth gerektir.

---

**Senior / Architect:**

5. mTLS nedir? Microservice mimarisinde neden gereklidir?

   > **Beklened:** TLS: server → client'a sertifika sunar. Client doğrular. mTLS (Mutual TLS): her iki taraf sertifika sunar. Client de sertifika → server doğrular. "Sen gerçekten OrderService misin?" Microservice'te neden: servislerin birbirini doğrulaması. OrderService → PaymentService: "Bu istek gerçekten OrderService'ten mi yoksa compromised bir pod'dan mı?" Network içi trafik: K8s pod'ları aynı cluster'da → ağ izole değil. Saldırgan bir pod'u ele geçirdi → PaymentService'e istek atar → mTLS → sertifika yoksa → reddedildi. Servis Mesh (Istio/Envoy): mTLS otomatik. Her pod: sidecar proxy (Envoy). Envoy: sertifika yönetimi + mTLS handshake → uygulama kodu değişmez. Istio: STRICT mPeerAuthentication → mTLS olmayan trafik → reddedildi. PERMISSIVE → geçiş dönemi. Avantajları: mutual auth, şifreli in-cluster trafik, fine-grained authz (RBAC hangi servis hangi endpoint'e erişebilir).

6. Vault dynamic secret nedir? Static secret'a göre ne avantaj sağlar?

   > **Beklened:** Static secret: DB şifresi → bir kez oluştur, Vault'ta sakla, uygulamalar okur. Rotasyon: manuel, riski yüksek, tüm uygulamaları aynı anda güncellemelisin. Dynamic secret: Vault → her uygulama instance'ı için anlık credential üretir. OrderService-pod-1 → Vault: "DB erişimi lazım." Vault → PostgreSQL: `CREATE USER vault_order_1 WITH PASSWORD 'xyz' VALID UNTIL '2h';`. OrderService-pod-1: bu credential ile bağlanır (2 saat TTL). Pod kapanınca Vault → `DROP USER vault_order_1`. Avantajları: (1) Her instance farklı credential → credential leak → sadece o instance'ı etkiler. (2) TTL: 2 saat sonra otomatik expire → rotation otomatik. (3) Audit: hangi pod, ne zaman, hangi credential → tam iz. (4) Breach'te: "DB şifresi çalındı" → sadece o instance'ın 2 saatlik şifresi, tüm sistem değil. Static secret + rotation: tüm uygulamaları aynı anda durdur, güncelle, başlat → downtime. Dynamic: seamless.

---

## Karma — Architect Seviyesi

7. **"Yeni bir microservice sistemi için güvenlik mimarisi nasıl tasarlanır?"**

   > **Beklened:** Kimlik ve erişim: Merkezi Auth Server (Keycloak/Auth0). OAuth 2.0 OIDC: kullanıcı → id_token + access_token. Servis arası: Client Credentials Flow + mTLS. JWT: RS256, kısa TTL (15 dk), refresh token rotation. API güvenliği: API Gateway: TLS termination, rate limiting, WAF. Input validation: server-side zorunlu (Bean Validation). BOLA: her endpoint → ownership check. Excessive data: DTO pattern, field-level authz. Infrastructure: Secrets → Vault dynamic secret. mTLS → Istio (servisler arası). Network Policy → deny-all default. Non-root container + readOnlyFilesystem. Image signing (Cosign). Monitoring: Audit log: kim, ne, ne zaman (ELK/Loki). Anomaly detection (Falco). Rate limit breach → alert. Failed auth spike → alert. Çok fazla secret → PagerDuty. DevSecOps: SAST (SonarQube), DAST (OWASP ZAP), Dependency Check (Snyk). Pre-commit: gitleaks → secret commit engeli. Pentest: yılda 1 kez veya major release öncesi. Threat modeling: STRIDE.

8. **"JWT vs Session Cookie — hangisi ne zaman, tradeoff'lar neler?"**

   > **Beklened:** Session Cookie: server-side state (Redis/DB). Her request: cookie → server → session lookup → user. Avantaj: anında revoke (logout → session sil → bitti). Dezavantaj: her request → Redis lookup (latency). Horizontal scale: sticky session veya dağıtık session store gerekli. JWT: stateless. Token içinde claim'ler. Server-side state yok. Avantaj: horizontal scale kolay (her server doğrulayabilir). Microservice: her servis token'ı doğrulayabilir (public key). API gateway'e ihtiyaç azalır. Dezavantaj: revoke sorunu. Token 1 saat → logout sonrası 1 saat hâlâ geçerli! Çözüm: kısa TTL (15 dk) + refresh token (DB'de) + token blacklist (Redis). Blacklist olunca stateless avantajı azalır. Seçim kriteri: Anlık revoke kritikmi? (ödeme, finans) → Session veya JWT + blacklist. Microservice, çok sayıda servis → JWT (her servise session store bağlamak karmaşık). Mobile: JWT (cookie mobile'da karmaşık). Browser SPA: her ikisi çalışır — cookie (HttpOnly, CSRF) veya localStorage (XSS riski). Security-first: HttpOnly cookie + SameSite + CSRF token.
