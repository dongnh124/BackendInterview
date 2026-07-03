# Partitioning Strategies — Chiến Lược Phân Vùng

> Partition (Phân Vùng) là đơn vị parallelism trong Kafka và ảnh hưởng trực tiếp tới throughput, ordering, và khả năng scale consumer. Thiết kế partition key sai là nguyên nhân phổ biến của **hot partition (Phân Vùng Nóng)** — một partition nhận 80% traffic.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Partition Là Gì và Tại Sao Quan Trọng](#partition-là-gì-và-tại-sao-quan-trọng)
3. [Partition Key Design](#partition-key-design)
4. [Chọn Số Partition](#chọn-số-partition)
5. [Hot Partition — Phát Hiện & Xử Lý](#hot-partition--phát-hiện--xử-lý)
6. [Rebalancing Impact](#rebalancing-impact)
7. [Partitioning Trong RabbitMQ & Cloud](#partitioning-trong-rabbitmq--cloud)
8. [Migration & Re-partitioning](#migration--re-partitioning)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│           PARTITIONING — GOLDEN RULES                              │
├─────────────────────────────────────────────────────────────────┤
│  1. Key = entity cần ordering (orderId, userId, accountId)      │
│  2. Tránh key skew — không dùng constant key hoặc low-cardinality │
│  3. Partition count ≥ max consumer instances bạn cần scale       │
│  4. Tăng partition KHÔNG fix hot partition do bad key design     │
│  5. Plan partition count trước — giảm partition rất khó            │
└─────────────────────────────────────────────────────────────────┘
```

| Yếu Tố | Ảnh Hưởng |
| ------ | --------- |
| **Partition count** | Max parallelism, throughput ceiling |
| **Partition key** | Ordering scope, load distribution |
| **Key cardinality** | Độ đều của load across partitions |
| **Hash function** | Kafka: `murmur2(key) % numPartitions` |

---

## Partition Là Gì và Tại Sao Quan Trọng

Trong Kafka, mỗi **topic** chia thành N **partitions** — mỗi partition là append-only log độc lập, nằm trên một broker leader.

```
Topic: orders (6 partitions)

  P0 ──► Broker-1 (leader)     Consumer-1 đọc P0, P1
  P1 ──► Broker-1 (leader)
  P2 ──► Broker-2 (leader)       Consumer-2 đọc P2, P3
  P3 ──► Broker-2 (leader)
  P4 ──► Broker-3 (leader)       Consumer-3 đọc P4, P5
  P5 ──► Broker-3 (leader)
```

### Ba Vai Trò Của Partition

1. **Parallelism (Song Song)** — mỗi partition consume bởi tối đa 1 consumer trong group
2. **Ordering (Thứ Tự)** — ordering đảm bảo **trong partition**, không cross-partition
3. **Retention & Storage (Lưu Trữ)** — mỗi partition là file log riêng trên disk

> **Đọc thêm:** [03-apache-kafka/2-topics-partitions.md](../03-apache-kafka/2-topics-partitions.md)

---

## Partition Key Design

### Nguyên Tắc Chọn Key

| Use Case | Key Gợi Ý | Lý Do |
| -------- | ---------- | ----- |
| Order lifecycle | `orderId` | Tất cả events của 1 order cùng partition → ordering |
| User activity | `userId` | Per-user ordering |
| Account balance | `accountId` | Tránh race condition khi update balance |
| Log aggregation | `null` (round-robin) | Không cần ordering, phân bổ đều |
| Multi-tenant SaaS | `tenantId` | Isolate tenant, có thể hot nếu 1 tenant lớn |

### Key Routing

```javascript
// Kafka — key quyết định partition
await producer.send({
  topic: 'orders',
  messages: [{
    key: order.userId,           // murmur2 hash → partition
    value: JSON.stringify(order),
  }],
});

// Không có key → round-robin (sticky partitioner trong Kafka 2.4+)
await producer.send({
  topic: 'logs',
  messages: [{ value: logLine }],  // no key
});
```

### Composite Key (Khóa Tổ Hợp)

Khi cần ordering theo nhiều dimension:

```javascript
// Ordering per user per day — tránh hot user block toàn bộ partition
const key = `${userId}:${dateBucket}`;  // dateBucket = YYYY-MM-DD

// Salt key — phá hot partition cho user lớn
const salt = hash(userId) % 10;
const key = `${userId}:${salt}`;  // Trade-off: mất ordering cross-salt
```

### Custom Partitioner

```java
// Kafka custom partitioner — route VIP users riêng
public class VipPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String userId = (String) key;
        if (vipUsers.contains(userId)) {
            return 0;  // Dedicated partition cho VIP (cẩn thận hot!)
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}
```

---

## Chọn Số Partition

### Công Thức Ước Tính

```
partitions = max(
  desired_throughput / throughput_per_partition,
  max_consumer_instances
)
```

**Ví dụ:**

```
Target: 100 MB/s ingest
Per partition: ~10 MB/s (benchmark trên hardware của bạn)
→ Cần ít nhất 10 partitions cho throughput

Max consumer instances dự kiến: 20
→ partitions ≥ 20

Chọn: 24 partitions (làm tròn, để headroom)
```

### Bảng Tham Khảo

| Throughput Mục Tiêu | Partition Gợi Ý | Ghi Chú |
| ------------------ | --------------- | ------- |
| < 10 MB/s | 3–6 | Dev/staging |
| 10–100 MB/s | 12–48 | Production nhỏ |
| 100 MB/s – 1 GB/s | 48–200 | Cần benchmark |
| > 1 GB/s | 200+ | Chuyên gia, multi-cluster |

### Trade-offs Khi Tăng Partition

| Tăng Partition | Ưu | Nhược |
| -------------- | -- | ----- |
| ⬆ Parallelism | Scale consumer tốt hơn | Nhiều file log, metadata |
| ⬆ Throughput ceiling | Nhiều I/O song song | Rebalance chậm hơn |
| | | Consumer memory tăng |
| | | End-to-end latency có thể tăng (more files) |

> **Khuyến nghị:** Bắt đầu conservative, tăng khi benchmark chứng minh cần. Giảm partition count **không được hỗ trợ** trực tiếp trong Kafka.

---

## Hot Partition — Phát Hiện & Xử Lý

**Hot partition** xảy ra khi một partition nhận disproportionate traffic — thường do **skewed key distribution (Phân Phối Key Lệch)**.

### Dấu Hiệu

```
Partition message rate:
  P0: ████████████████████ 45,000 msg/s  ← HOT
  P1: ████ 8,000 msg/s
  P2: ████ 7,500 msg/s
  P3: ████ 8,200 msg/s
  ...

Consumer lag:
  Consumer-1 (P0): lag = 500,000  ← Stuck
  Consumer-2 (P1): lag = 0
```

### Metrics Monitor

| Metric | Nguồn | Ngưỡng |
| ------ | ----- | ------ |
| `MessagesInPerSec` per partition | JMX / Prometheus | > 3x average |
| Consumer lag per partition | Burrow / Kafka Exporter | Partition lag >> others |
| Disk usage per partition | Broker metrics | Uneven growth |

### Chiến Lược Fix

| Strategy | Mô Tả | Trade-off |
| -------- | ----- | --------- |
| **Salt key** | `key = userId:salt` phân tán load | Mất ordering cross-salt |
| **Composite key** | `userId:hour` thay vì chỉ `userId` | Ordering scope nhỏ hơn |
| **Dedicated topic** | Tách VIP/large tenant sang topic riêng | Operational complexity |
| **Custom partitioner** | Route logic đặc biệt | Khó maintain |
| **Tăng partition** | Chỉ giúp nếu key có cardinality cao | **Không fix** constant key |

### Ví Dụ: E-commerce Flash Sale

```
Vấn đề: 90% orders từ 1 sellerId trong flash sale
Key hiện tại: sellerId → 1 partition nhận hết

Fix 1: key = orderId (unique) → phân bổ đều, mất per-seller ordering
Fix 2: key = sellerId:orderId → ordering per order, phân bổ đều
Fix 3: topic riêng cho flash-sale events
```

---

## Rebalancing Impact

Khi consumer join/leave group hoặc partition count thay đổi → **rebalance (Cân Bằng Lại)**.

```
Trước rebalance:
  C1: P0, P1, P2
  C2: P3, P4, P5

C3 join → rebalance:
  [STOP THE WORLD — pause consume]
  C1: P0, P1
  C2: P2, P3
  C3: P4, P5
```

### Giảm Rebalance Impact

| Technique | Mô Tả |
| --------- | ----- |
| **Cooperative sticky assignor** | Chỉ revoke partitions cần move |
| **Static membership** | `group.instance.id` — restart không trigger rebalance ngay |
| **Scale gradually** | Thêm 1-2 consumer, không nhảy từ 3 → 20 |
| **session.timeout.ms tune** | Tránh false positive rebalance |

```properties
# Cooperative rebalancing (Kafka 2.4+)
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Static membership
group.instance.id=consumer-pod-1
```

> Xem [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md)

---

## Partitioning Trong RabbitMQ & Cloud

### RabbitMQ — Consistent Hash Exchange

RabbitMQ không có partition native như Kafka. Dùng **Consistent Hash Exchange** để route message cùng key vào cùng queue:

```
                    ┌──► Queue-1 (hash % 3 = 0)
Publisher ──► CHX ──┼──► Queue-2 (hash % 3 = 1)
  routing_key       └──► Queue-3 (hash % 3 = 2)
  = userId
```

```javascript
channel.assertExchange('orders-hash', 'x-consistent-hash', { durable: true });
channel.assertQueue('orders-q-1', { durable: true });
channel.bindQueue('orders-q-1', 'orders-hash', '1');  // weight
```

### Amazon SQS FIFO — Message Group ID

```javascript
// SQS FIFO — ordering trong message group
await sqs.sendMessage({
  QueueUrl: fifoQueueUrl,
  MessageBody: JSON.stringify(order),
  MessageGroupId: order.userId,      // Ordering scope
  MessageDeduplicationId: order.id,
});
```

### GCP Pub/Sub — Ordering Key

```javascript
await pubsub.topic('orders').publishMessage({
  data: Buffer.from(JSON.stringify(order)),
  orderingKey: order.userId,
});
```

---

## Migration & Re-partitioning

Tăng partition count trong Kafka:

```bash
# Tăng từ 6 → 12 partitions
kafka-topics --alter --topic orders \
  --partitions 12 \
  --bootstrap-server localhost:9092
```

**Hậu quả:**

- Message **mới** phân bổ theo 12 partitions
- Message **cũ** vẫn ở partition cũ
- Key routing thay đổi → **không** migrate data tự động
- Consumer rebalance

**Khi cần re-partition data:** Tạo topic mới + dual-write + migrate consumer + deprecate topic cũ.

---

## Anti-Patterns

| Anti-Pattern | Hậu Quả | Fix |
| ------------ | ------- | --- |
| Key = `null` cho business events cần ordering | Mất ordering | Dùng entity ID làm key |
| Key = constant `"default"` | 1 partition nhận 100% | Round-robin hoặc proper key |
| 3 partitions, 50 consumers | 47 consumers idle | Tăng partition hoặc giảm consumer |
| Tăng partition để fix hot key | Vẫn hot (cùng key → cùng partition) | Fix key design |
| Quá nhiều partitions (1000+) | Rebalance chậm, metadata overhead | Benchmark, right-size |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Làm sao đảm bảo ordering cho tất cả orders?

**Trả lời:** **Không thể** với multi-partition topic trừ khi dùng **1 partition** (mất parallelism). Thực tế: ordering **per order** bằng `key=orderId`. Global ordering hiếm khi cần và rất tốn kém.

### Câu 2: 100 partitions hay 10 partitions cho topic 50 MB/s?

**Trả lời:** Benchmark `throughput_per_partition` trên cluster thực tế. Nếu mỗi partition handle 5 MB/s → cần 10 cho throughput. Nếu cần scale tới 50 consumers → cần ≥50 partitions. Chọn max của hai, cộng headroom 20%.

### Câu 3: Hot partition do 1 tenant lớn — xử lý thế nào?

**Trả lời:** (1) Salt key `tenantId:shard` nếu không cần ordering toàn tenant. (2) Dedicated topic/partition cho tenant lớn. (3) Rate limit tenant ở producer. (4) Tách processing path cho enterprise tier.

---

**Cập Nhật:** 2026-07-03
