# 09 — Güvenlik

## 1. Authentication vs Authorization

```
Authentication (Kimsin?):
"Ben Ali'yim" → kanıtla → username/password, token, certificate

Authorization (Ne yapabilirsin?):
"Ali admin mi? Admin sayfasına girebilir mi?"

Sıra önemli: Önce authn, sonra authz
```

---

## 2. OAuth 2.0 & OpenID Connect (OIDC)

### OAuth 2.0 Nedir?

Authorization framework. "X uygulaması, benim adıma Y uygulamasındaki veriye erişebilsin."

**Roller:**
```
Resource Owner: Kullanıcı (Ali)
Client:         3. taraf uygulama (Spotify)
Authorization Server: Google OAuth Server
Resource Server: Google API (takvim, drive, vb.)
```

### Authorization Code Flow (En Güvenli)

```
1. Kullanıcı "Google ile giriş yap" tıklar

2. Client → Authorization Server:
   GET /authorize?
     client_id=spotify123
     redirect_uri=https://spotify.com/callback
     scope=email,profile
     response_type=code
     state=xyz789  ← CSRF koruması

3. Kullanıcı Google'da login olur, izin verir

4. Authorization Server → Client (redirect):
   https://spotify.com/callback?code=AUTH_CODE&state=xyz789

5. Client → Authorization Server (server-to-server, gizli):
   POST /token
   client_id=spotify123
   client_secret=SECRET  ← tarayıcıda asla görünmez
   code=AUTH_CODE
   grant_type=authorization_code

6. Authorization Server → Client:
   {
     "access_token": "eyJ...",  ← kısa ömürlü (1 saat)
     "refresh_token": "...",    ← uzun ömürlü (30 gün)
     "expires_in": 3600
   }

7. Client → Resource Server:
   GET /api/userinfo
   Authorization: Bearer eyJ...
```

### Client Credentials Flow

Kullanıcı yok, servis-to-servis.

```
OrderService → Auth Server: client_id + client_secret
Auth Server → OrderService: access_token
OrderService → InventoryService: Bearer access_token
InventoryService → Auth Server: token valid mi?
```

### PKCE (Proof Key for Code Exchange)

Mobile/SPA için — client_secret yerine.

```
1. Client code_verifier üretir (random string)
2. code_challenge = SHA256(code_verifier) → Auth server'a gönderir
3. Auth server code'u verirken code_challenge'ı saklar
4. Client token isterken code_verifier'ı gönderir
5. Auth server: SHA256(code_verifier) == code_challenge? → güvenli
```

### OIDC (OpenID Connect)

OAuth 2.0 üzerine identity katmanı. "Kullanıcı kim?" sorusunu cevaplar.

```
OAuth 2.0:  access_token → kaynaklara erişim
OIDC:       id_token (JWT) → kullanıcı kimliği

id_token payload:
{
  "sub": "user123",       ← kullanıcı ID (subject)
  "name": "Ali Yılmaz",
  "email": "ali@example.com",
  "iss": "https://accounts.google.com",  ← issuer
  "aud": "spotify123",    ← audience (client_id)
  "exp": 1705312800,      ← expiry
  "iat": 1705309200       ← issued at
}
```

---

## 3. JWT (JSON Web Token)

### Yapısı

```
Header.Payload.Signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.signature

Header (Base64URL):
{ "alg": "RS256", "typ": "JWT" }

Payload (Base64URL):
{ "sub": "user123", "roles": ["admin"], "exp": 1705312800 }

Signature:
RSASHA256(base64(header) + "." + base64(payload), privateKey)
```

**Önemli:** Base64URL encoding şifreleme değil! Payload herkes okuyabilir. Hassas veri koyma.

### JWT Doğrulama

```java
// RS256: asymmetric (önerilen production için)
// Auth server imzalar (private key)
// Resource server doğrular (public key)

// Auth server
String token = Jwts.builder()
    .subject("user123")
    .claim("roles", List.of("admin"))
    .expiration(new Date(System.currentTimeMillis() + 3600000))
    .signWith(privateKey)  // RS256
    .compact();

// Resource server
Jws<Claims> claims = Jwts.parser()
    .verifyWith(publicKey)
    .build()
    .parseSignedClaims(token);
// İmza geçersizse → exception
// exp geçmişse → exception
```

### Refresh Token Stratejisi

```
Access Token:  kısa ömürlü (15 dk - 1 saat)
Refresh Token: uzun ömürlü (7-30 gün), DB'de saklanır, revoke edilebilir

Flow:
Client → API: Bearer access_token
API: Token süresi dolmuş (401)
Client → Auth: POST /refresh, refresh_token=xxx
Auth: refresh_token valid mi? (DB kontrolü)
Auth → Client: yeni access_token + yeni refresh_token (rotation)
Client: yeni access_token ile devam

Refresh token rotation: Her kullanımda yeni token → çalınan refresh token tek kullanımlık
```

---

## 4. TLS/mTLS Nasıl Çalışır

### TLS Handshake

```
Client                          Server
  │                               │
  │──── ClientHello ─────────────→│
  │     (TLS version, cipher suites, random)
  │                               │
  │←─── ServerHello ──────────────│
  │     (seçilen cipher, random)  │
  │←─── Certificate ──────────────│
  │     (server public key + CA signature)
  │                               │
  │  [Client CA'yı doğrular]      │
  │                               │
  │──── ClientKeyExchange ───────→│
  │     (pre-master secret, server public key ile şifreli)
  │                               │
  │  [Her iki taraf session key'i türetir]
  │                               │
  │──── Finished ────────────────→│
  │←─── Finished ─────────────────│
  │                               │
  │════ Encrypted data ═══════════│
```

**mTLS (Mutual TLS):**
Client de certificate sunar. Servis mesh'te (Istio) servisler birbirini mTLS ile doğrular.

---

## 5. API Security

### OWASP Top 10 API

**1. Broken Object Level Authorization (BOLA)**
```
GET /api/orders/123  → kullanıcı 456 siparişini görebilir mi?

YANLIŞ:
@GetMapping("/orders/{id}")
Order getOrder(@PathVariable Long id) {
    return orderRepo.findById(id).orElseThrow(); // ID kontrolü yok!
}

DOĞRU:
@GetMapping("/orders/{id}")
Order getOrder(@PathVariable Long id, @AuthenticationPrincipal User user) {
    return orderRepo.findByIdAndCustomerId(id, user.getId())
        .orElseThrow(() -> new AccessDeniedException("Not your order"));
}
```

**2. Broken Authentication**
- Zayıf şifre politikası
- Brute force koruması yok
- JWT imzası `none` algoritması kabul edilmesi

**3. Excessive Data Exposure**
```
// YANLIŞ: tüm entity'i döndür
return userRepo.findById(id); // password hash, internal ID vs. dahil

// DOĞRU: DTO kullan
return new UserProfileDTO(user.getName(), user.getEmail());
```

**4. Injection**
```java
// SQL Injection — YANLIŞ
String query = "SELECT * FROM users WHERE email = '" + email + "'";
// email = "' OR '1'='1" → tüm kullanıcılar

// DOĞRU: Parameterized query / JPA
userRepo.findByEmail(email); // otomatik escape
```

**5. Security Misconfiguration**
- Debug endpoint açık (actuator/env, actuator/beans)
- Default credentials
- Stack trace production'da görünür

### Rate Limiting

```
API Gateway seviyesinde rate limiting (Kong, AWS API GW):
- Per IP: 100 req/min (DDoS koruması)
- Per User: 1000 req/min (abuse koruması)
- Per Endpoint: /login 5 req/min (brute force koruması)
```

### Input Validation

```java
@PostMapping("/users")
ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest req) { }

class CreateUserRequest {
    @NotBlank
    @Size(max = 100)
    @Pattern(regexp = "^[a-zA-Z0-9 ]+$")
    String name;
    
    @Email
    @NotBlank
    String email;
    
    @Size(min = 8, max = 72)
    String password;
}
```

---

## 6. Secrets Management (Vault)

```
YANLIŞ: Secrets kaynak kodda / config dosyasında
DB_PASSWORD=secret123  # git'e commit edildi → güvenlik ihlali

DOĞRU: HashiCorp Vault veya AWS Secrets Manager

Spring Boot + Vault:
spring.cloud.vault:
  host: vault.internal
  port: 8200
  authentication: KUBERNETES  # K8s service account ile

# Kod:
@Value("${database.password}")  # Vault'tan otomatik inject
String dbPassword;
```

**Vault özellikleri:**
- Dynamic secrets: her uygulama instance'ı için ayrı DB kullanıcı/şifre üret, TTL ile sil
- Secret rotation: şifre değişince Vault üzerinden → uygulamalar otomatik alır
- Audit log: kim hangi secret'a ne zaman erişti

---

## 7. Architect Security Checklist

```
□ HTTPS everywhere (HTTP → HTTPS redirect)
□ TLS 1.2+ (TLS 1.0/1.1 disabled)
□ JWT: RS256 (symmetric HS256 production'da riskli)
□ Refresh token rotation
□ Rate limiting (login, API, registration)
□ Input validation (server-side, client-side yetmez)
□ CORS konfigürasyonu (wildcard * olmasın)
□ Security headers:
    Strict-Transport-Security: max-age=31536000
    X-Content-Type-Options: nosniff
    X-Frame-Options: DENY
    Content-Security-Policy: default-src 'self'
□ Secrets Vault'ta (git'te değil)
□ Dependency vulnerability scan (Snyk, OWASP Dependency Check)
□ SQL injection: parameterized queries
□ Object-level authorization (BOLA kontrolü)
□ Audit logging (kim, ne zaman, ne yaptı)
□ mTLS servisler arası (production)
```
