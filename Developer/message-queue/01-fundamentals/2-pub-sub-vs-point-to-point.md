# Pub/Sub vs Point-to-Point — Hai Mô Hình Messaging Cốt Lõi

> Publish/Subscribe (Pub/Sub — Xuất Bản/Đăng Ký) và Point-to-Point (P2P — Điểm-Điểm) là hai mô hình giao tiếp messaging cơ bản. Hiểu rõ sự khác biệt để thiết kế đúng topology (cấu trúc mạng) cho hệ thống.

## Mục Lục

1. [Point-to-Point (P2P)](#point-to-point-p2p)
2. [Publish/Subscribe (Pub/Sub)](#publishsubscribe-pubsub)
3. [So Sánh Trực Tiếp](#so-sánh-trực-tiếp)
4. [Competing Consumers vs Fan-out](#competing-consumers-vs-fan-out)
5. [Triển Khai Trên Các Broker](#triển-khai-trên-các-broker)
6. [Mô Hình Kết Hợp](#mô-hình-kết-hợp)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Point-to-Point (P2P)

**Point-to-Point (P2P — Điểm-Điểm)** — mỗi message được gửi từ **một producer** đến **một consumer** thông qua queue (hàng đợi).

```
Producer ──► [ Queue: tasks ] ──► Consumer (chỉ 1 nhận message)
```

### Đặc Điểm

| Đặc Điểm | Chi Tiết |
| -------- | -------- |
| **1 message → 1 consumer** | Message bị "claim" bởi consumer đầu tiên available |
| **FIFO** | Thường xử lý theo thứ tự vào queue (trừ priority queue) |
| **Message removal** | Sau khi ack, message bị xóa khỏi queue |
| **Use case** | Task queue, work distribution, job processing |

### Competing Consumers (Consumer Cạnh Tranh)

Khi scale horizontal (theo chiều ngang), nhiều consumer instance cùng subscribe một queue:

```
                    ┌──► Worker 1 (nhận msg 1, 4, 7...)
Producer ──► Queue ─┼──► Worker 2 (nhận msg 2, 5, 8...)
                    └──► Worker 3 (nhận msg 3, 6, 9...)
```

**Lợi ích:** Tăng throughput (thông lượng) mà không duplicate work  
**Lưu ý:** Không đảm bảo message nào đến worker nào — chỉ đảm bảo mỗi message xử lý **đúng một lần** (với at-least-once)

### Ví Dụ RabbitMQ — Work Queue

```javascript
// Producer
channel.sendToQueue('task_queue', Buffer.from('resize-image-123'));

// 3 worker instances — competing consumers
channel.consume('task_queue', handler, { noAck: false });
```

---

## Publish/Subscribe (Pub/Sub)

**Publish/Subscribe (Pub/Sub — Xuất Bản/Đăng Ký)** — producer **publish** message lên **topic/channel**, tất cả **subscriber** đăng ký nhận bản copy.

```
                    ┌──► Subscriber A (Email Service)
Publisher ──► Topic ─┼──► Subscriber B (Analytics)
                    └──► Subscriber C (Audit Log)
```

### Đặc Điểm

| Đặc Điểm | Chi Tiết |
| -------- | -------- |
| **1 message → N subscribers** | Mỗi subscriber nhận bản copy riêng |
| **Decoupling** | Publisher không biết có bao nhiêu subscriber |
| **Dynamic subscription** | Subscriber có thể join/leave runtime |
| **Use case** | Event notification, fan-out, microservices sync |

### Consumer Groups trong Kafka (Nhóm Consumer)

Kafka kết hợp cả hai mô hình:

```
Topic: orders (3 partitions)
  Consumer Group "payment"     → 3 consumers, mỗi partition 1 consumer
  Consumer Group "analytics" → đọc độc lập, cùng data
```

- **Trong một consumer group:** competing consumers (P2P semantics per partition)
- **Giữa các consumer group:** Pub/Sub semantics (mỗi group nhận full stream)

---

## So Sánh Trực Tiếp

| Tiêu Chí | Point-to-Point | Pub/Sub |
| -------- | -------------- | ------- |
| **Số receiver** | 1 consumer/message | N subscribers/message |
| **Mục đích** | Phân phối công việc | Phát sóng sự kiện |
| **Message sau consume** | Xóa (thường) | Giữ (event log) hoặc xóa per-subscriber |
| **Scale pattern** | Thêm worker vào queue | Thêm subscriber hoặc consumer group |
| **Ví dụ** | Resize ảnh, gửi email batch | OrderCreated → 5 services |
| **Broker** | RabbitMQ queue, SQS | Kafka topic, SNS, RabbitMQ fanout |

```
┌─────────────────────────────────────────────────────────────┐
│  CÂU HỎI THIẾT KẾ: "Ai cần nhận message này?"               │
├─────────────────────────────────────────────────────────────┤
│  Chỉ 1 worker xử lý        →  Point-to-Point / Work Queue   │
│  Nhiều service cùng biết   →  Pub/Sub / Event Broadcast     │
└─────────────────────────────────────────────────────────────┘
```

---

## Competing Consumers vs Fan-out

Hai pattern thường bị nhầm lẫn:

### Competing Consumers (P2P Semantics)

```
Goal: Xử lý NHANH HƠN cùng loại task
       10 workers × 100 msg/phút = 1000 msg/phút

Queue ──► [W1] [W2] [W3] ... [W10]
          Mỗi message → đúng 1 worker
```

### Fan-out (Pub/Sub Semantics)

```
Goal: NHIỀU SERVICE phản ứng cùng event
       1 OrderCreated → Payment + Inventory + Email + Analytics

Event ──► [Payment] [Inventory] [Email] [Analytics]
          Mỗi service nhận bản copy
```

### Anti-pattern: Dùng Pub/Sub Cho Work Queue

```
❌ SAI: 5 email workers cùng subscribe topic "send-email"
         → 1 email gửi 5 lần!

✅ ĐÚNG: 5 workers compete trên queue "send-email"
         → 1 email, 1 worker xử lý
```

---

## Triển Khai Trên Các Broker

### RabbitMQ

| Pattern | Cơ Chế |
| ------- | ------ |
| **P2P** | Default queue — `sendToQueue` + `consume` |
| **Pub/Sub** | Fanout exchange → nhiều queue bound |
| **Routing** | Direct/Topic exchange — selective pub/sub |
| **RPC** | Reply queue + correlationId |

```
Fanout Exchange "events"
    ├── binding → queue "email"    → Email Consumer
    ├── binding → queue "audit"    → Audit Consumer
    └── binding → queue "metrics"  → Metrics Consumer
```

### Apache Kafka

| Pattern | Cơ Chế |
| ------- | ------ |
| **P2P** | Consumer group — partition assigned to 1 consumer |
| **Pub/Sub** | Nhiều consumer group cùng subscribe topic |
| **Key routing** | Cùng key → cùng partition → ordering |

### Amazon SNS + SQS

```
SNS Topic "order-events"  (Pub/Sub fan-out)
    ├── SQS Queue "payment-queue"     → Payment Lambda
    ├── SQS Queue "inventory-queue"  → Inventory Lambda
    └── SQS Queue "email-queue"      → Email Lambda

Mỗi SQS queue = P2P competing consumers bên trong
```

---

## Mô Hình Kết Hợp

Hệ thống thực tế thường kết hợp cả hai:

### Pattern: Event Bus + Work Queue

```
Order Service
    │
    ▼ publish OrderCreated (Pub/Sub)
[Kafka Topic: orders]
    │
    ├── Consumer Group "notifications"
    │       └── publish to SQS "email-tasks" (P2P)
    │               └── 10 email workers compete
    │
    └── Consumer Group "analytics"
            └── write to data warehouse
```

### Pattern: Saga với Pub/Sub

```
OrderCreated (event)
  → PaymentRequested (command queue — P2P)
  → PaymentCompleted (event — Pub/Sub)
  → InventoryReserved (event — Pub/Sub)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào dùng Queue, khi nào dùng Topic?

**Đáp án mẫu:** Dùng **Queue** khi cần **một** consumer xử lý mỗi message (task distribution). Dùng **Topic** khi cần **nhiều** independent consumer nhận cùng message (event broadcast). Kafka dùng topic cho cả hai — phân biệt bằng consumer group.

### Câu 2: Kafka consumer group hoạt động như P2P hay Pub/Sub?

**Đáp án mẫu:** **Cả hai.** Trong group: partitions được assign cho competing consumers (P2P). Giữa các group: mỗi group đọc full topic độc lập (Pub/Sub). Đây là lý do Kafka linh hoạt cho cả work queue và event streaming.

### Câu 3: RabbitMQ Fanout vs Topic exchange?

**Đáp án mẫu:** **Fanout:** broadcast tất cả message đến mọi bound queue — không filter. **Topic:** routing theo routing key pattern (`order.*`, `order.created`) — selective subscription. Topic linh hoạt hơn cho event-driven microservices.

### Câu 4: Làm sao tránh duplicate processing trong Pub/Sub?

**Đáp án mẫu:** Pub/Sub **cố ý** gửi đến nhiều subscriber — không phải duplicate. Nếu **cùng subscriber** nhận trùng (at-least-once), dùng **idempotency key** hoặc dedup store. Phân biệt "intentional fan-out" vs "unintentional duplicate delivery".

---

**Xem tiếp:** [3-delivery-guarantees.md](./3-delivery-guarantees.md) — chủ đề phỏng vấn quan trọng nhất.
