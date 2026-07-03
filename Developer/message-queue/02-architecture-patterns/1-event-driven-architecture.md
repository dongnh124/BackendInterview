# Event-Driven Architecture — Kiến Trúc Hướng Sự Kiện

> Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện) là mô hình thiết kế hệ thống trong đó các service giao tiếp thông qua **events (sự kiện)** thay vì gọi trực tiếp lẫn nhau — nền tảng cho microservices scale và decouple.

## Mục Lục

1. [EDA Là Gì?](#eda-là-gì)
2. [Event Types — Các Loại Sự Kiện](#event-types--các-loại-sự-kiện)
3. [EDA vs Request-Response](#eda-vs-request-response)
4. [Bounded Context & Service Boundaries](#bounded-context--service-boundaries)
5. [Event Notification vs Event-Carried State Transfer](#event-notification-vs-event-carried-state-transfer)
6. [Thành Phần EDA](#thành-phần-eda)
7. [Thiết Kế Event Schema](#thiết-kế-event-schema)
8. [Anti-Patterns & Pitfalls](#anti-patterns--pitfalls)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## EDA Là Gì?

**Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện)** mô tả hệ thống nơi **state change (thay đổi trạng thái)** trong một service được broadcast dưới dạng event, và các service khác **react (phản ứng)** khi cần.

```
┌─────────────┐     OrderCreated      ┌─────────────┐
│   Order     │ ────────────────────► │   Event     │
│   Service   │                       │   Broker    │
└─────────────┘                       └──────┬──────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
             ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
             │  Inventory  │          │  Payment    │          │ Notification│
             │  Service    │          │  Service    │          │  Service    │
             └─────────────┘          └─────────────┘          └─────────────┘
```

**Đặc điểm cốt lõi:**

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Asynchronous (Bất Đồng Bộ)** | Producer không chờ consumer xử lý xong |
| **Loose Coupling (Liên Kết Lỏng)** | Producer không biết consumer cụ thể |
| **Eventual Consistency (Nhất Quán Cuối Cùng)** | Data across services đồng bộ theo thời gian |
| **Scalability (Khả Năng Mở Rộng)** | Thêm consumer mới không sửa producer |
| **Resilience (Khả Năng Phục Hồi)** | Consumer offline — broker buffer message |

---

## Event Types — Các Loại Sự Kiện

### Domain Event (Sự Kiện Nghiệp Vụ)

Một **fact (sự thật)** đã xảy ra trong domain — không thể thay đổi, chỉ append.

```
OrderCreated { orderId, customerId, items[], totalAmount, occurredAt }
PaymentFailed { orderId, reason, occurredAt }
InventoryReserved { orderId, sku, quantity, occurredAt }
```

**Quy tắc đặt tên:** quá khứ — `OrderCreated`, không phải `CreateOrder`.

### Integration Event (Sự Kiện Tích Hợp)

Domain event được publish ra ngoài bounded context để service khác consume.

```
// Internal domain event
OrderAggregate.raise(OrderCreated)

// Integration event (có thể enrich thêm metadata)
{
  "eventType": "order.created.v1",
  "eventId": "evt-uuid",
  "correlationId": "corr-uuid",
  "payload": { ... }
}
```

### Command vs Event

| | Command (Lệnh) | Event (Sự Kiện) |
| --- | -------------- | --------------- |
| **Ý nghĩa** | Yêu cầu làm gì đó | Sự thật đã xảy ra |
| **Đích** | Một handler cụ thể | Zero hoặc nhiều subscriber |
| **Tên** | Imperative — `CreateOrder` | Past tense — `OrderCreated` |
| **Thất bại** | Có thể reject | Không "reject" — đã xảy ra rồi |

---

## EDA vs Request-Response

```
REQUEST-RESPONSE (Sync — Đồng Bộ):
Client ──HTTP──► Service A ──HTTP──► Service B ──HTTP──► Service C
         │              │                    │
         └── Chờ response chain ────────────┘
         Latency = tổng tất cả hops
         Service B down → Service A fail

EVENT-DRIVEN (Async — Bất Đồng Bộ):
Service A ──publish──► Broker ──► Service B
                         └──► Service C
Service A done ngay sau publish
Service B/C xử lý độc lập, retry riêng
```

| Tiêu Chí | Request-Response | Event-Driven |
| -------- | ---------------- | ------------ |
| **Coupling** | Tight — biết endpoint | Loose — biết topic/queue |
| **Latency** | Tổng chain | Thấp cho producer |
| **Consistency** | Strong (trong transaction) | Eventual |
| **Failure handling** | Cascade fail | Buffer + retry |
| **Debugging** | Dễ trace sync call | Cần correlation ID, tracing |
| **Phù hợp** | Query, validation cần response ngay | Side effects, notifications, analytics |

### Khi Nào Dùng EDA

| Use Case | Lý Do |
| -------- | ----- |
| Order placed → email + analytics + inventory | Nhiều side effect, không cần chờ tất cả |
| Peak traffic (flash sale) | Buffer qua queue, consumer scale |
| Cross-team integration | Publish contract, teams subscribe độc lập |
| Audit trail | Event log lưu lịch sử |

### Khi KHÔNG Dùng EDA

| Use Case | Lý Do |
| -------- | ----- |
| User login cần token ngay | Cần sync response |
| Validate inventory trước checkout | Cần strong consistency tại thời điểm đó |
| Simple CRUD ít integration | Overhead không xứng đáng |

---

## Bounded Context & Service Boundaries

**Bounded Context (Ngữ Cảnh Giới Hạn)** — ranh giới domain nơi một model có nghĩa nhất quán (từ Domain-Driven Design — DDD).

```
┌─────────────────────────┐     ┌─────────────────────────┐
│   ORDER CONTEXT         │     │   SHIPPING CONTEXT      │
│                         │     │                         │
│   Order (aggregate)     │     │   Shipment (aggregate)  │
│   OrderLine             │     │   DeliveryAddress       │
│   OrderStatus           │     │   TrackingNumber        │
│                         │     │                         │
│   Event: OrderCreated   │────►│   Listen → CreateShipment│
└─────────────────────────┘     └─────────────────────────┘
         │                                    │
    Own database                         Own database
    (không share table)                  (không share table)
```

**Nguyên tắc:**

1. **Database per service** — không JOIN cross-service
2. **Event là contract** giữa contexts — versioned schema
3. **Không leak internal model** — publish integration event, không expose entity nội bộ
4. Mỗi context **own data** và publish fact khi state thay đổi

---

## Event Notification vs Event-Carried State Transfer

Hai pattern phổ biến khi thiết kế integration event:

### Event Notification (Thông Báo Sự Kiện)

Event chỉ chứa **ID và metadata tối thiểu** — consumer phải gọi API để lấy data.

```json
{
  "eventType": "order.created",
  "orderId": "ORD-12345",
  "occurredAt": "2026-07-03T10:00:00Z"
}
```

| Ưu | Nhược |
| --- | ----- |
| Payload nhỏ | Consumer phải gọi thêm API (coupling) |
| Luôn fresh data | Thêm latency, dependency availability |
| Schema ổn định | Không hoạt động khi source service down |

### Event-Carried State Transfer (Chuyển Trạng Thái Qua Sự Kiện)

Event mang **đủ data** consumer cần — không cần gọi lại.

```json
{
  "eventType": "order.created",
  "orderId": "ORD-12345",
  "customerId": "CUST-99",
  "items": [{ "sku": "ABC", "qty": 2, "price": 50000 }],
  "totalAmount": 100000,
  "shippingAddress": { "city": "HCM", "..." }
}
```

| Ưu | Nhược |
| --- | ----- |
| Consumer autonomous | Payload lớn, schema phức tạp |
| Không phụ thuộc source API | Data có thể stale nếu event cũ |
| Phù hợp analytics, cache | Duplicate data across services |

**Khuyến nghị:** Event notification cho core flows cần fresh data; event-carried state transfer cho read models, notifications, analytics.

---

## Thành Phần EDA

```
┌──────────────────────────────────────────────────────────────┐
│                        EDA STACK                              │
├──────────────────────────────────────────────────────────────┤
│  Event Producer    → Service publish event sau state change  │
│  Event Broker      → Kafka, RabbitMQ, SQS — routing, buffer  │
│  Event Consumer    → Handler idempotent, at-least-once safe    │
│  Event Schema      → Avro/Protobuf + Schema Registry           │
│  Correlation ID    → Trace request xuyên services              │
│  Dead Letter Queue → Message fail sau N retry                  │
│  Outbox Relay      → Atomic DB write + publish                 │
└──────────────────────────────────────────────────────────────┘
```

### Correlation ID & Causation ID

```json
{
  "eventId": "evt-001",
  "correlationId": "req-abc",
  "causationId": "evt-previous",
  "eventType": "payment.processed"
}
```

- **Correlation ID:** trace toàn bộ flow từ user request
- **Causation ID:** event nào trigger event hiện tại
- Essential cho **distributed tracing (truy vết phân tán)** và debug

---

## Thiết Kế Event Schema

### Versioning (Phiên Bản Hóa)

```
order.created.v1  →  order.created.v2 (thêm field optional)
                   → backward compatible consumers vẫn hoạt động
```

**Quy tắc:**

1. Chỉ **thêm field optional** — không xóa/rename field
2. Consumer **ignore unknown fields**
3. Dùng **Schema Registry** enforce compatibility
4. Event type name có version: `order.created.v1`

### Event Envelope (Bọc Sự Kiện)

```json
{
  "metadata": {
    "eventId": "uuid",
    "eventType": "order.created.v1",
    "timestamp": "2026-07-03T10:00:00Z",
    "source": "order-service",
    "correlationId": "uuid",
    "idempotencyKey": "order-12345-created"
  },
  "payload": {
    "orderId": "ORD-12345",
    "totalAmount": 100000
  }
}
```

---

## Anti-Patterns & Pitfalls

### 1. Distributed Monolith (Monolith Phân Tán)

```
❌ Mọi service listen mọi event, gọi nhau qua event chain phức tạp
✅ Rõ bounded context, event chỉ cross boundary khi cần
```

### 2. Event as Command Disguise

```
❌ Event "ProcessPayment" — yêu cầu action
✅ Event "OrderCreated" — fact, Payment service tự quyết có process không
```

### 3. Missing Idempotency

```
❌ Consumer assume message chỉ đến 1 lần
✅ Idempotency key + dedup — xem 5-idempotency-dedup.md
```

### 4. God Event

```
❌ Một event chứa toàn bộ system state
✅ Small, focused domain events — Single Responsibility
```

### 5. Sync-over-Async

```
❌ Publish event rồi poll/wait cho consumer xử lý xong
✅ Nếu cần response → dùng sync API hoặc request-reply pattern
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: EDA giải quyết vấn đề gì trong microservices?

**Đáp án mẫu:** **Decoupling** — services không phụ thuộc availability trực tiếp. **Scalability** — scale consumer độc lập. **Resilience** — broker buffer khi downstream chậm. **Extensibility** — thêm subscriber mới không sửa producer. Trade-off: eventual consistency, complexity trong debugging và testing.

### Câu 2: Domain Event khác Integration Event?

**Đáp án mẫu:** **Domain event** là fact trong bounded context nội bộ — part of domain model. **Integration event** là representation publish ra ngoài cho service khác — có thể map, enrich metadata (correlationId, schema version), và ẩn internal details.

### Câu 3: Làm sao debug event-driven flow?

**Đáp án mẫu:** **Correlation ID** xuyên suốt chain. **Structured logging** với eventId, eventType. **Distributed tracing** (OpenTelemetry). **Event audit log** — replay trong staging. Dashboard: consumer lag, error rate per handler.

### Câu 4: Eventual consistency — user thấy data cũ, xử lý thế nào?

**Đáp án mẫu:** (1) **UI optimistic update** + refresh. (2) **Read-your-writes** — route read về source service ngay sau write. (3) **Status polling/WebSocket** cho long-running flow. (4) Thiết kế UX chấp nhận delay ("Đơn hàng đang xử lý"). (5) Saga với clear state machine cho user.

### Câu 5: Khi nào KHÔNG nên dùng event-driven?

**Đáp án mẫu:** Cần **strong consistency** ngay lập tức (transfer tiền đồng bộ). Flow đơn giản 2 service — HTTP đủ. Team thiếu kinh nghiệm ops messaging. Không có monitoring/tracing infrastructure. Over-engineering cho CRUD đơn giản.

---

**Xem tiếp:** [5-idempotency-dedup.md](./5-idempotency-dedup.md) — nền tảng bắt buộc trước khi implement EDA production.
