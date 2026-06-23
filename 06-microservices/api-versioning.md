# 06b — API Versioning Stratejileri

## Ne?

**API Versioning:** Mevcut API'yi kullanan client'ları bozmadan yeni API değişiklikleri yayınlama stratejisi.

```
Senaryo:
  v1 API: GET /orders → { "customerId": "123" }
  v2 API: GET /orders → { "customer": { "id": "123", "name": "Ali" } }

  v1 client'lar "customerId" alanını bekler → v2'ye geçince kırılır
  Çözüm: Versioning ile iki versiyonu aynı anda destekle
```

---

## Neden?

**Çözdüğü problem:** Backend ekibi API'yi geliştirir, ama tüm client'lar (mobil app, partner entegrasyonlar, 3rd party) aynı anda güncellenemez.

```
Versioning olmadan:
  API değişti → tüm client'lar aynı anda güncellenmeli
  Mobil app → App Store review süreci (2-7 gün)
  Partner entegrasyonlar → kendi release döngüleri
  Eski sürüm cihazlardaki app'ler → forever broken

Versioning ile:
  v1 destekleniyor → eski client'lar çalışmaya devam
  v2 eklendi → yeni özellik
  v1 deprecation süreci → client'lara migration süresi tanı
  v1 sunset → sonunda kapatılır
```

---

## Nasıl?

### Strateji 1: URL Path Versioning (En Yaygın)

```
GET /api/v1/orders
GET /api/v2/orders

POST /api/v1/payments
POST /api/v2/payments
```

```java
// Spring Boot — URL Path Versioning
@RestController
@RequestMapping("/api/v1/orders")
class OrderControllerV1 {

    @GetMapping("/{id}")
    OrderResponseV1 getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        return OrderResponseV1.builder()
            .id(order.getId())
            .customerId(order.getCustomer().getId())  // flat structure
            .totalAmount(order.getTotal())
            .build();
    }
}

@RestController
@RequestMapping("/api/v2/orders")
class OrderControllerV2 {

    @GetMapping("/{id}")
    OrderResponseV2 getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        return OrderResponseV2.builder()
            .id(order.getId())
            .customer(CustomerDto.from(order.getCustomer()))  // nested object
            .total(MoneyDto.from(order.getTotal()))
            .items(order.getItems().stream().map(ItemDto::from).toList())
            .build();
    }
}
```

**Avantaj:** Basit, URL'de görünür, cache-friendly, belgeleme kolay.
**Dezavantaj:** URL'ler "versiyon" bilgisi taşımamalı (REST puristleri), controller çoğalır.

---

### Strateji 2: Header Versioning

```
GET /api/orders
Accept-Version: v1

GET /api/orders
Accept-Version: v2

veya:
Accept: application/vnd.mycompany.v2+json
```

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

    @GetMapping(value = "/{id}", headers = "Accept-Version=v1")
    OrderResponseV1 getOrderV1(@PathVariable String id) {
        return orderService.getV1(id);
    }

    @GetMapping(value = "/{id}", headers = "Accept-Version=v2")
    OrderResponseV2 getOrderV2(@PathVariable String id) {
        return orderService.getV2(id);
    }
}

// veya Media Type versioning (Content Negotiation)
@GetMapping(
    value = "/{id}",
    produces = "application/vnd.mycompany.order.v1+json"
)
OrderResponseV1 getOrderV1(@PathVariable String id) { ... }

@GetMapping(
    value = "/{id}",
    produces = "application/vnd.mycompany.order.v2+json"
)
OrderResponseV2 getOrderV2(@PathVariable String id) { ... }
```

**Avantaj:** URL temiz, REST prensiplere uygun.
**Dezavantaj:** Tarayıcıda test zor (header göndermek için araç gerekir), cache header bazlı çalışmaz (URL aynı).

---

### Strateji 3: Query Parameter Versioning

```
GET /api/orders?version=1
GET /api/orders?version=2
```

```java
@GetMapping("/orders/{id}")
ResponseEntity<?> getOrder(
        @PathVariable String id,
        @RequestParam(defaultValue = "2") int version) {

    Order order = orderService.findById(id);

    return switch (version) {
        case 1 -> ResponseEntity.ok(OrderResponseV1.from(order));
        case 2 -> ResponseEntity.ok(OrderResponseV2.from(order));
        default -> ResponseEntity.badRequest().body("Unsupported version: " + version);
    };
}
```

**Avantaj:** URL temiz, test kolay (tarayıcıda).
**Dezavantaj:** Query param karmaşıklığı, cache bozabilir.

---

### Strateji 4: API Gateway ile Versioning (Önerilen)

```
Client                 API Gateway              Service
  │                        │                       │
  ├── GET /v1/orders ──────►│                       │
  │                        │── GET /orders ─────────►│
  │                        │   X-API-Version: 1     │
  │◄── v1 response ────────┤◄── response ───────────┤
  │                        │                       │
  ├── GET /v2/orders ──────►│                       │
  │                        │── GET /orders ─────────►│
  │                        │   X-API-Version: 2     │
  │◄── v2 response ────────┤◄── response ───────────┤
```

```yaml
# Spring Cloud Gateway
spring:
  cloud:
    gateway:
      routes:
        - id: orders-v1
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=2
            - AddRequestHeader=X-API-Version, 1

        - id: orders-v2
          uri: lb://order-service
          predicates:
            - Path=/api/v2/orders/**
          filters:
            - StripPrefix=2
            - AddRequestHeader=X-API-Version, 2
```

```java
// Service içinde versiyona göre davran
@GetMapping("/orders/{id}")
ResponseEntity<?> getOrder(
        @PathVariable String id,
        @RequestHeader(value = "X-API-Version", defaultValue = "2") String version) {

    Order order = orderService.findById(id);
    return "1".equals(version)
        ? ResponseEntity.ok(OrderResponseV1.from(order))
        : ResponseEntity.ok(OrderResponseV2.from(order));
}
```

---

### Backward-Compatible Değişiklikler (Breaking vs Non-Breaking)

```
Non-breaking (versiyonlama gerekmez):
✓ Yeni field ekle (client bilmediği alanı ignore eder)
✓ Yeni endpoint ekle
✓ Optional parametre ekle (default değer ile)
✓ Response'a yeni alan ekle

Breaking (yeni versiyon gerekir):
✗ Alan adı değiştir: customerId → customer.id
✗ Alan tipini değiştir: string → int
✗ Zorunlu alan ekle (request'e)
✗ Alan sil (response'tan)
✗ HTTP status code değiştir (200 → 201)
✗ URL path değiştir
✗ Davranış değişikliği (FIFO → LIFO)
```

---

### Deprecation Lifecycle

```
Versiyon lifecycle yönetimi:

GA (General Availability):
  v2 çıktı → v1 hâlâ tam destek

Deprecation:
  v1 deprecated → header ile bildir
  Response header: Deprecation: true
  Response header: Sunset: Sat, 31 Dec 2025 23:59:59 GMT
  Response header: Link: <https://api.example.com/v2/orders>; rel="successor-version"

Sunset:
  Belirtilen tarihte v1 kapatılır → 410 Gone döner

Minimum: 6 ay deprecation süresi (partner API'lar için 1 yıl)
```

```java
// Deprecation header middleware (Spring Interceptor)
@Component
class DeprecationHeaderInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) {
        String path = request.getRequestURI();
        if (path.contains("/v1/")) {
            response.addHeader("Deprecation", "true");
            response.addHeader("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT");
            response.addHeader("Link",
                "<https://api.example.com/v2" + path.substring(3) + ">; rel=\"successor-version\"");
        }
        return true;
    }
}
```

---

### gRPC API Versioning

```protobuf
// Proto package ile versioning
package com.example.orders.v1;
service OrderService { ... }

package com.example.orders.v2;
service OrderService { ... }

// Backward compatible proto değişiklikler:
// ✓ Yeni field ekle (yeni field number)
// ✓ Yeni RPC ekle
// ✗ Field number değiştirme
// ✗ Field type değiştirme
// Sil → reserved keyword ile işaretle:
//   reserved 5, 6;
//   reserved "old_field";
```

---

## Ne zaman?

```
URL versioning seç:
✓ Public API, 3rd party kullanıcılar
✓ Tarayıcı tabanlı test edilecek
✓ Cache kritik

Header versioning seç:
✓ Internal API, developer-friendly ekip
✓ URL temiz kalmalı
✓ REST orthodoxy önemli

API Gateway versioning seç:
✓ Versiyon routing'i service'ten izole etmek
✓ Traffic shifting gerekiyor (v1:%80, v2:%20 canary)
✓ Birden fazla service versioning'i tek noktada yönet
```

---

## Trade-off?

| Strateji | Basitlik | Cache | REST Uyum | Test Kolaylığı |
|----------|---------|-------|-----------|----------------|
| URL Path | Yüksek | ✓ | Orta | ✓ |
| Header | Orta | ✗ | ✓ | Orta |
| Query Param | Yüksek | Orta | Düşük | ✓ |
| API Gateway | Yüksek (client için) | ✓ | ✓ | ✓ |

| Avantaj (genel) | Dezavantaj |
|-----------------|-----------|
| Eski client'lar çalışmaya devam | Çoklu versiyon bakımı (kod kopyası) |
| Bağımsız deployment | Hangi versionun ne zaman sunset? (yönetim) |
| İş sürekliliği | Test yükü (her version test edilmeli) |
| Partner entegrasyon stabilitesi | Ekip "eski versiyonu silmek istemiyor" |
