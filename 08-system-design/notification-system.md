# 08f — System Design: Notification Sistemi

## Gereksinimler

```
Functional:
  ✓ Push notification (iOS APNs, Android FCM)
  ✓ Email bildirimi
  ✓ SMS bildirimi
  ✓ In-app notification (WebSocket / SSE)
  ✓ Kullanıcı tercihleri (kanal seçimi, sessiz mod, kategori bazlı)
  ✓ Template yönetimi (i18n, kişiselleştirilmiş içerik)
  ✓ Scheduled notification (kampanya, hatırlatıcı)
  ✓ Notification gruplama / collapsing
  ✓ Delivery receipt (APNs/FCM geri bildirim)
  ✓ Unsubscribe / GDPR opt-out
  ✓ Idempotency (aynı bildirim iki kez gönderilmez)
  ✗ WhatsApp Business, Telegram (out of scope — eklenebilir)

Non-Functional:
  10M bildirim/gün → 116/sn ortalama
  Peak (kampanya/event): 50,000/sn
  Gecikme: OTP/security < 5sn, transactional < 30sn, marketing < 5dk
  Delivery guarantee: at-least-once (idempotency ile tam once semantiği)
  Availability: 99.9% (bildirim sistemi down → kullanıcı işlem göremez)
```

---

## Capacity Estimation

```
Hacim:
  10M bildirim/gün → 116/sn ortalama.
  Peak: kampanya → 10M kullanıcıya 1 saatte gönder = 2,778/sn.
  Büyük kampanya: 50M kullanıcı × %20 açılır → peak 10,000 Kafka msg/sn.

Kanal dağılımı:
  Push:  %60 → 6M/gün
  Email: %30 → 3M/gün
  SMS:   %10 → 1M/gün
  In-app: tüm push mesajları aynı zamanda in-app → +6M/gün

Storage (notification_delivery tablosu):
  Her kayıt: ~1 KB (metadata + content hash).
  10M × 1KB = 10 GB/gün.
  Retention 90 gün: 900 GB.
  Partitioning: aylık → 300 GB/ay bölüm.

Storage (in-app notification center):
  Kullanıcı başına: son 30 gün, max 100 bildirim.
  100M aktif kullanıcı × 100 notif × 500 byte = 5 TB (DynamoDB).
  Redis: unread_count:{userId} → her okuma anında güncellenir.

Device token storage:
  100M DAU × 1.5 cihaz/kullanıcı = 150M token kaydı.
  Her kayıt: ~200 byte → 30 GB (PostgreSQL).
  TTL: 90 gün kullanılmayan token → temizleme.

Provider maliyeti tahmini:
  FCM: ücretsiz (Google).
  APNs: ücretsiz (Apple).
  SendGrid: 3M email/gün × $0.0001 = $300/gün → $9,000/ay.
  Twilio SMS: 1M SMS/gün × $0.0075 = $7,500/gün → öncelik: uygulama push.
```

---

## High-Level Mimari

```
          Trigger Sources (bildirim tetikleyiciler)
   ┌──────────┬──────────┬─────────────┬──────────────┐
   │          │          │             │              │
[Order Svc] [Auth Svc] [Marketing]  [Scheduler]  [Admin]
   │          │          │             │              │
   └──────────┴──────────┴──────────────────┴──────────┘
                              │ HTTP / gRPC
                              ▼
                  ┌───────────────────────┐
                  │   Notification API    │
                  │  - Validation         │
                  │  - Idempotency check  │
                  │  - Preference filter  │
                  │  - Priority routing   │
                  │  - Rate limiting      │
                  │  - Template render    │
                  └──────────┬────────────┘
                             │
              ┌──────────────┼──────────────┬───────────────┐
              ▼              ▼              ▼               ▼
        [push.critical] [push.normal] [email.queue]   [sms.queue]
        (Kafka P0)      (Kafka P1)    (Kafka P2)      (Kafka P3)
              │              │              │               │
       ┌──────┴────┐  ┌──────┴────┐  ┌─────┴────┐  ┌──────┴─────┐
       │  Push     │  │  Push     │  │  Email   │  │    SMS     │
       │  Worker   │  │  Worker   │  │  Worker  │  │   Worker   │
       │(critical) │  │(bulk)     │  │          │  │            │
       └───┬───┬───┘  └───┬───┬───┘  └────┬─────┘  └──────┬─────┘
           │   │          │   │            │               │
         APNs FCM       APNs FCM      SendGrid         Twilio
                                      Mailgun          Nexmo (fallback)
                                                       AWS SNS (fallback)

Write path (in-app):
  Notification API → notification_center DB (DynamoDB) → WebSocket push
                                                        → SSE (browser)

Delivery receipt:
  FCM → webhook → Delivery Receipt Service → update notification_delivery
  APNs → feedback channel → token cleanup
```

---

## Öncelik Kuyruğu (Priority Queue)

```
OTP, güvenlik alert'leri → anında → 2FA kodu 30sn geçerli.
Sipariş güncellemesi → 30sn içinde.
Marketing kampanyası → 5dk içinde (dakik olması gerekmiyor).

Kafka topic ayrımı (thread pool isolation):
  push.critical (P0): OTP, 2FA, şifre sıfırlama, ödeme onayı.
  push.transactional (P1): sipariş durumu, kargo takip, randevu.
  push.marketing (P2): kampanya, öneri, hatırlatıcı.

Worker konfigürasyonu:
  P0 worker: concurrency=20, timeout=5sn, retry agresif (3x, 1sn interval).
  P1 worker: concurrency=10, timeout=15sn, retry moderate (3x, 5sn interval).
  P2 worker: concurrency=5, timeout=60sn, retry lazy (2x, 1dk interval).

Backpressure:
  Kampanya burst → P2 queue dolunca → producer throttle (beklet, hata verme).
  P0 ayrı topic → kampanya spike P0'ı etkilemez.
  Thread pool isolation: P2 thread'ler dolsa bile P0 thread'leri boş.

Kafka Consumer Group Lag monitoring:
  P0 lag > 1,000 → anında alert (critical notifications gecikiyor).
  P2 lag > 100,000 → uyarı (marketing birikmiş, normal).
```

---

## Notification API

```java
@RestController
@RequestMapping("/api/notifications")
class NotificationController {

    @PostMapping("/send")
    ResponseEntity<SendResponse> sendNotification(@RequestBody NotificationRequest req) {

        // 1. İdempotency kontrolü
        String idempotencyKey = req.getIdempotencyKey();
        if (idempotencyKey != null) {
            Optional<String> existing = idempotencyCache.get(idempotencyKey);
            if (existing.isPresent()) {
                return ResponseEntity.ok(SendResponse.alreadySent(existing.get()));
            }
        }

        // 2. Validasyon
        validator.validate(req);

        // 3. Kullanıcı tercihleri kontrolü
        UserPreference pref = preferenceService.get(req.getUserId());
        List<Channel> enabledChannels = resolveChannels(req, pref);
        if (enabledChannels.isEmpty()) {
            return ResponseEntity.ok(SendResponse.suppressed("User preferences"));
        }

        // 4. Sessiz saatler (quiet hours) kontrolü
        if (pref.isInQuietHours(ZonedDateTime.now(pref.getTimezone()))
                && req.getPriority() != Priority.CRITICAL) {
            scheduleAfterQuietHours(req, pref);
            return ResponseEntity.accepted().body(SendResponse.scheduled());
        }

        // 5. Rate limiting
        for (Channel ch : enabledChannels) {
            rateLimiter.checkOrThrow("user:" + req.getUserId() + ":" + ch + ":" + req.getCategory());
        }

        // 6. Template render (i18n)
        String lang = userService.getLanguage(req.getUserId());
        String content = templateEngine.render(req.getTemplateId(), req.getParams(), lang);

        // 7. Notification kaydı
        String notifId = UUID.randomUUID().toString();
        notificationRepo.save(NotificationRecord.builder()
            .notificationId(notifId)
            .userId(req.getUserId())
            .channels(enabledChannels)
            .content(content)
            .status(Status.PENDING)
            .build());

        // 8. Kafka'ya publish (kanal başına)
        for (Channel ch : enabledChannels) {
            String topic = resolveTopic(ch, req.getPriority());
            kafkaTemplate.send(topic, notifId, NotificationMessage.builder()
                .notificationId(notifId)
                .userId(req.getUserId())
                .channel(ch)
                .content(content)
                .priority(req.getPriority())
                .build());
        }

        // 9. İdempotency cache güncelle (TTL: 24 saat)
        if (idempotencyKey != null) {
            idempotencyCache.set(idempotencyKey, notifId, Duration.ofHours(24));
        }

        return ResponseEntity.ok(SendResponse.queued(notifId));
    }
}
```

---

## Push Notification Worker (FCM / APNs)

### Çok Cihaz Yönetimi

```java
@Component
class PushNotificationWorker {

    @KafkaListener(topics = {"push.critical", "push.transactional", "push.marketing"},
                   groupId = "push-workers", concurrency = "10")
    void process(NotificationMessage message) {
        // Kullanıcının TÜM aktif cihazları
        List<DeviceToken> devices = deviceService.getActiveDevices(message.getUserId());

        if (devices.isEmpty()) {
            notificationRepo.updateStatus(message.getNotificationId(), Status.NO_DEVICE);
            return;
        }

        // Gruplama/collapsing: aynı collapse_key → eski bildirimi değiştir
        String collapseKey = buildCollapseKey(message);

        List<CompletableFuture<DeliveryResult>> futures = devices.stream()
            .map(device -> CompletableFuture.supplyAsync(() ->
                sendToDevice(device, message, collapseKey)))
            .collect(Collectors.toList());

        // En az bir cihaza ulaştı → SUCCESS
        List<DeliveryResult> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());

        boolean anySuccess = results.stream().anyMatch(DeliveryResult::isSuccess);
        notificationRepo.updateStatus(message.getNotificationId(),
            anySuccess ? Status.SENT : Status.FAILED);
    }

    private DeliveryResult sendToDevice(DeviceToken device, NotificationMessage msg, String collapseKey) {
        try {
            if (device.isIos()) {
                return apnsService.send(ApnsPayload.builder()
                    .deviceToken(device.getToken())
                    .title(msg.getTitle())
                    .body(msg.getContent())
                    .badge(unreadCountService.get(msg.getUserId()))
                    .collapseId(collapseKey)   // iOS: aynı collapseId → günceller
                    .threadId(msg.getThreadId()) // iOS: bildirim gruplama
                    .sound(msg.getPriority() == Priority.CRITICAL ? "critical.caf" : "default")
                    .mutableContent(true)  // rich push için (resim)
                    .build());
            } else {
                return fcmService.send(FcmMessage.builder()
                    .token(device.getToken())
                    .notification(Notification.builder()
                        .title(msg.getTitle())
                        .body(msg.getContent())
                        .build())
                    .android(AndroidConfig.builder()
                        .collapseKey(collapseKey)  // FCM: aynı key → son mesaj kalır
                        .priority(msg.getPriority() == Priority.CRITICAL
                            ? AndroidConfig.Priority.HIGH
                            : AndroidConfig.Priority.NORMAL)
                        .ttl(Duration.ofHours(24))
                        .build())
                    .data(msg.getExtraData())
                    .build());
            }
        } catch (InvalidTokenException e) {
            deviceService.deactivateToken(device.getToken());
            return DeliveryResult.invalidToken();
        }
    }
}
```

### Token Yönetimi

```
Token lifecycle:
  Kullanıcı app yükledi → FCM/APNs token üretildi → backend'e kaydet.
  Token değişimi: app güncellemesi, cihaz değişimi, OS güncellemesi.
  
  Client: onTokenRefresh() callback → yeni tokeni PUT /devices/{deviceId} ile güncelle.
  
  Geçersiz token tespiti:
    FCM: HTTP 404 → token artık geçerli değil → hemen sil.
    APNs: HTTP 410 → token geçersiz + son geçerlilik zamanı döner.
    APNs: HTTP 400 BadDeviceToken → yanlış format → sil.

Token temizliği:
  Pasif token: 90 gün boyunca hiç push gönderilmedi → pasif işaretle.
  APNs Feedback Service: periyodik sorgula → geçersiz tokenleri toplu sil.
  FCM Topic Messaging: token listesi yerine topic → FCM kendi yönetir.

Çoklu cihaz senaryoları:
  Kullanıcının 3 telefonu var:
    iPhone (aktif), Android (eski, 60 gün kullanılmadı), iPad (aktif).
  Strateji:
    OTP: sadece en son aktif cihaza → güvenlik.
    Marketing: tüm aktif cihazlara → ama gruplama ile tek görünsün.
    Son aktif cihaz: last_seen_at → en son aktif → primary.

Device tablo:
CREATE TABLE user_devices (
    device_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id        UUID NOT NULL,
    platform       VARCHAR(10) NOT NULL,  -- IOS, ANDROID, WEB
    token          VARCHAR(255) NOT NULL UNIQUE,
    device_name    VARCHAR(100),          -- "Muhammed'in iPhone'u"
    is_active      BOOLEAN DEFAULT TRUE,
    last_seen_at   TIMESTAMPTZ DEFAULT NOW(),
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_user_devices (user_id, is_active)
);
```

---

## Email Worker

```java
@Component
class EmailWorker {

    @KafkaListener(topics = "email.queue", concurrency = "5")
    void process(NotificationMessage message) {
        UserProfile user = userService.getProfile(message.getUserId());

        // Blacklist kontrolü (hard bounce, unsubscribe)
        if (emailBlacklist.isBlocked(user.getEmail())) {
            notificationRepo.updateStatus(message.getNotificationId(), Status.SUPPRESSED);
            return;
        }

        // Unsubscribe token (RFC 8058 — one-click unsubscribe)
        String unsubToken = unsubscribeService.generateToken(
            message.getUserId(), message.getCategory());

        EmailRequest email = EmailRequest.builder()
            .to(user.getEmail())
            .subject(message.getTitle())
            .htmlBody(message.getContent())
            .textBody(markdownToText(message.getContent()))
            .from("no-reply@myapp.com")
            .replyTo("support@myapp.com")
            // RFC 8058: Gmail/Outlook "Aboneliği Kaldır" düğmesi için
            .listUnsubscribe("<https://myapp.com/unsubscribe?token=" + unsubToken + ">")
            .listUnsubscribePost("List-Unsubscribe=One-Click")
            // Tracking pixel (open rate)
            .trackingPixelUrl("https://track.myapp.com/open/" + message.getNotificationId())
            .build();

        try {
            ProviderResponse resp = sendGridClient.send(email);
            notificationRepo.markSent(message.getNotificationId(), resp.getMessageId(), "SENDGRID");
        } catch (ProviderException e) {
            // Provider fallback
            try {
                ProviderResponse resp = mailgunClient.send(email);
                notificationRepo.markSent(message.getNotificationId(), resp.getMessageId(), "MAILGUN");
            } catch (ProviderException e2) {
                throw e2; // Kafka retry tetiklensin
            }
        }
    }
}
```

### Email Deliverability

```
Domain reputation:
  Kendi domain'inden gönder: @myapp.com (shared provider IP değil).
  SPF: TXT kaydı → hangi sunucular bu domain'den mail gönderebilir.
  DKIM: email imzalama → alıcı doğrulabilir (değiştirilmedi mi?).
  DMARC: SPF ve DKIM başarısız → ne yapılsın? (reject, quarantine, none).
  
  DMARC policy (kademeli):
    p=none → önce izle (rapor al, reddetme).
    p=quarantine → spam'e düşür.
    p=reject → tamamen reddet.
  
  Warm-up: yeni IP/domain → günde 100 → 500 → 2,000 → 10,000 → kademeli artış.
  Spam şikayeti > %0.1 → Gmail bulk sender threshold → inbox'a girmez.

Bounce yönetimi:
  Hard bounce (email adresi yok, domain yok):
    → anında blacklist'e al → bir daha gönderme.
    Tekrar gönderme: spam şikayeti artırır → domain itibarı düşer.
  
  Soft bounce (mailbox dolu, sunucu geçici down):
    → 1 saat → 6 saat → 24 saat → 3. denemede başarısız → hard bounce muamelesi.
  
  SendGrid webhook: bounce event → POST /webhooks/sendgrid → email_blacklist'e ekle.

IP warmup:
  Dedicated IP: günde 50,000+ email → dedicated IP zorunlu.
  Shared IP: daha az hacim → ucuz ama başka hesapların kötü davranışından etkilenirsin.
```

---

## SMS Worker

```java
@Component
class SmsWorker {

    @KafkaListener(topics = "sms.queue", concurrency = "3")
    void process(NotificationMessage message) {
        UserProfile user = userService.getProfile(message.getUserId());

        // Telefon formatı normalize (E.164: +905551234567)
        String phoneNumber = phoneNormalizer.toE164(user.getPhone(), user.getCountry());

        // Provider seçimi (ülke bazlı)
        SmsProvider provider = providerSelector.select(user.getCountry(), message.getPriority());

        try {
            SmsResponse resp = provider.send(SmsRequest.builder()
                .to(phoneNumber)
                .from(getSenderId(user.getCountry())) // TR: sayısal ID, US: short code
                .body(message.getContent())
                .build());

            notificationRepo.markSent(message.getNotificationId(),
                resp.getMessageId(), provider.name());

        } catch (ProviderException e) {
            // Fallback provider
            SmsProvider fallback = providerSelector.getFallback(provider, user.getCountry());
            if (fallback != null) {
                SmsResponse resp = fallback.send(...);
                notificationRepo.markSent(message.getNotificationId(),
                    resp.getMessageId(), fallback.name());
            } else {
                throw e; // Kafka retry
            }
        }
    }
}
```

### Provider Failover

```
SMS provider tier:
  Birincil:  Twilio (global, güvenilir, pahalı).
  İkincil:   Nexmo/Vonage (iyi coverage, orta fiyat).
  Üçüncül:   AWS SNS (ucuz, bazı ülkeler sorunlu).
  Yerele özel: Türkiye için NetGSM/iletimerkezi (daha iyi deliverability, ucuz).

Failover logic:
  Twilio → timeout veya 5xx → Nexmo'ya geç → başarısız → AWS SNS.
  Ülke bazlı routing: TR → NetGSM (birincil) → Twilio (fallback).
  OTP için: provider'ı dene → başarısız → sesli arama (IVR) fallback!

Operator blocking:
  Türk telekomunikasyon: bazı içerikler filtreli → "KARGO" kelimesi bazen engellenir.
  Çözüm: içerik template'lerini operatör kurallarına göre test et.
  Sender ID: ticari SMS → onaylı gönderici ID zorunlu.

Delivery receipt:
  Twilio: webhook → POST /webhooks/twilio → message SID ile durum güncelle.
  DELIVERED: kullanıcı telefonu aldı.
  UNDELIVERED: telefon kapalı, kapsama yok.
  FAILED: geçersiz numara.
```

---

## In-App Notification Center

```
Kullanıcı uygulamayı açtığında bildirimleri görmeli:
  "5 bildirim okunmadı" → badge sayacı.
  Bildirim listesi: en yeni önce, okunmuş/okunmamış ayrımı.

WebSocket (aktif kullanıcı, anlık):
  Kullanıcı online → WebSocket bağlantısı.
  Yeni bildirim → Redis pub/sub: PUBLISH notifications:user-123 {notifId}.
  WebSocket handler → subscribe → push → anında göster.

SSE (Server-Sent Events, browser için daha basit):
  Tek yönlü: server → client.
  HTTP/2 keep-alive → polling yerine daha verimli.
  GET /notifications/stream → text/event-stream.

Okunmamış sayaç (Redis):
  INCR unread:{userId} → yeni bildirim geldi.
  GET unread:{userId} → badge sayısını göster.
  SET unread:{userId} 0 → kullanıcı notification center'ı açtı → sıfırla.

Notification Center API:
  GET /notifications?cursor=...&limit=20
  → cursor-based pagination (sayfalama).
  → status: READ, UNREAD.

  PATCH /notifications/{id}/read
  → status = READ, read_at = now.
  → Redis DECR unread:{userId}.

  POST /notifications/read-all
  → batch update → tümünü oku.

DynamoDB şeması (notification center):
  Partition key: user_id
  Sort key: created_at#notification_id (yeniden eskiye sıralama).
  TTL: 30 gün (eski bildirimler otomatik silinir).
  GSI: (user_id, status) → okunmamışları hızla bul.
```

---

## Bildirim Gruplama (Collapsing / Bundling)

```
Sorun: 50 arkadaş aynı anda yorum yaptı → 50 ayrı bildirim → rahatsız edici.
Çözüm: gruplama (iOS Grouped Notifications, FCM collapse_key).

APNs thread-id (iOS):
  Aynı thread-id → iOS otomatik gruplar → "50 yorum" tek bildirim.
  thread-id: "comments-post-456" → post ID bazlı gruplama.

FCM collapse_key (Android):
  Aynı collapse_key gelirse → eski bildirimi siler, yenisi gösterilir.
  "Son mesajı göster" davranışı.
  Dikkat: kollapstan önceki bildirimler kaybolur.

Backend collapse (sunucu tarafı gruplama):
  50 yorum bildirimi → kuyruğa → worker biriktiriyor.
  5sn bekleniyor → hepsi toplandı → tek "Ahmet ve 49 diğeri yorum yaptı" bildirimi.
  
  Redis sliding window:
    pending_group:{userId}:{postId} LPUSH {commentorId}
    EXPIRE pending_group:{userId}:{postId} 5  (5 saniyelik pencere)
    
    Scheduler (her 5sn): bekleyen grupları kontrol et → tek bildirim gönder.
  
  Avantaj: kullanıcı deneyimi çok daha iyi.
  Dezavantaj: 5sn gecikme ekler (marketing için kabul edilebilir, OTP için uygulanmaz).

Push içeriği güncelleme (yerine geçme):
  "Ahmet yorum yaptı" → 2sn sonra → "Ahmet ve Fatma yorum yaptı" → öncekini değiştir.
  iOS: apns-collapse-id header → aynı ID → eski bildirimi güncelle.
  FCM: collapse_key → son mesaj cihazda görünür.
```

---

## Template Yönetimi (i18n + Kişiselleştirme)

```sql
CREATE TABLE notification_templates (
    template_id    VARCHAR(100),
    lang           VARCHAR(10),  -- tr, en, de, fr
    channel        VARCHAR(20),  -- PUSH, EMAIL, SMS
    category       VARCHAR(50),  -- ORDER_UPDATE, MARKETING, SECURITY
    title_tpl      TEXT,         -- "{{name}}, siparişin yolda!"
    body_tpl       TEXT,         -- HTML (email) veya düz metin (push/SMS)
    version        INT DEFAULT 1,
    is_active      BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (template_id, lang, channel)
);
```

```
Template render pipeline:
  1. Template seç: template_id + user lang + channel.
  2. Fallback: TR yoksa → EN kullan.
  3. Mustache/Handlebars render:
     "{{name}}, siparişin #{{orderId}} yolda!" + params → "Muhammed, sipariş #1234 yolda!"
  4. Uzunluk kısıtı:
     SMS: max 160 karakter (tek SMS) → 160+ → çok parçalı (ücret artar!).
     Push title: max 65 karakter (iOS kesiyor).
     Push body: max 200 karakter.
  5. Personalizasyon:
     Kullanıcı adı, kullanıcı dili, yerel para birimi, tarih formatı (DD.MM.YYYY vs MM/DD/YYYY).
  6. A/B test:
     Template A / Template B → hangisi daha fazla tıklanıyor?
     %50 kullanıcı → A, %50 → B → click-through rate karşılaştır.

Template cache:
  Redis: template:{template_id}:{lang}:{channel} → rendered string (basit case).
  Cache TTL: 1 saat (template değişince invalidate).
  Avantaj: DB'ye her bildirim için sorgu atmaz.
```

---

## Kullanıcı Tercihleri

```sql
CREATE TABLE notification_preferences (
    user_id         UUID         PRIMARY KEY,
    push_enabled    BOOLEAN      DEFAULT TRUE,
    email_enabled   BOOLEAN      DEFAULT TRUE,
    sms_enabled     BOOLEAN      DEFAULT FALSE,
    quiet_hours     BOOLEAN      DEFAULT FALSE,
    quiet_start     TIME,                        -- 22:00
    quiet_end       TIME,                        -- 08:00
    timezone        VARCHAR(50)  DEFAULT 'UTC',  -- IANA: "Europe/Istanbul"
    updated_at      TIMESTAMPTZ
);

-- Kategori bazlı ince kontrol
CREATE TABLE notification_category_prefs (
    user_id     UUID,
    category    VARCHAR(50),  -- ORDER_UPDATE, PROMOTION, SECURITY, CHAT, REMINDER
    push        BOOLEAN DEFAULT TRUE,
    email       BOOLEAN DEFAULT TRUE,
    sms         BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (user_id, category)
);

-- E-posta abonelik listesi (GDPR)
CREATE TABLE email_subscriptions (
    email           VARCHAR(255) PRIMARY KEY,
    user_id         UUID,
    status          VARCHAR(20) DEFAULT 'SUBSCRIBED',  -- SUBSCRIBED, UNSUBSCRIBED, BOUNCED
    unsub_reason    TEXT,
    subscribed_at   TIMESTAMPTZ DEFAULT NOW(),
    unsubscribed_at TIMESTAMPTZ,
    unsub_source    VARCHAR(50)   -- LINK, ONE_CLICK, COMPLAINT, ADMIN
);
```

```
Quiet Hours (sessiz saatler):
  Kullanıcı "22:00 - 08:00 arası bildirim istemiyorum" dedi.
  Kritik önemi: timezone dikkate alınmalı!
  
  Hatalı yaklaşım: quiet_start/end UTC'de karşılaştır.
    Türkiye kullanıcısı (UTC+3): 22:00 local = 19:00 UTC.
    UTC'de 22:00 kontrolü → 01:00 local → yanlış!
  
  Doğru yaklaşım:
    IANA timezone string sakla: "Europe/Istanbul".
    ZonedDateTime.now(ZoneId.of(user.getTimezone())) → local time.
    local time >= quiet_start VE local time < quiet_end → sessiz.
  
  Erteleme (quiet hours bitince gönder):
    Bildirim → quiet hours → Redis ZADD: delayed:{userId} {quiet_end_epoch} {notifId}.
    Scheduler: epoch geçti mi → Kafka'ya push.
    Not: quiet hours biterken burst → her kullanıcının 22:00-08:00 arası birikenler
    → 08:00'de aynı anda → spike. Çözüm: 08:00-08:15 arası yay (jitter).
```

---

## Scheduled Notifications

```
Seçenek 1: Cron job (kötü)
  Her gece 00:00 → yarın gönderilecekleri Kafka'ya yükle.
  Sorun: tüm bildirimleri aynı anda Kafka'ya at → spike.
  Sorun: dakika hassasiyeti yok (her gece çalışıyor).

Seçenek 2: Redis Sorted Set + Poller (önerilen)
  ZADD scheduled_notifications {sendAt_unix_ms} {notificationId}
  
  Scheduler (her 1sn):
    ZRANGEBYSCORE scheduled_notifications 0 {now_ms} LIMIT 0 1000
    → Kafka'ya push et (bulk send).
    ZREM scheduled_notifications {notificationIds}
  
  Distributed lock (tek scheduler):
    Scheduler birden fazla instance olabilir → her ikisi de ZRANGEBYSCORE → duplicate.
    Redis SETNX scheduler:lock 30sn TTL → kilidi alan işler, diğeri atlar.

Seçenek 3: Kafka Delayed Messages (Kafka 3.x veya Confluent)
  Kafka: header'a delivery_timestamp ekle.
  Consumer: henüz zamanı gelmediyse → yeniden publish et (geç mesaj).
  Sorun: polling overhead, düşük precision.

Seçenek 4: Dedicated Job Scheduler (karmaşık ama sağlam)
  Quartz Scheduler (cluster mod) veya dahili job-scheduler.
  Her scheduled notification → job olarak kayıt.
  Partition-based execution → ölçeklenebilir.
  (08-system-design/job-scheduler.md ile aynı pattern.)

Kampanya zamanlama:
  Marketing: "Tüm kullanıcılara Salı 10:00'da gönder."
  10M kullanıcı → 10:00'da başla → scheduler → 1,000/sn rate ile yay → 10,000 sn (~3 saat).
  Rate control: ZADD ile yayarak gönder, burst yapma.
  Timezone-aware kampanya: "Herkes kendi 10:00'ında alsın" → timezone bazlı gruplama.
```

---

## Idempotency

```
Problem: sipariş onayı bildirimi iki kez gönderildi.
  Senaryo: Kafka consumer → bildirim gönder → ACK vermeden crash.
  Kafka: tekrar gönder → bildirim tekrar işlenir → duplicate.
  Kullanıcı: iki "Siparişiniz onaylandı" bildirimi → kötü UX.

Çözüm:
  a) Notification ID kontrolü:
     Her notification → UUID.
     Worker: bildirim göndermeden önce → notification_delivery tablosunu kontrol et.
     status = SENT → zaten gönderildi → skip.
     Atomik: UPDATE notification_delivery SET status='SENT' WHERE notification_id=? AND status='PENDING'
     → 0 satır güncellendi → zaten gönderilmiş.

  b) Provider tarafı idempotency:
     FCM: idempotent (aynı token + aynı collapse_key → son mesaj gösterilir).
     APNs: apns-push-id header → tekrar gönderilirse yeni bildirim (idempotent DEĞİL).
     SendGrid: x-message-id → tekrar gönderilirse 200 döner ama gerçekte göndermez (cache).
     SMS Twilio: idempotency_key → aynı key → tekrar göndermez.

  c) Upstream idempotency key:
     Çağıran servis: notificationRequest.idempotencyKey = "order-123-confirmation".
     Notification API: Redis'te kontrol → zaten işlendi mi?
     TTL: 24 saat → eski key'ler silinir.
```

---

## Delivery Tracking ve Webhooks

```
Provider geri bildirimleri:

FCM Delivery Receipt:
  FCM → uygulamamıza webhook gönderemez (push gateway aracı).
  Alternatif: mobil app → bildirim alındı → POST /notifications/{id}/delivered.
  App kapalıysa: bildirim geldi ama uygulama bilmiyor → app açılınca sync.

APNs Feedback:
  APNs: HTTP/2 stream → bildirim gönder → response: DeviceNotRegistered → token sil.
  APNs Feedback Service (eski yöntem): POST /3/device/{token} → HTTP 410 → bu token ölü.
  
SendGrid webhooks:
  POST /webhooks/sendgrid:
    event: "delivered" → email sunucuya ulaştı.
    event: "open" → tracking pixel görüldü → status = READ.
    event: "click" → linkten tıklandı.
    event: "bounce" → hard/soft bounce → blacklist güncelle.
    event: "unsubscribe" → email_subscriptions update.
    event: "spam_report" → şikâyet → blacklist + sender rep düşer.

Twilio webhooks:
  POST /webhooks/twilio:
    MessageStatus: delivered / undelivered / failed.
    ErrorCode: 30003 (unreachable), 30008 (unknown).

Webhook güvenliği:
  SendGrid: X-Twilio-Email-Event-Webhook-Signature → imza doğrula.
  Twilio: X-Twilio-Signature → HMAC-SHA1 doğrula.
  Bunlar olmadan: sahte webhook → yanlış durum güncellemesi.

Delivery dashboard metrikleri:
  Push delivery rate = DELIVERED / SENT (hedef: > %95).
  Email open rate = READ / DELIVERED (benchmark: %20-25).
  Email click rate = CLICK / DELIVERED (benchmark: %2-3).
  SMS delivery rate = DELIVERED / SENT (hedef: > %98).
  Bounce rate = HARD_BOUNCE / SENT (alarm: > %2 → domain itibarı riski).
```

---

## Rate Limiting & Anti-Spam

```
Kullanıcı başına (Redis token bucket):
  SECURITY (OTP, 2FA): sınırsız (ama IP bazlı brute-force koruması ayrıca).
  ORDER_UPDATE: sınırsız transactional.
  CHAT: max 100/saat (spam bot önleme).
  MARKETING push: max 3/gün.
  MARKETING email: max 1/gün.
  MARKETING SMS: max 2/hafta.

Redis implementasyonu:
  Key: rl:{userId}:{category}:{channel}:{date}
  INCR → count > limit → SUPPRESSED olarak kaydet, gönderme.
  EXPIRE: 86400 (günlük reset) veya 604800 (haftalık).

Global rate limiting (provider koruma):
  FCM: 600,000 mesaj/dakika → aşarsa 429 → exponential backoff.
  APNs: bağlantı başına 300 req/sn → connection pool yönet.
  Twilio: API tier'a göre limit (Enterprise: sınırsız, Basic: 1 MPS).

Anti-spam kontroller:
  İçerik analizi: URL blacklist, spam keyword tespiti (marketing için).
  Unsubscribe sonrası: o kullanıcıya marketing gönderme → GDPR zorunluluğu.
  Complaint handling: kullanıcı "spam" dedi → hemen unsubscribe → tüm marketing dur.

Bounce handling:
  Hard bounce: email yok → email_blacklist → bir daha ASLA gönderme.
  Soft bounce: geçici hata → exponential retry (1h, 6h, 24h) → 3. denemede hard bounce.
  GDPR: kullanıcı silme isteği → tüm kanallardan kaldır → 30 gün içinde sil.
```

---

## Retry Stratejisi ve Dead Letter

```
Retry topic hiyerarşisi (Kafka):

push.critical:
  → push.critical.retry-1 (5sn bekle, 3 deneme)
  → push.critical.DLT → alert → manual review.

push.transactional:
  → push.transactional.retry-1 (30sn)
  → push.transactional.retry-2 (5dk)
  → push.transactional.retry-3 (30dk)
  → push.transactional.DLT

push.marketing:
  → push.marketing.retry-1 (5dk)
  → push.marketing.retry-2 (1saat)
  → push.marketing.DLT (marketing için DLT'deki kayıp: OK)

Kalıcı hata → hemen DLT (retry etme):
  InvalidTokenException → deviceService.deactivateToken() → DLT.
  UserUnsubscribedException → SUPPRESSED kaydet → DLT.
  HardBounceException → blacklist ekle → DLT.

DLT monitoring:
  DLT'deki mesaj sayısı → Kafka consumer lag → Grafana alert.
  P0 DLT'de herhangi bir mesaj → hemen PagerDuty alarm.
  P2 DLT'de biriken → daily rapor → geliştirici incelemesi.

Circuit Breaker (provider down):
  Provider → 5 ardışık timeout → circuit OPEN.
  Circuit OPEN: direkt DLT veya fallback provider'a yönlendir.
  30sn sonra: HALF-OPEN → 1 deneme → başarılı → CLOSED.
  Resilience4j veya manuel Redis counter ile implementasyon.

Exponential backoff:
  1. deneme: hemen.
  2. deneme: 2^1 × base (30sn) = 30sn.
  3. deneme: 2^2 × base = 60sn.
  4. deneme: 2^3 × base = 120sn.
  + jitter: ± %20 rastgele → tüm retry'lar aynı anda gelmez (thundering herd).
```

---

## Veri Modeli

```sql
-- Bildirim gönderme kaydı
CREATE TABLE notification_delivery (
    notification_id  UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID         NOT NULL,
    channel          VARCHAR(20)  NOT NULL,   -- PUSH, EMAIL, SMS, IN_APP
    category         VARCHAR(50)  NOT NULL,   -- ORDER_UPDATE, SECURITY, MARKETING
    priority         VARCHAR(20)  DEFAULT 'NORMAL',  -- CRITICAL, HIGH, NORMAL, LOW
    template_id      VARCHAR(100),
    title            TEXT,
    content          TEXT,
    status           VARCHAR(20)  DEFAULT 'PENDING',
    -- PENDING → SENT → DELIVERED → READ
    -- PENDING → SUPPRESSED (rate limit, preference, unsubscribe)
    -- PENDING → FAILED → (retry) → SENT | DLT
    provider         VARCHAR(50),             -- FCM, APNS, SENDGRID, TWILIO
    provider_msg_id  VARCHAR(255),
    device_id        UUID,
    idempotency_key  VARCHAR(255) UNIQUE,
    sent_at          TIMESTAMPTZ,
    delivered_at     TIMESTAMPTZ,
    read_at          TIMESTAMPTZ,
    failure_reason   TEXT,
    retry_count      SMALLINT DEFAULT 0,
    created_at       TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);  -- aylık partition

CREATE TABLE notification_delivery_2026_01 PARTITION OF notification_delivery
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Indexler
CREATE INDEX idx_nd_user_created ON notification_delivery (user_id, created_at DESC);
CREATE INDEX idx_nd_status ON notification_delivery (status) WHERE status = 'PENDING';
CREATE INDEX idx_nd_idempotency ON notification_delivery (idempotency_key) WHERE idempotency_key IS NOT NULL;

-- In-app notification center (DynamoDB)
-- PK: user_id, SK: created_at#notification_id
-- GSI: user_id + status (unread listesi için)
-- Attribute: title, body, action_url, icon, status (UNREAD/READ), created_at
-- TTL: expire_at (30 gün)
```

---

## Olası Sorunlar ve Çözümleri

### 1. Duplicate Bildirim — Kullanıcı Aynı Bildirimi İki Kez Aldı

```
Sorun:
  Sipariş onay bildirimi: worker → FCM'e gönder → FCM kabul etti (200) → DB güncelle.
  Kafka ACK vermeden önce: worker crash → Kafka mesajı retry.
  Yeni worker: aynı notification_id → tekrar FCM'e gönder → kullanıcı 2 bildirim aldı.

Neden olur:
  Kafka: at-least-once semantiği → crash recovery → mesaj tekrar işlenir.
  DB güncelleme ve Kafka ACK: atomik değil → crash aralarında olabilir.

Çözüm:
  a) DB'de idempotent check (primary):
     Worker başlamadan: UPDATE notification_delivery
       SET status='PROCESSING', worker_id=?
       WHERE notification_id=? AND status='PENDING'
     → 0 satır → zaten işleniyor/bitti → skip.
     
  b) Redis lock (hızlı check):
     SETNX processing:{notificationId} {workerId} EX 60
     → Başarısız → başka worker işliyor → skip.
     İşlem bitti → DEL processing:{notificationId}.
  
  c) Provider idempotency key:
     FCM: Data payload'a notification_id ekle → client: daha önce gördüm → ignore.
     Twilio: idempotency_key parameter → API düzeyinde dedupe.
  
  d) Kafka exactly-once (EOS):
     Kafka transactional API → producer + consumer tek transaction.
     Overhead var ama critical path için değer.
```

---

### 2. Kampanya Spike — 10M Bildirim Aynı Anda Kuyruğa Girdi

```
Sorun:
  Marketing: "Bugün tüm kullanıcılara indirim bildirimi" → tek tuşa basıldı.
  10M mesaj anında Kafka'ya → Push Worker: 10M message → FCM'e.
  FCM: rate limit (600K/dk) → 429 Too Many Requests → push worker'lar hata alıyor.
  Retry storm: 10M retry → sistem daha da yükleniyour → çöküş.

Çözüm:
  a) Rate-limited campaign sender:
     Kampanya gönderim servisi: 10M kullanıcı listesi → 1,000/sn hızında Kafka'ya push.
     10M / 1,000/sn = 10,000 sn ≈ 2.8 saat → kabul edilebilir, burst yok.
     Redis token bucket: campaignSender → 1,000 token/sn → tüketti → bekle.

  b) Kampanya batch API:
     FCM Batch Messages API: tek request → 500 token'a gönder.
     10M / 500 = 20,000 batch request → çok daha verimli.

  c) Scheduled spreading:
     "Tüm kullanıcılara 10:00'da" → timezone bazlı gruplama.
     TR kullanıcılar: 10:00 Istanbul → UTC 07:00.
     DE kullanıcılar: 10:00 Berlin → UTC 09:00.
     → İki farklı zaman diliminde gönderim → doğal spread.

  d) Dedicated campaign worker:
     Kampanya trafiği: push.marketing topic → düşük priority worker.
     Transactional bildirimler: push.transactional → ayrı worker → etkilenmez.
     Kampanya yavaş gidebilir, OTP gidemez.
```

---

### 3. Quiet Hours Timezone Bug — Yanlış Saatte Bildirim

```
Sorun:
  Türk kullanıcı: quiet hours 22:00-08:00 ayarladı, timezone kaydedilmedi.
  Sistem: UTC'de karşılaştırma → 22:00 UTC = 01:00 İstanbul.
  Kullanıcı: gece 01:00'de bildirim aldı → "Sizi gece uyandırdı mı?"

Çözüm:
  a) IANA timezone string zorunlu:
     Kullanıcı timezone: "Europe/Istanbul" (UTC+3 değil! → DST değişimine karşı korumalı).
     UTC+3 sabit → Türkiye DST kaldırdı (artık sorun yok) ama genel prensip: IANA kullan.
  
  b) Doğru karşılaştırma:
     LocalTime now = ZonedDateTime.now(ZoneId.of(user.getTimezone())).toLocalTime();
     if (now.isAfter(quietStart) || now.isBefore(quietEnd)) → quiet hours.
     Gece geçiyor (22:00-08:00): quietStart.isAfter(quietEnd) → özel kontrol.
  
  c) Kullanıcı timezone tespiti:
     Kayıt: IP'den timezone tahmin et → default öner → kullanıcı onaylasın.
     Mobil: device timezone API → otomatik güncelle.
     Yurt dışına çıktı: yeni timezone → bildirimler yeni timezone'a uyum sağlasın.
  
  d) Quiet hours birikenler 08:00'de spike:
     Tüm Türk kullanıcılar 22:00-08:00 → 08:00'de hepsi birden → burst.
     Çözüm: 08:00-08:15 arası random jitter ekle → yay.
```

---

### 4. Token Birikmesi — Geçersiz Push Token'lar DB'yi Doldurdu

```
Sorun:
  Kullanıcı uygulamayı sildi → token artık geçersiz.
  Token DB'de duruyor → her bildirimde FCM'e gönderiliyor.
  FCM: 404 (DeviceNotRegistered) → token geçersiz.
  1M geçersiz token → 1M gereksiz FCM request/gün → maliyet ve yavaşlık.

Neden birikirdi:
  Token cleanup sistemi yoktu → her 404'te silmiyorduk.
  Kullanıcı uygulamayı silince bildirim gelmiyor → fark edilmiyor.

Çözüm:
  a) 404 anında sil:
     FCM 404 → DeviceNotRegistered → o tokeni hemen deactivate et.
     APNs 410 → DeviceNotRegistered → sil.
     Worker: InvalidTokenException → deviceService.deactivateToken(token).

  b) Periyodik temizleme:
     Günlük job: son 90 gün bildirim gönderilmeyen token → soft delete.
     Push sonrası başarısız → retry_count > 3 → passive işaretle.

  c) APNs Feedback Service:
     APNs: /3/device/{token} HTTP 410 → "Bu cihaz artık bu uygulamayı kullanmıyor."
     Periyodik: tüm tokenları feedback endpoint ile kontrol et.
     Apple deprecated etmedi (HTTP/2 API'de response anında geliyor).

  d) Token refresh zorunluluğu:
     Client: onTokenRefresh() → her zaman backend'e bildir.
     Eski token + yeni token çakışması → PUT /devices/{deviceId} ile güncelle.
     Duplicate token → UNIQUE constraint → insert → conflict → update.
```

---

### 5. Email Deliverability Düştü — Maillar Spam'e Düşüyor

```
Sorun:
  Yeni kampanya: opt-in olmayan kullanıcılara email gönderildi.
  Gmail: spam şikayeti %0.3'e çıktı (Gmail bulk sender threshold: %0.1).
  Google: domain'i "bulk spammer" listesine aldı.
  Tüm emaillar spam klasörüne düşmeye başladı → open rate %2'ye indi.

Çözüm:
  a) Permission only:
     Sadece açıkça "email almak istiyorum" diyen kullanıcılara gönder.
     Double opt-in: email al → confirm link tıkla → sonra gönder.
     Eski kayıtlar: opt-in durumu belli değilse → confirmation email gönder.

  b) Spam şikayeti monitoring:
     SendGrid: feedback loop → şikayetler gerçek zamanlı.
     Alert: complaint rate > %0.05 → marketing emailları durdur, araştır.
     Postmaster Tools (Google): domain reputation → "High", "Medium", "Low".

  c) List hijiene:
     Bounce olan emailları temizle → eski/geçersiz adresler.
     Son 6 ayda açmayan → unsubscribe email gönder → yanıt yok → listeden çıkar.
     Satın alınmış email listesi KULLANMA → en büyük spam sinyali.

  d) Email içeriği:
     Spam trigger keywords: "FREE!!! CLICK NOW!!!" → spam filtresi.
     Metin/resim oranı: sadece resim email → spam şüpheli.
     Unsubscribe linki: RFC 2369 zorunlu, gizleme → spam filtresi.

  e) IP warm-up:
     Yeni IP: günde 100 → 500 → 2,000 → kademeli artış (4-6 hafta).
     Reputation oluşmadan burst → otomatik spam işaretleme.
```

---

### 6. SMS Teslim Edilemedi — Sınır Ötesi Ülke Engeli

```
Sorun:
  Türk kullanıcı yurt dışına çıktı (Almanya).
  Sistem: TR numara → Twilio → Alman operatör → engellendi.
  OTP kodu gilemedi → hesaba erişemedi.

Neden:
  Bazı ülke operatörleri: A2P SMS (uygulama-to-kişi) uluslararası → engeller.
  Sender ID: +1 numara → Avrupa → şüpheli → engel.
  Kısa kodlar: ülke bazlı → Türk kısa kodu Almanya'da çalışmaz.

Çözüm:
  a) Ülke bazlı provider routing:
     Alıcı numarası ülkesi → Almanya → Almanya'da route eden provider seç.
     Nexmo: local presence → daha iyi delivery.
     ülke_kodu → provider map → otomatik routing.

  b) OTP için sesli arama fallback:
     SMS 60sn içinde teslim edilmedi → otomatik IVR çağrısı.
     "Doğrulama kodunuz: X Y Z. Tekrar: X Y Z."
     Twilio Verify API: SMS + sesli arama otomatik yönetir.

  c) Totp / TOTP app (uzun vadeli):
     Google Authenticator, Authy → SMS'e bağımlılık ortadan kalkar.
     SMS: fallback → app önce.
     Kullanıcı SMS tercih etsin ama uygulamayı daha çok öner.

  d) WhatsApp Business API:
     WhatsApp: dünya genelinde yüksek penetrasyon.
     İnternete bağlıysa: OTP WhatsApp'tan → telco bağımsız.
     Twilio WhatsApp: aynı API, farklı channel.
```

---

### 7. In-App Bildirim WebSocket Bağlantısı Kesildi — Bildirimleri Kaçırdı

```
Sorun:
  Kullanıcı mobil uygulaması: WebSocket bağlantısı kurdu → 5 dakika sonra ağ değişti.
  WebSocket kesildi → sunucu fark etti (ping/pong timeout: 30sn).
  Bu 30sn arasında: 3 önemli bildirim → WebSocket'e push → kayıp!
  Kullanıcı uygulamayı kapayıp açtı → "3 bildirim kaçırdım."

Çözüm:
  a) Last-seen sequence number:
     Her bildirim: sequence_number monoton artan (per user).
     Client: son gördüğü sequence = 42.
     WebSocket yeniden bağlandı → "lastSeq=42" gönder.
     Server: sequence > 42 olan tüm bildirimleri toplu gönder (catch-up).

  b) Notification center DB:
     Tüm in-app bildirimler: DynamoDB'ye yazılır (kayıp yok).
     App açıldı → GET /notifications?since={last_seen_at} → eksikleri çek.
     WebSocket: gerçek zamanlı güncelleme, DB: kalıcı kayıt.

  c) Heartbeat / Ping-Pong:
     Client: her 25sn → ping.
     Server: 30sn içinde ping gelmezse → bağlantı ölü → temizle.
     Client: bağlantı koptu → 1sn, 2sn, 4sn, 8sn → exponential backoff reconnect.

  d) Push notification + in-app:
     Kritik bildirim: hem push (APNs/FCM) hem in-app → birini kaçırsa diğeri gelir.
     Marketing: sadece in-app (push izni yoksa bile görür).
     Duplike önleme: her ikisi de gelirse → notification_id ile dedupe.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Queue sistemi | RabbitMQ | Kafka | Kafka: replay, DLT, high throughput, partition |
| Priority | Tek queue | Ayrı topic (P0/P1/P2) | Tek: marketing spike → OTP gecikir |
| Push provider | Tek (FCM only) | FCM + APNs + fallback | iOS: sadece APNs; redundancy gerekli |
| Email provider | Tek SendGrid | SendGrid + Mailgun fallback | Provider outage → tek puan arıza |
| SMS routing | Global tek provider | Ülke bazlı routing | Global: bazı ülkelerde teslim sorunu |
| Zamanlama | Cron job | Redis ZSET + poller | Cron: dakika hassasiyeti yok, spike |
| Rate limit | Uygulama içi | Redis (atomik) | Uygulama içi: cluster'da tutarsız sayaç |
| Idempotency | Yok | DB check + Redis lock | Yok: at-least-once → duplicate bildirim |
| In-app güncelleme | Polling (30sn) | WebSocket + catch-up | Polling: gecikme ve DB yükü |
| Token temizliği | Manuel | 404 anında + daily job | Manuel: geçersiz token birikir, maliyet artar |
| Quiet hours | UTC bazlı | IANA timezone | UTC: gece bildirimlerine neden olur |
| Gruplama | Her bildirim ayrı | Backend collapse + APNs thread | Ayrı: 50 bildirim → rahatsız edici |
