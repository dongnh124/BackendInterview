# Publisher Confirms & Consumer Ack — Xác Nhận Publisher & Consumer

> Đảm bảo reliable delivery (giao hàng tin cậy) trong RabbitMQ: Publisher Confirms (Xác Nhận Publisher), Consumer Acknowledgment (Xác Nhận Consumer), mandatory flag, và mapping sang delivery semantics (at-most-once, at-least-once).

## Mục Lục

1. [Delivery Semantics Trong RabbitMQ](#delivery-semantics-trong-rabbitmq)
2. [Publisher Confirms](#publisher-confirms)
3. [Mandatory & Returns](#mandatory--returns)
4. [Consumer Acknowledgment](#consumer-acknowledgment)
5. [Ack Modes](#ack-modes)
6. [End-to-End Reliability](#end-to-end-reliability)
7. [Transaction vs Confirm](#transaction-vs-confirm)
8. [Troubleshooting](#troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Delivery Semantics Trong RabbitMQ

| Semantic | Publisher | Consumer | Rủi Ro |
| -------- | --------- | -------- | ------ |
| **At-most-once** | Không confirm | auto-ack | Message loss |
| **At-least-once** | Confirms + persistent | manual ack sau xử lý | Duplicate |
| **Exactly-once** | Không native | Cần idempotent consumer + dedup | Phức tạp |

```
RabbitMQ native: at-most-once hoặc at-least-once
Exactly-once: at-least-once + idempotent consumer + deduplication key
```

> Xem thêm: [delivery guarantees](../01-fundamentals/3-delivery-guarantees.md), [idempotency](../02-architecture-patterns/5-idempotency-dedup.md).

---

## Publisher Confirms

**Publisher Confirms (Xác Nhận Publisher)** — broker ack async rằng message đã được nhận và (với persistent) ghi disk.

### Bật Confirm Mode

```javascript
const channel = await connection.createConfirmChannel();

channel.publish('orders.topic', 'order.created', Buffer.from(payload), {
  persistent: true,
}, (err, ok) => {
  if (err) {
    console.error('Message NOT confirmed:', err);
    // Retry hoặc lưu vào outbox
  } else {
    console.log('Message confirmed by broker');
  }
});

// Hoặc dùng Promise wrapper
await new Promise((resolve, reject) => {
  channel.publish('orders.topic', 'order.created', Buffer.from(payload),
    { persistent: true },
    (err) => err ? reject(err) : resolve()
  );
});
```

```python
# Python pika — confirm_delivery
channel.confirm_delivery()
channel.basic_publish(
    exchange='orders.topic',
    routing_key='order.created',
    body=payload,
    properties=pika.BasicProperties(delivery_mode=2),
)
# Raises exception nếu broker nack
```

### Confirm vs Transaction

| | Publisher Confirm | AMQP Transaction |
| - | ----------------- | ---------------- |
| **Performance** | Cao — async batch | Thấp — synchronous |
| **Scope** | Per-message confirm | Multi-operation atomic |
| **Khuyến nghị** | Production default | Legacy — tránh dùng |

### Batch Confirms

```javascript
// Gửi nhiều message, wait tất cả confirms
for (const order of orders) {
  channel.publish('orders.topic', 'order.created', Buffer.from(JSON.stringify(order)), {
    persistent: true,
  });
}

await new Promise((resolve, reject) => {
  channel.waitForConfirms((err) => err ? reject(err) : resolve());
});
```

### Outbox Pattern Integration

```
DB Transaction:
  1. INSERT order
  2. INSERT outbox_event
COMMIT

Outbox Poller:
  3. Read outbox → publish với confirms
  4. Confirm OK → mark outbox processed
  5. Confirm fail → retry poller
```

> Xem: [Outbox Pattern](../02-architecture-patterns/4-outbox-inbox-pattern.md)

---

## Mandatory & Returns

### Mandatory Flag

Khi `mandatory=true`, nếu message **không route được** đến queue nào, broker **return** message về publisher thay vì drop im lặng.

```javascript
channel.publish('orders.topic', 'unknown.key', Buffer.from(payload), {
  mandatory: true,
  persistent: true,
});

channel.on('return', (msg) => {
  console.error('Unroutable message:', msg.fields.routingKey);
  // Alert — binding config sai
});
```

| Flag | Message Không Route Được |
| ---- | ------------------------ |
| `mandatory: false` (default) | **Drop** silently |
| `mandatory: true` | **Return** to publisher via `basic.return` |

### Alternate Exchange

Cấu hình **alternate exchange** — catch message không match binding:

```javascript
await channel.assertExchange('orders.topic', 'topic', {
  durable: true,
  arguments: { 'alternate-exchange': 'orders.unrouted' },
});

await channel.assertExchange('orders.unrouted', 'fanout', { durable: true });
await channel.assertQueue('orders.unrouted.log', { durable: true });
await channel.bindQueue('orders.unrouted.log', 'orders.unrouted', '');
```

---

## Consumer Acknowledgment

**Consumer Ack (Xác Nhận Consumer)** — consumer báo broker đã xử lý xong, broker mới xóa message khỏi queue.

```
Broker ──deliver──► Consumer ──process──► ack ──► Broker xóa message
                                          nack ──► requeue hoặc DLQ
```

### Manual Ack

```javascript
channel.consume('orders.process', async (msg) => {
  try {
    await processOrder(JSON.parse(msg.content.toString()));
    channel.ack(msg);  // basic.ack — success
  } catch (err) {
    channel.nack(msg, false, false);  // basic.nack — multiple=false, requeue=false → DLQ
  }
}, { noAck: false });  // manual ack mode
```

### Ack Methods

| Method | Ý Nghĩa |
| ------ | ------- |
| **basic.ack** | Xử lý thành công — broker xóa message |
| **basic.nack** | Xử lý thất bại — `requeue=true` retry, `false` → DLQ |
| **basic.reject** | Tương tự nack nhưng chỉ 1 message |

```javascript
// ack multiple messages cùng lúc
channel.ack(msg, true);  // multiple=true — ack tất cả unacked đến deliveryTag này

// nack với requeue
channel.nack(msg, false, true);   // requeue=true — thử lại
channel.nack(msg, false, false);  // requeue=false — dead letter
```

---

## Ack Modes

| Mode | Config | Hành Vi | Use Case |
| ---- | ------ | ------- | -------- |
| **Auto-ack** | `noAck: true` | Ack ngay khi deliver | Dev/test, message loss acceptable |
| **Manual ack** | `noAck: false` | Ack sau khi xử lý xong | Production |
| **Publisher ack** | confirm channel | Broker confirm nhận message | Production publishers |

### Auto-Ack Risk

```javascript
// NGUY HIỂM trong production
channel.consume('orders', handler, { noAck: true });
// Consumer crash sau nhận message nhưng trước xử lý xong → MESSAGE LOST
```

### Ack Timing Best Practice

```
ĐÚNG:  receive → process → side effects commit → ack
SAI:   receive → ack → process  (crash giữa chừng = message lost)
SAI:   receive → process → ack → DB commit fails  (duplicate khi redeliver)
```

**Thứ tự an toàn:**

1. Xử lý business logic
2. Commit side effects (DB, external API)
3. **Sau đó** `ack`

Hoặc dùng **idempotent consumer** + ack sau process (chấp nhận at-least-once duplicate).

---

## End-to-End Reliability

### Reliable Publishing Checklist

```
□ confirm channel (publisher confirms)
□ durable exchange + queue
□ persistent messages (deliveryMode=2)
□ mandatory=true hoặc alternate exchange
□ Outbox pattern cho DB + publish atomicity
```

### Reliable Consuming Checklist

```
□ manual ack (noAck: false)
□ ack SAU KHI xử lý thành công
□ idempotent handler
□ DLQ cho permanent failures
□ prefetch phù hợp
□ graceful shutdown — ack/nack pending messages
```

### Graceful Shutdown

```javascript
process.on('SIGTERM', async () => {
  // Stop nhận message mới
  await channel.cancel(consumerTag);

  // Chờ xử lý in-flight (với timeout)
  await drainInFlightMessages(30000);

  await channel.close();
  await connection.close();
});
```

---

## Transaction vs Confirm

**AMQP Transactions** (`tx.select`, `tx.commit`) — atomic multi-operation nhưng **rất chậm**, không khuyến nghị.

```javascript
// KHÔNG khuyến nghị production
channel.tx.select();
channel.publish(...);
channel.publish(...);
channel.tx.commit();
```

**Publisher Confirms** — lightweight, async, đủ cho hầu hết use case.

---

## Troubleshooting

### Message Loss Scenarios

| Scenario | Nguyên Nhân | Fix |
| -------- | ----------- | --- |
| Broker crash trước persist | Non-persistent message | `deliveryMode=2` |
| Publisher không biết fail | Không dùng confirms | Confirm channel |
| Consumer crash mid-process | Auto-ack | Manual ack |
| Unroutable message | Binding sai | `mandatory=true` |
| Dual-write DB + publish | Không atomic | Outbox pattern |

### Duplicate Scenarios

| Scenario | Nguyên Nhân | Fix |
| -------- | ----------- | --- |
| Consumer ack sau crash | Redelivery | Idempotent consumer |
| Publisher retry | Confirm timeout | Dedup by messageId |
| Requeue loop | nack requeue=true | Max retry + DLQ |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Publisher Confirm đảm bảo gì?

**Trả lời:** Broker đã **nhận** message và với persistent message đã **ghi disk** (hoặc vào memory queue durable). **Không** đảm bảo consumer đã xử lý — cần consumer ack riêng. Confirm = at-least-once từ producer đến broker.

### Câu 2: Ack trước hay sau khi xử lý?

**Trả lời:** **Sau** khi xử lý thành công và side effects committed. Ack trước → crash = message loss. Nếu ack sau nhưng chưa idempotent → redelivery = duplicate. Giải pháp: ack sau + idempotent handler.

### Câu 3: `mandatory` khác `persistent` thế nào?

**Trả lời:** **Persistent** — message survive broker restart (ghi disk). **Mandatory** — message phải route được đến ít nhất 1 queue, không thì return về publisher. Hai flag độc lập, giải quyết vấn đề khác nhau.

### Câu 4: Làm exactly-once với RabbitMQ?

**Trả lời:** RabbitMQ **không có** native exactly-once. Thực tế: **at-least-once** (confirms + manual ack) + **idempotent consumer** (dedup bằng `messageId` hoặc business key) + **Outbox pattern** cho DB consistency. Tương tự cách hầu hết team xử lý với Kafka.

### Câu 5: Confirm timeout — publisher nên làm gì?

**Trả lời:** (1) Retry publish với cùng `messageId` — consumer dedup. (2) Outbox pattern — poller retry đến khi confirm. (3) Không assume message lost ngay — có thể broker đã nhận nhưng confirm chậm. (4) Monitor unroutable returns và nack từ broker.

---

**Xem tiếp:** [5-clustering-ha.md](./5-clustering-ha.md) — clustering, mirrored queues, quorum queues.
