# Idempotency & Deduplication — Tính Bất Biến & Khử Trùng Lặp

> Với **At-least-once delivery (Giao Hàng Ít Nhất Một Lần)** — mặc định production — consumer **phải idempotent (bất biến khi lặp lại)**. Đây là best practice số 1 khi thiết kế message handlers.

## Mục Lục

1. [Idempotency Là Gì?](#idempotency-là-gì)
2. [Tại Sao Messaging Cần Idempotency](#tại-sao-messaging-cần-idempotency)
3. [Idempotency Key Design](#idempotency-key-design)
4. [Dedup Strategies — Chiến Lược Khử Trùng](#dedup-strategies--chiến-lược-khử-trùng)
5. [Natural Idempotency](#natural-idempotency)
6. [Inbox Pattern Overview](#inbox-pattern-overview)
7. [Implementation Patterns](#implementation-patterns)
8. [TTL & Cleanup](#ttl--cleanup)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Idempotency Là Gì?

**Idempotency (Tính Bất Biến Khi Lặp Lại)** — thực hiện cùng operation nhiều lần cho **cùng kết quả** như thực hiện một lần.

```
Lần 1: processPayment(orderId=123, amount=100) → charged ✓
Lần 2: processPayment(orderId=123, amount=100) → no double charge ✓
Lần 3: processPayment(orderId=123, amount=100) → no double charge ✓
```

**Không idempotent:**

```
Lần 1: balance += 100  → balance = 200
Lần 2: balance += 100  → balance = 300  ❌ (duplicate effect)
```

**Idempotent:**

```
Lần 1: SET balance = 200 WHERE id=1  → balance = 200
Lần 2: SET balance = 200 WHERE id=1  → balance = 200  ✓
```

---

## Tại Sao Messaging Cần Idempotency

Duplicate message xảy ra ở **nhiều điểm**:

```
1. Producer retry     → cùng message gửi 2 lần
2. Consumer crash     → chưa ack, broker redeliver
3. Rebalance          → partition chuyển, message xử lý lại
4. Network timeout    → producer không chắc đã gửi → retry
```

```
Producer ──msg──► Broker ──msg──► Consumer (crash trước ack)
                  │
                  └── redeliver ──► Consumer (duplicate!)
```

| Không Idempotent | Hậu Quả |
| ---------------- | ------- |
| Charge payment 2 lần | Mất tiền khách |
| Reserve inventory 2 lần | Oversell |
| Gửi email 2 lần | UX tệ |
| Insert duplicate record | Data inconsistency |

> **Quy tắc vàng:** Thiết kế mọi consumer **idempotent by default** — assume message có thể đến nhiều lần.

---

## Idempotency Key Design

**Idempotency Key (Khóa Bất Biến)** — unique identifier cho một logical operation.

### Nguồn Key

| Nguồn | Ví Dụ | Ghi Chú |
| ----- | ----- | ------- |
| **Business ID** | `orderId + eventType` | Đơn giản, dễ debug |
| **Message ID** | `eventId` từ envelope | Unique per message |
| **Broker metadata** | Kafka offset, RabbitMQ deliveryTag | Không portable khi redeliver |
| **Client-generated** | UUID từ API caller | Cho HTTP + async combo |

### Best Practices

```
✅ Key stable across retries của CÙNG logical event
   idempotencyKey = "order-12345-created"

✅ Key unique across KHÁC logical events
   "order-12345-created" vs "order-12345-paid"

❌ Dùng timestamp làm key — mỗi retry khác key
❌ Dùng auto-increment message ID nếu producer retry tạo message mới
```

### Event Envelope

```json
{
  "metadata": {
    "eventId": "evt-uuid-001",
    "idempotencyKey": "order-ORD-12345-created",
    "correlationId": "req-abc"
  },
  "payload": { "orderId": "ORD-12345" }
}
```

**Quy ước:** `idempotencyKey` = business-meaningful; `eventId` = unique per publish attempt.

---

## Dedup Strategies — Chiến Lược Khử Trùng

### 1. Dedup Table (Bảng Khử Trùng)

```
┌─────────────────────────────────────┐
│  processed_messages                  │
│  ─────────────────────────────────  │
│  idempotency_key (PK)               │
│  processed_at                       │
│  result_hash (optional)             │
└─────────────────────────────────────┘
```

**Flow:**

```
1. BEGIN transaction
2. SELECT FROM processed_messages WHERE key = ?
3. IF exists → COMMIT, return (skip processing)
4. Process business logic
5. INSERT INTO processed_messages (key, processed_at)
6. COMMIT
```

```sql
-- PostgreSQL example
CREATE TABLE processed_messages (
  idempotency_key VARCHAR(255) PRIMARY KEY,
  processed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload_hash    VARCHAR(64)
);

-- Trong consumer handler (pseudo)
BEGIN;
  IF EXISTS (SELECT 1 FROM processed_messages WHERE idempotency_key = $1) THEN
    ROLLBACK; RETURN 'already_processed';
  END IF;

  -- business logic
  UPDATE inventory SET qty = qty - 2 WHERE sku = 'ABC';

  INSERT INTO processed_messages (idempotency_key) VALUES ($1);
COMMIT;
```

### 2. Redis Dedup (TTL-based)

```javascript
async function processMessage(msg) {
  const key = `dedup:${msg.idempotencyKey}`;
  const isNew = await redis.set(key, '1', 'NX', 'EX', 86400); // 24h TTL

  if (!isNew) {
    console.log('Duplicate — skip');
    return;
  }

  await handleBusinessLogic(msg);
}
```

| Ưu | Nhược |
| --- | ----- |
| Nhanh, đơn giản | Không atomic với DB business logic |
| TTL tự cleanup | Mất dedup nếu Redis flush |
| Phù hợp high throughput | Cần SET NX trong cùng flow cẩn thận |

**Khuyến nghị:** Redis cho fast path check; DB dedup table khi cần **atomic với business transaction**.

### 3. Unique Constraint (Ràng Buộc Duy Nhất)

```sql
CREATE TABLE payments (
  id              UUID PRIMARY KEY,
  order_id        VARCHAR(50) NOT NULL,
  idempotency_key VARCHAR(255) UNIQUE NOT NULL,
  amount          DECIMAL,
  status          VARCHAR(20)
);

-- Duplicate insert → constraint violation → treat as success
INSERT INTO payments (id, order_id, idempotency_key, amount, status)
VALUES (gen_random_uuid(), 'ORD-123', 'pay-ORD-123', 100000, 'COMPLETED')
ON CONFLICT (idempotency_key) DO NOTHING;
```

---

## Natural Idempotency

Một số operation **tự nhiên idempotent** — không cần dedup table riêng:

| Operation | Cách |
| --------- | ---- |
| **UPSERT** | `INSERT ... ON CONFLICT UPDATE` |
| **Conditional update** | `UPDATE ... WHERE status = 'PENDING'` |
| **SET absolute value** | `SET balance = 200` thay vì `+= 100` |
| **Delete by ID** | Xóa lần 2 không effect |
| **External API với idempotency key** | Stripe, PayPal hỗ trợ sẵn |

```javascript
// Conditional update — chỉ transition 1 lần
const result = await db.query(`
  UPDATE orders SET status = 'PAID'
  WHERE order_id = $1 AND status = 'PENDING'
  RETURNING *
`, [orderId]);

if (result.rowCount === 0) {
  // Đã PAID rồi hoặc không tồn tại — idempotent skip
  return;
}
```

---

## Inbox Pattern Overview

**Inbox Pattern (Mẫu Hộp Thư Đến)** — lưu incoming message vào DB trước khi xử lý, đảm bảo **exactly-once processing semantics (ngữ nghĩa xử lý đúng một lần)** trong phạm vi service.

```
Broker ──► Consumer ──► INSERT inbox (message_id UNIQUE)
                              │
                              ▼
                         Process + mark processed
                         (same DB transaction)
```

Chi tiết kết hợp với Outbox tại [4-outbox-inbox-pattern.md](./4-outbox-inbox-pattern.md).

---

## Implementation Patterns

### Pattern A: Check-Then-Act (Đơn Giản)

```javascript
async function handleOrderCreated(event) {
  const key = `order-created:${event.orderId}`;

  const existing = await db.findProcessed(key);
  if (existing) return;

  await reserveInventory(event);
  await db.markProcessed(key);
}
```

**Rủi ro:** Race condition — 2 consumer cùng check, cùng pass. Cần **unique constraint** hoặc **transaction lock**.

### Pattern B: Process-in-Transaction (An Toàn)

```javascript
async function handleOrderCreated(event) {
  await db.transaction(async (tx) => {
    const inserted = await tx.tryInsert('processed_messages', {
      idempotency_key: `order-created:${event.orderId}`,
    });
    if (!inserted) return; // duplicate

    await reserveInventory(event, tx);
  });
}
```

### Pattern C: Outbox + Idempotent Consumer (Production)

```
Service A: DB transaction (business + outbox)
     ↓
Relay publish (at-least-once)
     ↓
Service B: Inbox dedup + idempotent handler
```

---

## TTL & Cleanup

Dedup records tích lũy — cần **TTL (Time To Live — Thời Gian Sống)** và cleanup:

| Strategy | Mô Tả |
| -------- | ----- |
| **TTL column** | `expires_at = processed_at + interval '7 days'` |
| **Partition by date** | Drop partition cũ hàng tháng |
| **Redis EX** | Auto expire sau N giây |
| **Retention policy** | Giữ đủ lâu cho max redelivery window |

```sql
-- Cleanup job chạy hàng ngày
DELETE FROM processed_messages
WHERE processed_at < NOW() - INTERVAL '30 days';
```

**Cân nhắc:** TTL phải **dài hơn** max retry/redelivery period — nếu message redeliver sau 7 ngày mà dedup đã xóa → duplicate effect.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Idempotency khác deduplication?

**Đáp án mẫu:** **Idempotency** là property của operation — kết quả không đổi khi lặp lại. **Deduplication** là mechanism detect và skip duplicate message. Dedup là cách **implement** idempotency trong messaging context.

### Câu 2: Idempotency key nên đặt ở đâu?

**Đáp án mẫu:** Trong **message metadata/envelope** — không chỉ rely broker message ID vì producer retry có thể tạo message mới. Key nên **business-meaningestable**: `orderId + eventType` hoặc client-provided UUID. Producer và consumer agree on format.

### Câu 3: Redis dedup vs DB dedup table?

**Đáp án mẫu:** **Redis:** nhanh, TTL built-in, phù hợp high volume — nhưng không atomic với DB business logic, risk mất data Redis. **DB dedup table:** atomic trong transaction với business write — reliable hơn cho payment, inventory. Production thường: DB cho critical path, Redis cho best-effort notifications.

### Câu 4: Handler đã chạy xong nhưng crash trước mark processed — xử lý?

**Đáp án mẫu:** Message redeliver → handler chạy lại. Nếu business logic **idempotent** (conditional update, UPSERT) → OK. Nếu không → cần dedup check **trước** side effect, hoặc mark processed **trong cùng transaction** với business write. Pattern: insert dedup record + business logic atomic.

### Câu 5: Có cần idempotency với exactly-once Kafka?

**Đáp án mẫu:** **Có** — Kafka EOS chỉ cover write-to-Kafka và read-process-write trong Kafka ecosystem. External side effects (DB, email, payment API) vẫn at-least-once. **Effective exactly-once = broker guarantees + idempotent handlers** — xem [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md).

---

**Xem tiếp:** [4-outbox-inbox-pattern.md](./4-outbox-inbox-pattern.md) — atomic publish và inbox dedup trong production.
