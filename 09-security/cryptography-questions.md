# 09c — Kriptografi: Mülakat Soruları & Gerçek Hayat Sorunları

---

## Bölüm 1: Şifreleme & Hashing Hataları

### Gerçek Hayat Sorunları

---

**Sorun 1: AES-ECB modu kullanıldı — pattern sızdı, şifrelenmiş veriden anlam çıkarıldı**

```
Senaryo:
  Ekip: "Veritabanındaki hassas alanları şifreleyelim."

  // YANLIŞ — ECB modu
  Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
  cipher.init(Cipher.ENCRYPT_MODE, secretKey);
  byte[] encrypted = cipher.doFinal(plaintext.getBytes());

  Sorun:
    Aynı plaintext → her zaman aynı ciphertext üretir.
    
    Kullanıcı tablosunda şifrelenmiş "şehir" alanı:
      "İstanbul" → "3f9a..." (her kayıtta aynı!)
      "Ankara"   → "7c2b..."
      "İstanbul" → "3f9a..." (tekrar!)

    Saldırgan (DB erişimi olmadan, sadece şifreli veri):
      Hangi alanlar sık tekrar ediyor → hangi değer popüler.
      Frekans analizi → şifreli değer tablosu oluştur.
      Penguen saldırısı: resim ECB ile şifrelenirse → şekil hâlâ görünür.
      
  Gerçek zarar:
    Tıp uygulaması: şifrelenmiş "teşhis" alanında belirli pattern sık tekrar ediyor.
    Hangi şehirde hangi hastalık yaygın → demografik profil çıkarıldı.
    KVKK/GDPR: veri ihlali — şifreleme var ama anlamsız.

Düzeltme (AES-256-GCM):
  public byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
      byte[] nonce = new byte[12];
      SecureRandom.getInstanceStrong().nextBytes(nonce);  // HER şifrelem. yeni nonce!

      Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
      cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, nonce));
      byte[] ciphertext = cipher.doFinal(plaintext);

      // nonce + ciphertext birlikte sakla (nonce gizli değil, unique olmalı)
      ByteBuffer result = ByteBuffer.allocate(12 + ciphertext.length);
      result.put(nonce);
      result.put(ciphertext);
      return result.array();
  }

  Sonuç:
    "İstanbul" → "a1b2...nonce1" (kayıt 1)
    "İstanbul" → "f9c3...nonce2" (kayıt 2 — tamamen farklı!)
    Authentication tag: şifreli veri değiştirilirse → decrypt başarısız → fark edilir.
    ECB: şifreleme var, güvenlik yok. GCM: hem gizlilik hem bütünlük.
```

---

**Sorun 2: Şifre MD5 ile hashlenmiş — veri tabanı sızıntısında tüm şifreler kırıldı**

```
Senaryo:
  2019: büyük e-ticaret sitesi SQL injection saldırısına uğradı.
  users tablosu dump edildi: email + password_hash.

  Şema:
    INSERT INTO users (email, password_hash) VALUES
    ('user@example.com', '5f4dcc3b5aa765d61d8327deb882cf99');  -- "password"
    ('admin@site.com',   '21232f297a57a5a743894a0e4a801fc3'); -- "admin"

  Saldırgan:
    Hashcat GPU saldırısı — MD5:
    Hız: 50 milyar MD5/saniye (RTX 4090)
    Rainbow table: önceden hesaplanmış MD5 → plaintext tablosu.
    '5f4dcc3b5aa765d61d8327deb882cf99' → Google araması → "password".
    8 karakterli alfanümerik şifre: MD5 ile dakikalar içinde kırılır.
    
    Sonuç: 2 saat içinde %80 kullanıcının şifresi açık.
    Credential stuffing: aynı şifreler → Gmail, Twitter, Amazon'da denendi.

  Neden MD5 yetersiz:
    Çok hızlı (tasarımı: hız).
    Salt yok → rainbow table saldırısı.
    Bilinen collision'lar → kriptografik olarak kırık.
    SHA-256 da bu açıdan yetersiz — hâlâ çok hızlı!

Düzeltme (bcrypt + Argon2):
  // bcrypt — cost 12 (≈250ms/hash, GPU: 10,000/sn değil ~10/sn)
  @Bean
  PasswordEncoder passwordEncoder() {
      return new BCryptPasswordEncoder(12);
  }

  String hash = encoder.encode("userPassword");
  // $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
  // ^^^  ^^^ —— cost=12, dahili salt dahil, her encode farklı hash!

  // Argon2id — 2023+ önerilen
  return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
  // memory=64MB → GPU'nun hızını kırar (az VRAM)

  bcrypt avantajı:
    Her hash'e dahili 128-bit salt → rainbow table imkansız.
    cost=12: ~250ms → 4 hash/sn → saldırgan için milyar yerine 4!
    Upgrade path: login'de bcrypt → Argon2id re-hash (transparent migration).
```

---

**Sorun 3: Nonce/IV yeniden kullanıldı — AES-GCM tamamen kırıldı**

```
Senaryo:
  Geliştirici: "Nonce her zaman aynı olsun, basit olsun."

  private static final byte[] FIXED_NONCE = new byte[12]; // hep sıfır!

  public byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
      Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
      cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, FIXED_NONCE));
      return cipher.doFinal(plaintext);
  }

  Matematiksel felaket (Nonce reuse attack):
    AES-GCM: ciphertext = plaintext XOR keystream(nonce, key)
    
    C1 = P1 XOR KS(nonce, key)
    C2 = P2 XOR KS(nonce, key)   ← aynı nonce!
    
    C1 XOR C2 = P1 XOR P2        ← keystream iptal oldu!
    
    Saldırgan C1 ve C2'yi biliyor (network trafiği):
    P1 biliniyorsa → P2 hesaplanabilir.
    Authentication tag: nonce reuse → tag forgery mümkün (authentication kırıldı).

  Gerçek dünya örneği:
    TLS 1.3 nonce reuse → NIST uyarısı.
    WPA2 KRACK saldırısı: nonce reuse → Wi-Fi şifrelemesi kırıldı.

Düzeltme:
  // Her şifreleme için cryptographically secure random nonce:
  byte[] nonce = new byte[12];
  SecureRandom.getInstanceStrong().nextBytes(nonce);
  // SecureRandom — Math.random() değil, /dev/urandom tabanlı, PRNG değil CSPRNG.

  Mesaj sayacı alternatifi (deterministic nonce):
    96-bit nonce = 32-bit senderID + 64-bit counter
    Counter hiç tekrar etmemeli (long overflow dikkat).
    Counter durumu kalıcı saklanmalı (restart sonrası sıfırlanırsa → felaket).
    → Pratikte SecureRandom daha güvenli.
```

---

**Sorun 4: HMAC doğrulamasında timing attack — string equals ile karşılaştırma**

```
Senaryo:
  GitHub webhook doğrulama implementasyonu:

  // YANLIŞ — timing attack açığı
  @PostMapping("/webhook")
  public ResponseEntity<?> handleWebhook(
          @RequestHeader("X-Hub-Signature-256") String signature,
          @RequestBody byte[] body) {

      String expected = computeHmac(body, webhookSecret);
      
      if (signature.equals("sha256=" + expected)) {  // YANLIŞ!
          processWebhook(body);
          return ResponseEntity.ok().build();
      }
      return ResponseEntity.status(403).build();
  }

  Timing attack:
    String.equals(): ilk farklı byte'ta durur.
    "sha256=abc" vs "sha256=xyz":
      'a' vs 'x' → farklı → hemen return false (çok hızlı).
      "sha256=abc" vs "sha256=abc123":
        'a'='a', 'b'='b', 'c'='c', null!='1' → biraz daha yavaş.
    Saldırgan: 1000 istek → her prefix için yanıt süresini ölç.
    Hangi prefix daha uzun sürdü → o prefix doğru → bir sonraki byte'ı dene.
    Brute force yerine O(n) → HMAC kırıldı!

Düzeltme (Constant-time comparison):
  // DOĞRU — sabit zamanlı karşılaştırma
  boolean valid = MessageDigest.isEqual(
      expected.getBytes(StandardCharsets.UTF_8),
      received.getBytes(StandardCharsets.UTF_8));
  // HER ZAMAN tüm byte'ları karşılaştırır → eşit uzunluk bile olsa aynı süre.

  // Tam implementasyon:
  @PostMapping("/webhook")
  public ResponseEntity<?> handleWebhook(
          @RequestHeader("X-Hub-Signature-256") String receivedSig,
          @RequestBody byte[] body) {

      Mac mac = Mac.getInstance("HmacSHA256");
      mac.init(new SecretKeySpec(webhookSecret.getBytes(), "HmacSHA256"));
      String expected = "sha256=" + HexFormat.of().formatHex(mac.doFinal(body));

      if (!MessageDigest.isEqual(expected.getBytes(), receivedSig.getBytes())) {
          return ResponseEntity.status(403).build();
      }
      processWebhook(body);
      return ResponseEntity.ok().build();
  }
```

---

**Sorun 5: RSA ile büyük veri şifreleme — performans çöküşü ve güvenlik hatası**

```
Senaryo:
  Ekip: "Kullanıcı dosyalarını RSA public key ile şifreleyelim."

  // YANLIŞ — büyük dosyayı direkt RSA ile şifreleme
  Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
  cipher.init(Cipher.ENCRYPT_MODE, rsaPublicKey);
  byte[] encrypted = cipher.doFinal(fileBytes);  // 10MB dosya → HATA!

  Sorunlar:
    RSA-2048: maksimum plaintext = 245 byte (PKCS1Padding).
    10MB dosya → doğrudan şifrelenemez → IllegalBlockSizeException.
    Eğer chunk'lara bölünse: çok yavaş. RSA: AES'ten 1000x daha yavaş.
    PKCS1Padding: güvenlik açığı (Bleichenbacher saldırısı). OAEP kullan.

Doğru yaklaşım (Hybrid Encryption):
  // 1. AES key üret (rastgele, tek kullanım)
  KeyGenerator keyGen = KeyGenerator.getInstance("AES");
  keyGen.init(256);
  SecretKey aesKey = keyGen.generateKey();

  // 2. Dosyayı AES-GCM ile şifrele (hızlı, büyük veri için)
  byte[] encryptedFile = aesEncrypt(fileBytes, aesKey);

  // 3. AES key'i RSA public key ile şifrele (sadece 32 byte!)
  Cipher rsaCipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
  rsaCipher.init(Cipher.ENCRYPT_MODE, rsaPublicKey);
  byte[] encryptedAesKey = rsaCipher.doFinal(aesKey.getEncoded()); // küçük, hızlı

  // 4. Paketi gönder: encryptedAesKey + encryptedFile
  // Alıcı: RSA private key → AES key → dosyayı çöz

  Bu: TLS'nin de yaptığı şey (TLS handshake: RSA/ECDH ile key exchange → AES ile veri).
  
  OAEP neden?
    PKCS1v1.5: Bleichenbacher '98 oracle saldırısı → TLS ROBOT saldırısı (2017).
    OAEP: modern, güvenli dolgu şeması.
    Her zaman: "RSA/ECB/OAEPWithSHA-256AndMGF1Padding"
```

---

**Sorun 6: Pepper'sız bcrypt — DB sızıntısında offline saldırı**

```
Senaryo:
  Uygulama bcrypt kullanıyor (cost=12). DB dump edildi.

  Saldırgan:
    bcrypt: GPU'da yavaş. Ama boş vakti var.
    Yaygın şifreler listesi (RockYou: 14 milyon şifre):
      "123456", "password", "qwerty", ... → bcrypt hash → DB ile karşılaştır.
      cost=12: ~4 hash/sn → 14 milyon şifre → ~40 gün.
      Ama gerçekçi: kullanıcıların %60'ı top 1000 şifre → 4 dakika!

  Pepper ile ek koruma:
    pepper = random 32-byte secret, APP CONFIG'de (DB'de değil!).
    finalHash = bcrypt(password + pepper)
    
    DB sızıntısı: pepper olmadan kırma imkansız.
    Pepper çalınmadıkça (ayrı depolama → Vault, HSM) → güvenli.

  // Implementasyon
  @Service
  public class PasswordService {
      @Value("${app.security.pepper}")  // env var veya Vault'tan
      private String pepper;

      public String hash(String rawPassword) {
          return encoder.encode(rawPassword + pepper);
      }

      public boolean verify(String rawPassword, String hash) {
          return encoder.matches(rawPassword + pepper, hash);
      }
  }

  Pepper rotation:
    Pepper değişirse: tüm hash'ler geçersiz → login'de re-hash.
    Version'lama: hash'e pepper_version ekle → gradual migration.
    pepper_v1_$2a$12$...  → v1 ile doğrula, v2'ye upgrade.

  Özet: bcrypt = brute force yavaşlatır. Pepper = DB çalınsa bile korur. İkisi birlikte.
```

---

## Bölüm 2: Asimetrik Şifreleme & İmzalama

### Gerçek Hayat Sorunları

---

**Sorun 7: SHA-1 kullanımı — collision saldırısı ile sahte sertifika**

```
Senaryo (SHAttered saldırısı, 2017):
  Google araştırmacıları: aynı SHA-1 hash'e sahip iki farklı PDF ürettiler.
  Maliyet: ~110 GPU-yıl (2017'de ~$110,000 → 2026'da çok daha ucuz).

  Etkisi:
    Git: SHA-1 kullanıyordu → aynı hash'e iki farklı dosya → güven sorunu.
    SSL/TLS: eski CA'lar SHA-1 ile imzalanmış sertifikalar verdi.
    Saldırı senaryosu:
      saldırgan SHA-1 collision → "legitsite.com" ile "evilsite.com"
      aynı SHA-1 hash'e sahip iki sertifika isteği gönder.
      CA imzaladı (SHA-1 ile) → sertifika sahte siteye de uyuyor.
      MITM: gerçek sertifika yerine sahte → geçerli görünür.

  Git'in SHAttered yanıtı:
    git log --oneline → SHA-1 commit ID'leri.
    GitHub: SHA-256'ya geçiş başladı (2021+).
    Git 2.29+: SHA-256 depo desteği.

  Günümüzde SHA-1:
    TLS: 2017'de browserlar SHA-1 sertifikaları reddetti.
    JWT: HS1/RS1 → asla kullanma.
    Dosya checksum: SHA-256 veya SHA-3 kullan.
    Sadece eski sistem uyumu için → explicit deprecation planı yap.

Düzeltme:
  MessageDigest.getInstance("SHA-256");  // minimum
  MessageDigest.getInstance("SHA-3-256"); // alternatif, farklı yapı
  // SHA-1 ve MD5: sadece non-security amaçlı (checksum performance) geçici.
```

---

## Mülakat Soruları

**Junior / Mid:**

1. Hashing, encryption ve MAC arasındaki fark nedir? Hangisi hangi durumda kullanılır?

   > **Beklened:** Hashing: tek yönlü, geri döndürülemez. Aynı input → aynı output. Şifre saklama (bcrypt), checksum, imza. SHA-256: veri bütünlüğü — gizlilik yok. Encryption: iki yönlü, anahtar ile şifrele/çöz. Symmetric (AES): aynı anahtar. Asymmetric (RSA): public ile şifrele, private ile çöz. Veriyi gizli tutmak için. MAC/HMAC: mesajın hem bütünlüğünü hem kaynağını doğrula. Paylaşılan secret key ile. "Bu mesaj gerçekten Ali'den mi geldi?" → JWT HS256, webhook doğrulama. Digital Signature: HMAC gibi ama asimetrik. Private key ile imzalar, public key ile herkes doğrular. "Bu belgeyi Ali imzaladı mı?" → RSA-SHA256, ECDSA, JWT RS256. Hangi durumda: şifre sakla → bcrypt/Argon2 (hashing). Dosyayı gizli transfer → AES-GCM (encryption). API webhook → HMAC-SHA256 (MAC). Sertifika/JWT → ECDSA (digital signature).

2. Neden şifre saklama için SHA-256 kullanılmamalı? bcrypt ne fark yaratır?

   > **Beklened:** SHA-256 çok hızlı — kasıtlı olarak tasarlandı (veri bütünlüğü için). GPU ile 10 milyar SHA-256/sn. 8 karakterli şifre: saniyeler içinde kırılır. Salt olmadan: rainbow table saldırısı. bcrypt farkı: (1) Kasıtlı yavaş — cost factor: 2^12 iterasyon → ~250ms/hash. GPU'da: 50 milyar SHA-256/sn vs 10 bcrypt/sn. (2) Dahili salt: her hash farklı → rainbow table imkansız. Hash formatı: `$2a$12$[22char salt][31char hash]` — salt hash'e gömülü. (3) Adaptive: sunucu hızlandıkça cost artır. Argon2id daha da iyi: memory-hard → GPU'nun az VRAM'i → daha da yavaş. 64MB bellek kullanımı → aynı anda kaç hash hesaplayabilir? Çok az. Kural: kullanıcı şifreleri → bcrypt (cost≥12) veya Argon2id. Asla: MD5, SHA-256, SHA-1 şifre için.

3. AES-GCM neden AES-CBC veya AES-ECB'ye tercih edilir?

   > **Beklened:** ECB: aynı plaintext → aynı ciphertext → pattern sızdırır. "Penguen saldırısı" — şifrelenmiş resimde şekil görünür. HİÇBİR ZAMAN kullanma. CBC: önceki ciphertext ile XOR → pattern yok, iyi. Ama sadece şifreleme, bütünlük yok. Padding oracle saldırısı (POODLE). IV güvenli taşınmalı. GCM üstünlükleri: (1) AEAD (Authenticated Encryption with Associated Data): hem şifreleme hem bütünlük. 16-byte authentication tag: şifreli veri değiştirilirse → AEADBadTagException. (2) Paralel işleme: counter modu tabanlı → hızlı. (3) Nonce: 12-byte, her şifrelemede farklı olmalı. (4) Associated Data: şifrelenmemiş ama doğrulanmış veri (header gibi). Production standardı: AES-256-GCM. Nonce reuse: aynı nonce → tam güvenlik kaybı → SecureRandom ile üret.

4. HMAC ile Digital Signature arasındaki temel fark nedir?

   > **Beklened:** HMAC: simetrik. Paylaşılan gizli anahtar. Hem gönderen hem alıcı aynı anahtarı bilir. Kullanım: iki taraf birbirine güveniyor. JWT HS256, GitHub webhook, AWS SigV4. "Birlikte gizli tutulan key" gerekli. Digital Signature: asimetrik. Private key imzalar (sadece sahipte). Public key doğrular (herkes). Kullanım: üçüncü taraf doğrulaması, inkar edilemezlik (non-repudiation). JWT RS256/ES256, TLS sertifika, kod imzalama, blockchain. "Ali bu belgeyi imzaladı" → Ali'nin private key'i → herkes public key ile doğrular. HMAC sorusu: "Ali mi gönderdi?" → belki, ama ortak key'i bilen herkes HMAC üretebilir. DSig sorusu: "Ali mi imzaladı?" → kesinlikle, private key sadece Ali'de. Seçim: internal API → HMAC (performans). Harici doğrulama, notarizasyon → Digital Signature.

---

**Senior / Architect:**

5. Hybrid encryption nedir? TLS'deki key exchange ile ilişkisi nasıldır?

   > **Beklened:** Hybrid encryption: asimetrik + simetrik kombinasyonu. Asimetrik (RSA/ECDH): yavaş, küçük veri → sadece symmetric key'i şifreler. Symmetric (AES): hızlı, büyük veri → gerçek veriyi şifreler. Neden: RSA-2048 ile doğrudan 10MB şifreleme → imkansız (245 byte limit) ve çok yavaş. Çözüm: AES-256 key üret (32 byte) → RSA ile şifrele → gönder. Veriyi AES ile şifrele → gönder. TLS 1.3 bağlantısı: (1) Client Hello: desteklenen cipher suite'ler. (2) Server: sertifika + public key gönder. (3) Key exchange: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral). Her session için yeni key pair → forward secrecy! (4) Shared secret → HKDF → session key (AES-256-GCM). (5) Bundan sonra: AES-256-GCM ile şifreli iletişim. TLS 1.3 RSA key exchange KALDIRILDI (forward secrecy yok). Sadece ECDHE → her session bağımsız → geçmiş session'lar güvende. ECDHE avantajı: RSA-3072 ≈ ECC-256 güvenlik, ECC çok daha hızlı.

6. Argon2id neden bcrypt'ten üstün? Üretimde hangisini seçersin?

   > **Beklened:** bcrypt sınırlamaları: 72 byte limit (uzun şifreler truncate, güvenlik sorunu). GPU optimizasyon: bcrypt bellek hafif → özel ASIC/GPU ile hızlandırılabilir. Argon2id üstünlükleri: (1) Memory-hard: 64MB bellek kullanımı. GPU'nun VRAM'i sınırlı → aynı anda az paralel hesaplama → GPU saldırısı engellenir. (2) Parametreler esnek: memory, iterations, parallelism → donanım güçlendikçe ayarla. (3) 72 byte sınırı yok. (4) Side-channel koruması (Argon2id = Argon2i + Argon2d hybrid). Seçim kriteri: yeni proje → Argon2id (Argon2PasswordEncoder Spring Security). Mevcut sistem → bcrypt geçerliliğini korur, cost≥12 ise dokunma. Migration: login'de mevcut bcrypt doğrula → Argon2id ile re-hash → transparent upgrade. Parametreler (OWASP önerisi): memory=64MB, iterations=3, parallelism=4. Yüksek güvenlik: memory=256MB. Cloud/container: bellek limiti kontrol et — 64MB/request, 10 concurrent login = 640MB sadece hash için. Ayarla.

7. Forward secrecy nedir? TLS'de nasıl sağlanır?

   > **Beklened:** Forward secrecy: gelecekte server private key çalınsa bile, geçmiş session'ların şifresi çözülemez. Neden kritik: NSA/saldırgan tüm TLS trafiğini kaydedebilir. Server private key sonra çalınırsa: RSA key exchange → kaydedilmiş trafiği decrypt edebilir. Geçmiş yıllara ait tüm şifreli trafik açık! TLS 1.2 RSA handshake: client, session key'i server public key ile şifreler → server private key → session key. Private key çalınırsa → tüm geçmiş session'lar açık. ECDHE (Ephemeral Diffie-Hellman): her session için yeni key pair üretilir. Client + server: ephemeral ECDH key pair → shared secret → session key. Session bitti → ephemeral key yok edildi. Private key çalınsa bile: geçmiş ephemeral key'ler yok → decrypt imkansız. TLS 1.3: RSA key exchange tamamen kaldırıldı. Sadece ECDHE veya DHE → forward secrecy zorunlu. Mimariye etkisi: session key'leri saklamama. TLS termination proxy (nginx, ALB) → ECDHE kullandığından emin ol. Cipher suite: ECDHE-* önekli → forward secrecy.

---

## Karma — Architect Seviyesi

8. **"Kriptografide yanlış algorithm seçimleri nasıl tespit edilir ve ne zaman rotasyon yapılır?"**

   > **Beklened:** Tespit: static analysis tool (SpotBugs security plugin, SonarQube security rules): `Cipher.getInstance("AES/ECB/")` → uyarı. `MessageDigest.getInstance("MD5")` → uyarı. `new SecureRandom(seed)` → seed kullanımı uyarı. Dependency track: kullanılan kriptografi kütüphanesi CVE'si. Algorithm audit: kod tabanında grep: `"DES"`, `"RC4"`, `"SHA1"`, `"MD5"` → gözden geçir. Rotasyon ne zaman: (1) Kırık algoritma: SHA-1 collision (2017) → immediate migration. MD5: 10 yıldır kırık → acil. (2) Yeterli güvenlik seviyesi altı: RSA-1024 → 2010'da deprecated. RSA-2048: hâlâ güvenli, 2030+ için RSA-3072 öneriliyor. (3) Quantum computing (Post-Quantum): RSA ve ECC → Shor's algorithm ile kırılabilir. NIST PQC standartları (2024): ML-KEM (Kyber), ML-DSA (Dilithium) → geçiş planı. TLS: CRYSTALS-Kyber hybrid key exchange (X25519+Kyber768). (4) Keşfedilmiş implementation hatası: Heartbleed (OpenSSL) → immediate patch. Rotasyon stratejisi: algorithm agility → konfigürasyon tabanlı (kod değil config). Versioned hash: `$argon2id$v=19$...` → v sonra değişirse → login'de re-hash.

9. **"JWT güvenliği için hangi signing algorithm kullanılmalı? 'none' algoritması ve alg confusion saldırısı ne demek?"**

   > **Beklened:** Algorithm seçimi: HS256 (HMAC-SHA256): paylaşılan secret. Microservice'ler arası dahili kullan. Secret rotation zor. ES256 (ECDSA P-256): asimetrik. Public key herkesin erişebileceği yerde. Private key sadece auth server. Dışarıya açık API, SSO. RS256 (RSA-SHA256): ES256 ile benzer ama daha büyük key. ES256 tercih edilir (hız, boyut). "none" algoritması saldırısı: JWT header: `{"alg": "none"}` → imza yok. Bazı kütüphaneler (eski): `none` → imzayı doğrulama! Saldırgan: kendi payload'ını yazar, alg=none, imzasız gönderir → kabul edildi. Önlem: kütüphanede allowed algorithms explicitly belirt. `jwtParser().requireSignedWith(publicKey)` → unsigned reddedilir. alg confusion saldırısı: Server RS256 kullanıyor (public key biliniyor). Saldırgan: token üretir, `"alg": "HS256"` → secret olarak public key kullanır. Kütüphane RS256 → HS256 olarak interpret ederse → public key ile HMAC doğrulama → public key herkese açık → saldırgan kendi token'ını imzalayabilir. Önlem: `jwtParser().verifyWith(publicKey).build()` — explicit key türü. Algoritma listesi sadece: `["ES256"]` — tek algoritma. Asla: `["RS256", "HS256"]` → confusion mümkün.
