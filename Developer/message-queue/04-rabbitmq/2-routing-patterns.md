# Routing Patterns — Mẫu Định Tuyến

> Các Enterprise Integration Patterns (Mẫu Tích Hợp Doanh Nghiệp) phổ biến với RabbitMQ: Work Queue (Hàng Đợi Công Việc), Pub/Sub (Publish/Subscribe), Routing, Topics, RPC (Remote Procedure Call), và Priority Queue.

## Mục Lục

1. [Work Queue Pattern](#work-queue-pattern)
2. [Publish/Subscribe Pattern](#publishsubscribe-pattern)
3. [Routing Pattern](#routing-pattern)
4. [Topics Pattern](#topics-pattern)
5. [RPC Pattern](#rpc-pattern)
6. [Priority Queue](#priority-queue)
7. [Competing Consumers](#competing-consumers)
8. [Kết Hợp Patterns](#kết-hợp-patterns)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Work Queue Pattern

**Work Queue (Hàng Đợi Công Việc)** — phân phối task cho nhiều worker, mỗi task chỉ xử lý **một lần** bởi **một worker**.

```
Producer ──► Queue "tasks" ──► Worker 1
                          ──► Worker 2
                          ──► Worker 3
```

### Fair Dispatch với Prefetch

Mặc định RabbitMQ **round-robin** — worker chậm vẫn nhận message mới. Dùng **prefetch (QoS — Quality of Service)** để fair dispatch:

```javascript
await channel.prefetch(1);  // Mỗi consumer chỉ nhận 1 unacked message

channel.consume('tasks', async (msg) => {
  try {
    await processTask(msg.content.toString());
    channel.ack(msg);
  } catch (err) {
    channel.nack(msg, false, true);  // requeue
  }
});
```

| prefetch | Hành Vi |
| -------- | ------- |
| Không set | Round-robin không chờ ack |
| `prefetch(1)` | Worker xong mới nhận tiếp — fair cho task dài |
| `prefetch(N)` | Pipeline N message — cân bằng throughput vs fairness |

### Use Cases

- Background job processing (email, image resize, report generation)
- Order fulfillment workers
- Data import/export tasks

---

## Publish/Subscribe Pattern

**Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký)** — một message đến **nhiều consumer độc lập**.

```
Publisher ──► Fanout Exchange ──► Queue A ──► Subscriber 1
                              ──► Queue B ──► Subscriber 2
                              ──► Queue C ──► Subscriber 3
```

```javascript
// Publisher
await channel.assertExchange('logs.fanout', 'fanout', { durable: true });
channel.publish('logs.fanout', '', Buffer.from(logEntry));

// Mỗi subscriber tạo queue riêng (hoặc shared queue tùy use case)
await channel.assertQueue('logs.elasticsearch', { durable: true });
await channel.bindQueue('logs.elasticsearch', 'logs.fanout', '');
```

### Ephemeral vs Durable Subscriptions

| Loại Queue | Hành Vi | Use Case |
| ---------- | ------- | -------- |
| **Durable queue** | Message persist, consumer offline vẫn nhận sau | Audit, analytics |
| **Exclusive + auto-delete** | Chỉ consumer hiện tại, mất khi disconnect | Real-time UI updates |

---

## Routing Pattern

**Routing Pattern** — subscriber chọn nhận message theo **routing key** cụ thể qua Direct exchange.

```
Publisher ──► Direct Exchange ──routing_key="error"──► Queue "error-logs"
                            ──routing_key="info"───► Queue "info-logs"
                            ──routing_key="warn"───► Queue "warn-logs"
```

```javascript
const severity = 'error';
channel.publish('logs.direct', severity, Buffer.from(logMessage));
```

**Use case:** Log level filtering, task type routing, environment-specific queues (`staging`, `production`).

---

## Topics Pattern

**Topics Pattern** — mở rộng Routing với **wildcard matching** — linh hoạt nhất cho event-driven microservices.

```
                    ┌── "order.*" ──────► order-service queue
Publisher ──► Topic ──├── "*.critical" ───► alerting queue
  Exchange          └── "payment.#" ────► payment-service queue
```

### Ví Dụ Event Bus Microservices

```javascript
// Order Service publish
channel.publish('events.topic', 'order.created', Buffer.from(JSON.stringify({
  orderId: '123', userId: '456', amount: 99.99
})));

// Bindings
// inventory-service:  "order.created", "order.cancelled"
// email-service:      "order.*"
// analytics-service:  "#"  (tất cả events)
// alerting-service:   "*.critical"
```

### Thiết Kế Routing Key

```
{entity}.{action}[.{detail}]

order.created
order.cancelled
payment.completed
payment.failed.critical
user.registered
inventory.low.critical
```

**Quy tắc:** Routing key hierarchy nhất quán — dễ bind pattern, dễ debug.

---

## RPC Pattern

**RPC (Remote Procedure Call — Gọi Thủ Tục Từ Xa)** qua RabbitMQ — request/response async không cần HTTP.

```
Client                          Server
  │                               │
  │── request ──► request queue ──►│
  │   reply_to: "amq.gen-xxx"     │ process
  │   correlation_id: "abc-123"   │
  │                               │
  │◄── response ── reply queue ◄──│
  │   correlation_id: "abc-123"   │
```

### Client

```javascript
const { correlationId, replyQueue } = await setupRpcClient(channel);

const response = await new Promise((resolve, reject) => {
  const timeout = setTimeout(() => reject(new Error('RPC timeout')), 30000);

  channel.consume(replyQueue, (msg) => {
    if (msg.properties.correlationId === correlationId) {
      clearTimeout(timeout);
      resolve(JSON.parse(msg.content.toString()));
      channel.ack(msg);
    }
  }, { noAck: false });

  channel.sendToQueue('rpc_queue', Buffer.from(JSON.stringify(request)), {
    correlationId,
    replyTo: replyQueue,
    expiration: '30000',
  });
});
```

### Server

```javascript
channel.consume('rpc_queue', async (msg) => {
  const request = JSON.parse(msg.content.toString());
  const result = await handleRequest(request);

  channel.sendToQueue(msg.properties.replyTo, Buffer.from(JSON.stringify(result)), {
    correlationId: msg.properties.correlationId,
  });
  channel.ack(msg);
});
```

### RPC vs HTTP

| Tiêu Chí | RabbitMQ RPC | HTTP REST |
| -------- | ------------ | --------- |
| **Coupling** | Loose — không cần biết địa chỉ server | Tight — cần endpoint URL |
| **Load balancing** | Work queue tự động | Cần load balancer |
| **Timeout** | Tự implement | Built-in |
| **Phù hợp** | Internal async, long-running | Sync request/response |

> **Lưu ý:** RPC qua RabbitMQ phù hợp **internal services**. API public vẫn nên dùng HTTP/gRPC.

---

## Priority Queue

**Priority Queue (Hàng Đợi Ưu Tiên)** — message quan trọng xử lý trước.

```javascript
await channel.assertQueue('tasks.priority', {
  durable: true,
  arguments: { 'x-max-priority': 10 },
});

channel.sendToQueue('tasks.priority', Buffer.from('urgent task'), {
  priority: 9,
});
channel.sendToQueue('tasks.priority', Buffer.from('normal task'), {
  priority: 1,
});
```

| Lưu Ý | Chi Tiết |
| ----- | -------- |
| **Range** | 0 đến `x-max-priority` (khuyến nghị ≤ 10) |
| **Chỉ khi consumer idle** | Priority chỉ áp dụng khi có message chờ |
| **Không thay prefetch** | Vẫn cần prefetch cho fair dispatch |

---

## Competing Consumers

**Competing Consumers (Consumer Cạnh Tranh)** — nhiều consumer instance cùng đọc một queue, scale horizontal.

```
                    ┌──► Consumer Pod 1
Queue "orders" ─────┼──► Consumer Pod 2
                    └──► Consumer Pod 3
```

**Scale rules:**

- Thêm consumer instance → throughput tăng (đến giới hạn queue single-threaded)
- RabbitMQ **không** có partition như Kafka — 1 queue = 1 logical pipe
- Scale nhiều hơn 1 node cần **quorum queue** hoặc **sharded queues**

### Sharded Queues Pattern

Khi 1 queue không đủ throughput:

```
Producer ──hash(key)──► Queue shard-0 ──► Consumer group 0
                     ──► Queue shard-1 ──► Consumer group 1
                     ──► Queue shard-2 ──► Consumer group 2
```

```javascript
const shard = hash(orderId) % SHARD_COUNT;
channel.publish('orders.direct', `shard-${shard}`, Buffer.from(payload));
```

---

## Kết Hợp Patterns

### Pattern: Event Bus + Work Queue

```
                    ┌── topic bind ──► notification queue ──► email workers
Event Publisher ──► │
                    └── topic bind ──► processing queue ──► order workers
```

### Pattern: RPC + DLQ

```
RPC request queue ──► Server
                   └── DLX ──► RPC timeout/failure queue (monitoring)
```

### Pattern: Retry với TTL (xem chi tiết [3-dead-letter-queue.md](./3-dead-letter-queue.md))

```
Main queue ──fail──► Retry queue (TTL 30s) ──expire──► DLX ──► Main queue (retry)
                                                              └── max retries ──► DLQ
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Work Queue khác Pub/Sub thế nào?

**Trả lời:** **Work Queue** — nhiều worker **cùng** đọc **một** queue, mỗi message chỉ **một** worker xử lý (competing consumers). **Pub/Sub** — **mỗi** subscriber có queue riêng, **cùng** message đến **tất cả** subscribers (fanout/topic).

### Câu 2: Prefetch=1 có làm chậm throughput không?

**Trả lời:** Có thể — worker phải ack trước khi nhận message tiếp. Trade-off: fairness vs throughput. Task ngắn đồng đều → prefetch cao hơn (5–10). Task dài, thời gian xử lý chênh lệch → prefetch=1.

### Câu 3: RPC qua RabbitMQ có đảm bảo exactly-once không?

**Trả lời:** Không. Cần **idempotent handler** — request có thể duplicate (network retry, consumer redelivery). Dùng `correlationId` + `messageId` để dedup. Timeout phía client cần xử lý "request đã xử lý nhưng response mất".

### Câu 4: Làm sao scale RabbitMQ consumer khi lag tăng?

**Trả lời:** (1) Thêm consumer instance cùng queue. (2) Tăng prefetch nếu consumer underutilized. (3) Nếu 1 queue bottleneck → **shard queues** theo key. (4) Tối ưu processing time. (5) Khác Kafka — không thể tăng partition, phải shard thủ công.

### Câu 5: Topic pattern `order.#` vs `order.*`?

**Trả lời:** `order.*` nhận `order.created`, `order.cancelled` — **một** level. `order.#` nhận thêm `order.payment.completed`, `order.item.added` — **nhiều** levels. Dùng `#` khi cần catch-all subtree.

---

**Xem tiếp:** [3-dead-letter-queue.md](./3-dead-letter-queue.md) — DLX, DLQ, poison message handling.
