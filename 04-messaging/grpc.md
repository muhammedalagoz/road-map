# 04e — gRPC

## Ne?

Google'ın geliştirdiği **yüksek performanslı RPC (Remote Procedure Call) framework**'ü. HTTP/2 üzerinde çalışır, Protocol Buffers (protobuf) ile binary serializasyon kullanır. Interface tanımı `.proto` dosyasında yapılır; client ve server kodu otomatik generate edilir.

---

## Neden?

**Çözdüğü problem:** REST/JSON ile servisler arası iletişimde JSON parse overhead, HTTP/1.1 head-of-line blocking ve loose contract (client ne bekleyeceğini tam bilmez).

```
REST/JSON:
  - JSON parse: CPU yoğun, büyük payload
  - HTTP/1.1: bir response tamamlanmadan diğeri gönderilmez
  - Contract: OpenAPI spec opsiyonel, drift riski
  - Streaming: WebSocket ile ekstra karmaşıklık

gRPC:
  - Protobuf binary: 3-10x daha küçük, çok daha hızlı parse
  - HTTP/2: multiplexing, header compression, binary framing
  - Contract: .proto = tek kaynak (server + client kodu generate edilir)
  - Streaming: 4 tip (unary, server, client, bidirectional) yerleşik
```

---

## Nasıl?

### Protocol Buffers (Protobuf)

```protobuf
// orders.proto
syntax = "proto3";
package com.example.orders;
option java_package = "com.example.orders.grpc";

// Service tanımı
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);                   // Unary
  rpc StreamOrders (StreamOrdersRequest) returns (stream Order);   // Server streaming
  rpc CreateOrders (stream CreateOrderRequest) returns (OrderSummary); // Client streaming
  rpc OrderChat (stream ChatMessage) returns (stream ChatMessage); // Bidirectional
}

// Message tanımları
message Order {
  string id = 1;                    // field number (1-15: 1 byte, 16-2047: 2 byte)
  string customer_id = 2;
  double total_amount = 3;
  OrderStatus status = 4;
  repeated OrderItem items = 5;     // List
  google.protobuf.Timestamp created_at = 6;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double unit_price = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // proto3: 0 her zaman default
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}

message GetOrderRequest { string order_id = 1; }
message StreamOrdersRequest { string customer_id = 1; }
message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}
message OrderSummary { int32 created_count = 1; double total_value = 2; }
```

**Protobuf binary encoding:**
```
JSON:   {"id":"ord-123","status":"CONFIRMED","total":99.99}
bytes:  51 bytes (UTF-8)

Protobuf: <field1><len><ord-123><field4><2><field3><99.99>
bytes:  ~15 bytes (binary, varsayılan değerler yazılmaz)

Neden küçük?
- Field name yazılmaz, sadece number (1, 2, 3...)
- Varsayılan değerler (0, "", false) wire'a yazılmaz
- Varint encoding: küçük sayılar az byte
- Binary: string parse overhead yok
```

---

### HTTP/2 Multiplexing

```
HTTP/1.1:
  Connection 1: [REQ1] ──── [RESP1] ──── [REQ2] ──── [RESP2]
  Connection 2: [REQ3] ──── [RESP3]
  Sorun: Head-of-line blocking (RESP1 gelmeden REQ2 gönderilemez aynı connection'da)

HTTP/2:
  Connection 1:
    Stream 1: [REQ1] →→→ ←←← [RESP1]  (paralel)
    Stream 2: [REQ2] →→→ ←←← [RESP2]  (aynı connection)
    Stream 3: [REQ3] →→→ ←←← [RESP3]
  Avantaj: 1 TCP connection, N paralel istek, no head-of-line blocking
```

---

### 4 Streaming Modu

```
1. Unary (klasik request-response):
   Client → [single request] → Server → [single response] → Client
   Kullanım: getOrder, createPayment

2. Server Streaming:
   Client → [single request] → Server → [stream of responses] → Client
   Kullanım: sipariş güncellemelerini canlı izle, büyük veri çek

3. Client Streaming:
   Client → [stream of requests] → Server → [single response] → Client
   Kullanım: dosya yükleme, batch mesaj gönderme

4. Bidirectional Streaming:
   Client ⇄ [stream] ⇄ Server (her iki yönde bağımsız akış)
   Kullanım: chat, gerçek zamanlı oyun, canlı dashboard
```

---

### Spring Boot gRPC Implementasyonu

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
```

```java
// Server (OrderService implementasyonu)
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    @Autowired OrderRepository orderRepo;

    // 1. Unary RPC
    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<Order> responseObserver) {
        orderRepo.findById(request.getOrderId())
            .ifPresentOrElse(
                order -> {
                    responseObserver.onNext(toProto(order));
                    responseObserver.onCompleted();
                },
                () -> responseObserver.onError(
                    Status.NOT_FOUND
                        .withDescription("Order not found: " + request.getOrderId())
                        .asRuntimeException()
                )
            );
    }

    // 2. Server Streaming
    @Override
    public void streamOrders(StreamOrdersRequest request,
                             StreamObserver<Order> responseObserver) {
        orderRepo.findByCustomerId(request.getCustomerId())
            .forEach(order -> responseObserver.onNext(toProto(order)));
        responseObserver.onCompleted();
    }

    // 3. Client Streaming
    @Override
    public StreamObserver<CreateOrderRequest> createOrders(
            StreamObserver<OrderSummary> responseObserver) {
        List<OrderEntity> created = new ArrayList<>();

        return new StreamObserver<>() {
            @Override public void onNext(CreateOrderRequest request) {
                created.add(orderService.create(request));
            }
            @Override public void onError(Throwable t) {
                log.error("Client stream error", t);
            }
            @Override public void onCompleted() {
                double total = created.stream()
                    .mapToDouble(OrderEntity::getTotal).sum();
                responseObserver.onNext(OrderSummary.newBuilder()
                    .setCreatedCount(created.size())
                    .setTotalValue(total)
                    .build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

```java
// Client
@GrpcClient("order-service")
OrderServiceGrpc.OrderServiceBlockingStub orderStub; // sync

@GrpcClient("order-service")
OrderServiceGrpc.OrderServiceStub orderAsyncStub;   // async

// Unary call
Order order = orderStub.getOrder(
    GetOrderRequest.newBuilder().setOrderId("ord-123").build()
);

// Server streaming
orderStub.streamOrders(
    StreamOrdersRequest.newBuilder().setCustomerId("cust-456").build()
).forEachRemaining(order -> processOrder(order));

// Async unary
orderAsyncStub.getOrder(request, new StreamObserver<Order>() {
    @Override public void onNext(Order order) { process(order); }
    @Override public void onError(Throwable t) { handleError(t); }
    @Override public void onCompleted() { log.info("Done"); }
});
```

**Konfigürasyon:**
```yaml
grpc:
  server:
    port: 9090
    security:
      certificate-chain: classpath:server.crt
      private-key: classpath:server.key
  client:
    order-service:
      address: discovery:///order-service  # service discovery
      negotiation-type: tls
      security:
        trust-cert-collection: classpath:ca.crt
```

---

### Interceptor (gRPC'de AOP gibi)

```java
// Server interceptor — auth, logging, tracing
@GrpcGlobalServerInterceptor
class AuthServerInterceptor implements ServerInterceptor {

    static final Context.Key<String> USER_ID_KEY = Context.key("userId");

    @Override
    public <Q, A> ServerCall.Listener<Q> interceptCall(
            ServerCall<Q, A> call,
            Metadata headers,
            ServerCallHandler<Q, A> next) {

        String token = headers.get(
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER));

        if (token == null) {
            call.close(Status.UNAUTHENTICATED.withDescription("Token required"), new Metadata());
            return new ServerCall.Listener<>() {};
        }

        String userId = jwtValidator.validate(token);
        Context ctx = Context.current().withValue(USER_ID_KEY, userId);
        return Contexts.interceptCall(ctx, call, headers, next);
    }
}

// Service içinde kullan
@GrpcService
class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {
    void getOrder(GetOrderRequest req, StreamObserver<Order> obs) {
        String userId = AuthServerInterceptor.USER_ID_KEY.get(); // context'ten al
    }
}
```

---

### Error Handling

```java
// gRPC status codes (HTTP status'a benzer ama farklı)
Status.NOT_FOUND         → 404 benzeri
Status.INVALID_ARGUMENT  → 400 benzeri
Status.UNAUTHENTICATED   → 401 benzeri
Status.PERMISSION_DENIED → 403 benzeri
Status.INTERNAL          → 500 benzeri
Status.UNAVAILABLE       → 503 benzeri (retry edilebilir)
Status.RESOURCE_EXHAUSTED→ 429 benzeri (rate limit)

// Detaylı hata
StatusRuntimeException ex = Status.NOT_FOUND
    .withDescription("Order not found: " + orderId)
    .augmentDescription("Try with a valid order ID")
    .asRuntimeException();

// Client'ta handle et
try {
    Order order = orderStub.getOrder(request);
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
        // handle not found
    } else if (e.getStatus().getCode() == Status.Code.UNAVAILABLE) {
        // retry
    }
}
```

---

### Schema Evolution (Backward Compatibility)

```protobuf
// v1
message Order {
  string id = 1;
  string customer_id = 2;
  double total = 3;
}

// v2 (backward compatible — yeni field ekle)
message Order {
  string id = 1;
  string customer_id = 2;
  double total = 3;
  string shipping_address = 4;  // YENİ: eski client görmez, varsayılan ""
  repeated string tags = 5;     // YENİ: eski client görmez, varsayılan []
}

// YAPMA (backward incompatible):
// - Field number değiştirme (1 → 10 gibi)
// - Field type değiştirme (string → int32)
// - Required field ekleme (proto3'te required yok ama dikkat)
// - Field silme VE aynı number'ı farklı type için kullanma

// GÜVENLİ:
// - Yeni field ekleme (yeni number ile)
// - Field silme (number'ı reserved ile işaretle)
reserved 6, 7;
reserved "old_field_name";
```

---

## Ne zaman?

**gRPC kullan:**
```
✓ Servisler arası internal iletişim (microservice-to-microservice)
✓ Yüksek throughput, düşük latency kritikse
✓ Streaming gerekiyorsa (server push, bidirectional)
✓ Strong typing, contract-first API gerekiyorsa
✓ Polyglot (Java, Go, Python, C++, Rust — her dilde client generate edilir)
✓ Mobile backend (küçük payload → az bant genişliği)
```

**gRPC kullanma:**
```
✗ Public API (browser doğrudan HTTP/2 gRPC desteklemez → grpc-web gerekir)
✗ Basit CRUD, ekip REST'e aşina → REST daha kolay
✗ Human-readable mesajlar gerekiyorsa → JSON/REST
✗ Firewall/proxy HTTP/2'yi kesiyor mu? → kontrol et
```

---

## Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| 5-10x daha hızlı (binary + HTTP/2) | Browser doğrudan kullanamaz (grpc-web gerekir) |
| Strong typing → compile-time hata | .proto dosyası yönetimi (versioning, sharing) |
| Streaming yerleşik (4 mod) | Debug zor (binary → curl ile göremezsin) |
| Code generation → boilerplate yok | Ekip öğrenme eğrisi |
| Polyglot (her dilde client) | HTTP/2 desteklemeyen proxy sorun çıkarır |
| HTTP/2 multiplexing | REST kadar yaygın değil (araç desteği az) |
| Bidirectional streaming | Schema evolution dikkat ister |
