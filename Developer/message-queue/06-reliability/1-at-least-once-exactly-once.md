# At-Least-Once vs Exactly-Once — Delivery Semantics Thực Tế

> Trong production, câu hỏi không phải "Exactly-once có khả thi không?" mà là **"Làm sao đạt effective exactly-once với At-least-once + idempotency?"** — đây là pattern được dùng rộng rãi nhất.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [At-Least-Once Trong Production](#at-least-once-trong-production)
3. [Exactly-Once — Lời Hứa và Thực Tế](#exactly-once--lời-hứa-và-thực-tế)
4. [Effective Exactly-Once Pattern](#effective-exactly-once-pattern)
5. [So Sánh Theo Broker](#so-sánh-theo-broker)
6. [Ma Trận Quyết Định](#ma-trận-quyết-định)
7. [Anti-Patterns](#anti-patterns)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│           DELIVERY SEMANTICS TRONG PRODUCTION                    │
├─────────────────────────────────────────────────────────────────┤
│  At-most-once     → Metrics, telemetry (chấp nhận mất data)     │
│  At-least-once    → DEFAULT — 90%+ use cases                    │
│  Exactly-once     → Hiếm — billing, ledger (rất phức tạp)     │
│  Effective E-O    → At-least-once + Idempotency (thực tế nhất)  │
└─────────────────────────────────────────────────────────────────┘
```

| Semantic | Message Loss | Duplicate | Độ Phức Tạp | Production Usage |
| -------- | ------------ | --------- | ----------- | ---------------- |
| **At-most-once (Tối Đa Một Lần)** | Có thể | Không | Thấp | ~5% |
| **At-least-once (Ít Nhất Một Lần)** | Không* | Có thể | Trung bình | **~85%** |
| **Exactly-once (Đúng Một Lần)** | Không | Không | Rất cao | ~5% |
| **Effective exactly-once** | Không* | Không (logic) | Trung bình | **Khuyến nghị** |

\* Với broker replication và persistence đúng cách.

> **Đọc thêm:** Lý thuyết cơ bản tại [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md).

---

## At-Least-Once Trong Production

**At-least-once (Ít Nhất Một Lần)** — message được deliver **ít nhất một lần**, có thể **duplicate (trùng lặp)**.

### Tại Sao Là Default

```
1. Không mất message — critical cho business data
2. Đơn giản hơn exactly-once — ít moving parts
3. Duplicate xử lý được bằng idempotency — pattern đã mature
4. Mọi broker chính đều hỗ trợ tốt
```

### Ba Điểm Duplicate Xảy Ra

```
Producer                    Broker                     Consumer
    │                          │                           │
    ├── retry send ───────────►│ (duplicate in broker)     │
    │                          │                           │
    │                          ├── redeliver ─────────────►│ (chưa ack)
    │                          │                           │
    │                          │◄── rebalance ─────────────┤ (partition move)
```

| Điểm | Nguyên Nhân | Ví Dụ |
| ---- | ----------- | ----- |
| **Producer** | Retry sau timeout | `send()` timeout → gửi lại → 2 message |
| **Broker** | Redelivery khi consumer chưa ack | Consumer crash trước commit offset |
| **Consumer** | Rebalance, restart | Kafka partition reassignment |

### Cấu Hình At-Least-Once

```javascript
// Kafka — at-least-once
const producer = kafka.producer({
  'acks': 'all',                    // Chờ tất cả ISR (In-Sync Replicas) ack
  'enable.idempotence': true,     // Tránh duplicate từ producer retry
});

// Consumer — manual commit SAU khi xử lý xong
await consumer.run({
  autoCommit: false,
  eachMessage: async ({ message }) => {
    await processMessage(message);
    await consumer.commitOffsets([...]);  // Ack sau success
  },
});
```

```javascript
// RabbitMQ — at-least-once
channel.consume('orders', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg);           // Manual ack sau xử lý
  } catch (err) {
    channel.nack(msg, false, true);  // Requeue → at-least-once
  }
}, { noAck: false });
```

### Trade-off Chấp Nhận

| Ưu Điểm | Nhược Điểm | Giải Pháp |
| ------- | ---------- | --------- |
| Không mất message | Có duplicate | Idempotent handler |
| Đơn giản, proven | Cần dedup logic | Idempotency key + dedup table |
| Scale tốt | Side effect nếu không idempotent | Natural idempotency, UPSERT |

---

## Exactly-Once — Lời Hứa và Thực Tế

**Exactly-once (Đúng Một Lần)** — message được process **đúng một lần**, không mất, không trùng — **end-to-end**.

### Vấn Đề Cốt Lõi

Exactly-once yêu cầu **atomicity (Tính Nguyên Tử)** giữa:
- Đọc message từ broker
- Xử lý business logic
- Ghi side effect (DB, API)
- Ack/commit offset

```
Đây là distributed transaction — không có silver bullet.
```

### Exactly-Once Trong Kafka

Kafka cung cấp **EOS (Exactly-Once Semantics)** cho:
- **Producer → Broker:** `enable.idempotence=true` + transactional producer
- **Broker → Consumer (Kafka Streams):** transactional processing với offset commit trong cùng transaction

```javascript
// Kafka transactional producer
const producer = kafka.producer({
  transactionalId: 'order-producer-1',
  'enable.idempotence': true,
});

const transaction = await producer.transaction();
try {
  await transaction.send({ topic: 'orders', messages: [...] });
  await transaction.sendOffsets({ consumerGroupId, topics: [...] });
  await transaction.commit();
} catch (err) {
  await transaction.abort();
}
```

**Giới hạn:**
- Chỉ trong **Kafka ecosystem** (producer, streams, connect)
- Không cover **external side effects** (gọi Payment API, ghi PostgreSQL)
- Overhead: latency, complexity, operational burden

### Exactly-Once End-to-End — Gần Như Không Khả Thi

```
Consumer đọc message
    → Gọi Stripe API (charge $100)     ← KHÔNG transactional với Kafka
    → Ghi PostgreSQL
    → Commit Kafka offset

Nếu crash sau charge nhưng trước commit → redeliver → double charge
```

| Layer | Exactly-Once Khả Thi? |
| ----- | --------------------- |
| Producer → Broker (Kafka) | Có (với idempotent + transactional) |
| Broker → Consumer (commit offset) | Có (với transactional consumer) |
| Consumer → External system | **Không** (trừ 2PC — Two-Phase Commit) |
| End-to-end business flow | **Thực tế: dùng idempotency** |

---

## Effective Exactly-Once Pattern

**Effective Exactly-Once (Đúng Một Lần Thực Tế)** — đạt **semantic exactly-once** bằng **At-least-once delivery + Idempotent processing**.

```
┌─────────────────────────────────────────────────────────────┐
│  EFFECTIVE EXACTLY-ONCE = At-least-once + Idempotency     │
│                                                             │
│  Message có thể đến N lần                                   │
│  Nhưng side effect chỉ xảy ra 1 lần                         │
└─────────────────────────────────────────────────────────────┘
```

### Pattern Triển Khai

```javascript
async function handleOrderEvent(event) {
  const idempotencyKey = event.orderId;  // hoặc eventId

  // 1. Check dedup — đã xử lý chưa?
  const existing = await db.idempotencyKeys.findOne({ key: idempotencyKey });
  if (existing?.status === 'completed') {
    return;  // Skip — đã xử lý, ack message
  }

  // 2. Reserve key (tránh race condition)
  await db.idempotencyKeys.upsert({
    key: idempotencyKey,
    status: 'processing',
    createdAt: new Date(),
  });

  try {
    // 3. Business logic — idempotent operations
    await chargePayment(event.orderId, event.amount);
    await updateInventory(event.items);

    // 4. Mark completed
    await db.idempotencyKeys.update(
      { key: idempotencyKey },
      { status: 'completed' }
    );
  } catch (err) {
    await db.idempotencyKeys.update(
      { key: idempotencyKey },
      { status: 'failed' }
    );
    throw err;  // Trigger retry
  }
}
```

### Kết Hợp Với Outbox Pattern

Cho **producer side** — đảm bảo "publish exactly once" khi ghi DB:

```
DB Transaction:
  INSERT INTO orders (...)
  INSERT INTO outbox (event_id, payload, status)

Outbox Relay (polling hoặc CDC):
  SELECT * FROM outbox WHERE status = 'pending'
  → Publish to Kafka
  → UPDATE outbox SET status = 'published'
```

Xem chi tiết: [02-architecture-patterns/4-outbox-inbox-pattern.md](../02-architecture-patterns/4-outbox-inbox-pattern.md)

### Checklist Effective Exactly-Once

```
□ At-least-once: manual ack / commit offset sau xử lý
□ Idempotency key: eventId, orderId, hoặc business key
□ Dedup store: Redis, DB table với TTL
□ Natural idempotency: UPSERT, SET thay vì INCREMENT
□ Outbox pattern cho producer (nếu cần atomic DB + publish)
□ DLQ cho message không xử lý được sau retries
□ Monitoring: duplicate detection rate
```

---

## So Sánh Theo Broker

| Broker | At-Least-Once | Exactly-Once Support | Ghi Chú |
| ------ | ------------- | -------------------- | ------- |
| **Kafka** | Manual commit offset | EOS (Streams, Connect, transactional) | EOS không cover external side effects |
| **RabbitMQ** | Manual ack | Không native EOS | Dùng idempotency + Outbox |
| **Amazon SQS** | Visibility timeout + delete | FIFO dedup (5 phút window) | Dedup by message group |
| **Redis Streams** | XACK sau xử lý | Không | Idempotency bắt buộc |
| **NATS JetStream** | Ack/Nak | Không | At-least-once default |

---

## Ma Trận Quyết Định

```
                    Cần exactly-once end-to-end?
                              │
              ┌───────────────┴───────────────┐
              │                               │
             YES                              NO
              │                               │
    Financial/Ledger?              At-least-once + Idempotency
              │                    (90% use cases)
    ┌─────────┴─────────┐
    │                   │
   YES                 NO
    │                   │
 Kafka EOS +         Effective exactly-once
 Idempotency         (đơn giản hơn, đủ dùng)
 cho external
```

| Use Case | Khuyến Nghị |
| -------- | ----------- |
| Order processing | At-least-once + idempotency |
| Payment, billing | Effective exactly-once hoặc Kafka EOS + idempotency |
| Metrics, logs | At-most-once (chấp nhận mất) |
| Inventory reservation | At-least-once + idempotency + saga compensation |
| Event sourcing | At-least-once + eventId dedup |

---

## Anti-Patterns

| Anti-Pattern | Vấn Đề | Cách Đúng |
| ------------ | ------ | --------- |
| Auto-ack trước xử lý | At-most-once — mất message khi crash | Manual ack sau success |
| Assume no duplicate | Double charge, oversell | Idempotent by default |
| Pursue true EOS everywhere | Over-engineering, operational cost | Effective exactly-once |
| Commit offset trước xử lý | Mất message nếu crash sau commit | Commit sau xử lý (at-least-once) |
| Không có idempotency key | Replay/DLQ gây duplicate | Luôn có business key |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Exactly-once có thực sự tồn tại không?

**Trả lời:** **Trong broker layer** (Kafka producer, Streams) — có, với transactional semantics. **End-to-end** qua external systems (DB, API) — **không** mà không dùng 2PC hoặc idempotency. Production dùng **effective exactly-once**: at-least-once + idempotent handlers.

### Câu 2: At-least-once + idempotency khác gì exactly-once?

**Trả lời:** **Kết quả business giống nhau** — mỗi logical operation chỉ có effect một lần. Khác ở implementation: at-least-once cho phép duplicate delivery nhưng consumer dedup; exactly-once cố gắng ngăn duplicate từ đầu (phức tạp, giới hạn scope).

### Câu 3: Khi nào cần Kafka EOS thay vì idempotency?

**Trả lời:** Khi processing **hoàn toàn trong Kafka** (Streams, Connect sink-to-Kafka) và cần guarantee chặt. Khi có **external side effects** — idempotency vẫn cần. Kafka EOS + idempotency có thể kết hợp cho critical flows.

### Câu 4: Commit offset trước hay sau xử lý?

**Trả lời:** **Sau xử lý** cho at-least-once. Commit trước = at-most-once (mất nếu crash sau commit). Trade-off: commit sau = có thể duplicate nếu crash sau xử lý nhưng trước commit — cần idempotency.

### Câu 5: SQS FIFO "exactly-once" có đúng không?

**Trả lời:** SQS FIFO có **deduplication** trong 5-phút window (by deduplication ID) — không phải true exactly-once. Vẫn cần idempotent consumer cho redelivery sau visibility timeout.

---

**Xem tiếp:** [2-retry-backoff.md](./2-retry-backoff.md) — Retry strategy và exponential backoff.
