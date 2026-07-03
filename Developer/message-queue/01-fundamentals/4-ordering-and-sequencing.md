# Message Ordering & Sequencing — Thứ Tự Tin Nhắn

> Message Ordering (Thứ Tự Tin Nhắn) ảnh hưởng trực tiếp đến tính đúng đắn của business logic. Hiểu global ordering vs partial ordering, partition key, FIFO queue, và trade-offs khi scale.

## Mục Lục

1. [Tại Sao Ordering Quan Trọng](#tại-sao-ordering-quan-trọng)
2. [Các Mức Ordering](#các-mức-ordering)
3. [Partition Key & Key-Based Routing](#partition-key--key-based-routing)
4. [FIFO Queue](#fifo-queue)
5. [Ordering Trên Các Broker](#ordering-trên-các-broker)
6. [Ordering vs Parallelism Trade-off](#ordering-vs-parallelism-trade-off)
7. [Anti-patterns & Giải Pháp](#anti-patterns--giải-pháp)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Ordering Quan Trọng

Khi message đến **không đúng thứ tự**, business logic có thể sai:

```
❌ Thứ tự sai:
   Event 1: AccountBalance = $100
   Event 2: Withdraw $80  → Balance = $20
   Event 3: Deposit $50   → Balance = $70

   Nếu Event 3 đến trước Event 2:
   Deposit $50 trước → $150 → Withdraw $80 → $70 (SAI logic nếu overdraft check)
```

```
✅ Cần ordering cho:
   - Cùng user/account: thay đổi trạng thái tuần tự
   - Cùng order: Created → Paid → Shipped → Delivered
   - Cùng partition key: events của 1 entity
```

---

## Các Mức Ordering

### Global Ordering (Thứ Tự Toàn Cục)

Mọi message trong topic/queue được xử lý **đúng thứ tự publish**.

```
Producer: m1 → m2 → m3 → m4
Consumer: m1 → m2 → m3 → m4  (luôn đúng thứ tự)
```

**Yêu cầu:** Chỉ **1 partition** hoặc **1 consumer** — bottleneck nghiêm trọng.

**Khi nào cần:** Hiếm — audit log tuần tự tuyệt đối, single-threaded state machine.

### Partial Ordering (Thứ Tự Một Phần)

Message **cùng key** giữ thứ tự; message **khác key** có thể xen kẽ.

```
Key=user-123:  [login, update-profile, logout]  → đúng thứ tự
Key=user-456:  [login, purchase]                → đúng thứ tự
Hai key có thể interleave ( xen kẽ ) trên consumer
```

**Đây là mô hình production phổ biến nhất** — balance giữa ordering và parallelism.

### No Ordering (Không Đảm Bảo Thứ Tự)

Message xử lý theo thứ tự available — acceptable cho independent events.

```
Metrics, logs, notifications không phụ thuộc thứ tự
```

---

## Partition Key & Key-Based Routing

**Partition Key (Khóa Phân Vùng)** quyết định message đi vào partition nào — message cùng key → cùng partition → **ordering guaranteed trong partition**.

### Kafka

```javascript
await producer.send({
  topic: 'orders',
  messages: [{
    key: orderId,        // Cùng orderId → cùng partition
    value: JSON.stringify(event),
  }],
});
```

```
Topic: orders (4 partitions)

orderId=100 → hash(100) % 4 = Partition 2
orderId=200 → hash(200) % 4 = Partition 1
orderId=100 → Partition 2 (cùng partition → FIFO)
```

### Chọn Partition Key

| Entity | Key | Lý Do |
| ------ | --- | ----- |
| Order lifecycle | `orderId` | Events cùng order cần tuần tự |
| User actions | `userId` | Profile updates tuần tự per user |
| Bank account | `accountId` | Balance changes tuần tự |
| Không cần ordering | `null` (round-robin) | Max parallelism |

### Hot Partition (Phân Vùng Nóng)

```
⚠️ Nếu 80% traffic dùng cùng key → 1 partition overload
   Ví dụ: key = "GLOBAL" cho tất cả events

Giải pháp:
- Chia key finer-grained (userId thay vì tenantId)
- Tăng partition count
- Accept no ordering cho hot key
```

---

## FIFO Queue

**FIFO (First In, First Out — Vào Trước Ra Trước)** — message xử lý đúng thứ tự enqueue.

### Amazon SQS FIFO

```
Queue: orders.fifo
MessageGroupId: order-123    → ordering trong group
MessageDeduplicationId: ...  → chống duplicate trong 5 phút
```

| Tính Năng | Standard SQS | FIFO SQS |
| --------- | ------------ | -------- |
| Ordering | Không đảm bảo | FIFO per MessageGroupId |
| Throughput | Không giới hạn | 300 msg/s (hoặc 3000 với batching) |
| Duplicate | Có thể | Deduplication ID |
| Use case | High throughput | Order processing |

### RabbitMQ — Single Active Consumer

```javascript
// Đảm bảo chỉ 1 consumer active — ordering trong queue
await channel.assertQueue('ordered-tasks', {
  arguments: { 'x-single-active-consumer': true },
});
```

### Kafka — Single Partition Topic

```
Topic với 1 partition = global FIFO
→ Throughput giới hạn ~ vài chục nghìn msg/s
→ Chỉ dùng khi thực sự cần global order
```

---

## Ordering Trên Các Broker

| Broker | Ordering Mechanism | Scope |
| ------ | ------------------ | ----- |
| **Kafka** | Partition + key | Per partition (partial với key) |
| **RabbitMQ** | Single queue, single consumer | Per queue |
| **SQS FIFO** | MessageGroupId | Per group |
| **SQS Standard** | Không | - |
| **Redis Streams** | Stream ID (timestamp-based) | Per stream |
| **Pulsar** | Key_shared / Key_hash subscription | Per key |

### Kafka Consumer Group & Ordering

```
⚠️ QUAN TRỌNG:
   Ordering chỉ đảm bảo TRONG partition.
   1 partition = tối đa 1 consumer trong group xử lý.

Topic 4 partitions + Consumer group 4 consumers = OK (1:1)
Topic 4 partitions + Consumer group 8 consumers = 4 idle (waste)
Topic 4 partitions + Consumer group 2 consumers = mỗi consumer 2 partitions
```

### Rebalance & Ordering Risk

```
Consumer A đang xử lý partition 2
→ Rebalance xảy ra
→ Partition 2 chuyển sang Consumer B
→ Message chưa commit offset có thể redeliver
→ Cần idempotency + có thể thấy "out of order" tạm thời
```

---

## Ordering vs Parallelism Trade-off

```
                    Parallelism (Song Song)
                           ▲
                           │
         No ordering       │      Partial ordering
         (max parallel)    │      (key-based)
                           │
                           │
    ───────────────────────┼──────────────────────► Ordering
                           │
                           │      Global ordering
                           │      (single partition)
                           │
```

| Mức Ordering | Parallelism | Throughput | Use Case |
| ------------ | ----------- | ---------- | -------- |
| None | Cao nhất | Cao nhất | Logs, metrics |
| Per-key | Cao | Cao | Orders, users |
| Global | 1 consumer | Thấp | Audit trail tuần tự |

**Quy tắc thiết kế:** Chọn **lowest ordering scope** đủ cho business — đừng over-order.

---

## Anti-patterns & Giải Pháp

### Anti-pattern 1: Global Order Cho Mọi Thứ

```
❌ Topic "all-events" với 1 partition
   → Bottleneck, không scale

✅ Topic per domain, partition by entity ID
```

### Anti-pattern 2: Nhiều Consumer Cùng Partition

```
❌ 2 consumers cùng đọc 1 queue không có competing consumer protocol
   → Duplicate + out of order

✅ Competing consumers qua broker protocol (1 message → 1 consumer)
```

### Anti-pattern 3: Timestamp Làm Ordering

```
❌ Sắp xếp theo event.timestamp
   → Clock skew giữa producers
   → Network delay làm event đến sai thứ tự

✅ Dùng sequence number per entity hoặc partition key
```

### Giải Pháp: Sequence Number

```json
{
  "orderId": "ORD-001",
  "sequence": 3,
  "eventType": "OrderShipped",
  "previousSequence": 2
}
```

Consumer reject hoặc buffer nếu `sequence != expected + 1`.

### Giải Pháp: Version Vector / Optimistic Locking

```sql
UPDATE orders SET status = 'SHIPPED', version = version + 1
WHERE id = 'ORD-001' AND version = 2;
-- Nếu 0 rows affected → event out of order hoặc duplicate
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka đảm bảo ordering thế nào?

**Đáp án mẫu:** Ordering đảm bảo **trong một partition**. Message cùng key → cùng partition → FIFO. Không có ordering **giữa** các partition. Muốn partial ordering: dùng partition key. Muốn global ordering: 1 partition (trade-off throughput).

### Câu 2: Tăng partition count có ảnh hưởng ordering không?

**Đáp án mẫu:** Có — nhiều partition hơn = parallelism cao hơn nhưng **scope ordering nhỏ hơn** (chỉ trong partition). Khi tăng partition, **key routing thay đổi** — không reorder message cũ nhưng message mới có thể đi partition khác nếu thuật toán hash thay đổi (cần cẩn thận khi thay đổi partition count).

### Câu 3: Làm sao xử lý event out-of-order?

**Đáp án mẫu:** (1) **Partition key** để prevent. (2) **Sequence number** + buffer/reject. (3) **Idempotent upsert** với version check. (4) **CQRS/Event Sourcing** — replay theo đúng thứ tự từ event store. Chọn theo tolerance của business.

### Câu 4: SQS FIFO vs Standard — khi nào dùng?

**Đáp án mẫu:** **FIFO** khi cần ordering per MessageGroupId và deduplication — order processing, workflow steps. **Standard** khi cần throughput cao, ordering không quan trọng — background jobs, notifications. FIFO giới hạn 300 msg/s per queue.

---

**Xem tiếp:** [5-backpressure-flow-control.md](./5-backpressure-flow-control.md) — xử lý khi producer vượt khả năng consumer.
