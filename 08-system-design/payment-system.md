# 08j — System Design: Ödeme Sistemi

## Gereksinimler

```
Functional:
  ✓ Ödeme başlatma (kart, banka havalesi, cüzdan)
  ✓ Ödeme durumu sorgulama
  ✓ İade (refund — tam ve kısmi)
  ✓ Çift ödeme önleme (idempotency)
  ✓ Ödeme geçmişi
  ✓ Risk/fraud kontrolü
  ✓ Multi-currency desteği
  ✓ Webhook alma ve doğrulama
  ✓ Chargeback yönetimi
  ✓ Mutabakat (reconciliation)
  ✓ Merchant payout

Non-Functional:
  QPS: 1,000 ödeme/sn (peak)
  Latency: < 3s (ödeme tamamlanma), < 500ms (API response)
  Availability: 99.999% (five nines) — yılda max 5.26 dk downtime
  Veri kaybı: sıfır (ACID garantisi şart)
  Audit log: tüm işlemler kayıt altında, 7 yıl saklanacak
  Compliance: PCI DSS Level 1, PSD2 (Avrupa), TCMB (Türkiye)
  Consistency: strong (para miktarı yanlış hesaplanmamalı)
```

---

## Capacity Estimation

```
İşlem hacmi:
  1,000 ödeme/sn peak → günde: 1,000 × 86,400 = 86.4M ödeme
  Ortalama işlem tutarı: 150 TL → günlük ciro: 12.9B TL

Storage:
  Ödeme kaydı: ~2 KB (metadata + state history)
  Günlük: 86.4M × 2 KB = 172.8 GB/gün
  Audit log (her event): 86.4M × 6 event × 500B = 259 GB/gün
  Ledger entries: 86.4M × 3 entry × 256B = 66 GB/gün
  7 yıl toplam: (172.8 + 259 + 66) × 365 × 7 ≈ 1.3 PB → arşiv şart

Idempotency key storage:
  86.4M × 64 byte = 5.5 GB/gün
  7 gün retention: 38 GB → Redis Cluster

Risk engine:
  Her ödeme → risk feature vector → ML inference: < 50ms
  1,000 ödeme/sn → 1,000 inference/sn → GPU inference cluster

Webhook storage:
  86.4M ödeme → ortalama 3 webhook event/ödeme = 259M webhook/gün
  Her webhook: 2 KB → 518 GB/gün → Kafka topic, 30 gün retention
```

---

## High-Level Mimari

```
         E-commerce / Mobile App
                │ POST /payments (Idempotency-Key header)
                ▼
    ┌────────────────────────┐
    │     Payment Gateway    │  ← TLS termination, auth, rate limit
    └────────────┬───────────┘
                 │
    ┌────────────▼───────────┐
    │    Payment Service     │
    │  - Idempotency check   │
    │  - Validation          │
    │  - 3DS orchestration   │
    │  - State machine       │
    └──────┬─────────┬───────┘
           │         │
    ┌──────▼──┐  ┌───▼──────────┐
    │  Risk   │  │  PSP Adapter │  ← Stripe / Adyen / iyzico soyutlama
    │ Engine  │  │  (circuit    │
    │ (ML+    │  │   breaker)   │
    │  rules) │  └───┬──────────┘
    └──────────┘     │ HTTPS
                 [Stripe/Adyen/iyzico]
                     │ Webhook
                     ▼
    ┌────────────────────────┐
    │   Webhook Service      │  ← HMAC verify, idempotent handler
    └────────────┬───────────┘
                 │
         Kafka "payment-events"
                 │
    ┌────────────┼─────────────┬──────────────┐
    ▼            ▼             ▼              ▼
[Payment DB] [Ledger DB]  [Notification] [Reconciliation]
(PostgreSQL)  (PostgreSQL)               (Batch)

Write path:
  Payment Service → PostgreSQL (ACID) + Outbox → Kafka → downstream
  
Query path:
  Ödeme geçmişi: Payment DB (son 90 gün) veya Data Warehouse (eski)
  Anlık durum: Redis cache (TTL 5dk) + DB fallback
```

---

## Ödeme Akışı (Kart Ödemesi)

```
1. Client → POST /payments
   Header: Idempotency-Key: "uuid-123"
   Body: {amount: 99.99, currency: "TRY", card_token: "tok_xxx", orderId: "ord-456"}

2. Payment Service:
   a. Idempotency check → Redis → yeni istek
   b. Validasyon: tutar > 0, desteklenen para birimi, token geçerli
   c. Risk Engine: fraud score hesapla → threshold geç → devam
   d. DB: payment kaydı INSERT (status: INITIATED)
   e. 3DS gerekli mi? → evet → challenge URL döndür

3. (3DS varsa) Client → banka sayfasına yönlendir → doğrulama
   → webhook / redirect ile onay

4. PSP API'ye gönder:
   Stripe: POST /v1/payment_intents → {status: "processing", id: "pi_xxx"}
   DB: status → PROCESSING

5. PSP Webhook: POST /webhooks/stripe
   {type: "payment_intent.succeeded", id: "pi_xxx"}
   Webhook Service: imza doğrula → idempotent handler
   DB: status → COMPLETED
   Ledger: double-entry kayıt
   Kafka: PaymentCompleted event yayınla

6. Order Service (Kafka consumer):
   PaymentCompleted → sipariş onayla → stok düş → kargo başlat
```

---

## İdempotency (Çift Ödeme Önleme)

```
Problem:
  Client → POST /payments → Timeout (5s)
  Client: "Ödeme oldu mu?" bilmiyorum → RETRY
  Sunucu: İki kez işledi → iki ödeme çekildi!

Çözüm katmanları:

Katman 1 — Redis (hızlı):
  Key: idempotency:{idempotencyKey}
  SETNX → yok → ilk istek → işle.
  EXISTS → var → önbelleğe alınmış sonucu döndür.
  TTL: 24 saat.

Katman 2 — DB (kalıcı):
  payments tablosuna idempotency_key UNIQUE constraint.
  INSERT → conflict → daha önce eklendi → SELECT ile döndür.
  Redis down olsa da DB'den koruma.

Katman 3 — PSP idempotency:
  Stripe: Idempotency-Key header ile API çağrısı.
  Aynı key → Stripe aynı sonucu döndürür (provider düzeyinde).
```

```java
@PostMapping("/payments")
ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest req) {

    // Katman 1: Redis hızlı check
    String cacheKey = "idempotency:" + idempotencyKey;
    String cached = redis.get(cacheKey);
    if (cached != null) {
        return ResponseEntity.ok(deserialize(cached, PaymentResponse.class));
    }

    // Eş zamanlı istekler için mutex (aynı key, iki paralel istek)
    boolean locked = redis.setNX("idem-lock:" + idempotencyKey, "1", 30, TimeUnit.SECONDS);
    if (!locked) {
        return ResponseEntity.status(429).body(PaymentResponse.processing());
    }

    try {
        // Katman 2: DB insert (unique constraint korur)
        Payment payment = paymentRepo.insertOrGet(Payment.builder()
            .idempotencyKey(idempotencyKey)
            .amount(req.getAmount())
            .currency(req.getCurrency())
            .status(PaymentStatus.INITIATED)
            .build());

        if (payment.getStatus() != PaymentStatus.INITIATED) {
            // Daha önce işlenmiş
            return ResponseEntity.ok(PaymentResponse.from(payment));
        }

        // Katman 3: PSP'ye idempotency key ile gönder
        PspResponse pspResp = pspAdapter.charge(req, idempotencyKey);

        PaymentResponse response = processAndSave(payment, pspResp);

        // Cache'e al
        redis.setEx(cacheKey, serialize(response), 86400);
        return ResponseEntity.status(201).body(response);
    } finally {
        redis.delete("idem-lock:" + idempotencyKey);
    }
}
```

---

## Risk Engine / Fraud Detection

```
Her ödemede çalışır, karar: APPROVE / REVIEW / REJECT.
Hedef: < 50ms (ödeme akışı bloke etmemeli).

İki katman:

Katman 1 — Kural Motoru (deterministik, hızlı < 5ms):
  Velocity checks (Redis Counter):
    Son 1 saatte aynı kart → 5+ farklı tutar → şüpheli.
    Aynı IP, son 10 dakika → 3+ farklı kart → şüpheli.
    Yeni hesap (< 24 saat) → yüksek tutar → review.
  
  Blacklist:
    Card BIN (ilk 6 hane) blacklist → anında REJECT.
    IP/cihaz parmakizi blacklist.
    Email domain blacklist (geçici email servisleri).
  
  Velocity rule örnekleri:
    INCR velocity:card:{cardHash}:1h → > 5 → flag
    INCR velocity:ip:{ip}:10m → > 3 → flag
    INCR velocity:user:{userId}:24h:amount → > 10,000 TL → review

Katman 2 — ML Modeli (istatistiksel, 10-50ms):
  Feature vector:
    - amount, currency, time_of_day, day_of_week
    - user_account_age_days
    - card_country vs billing_country uyumsuz mu?
    - device_fingerprint (daha önce görüldü mü?)
    - shipping_address vs billing_address
    - son 30 günde kullanıcı için ortalama işlem tutarı (mean, std)
    - geolocation: IP → ülke vs kart ülkesi
    - is_new_device (ilk kez bu cihazdan)
  
  Model: Gradient Boosting (XGBoost) veya Neural Network.
  Çıkış: fraud_score 0.0-1.0.
    < 0.3: APPROVE.
    0.3-0.7: REVIEW (manüel inceleme veya 3DS zorunlu).
    > 0.7: REJECT.
  
  Günlük model yenileme: dünün chargeback + confirmed fraud → yeniden eğit.

Fraud kararı entegrasyonu:
  APPROVE → Payment flow devam.
  REVIEW → 3DS challenge tetikle veya kullanıcı doğrulama.
  REJECT → Ödeme reddedildi, genel hata mesajı (ayrıntı verme → fraud'a bilgi sızdırma).

  risk_decisions tablosuna her karar kaydet (audit + model feedback).
```

---

## 3D Secure / Strong Customer Authentication (SCA)

```
PSD2 (Avrupa Ödeme Direktifi 2): Avrupa'da zorunlu.
Amaç: "Bu kartı kullanan kişi gerçekten kart sahibi mi?"

3DS akışı:
  1. Ödeme servisi: Stripe'a kart token + müşteri bilgisi → risk skoru.
  2. Stripe: Kart issuing bankasına sor → challenge gerekli mi?
  3. Challenge gerekli:
     Stripe → {status: "requires_action", next_action: {redirect_to_url: "https://..."}}
     Client: kullanıcıyı banka sayfasına yönlendir.
     Kullanıcı: SMS kodu, biyometrik, push notification onay.
     Banka: onayladı → redirect back → Payment Service'e bildir.
  4. Challenge gerekmedi (Frictionless):
     Düşük riskli işlem → banka otomatik onay → kullanıcıya müdahale yok.

3DS entegrasyonu (Stripe):
  payment_intent.status = "requires_action"
  → client_secret → frontend Stripe.js → handleNextAction()
  → kullanıcı banka doğrulamasını yaptı
  → payment_intent.status = "succeeded"

Exemptions (muafiyet):
  Düşük değerli işlem (< 30 EUR): 3DS atla.
  Güvenilir alıcı (daha önce doğrulandı): atla.
  Merchant-initiated (abonelik): kullanıcı zaten onayladı, tekrar sorma.
  Risk Tabanlı Muafiyet: banka düşük risk kararı → frictionless.

Liability shift:
  3DS ile ödeme onaylandı → chargeback geldi → sorumluluk issuing bank'ta.
  3DS olmadan → sorumluluk merchant'ta (sen ödersin).
  Bu yüzden 3DS kritik: finansal risk transferi.
```

---

## Ödeme State Machine

```
                 ┌─────────────┐
                 │  INITIATED  │
                 └──────┬──────┘
                        │ Validasyon OK
                 ┌──────▼──────┐
                 │  VALIDATED  │
                 └──────┬──────┘
                        │ Risk APPROVE
                 ┌──────▼──────┐
                 │RISK_APPROVED│
                 └──────┬──────┘
                        │ PSP'ye gönder
           ┌────────────▼─────────────┐
           │        PROCESSING        │
           └────┬─────────────┬───────┘
                │             │
         ┌──────▼──┐    ┌──────▼──────┐
         │COMPLETED│    │   FAILED    │
         └──┬──────┘    └─────────────┘
            │
     ┌──────▼──────┐
     │  REFUND_    │
     │  REQUESTED  │
     └──────┬──────┘
            │
     ┌──────▼──────┐        ┌──────────────┐
     │  REFUND_    │        │CHARGEBACK_   │
     │  PROCESSING │        │RECEIVED      │
     └──────┬──────┘        └──────────────┘
            │
    ┌────────┼────────┐
    ▼        ▼        ▼
[REFUNDED][PARTIAL_ [REFUND_
           REFUNDED] FAILED]

Her state değişikliği:
  → DB'ye kaydet (payment_events tablosu, immutable log)
  → Outbox → Kafka event yayınla
  → Audit log
```

```java
@Service
@Transactional
class PaymentStateMachine {

    private static final Map<PaymentStatus, Set<PaymentStatus>> VALID_TRANSITIONS = Map.of(
        INITIATED,      Set.of(VALIDATED, FAILED, CANCELLED),
        VALIDATED,      Set.of(RISK_APPROVED, RISK_REJECTED, FAILED),
        RISK_APPROVED,  Set.of(PROCESSING, FAILED),
        PROCESSING,     Set.of(COMPLETED, FAILED, REQUIRES_ACTION),
        REQUIRES_ACTION,Set.of(PROCESSING, FAILED, CANCELLED),
        COMPLETED,      Set.of(REFUND_REQUESTED, CHARGEBACK_RECEIVED),
        REFUND_REQUESTED, Set.of(REFUND_PROCESSING),
        REFUND_PROCESSING, Set.of(REFUNDED, PARTIAL_REFUNDED, REFUND_FAILED)
    );

    void transition(Payment payment, PaymentStatus newStatus, String reason) {
        Set<PaymentStatus> allowed = VALID_TRANSITIONS.getOrDefault(
            payment.getStatus(), Set.of());
        if (!allowed.contains(newStatus)) {
            throw new InvalidStateTransitionException(
                payment.getStatus() + " → " + newStatus + " geçersiz");
        }

        PaymentStatus oldStatus = payment.getStatus();
        payment.setStatus(newStatus);
        payment.setUpdatedAt(Instant.now());

        // Immutable event log (append-only)
        paymentEventRepo.save(PaymentEvent.builder()
            .paymentId(payment.getId())
            .fromStatus(oldStatus)
            .toStatus(newStatus)
            .reason(reason)
            .occurredAt(Instant.now())
            .build());

        paymentRepo.save(payment);

        // Outbox pattern: event atomik olarak DB'ye yaz
        outboxRepo.save(OutboxEvent.of(
            "PaymentStatusChanged", payment.getId().toString(),
            new PaymentStatusChangedEvent(payment.getId(), oldStatus, newStatus)));
    }
}
```

---

## Double-Entry Ledger (Muhasebe)

```
DEBITS = CREDITS kuralı her zaman geçerli (double-entry accounting).

Ödeme: Kullanıcı 100 TL ödedi (%3 komisyon):
  DEBIT:  accounts_receivable:alice        100.00 TL  (kullanıcıdan alınacak)
  CREDIT: accounts_payable:bob              97.00 TL  (merchant'a ödenecek)
  CREDIT: revenue:platform_fee              3.00 TL   (komisyon)
  ─────────────────────────────────────────────────
  SUM: 100.00 - 97.00 - 3.00 = 0.00 ✓ (balanced)

Tam iade: 100 TL iade
  DEBIT:  accounts_payable:bob              97.00 TL
  DEBIT:  revenue:platform_fee               3.00 TL  (komisyon iade — politikaya göre)
  CREDIT: accounts_receivable:alice         100.00 TL
  ─────────────────────────────────────────────────
  SUM: 97.00 + 3.00 - 100.00 = 0.00 ✓

Kısmi iade: 40 TL iade
  DEBIT:  accounts_payable:bob              38.80 TL  (40 × 0.97)
  DEBIT:  revenue:platform_fee               1.20 TL  (40 × 0.03)
  CREDIT: accounts_receivable:alice          40.00 TL
  ─────────────────────────────────────────────────
  SUM: 0.00 ✓

INVARIANT: SUM(DEBIT) - SUM(CREDIT) = 0 her zaman.
  Günlük kontrol: bu sıfırdan farklıysa → kritik muhasebe hatası → alarm!

Neden ledger tablosu?
  Tek "balance" kolonu: UPDATE users SET balance = balance - 100 → concurrent race condition.
  Ledger: append-only → race condition yok, tam audit trail, zaman yolculuğu (herhangi anda bakiye).
```

```sql
CREATE TABLE ledger_entries (
    entry_id      UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id    UUID          NOT NULL,
    account_id    VARCHAR(100)  NOT NULL,    -- accounts_receivable:alice, revenue:fee
    entry_type    VARCHAR(10)   NOT NULL,    -- DEBIT, CREDIT
    amount        DECIMAL(19,4) NOT NULL,
    currency      CHAR(3)       NOT NULL,
    description   TEXT,
    created_at    TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    CONSTRAINT positive_amount CHECK (amount > 0)
) PARTITION BY RANGE (created_at);  -- aylık partition

-- Anlık bakiye (materialized view — saatte bir yenile)
CREATE MATERIALIZED VIEW account_balances AS
SELECT
    account_id,
    currency,
    SUM(CASE WHEN entry_type = 'CREDIT' THEN amount ELSE -amount END) AS balance,
    MAX(created_at) AS as_of
FROM ledger_entries
GROUP BY account_id, currency;

-- Denge kontrolü (sıfır olmalı)
SELECT SUM(CASE WHEN entry_type = 'DEBIT' THEN amount ELSE -amount END) AS net
FROM ledger_entries
WHERE payment_id = 'pay-123';
-- 0.0000 → balanced ✓

-- Günlük denge kontrolü (scheduled job):
SELECT DATE(created_at), SUM(CASE WHEN entry_type='DEBIT' THEN amount ELSE -amount END)
FROM ledger_entries
GROUP BY DATE(created_at)
HAVING ABS(SUM(CASE WHEN entry_type='DEBIT' THEN amount ELSE -amount END)) > 0.0001;
-- Sonuç dolu → kritik alarm!
```

---

## PSP Soyutlama (Adapter Pattern)

```
Problem: Stripe kullanyorsun → Adyen'e geçmek istiyorsun → her yerde Stripe kodu var.
Çözüm: PSP Adapter → interface arkasında her provider gizli.

interface PspAdapter {
    ChargeResult charge(ChargeRequest req, String idempotencyKey);
    RefundResult refund(String transactionId, BigDecimal amount);
    PaymentStatus getStatus(String transactionId);
}

class StripeAdapter implements PspAdapter {
    ChargeResult charge(ChargeRequest req, String idempotencyKey) {
        PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
            .setAmount(req.getAmountCents())
            .setCurrency(req.getCurrency().toLowerCase())
            .setPaymentMethod(req.getPaymentMethodId())
            .setConfirm(true)
            .setIdempotencyKey(idempotencyKey)
            .build();
        
        PaymentIntent intent = PaymentIntent.create(params,
            RequestOptions.builder().setIdempotencyKey(idempotencyKey).build());
        
        return ChargeResult.builder()
            .transactionId(intent.getId())
            .status(mapStatus(intent.getStatus()))
            .requiresAction("requires_action".equals(intent.getStatus()))
            .clientSecret(intent.getClientSecret())
            .build();
    }
}

class AdyenAdapter implements PspAdapter { ... }
class IyzicoAdapter implements PspAdapter { ... }

PSP yönlendirme:
  Ülke = "TR" → iyzico (düşük komisyon, yerel destek).
  Ülke = "DE" → Stripe veya Adyen.
  High risk → Adyen (daha iyi fraud araçları).
  Failover: Stripe down → Adyen'e yönlendir.

Circuit Breaker (Resilience4j):
  Stripe → 5 ardışık hata → circuit OPEN.
  Yeni ödemeler → Adyen'e yönlendir.
  30sn sonra HALF-OPEN → 1 deneme → başarılı → CLOSED.
```

---

## Multi-Currency ve Yuvarlama

```
Problem: 0.1 + 0.2 = 0.30000000000000004 (IEEE 754 floating point)
Çözüm: Tüm tutarları en küçük birim (cent/kuruş) olarak integer sakla.
  99.99 TRY → 9999 kuruş (Integer/Long).
  DB: DECIMAL(19,4) — asla FLOAT, DOUBLE değil!

Döviz çevrimi:
  FX rate: her 5 dakikada bir ECB/TCMB'den çek → Redis'e yaz.
  Kullanıcı: "EUR'la öde" → mevcut FX rate ile TRY'ye çevir → PSP'ye TRY gönder.
  Saklama: her zaman orijinal currency + orijinal tutar + çevrim kuru + çevrilmiş tutar.

Yuvarlama kuralı:
  Banker's rounding (HALF_EVEN): 2.5 → 2, 3.5 → 4 (istatistiksel tarafsızlık).
  Java: new BigDecimal("99.999").setScale(2, RoundingMode.HALF_EVEN) = 100.00
  Her yuvarlama noktası → kayıt (audit için).

FX risk:
  Kullanıcı EUR öder → biz TRY alırız → merchant USD alır.
  Kur değişimi riskimizde (merchant ödemesine kadar).
  Çözüm: anlık çevrim, T+1 settlement → minimum exposure.

Currency isolation:
  Ledger: her entry para birimiyle → farklı currency'ler asla toplanmaz.
  Bakiye: (account_id, currency) çifti → USD ve TRY ayrı satır.
```

---

## Chargeback Yönetimi

```
Chargeback: Kullanıcı bankasına "bu ödemeyi ben yapmadım" dedi → banka geri alıyor.
Süreci:
  1. Banka: kart sahibi şikayeti → Webhook: chargeback.created.
  2. Payment Service: payments.status → CHARGEBACK_RECEIVED.
  3. Merchant: kanıt toplama süresi (genellikle 7-14 gün).
  4. Kanıt gönder: teslimat kanıtı, kullanım logu, iletişim geçmişi.
  5. Banka kararı: merchant lehine → ödeme geri döner; kullanıcı lehine → kayıp.

Chargeback types:
  Fraud: "Kartımı çaldılar, ödemeyi ben yapmadım" → en yaygın.
  Item not received: ürün gelmedi (merchant anlaşmazlığı).
  Not as described: ürün farklıydı.
  Duplicate: iki kez çekildi (idempotency ile önlenir).

Kanıt otomasyonu:
  Chargeback geldi → otomatik kanıt paketi oluştur:
    IP adresi + user agent logu.
    Teslimat onayı (kargo takip).
    Kullanım logu (dijital ürün: ne zaman indirildi).
    3DS doğrulama logu (liability shift!).
  → PSP API'sine yükle → dispute yanıtla.

Chargeback oranı:
  > %1 → Visa/Mastercard: merchant hesabı tehlikede.
  > %2 → Hesap kapatılabilir.
  Monitoring: günlük chargeback_count / total_payment_count.

İyzicoAdapter chargeback:
CREATE TABLE chargebacks (
    chargeback_id    UUID PRIMARY KEY,
    payment_id       UUID REFERENCES payments(payment_id),
    amount           DECIMAL(19,4),
    currency         CHAR(3),
    reason_code      VARCHAR(20),     -- FR (fraud), NR (not received), ND (not described)
    status           VARCHAR(30),     -- RECEIVED, EVIDENCE_SUBMITTED, WON, LOST
    deadline_at      TIMESTAMPTZ,
    evidence_url     TEXT,
    provider_case_id VARCHAR(100),
    created_at       TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Settlement (Hesaplaşma)

```
Settlement: Merchant'ın parasını gerçekten alması.
  Stripe: T+2 (ödeme günü + 2 iş günü) → merchant banka hesabına.
  Adyen: T+1 veya T+0 (anlaşmaya göre).
  iyzico: T+1.

Settlement süreç:
  1. Günlük: tüm COMPLETED ödemeleri toparla (batch job, gece 01:00).
  2. Her merchant için: toplam tutar - komisyon - chargeback reserve.
  3. PSP'den settlement raporu al (CSV/API).
  4. Kendi hesabımızla karşılaştır (reconciliation).
  5. Merchant banka hesabına transfer emri (EFT/SWIFT).
  6. Ledger: settlement entry yaz.

Rolling reserve:
  Yüksek riskli merchant: ödemelerin %10'unu 90 gün tut.
  Chargeback gelirse → reserve'den karşıla → merchant'a dönme.
  90 gün sonra sorun yoksa → reserve serbest bırak.

Settlement DB:
CREATE TABLE settlement_batches (
    batch_id        UUID PRIMARY KEY,
    merchant_id     UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    gross_amount    DECIMAL(19,4),
    fee_amount      DECIMAL(19,4),
    refund_amount   DECIMAL(19,4),
    chargeback_amount DECIMAL(19,4),
    net_amount      DECIMAL(19,4),
    currency        CHAR(3),
    status          VARCHAR(20),   -- PENDING, PROCESSING, SETTLED, FAILED
    settled_at      TIMESTAMPTZ,
    bank_reference  VARCHAR(100)
);
```

---

## Reconciliation (Mutabakat)

```
Sorun: Payment Service DB ile PSP DB tutarsız olabilir.
  Network hatası → timeout → bizde PROCESSING, PSP'de COMPLETED.
  Bug → ledger kaydı eksik → finansal kayıp.

Günlük mutabakat (batch job, sabah 06:00):
  1. PSP'den transaction listesi al (önceki gün, CSV/API).
  2. Kendi DB'deki kayıtlarla JOIN.
  3. Tutarsızlık tipleri:
     a. Bizde var, PSP'de yok → araştır (ağ hatası, bug).
     b. PSP'de var, bizde yok → eksik kayıt oluştur + alert.
     c. Tutar farklı → kritik → dondur + finance ekibini uyar.
     d. Durum farklı → bizde PROCESSING, PSP'de COMPLETED → güncelle.
  4. Reconciliation raporu → finance dashboard.

Otomatik düzeltme vs manuel:
  Durum farkı (PROCESSING → COMPLETED): güvenli → otomatik düzelt.
  Tutar farkı: kritik → otomatik ASLA → finance onayı → düzelt.

Anlık reconciliation (webhook tabanlı):
  Her webhook: PSP durumu → bizim durumumuzla karşılaştır → fark varsa düzelt.
  Günlük batch: kalan eksiklikleri yakala.

Stuck ödemeleri yakala:
  Scheduler: her 5 dakika → PROCESSING durumunda 15 dakikayı geçen ödemeler.
  → PSP API'ye sorgula: GET /payment_intents/{id}
  → Gerçek durum → güncelle.

CREATE TABLE reconciliation_runs (
    run_id          UUID PRIMARY KEY,
    provider        VARCHAR(20),
    period_date     DATE,
    total_provider  INT,
    total_ours      INT,
    matched         INT,
    discrepancies   INT,
    status          VARCHAR(20),  -- RUNNING, COMPLETED, FAILED
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

CREATE TABLE reconciliation_discrepancies (
    id              UUID PRIMARY KEY,
    run_id          UUID REFERENCES reconciliation_runs(run_id),
    payment_id      UUID,
    provider_txn_id VARCHAR(100),
    issue_type      VARCHAR(50),  -- MISSING_IN_OURS, MISSING_IN_PROVIDER, AMOUNT_MISMATCH, STATUS_MISMATCH
    our_amount      DECIMAL(19,4),
    provider_amount DECIMAL(19,4),
    our_status      VARCHAR(30),
    provider_status VARCHAR(30),
    resolved        BOOLEAN DEFAULT FALSE,
    resolved_by     UUID,
    resolved_at     TIMESTAMPTZ
);
```

---

## PCI DSS Uyumu (Kart Güvenliği)

```
PCI DSS Level 1: 6M+ transaction/yıl → yıllık QSA denetimi zorunlu.

Kart verisi (CHD — Cardholder Data):
  PAN (Primary Account Number = kart numarası): en hassas.
  CVV/CVV2: ASLA saklanmaz (ödeme sonrası bile).
  Expiry date: şifreli saklana bilir.
  Cardholder name: şifreli.

Pratik yaklaşım — Tokenization:
  Kart numarasını hiç görmeme: Stripe.js, Adyen Web Components.
  Client: kart formunu doğrudan Stripe/Adyen'e gönderir (backend bypass).
  Backend: sadece token alır → token ile charge eder.
  PCI scope: minimal (SAQ A seviyesi).

Network Tokenization (Visa/Mastercard):
  Gerçek PAN → Visa Token Service (VTS) → network token.
  Network token: sadece bu merchant için geçerli → çalınsa işe yaramaz.
  Otomatik token refresh: kart yenilenince token güncellenir (kullanıcı kart numarası girmez).

Kendi tokenization sistemi (gerekirse):
  Token Vault: ayrı, izole veritabanı.
  PAN → AES-256-GCM şifreleme → token (rastgele UUID).
  Token Vault → ayrı HSM (Hardware Security Module) ile anahtar yönetimi.
  PCI scope: sadece Token Vault → en küçük saldırı yüzeyi.

Zorunlu kontroller:
  Transit: TLS 1.2+ (1.3 tercihli).
  At-rest: AES-256 (kart verisi, PII).
  Erişim: minimum privilege → sadece yetkili servis.
  Log: kart datasına erişim → audit log (kim, ne zaman, hangi kart).
  Ağ segmentasyonu: kart verisi sistemi → ayrı subnet, WAF önünde.
  Penetration test: yılda en az 1 kez.
```

---

## Veri Modeli

```sql
CREATE TABLE payments (
    payment_id       UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id         UUID,
    user_id          UUID          NOT NULL,
    merchant_id      UUID          NOT NULL,
    idempotency_key  VARCHAR(255)  UNIQUE NOT NULL,
    amount           DECIMAL(19,4) NOT NULL,
    currency         CHAR(3)       NOT NULL,
    -- Döviz çevrimi (varsa)
    original_amount  DECIMAL(19,4),
    original_currency CHAR(3),
    fx_rate          DECIMAL(10,6),
    status           VARCHAR(30)   NOT NULL DEFAULT 'INITIATED',
    payment_method   VARCHAR(20)   NOT NULL,  -- CARD, BANK_TRANSFER, WALLET
    card_token       VARCHAR(255),             -- PSP token (PAN değil)
    card_last4       CHAR(4),
    card_brand       VARCHAR(20),              -- VISA, MASTERCARD, AMEX
    psp_provider     VARCHAR(20),              -- STRIPE, ADYEN, IYZICO
    psp_transaction_id VARCHAR(100) UNIQUE,
    risk_score       DECIMAL(4,3),
    risk_decision    VARCHAR(20),              -- APPROVE, REVIEW, REJECT
    three_ds_status  VARCHAR(20),              -- NOT_REQUIRED, PASSED, FAILED
    failure_code     VARCHAR(50),
    failure_message  TEXT,
    metadata         JSONB,                    -- esnek ek bilgi
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);  -- aylık partition

-- State machine için immutable event log
CREATE TABLE payment_events (
    event_id    UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id  UUID          NOT NULL REFERENCES payments(payment_id),
    from_status VARCHAR(30),
    to_status   VARCHAR(30)   NOT NULL,
    reason      TEXT,
    actor       VARCHAR(100), -- SYSTEM, USER, WEBHOOK, SCHEDULER
    metadata    JSONB,
    occurred_at TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Indexler
CREATE INDEX idx_payments_user ON payments (user_id, created_at DESC);
CREATE INDEX idx_payments_merchant ON payments (merchant_id, created_at DESC);
CREATE INDEX idx_payments_status ON payments (status) WHERE status IN ('INITIATED','PROCESSING','REQUIRES_ACTION');
CREATE INDEX idx_payments_psp_txn ON payments (psp_transaction_id) WHERE psp_transaction_id IS NOT NULL;
```

---

## Saga Pattern — Sipariş + Ödeme Koordinasyonu

```
Problem: Sipariş oluştur → stok düş → ödeme al → 3. adımda ödeme başarısız.
  Stok düşürüldü, ödeme alınamadı → tutarsız durum.

Çözüm: Saga (Choreography-based):
  Her servis başarılı/başarısız event yayınlar.
  Başarısız → compensating transaction.

Akış:
  1. Order Service: sipariş oluştur → OrderCreated event.
  2. Inventory Service: stok düş → StockReserved event.
  3. Payment Service: ödeme al → PaymentCompleted event.
  4. Order Service: siparişi onayla → OrderConfirmed event.

Başarısız senaryo (ödeme reddedildi):
  Payment Service: PaymentFailed event yayınla.
  Inventory Service: PaymentFailed'ı dinle → stok iade (compensating).
  Order Service: sipariş iptal et → OrderCancelled event.

Compensating transaction:
  Her adım için geri alma işlemi tasarla.
  Stok iade: StockReserved → StockReleased.
  Ödeme iade: gerekmiyor (ödeme zaten başarısız).
  Idempotent: compensating transaction birden fazla kez güvenle çalışabilmeli.

Orchestration-based (alternatif):
  Payment Orchestrator: tüm süreci yönetir.
    1. OrderService.createOrder()
    2. InventoryService.reserveStock()
    3. PaymentService.charge()
    4. Hepsi OK → OrderService.confirm()
    5. Herhangi biri hata → rollback sequence.
  Avantaj: akışı tek yerden görmek kolay.
  Dezavantaj: orchestrator → SPOF, tüm servisleri biliyor (coupling).
```

---

## Olası Sorunlar ve Çözümleri

### 1. Split Brain — Ödeme PSP'de Tamamlandı, DB'de Kayıp

```
Sorun:
  Payment Service: PSP'ye ödeme isteği gönder.
  PSP: ödemeyi işledi (COMPLETED).
  Network: PSP'nin response'u bize ulaşmadan timeout.
  Payment Service: "timeout → FAILED" dedi.
  DB: status = FAILED.
  Gerçek: para müşteriden çekildi, sipariş iptal edildi → çok kötü UX.

Neden olur:
  Network arızası, load balancer timeout, PSP yavaş response.

Çözüm:
  a) İdempotent retry:
     Timeout alındı → hemen FAILED deme → PSP'ye sorgula.
     GET /payment_intents/{psp_transaction_id} → gerçek durumu öğren.
     PSP: COMPLETED → payment'ı COMPLETED yap.

  b) Webhook fallback:
     PSP: ödeme tamamlandı → webhook gönderir (timeout'dan bağımsız).
     Webhook: bizim FAILED kaydımızı COMPLETED'a günceller.
     Webhook gecikirse → polling fallback (5dk, 15dk, 1saat).

  c) Polling worker:
     Scheduler: her 5 dk → PROCESSING durumunda 10 dk geçmiş ödemeler.
     → PSP API sorgula → gerçek durum → güncelle.
     Stuck payment'lar otomatik çözülür.

  d) Atomic state güncellemesi:
     PSP'ye istek gönder → response AL → SONRA status güncelle.
     Response almadan "FAILED" yazma → belirsizlik penceresi.
     2PC değil: timeout = belirsiz, not failure.
```

---

### 2. İdempotency Failure — Para İki Kez Çekildi

```
Sorun:
  Client: POST /payments (Idempotency-Key: "key-123").
  Network hiccup → response gelmiyor.
  Client: 5sn sonra retry → aynı Idempotency-Key ile.
  Redis: idempotency key var → önceki sonuç döndürmeli.
  Ama: Redis'te key yazılmadan önce Redis restart olmuş → key yok.
  Server: "ilk istek" sanıyor → yeniden işliyor → çift ödeme.

Çözüm:
  a) DB katmanı zorunlu:
     Redis sadece cache → DB UNIQUE constraint asıl güvence.
     payments tablosu: idempotency_key UNIQUE.
     INSERT ON CONFLICT (idempotency_key) DO NOTHING → var olanı döndür.

  b) Redis + DB çift katman:
     1. Redis check → miss.
     2. DB insert → conflict → çift ödeme önlendi → mevcut kaydı döndür.
     3. Redis miss olsa bile DB kurtarır.

  c) PSP idempotency key:
     Stripe'a aynı idempotency key ile aynı isteği gönder.
     Stripe: "bu isteği daha önce gördüm" → aynı sonucu döndür.
     Üç katman: Redis + DB + PSP → triangle of protection.

  d) Client idempotency key üretimi:
     Her ödeme isteği için yeni UUID.
     Retry: aynı UUID ile dene.
     Yeni ödeme girişimi: yeni UUID.
     Karıştırma → çift ödeme yapar; UUID kütüphanesi kullan.
```

---

### 3. Fraud Spike — 5 Dakikada 10,000 Sahte Ödeme

```
Sorun:
  Bot saldırısı: çalıntı kart listesiyle otomatik ödeme denemeleri.
  Sistemimiz: her isteği işliyor → PSP rate limit aşıldı.
  PSP: hesabımızı geçici askıya aldı → gerçek ödemeler de bloke.
  Müşteriler: ödeme yapamıyor → revenue kaybı.

Çözüm:
  a) Velocity check (Redis, < 1ms):
     Aynı IP → 10 dk'da 5+ farklı kart → anında REJECT + geçici IP ban.
     Aynı BIN (kart ilk 6 hane) → 1 dk'da 10+ istek → bu BIN'i 1 saat block.
     Aynı cihaz parmakizi → 5 dakika 3+ farklı kart → block.
     Bu kontroller PSP'ye gitmeden → çok hızlı, PSP korunur.

  b) CAPTCHA for suspicious requests:
     Risk score > 0.5 → "İnsan mısın?" doğrulama.
     Bot: CAPTCHA geçemez → bloke.
     Gerçek kullanıcı: geçer → ödeme devam.

  c) Card BIN validation:
     BIN → kart bilgisi API → geçerli mi? → ülke, kart tipi.
     Test BIN'leri (Stripe test kartları) → production'da reddet.

  d) PSP rate limit aşımı:
     Kendi rate limit: 1,000 ödeme/sn → aşınca queue → sıra ile işle.
     Adaptive: hata arttı → rate'i düşür → PSP korunur.
     Ayrı endpoint: test/fraud → production PSP'yi etkilemesin.
```

---

### 4. Currency Rounding Error — Muhasebe Dengesizliği

```
Sorun:
  1,000 ödeme: her biri 99.999 TL (yuvarlama öncesi).
  Sistem A: 100.00 TL yuvarladı → 100 × 1,000 = 100,000 TL.
  Sistem B: 99.99 TL yuvarladı → 99.99 × 1,000 = 99,990 TL.
  Fark: 10 TL → günlük milyonlarca işlemde → yüz binlerce TL kayıp.
  Daha da kötü: DEBIT - CREDIT ≠ 0 → muhasebe dengesizliği.

Çözüm:
  a) BigDecimal her yerde (asla float/double):
     Java: BigDecimal.valueOf("99.999")
     Python: from decimal import Decimal; Decimal("99.999")
     DB: DECIMAL(19,4) — float/double column yasak.
  
  b) Tek yuvarlama noktası:
     Yuvarlama: sadece son adımda (kullanıcıya gösterim, ledger yazımı).
     Ara hesaplamalar: tam precision ile.
     Banker's rounding (HALF_EVEN): istatistiksel tarafsızlık.
  
  c) Yuvarlama farkı kaydı:
     Toplu işlem: 1,000 × 99.999 = 99,999 TL → ledger: 100,000 TL.
     Fark: 1 TL → "rounding_adjustment" ledger entry yaz.
     Bu şekilde DEBIT = CREDIT korunur.

  d) Günlük denge doğrulaması:
     Scheduled job: SUM(DEBIT) - SUM(CREDIT) ≠ 0 → kritik alarm.
     Tolerance: mutlak sıfır (1 kuruş bile hata = alarm).
```

---

### 5. Webhook Gecikti — Ödeme 6 Saat PROCESSING'de Kaldı

```
Sorun:
  PSP: ödemeyi işledi (COMPLETED) → webhook gönderdi.
  Bizim webhook endpoint: o an down/meşgul → webhook başarısız.
  PSP: 1 saat sonra retry → yine başarısız.
  PSP: 6 saat sonra vazgeçti → webhook artık gelmiyor.
  Kullanıcı: ödeme yapıldı ama sipariş "bekliyor" → destek ekibini aradı.

Neden kritik:
  Müşteri para ödedi, ürünü alamadı → churn, şikâyet.

Çözüm:
  a) Webhook endpoint'ini HA yap:
     Load balancer + 3 replica → birisi down → diğerleri karşılar.
     Webhook: Kafka'ya yaz (durable) → anlık işleme başarısız → Kafka'dan tekrar.

  b) Polling fallback (birincil savunma):
     Scheduler: her 5 dk → PROCESSING + 10 dk geçmiş → PSP API sorgula.
     PSP: COMPLETED → güncelle.
     Asla sadece webhook'a güvenme.

  c) PSP'den webhook retry yapılandırması:
     Stripe: 3 gün, 7 retry → exponential backoff.
     Webhook endpoint'im 200 dönene kadar retry.
     Endpoint 200 dönmeli → Kafka'ya yaz → sonra işle (hızlı 200).

  d) Webhook idempotency + async processing:
     Webhook geldi → Redis/DB'ye kaydet → 200 hemen dön.
     Kafka: async işleme → hata olsa bile webhook kaydı var → tekrar işle.
     PSP gereksiz retry yapmaz (200 aldı).
```

---

### 6. Ledger Dengesizliği — SUM(DEBIT) ≠ SUM(CREDIT)

```
Sorun:
  Bug: ödeme COMPLETED → DEBIT yazıldı → exception → CREDIT yazılmadı.
  Exception: transaction rollback → DEBIT de geri alındı? Belki, belki değil.
  Partial write durumunda: tek taraflı ledger entry → denge bozuldu.
  Muhasebe: "Para nerede?" → bulunamıyor → audit başlatıldı.

Çözüm:
  a) Atomik ledger yazımı:
     TÜM ledger entries aynı DB transaction'ında INSERT.
     Herhangi biri başarısız → hepsi rollback → kısmi yazma olmaz.
     DEBIT + CREDIT: ya ikisi de var, ya hiçbiri.

  b) Ledger balance check (trigger):
     payment_id başına: SUM(DEBIT) - SUM(CREDIT) = 0 zorunlu.
     DB CHECK constraint değil (toplu) ama application layer:
       void writeLedgerEntries(List<LedgerEntry> entries) {
         BigDecimal net = entries.stream().map(e ->
           e.isDebit() ? e.getAmount() : e.getAmount().negate())
           .reduce(BigDecimal.ZERO, BigDecimal::add);
         if (net.compareTo(BigDecimal.ZERO) != 0)
           throw new UnbalancedLedgerException("Net: " + net);
         ledgerRepo.saveAll(entries);
       }

  c) Günlük denge doğrulaması:
     Her sabah: tüm payment_id'ler için net = 0 kontrolü.
     Hata varsa: o payment_id dondur → manuel inceleme.

  d) Event sourcing yaklaşımı:
     Ledger: append-only event store.
     Hesaplama: event'lerden türetilen projeksiyon.
     Bug: projeksiyon yanlış → event'ler doğru → event'lerden yeniden hesapla.
     Immutability: event silinmez → hata izlenebilir.
```

---

### 7. Settlement Batch Failure — Merchant Günlerce Para Alamadı

```
Sorun:
  Gece 01:00 settlement batch: bir merchant'ın 500 siparişini topluyoruz.
  DB bağlantı hatası → batch yarıda kesildi.
  200 sipariş işlendi, 300 beklemede.
  Sabah: merchant "paramı almadım" → 300 sipariş settle edilmemiş.
  Idempotency: 200 işlenmiş sipariş tekrar işlenirse → çift ödeme.

Çözüm:
  a) Idempotent batch processing:
     Her payment_id için: settlement_batches tablosunda var mı kontrol et.
     Var → skip → sadece işlenmemişleri işle.
     Batch yeniden çalıştırma güvenli.

  b) Checkpointing:
     Batch: her 1,000 kayıt → checkpoint kaydet.
     Restart: son checkpoint'ten devam.
     10M kayıt → hata → baştan değil checkpoint'ten.

  c) Partial settlement:
     Merchant bazlı isolation: her merchant → ayrı batch parçası.
     Bir merchant hatası → sadece o merchant gecikiyor, diğerleri etkilenmiyor.

  d) Settlement raporu + alert:
     Batch bitti: "500/500 başarılı" vs "200/500 başarılı".
     Hata → finance ekibine alert → "settlement_batch_id=X için 300 kayıt manuel onay gerekiyor."
     Manual retry: sadece başarısız kayıtlar → checkbox ile seç → tekrar çalıştır.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Veritabanı | MongoDB | PostgreSQL | ACID zorunlu; para kaybı kabul edilemez |
| Idempotency | Sadece Redis | Redis + DB UNIQUE | Redis crash → çift ödeme; DB asıl güvence |
| Ledger | Balance kolonu | Double-entry ledger | Balance: race condition; Ledger: immutable, audit |
| PSP entegrasyonu | Direkt Stripe kodu | Adapter pattern | Direkt: vendor lock-in; Adapter: kolay geçiş |
| Fraud detection | Sadece kural motoru | Kural + ML modeli | Kurallar: bilinen pattern; ML: yeni anomali |
| Webhook güvenilirliği | Webhook only | Webhook + polling | Webhook: gecikmez; Polling: recovery |
| Para birimi | Float/Double | BigDecimal + kuruş | Float: yuvarlama hatası → muhasebe bozulur |
| 3DS | Opsiyonel | Risk bazlı zorunlu | Opsiyonel: fraud riski; Zorunlu: liability shift |
| Kart verisi | Kendi saklama | PSP tokenization | Kendi: PCI Level 1 denetimi; PSP: scope minimal |
| Settlement | Anlık | T+1 batch | Anlık: karmaşık, FX riski; Batch: basit, standar |
| Saga | Orchestration | Choreography | Orchestration: SPOF; Choreography: loose coupling |
| Reconciliation | Webhook tetikli | Batch + polling | Webhook: gecikme; Batch: sistematik kapsam |
