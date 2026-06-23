# 06c — Backend for Frontend (BFF) Pattern

## Ne?

**BFF (Backend for Frontend):** Her client tipi (web, mobil, partner API) için ayrı bir API katmanı. Her BFF, o client'ın ihtiyaçlarına özel veri şekillendirme, aggregation ve protokol dönüşümü yapar.

```
Olmadan (Generic API):
  [Web App]  ──────────────────────→ [Generic API]
  [iOS App]  ──────────────────────→ [Generic API] → [Microservices]
  [Android]  ──────────────────────→ [Generic API]
  [Partner]  ──────────────────────→ [Generic API]

BFF ile:
  [Web App]  → [Web BFF]     ──────→ [OrderService]
  [iOS App]  → [Mobile BFF]  ──────→ [UserService]  → Microservices
  [Android]  → [Mobile BFF]  ──────→ [PaymentService]
  [Partner]  → [Partner BFF] ──────→ [CatalogService]
```

---

## Neden?

**Çözdüğü problem:** Farklı client'ların farklı ihtiyaçları var. Generic API her ikisini de yarım karşılar.

```
Senaryo — Sipariş listesi:

Web app ihtiyacı:
  - Sipariş + müşteri detayları + ürün görselleri + fatura bilgisi
  - 1 request'te 50 siparişin tamamı

Mobile app ihtiyacı:
  - Sadece sipariş no, tutar, durum (küçük ekran)
  - Sayfalama: 10'ar 10'ar
  - Düşük bant genişliği (3G/4G)
  - Önce kritik alan, görseller lazy load

Generic API ile sorunlar:
  Mobile: fazla veri alıyor → over-fetching → bant genişliği israfı
  Web: birden fazla servis çağırıp birleştirmek zorunda → N+1 çağrı
  Her ikisi de tam istediğini alamıyor
```

---

## Nasıl?

### Web BFF

```java
// Web BFF — React/Angular dashboard için
@RestController
@RequestMapping("/web/api")
class WebOrderBff {

    @Autowired OrderServiceClient orderClient;
    @Autowired UserServiceClient userClient;
    @Autowired PaymentServiceClient paymentClient;
    @Autowired ProductServiceClient productClient;

    @GetMapping("/orders/dashboard")
    DashboardResponse getDashboard(@AuthenticationPrincipal User user) {
        // Web dashboard için tek request'te tüm veri
        List<Order> orders = orderClient.getRecentOrders(user.getId(), 50);

        // Parallel calls — CompletableFuture
        List<CompletableFuture<OrderDashboardItem>> futures = orders.stream()
            .map(order -> CompletableFuture.supplyAsync(() -> {
                CustomerDto customer = userClient.getCustomer(order.getCustomerId());
                List<ProductDto> products = productClient.getProducts(order.getProductIds());
                InvoiceDto invoice = paymentClient.getInvoice(order.getId());

                return OrderDashboardItem.builder()
                    .orderId(order.getId())
                    .customer(customer)
                    .products(products)
                    .invoice(invoice)
                    .totalAmount(order.getTotal())
                    .status(order.getStatus())
                    .createdAt(order.getCreatedAt())
                    .build();
            }))
            .toList();

        List<OrderDashboardItem> items = futures.stream()
            .map(CompletableFuture::join)
            .toList();

        return DashboardResponse.builder()
            .orders(items)
            .totalCount(items.size())
            .build();
    }
}
```

### Mobile BFF

```java
// Mobile BFF — iOS/Android için
@RestController
@RequestMapping("/mobile/api")
class MobileOrderBff {

    @GetMapping("/orders")
    PagedResponse<MobileOrderSummary> getOrders(
            @AuthenticationPrincipal User user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {

        // Mobil için minimal veri — sadece gereken alanlar
        Page<Order> orders = orderClient.getOrders(user.getId(),
            PageRequest.of(page, size));

        List<MobileOrderSummary> items = orders.getContent().stream()
            .map(order -> MobileOrderSummary.builder()
                .orderId(order.getId())
                .statusLabel(localizeStatus(order.getStatus()))  // lokalizasyon
                .totalAmount(formatCurrency(order.getTotal()))   // formatlanmış
                .orderDate(formatDate(order.getCreatedAt()))     // human-readable
                // Ürün görseli yok (mobil bant genişliği için)
                .build())
            .toList();

        return PagedResponse.<MobileOrderSummary>builder()
            .content(items)
            .page(page)
            .size(size)
            .totalPages(orders.getTotalPages())
            .hasNext(orders.hasNext())
            .build();
    }

    // Mobile-specific: push notification token kayıt
    @PostMapping("/device/register")
    void registerDevice(@RequestBody DeviceRegistrationRequest req,
                        @AuthenticationPrincipal User user) {
        notificationService.registerDevice(user.getId(), req.getFcmToken());
    }
}
```

### Partner BFF (Rate-Limited, Authenticated)

```java
// Partner BFF — 3rd party entegrasyon için
@RestController
@RequestMapping("/partner/api/v1")
@RateLimiter(name = "partner-api")
class PartnerBff {

    // Partner'ın ihtiyacı: sipariş durumu + tracking
    @GetMapping("/orders/{externalOrderId}/status")
    PartnerOrderStatus getOrderStatus(
            @PathVariable String externalOrderId,
            @RequestHeader("X-Partner-Key") String partnerKey) {

        partnerAuthService.validateKey(partnerKey);

        // External order ID → internal ID mapping
        Order order = orderClient.findByExternalId(externalOrderId)
            .orElseThrow(() -> new OrderNotFoundException(externalOrderId));

        TrackingInfo tracking = shippingClient.getTracking(order.getId());

        // Partner API contract — sadece partner'ın ihtiyacı olan alanlar
        return PartnerOrderStatus.builder()
            .externalOrderId(externalOrderId)
            .status(order.getStatus().toPartnerStatus())  // mapping
            .estimatedDelivery(tracking.getEstimatedDelivery())
            .trackingUrl(tracking.getPublicTrackingUrl())
            .build();
    }
}
```

---

### GraphQL ile BFF (Modern Yaklaşım)

```graphql
# GraphQL BFF — client her query'de ihtiyacı kadar veri seçer
# Web app fazla isteyebilir, mobile az

type Query {
  order(id: ID!): Order
  orders(customerId: ID!, first: Int, after: String): OrderConnection
}

type Order {
  id: ID!
  status: OrderStatus!
  totalAmount: Money!
  customer: Customer        # Web alır, mobile almayabilir
  items: [OrderItem!]!      # Web 100, mobile 3 alabilir
  invoice: Invoice          # Sadece web dashboard alır
}
```

```java
// Spring for GraphQL
@Controller
class OrderGraphQLController {

    @QueryMapping
    Order order(@Argument String id) {
        return orderService.findById(id);
    }

    // Lazy loading — sadece istenen alanlar için çağrı yapılır
    @SchemaMapping(typeName = "Order")
    Customer customer(Order order) {
        return userClient.getCustomer(order.getCustomerId());
    }

    @SchemaMapping(typeName = "Order")
    Invoice invoice(Order order) {
        return paymentClient.getInvoice(order.getId());
    }
}
```

---

### BFF Organizasyonu — Kim Yazar?

```
Seçenek 1: Frontend ekibi BFF'i yazar (önerilen):
  React ekibi → Web BFF (Node.js/TypeScript)
  iOS ekibi   → Mobile BFF (Swift veya shared Node.js)
  
  Avantaj: Frontend'in kendi ihtiyacını backend'e bağımlı olmadan şekillendirir
  Avantaj: Backend microservice'ler değişince BFF güncellenir, client etkilenmez

Seçenek 2: Backend ekibi yazar:
  Avantaj: Backend domain bilgisi
  Dezavantaj: Frontend'in ne istediğini tam bilmeyebilir

Pratikte: BFF = "experience layer" — frontend odaklı bir backend katmanı
```

---

## Ne zaman?

```
BFF kullan:
✓ Web + Mobile + Partner gibi farklı client tipleri var
✓ Her client'ın veri ihtiyacı çok farklı
✓ Mobile'da over-fetching sorun (bant genişliği)
✓ Frontend farklı teknoloji seçmek istiyor (GraphQL, REST, WebSocket)
✓ Client-specific auth/rate limiting gerekiyor
✓ Frontend ekibi backend'den bağımsız deployment istiyor

BFF kullanma:
✗ Tek tip client (sadece web)
✗ Küçük ekip — her BFF ayrı bakım yükü
✗ Tüm client'ların veri ihtiyacı neredeyse aynı
✗ API Gateway zaten yeterli (aggregation, routing)

Alternatifler:
  GraphQL: Client-driven query → BFF'e gerek kalmayabilir
  API Gateway + Backend Aggregation Service
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| Her client'a optimize veri | Her BFF ayrı servis → deploy, monitor |
| Frontend bağımsızlığı | Kod tekrarı (benzer logic birden fazla BFF'te) |
| Güvenli aggregation (client'a çok veri expose edilmez) | Ekstra network hop (client → BFF → microservices) |
| Client-specific rate limiting / auth | BFF'in kendi bottleneck olmamasına dikkat |
| Microservices değişince sadece BFF güncellenir | "Monolithic BFF" anti-pattern (çok büyüyen BFF) |
