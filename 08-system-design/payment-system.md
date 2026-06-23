# 08j — System Design: Ödeme Sistemi

## Gereksinimler

```
Functional:
  ✓ Ödeme başlatma (kart, banka havalesi)
  ✓ Ödeme durumu sorgulama
  ✓ İade (refund)
  ✓ Çift ödeme önleme (idempotency)
  ✓ Ödeme geçmişi

Non-Functional:
  QPS: 1,000 ödeme/sn (peak)
  Latency: < 3s (ödeme tamamlanma)
  Availability: 99.999% (five nines)
  Veri kaybı: sıfır (ACID garantisi şart)
  Audit log: tüm işlemler kayıt altında
  Compliance: PCI DSS (kart verisi güvenliği)
```

---

## Capacity Estimation

```
1,000 ödeme/sn peak
  Günde: 1,000 × 86,400 = 86.4M ödeme
  
Storage:
  Ödeme kaydı: ~1 KB
  Günlük: 86.4M × 1 KB = 86.4 GB/gün
  Audit log (tüm event'ler): 86.4M × 5 event × 500B = 215 GB/gün
  5 yıl: (86.4 + 215) GB × 365 × 5 = 550 TB

Idempotency key storage:
  86.4M × 64 bytes (UUID) = 5.5 GB/gün
  7 gün retention: 38 GB → Redis cluster
```

---

## High-Level Tasarım

```
         E-commerce App
              │
              │ POST /payments
              │ Idempotency-Key: uuid
              ▼
    ┌──────────────────────┐
    │   Payment Service    │
    │   (validation,       │
    │    idempotency check)│
    └──────────┬───────────┘
               │
    ┌──────────▼───────────┐
    │   Payment Processor  │ ←──→ Risk Engine
    │   (state machine)    │
    └──────────┬───────────┘
               │
    ┌──────────┼──────────────┐
    ▼          ▼              ▼
[Payment DB]  [Ledger]   [Event Bus]
(PostgreSQL)  (double-   (Kafka)
              entry)        │
                        ┌───┴──────────────────┐
                    [Notification] [Analytics] [Reconciliation]
```

---

## İdempotency (Çift Ödeme Önleme)

```
Problem:
  Client → POST /payments → Timeout (5s)
  Client: "Ödeme oldu mu?" → bilmiyorum → RETRY
  Sunucu: İki kez işledi → iki ödeme çekildi!

Çözüm: Idempotency Key
  Client: POST /payments
          Idempotency-Key: "550e8400-e29b-41d4-a716-446655440000"
          {amount: 99.99, currency: TRY}

  Server logic:
    1. Redis: key="idempotency:550e8400-..."
       SETNX → key yoktu → ilk istek → işle
       EXISTS → zaten var → önceki sonucu döndür

    2. İşle ve sonucu kaydet:
       Redis: SET "idempotency:550e8400-..." {result_json} EX 86400
       (24 saat boyunca aynı key gelirse aynı sonuç döner)

    3. Client retry geldi → Redis HIT → önceki sonuç döner
       Ödeme bir kez yapıldı ✓
```

```java
@PostMapping("/payments")
ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest req) {

    // 1. Idempotency kontrolü
    String cacheKey = "idempotency:" + idempotencyKey;
    String cachedResult = redis.opsForValue().get(cacheKey);
    if (cachedResult != null) {
        return ResponseEntity.ok(deserialize(cachedResult, PaymentResponse.class));
    }

    // 2. Mutex (aynı key ile eş zamanlı iki istek)
    String lockKey = "payment-lock:" + idempotencyKey;
    boolean locked = redis.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(30));
    if (!locked) {
        return ResponseEntity.status(429).body(null); // İşleniyor, bekle
    }

    try {
        // 3. Ödemeyi işle
        PaymentResponse response = processPayment(req);

        // 4. Sonucu cache'le
        redis.opsForValue().set(cacheKey, serialize(response), Duration.ofDays(1));

        return ResponseEntity.status(201).body(response);
    } finally {
        redis.delete(lockKey);
    }
}
```

---

## Ödeme State Machine

```
Başlangıç → INITIATED
  Doğrulama OK → VALIDATED
  Risk engine geçti → RISK_APPROVED
  Ödeme sağlayıcısına gönderildi → PROCESSING
  Sağlayıcı onayladı → COMPLETED
  Sağlayıcı reddetti → FAILED
  Kullanıcı iptal → CANCELLED

İade:
  COMPLETED → REFUND_REQUESTED
  İade sağlayıcısına gönderildi → REFUND_PROCESSING
  İade tamamlandı → REFUNDED
  İade başarısız → REFUND_FAILED

Her state değişikliği → Event yayınla (Kafka)
  PaymentInitiated, PaymentCompleted, PaymentFailed, RefundRequested...
```

```java
@Service
@Transactional
class PaymentStateMachine {

    void transition(Payment payment, PaymentStatus newStatus) {
        // Geçerli geçiş mi?
        validateTransition(payment.getStatus(), newStatus);

        PaymentStatus oldStatus = payment.getStatus();
        payment.setStatus(newStatus);
        payment.setUpdatedAt(Instant.now());

        // DB'ye kaydet
        paymentRepo.save(payment);

        // Audit log
        auditLogRepo.save(AuditLog.builder()
            .paymentId(payment.getId())
            .fromStatus(oldStatus)
            .toStatus(newStatus)
            .timestamp(Instant.now())
            .build());

        // Event yayınla (Outbox pattern ile)
        outboxRepo.save(OutboxEvent.of(
            "PaymentStatusChanged",
            payment.getId(),
            new PaymentStatusChangedEvent(payment.getId(), oldStatus, newStatus)
        ));
    }
}
```

---

## Double-Entry Ledger (Muhasebe)

```
Her finansal işlem DEBITS = CREDITS kuralını korur (double-entry accounting)

Ödeme: Kullanıcı 100 TL ödedi
  DEBIT:  user:alice:receivable     100 TL  (alacak azaldı)
  CREDIT: merchant:bob:payable      100 TL  (borç arttı)
  CREDIT: platform:fee              5 TL    (komisyon)

İade: 100 TL iade
  DEBIT:  merchant:bob:payable      100 TL
  CREDIT: user:alice:receivable     100 TL

INVARIANT: SUM(all debits) == SUM(all credits) == 0

Neden ledger?
  Tek "balance" kolonu güvenilir değil (eş zamanlı güncelleme sorunu)
  Ledger: immutable transaction log → herhangi anda bakiye = sum(transactions)
  Audit trail: ne zaman, kim, neden — her şey kayıtlı
```

```sql
CREATE TABLE ledger_entries (
    entry_id      UUID         PRIMARY KEY,
    payment_id    UUID         NOT NULL,
    account_id    VARCHAR(100) NOT NULL, -- user:alice, merchant:bob, platform:fee
    entry_type    VARCHAR(10)  NOT NULL, -- DEBIT, CREDIT
    amount        DECIMAL(19,4) NOT NULL,
    currency      CHAR(3)      NOT NULL,
    created_at    TIMESTAMP    NOT NULL,

    CONSTRAINT positive_amount CHECK (amount > 0)
);

-- Bakiye hesabı (view olarak)
CREATE VIEW account_balance AS
SELECT
    account_id,
    currency,
    SUM(CASE WHEN entry_type = 'CREDIT' THEN amount ELSE -amount END) AS balance
FROM ledger_entries
GROUP BY account_id, currency;

-- Kontrol: Debits = Credits?
SELECT SUM(CASE WHEN entry_type = 'DEBIT' THEN amount ELSE -amount END)
FROM ledger_entries
WHERE payment_id = 'pay-123';
-- Sonuç: 0 olmalı (balanced transaction)
```

---

## Ödeme Sağlayıcısı Entegrasyonu

```
Asenkron ödeme (en yaygın):
  1. POST payment API → Provider
  2. Provider: "İşleniyor" → {transactionId: "txn-456"}
  3. Provider → Webhook: POST /webhook/payment-status
     {transactionId: "txn-456", status: "COMPLETED"}
  4. Webhook → Payment Service: status güncelle

Webhook güvenliği:
  Provider → Webhook'u HMAC-SHA256 ile imzalar
  X-Payment-Signature: hmac-sha256={signature}
  
  Server:
    expectedSig = HMAC-SHA256(secret, request_body)
    receivedSig = header["X-Payment-Signature"]
    If not timing-safe-compare(expectedSig, receivedSig): reject

Webhook idempotency:
  Provider aynı webhook'u birden fazla gönderebilir (retry)
  processed_webhooks tablosu: (transactionId, eventType) → unique key
  Duplicate gelirse → ignore (idempotent handler)

Polling fallback (webhook başarısız olursa):
  Scheduler: Her 5 dk pending ödeme var mı? → Provider API'ye sor
  "Stuck" ödemeler recover edilir
```

---

## Reconciliation (Mutabakat)

```
Sorun: Payment Service DB ile Provider DB tutarsız olabilir
  Network hatası, timeout, bug → farklı durum

Günlük mutabakat:
  1. Provider'dan tüm transaction listesi al (CSV/API)
  2. Kendi DB ile karşılaştır:
     - Bizde var, provider'da yok → araştır
     - Provider'da var, bizde yok → eksik kaydı ekle
     - Durum farklı → düzelt

Alarm:
  Tutarsızlık varsa → alert → finance ekibi inceler

Reconciliation DB:
  reconciliation_runs tablosu: ne zaman çalıştı, kaç kayıt, kaç tutarsızlık
  reconciliation_discrepancies: tutarsız kayıtlar
```

---

## PCI DSS Uyumu (Kart Güvenliği)

```
PCI DSS Level 1 (en sıkı): 6M+ transaction/yıl

Kural 1: Kart numarası (PAN) asla plaintext saklanmaz
  Maskeleme: 4111 **** **** 1234 (display için)
  Tokenization: gerçek kart no → token (Stripe gibi provider halleder)

Kural 2: Kart verisi şifrelenir (AES-256)
  Transit: TLS 1.2+ (HTTPS)
  At-rest: DB encryption

Kural 3: Erişim log'u
  Kim, ne zaman, hangi kart datasına erişti → audit log
  
Kural 4: Segmentation
  Kart datasını tutan sistemler → ayrı network segment
  Diğer microservice'ler bu segment'e erişemez

Pratik: Stripe/Adyen gibi provider kullan
  Kart verisi onların sisteminde → PCI DSS onlara
  Sen sadece token ile çalışıyorsun → çok daha az PCI scope
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| DB | MongoDB | PostgreSQL | PostgreSQL (ACID, OLTP) |
| Idempotency | DB unique | Redis SETNX | Redis (hız) + DB (kalıcı) |
| Ledger | Balance column | Double-entry | Double-entry (audit, güvenli) |
| Webhook | Polling | Webhook + polling | Webhook + polling fallback |
| Kart güvenliği | Kendi sakla | Provider token | Provider (PCI scope dışı) |
