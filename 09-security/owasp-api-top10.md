# 09a — OWASP API Security Top 10 (2023)

## API1 — Broken Object Level Authorization (BOLA)

### Ne?
Her kullanıcı sadece kendi kaynaklarına erişebilmeli. Başkasının kaydına ID ile erişmek BOLA'dır.

### Nasıl Saldırı?
```
Kullanıcı A (userId=1):
  GET /api/orders/1001 → kendi siparişi ✓
  GET /api/orders/1002 → başkasının siparişi! (ID değiştirdi)
  → Server kontrol etmediyse → veri sızıntısı
```

### Nasıl Önlenir?

```java
// YANLIŞ — sadece ID ile fetch, sahiplik kontrolü yok
@GetMapping("/orders/{orderId}")
Order getOrder(@PathVariable Long orderId) {
    return orderRepo.findById(orderId).orElseThrow();
}

// DOĞRU — sahiplik kontrolü
@GetMapping("/orders/{orderId}")
Order getOrder(@PathVariable Long orderId,
               @AuthenticationPrincipal UserDetails user) {
    return orderRepo.findByIdAndOwnerId(orderId, user.getId())
        .orElseThrow(() -> new ResponseStatusException(
            HttpStatus.FORBIDDEN, "Access denied"));
}

// Alternatif: Spring Security Method Security
@PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")
@GetMapping("/orders/{orderId}")
Order getOrder(@PathVariable Long orderId) { ... }
```

---

## API2 — Broken Authentication

### Saldırı Vektörleri

```
1. Brute Force: Şifre kısa/sözlük → sonsuz deneme
   Önlem: 5 başarısız denemede hesap kilitle, CAPTCHA, exponential backoff

2. Credential Stuffing: Sızdırılmış kullanıcı adı/şifre listeleri
   → Haveibeenpwned.com kontrolü, rate limiting, anomaly detection

3. JWT Algorithm Confusion:
   YANLIŞ: alg: "none" kabul etmek → imzasız token geçer!
   YANLIŞ: HS256 + public key → algoritma downgrade saldırısı
   DOĞRU: Sadece RS256 / ES256 kabul et (whitelist), alg'yi her zaman doğrula

4. Token Predictability:
   YANLIŞ: token = "user_" + userId → tahmin edilebilir
   DOĞRU: cryptographically random token (UUID v4, SecureRandom)
```

```java
// JWT algorithm validation (Spring Security)
@Bean
JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withPublicKey(rsaPublicKey)
        .build();
    
    // Sadece RS256 kabul et
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        new JwtTimestampValidator(),
        new JwtIssuerValidator("https://auth.myapp.com"),
        jwt -> {
            if (!"RS256".equals(jwt.getHeaders().get("alg"))) {
                return OAuth2TokenValidatorResult.failure(
                    new OAuth2Error("invalid_algorithm"));
            }
            return OAuth2TokenValidatorResult.success();
        }
    ));
    return decoder;
}
```

---

## API3 — Broken Object Property Level Authorization

### Ne?
Kullanıcı kendi kaydına erişebiliyor ama değiştirememesi gereken alanları değiştirebiliyor.

```
YANLIŞ: Tüm request body'yi entity'ye map et
  PUT /users/123
  {"name": "Ali", "role": "ADMIN"}   ← role değiştirdi!

  @PutMapping("/users/{id}")
  User update(@PathVariable Long id, @RequestBody User user) {
      return userRepo.save(user); // role dahil her şeyi güncelledi!
  }

DOĞRU: İzin verilen alanları açıkça belirt
  record UpdateUserRequest(@NotBlank String name, @Email String email) {}
  // role, isActive, balance → bu DTO'da yok → değiştirilemez
```

```java
// Mass Assignment Protection
@PutMapping("/users/{id}")
User update(@PathVariable Long id,
            @RequestBody UpdateUserRequest req,  // ← sadece izinli alanlar
            @AuthenticationPrincipal UserDetails auth) {
    User user = userRepo.findByIdAndEmail(id, auth.getUsername())
        .orElseThrow();
    user.setName(req.name());    // sadece izinli alanları güncelle
    user.setEmail(req.email());  // role, isActive, balance dokunulmadı
    return userRepo.save(user);
}
```

---

## API4 — Unrestricted Resource Consumption

### Ne?
Sınırsız kaynak tüketimi: dosya upload, batch request, pagination limiti yok.

```
Saldırı örnekleri:
  POST /images → 10 GB dosya upload → disk dolu
  GET /users?page=1&size=1000000 → tüm DB belleğe
  POST /send-sms → tek endpoint'ten 10M SMS (maliyet saldırısı)
  POST /batch → 10,000 item batch → CPU spike

Önlemler:
  Upload: max dosya boyutu (örn. 10 MB)
  Pagination: max page size = 100
  Timeout: Her request için zaman sınırı
  Rate limit: endpoint bazlı
  Batch: max item sayısı = 100
  Maliyet izleme: SMS, email gönderim limiti
```

```java
@PostMapping("/upload")
ResponseEntity<?> uploadFile(
        @RequestParam MultipartFile file) {
    if (file.getSize() > 10 * 1024 * 1024) { // 10 MB
        throw new ResponseStatusException(HttpStatus.PAYLOAD_TOO_LARGE);
    }
    // ...
}

@GetMapping("/users")
Page<User> listUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    size = Math.min(size, 100); // max 100 — büyük değer gelirse kapat
    return userRepo.findAll(PageRequest.of(page, size));
}
```

---

## API5 — Broken Function Level Authorization (BFLA)

### Ne?
BOLA: objeye erişim. BFLA: fonksiyona/işleme erişim.
Kullanıcı, admin fonksiyonlarını çağırabilmeli mi?

```
Saldırı:
  Normal kullanıcı:
  GET  /api/v1/users/123/profile       ← izinli
  DELETE /api/v1/users/456             ← admin endpoint! kullanıcı çağırdı?
  POST /api/v1/users/123/upgrade-to-admin ← açık mı?

Önlem:
  Her endpoint için açıkça rol kontrolü
  "Security by obscurity" yetmez (endpoint gizli olsa da güvensiz)
```

```java
@DeleteMapping("/users/{id}")
@PreAuthorize("hasRole('ADMIN')")  // ← açık yetki tanımı
ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}

// Admin endpoint'lerini ayrı path'e koy ve ayrı güvenlik filtresi ekle
// /api/admin/** → ADMIN rolü zorunlu (SecurityConfig'de)
http.securityMatcher("/api/admin/**")
    .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("ADMIN"));
```

---

## API6 — Unrestricted Access to Sensitive Business Flows

### Ne?
Otomasyon ile kötüye kullanılabilen iş akışları.

```
Örnekler:
  İndirim kuponu sistemi: Sınırsız kupon oluşturma bot'u
  Bilet satışı: Bot saniyeler içinde tüm biletleri satın alır
  Referral bonus: Aynı kişi yüzlerce hesap açıp bonus toplar
  Flash sale: Bot saniyelik avantajlı ürünleri kapıyor

Önlemler:
  CAPTCHA (bilet satışı, kayıt)
  Device fingerprinting (bot tespiti)
  İş kuralı limitleri: "Aynı email max 3 kupon"
  Davranış analizi: 100 ms'de satın alma? → bot
  Hesap doğrulama: email/phone verify olmadan limit
```

---

## API7 — Server Side Request Forgery (SSRF)

### Ne?
Server, kullanıcının verdiği URL'ye istek atar → iç ağa veya cloud metadata'ya erişim.

```
Saldırı:
  POST /api/fetch-url
  {"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}
  
  → AWS EC2 metadata endpoint → IAM credentials çalınır!
  
  Diğer hedefler:
  http://localhost:8080/actuator/env    (internal endpoints)
  http://redis:6379/                    (internal services)
  file:///etc/passwd                    (local files)
```

```java
// SSRF Önlemi: URL whitelist veya validation
@PostMapping("/webhooks/test")
void testWebhook(@RequestBody @Valid WebhookTestRequest req) {
    URI uri = URI.create(req.getUrl());
    
    // Private IP aralıklarını reddet
    InetAddress address = InetAddress.getByName(uri.getHost());
    if (isPrivateAddress(address)) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST,
            "Private IP addresses not allowed");
    }
    
    // Sadece HTTP/HTTPS
    if (!Set.of("http", "https").contains(uri.getScheme())) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST);
    }
    
    // Whitelist yaklaşımı (daha güvenli)
    if (!allowedDomains.contains(uri.getHost())) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST,
            "Domain not in whitelist");
    }
    
    httpClient.get(req.getUrl());
}

private boolean isPrivateAddress(InetAddress addr) {
    return addr.isLoopbackAddress()     // 127.0.0.1
        || addr.isLinkLocalAddress()    // 169.254.x.x (cloud metadata)
        || addr.isSiteLocalAddress();   // 10.x.x.x, 192.168.x.x
}
```

---

## API8 — Security Misconfiguration

```
Yaygın hatalar:

1. Debug endpoint'leri açık:
   /actuator/env     → tüm environment variable'lar görünür
   /actuator/heapdump → JVM heap dump → plaintext şifre içerebilir!
   /h2-console       → in-memory DB admin console
   
   Önlem:
   management.endpoints.web.exposure.include=health,info
   # env, heapdump, beans → kapalı

2. Stack trace production'da:
   {"error": "SQLException: table 'users' doesn't exist..."}
   → DB şema bilgisi sızdı
   
   Önlem:
   @ExceptionHandler
   ErrorResponse handleException(Exception e) {
       log.error("Error", e); // logla ama response'a koyma
       return new ErrorResponse("Internal server error"); // generic mesaj
   }

3. Default credentials:
   MongoDB admin:admin, Redis şifresiz, Elasticsearch public
   
4. CORS wildcard:
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Credentials: true
   → Session cookie başka domain'den gönderilir!

5. HTTP'de API:
   Hassas veri → plaintext → sniffing
```

---

## API9 — Improper Inventory Management

```
Eski/unutulmuş API versiyonları güvenlik açığı!

Örnekler:
  /api/v1/login    → eski, güvenlik düzeltmeleri yapılmadı
  /api/v2/login    → güncel
  
  /api/internal/   → production'da açık kaldı
  /api/debug/      → test sırasında eklendi, kaldırılmadı
  /api/beta/       → güvenlik test edilmedi

Önlemler:
  API versiyonlarını belge et (Swagger/OpenAPI)
  Kullanılmayan endpoint'leri kaldır
  API Gateway'de bilinen endpoint'leri whitelist et (unknown → 404)
  Eski version'ı deprecate et → sunset tarihi belirle → kapat
  API discovery araçları (OWASP Amass, nuclei)
```

---

## API10 — Unsafe Consumption of APIs

```
3rd party API'den gelen veriyi güvensizce kullanmak

Senaryo:
  3rd party payment API → ödeme bilgileri
  Response: {"status": "success", "redirectUrl": "http://evil.com"}
  → Uygulama bu URL'ye redirect etti!

Senaryo 2:
  3rd party haber API → içerik → HTML olarak render
  İçerik: "<script>stealCookie()</script>"
  → XSS!

Önlemler:
  3rd party API response'unu da validate et
  HTML içerik → sanitize et (DOMPurify, OWASP Java HTML Sanitizer)
  URL → whitelist/validation (SSRF önlemiyle aynı)
  3rd party'yi de güvensiz say → defense in depth
  Timeout + circuit breaker (3rd party down → sistem çökmesin)
```

---

## OWASP Checklist (Architect için)

```
□ API1 BOLA: Her endpoint'te object owner check
□ API2 Auth: JWT alg whitelist, brute force protection, rate limit /login
□ API3 Mass Assignment: DTO kullan, entity direkt expose etme
□ API4 Resource Limit: Upload size, pagination max, request timeout
□ API5 BFLA: Tüm admin endpoint'leri @PreAuthorize ile koru
□ API6 Business Flow: CAPTCHA, device fingerprint, iş kuralı limiti
□ API7 SSRF: URL validation, private IP block, whitelist
□ API8 Misconfiguration: actuator kapalı, stack trace gizli, CORS kısıtlı
□ API9 Inventory: Eski API'ler kaldırıldı, API Gateway whitelist
□ API10 3rd Party: Response validation, HTML sanitize, URL kontrolü
```
