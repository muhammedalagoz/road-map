# 08f — System Design: Notification Sistemi

## Gereksinimler

```
Functional:
  ✓ Push notification (iOS APNs, Android FCM)
  ✓ Email bildirimi
  ✓ SMS bildirimi
  ✓ In-app notification
  ✓ Kullanıcı tercihleri (kanal seçimi, sessiz mod)
  ✓ Template yönetimi (kişiselleştirilmiş içerik)

Non-Functional:
  10M bildirim/gün
  Soft real-time: <30 saniye gecikme kabul
  Delivery guarantee: at-least-once
  Scheduled notification desteği
  Rate limiting (spam önleme)
```

---

## Capacity Estimation

```
10M bildirim/gün:
  Average QPS: 10M / 86400 ≈ 116/sn
  Peak (kampanya/event): 10,000/sn

Kanal dağılımı (tipik):
  Push: %60 → 6M/gün
  Email: %30 → 3M/gün
  SMS:   %10 → 1M/gün

Storage:
  Her bildirim kaydı: ~1 KB
  10M × 1KB = 10 GB/gün
  Retention 90 gün: 900 GB
```

---

## High-Level Tasarım

```
                    Trigger Sources
           ┌────────────┬──────────────┬──────────────┐
           │            │              │              │
      [Order Svc]  [Marketing]   [Scheduler]   [User Action]
           │            │              │              │
           └────────────┴──────────────┴──────────────┘
                                │
                    ┌───────────┴───────────┐
                    │   Notification API    │
                    │   (validation, rate   │
                    │    limit, routing)    │
                    └───────────┬───────────┘
                                │
                        Kafka Topics
                    ┌───────────┴────────────┐
              [push.queue]  [email.queue]  [sms.queue]
                    │            │              │
              ┌─────┴────┐  ┌───┴───┐    ┌────┴────┐
              │  Push    │  │ Email │    │   SMS   │
              │ Worker   │  │Worker │    │ Worker  │
              └──┬───┬───┘  └───┬───┘    └────┬────┘
                 │   │          │              │
              APNs FCM       SendGrid      Twilio
                             Mailgun
```

---

## Notification API

```java
// API — notification gönderme isteği
@RestController
@RequestMapping("/api/notifications")
class NotificationController {

    @PostMapping("/send")
    void sendNotification(@RequestBody NotificationRequest req) {
        // 1. Validasyon
        validator.validate(req);

        // 2. Kullanıcı tercihleri kontrolü
        UserPreference pref = preferenceService.get(req.getUserId());
        if (!pref.isChannelEnabled(req.getChannel())) {
            return; // kanal kapalı
        }
        if (pref.isQuietHours(req.getChannel())) {
            scheduleLater(req, pref.getQuietHoursEnd()); // sonraya ertele
            return;
        }

        // 3. Rate limiting
        rateLimiter.check("user:" + req.getUserId(), req.getChannel());

        // 4. Template render
        String content = templateEngine.render(
            req.getTemplateId(), req.getTemplateParams());

        // 5. Kafka'ya publish
        String topic = resolveKafkaTopic(req.getChannel());
        kafkaTemplate.send(topic, NotificationMessage.builder()
            .notificationId(UUID.randomUUID().toString())
            .userId(req.getUserId())
            .channel(req.getChannel())
            .content(content)
            .priority(req.getPriority())
            .build());
    }
}
```

---

## Push Notification Worker

```java
// FCM/APNs gönderimi
@Component
class PushNotificationWorker {

    @KafkaListener(topics = "push.queue", concurrency = "10")
    void process(NotificationMessage message) {
        UserDevice device = deviceService.getDevice(message.getUserId());

        if (device == null) {
            log.warn("No device token for user: {}", message.getUserId());
            return;
        }

        try {
            if (device.isIos()) {
                apnsService.send(ApnsNotification.builder()
                    .deviceToken(device.getToken())
                    .title(message.getTitle())
                    .body(message.getContent())
                    .badge(device.getUnreadCount())
                    .build());
            } else {
                fcmService.send(FcmMessage.builder()
                    .token(device.getToken())
                    .notification(FcmNotification.builder()
                        .title(message.getTitle())
                        .body(message.getContent())
                        .build())
                    .data(message.getExtraData())
                    .build());
            }

            // Başarılı — kayıt et
            deliveryRepo.markDelivered(message.getNotificationId());

        } catch (InvalidTokenException e) {
            // Token geçersiz → cihazı sil
            deviceService.removeDevice(device.getToken());
        } catch (Exception e) {
            // Retry için tekrar kuyruğa
            throw e; // Kafka retry mekanizması devreye girer
        }
    }
}
```

---

## Email Worker

```java
@Component
class EmailWorker {

    @KafkaListener(topics = "email.queue", concurrency = "5")
    void process(NotificationMessage message) {
        UserProfile user = userService.getProfile(message.getUserId());

        EmailRequest email = EmailRequest.builder()
            .to(user.getEmail())
            .subject(message.getTitle())
            .htmlBody(message.getContent())
            .textBody(stripHtml(message.getContent()))
            .from("noreply@myapp.com")
            .replyTo("support@myapp.com")
            .build();

        // SendGrid / Mailgun API
        sendGridClient.send(email);

        // Unsubscribe tracking
        trackingService.trackSent(message.getNotificationId(), user.getEmail());
    }
}
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
    quiet_start     TIME,                          -- 22:00
    quiet_end       TIME,                          -- 08:00
    timezone        VARCHAR(50)  DEFAULT 'UTC',
    marketing_push  BOOLEAN      DEFAULT TRUE,
    marketing_email BOOLEAN      DEFAULT TRUE,
    updated_at      TIMESTAMP
);

-- Notification kategorileri (fine-grained kontrol)
CREATE TABLE notification_category_prefs (
    user_id     UUID,
    category    VARCHAR(50),  -- ORDER_UPDATE, PROMOTION, SECURITY, CHAT
    push        BOOLEAN DEFAULT TRUE,
    email       BOOLEAN DEFAULT TRUE,
    sms         BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (user_id, category)
);
```

---

## Scheduled Notifications

```
Zamanlı bildirim (kampanya, hatırlatıcı):

Seçenek 1: Cron job
  Her gece 00:00 → "Yarın gönderilecek" bildirimleri Kafka'ya yükle
  ✗ Peak yük (tüm bildirimleri aynı anda Kafka'ya at)

Seçenek 2: Delayed Kafka
  Kafka mesajını timestamp ile kaydet → Scheduler delivery'yi bekletir
  ✓ Daha düzgün dağıtım

Seçenek 3: Redis Sorted Set + Scheduler (önerilen)
  scheduled_notifications: {notificationId → sendAt timestamp}

  ZADD scheduled_notifications {sendAt} {notificationId}
  
  Scheduler (her 1s):
    ZRANGEBYSCORE scheduled_notifications 0 {now} LIMIT 0 1000
    → Kafka'ya push et
    → ZREM scheduled_notifications {notificationIds}
```

---

## Rate Limiting & Anti-Spam

```
Kullanıcı başına:
  Push:  max 5/gün marketing, unlimited transactional
  Email: max 3/gün marketing, unlimited transactional
  SMS:   max 2/gün, unlimited OTP/security

Tip bazlı:
  SECURITY (2FA, şifre sıfırlama) → rate limit yok
  ORDER_UPDATE → limit yok
  PROMOTION → günlük limit

Redis ile rate limit:
  Key: ratelimit:user:{userId}:push:marketing:{date}
  INCR → count > 5 → skip
  EXPIRE: 86400 (gün sonu reset)

Bounce handling (email):
  Hard bounce (email yok) → email'i blacklist'e al
  Soft bounce (mailbox dolu) → 3 gün retry → hard bounce muamelesi
```

---

## Delivery Tracking

```sql
CREATE TABLE notification_delivery (
    notification_id  UUID         PRIMARY KEY,
    user_id          UUID         NOT NULL,
    channel          VARCHAR(20),    -- PUSH, EMAIL, SMS
    status           VARCHAR(20),    -- PENDING, SENT, DELIVERED, FAILED, READ
    provider         VARCHAR(50),    -- FCM, APNS, SENDGRID, TWILIO
    provider_msg_id  VARCHAR(255),   -- Provider'dan dönen ID (tracking için)
    sent_at          TIMESTAMP,
    delivered_at     TIMESTAMP,
    read_at          TIMESTAMP,
    failure_reason   TEXT,
    retry_count      INT DEFAULT 0
);

-- Dashboard metrikleri:
-- Delivery rate = DELIVERED / SENT
-- Open rate = READ / DELIVERED
-- Bounce rate = FAILED (hard bounce) / SENT
```

---

## Retry Stratejisi

```
Geçici hata (provider timeout, rate limit):
  → Kafka retry topic: push.queue.retry-1 (1 dk bekle)
  → push.queue.retry-2 (5 dk bekle)
  → push.queue.retry-3 (30 dk bekle)
  → push.queue.DLT (dead letter, 3 deneme sonrası)

Kalıcı hata (invalid token, user unsubscribed):
  → Hemen DLT'ye at (retry etme)
  → Device/email cleanup

DLT monitoring:
  DLT'deki mesajlar → Alert → Manuel inceleme veya otomatik re-process
```

---

## Trade-off Özeti

| Karar | Seçenek | Gerekçe |
|-------|---------|---------|
| Queue | Kafka | Retry, DLT, paralel consumer |
| Push provider | FCM + APNs | Multi-platform |
| Email provider | SendGrid/Mailgun | Deliverability, bounce mgmt |
| Schedule | Redis ZSET | Dakika hassasiyeti, ölçeklenebilir |
| Rate limit | Redis | Atomic increment, TTL |
