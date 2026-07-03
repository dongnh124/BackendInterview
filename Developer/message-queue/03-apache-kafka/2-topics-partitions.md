# Topics & Partitions — Thiết Kế Topic Và Phân Vùng

> Thiết kế Topic (Chủ Đề), Partition (Phân Vùng), partition key (khóa phân vùng), retention (giữ lại), compaction (nén log), và tránh hot partition (phân vùng nóng) trong Apache Kafka.

## Mục Lục

1. [Topic Là Gì?](#topic-là-gì)
2. [Partition Design](#partition-design)
3. [Partition Key & Routing](#partition-key--routing)
4. [Chọn Số Partition](#chọn-số-partition)
5. [Hot Partition Problem](#hot-partition-problem)
6. [Retention & Compaction](#retention--compaction)
7. [Topic Naming Convention](#topic-naming-convention)
8. [Thay Đổi Partition Sau Deploy](#thay-đổi-partition-sau-deploy)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Topic Là Gì?

**Topic** là **category (danh mục)** hoặc **feed name (tên luồng)** cho stream of records — tương tự table trong database nhưng là append-only log.

```
Topic "order-events"
├── Partition 0: [e0, e1, e2, e3, ...]
├── Partition 1: [e0, e1, e2, ...]
└── Partition 2: [e0, e1, e2, e3, e4, ...]
```

| Khái Niệm | Mô Tả |
| -------- | ----- |
| **Topic** | Logical name — `orders`, `payments`, `user-signups` |
| **Partition** | Physical shard — parallelism + ordering boundary |
| **Offset** | Vị trí tuần tự trong partition (bắt đầu từ 0) |
| **Record** | Key + Value + Timestamp + Headers |

### Record Structure

```json
{
  "key": "order-12345",
  "value": { "eventType": "OrderCreated", "amount": 150000 },
  "timestamp": 1710000000000,
  "headers": { "correlation-id": "abc-uuid" }
}
```

---

## Partition Design

### Tại Sao Cần Nhiều Partition?

| Lý Do | Giải Thích |
| ----- | ---------- |
| **Parallelism (Song Song)** | Nhiều consumer đọc đồng thời |
| **Throughput (Thông Lượng)** | Ghi/đọc song song trên nhiều broker |
| **Scale** | Thêm partition để tăng capacity |

### Ordering Guarantee (Đảm Bảo Thứ Tự)

```
✅ Ordering TRONG partition:
   Partition 0: msg1 → msg2 → msg3 (FIFO)

❌ Ordering GIỮA partitions:
   P0: msgA          P1: msgX
   P0: msgB          P1: msgY
   → Không biết msgA hay msgX đến trước globally
```

> **Rule:** Cần ordering cho entity X → dùng **cùng partition key** cho mọi event của X.

---

## Partition Key & Routing

### Cách Broker Chọn Partition

```
Producer gửi record
        │
        ▼
   Có key không?
    /        \
  Có          Không
   │            │
   ▼            ▼
hash(key)    Round-robin
% numPartitions   hoặc sticky
   │            partition
   ▼
Partition ID
```

### Default Partitioner

```java
// Kafka default: murmur2 hash
partition = Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
```

```javascript
// Node.js — kafkajs
await producer.send({
  topic: 'orders',
  messages: [
    { key: 'user-42', value: JSON.stringify({ orderId: 'ORD-1' }) },
    { key: 'user-42', value: JSON.stringify({ orderId: 'ORD-2' }) },
    // Cùng key → cùng partition → ordering cho user-42
  ],
});
```

### Custom Partitioner

Dùng khi cần routing logic đặc biệt — ví dụ route VIP customer sang partition riêng:

```java
public class VipPartitioner implements Partitioner {
  @Override
  public int partition(String topic, Object key, byte[] keyBytes,
                       Object value, byte[] valueBytes, Cluster cluster) {
    OrderEvent event = parse(valueBytes);
    if (event.isVip()) return 0; // dedicated VIP partition
    return Utils.toPositive(Utils.murmur2(keyBytes)) % cluster.partitionCountForTopic(topic);
  }
}
```

---

## Chọn Số Partition

### Công Thức Ước Lượng

```
partitions = max(
  desired_throughput / producer_throughput_per_partition,
  desired_throughput / consumer_throughput_per_partition
)
```

**Ví dụ:**

```
Target: 100 MB/s write
Per-partition capacity: ~10 MB/s
→ Cần ít nhất 10 partitions
```

### Guidelines

| Yếu Tố | Khuyến Nghị |
| ------ | ----------- |
| **Consumer parallelism** | partitions ≥ số consumer instances tối đa |
| **Ordering granularity** | Nhiều partition = ordering chỉ trong key group |
| **Over-partitioning** | Quá nhiều partition → metadata overhead, rebalance chậm |
| **Under-partitioning** | Quá ít → không scale consumer, hot partition |

| Quy Mô | Partitions/Topic | Ghi Chú |
| ------ | ---------------- | ------- |
| Dev/test | 1–3 | Đủ test logic |
| Small prod | 6–12 | Vài consumer instances |
| Medium prod | 12–48 | Scale theo throughput |
| Large prod | 48–200+ | Cần capacity planning |

> **Lưu ý:** Tăng partition **không** redistribute message cũ — chỉ ảnh hưởng message mới.

---

## Hot Partition Problem

### Nguyên Nhân

**Hot partition (phân vùng nóng)** xảy ra khi partition key **không đều** — một key hoặc nhóm key chiếm phần lớn traffic.

```
Ví dụ sai:
  key = "GLOBAL" cho mọi event → tất cả vào 1 partition

Ví dụ skew:
  key = country_code
  90% traffic từ "VN" → partition hash("VN") quá tải
```

### Giải Pháp

| Giải Pháp | Mô Tả |
| --------- | ----- |
| **Key redesign** | Dùng `userId`, `orderId` thay vì key tập trung |
| **Salted key** | `key = userId + "-" + random(0..N)` — trade-off: mất ordering |
| **Composite key** | `merchantId-orderId` phân tán theo merchant |
| **Tách topic** | VIP traffic sang topic riêng |
| **Monitor** | Metric bytes-in per partition |

```
Trước: key = tenantId (1 tenant lớn = 80% traffic)
Sau:  key = tenantId + "-" + shardId
      shardId = hash(entityId) % 10
      → ordering trong (tenant, entity) vẫn giữ
```

---

## Retention & Compaction

### Delete Policy (Mặc Định)

Message bị xóa sau khi hết retention:

```properties
retention.ms=604800000        # 7 ngày
retention.bytes=-1            # unlimited size
```

### Log Compaction (Nén Log)

**Compacted topic** giữ **latest value per key** — phù hợp changelog, CDC state:

```
Trước compaction:
  offset 0: key=user-1, value={name: "Alice"}
  offset 1: key=user-2, value={name: "Bob"}
  offset 2: key=user-1, value={name: "Alice Updated"}

Sau compaction:
  offset 0: (tombstone hoặc xóa)
  offset 1: key=user-2, value={name: "Bob"}
  offset 2: key=user-1, value={name: "Alice Updated"}
```

```bash
kafka-topics.sh --create --topic user-profiles \
  --config cleanup.policy=compact \
  --bootstrap-server localhost:9092
```

| cleanup.policy | Use Case |
| -------------- | -------- |
| `delete` | Event stream, logs, metrics |
| `compact` | KTable state, config, user profile changelog |
| `compact,delete` | Compact + time-based delete |

### Tombstone (Đánh Dấu Xóa)

Record với `value=null` là **tombstone** — báo compaction xóa key đó.

---

## Topic Naming Convention

### Best Practices

```
<domain>.<entity>.<event-type>.<version>

Ví dụ:
  commerce.order.created.v1
  payment.transaction.completed.v1
  analytics.page-view.raw.v1
```

| Pattern | Mô Tả |
| ------- | ----- |
| **Environment prefix** | `prod.`, `staging.` — hoặc tách cluster |
| **Version suffix** | `.v1`, `.v2` — schema evolution |
| **Internal vs external** | `__` prefix cho internal topics (Kafka convention) |

### Anti-Patterns

```
❌ topic-per-consumer     → orders-for-email, orders-for-analytics
✅ topic-per-event        → order.created + nhiều consumer group

❌ quá nhiều topic nhỏ    → metadata overhead
❌ tên không có namespace → "events", "data"
```

---

## Thay Đổi Partition Sau Deploy

### Tăng Partitions

```bash
kafka-topics.sh --alter --topic orders \
  --partitions 12 \
  --bootstrap-server localhost:9092
```

**Hậu quả:**

- Message **mới** phân phối theo partition mới
- Message **cũ** vẫn ở partition cũ
- **Key ordering có thể bị phá** — cùng key có thể ở partition khác trước/sau alter
- Consumer group cần **rebalance**

### Giảm Partitions

Kafka **không hỗ trợ** giảm partition trực tiếp — phải tạo topic mới và migrate.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Làm sao đảm bảo ordering cho order events?

**Gợi ý trả lời:** Dùng **partition key = orderId** — mọi event của cùng order vào cùng partition, FIFO trong partition. Không cần global ordering toàn topic.

### Câu 2: 6 partitions, 10 consumers — chuyện gì xảy ra?

**Gợi ý trả lời:** Tối đa **6 consumer active** — mỗi partition gán 1 consumer. **4 consumer idle** — không đọc partition nào. Muốn scale hơn → tăng partitions.

### Câu 3: Compacted topic khác delete topic thế nào?

**Gợi ý trả lời:** **Delete:** xóa message theo thời gian/kích thước. **Compact:** giữ latest record per key — phù hợp state store, không phù hợp event history đầy đủ.

### Câu 4: Tăng partition có ảnh hưởng message cũ không?

**Gợi ý trả lời:** **Không di chuyển** message cũ. Chỉ message mới theo hash key mới. Cùng key trước/sau alter có thể ở partition khác → **ordering có thể break** — plan trước khi production.

### Câu 5: Khi nào dùng message không có key?

**Gợi ý trả lời:** Khi **không cần ordering** — metrics, logs, events độc lập. Broker dùng round-robin/sticky partitioner để phân phối đều.

---

**Xem tiếp:** [3-consumer-groups.md](./3-consumer-groups.md) — consumer groups và rebalancing.
