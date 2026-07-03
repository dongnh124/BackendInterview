# Bài Toán Thiết Kế Hệ Thống — System Design Scenarios

> Hướng dẫn thiết kế hệ thống messaging thực tế cho phỏng vấn Backend. Mỗi scenario bao gồm: yêu cầu, kiến trúc đề xuất, lựa chọn broker và trade-offs cần thảo luận.

## Mục Lục

1. [Cách Tiếp Cận System Design Interview](#cách-tiếp-cận-system-design-interview)
2. [Scenario 1: Order Processing Pipeline](#scenario-1-order-processing-pipeline)
3. [Scenario 2: Notification System](#scenario-2-notification-system)
4. [Scenario 3: CDC Pipeline](#scenario-3-cdc-pipeline)
5. [Scenario 4: Real-time Analytics Pipeline](#scenario-4-real-time-analytics-pipeline)
6. [Checklist Thiết Kế Messaging](#checklist-thiết-kế-messaging)

---

## Cách Tiếp Cận System Design Interview

### Framework RESHADED cho Messaging

```
R — Requirements: Functional + Non-functional (ordering, durability, latency)
E — Estimate: Events/s, retention days, peak vs average
S — Storage: Topic/queue design, event schema, partition strategy
H — High-level: Services, broker, data stores diagram
A — APIs: Event contracts, partition keys, idempotency keys
D — Detailed: Outbox, DLQ, retry, saga steps
E — Edge cases: Broker down, duplicate, poison message, hot partition
D — Delivery: At-least-once + idempotency strategy
```

### Thứ Tự Trình Bày (45 Phút)

```
5 phút:  Clarify requirements — hỏi về scale, ordering, consistency
10 phút: High-level architecture — vẽ producer → broker → consumers
15 phút: Deep dive: delivery semantics, outbox, partition design
10 phút: Failure scenarios + monitoring
5 phút:  Trade-offs và alternatives (Kafka vs RabbitMQ)
```

---

## Scenario 1: Order Processing Pipeline

**Tương tự:** E-commerce order fulfillment, food delivery order flow

### Yêu Cầu

**Functional (Chức Năng):**
- User đặt hàng → validate inventory → charge payment → reserve stock → notify warehouse
- Hủy đơn → refund + release stock
- Order status updates cho user (real-time hoặc near real-time)

**Non-functional (Phi Chức Năng):**
- 10,000 orders/giờ peak, 1,000 orders/giờ average
- Không mất order events (financial critical)
- Events cùng `orderId` phải ordered
- Availability 99.9%

### Ước Tính

```
Peak: 10,000 orders/hour ≈ 3 orders/s
Events per order: ~5 (Created, Paid, Reserved, Shipped, Delivered)
Event throughput: ~15 events/s peak

Storage (Kafka, 7 days retention):
  15 events/s × 86400 × 7 × 2KB ≈ 18 GB
```

### Kiến Trúc Đề Xuất

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────────────┐
│ Order API   │────►│ PostgreSQL   │     │ Kafka Topic: orders     │
│ (REST)      │     │ + Outbox     │────►│ partition key: orderId  │
└─────────────┘     └──────────────┘     └───────────┬─────────────┘
                                                     │
                    ┌────────────────────────────────┼────────────────┐
                    ▼                ▼               ▼                ▼
            ┌─────────────┐  ┌───────────┐  ┌─────────────┐  ┌──────────┐
            │ Payment     │  │ Inventory │  │ Warehouse   │  │ Notify   │
            │ Service     │  │ Service   │  │ Service     │  │ Service  │
            └─────────────┘  └───────────┘  └─────────────┘  └──────────┘
```

### Thiết Kế Chi Tiết

**Outbox Pattern — tại sao:**

```sql
BEGIN;
  INSERT INTO orders (id, user_id, total, status) VALUES (...);
  INSERT INTO outbox (aggregate_id, event_type, payload)
    VALUES (order_id, 'OrderCreated', '{"orderId":"...","items":[...]}');
COMMIT;
-- Outbox Relay (Debezium hoặc polling) → Kafka
```

**Partition Strategy:**
- Key = `orderId` → tất cả events cùng order vào cùng partition → ordering guaranteed

**Saga — distributed transaction:**

```
Choreography (đề xuất cho flow đơn giản):
  OrderCreated → PaymentService charge
  PaymentCompleted → InventoryService reserve
  InventoryReserved → WarehouseService pick
  PaymentFailed → OrderCancelled + compensation
```

**Idempotency:**
- Mỗi consumer lưu `processed_events(event_id)` — skip duplicate
- Payment: dùng `idempotency_key = orderId + "charge"`

### Edge Cases Cần Thảo Luận

| Scenario | Giải Pháp |
| -------- | --------- |
| Payment timeout | Saga timeout + `PaymentFailed` event |
| Duplicate `OrderCreated` | Idempotency key trên order API |
| Inventory insufficient | Compensating `OrderCancelled` + refund |
| Consumer lag spike | Scale consumers, alert lag > 5 phút |

### Trade-offs

| Quyết Định | Lựa Chọn | Alternative |
| ---------- | -------- | ----------- |
| Broker | Kafka (replay, ordering) | RabbitMQ nếu team nhỏ |
| Saga style | Choreography | Orchestration nếu > 5 steps |
| Consistency | Eventual + Outbox | 2PC (không khuyến nghị) |

📖 Xem thêm: [02-architecture-patterns/3-saga-pattern.md](../02-architecture-patterns/3-saga-pattern.md), [4-outbox-inbox-pattern.md](../02-architecture-patterns/4-outbox-inbox-pattern.md)

---

## Scenario 2: Notification System

**Tương tự:** Push notification, email/SMS fan-out, in-app alerts

### Yêu Cầu

**Functional:**
- Gửi notification qua nhiều kênh: email, SMS, push, in-app
- User preferences (opt-out per channel)
- Template-based messages
- Delivery status tracking

**Non-functional:**
- 1M notifications/ngày, burst 50K/giờ (marketing campaign)
- Email/SMS latency < 30s acceptable
- At-least-once delivery (duplicate email chấp nhận nếu idempotent)
- Không cần strict ordering giữa users

### Kiến Trúc Đề Xuất

```
┌──────────────┐     ┌─────────────┐     ┌──────────────────────┐
│ Event Source │────►│ SNS / Kafka │────►│ Fan-out              │
│ (orders,     │     │ Topic:      │     │                      │
│  user events)│     │ notifications│    │  ├─ Email Queue      │
└──────────────┘     └─────────────┘     │  ├─ SMS Queue        │
                                         │  ├─ Push Queue       │
                                         │  └─ In-app Queue     │
                                         └──────────┬───────────┘
                                                    ▼
                                         ┌──────────────────────┐
                                         │ Worker pools per     │
                                         │ channel + rate limit │
                                         └──────────────────────┘
```

### Thiết Kế Chi Tiết

**Fan-out Pattern:**

```
Option A — Kafka:
  Topic "notification-events" → Consumer Group per channel
  Email workers, SMS workers scale độc lập

Option B — RabbitMQ:
  Fanout Exchange → [email-queue, sms-queue, push-queue]
  Mỗi queue có workers riêng

Option C — AWS:
  SNS → SQS (per channel) → Lambda/ECS workers
```

**Rate Limiting (Giới Hạn Tốc Độ):**
- SMS provider: 100 msg/s limit → token bucket per provider
- Email: batch send, queue depth monitoring

**Idempotency:**
- `notificationId` = hash(userId + eventType + eventId)
- DB check trước khi gửi — skip nếu đã sent

**DLQ Design:**
- Mỗi channel queue có DLQ riêng
- Alert khi DLQ depth > 100
- Retry 3 lần với exponential backoff

### Scale Estimation

```
50K notifications/hour burst ≈ 14/s
Email worker: ~10/s per instance → 2 instances
SMS: rate-limited → queue absorbs burst
Push (FCM): batch API → 500/request
```

### Trade-offs Thảo Luận

| Topic | Recommendation |
| ----- | -------------- |
| Ordering | Không cần global — per-user optional |
| Broker | RabbitMQ/SQS cho task queue; Kafka nếu cần replay analytics |
| Sync vs async | Luôn async — API chỉ enqueue |
| Priority | Separate queues: transactional vs marketing |

---

## Scenario 3: CDC Pipeline

**Tương tự:** Debezium sync, read replica → search index, cache invalidation

### Yêu Cầu

**Functional:**
- Mọi thay đổi trong PostgreSQL → propagate sang Elasticsearch (search) và Redis (cache)
- Near real-time (< 5s lag)
- Không miss changes

**Non-functional:**
- 5,000 writes/s peak vào DB
- Schema changes phải handle gracefully
- Exactly-once vào search index (idempotent upsert)

### Kiến Trúc Đề Xuất

```
┌──────────────┐     ┌─────────────┐     ┌─────────────────────┐
│ PostgreSQL   │────►│ Debezium    │────►│ Kafka               │
│ (WAL)        │     │ Connector   │     │ Topic: db.public.*  │
└──────────────┘     └─────────────┘     └──────────┬──────────┘
                                                    │
                              ┌─────────────────────┼─────────────────┐
                              ▼                     ▼                 ▼
                      ┌──────────────┐      ┌──────────────┐  ┌────────────┐
                      │ ES Consumer  │      │ Redis        │  │ Data       │
                      │ (upsert doc) │      │ Invalidator  │  │ Warehouse  │
                      └──────────────┘      └──────────────┘  └────────────┘
```

### Thiết Kế Chi Tiết

**Tại sao CDC thay vì Outbox:**
- Cần capture **mọi** DB change (kể cả admin updates, migrations)
- Outbox chỉ capture events application explicitly publish

**Debezium Event Format:**
```json
{
  "op": "c",  // create, update, delete
  "after": { "id": 123, "name": "Product A", "price": 99 },
  "source": { "table": "products", "lsn": 12345 }
}
```

**Idempotent Consumer cho Elasticsearch:**
- Upsert by document ID = primary key
- DELETE event → delete doc
- Out-of-order: last-write-wins với timestamp

**Schema Evolution:**
- Schema Registry + Avro/Protobuf
- Backward compatible changes only
- Consumer handle `null` fields gracefully

### Edge Cases

| Scenario | Handling |
| -------- | -------- |
| Connector lag | Monitor `MilliSecondsBehindSource`, scale Kafka Connect |
| Initial snapshot | Snapshot mode + then streaming |
| Tombstone events | Kafka compacted topic cho DELETE |
| DB failover | Connector reconnect, resume from LSN/offset |

### Trade-offs

| | CDC (Debezium) | Outbox |
| - | -------------- | ------ |
| Coverage | All DB changes | App-published only |
| Coupling | Low (reads WAL) | Medium (app writes outbox) |
| Complexity | Kafka Connect ops | Simpler relay |

📖 Xem thêm: [11-advanced/1-change-data-capture.md](../11-advanced/1-change-data-capture.md), [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md)

---

## Scenario 4: Real-time Analytics Pipeline

**Bonus scenario** — thường gặp khi kết hợp messaging + stream processing

### Yêu Cầu

- Clickstream events → real-time dashboard (DAU, conversion funnel)
- 100K events/s peak
- Aggregation windows: 1 phút, 1 giờ
- Retention 30 ngày raw, 1 năm aggregated

### Kiến Trúc Ngắn Gọn

```
Producers → Kafka (100+ partitions, key=userId)
         → Kafka Streams / Flink (windowed aggregation)
         → ClickHouse / Druid (OLAP store)
         → Grafana dashboard
```

**Partition count:** ≥ peak consumers needed; 100 partitions cho 100K/s với ~1K msg/s/partition.

**Key design:** `userId` cho per-user ordering; salting nếu hot users.

---

## Checklist Thiết Kế Messaging

Trước khi kết thúc phỏng vấn system design, đảm bảo đã cover:

- [ ] **Delivery semantics** — at-least-once + idempotency
- [ ] **Partition/queue design** — key strategy, count
- [ ] **Schema** — event contract, versioning
- [ ] **Failure handling** — DLQ, retry, saga compensation
- [ ] **Monitoring** — lag, throughput, DLQ depth alerts
- [ ] **Security** — ACL, encryption in-transit
- [ ] **Scale path** — horizontal consumer scaling
- [ ] **Trade-off** — tại sao chọn broker này

---

**Cập Nhật Lần Cuối:** 2026-07-03
