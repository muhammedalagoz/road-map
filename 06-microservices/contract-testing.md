# 06d — Contract Testing (Pact)

## Ne?

**Contract Testing:** Bir servisin (consumer) bir diğer servisten (provider) ne beklediğini "sözleşme" (contract) olarak kaydedip, provider'ın bu sözleşmeyi karşılayıp karşılamadığını otomatik doğrulama yöntemi.

```
Consumer-Driven Contract Testing:
  Consumer (OrderService) → "PaymentService'ten şunu bekliyorum" → contract yazar
  Provider (PaymentService) → "bu contract'ı karşılıyor muyum?" → otomatik test

Pact: En yaygın contract testing framework'ü (Java, JS, Go, Python, .NET, Ruby)
```

---

## Neden?

**Çözdüğü problem:** Microservices entegrasyon testleri pahalı ve yavaş; ama entegrasyon testleri olmadan bir servis güncellenmesi diğerini bozuyor.

```
E2E test sorunları:
  10 microservice + E2E test → tüm servisler ayakta olmalı
  Test yavaş: 30-60 dakika
  Flaky: Herhangi bir servis instabil → tüm test başarısız
  Debugging zor: hangi servis bozuk?

Contract test avantajı:
  Consumer test: sadece consumer + mock provider → hızlı, izole
  Provider test: sadece provider + contract doğrulama → hızlı, izole
  E2E gerekmez → dakikalar içinde çalışır
  Tam olarak hangi contract kırıldı → anında belli
```

---

## Nasıl?

### Pact ile Consumer-Driven Contract Testing

#### Adım 1 — Consumer (OrderService) contract yazar

```java
// pom.xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>

// Consumer test — OrderService'in PaymentService'ten ne beklediği
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service")
class OrderServicePactConsumerTest {

    @Pact(consumer = "order-service", provider = "payment-service")
    RequestResponsePact createPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("payment service is available")   // provider state
            .uponReceiving("a charge request for order-123")
            .path("/api/payments")
            .method("POST")
            .headers(Map.of(
                "Content-Type", "application/json",
                "Authorization", "Bearer token"
            ))
            .body(new PactDslJsonBody()
                .stringType("orderId", "order-123")      // type matching
                .decimalType("amount", 99.99)
                .stringType("currency", "TRY")
            )
            .willRespondWith()
            .status(201)
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .uuid("paymentId")                        // UUID formatında
                .stringValue("status", "CHARGED")         // exact value
                .decimalType("chargedAmount")             // type: decimal
                .datetime("processedAt", "yyyy-MM-dd'T'HH:mm:ss") // format
            )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "createPaymentPact")
    void shouldChargePayment(MockServer mockServer) {
        // Pact → mock PaymentService başlatır (mockServer.getUrl())
        PaymentClient client = new PaymentClient(mockServer.getUrl());

        PaymentResponse response = client.charge(ChargeRequest.builder()
            .orderId("order-123")
            .amount(new BigDecimal("99.99"))
            .currency("TRY")
            .build());

        assertThat(response.getStatus()).isEqualTo("CHARGED");
        assertThat(response.getPaymentId()).isNotNull();
    }
}
// Test çalışınca → target/pacts/order-service-payment-service.json oluşur
```

**Oluşan Pact dosyası:**
```json
{
  "consumer": { "name": "order-service" },
  "provider": { "name": "payment-service" },
  "interactions": [
    {
      "description": "a charge request for order-123",
      "providerStates": [{"name": "payment service is available"}],
      "request": {
        "method": "POST",
        "path": "/api/payments",
        "body": {
          "orderId": "order-123",
          "amount": 99.99,
          "currency": "TRY"
        },
        "matchingRules": {
          "body": {
            "$.orderId": {"matchers": [{"match": "type"}]},
            "$.amount": {"matchers": [{"match": "decimal"}]}
          }
        }
      },
      "response": {
        "status": 201,
        "body": {
          "paymentId": "550e8400-e29b-41d4-a716-446655440000",
          "status": "CHARGED"
        }
      }
    }
  ]
}
```

---

#### Adım 2 — Pact Broker'a Publish

```bash
# CI pipeline'da pact dosyasını Pact Broker'a gönder
./mvnw pact:publish \
  -Dpact.broker.url=https://pact-broker.internal \
  -Dpact.broker.token=${PACT_TOKEN} \
  -Dpact.consumer.version=${GIT_COMMIT}
```

---

#### Adım 3 — Provider (PaymentService) contract doğrular

```java
// Provider verification — PaymentService kendi testlerinde contract'ı doğrular
@Provider("payment-service")
@PactBroker(
    url = "https://pact-broker.internal",
    authentication = @PactBrokerAuth(token = "${PACT_TOKEN}")
)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class PaymentServicePactProviderTest {

    @LocalServerPort
    int port;

    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Provider state handler — consumer'ın "given" kısmını setup et
    @State("payment service is available")
    void setupPaymentServiceAvailable() {
        // Test için herhangi bir setup gerekmiyorsa boş bırakılabilir
    }

    @State("payment with id pay-999 exists")
    void setupPaymentExists() {
        // Test DB'ye payment ekle
        paymentTestRepo.save(TestPayment.builder()
            .id("pay-999")
            .status("CHARGED")
            .build());
    }
}
```

---

### Can I Deploy? — Pact Broker

```bash
# Deploy öncesi kontrol:
# "order-service v1.2.3 → payment-service'in prod sürümüyle uyumlu mu?"
pact-broker can-i-deploy \
  --pacticipant order-service \
  --version 1.2.3 \
  --to-environment production

# Çıktı:
# ✅ order-service v1.2.3 → payment-service v2.1.0 (prod): COMPATIBLE
# ❌ order-service v1.2.3 → inventory-service v1.5.0 (prod): INCOMPATIBLE
#    Reason: contract "inventory.reservation" not verified
```

---

### Pact Broker Dashboard

```
Pact Broker Web UI:
  ┌─────────────────────────────────────────────────────┐
  │  Consumer          Provider       Status    Version  │
  │  ──────────────    ──────────     ──────    ───────  │
  │  order-service  →  payment-svc   ✅ Pass   v2.1.0   │
  │  order-service  →  inventory-svc ❌ FAIL   v1.5.0   │
  │  cart-service   →  catalog-svc   ✅ Pass   v3.0.1   │
  └─────────────────────────────────────────────────────┘

  PaymentService yeni sürüm çıkarmadan önce:
  → Can I Deploy? → tüm consumer'ların contract'larına uyuyor mu?
  → Hayır → breaking change var → contract güncellenmeli → consumer'lar uyarılmalı
```

---

### Message Contract Testing (Kafka/Event)

```java
// Kafka event'leri de contract test edilebilir
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "order-service", providerType = ProviderType.ASYNCH)
class InventoryServicePactConsumerTest {

    @Pact(consumer = "inventory-service", provider = "order-service")
    MessagePact orderCreatedEventPact(MessagePactBuilder builder) {
        return builder
            .expectsToReceive("an OrderCreated event")
            .withContent(new PactDslJsonBody()
                .stringType("orderId")
                .stringType("customerId")
                .array("items")
                    .object()
                        .stringType("productId")
                        .integerType("quantity")
                    .closeObject()
                .closeArray()
                .datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss")
            )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "orderCreatedEventPact")
    void shouldHandleOrderCreatedEvent(List<Message> messages) {
        // Pact mock event ile consumer'ı test et
        Message message = messages.get(0);
        OrderCreatedEvent event = objectMapper.readValue(
            message.contentsAsString(), OrderCreatedEvent.class);

        inventoryService.reserveItems(event.getItems());
        // assert...
    }
}
```

---

## Ne zaman?

```
Contract testing kullan:
✓ Microservices ekibi, farklı takımlar farklı servisleri geliştiriyor
✓ E2E test çok yavaş / pahalı
✓ CI/CD'de "safe to deploy?" kararı otomatize edilmeli
✓ API contract stability önemli (public API, partner)
✓ Sık deployment (haftada 10+ deploy) → her seferinde manual test imkansız

Contract testing yetersiz kalır:
✗ Infrastructure-level testing (DB migrations, network latency)
✗ Gerçek sistem entegrasyonu (performans, yük)
✗ Üçüncü taraf servisler (onların mock'unu yapamazsın — Wiremock ile stub)

Birlikte kullan:
  Unit test (fast, isolated)
  + Contract test (integration confidence, fast)
  + Smoke test (prod'da sanity check)
  → E2E test minimumdur (sadece critical path)
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| E2E testten çok hızlı | Pact Broker kurulumu gerekir |
| CI'da "can-i-deploy" kararı otomatik | Ekip disiplini gerekir (consumer contract yazmalı) |
| Breaking change anında tespit edilir | Provider state setup karmaşık olabilir |
| Servis bağımsız test edilir | Pact öğrenme eğrisi |
| Consumer-driven: provider API'yi consumer'a göre tasarlar | Sözleşme gerçek davranışı %100 test etmez (integration gerekir) |
