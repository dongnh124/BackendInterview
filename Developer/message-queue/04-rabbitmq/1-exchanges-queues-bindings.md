# Exchanges, Queues & Bindings — Trao Đổi, Hàng Đợi & Liên Kết

> Hiểu AMQP 0-9-1 model trong RabbitMQ: Exchange (Bộ Trao Đổi), Queue (Hàng Đợi), Binding (Liên Kết), routing key, và bốn loại exchange — Direct, Fanout, Topic, Headers.

## Mục Lục

1. [AMQP Model](#amqp-model)
2. [Connection & Channel](#connection--channel)
3. [Queue Properties](#queue-properties)
4. [Exchange Types](#exchange-types)
5. [Binding Rules](#binding-rules)
6. [Message Properties](#message-properties)
7. [Virtual Host](#virtual-host)
8. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## AMQP Model

**AMQP (Advanced Message Queuing Protocol — Giao Thức Hàng Đợi Tin Nhắn Nâng Cao)** định nghĩa mô hình messaging mà RabbitMQ implement:

```
Publisher ──► Exchange ──binding──► Queue ──► Consumer
                │
                └── routing key + exchange type quyết định route
```

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Publisher (Nhà Xuất Bản)** | Gửi message đến exchange (không gửi trực tiếp queue) |
| **Exchange** | Router — nhận message, quyết định queue đích |
| **Binding** | Rule liên kết exchange ↔ queue |
| **Queue** | Buffer lưu message cho consumer |
| **Consumer** | Nhận và xử lý message từ queue |

> **Quan trọng:** Publisher **không biết** queue nào sẽ nhận message — chỉ biết exchange và routing key. Đây là **decoupling (tách rời)** cốt lõi.

---

## Connection & Channel

### Connection (Kết Nối)

TCP connection giữa client và RabbitMQ broker — tốn tài nguyên, nên **ít connection, nhiều channel**.

### Channel (Kênh)

**Virtual connection** bên trong TCP connection — lightweight, thread-safe khi mỗi thread dùng channel riêng.

```javascript
// Node.js — amqplib
const connection = await amqp.connect('amqp://localhost');
const channel = await connection.createChannel();

await channel.assertExchange('orders.topic', 'topic', { durable: true });
await channel.assertQueue('orders.email', { durable: true });
await channel.bindQueue('orders.email', 'orders.topic', 'order.created');
```

```python
# Python — pika
connection = pika.BlockingConnection(pika.URLParameters('amqp://localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='orders.topic', exchange_type='topic', durable=True)
channel.queue_declare(queue='orders.email', durable=True)
channel.queue_bind(exchange='orders.topic', queue='orders.email', routing_key='order.created')
```

---

## Queue Properties

### Durable vs Transient

| Thuộc Tính | Ý Nghĩa | Khi Nào Dùng |
| ---------- | ------- | ------------ |
| **durable: true** | Queue survive broker restart | Production — luôn dùng |
| **durable: false** | Queue mất khi broker restart | Dev/test tạm thời |
| **exclusive: true** | Chỉ connection tạo queue mới dùng được | RPC reply queue |
| **auto-delete: true** | Xóa khi không còn consumer | Temporary queues |

### Message Persistence

```javascript
// Queue durable + message persistent
await channel.assertQueue('tasks', { durable: true });
channel.publish('tasks.direct', 'process', Buffer.from(payload), {
  deliveryMode: 2,  // persistent — ghi disk
});
```

| deliveryMode | Ý Nghĩa |
| ------------ | ------- |
| 1 | Transient — chỉ RAM, mất khi crash |
| 2 | Persistent — ghi disk (cần queue durable) |

> **Lưu ý:** Persistent ≠ guaranteed — cần **publisher confirms** để đảm bảo broker đã persist. Xem [4-publisher-confirms.md](./4-publisher-confirms.md).

### Queue Arguments

```javascript
await channel.assertQueue('orders.retry', {
  durable: true,
  arguments: {
    'x-message-ttl': 60000,           // TTL 60 giây
    'x-max-length': 10000,              // Giới hạn queue depth
    'x-dead-letter-exchange': 'orders.dlx',
    'x-dead-letter-routing-key': 'orders.failed',
  },
});
```

---

## Exchange Types

### 1. Direct Exchange

Route message khi **routing key khớp chính xác** với binding key.

```
Exchange "tasks.direct"
  routing_key="email"  ──► Queue "email-workers"
  routing_key="sms"    ──► Queue "sms-workers"
  routing_key="push"   ──► Queue "push-workers"
```

```javascript
await channel.assertExchange('tasks.direct', 'direct', { durable: true });
await channel.bindQueue('email-workers', 'tasks.direct', 'email');
channel.publish('tasks.direct', 'email', Buffer.from('Send welcome email'));
```

**Use case:** Task routing theo loại công việc, point-to-point với nhiều đích.

### 2. Fanout Exchange

**Broadcast** — bỏ qua routing key, gửi đến **tất cả** bound queues.

```
Exchange "events.fanout"
  message ──► Queue "audit-log"
          ──► Queue "analytics"
          ──► Queue "notifications"
```

```javascript
await channel.assertExchange('events.fanout', 'fanout', { durable: true });
await channel.bindQueue('audit-log', 'events.fanout', '');  // routing key ignored
channel.publish('events.fanout', '', Buffer.from(JSON.stringify(event)));
```

**Use case:** Event broadcast, cache invalidation, fan-out notifications.

### 3. Topic Exchange

Route theo **pattern matching** trên routing key (phân tách bằng `.`).

| Ký Tự | Ý Nghĩa |
| ----- | ------- |
| `*` | Khớp đúng **một** word |
| `#` | Khớp **không hoặc nhiều** words |

```
Exchange "events.topic"

Binding: "order.*"        → Queue "order-events"
Binding: "*.error"        → Queue "error-alerts"
Binding: "user.#"         → Queue "user-all"
Binding: "payment.failed" → Queue "payment-dlq-monitor"

Routing keys:
  "order.created"     → order-events, user-all (nếu có user.*)
  "payment.error"     → error-alerts
  "user.profile.updated" → user-all
```

```javascript
await channel.assertExchange('events.topic', 'topic', { durable: true });
await channel.bindQueue('order-events', 'events.topic', 'order.*');
await channel.bindQueue('error-alerts', 'events.topic', '*.error');
channel.publish('events.topic', 'order.created', Buffer.from(payload));
```

**Use case:** Event categorization, selective subscription — pattern phổ biến nhất trong microservices.

### 4. Headers Exchange

Route theo **message headers** thay vì routing key — ít dùng hơn Topic.

```javascript
await channel.assertExchange('match.headers', 'headers', { durable: true });
await channel.bindQueue('high-priority', 'match.headers', '', {
  'x-match': 'all',           // 'all' hoặc 'any'
  'priority': 'high',
  'region': 'ap-southeast',
});
```

**Use case:** Routing phức tạp theo metadata, multi-criteria matching.

### So Sánh Exchange Types

| Exchange | Routing Key | Use Case Chính |
| -------- | ----------- | -------------- |
| **Direct** | Exact match | Task queue, simple routing |
| **Fanout** | Ignored | Broadcast, pub/sub đơn giản |
| **Topic** | Pattern `*` `#` | Event bus, selective routing |
| **Headers** | Header match | Multi-attribute routing |

---

## Binding Rules

**Binding** là liên kết giữa exchange và queue, kèm **binding key** (routing key pattern).

```
┌─────────────┐     binding: "order.*"     ┌──────────────┐
│  Exchange   │ ─────────────────────────► │ Queue A      │
│  (topic)    │     binding: "*.error"     ┌──────────────┐
│             │ ─────────────────────────► │ Queue B      │
└─────────────┘     binding: "#"           ┌──────────────┐
               ─────────────────────────► │ Queue C (all)│
                                           └──────────────┘
```

**Một queue có thể bind nhiều exchange.** **Một exchange có thể bind nhiều queue.** Message có thể đến nhiều queue (fanout, multiple bindings match).

### Default Exchange

RabbitMQ có **default exchange** (`""`) — type direct, mỗi queue tự bind với routing key = tên queue:

```javascript
// Gửi trực tiếp đến queue qua default exchange
channel.sendToQueue('my-queue', Buffer.from('hello'));
// Tương đương: publish('', 'my-queue', message)
```

---

## Message Properties

| Property | Mô Tả |
| -------- | ----- |
| **deliveryMode** | 1=transient, 2=persistent |
| **contentType** | `application/json`, `text/plain` |
| **correlationId** | Ghép request/response (RPC) |
| **replyTo** | Queue nhận response |
| **messageId** | Unique ID — deduplication |
| **timestamp** | Thời điểm tạo |
| **headers** | Custom metadata (x-trace-id, x-retry-count) |
| **expiration** | Per-message TTL (ms string) |

```javascript
channel.publish('events.topic', 'order.created', Buffer.from(JSON.stringify(order)), {
  contentType: 'application/json',
  messageId: order.id,
  headers: { 'x-trace-id': traceId, 'x-source': 'order-service' },
  persistent: true,
});
```

---

## Virtual Host

**Virtual Host (vhost)** — namespace logic tách biệt exchanges, queues, permissions.

```
RabbitMQ Broker
├── vhost: /production
│   ├── exchanges, queues, bindings
│   └── users với permissions riêng
├── vhost: /staging
└── vhost: /dev
```

```bash
# rabbitmqctl
rabbitmqctl add_vhost /production
rabbitmqctl set_permissions -p /production app_user ".*" ".*" ".*"
```

**Production:** Mỗi môi trường hoặc tenant một vhost — tránh cross-contamination.

---

## Thiết Kế Thực Tế

### Naming Convention

```
Exchange:  {domain}.{type}     → orders.topic, notifications.fanout
Queue:     {service}.{purpose} → email-svc.send, inventory-svc.reserve
DLQ:       {queue}.dlq        → email-svc.send.dlq
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề | Giải Pháp |
| ------------ | ------ | --------- |
| Publisher gửi thẳng queue | Mất flexibility routing | Luôn qua exchange |
| Một queue cho nhiều loại message | Consumer phức tạp | Tách queue theo message type |
| Fanout khi chỉ cần 1 consumer | Lãng phí resources | Dùng Direct |
| Không durable trong production | Mất message khi restart | durable queue + persistent messages |

### Decision Tree Chọn Exchange

```
Cần gửi đến 1 queue cụ thể theo key?
  └─ YES → Direct Exchange

Cần broadcast đến tất cả subscribers?
  └─ YES → Fanout Exchange

Cần subscribe theo pattern/category?
  └─ YES → Topic Exchange

Cần route theo nhiều header attributes?
  └─ YES → Headers Exchange
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Publisher có gửi trực tiếp đến Queue không?

**Trả lời:** Trong AMQP model chuẩn, publisher gửi đến **exchange**. Tuy nhiên **default exchange** cho phép gửi "trực tiếp" bằng routing key = tên queue. Best practice production: dùng named exchange để decouple.

### Câu 2: Topic exchange `*` và `#` khác nhau thế nào?

**Trả lời:** `*` khớp **đúng một** segment (word giữa dấu `.`). `#` khớp **không hoặc nhiều** segments. Ví dụ binding `order.*` khớp `order.created` nhưng không khớp `order.payment.created`. Binding `order.#` khớp cả hai.

### Câu 3: Một message có thể vào nhiều queue không?

**Trả lời:** Có — khi **fanout** (tất cả bound queues) hoặc **nhiều binding match** trên topic/direct exchange. Mỗi queue nhận **bản copy riêng** của message.

### Câu 4: Durable queue có đảm bảo message không mất không?

**Trả lời:** Không hoàn toàn. Durable queue chỉ đảm bảo **queue definition** survive restart. Message cần `deliveryMode=2` (persistent) **và** publisher confirms để đảm bảo broker đã ghi disk trước khi coi là thành công.

### Câu 5: Khi nào dùng Headers exchange thay Topic?

**Trả lời:** Khi routing criteria **không map được** vào routing key string — ví dụ match theo combination `priority + region + tenant`. Topic đủ cho hầu hết event routing; Headers cho edge cases phức tạp.

---

**Xem tiếp:** [2-routing-patterns.md](./2-routing-patterns.md) — work queue, pub/sub, RPC pattern.
