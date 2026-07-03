# Delivery Guarantees — Đảm Bảo Giao Hàng Tin Nhắn

> Delivery Semantics (Ngữ Nghĩa Giao Hàng) — At-most-once, At-least-once, Exactly-once — là chủ đề **được hỏi nhiều nhất** trong phỏng vấn messaging. Hiểu trade-offs và cách đạt "effective exactly-once" trong production.

## Mục Lục

1. [Tổng Quan Ba Mức Đảm Bảo](#tổng-quan-ba-mức-đảm-bảo)
2. [At-Most-Once](#at-most-once)
3. [At-Least-Once](#at-least-once)
4. [Exactly-Once](#exactly-once)
5. [Ma Trận Trade-offs](#ma-trận-trade-offs)
6. [Effective Exactly-Once](#effective-exactly-once)
7. [Cấu Hình Thực Tế](#cấu-hình-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Ba Mức Đảm Bảo

```
                    ┌─────────────────────────────────────┐
                    │     DELIVERY GUARANTEE SPECTRUM      │
                    └─────────────────────────────────────┘

   At-most-once          At-least-once           Exactly-once
   (Tối đa 1 lần)       (Ít nhất 1 lần)         (Đúng 1 lần)
         │                     │                       │
    Có thể MẤT          Có thể TRÙNG            Không mất
    Không trùng         Không mất (lý tưởng)    Không trùng
         │                     │                       │
    Đơn giản nhất         Production default      Khó nhất
```

| Guarantee | Message Loss | Duplicate | Độ Phức Tạp | Production |
| --------- | ------------ | --------- | ----------- | ---------- |
| **At-most-once** | Có thể | Không | Thấp | Metrics, logs không critical |
| **At-least-once** | Không* | Có thể | Trung bình | **Phổ biến nhất** |
| **Exactly-once** | Không | Không | Cao | Financial, billing (hiếm) |

\* Với điều kiện broker replication và persistence đúng cách.

---

## At-Most-Once

**At-most-once (Tối Đa Một Lần)** — message được deliver **0 hoặc 1 lần**, không retry khi lỗi.

### Cơ Chế

```
Producer ──send──► Broker ──deliver──► Consumer
                         │
                    auto-ack ngay
                    (trước khi xử lý xong)
```

1. Producer gửi, không chờ confirm
2. Consumer **auto-ack** ngay khi nhận message
3. Nếu consumer crash sau ack nhưng trước xử lý xong → **message mất**

### Khi Nào Dùng

| Use Case | Lý Do |
| -------- | ----- |
| Metrics, telemetry | Mất vài data point chấp nhận được |
| Real-time sensor data | Data mới thay thế data cũ |
| Log aggregation (non-critical) | Volume cao, loss acceptable |

### Ví Dụ Cấu Hình

```javascript
// RabbitMQ — auto-ack (at-most-once semantics)
channel.consume('metrics', handler, { noAck: true });

// Kafka — fire and forget
producer.send({ acks: 0 }); // Không chờ broker ack
```

### Rủi Ro

- **Silent data loss** — khó phát hiện
- Không phù hợp order, payment, inventory

---

## At-Least-Once

**At-least-once (Ít Nhất Một Lần)** — message được deliver **ít nhất 1 lần**, có thể **duplicate (trùng lặp)**.

### Cơ Chế

```
Producer ──send──► Broker (persist) ──deliver──► Consumer
       ◄── ack ──                    ◄── manual ack sau xử lý

Nếu consumer crash trước ack → message redeliver → DUPLICATE
```

**Ba điểm có thể duplicate:**

1. **Producer retry:** Gửi lại khi không nhận broker ack
2. **Broker redeliver:** Consumer không ack kịp (timeout, crash)
3. **Consumer retry:** Xử lý xong, crash trước commit offset

### Giải Pháp Bắt Buộc: Idempotency

```javascript
async function handleOrderCreated(event) {
  const { orderId } = event;

  // Kiểm tra đã xử lý chưa
  const processed = await redis.get(`processed:${orderId}`);
  if (processed) return; // Skip duplicate

  await chargePayment(orderId);
  await redis.set(`processed:${orderId}`, '1', 'EX', 86400);
}
```

### Khi Nào Dùng

**Mặc định cho hầu hết production systems:**

- Order processing
- Payment notification
- Email/SMS delivery
- Inventory update

### Ví Dụ Cấu Hình

```javascript
// RabbitMQ — manual ack
channel.consume('orders', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg); // Ack SAU khi xử lý xong
  } catch (err) {
    channel.nack(msg, false, true); // Requeue để retry
  }
}, { noAck: false });

// Kafka — manual commit offset
await consumer.run({
  eachMessage: async ({ message }) => {
    await process(message);
    // Offset commit sau khi xử lý (enable.auto.commit=false)
  },
});
```

---

## Exactly-Once

**Exactly-once (Đúng Một Lần)** — mỗi message được xử lý **đúng 1 lần**, không mất, không trùng.

### Tại Sao Khó?

Exactly-once yêu cầu **atomicity (tính nguyên tử)** xuyên suốt:

```
Consume message + Side effect (DB write) + Ack offset
         ↑_________________ phải atomic _________________↑
```

Trong distributed system, network partition và crash làm điều này **gần như không thể** ở mức end-to-end thuần túy.

### Hai Loại Exactly-Once

| Loại | Phạm Vi | Ví Dụ |
| ---- | ------- | ----- |
| **Broker-level EOS** | Trong phạm vi broker | Kafka transactional producer + idempotent producer |
| **End-to-end EOS** | Producer → Consumer → DB | Outbox pattern + idempotent consumer |

### Kafka Exactly-Once Semantics (EOS)

Kafka hỗ trợ EOS trong phạm vi **read-process-write** với Kafka Streams:

```
enable.idempotence=true          → Producer không duplicate vào topic
transactional.id=order-producer  → Atomic write nhiều partition
isolation.level=read_committed   → Consumer chỉ đọc committed messages
```

**Giới hạn:** Chỉ áp dụng khi **cả input và output đều là Kafka topic** — không cover external DB trực tiếp.

### Distributed Transactions — Thực Tế

```
❌ Không khả thi: 2PC (Two-Phase Commit) across Kafka + PostgreSQL
✅ Khả thi: Outbox Pattern + at-least-once + idempotency
```

---

## Ma Trận Trade-offs

| Tiêu Chí | At-most-once | At-least-once | Exactly-once |
| -------- | ------------ | ------------- | ------------ |
| **Data loss risk** | Cao | Thấp | Không |
| **Duplicate risk** | Không | Cao | Không |
| **Throughput** | Cao nhất | Cao | Thấp hơn (~20-30%) |
| **Latency** | Thấp nhất | Trung bình | Cao hơn |
| **Consumer complexity** | Thấp | Cần idempotency | Cao |
| **Debugging** | Khó (silent loss) | Trung bình | Dễ hơn về logic |
| **Phỏng vấn** | Hiếm | **Luôn hỏi** | Senior+ |

---

## Effective Exactly-Once

Trong production, **"effective exactly-once"** là pattern thực tế thay vì true EOS:

```
At-least-once delivery
    +
Idempotent consumer (dedup key / natural idempotency)
    +
Transactional Outbox (atomic DB + message publish)
    =
Effective exactly-once (đủ tốt cho 99% use cases)
```

### Pattern 1: Natural Idempotency (Tính Bất Biến Tự Nhiên)

```sql
-- UPSERT — chạy 2 lần cùng kết quả
INSERT INTO inventory (sku, quantity)
VALUES ('SKU-001', 10)
ON CONFLICT (sku) DO UPDATE SET quantity = 10;
```

### Pattern 2: Deduplication Store

```
Message ID → Redis/DB lookup → skip if exists → process → mark processed
```

### Pattern 3: Outbox Pattern

```
BEGIN TRANSACTION;
  INSERT INTO orders (...);
  INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', ...);
COMMIT;

-- Separate poller publish từ outbox → broker
```

Xem chi tiết tại `02-architecture-patterns/4-outbox-inbox-pattern.md`.

---

## Cấu Hình Thực Tế

### RabbitMQ — Reliable Delivery Checklist

| Bước | Cấu Hình |
| ---- | -------- |
| Producer | `publisher confirms` enabled |
| Queue | `durable: true` |
| Message | `deliveryMode: 2` (persistent) |
| Consumer | Manual ack, ack sau xử lý |
| HA | Mirrored queue hoặc quorum queue |

### Kafka — Reliable Delivery Checklist

| Bước | Cấu Hình |
| ---- | -------- |
| Producer | `acks=all`, `enable.idempotence=true` |
| Broker | `min.insync.replicas >= 2` |
| Consumer | `enable.auto.commit=false`, commit sau xử lý |
| Replication | `replication.factor >= 3` |

### Decision Flow

```
                    ┌─────────────────────────┐
                    │ Message có critical?    │
                    │ (money, order, audit) │
                    └───────────┬─────────────┘
                          Có    │    Không
                    ┌───────────┴───────────┐
                    ▼                       ▼
            At-least-once +          At-most-once
            idempotency              (metrics, logs)
                    │
                    ▼
            Cần true EOS?
            (billing, ledger)
                    │
              Có ───┴─── Không
              ▼           ▼
        Kafka EOS +    At-least-once
        Outbox         + idempotency
        (rất hiếm)     (99% cases)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Production nên chọn delivery guarantee nào?

**Đáp án mẫu:** **At-least-once + idempotent consumer** là mặc định. Đảm bảo không mất data quan trọng, chấp nhận duplicate và xử lý bằng dedup key. True exactly-once chỉ khi regulatory requirement (tài chính) và chấp nhận complexity/performance cost.

### Câu 2: Message bị duplicate — xảy ra ở đâu và xử lý thế nào?

**Đáp án mẫu:** Duplicate xảy ra khi: (1) producer retry, (2) consumer crash trước ack, (3) rebalance giữa chừng xử lý. Xử lý: **idempotency key** trong message, dedup table/Redis, hoặc natural idempotency (UPSERT). Luôn thiết kế consumer **idempotent by default**.

### Câu 3: Kafka exactly-once hoạt động thế nào?

**Đáp án mẫu:** Kết hợp: **Idempotent Producer** (PID + sequence number chống duplicate write), **Transactions** (atomic multi-partition write), **Read Committed** isolation (consumer chỉ đọc committed). Giới hạn: trong phạm vi Kafka — external sink cần Outbox hoặc idempotency riêng.

### Câu 4: Auto-ack vs Manual-ack?

**Đáp án mẫu:** **Auto-ack:** broker xóa message ngay khi deliver → at-most-once, risk mất data. **Manual-ack:** consumer ack sau xử lý thành công → at-least-once, an toàn hơn. Production luôn dùng manual-ack trừ metrics/logs không critical.

### Câu 5: Làm sao đảm bảo không mất message từ producer?

**Đáp án mẫu:** (1) **Publisher confirms** — chờ broker ack trước khi coi là gửi thành công. (2) **Persistent messages** — ghi disk. (3) **Replication** — acks=all, min.insync.replicas. (4) **Outbox pattern** — atomic với DB transaction. Retry producer khi không nhận confirm.

---

**Xem tiếp:** [4-ordering-and-sequencing.md](./4-ordering-and-sequencing.md) — thứ tự message và partition key.
