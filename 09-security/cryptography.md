# 09c — Kriptografi Temelleri (Architect için)

## Temel Kavramlar

```
Hashing:          Tek yönlü. Geri döndürülemez. Veri bütünlüğü.
                  SHA-256("data") → "2cf24d..."
                  Şifre saklama, checksum, imza

Encryption:       İki yönlü. Anahtar ile şifrele/çöz. Veriyi gizle.
Symmetric:        Aynı anahtar ile şifrele ve çöz (AES)
Asymmetric:       Public key şifreler, private key çözer (RSA, ECC)

MAC / HMAC:       Mesaj bütünlüğü + kimlik doğrulama.
                  HMAC-SHA256(key, message) → kimden geldiğini kanıtlar

Digital Signature: Private key ile imzala, public key ile doğrula.
                  RSA-SHA256, ECDSA
```

---

## Simetrik Şifreleme: AES

### Ne?
Advanced Encryption Standard. Aynı anahtar şifreler ve çözer. Hızlı, blok şifresi.

### Nasıl Çalışır?

```
Blok boyutu: 128 bit (16 byte) — sabit
Anahtar: 128, 192 veya 256 bit

Modlar:
  ECB (Electronic Codebook): HER ZAMAN YANLIŞ
    Aynı plaintext → aynı ciphertext → pattern ortaya çıkar
    Penguen resmi örneği: ECB ile şifrelenmiş → penguen hâlâ görünür!

  CBC (Cipher Block Chaining): Eski standart
    IV (Initialization Vector) ile başlar
    Her blok önceki ciphertext ile XOR → pattern yok
    Sorun: IV'yi güvenli taşımak gerekir

  GCM (Galois/Counter Mode): Önerilen standart
    Counter mode + authentication tag
    Hem şifreleme hem bütünlük kontrolü (AEAD)
    Paralel işleme (CTR modlu)
    12 byte nonce + 16 byte auth tag

AES-256-GCM = production standardı
```

```java
// AES-256-GCM ile şifreleme
public class AesGcmEncryption {

    public byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
        byte[] nonce = new byte[12]; // 96-bit nonce
        SecureRandom.getInstanceStrong().nextBytes(nonce);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, nonce); // 128-bit auth tag
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);

        byte[] ciphertext = cipher.doFinal(plaintext);

        // nonce + ciphertext birleştir (nonce'u beraberinde sakla)
        ByteBuffer result = ByteBuffer.allocate(nonce.length + ciphertext.length);
        result.put(nonce);
        result.put(ciphertext);
        return result.array();
    }

    public byte[] decrypt(byte[] encryptedData, SecretKey key) throws Exception {
        ByteBuffer buffer = ByteBuffer.wrap(encryptedData);

        byte[] nonce = new byte[12];
        buffer.get(nonce);

        byte[] ciphertext = new byte[buffer.remaining()];
        buffer.get(ciphertext);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(128, nonce));
        return cipher.doFinal(ciphertext); // auth tag doğrulama otomatik
    }

    public SecretKey generateKey() throws Exception {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        return keyGen.generateKey();
    }
}
```

---

## Asimetrik Şifreleme: RSA ve ECC

### RSA

```
Public key: şifrelemek için (herkese açık)
Private key: çözmek için (sadece sahipte)

Math temeli: Büyük sayıları çarpanlarına ayırma çok zor

Boyutlar:
  RSA-2048: minimum (eski sistemler)
  RSA-4096: daha güvenli, daha yavaş

Kullanım alanları:
  TLS handshake (key exchange)
  JWT RS256 imzası
  Şifreli email (PGP)
  Certificate imzalama (CA → sertifika)

Neden şifreleme için doğrudan kullanılmaz?
  Yavaş (büyük veri için)
  Hybrid encryption:
    AES key → RSA ile şifrele → transfer et
    Veri → AES ile şifrele → transfer et
    Alıcı: RSA private key → AES key → veriyi çöz
```

### ECC (Elliptic Curve Cryptography)

```
RSA'ya göre:
  Daha küçük anahtar boyutu → aynı güvenlik seviyesi
  RSA-3072 ≈ ECC-256 (güvenlik açısından)
  Daha hızlı, daha az CPU, mobile için ideal

Eğriler:
  P-256 (prime256v1, secp256r1): NIST standart, FIPS onaylı
  P-384: daha güvenli
  Curve25519: modern, hızlı, yüksek güvenlik (Signal, WireGuard kullanır)
  secp256k1: Bitcoin kullanır

ECDSA: Elliptic Curve Digital Signature Algorithm
  JWT ES256 imzası için
  TLS 1.3 key exchange için (ECDHE)
```

```java
// ECDSA ile imzala/doğrula
KeyPairGenerator keyGen = KeyPairGenerator.getInstance("EC");
keyGen.initialize(new ECGenParameterSpec("secp256r1"));
KeyPair keyPair = keyGen.generateKeyPair();

// İmzala (private key)
Signature signer = Signature.getInstance("SHA256withECDSA");
signer.initSign(keyPair.getPrivate());
signer.update(data);
byte[] signature = signer.sign();

// Doğrula (public key)
Signature verifier = Signature.getInstance("SHA256withECDSA");
verifier.initVerify(keyPair.getPublic());
verifier.update(data);
boolean valid = verifier.verify(signature);
```

---

## Hashing: SHA-256 ve Türevleri

```
SHA-256 özellikleri:
  Tek yönlü (geri döndürülemez)
  Deterministic: aynı input → aynı output
  Avalanche effect: 1 bit değişiklik → tamamen farklı hash
  Collision resistant: aynı hash'e iki farklı input bulmak imkansız (pratikte)
  Sabit boyut: her zaman 256 bit = 32 byte

SHA ailesi:
  SHA-1 (160 bit): KIRILI — kullanma
  SHA-256 (256 bit): Güvenli, genel amaç
  SHA-384, SHA-512: Daha güvenli, daha yavaş
  SHA-3 (Keccak): Farklı yapı, güvenli alternatif

Kullanım:
  Dosya bütünlüğü: checksum (MD5 kullanma — kırık!)
  Git commit ID: SHA-1 (collision saldırısı mevcut — GitHub SHA-256'ya geçiyor)
  JWT payload imzası: SHA-256 (RS256, HS256, ES256)
  TLS certificate fingerprint
  Blockchain: SHA-256 (Bitcoin proof-of-work)
```

```java
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] hash = digest.digest("data".getBytes(StandardCharsets.UTF_8));
String hexHash = HexFormat.of().formatHex(hash);
```

---

## Şifre Saklama: bcrypt, Argon2, scrypt

### Neden SHA-256 Yetmez?

```
Problem: SHA-256 çok hızlı
  GPU: 10 milyar SHA-256/sn
  Leaked DB'den şifre kırmak: saniyeler

Çözüm: Kasıtlı yavaş hash (key derivation function)
  bcrypt: 100ms → GPU ile 10,000 hash/sn (milyar yerine!)
```

### bcrypt

```
Özellikler:
  Work factor (cost): 2^cost iterasyon
  cost=12: ~250ms (önerilen production değeri)
  Dahili salt: her hash'e random 128-bit salt eklenir
  Hash format: $2a$12$[22 char salt][31 char hash]

  Avantaj: çok yaygın, battle-tested, salt dahil
  Dezavantaj: 72 byte limit (uzun şifreler kısalır), GPU optimizasyonuna açık

Spring Security:
```

```java
// Spring Security BCryptPasswordEncoder
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost factor 12
}

// Kullanım
String hash = encoder.encode("userPassword");
boolean matches = encoder.matches("userPassword", hash);
// Hash'ten şifre kurtarılamaz — sadece matches() ile doğrulama
```

### Argon2 (Önerilen — 2023+)

```
Password Hashing Competition 2015 galibi
3 varyant:
  Argon2i:  Side-channel saldırılara karşı (CPU güvenliği)
  Argon2d:  GPU saldırılarına karşı (crypto)
  Argon2id: Hybrid — önerilen genel kullanım

Parametreler:
  memory: MB cinsinden bellek kullanımı (GPU'nun çok RAM'i yok!)
  iterations: iterasyon sayısı
  parallelism: paralel thread sayısı

Önerilen: memory=64MB, iterations=3, parallelism=4
```

```java
// Spring Security Argon2PasswordEncoder
@Bean
PasswordEncoder passwordEncoder() {
    // saltLength=16, hashLength=32, parallelism=1, memory=65536(64MB), iterations=3
    return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
}
```

### scrypt

```
Hem CPU hem bellek yoğun (Litecoin kullanır)
Argon2 öncesinin standardıydı
Parametreler: N (CPU), r (bellek), p (parallelism)

Argon2id genelde tercih edilir ama scrypt de geçerli.
```

### Şifre Saklama Kural Özeti

```
ASLA YAPMA:
  ✗ Plaintext şifre sakla
  ✗ MD5 veya SHA-256 ile şifre hashle
  ✗ Kendi hash algoritmanı yaz
  ✗ Salt'sız hash kullan

DOĞRU:
  ✓ bcrypt (cost ≥ 12) → legacy, yaygın destek
  ✓ Argon2id → modern, önerilen
  ✓ Her şifre için unique salt (bcrypt/Argon2 dahil halleder)
  ✓ Upgrade path: eski bcrypt → login'de Argon2'ye upgrade et
  ✓ Pepper: ek secret key (DB çalınsa bile + pepper gerekiyor)

Pepper pattern:
  finalHash = bcrypt(password + pepper_from_env)
  DB çalınsa bile pepper olmadan kırılamaz
```

---

## HMAC (Hash-based Message Authentication Code)

```
Amaç: Mesajın hem bütünlüğünü hem kaynağını doğrula

Nasıl: HMAC = Hash(secret_key || message)

Kullanım:
  JWT HS256 imzası
  Webhook doğrulama (GitHub, Stripe)
  API imzalama (AWS Signature v4)
  Cookie imzalama

HMAC vs Digital Signature:
  HMAC: paylaşılan secret key (simetrik) → sadece ikisi de doğrulayabilir
  DSig: private key imzalar, public key doğrular (asimetrik) → herkes doğrulayabilir
```

```java
// HMAC-SHA256 örneği (Webhook doğrulama)
Mac mac = Mac.getInstance("HmacSHA256");
SecretKeySpec secretKey = new SecretKeySpec(
    webhookSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
mac.init(secretKey);
byte[] hmac = mac.doFinal(requestBody.getBytes(StandardCharsets.UTF_8));
String expectedSignature = "sha256=" + HexFormat.of().formatHex(hmac);

// Timing-safe karşılaştırma (timing attack önlemi)
boolean valid = MessageDigest.isEqual(
    expectedSignature.getBytes(), 
    receivedSignature.getBytes());
```

---

## Trade-off Özeti

| Algoritma | Tip | Kullanım | Güvenlik Durumu |
|-----------|-----|----------|-----------------|
| AES-256-GCM | Symmetric encryption | Veri şifreleme, DB field | Güvenli |
| RSA-2048+ | Asymmetric | TLS, JWT RS256 | Güvenli (2048 minimum) |
| ECC P-256 | Asymmetric | TLS, JWT ES256 | Güvenli, hızlı |
| SHA-256 | Hash | Checksum, JWT payload | Güvenli |
| SHA-1 | Hash | --- | KIRILI — kullanma |
| MD5 | Hash | --- | KIRILI — kullanma |
| bcrypt (cost≥12) | Password hash | Şifre saklama | Güvenli |
| Argon2id | Password hash | Şifre saklama | Güvenli (önerilen) |
| HMAC-SHA256 | MAC | Webhook, API signing | Güvenli |
