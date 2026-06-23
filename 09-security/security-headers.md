# 09e — HTTP Security Headers

## Ne?
Tarayıcıya güvenlik politikaları bildiren HTTP response header'ları.
Server tarafında eklenir, tarayıcı tarafında enforce edilir.

## Neden?
Uygulama kodu doğru yazılmış olsa bile tarayıcı seviyesinde ek savunma katmanı.
Defense in depth: XSS, clickjacking, protocol downgrade, info leak gibi saldırılara karşı.

---

## Content-Security-Policy (CSP)

### Ne?
Tarayıcıya "hangi kaynaklardan script/style/image vb. yüklemeye izin var" bildirir.
XSS saldırılarına karşı en güçlü tarayıcı savunması.

### Direktifler

```
Content-Security-Policy:
  default-src 'self';                        ← default: sadece aynı origin
  script-src 'self' https://cdn.company.com; ← script'ler buradan
  style-src 'self' 'unsafe-inline';          ← inline style'a izin (mümkünse kaldır)
  img-src 'self' data: https:;               ← resimler: self + data URI + her HTTPS
  font-src 'self' https://fonts.gstatic.com; ← font kaynağı
  connect-src 'self' https://api.company.com; ← fetch/XHR hedefleri
  frame-ancestors 'none';                    ← iframe olarak gömülemez (clickjacking)
  object-src 'none';                         ← Flash/Java plugin yok
  base-uri 'self';                           ← <base> tag saldırısı önleme
  form-action 'self';                        ← form sadece aynı origin'e submit
  upgrade-insecure-requests;                 ← HTTP → HTTPS otomatik upgrade

  report-uri /csp-violation-report;          ← ihlal raporla (deprecated)
  report-to csp-endpoint;                    ← yeni yöntem
```

### Nonce ile Inline Script

```
// Her request'te unique nonce üret
String nonce = Base64.getEncoder().encodeToString(
    SecureRandom.getInstanceStrong().generateSeed(16));

// Header:
Content-Security-Policy: script-src 'nonce-{nonce}' 'strict-dynamic';

// HTML:
<script nonce="{nonce}">
  // Bu script çalışır (nonce eşleşiyor)
</script>
<script>
  // Bu çalışmaz (nonce yok)
</script>
<script src="evil.com/x.js"></script>
// Bu çalışmaz (allowlist'te değil)
```

### CSP Test Modu (Report-Only)

```
Content-Security-Policy-Report-Only: default-src 'self'; report-to csp-endpoint

Uygulamayı kırmadan önce ihlalleri logla.
Üretimde önce Report-Only modda başlat → ihlalleri gözlemle → kademeli sıkılaştır.
```

### Spring Security ile CSP

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; "
            + "script-src 'self' https://cdn.company.com; "
            + "frame-ancestors 'none'; "
            + "object-src 'none'")
    )
);
```

---

## Strict-Transport-Security (HSTS)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Ne yapar?
  Tarayıcıya: "Bu siteye 1 yıl boyunca SADECE HTTPS ile bağlan"
  HTTP isteği gelirse → tarayıcı otomatik HTTPS'e yönlendirir (server'a bile gitmiyor)
  MITM saldırısı: HTTP bağlantıyı araya alamaz (tarayıcı kabul etmez)

Parametreler:
  max-age=31536000: 1 yıl (saniye cinsinden)
  includeSubDomains: alt domainleri de kapsar
  preload: tarayıcının dahili HSTS listesine ekle

Dikkat:
  İlk HTTP isteği hâlâ savunmasız (TOFU — Trust on First Use)
  Preload ile bu da çözülür: tarayıcı site her zaman HTTPS olduğunu bilir

Kurulum sırası:
  1. Tüm sayfalar HTTPS'e çalışıyor mu? ← önce bunu sağla
  2. max-age=300 ile başla → sorun yoksa
  3. max-age=31536000; includeSubDomains
  4. preload listesine başvur (hstspreload.org)
```

---

## X-Frame-Options

```
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
X-Frame-Options: ALLOW-FROM https://partner.com  (deprecated)

Ne yapar?
  Clickjacking saldırısını önler
  Saldırgan sitenizi iframe olarak başka sayfaya gömer
  Şeffaf overlay ile kullanıcıyı kendi sitesinde tıklatır

  DENY: hiçbir yerde iframe olamaz
  SAMEORIGIN: sadece aynı origin'de iframe olabilir

Modern alternatif: CSP frame-ancestors direktifi
  frame-ancestors 'none';          → X-Frame-Options: DENY eşdeğeri
  frame-ancestors 'self';          → SAMEORIGIN eşdeğeri
  frame-ancestors https://a.com;  → belirli domain için izin
```

---

## X-Content-Type-Options

```
X-Content-Type-Options: nosniff

Ne yapar?
  MIME type sniffing saldırısını önler
  
  Saldırı senaryosu:
    .jpg uzantılı dosya yükle → içinde HTML/JS
    Tarayıcı: "Content-Type: image/jpeg ama içerik HTML gibi → HTML render"
    XSS!

  nosniff: Tarayıcı, server'ın söylediği Content-Type'a güven
    script için: yalnızca text/javascript türleri çalışır
    style için: yalnızca text/css
```

---

## Referrer-Policy

```
Referrer-Policy: strict-origin-when-cross-origin

Ne yapar?
  Link tıklandığında hedef siteye gönderilen Referer header'ını kontrol eder

Değerler:
  no-referrer:                   Hiç referrer gönderme
  same-origin:                   Sadece aynı origin'e referrer gönder
  strict-origin:                 Origin gönder (https→https), http'ye hiç gönderme
  strict-origin-when-cross-origin: Same-origin'de full URL, cross-origin'de sadece origin
  unsafe-url:                    Her zaman full URL (KULLANma)

Neden önemli?
  https://myapp.com/admin/users/secret → başka siteye tıklarsa
  Referrer: https://myapp.com/admin/users/secret → iç URL sızdı!
  
  strict-origin: Sadece "https://myapp.com" gönderilir (path gizlenir)
```

---

## Permissions-Policy (Feature-Policy'nin yeni adı)

```
Permissions-Policy: geolocation=(), camera=(), microphone=(), payment=(self)

Ne yapar?
  Tarayıcı API'lerini kısıtlar
  
  geolocation=():    Hiçbir frame konuma erişemez
  camera=():         Kamera yasak
  microphone=():     Mikrofon yasak
  payment=(self):    Sadece kendi origin payment API kullanabilir
  fullscreen=(self): Sadece kendi origin fullscreen

Neden önemli?
  Kötü amaçlı iframe → kullanıcının konumuna erişmeye çalışır → politika engeller
```

---

## Cache-Control (Hassas Veriler için)

```
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache  (HTTP/1.0 uyumluluğu için)

Hassas sayfalarda (admin, profil, ödeme):
  no-store: Asla önbelleğe alma (disk dahil)
  no-cache: Önbelleğe al ama her seferinde doğrula

Neden?
  Shared computer'da tarayıcı geri butonu → cache'den hassas veri görünür
  Proxy cache → başka kullanıcı görür
```

---

## Tam Spring Security Header Konfigürasyonu

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            // HSTS
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
                .preload(true)
            )
            // Frame options
            .frameOptions(frame -> frame.deny())
            // Content type
            .contentTypeOptions(Customizer.withDefaults())
            // Referrer policy
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
            )
            // CSP
            .contentSecurityPolicy(csp -> csp
                .policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self'; " +
                    "frame-ancestors 'none'; " +
                    "object-src 'none'")
            )
            // Permissions Policy (custom header)
            .addHeaderWriter(new StaticHeadersWriter(
                "Permissions-Policy",
                "geolocation=(), camera=(), microphone=()"
            ))
        );
    return http.build();
}
```

---

## SecurityHeaders.com Skoru

```
A+ almak için gereken minimum set:
  ✓ Strict-Transport-Security  (max-age ≥ 1 yıl + preload)
  ✓ Content-Security-Policy    (unsafe-inline yok, unsafe-eval yok)
  ✓ X-Frame-Options           (DENY)
  ✓ X-Content-Type-Options    (nosniff)
  ✓ Referrer-Policy           (strict-origin veya daha kısıtlı)
  ✓ Permissions-Policy        (gereksiz API'leri kapat)
```

---

## Trade-off Özeti

| Header | Koruma | Dezavantaj | Öncelik |
|--------|--------|------------|---------|
| CSP | XSS, data injection | Konfigürasyon karmaşık, legacy site ile sorun | Kritik |
| HSTS | SSL stripping, MITM | Yanlış config → site erişilemez (max-age) | Kritik |
| X-Frame-Options | Clickjacking | CSP ile örtüşür | Yüksek |
| X-Content-Type | MIME sniffing | Minimal etki | Orta |
| Referrer-Policy | Bilgi sızıntısı | Analytics etkilenebilir | Orta |
| Permissions-Policy | Kötüye kullanım | Progressive Web App ile çatışabilir | Orta |
