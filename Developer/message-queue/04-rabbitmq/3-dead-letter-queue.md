# Dead Letter Queue — Hàng Đợi Thư Chết

> Thiết kế Dead Letter Exchange (DLX — Bộ Trao Đổi Thư Chết), Dead Letter Queue (DLQ — Hàng Đợi Thư Chết), xử lý poison message (tin nhắn độc), retry pattern với TTL, và vận hành DLQ trong production.

## Mục Lục

1. [Dead Letter Là Gì?](#dead-letter-là-gì)
2. [Khi Nào Message Vào DLQ](#khi-nào-message-vào-dlq)
3. [Cấu Hình DLX](#cấu-hình-dlx)
4. [Retry Pattern với TTL](#retry-pattern-với-ttl)
5. [Poison Message Handling](#poison-message-handling)
6. [DLQ Operations](#dlq-operations)
7. [Monitoring & Alerting](#monitoring--alerting)
8. [Thiết Kế Production](#thiết-kế-production)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Dead Letter Là Gì?

**Dead Letter (Thư Chết)** — message không thể xử lý thành công sau các lần thử, được chuyển sang queue riêng để **phân tích, replay, hoặc loại bỏ** thay vì block main queue.

```
Main Queue ──fail/reject/TTL──► DLX (Dead Letter Exchange)
                                    │
                                    └──► DLQ (Dead Letter Queue)
                                              │
                                              ▼
                                         Manual review / Replay / Discard
```

| Khái Niệm | Mô Tả |
| -------- | ----- |
| **DLX** | Exchange nhận dead-lettered messages |
| **DLQ** | Queue bound to DLX — nơi lưu message thất bại |
| **Poison Message** | Message gây lỗi liên tục khi xử lý — block consumer nếu requeue vô hạn |

---

## Khi Nào Message Vào DLQ

RabbitMQ dead-letter message khi:

| Trigger | Mô Tả |
| ------- | ----- |
| **basic.reject / basic.nack** | Consumer reject với `requeue=false` |
| **Message TTL expired** | Per-message hoặc queue-level TTL hết hạn |
| **Queue length exceeded** | `x-max-length` hoặc `x-max-length-bytes` vượt ngưỡng |
| **Delivery limit** | Quorum queues — `x-delivery-limit` exceeded (RabbitMQ 3.12+) |

```
Consumer xử lý message:
  SUCCESS  → basic.ack
  RETRY    → basic.nack(requeue=true) hoặc throw → requeue
  FAIL     → basic.nack(requeue=false) → DLX → DLQ
```

---

## Cấu Hình DLX

### Setup Cơ Bản

```javascript
// 1. Tạo DLX và DLQ
await channel.assertExchange('orders.dlx', 'direct', { durable: true });
await channel.assertQueue('orders.dlq', { durable: true });
await channel.bindQueue('orders.dlq', 'orders.dlx', 'orders.failed');

// 2. Main queue với dead-letter config
await channel.assertQueue('orders.process', {
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'orders.dlx',
    'x-dead-letter-routing-key': 'orders.failed',
  },
});

// 3. Consumer reject → DLQ
channel.consume('orders.process', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg);
  } catch (err) {
    if (isPermanentError(err)) {
      channel.nack(msg, false, false);  // requeue=false → DLQ
    } else {
      channel.nack(msg, false, true);   // transient → requeue
    }
  }
});
```

### Dead Letter Headers

Khi message dead-lettered, RabbitMQ thêm headers:

| Header | Mô Tả |
| ------ | ----- |
| `x-death` | Array — lịch sử death (queue, reason, time, count) |
| `x-first-death-queue` | Queue gốc |
| `x-first-death-reason` | `rejected`, `expired`, `maxlen` |
| `x-first-death-exchange` | Exchange gốc |

```javascript
// Đọc death info khi xử lý DLQ
const deaths = msg.properties.headers['x-death'];
const retryCount = deaths?.[0]?.count ?? 0;
```

---

## Retry Pattern với TTL

### Delayed Retry qua TTL + DLX

Thay vì requeue ngay (gây tight loop), dùng **retry queue** với TTL:

```
┌─────────────┐  fail   ┌──────────────┐  TTL expire  ┌─────────────┐
│ Main Queue  │ ──────► │ Retry Queue  │ ───────────► │ Main Queue  │
│             │         │ (TTL: 30s)   │   via DLX    │ (retry)     │
└─────────────┘         └──────────────┘              └─────────────┘
       │ max retries exceeded
       ▼
┌─────────────┐
│    DLQ      │
└─────────────┘
```

```javascript
// Retry queue — message expire sau 30s, quay về main queue
await channel.assertQueue('orders.retry.30s', {
  durable: true,
  arguments: {
    'x-message-ttl': 30000,
    'x-dead-letter-exchange': '',  // default exchange
    'x-dead-letter-routing-key': 'orders.process',  // back to main
  },
});

// Consumer: transient error → gửi sang retry queue
channel.consume('orders.process', async (msg) => {
  const retryCount = msg.properties.headers['x-retry-count'] ?? 0;

  try {
    await processOrder(msg);
    channel.ack(msg);
  } catch (err) {
    if (retryCount >= MAX_RETRIES) {
      channel.nack(msg, false, false);  // → DLQ
    } else {
      channel.sendToQueue('orders.retry.30s', msg.content, {
        headers: { 'x-retry-count': retryCount + 1 },
        persistent: true,
      });
      channel.ack(msg);  // remove from main queue
    }
  }
});
```

### Exponential Backoff

```
Retry queues với TTL tăng dần:
  orders.retry.30s   → 30 giây
  orders.retry.5m    → 5 phút
  orders.retry.30m   → 30 phút
  → DLQ sau 3 retries
```

| Retry | TTL | Tổng Chờ |
| ----- | --- | -------- |
| 1 | 30s | 30s |
| 2 | 5m | ~5.5m |
| 3 | 30m | ~35.5m |
| 4 | — | DLQ |

> **Plugin:** `rabbitmq_delayed_message_exchange` — delay chính xác hơn TTL+DLX nhưng cần cài plugin.

---

## Poison Message Handling

**Poison Message (Tin Nhắn Độc)** — message luôn fail khi xử lý, gây infinite requeue loop.

### Triệu Chứng

```
Consumer log: liên tục cùng messageId fail
Queue depth: không giảm dù consumer chạy
CPU: spike do retry loop
```

### Phòng Ngừa

```javascript
// 1. Phân loại lỗi
function isPermanentError(err) {
  return err instanceof ValidationError
    || err instanceof NotFoundError
    || err.code === 'INVALID_PAYLOAD';
}

// 2. Giới hạn retry
const MAX_RETRIES = 3;

// 3. Delivery limit trên quorum queue
await channel.assertQueue('orders.process', {
  durable: true,
  arguments: {
    'x-queue-type': 'quorum',
    'x-delivery-limit': 5,  // auto dead-letter sau 5 deliveries
    'x-dead-letter-exchange': 'orders.dlx',
  },
});
```

### Quarantine Pattern

```
DLQ ──► Quarantine Service ──► classify:
                                  ├── fixable → fix + replay to main queue
                                  ├── bug → Jira ticket + discard
                                  └── data issue → notify upstream
```

---

## DLQ Operations

### Replay Message

```javascript
// Manual replay từ DLQ về main queue
channel.consume('orders.dlq', (msg) => {
  const payload = msg.content;
  const headers = { ...msg.properties.headers, 'x-retry-count': 0 };

  channel.sendToQueue('orders.process', payload, {
    headers,
    persistent: true,
  });
  channel.ack(msg);  // remove from DLQ
}, { noAck: false });
```

### Purge DLQ

```bash
# CLI — xóa tất cả message trong DLQ (cẩn thận!)
rabbitmqctl purge_queue orders.dlq -p /production
```

### Shovel Plugin — Auto Replay

**Shovel** — move message từ DLQ về main queue theo schedule hoặc điều kiện.

---

## Monitoring & Alerting

### Metrics Cần Theo Dõi

| Metric | Ngưỡng Alert | Ý Nghĩa |
| ------ | ------------ | ------- |
| **DLQ depth** | > 0 sustained | Có message fail cần xử lý |
| **DLQ growth rate** | > N/phút | Bug mới hoặc upstream issue |
| **x-death count** | retry > 3 | Poison message candidate |
| **Main queue depth** | tăng liên tục | Consumer không kịp hoặc poison loop |

### Management API

```bash
# Queue stats
curl -u guest:guest http://localhost:15672/api/queues/%2F/orders.dlq
```

```json
{
  "messages": 42,
  "messages_ready": 42,
  "message_stats": {
    "publish": 150,
    "deliver": 108
  }
}
```

---

## Thiết Kế Production

### Checklist DLQ

```
□ Mỗi business-critical queue có DLQ riêng
□ DLQ naming: {main-queue}.dlq
□ Alert khi DLQ depth > 0
□ Runbook: ai xử lý, SLA, replay procedure
□ Log x-death headers khi message vào DLQ
□ Max retry + exponential backoff
□ Idempotent consumer — replay an toàn
□ Dashboard: DLQ depth, death reason breakdown
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề |
| ------------ | ------ |
| Không có DLQ | Poison message block queue vô thời hạn |
| Requeue vô hạn | Tight loop, waste resources |
| DLQ chung cho tất cả | Khó debug, khó prioritize |
| Không monitor DLQ | Message fail im lặng |
| Replay không idempotent | Duplicate side effects |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Message vào DLQ qua những cách nào?

**Trả lời:** (1) Consumer `nack/reject` với `requeue=false`. (2) Message/queue TTL expired. (3) Queue đầy (`x-max-length`). (4) Quorum queue `x-delivery-limit` exceeded.

### Câu 2: Retry với requeue=true có đủ không?

**Trả lời:** Không cho production. Requeue ngay gây **tight loop** khi lỗi persistent. Nên dùng **retry queue + TTL + DLX** cho delay, hoặc **exponential backoff**. Giới hạn max retries trước khi vào DLQ.

### Câu 3: Làm sao replay message từ DLQ an toàn?

**Trả lời:** (1) **Idempotent consumer** — replay không gây duplicate side effects. (2) Reset retry counter. (3) Fix root cause trước khi replay hàng loạt. (4) Replay từng batch nhỏ, monitor. (5) Log `messageId` để audit.

### Câu 4: DLQ depth = 0 có nghĩa hệ thống healthy không?

**Trả lời:** Không hoàn toàn. DLQ=0 có thể do: (1) Thực sự không có lỗi. (2) Message bị **drop** thay vì DLQ (reject không cấu hình DLX). (3) DLQ bị purge thường xuyên che giấu vấn đề. Cần monitor cả error rate consumer và death reason.

### Câu 5: Quorum queue `x-delivery-limit` khác gì manual retry count?

**Trả lời:** `x-delivery-limit` là **broker-enforced** — đếm số lần deliver (bao gồm requeue), tự dead-letter khi vượt limit. Manual retry count trong headers do application quản lý — linh hoạt hơn (phân loại lỗi, backoff) nhưng cần implement đúng. Có thể kết hợp cả hai.

---

**Xem tiếp:** [4-publisher-confirms.md](./4-publisher-confirms.md) — publisher confirms và consumer acknowledgment.
