# 04g — MQTT

## Ne?

**MQTT (Message Queuing Telemetry Transport):** IoT ve mobil cihazlar için tasarlanmış, son derece hafif (2 byte minimum header), düşük bant genişliği ve yüksek güvenilirlik odaklı pub/sub mesajlaşma protokolü. TCP/IP üzerinde çalışır; broker üzerinden cihazlar birbirleriyle haberleşir.

---

## Neden?

**Çözdüğü problem:** IoT cihazları (sensör, mikrodenetleyici) kısıtlı bellek ve işlemci gücüne sahip. AMQP/HTTP gibi ağır protokollerle çalışamazlar. MQTT bu cihazlar için tasarlanmış: minimal header, bant genişliği tasarrufu, bağlantı kesilmesine karşı dayanıklılık.

```
HTTP/REST ile IoT:
  Header: ~200-800 byte (her request)
  Connection: Her request yeni TCP (veya keep-alive ama stateful değil)
  Sorun: Sensörün her 5 saniyede data göndermesi →
         düşük bant genişlikli ağlarda (LoRa, GSM) mümkün değil

MQTT ile IoT:
  Header: minimum 2 byte
  Connection: Kalıcı TCP bağlantısı (broker ile)
  Heartbeat: PINGREQ/PINGRESP (bağlantı canlı mı kontrol)
  Offline mesaj: QoS 1/2 ile broker saklar, cihaz gelince teslim eder
```

---

## Nasıl?

### Temel Model

```
MQTT Broker (merkez)
  ↑        ↓
Publisher  Subscriber
(Sensör)   (Sunucu/Uygulama)

Tüm iletişim broker üzerinden (direkt cihaz-cihaz yok)

Yaygın broker'lar:
  Mosquitto (open-source, hafif, embedded için)
  EMQX (enterprise, yüksek ölçek)
  HiveMQ (enterprise, clustering)
  AWS IoT Core (managed, cloud)
  Azure IoT Hub (managed, cloud)
```

---

### Topic Yapısı

```
Hierarchical topic (/ ile ayrılır):
  home/floor1/room1/temperature
  home/floor1/room1/humidity
  home/floor2/room3/motion
  factory/line1/machine5/speed
  factory/line1/machine5/vibration

Wildcard subscription:
  home/+/room1/temperature  → + = tam olarak 1 seviye
    eşleşir: home/floor1/room1/temperature
             home/floor2/room1/temperature
    eşleşmez: home/floor1/lounge/room1/temperature (2 seviye)

  home/#                    → # = 0 veya daha fazla seviye (sona gelir)
    eşleşir: home/floor1/room1/temperature
             home/floor2/room3/motion/sensor
             home/garden
    eşleşmez: factory/... (home ile başlamıyor)

  $SYS/#                   → broker istatistikleri (özel prefix)
```

---

### QoS (Quality of Service)

```
QoS 0 — At most once (Fire and forget):
  Publisher → Broker → Subscriber
  Garantisi yok. Mesaj kaybolabilir.
  En hızlı, en düşük overhead.
  Kullanım: yüksek frekanslı sensör verisi (bir kayıp kabul edilebilir)

QoS 1 — At least once:
  Publisher → Broker (PUBACK beklenir)
  Broker → Subscriber (PUBACK beklenir)
  Mesaj en az bir kez teslim edilir. Duplicate olabilir.
  Kullanım: her mesaj önemli ama duplicate tolere edilebilir

QoS 2 — Exactly once (4-way handshake):
  PUBLISH → PUBREC → PUBREL → PUBCOMP
  Her mesaj tam olarak bir kez teslim edilir.
  En yavaş, en yüksek overhead.
  Kullanım: finansal işlem, kritik komutlar (kapat/aç)
```

---

### MQTT 5 Özellikleri (MQTT 3.1.1'den farklar)

```
1. Message Expiry: Belirli süre sonra broker mesajı atar
   properties.setMessageExpiryInterval(3600); // 1 saat

2. User Properties: Key-value metadata
   properties.addUserProperty("region", "eu-west");

3. Reason Codes: Neden başarısız oldu?
   MQTT 3: sadece success/fail
   MQTT 5: 35+ reason code (auth fail, quota exceeded, not authorized...)

4. Shared Subscriptions: Aynı topic'i consumer group gibi dağıt
   $share/workers/sensors/temperature
   → MQTT 5 ile Kafka gibi consumer group

5. Request-Response: correlation ID ile RPC benzeri
   Publisher → correlation ID ekle → broker
   Subscriber cevap → aynı correlation ID → publisher filtreler

6. Flow Control: Client max concurrent işlem sayısını bildirir
```

---

### Spring Boot + MQTT

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
```

```java
@Configuration
class MqttConfig {

    @Bean
    MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[]{"tcp://broker.internal:1883"});
        options.setUserName("device-app");
        options.setPassword("secret".toCharArray());
        options.setCleanSession(false);     // offline mesajları al
        options.setKeepAliveInterval(60);   // 60sn heartbeat
        options.setAutomaticReconnect(true);
        factory.setConnectionOptions(options);
        return factory;
    }

    // Inbound — broker'dan mesaj al
    @Bean
    MessageChannel mqttInputChannel() { return new DirectChannel(); }

    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    MessageProducer inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
            new MqttPahoMessageDrivenChannelAdapter(
                "spring-client-" + UUID.randomUUID(),
                mqttClientFactory(),
                "sensors/+/temperature",   // subscribe
                "sensors/+/humidity"
            );
        adapter.setCompletionTimeout(5000);
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    // Outbound — broker'a mesaj gönder
    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    MessageHandler outbound() {
        MqttPahoMessageHandler handler = new MqttPahoMessageHandler(
            "spring-publisher-" + UUID.randomUUID(),
            mqttClientFactory()
        );
        handler.setAsync(true);
        handler.setDefaultQos(1);
        handler.setDefaultTopic("commands/devices");
        return handler;
    }

    @Bean
    MessageChannel mqttOutboundChannel() { return new DirectChannel(); }
}

// Mesaj işle
@Service
class SensorDataProcessor {

    @ServiceActivator(inputChannel = "mqttInputChannel")
    void processSensorData(Message<String> message) {
        String topic = (String) message.getHeaders()
            .get(MqttHeaders.RECEIVED_TOPIC);
        String payload = message.getPayload();

        // topic: "sensors/device-42/temperature"
        String[] parts = topic.split("/");
        String deviceId = parts[1];
        String dataType = parts[2];

        sensorRepo.save(new SensorReading(deviceId, dataType,
            Double.parseDouble(payload), Instant.now()));
    }
}

// Komut gönder (cihaza)
@Service
class DeviceCommandService {

    @Autowired
    @Qualifier("mqttOutboundChannel")
    MessageChannel outboundChannel;

    void sendCommand(String deviceId, String command) {
        outboundChannel.send(MessageBuilder.withPayload(command)
            .setHeader(MqttHeaders.TOPIC, "commands/" + deviceId)
            .setHeader(MqttHeaders.QOS, 2)      // komutlar exactly-once
            .setHeader(MqttHeaders.RETAINED, false)
            .build());
    }
}
```

---

### Last Will & Testament (LWT)

```
Cihaz bağlantıyı düzgün kapatmazsa (crash, ağ kesilmesi) broker ne yapsın?

MqttConnectOptions options = new MqttConnectOptions();
options.setWill(
    "devices/device-42/status",  // topic
    "offline".getBytes(),         // payload
    1,                            // QoS
    true                          // retained
);

Sonuç:
- Cihaz normalden bağlantı keser → hiçbir şey
- Cihaz crash → broker LWT mesajını yayınlar
- Diğer cihazlar/sunucu "devices/device-42/status" → "offline" alır
- Monitoring sistemi alert gönderir
```

---

### Retained Message

```
Retained = broker bu mesajı saklar, yeni subscriber gelince hemen gönderir

// Cihaz durumunu retained olarak yayınla
publisher.send("devices/device-42/status", "online", retained=true)

// Yeni bir monitoring dashboard bağlandığında:
// → Subscribe: "devices/+/status"
// → Broker anında tüm cihazların SON durumunu gönderir
// → Dashboard güncel state'i hemen görür, mesaj beklemez
```

---

## Ne zaman?

**MQTT kullan:**
```
✓ IoT cihazlar (sensör, ESP32, Arduino, Raspberry Pi)
✓ Kısıtlı bant genişliği (GSM/2G, LoRa, satellite)
✓ Güvenilmez ağ (cihaz sık sık bağlantıyı kopar)
✓ Mobil uygulama push (düşük pil, düşük veri)
✓ Cihaz durum izleme (Last Will ile offline detection)
✓ Home automation (Home Assistant, Zigbee2MQTT)
✓ Binlerce cihaz, düşük mesaj boyutu
```

**MQTT kullanma:**
```
✗ Karmaşık routing → AMQP/RabbitMQ
✗ Yüksek throughput enterprise messaging → Kafka
✗ Mesaj saklama/replay → Kafka
✗ Request-response ağırlıklı → REST/gRPC
✗ Büyük payload → HTTP daha uygun
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Ultra hafif (2 byte header) | Karmaşık routing yok (topic wildcard basit) |
| Düşük bant genişliği tüketimi | Broker SPOF (cluster kurulumu gerekir) |
| Güvenilmez ağlarda dayanıklı (QoS 1/2) | Mesaj saklama sınırlı (sadece retained) |
| LWT ile offline detection | Büyük ölçekte broker yönetimi karmaşık |
| Retained message ile yeni subscriber | Authentication/authorization ek konfigürasyon |
| Persistent session (offline mesaj) | Kafka gibi replay yok |
| Milyonlarca bağlantı desteği | |
