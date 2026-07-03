# Redis Streams & Pub/Sub — Luồng Tin Nhắn Trên Redis

> So sánh Redis Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký) và Redis Streams: fire-and-forget vs persistence, consumer groups (nhóm consumer), XREADGROUP, và khi nào dùng Redis làm lightweight message broker.

## Mục Lục

1. [Redis Trong Messaging](#redis-trong-messaging)
2. [Redis Pub/Sub](#redis-pubsub)
3. [Redis Streams](#redis-streams)
4. [Consumer Groups](#consumer-groups)
5. [Commands Thực Tế](#commands-thực-tế)
6. [Persistence & Durability](#persistence--durability)
7. [So Sánh Với Kafka](#so-sánh-với-kafka)
8. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Redis Trong Messaging

Redis thường được biết đến như **cache (bộ nhớ đệm)** và **session store**, nhưng cũng hỗ trợ messaging qua hai cơ chế:

| Cơ Chế | Persistence | Use Case |
| ------ | ----------- | -------- |
| **Pub/Sub** | ❌ Không | Real-time broadcast, không cần lưu |
| **Streams** | ✅ Có (AOF/RDB) | Lightweight queue, consumer groups |

```
┌─────────────────────────────────────────────────────────────┐
│                    REDIS MESSAGING                           │
│                                                             │
│  Pub/Sub:  Publisher ──► Channel ──► Subscribers (live only)│
│                                                             │
│  Streams:  XADD ──► Stream (log) ──► Consumer Groups       │
│                      │                      │               │
│                      └── retention ────────┘               │
└─────────────────────────────────────────────────────────────┘
```

**Khi nào xem xét Redis:**

- Đã có Redis trong stack (cache, session)
- Cần **ultra-low latency** (sub-millisecond)
- Volume vừa phải, retention ngắn–trung bình
- Real-time features: chat, live dashboard, notifications

---

## Redis Pub/Sub

### Cơ Chế

**Fire-and-forget (Gửi Và Quên)** — publisher publish message đến channel, tất cả subscriber **đang online** nhận ngay. Không lưu message.

```
Publisher                    Redis                    Subscribers
    │                          │                          │
    │── PUBLISH channel msg ──►│── push ─────────────────►│ Client A
    │                          │── push ─────────────────►│ Client B
    │                          │                          │
    │                          │    Client C offline ──► ❌ mất message
```

### Commands

```bash
# Terminal 1 — Subscriber
SUBSCRIBE notifications
# Chờ message...

# Terminal 2 — Publisher
PUBLISH notifications "User 123 logged in"
```

```javascript
// Node.js — ioredis
const Redis = require('ioredis');
const sub = new Redis();
const pub = new Redis();

sub.subscribe('notifications', (err, count) => {
  console.log(`Subscribed to ${count} channels`);
});

sub.on('message', (channel, message) => {
  console.log(`Received: ${message}`);
});

pub.publish('notifications', JSON.stringify({ userId: 123, event: 'login' }));
```

### Pattern Matching — PSUBSCRIBE

```bash
PSUBSCRIBE order.*
# Nhận message từ order.created, order.cancelled, ...
```

### Hạn Chế Pub/Sub

```
❌ Không persistence — subscriber offline = mất message
❌ Không ACK — không biết message đã xử lý chưa
❌ Không consumer groups — mọi subscriber nhận tất cả (broadcast)
❌ Không replay
❌ Backpressure yếu — slow subscriber có thể bị disconnect (client output buffer)
```

**Phù hợp:** Live updates, WebSocket fan-out qua Redis, cache invalidation broadcast, không cần durability.

---

## Redis Streams

**Redis Streams** — append-only log structure (cấu trúc log chỉ thêm), tương tự lightweight Kafka topic.

### Stream Entry

Mỗi entry có **ID** (timestamp-based) và **field-value pairs**:

```
Stream: "orders"
┌────────────────────────────────────────────────────────────┐
│ 1609459200000-0  │ orderId: 123 │ action: created │ ...  │
│ 1609459201000-0  │ orderId: 124 │ action: created │ ...  │
│ 1609459202000-0  │ orderId: 123 │ action: paid    │ ...  │
└────────────────────────────────────────────────────────────┘
```

### XADD — Thêm Message

```bash
# Auto-generate ID (*)
XADD orders * orderId 123 action created amount 99.99

# Explicit ID (phải lớn hơn ID cuối)
XADD orders 1609459203000-0 orderId 125 action created
```

```python
# Python — redis-py
import redis
r = redis.Redis()

entry_id = r.xadd('orders', {
    'orderId': '123',
    'action': 'created',
    'amount': '99.99'
})
```

### XREAD — Đọc Stream

```bash
# Đọc từ đầu
XREAD COUNT 10 STREAMS orders 0

# Đọc message mới (blocking)
XREAD BLOCK 5000 STREAMS orders $
# $ = chỉ message mới sau lệnh này
```

**Vấn đề XREAD đơn giản:** Mỗi consumer đọc độc lập — không có work distribution (phân phối công việc) như queue.

---

## Consumer Groups

**Consumer Group (Nhóm Consumer)** — nhiều consumer chia sẻ xử lý stream, mỗi message chỉ giao cho **một** consumer trong group.

```
Stream "orders"
       │
       ▼
┌──────────────────┐
│ Consumer Group   │
│ "fulfillment"    │
├──────────────────┤
│ Consumer A ──────┼──► message 1, 3, 5...
│ Consumer B ──────┼──► message 2, 4, 6...
└──────────────────┘
```

### Tạo Group & Đọc

```bash
# Tạo consumer group (đọc từ đầu stream)
XGROUP CREATE orders fulfillment 0 MKSTREAM

# Consumer đọc message
XREADGROUP GROUP fulfillment consumer-1 COUNT 1 BLOCK 5000 STREAMS orders >
# > = message chưa deliver cho group này
```

### ACK & Pending

```bash
# Sau khi xử lý xong — ACK
XACK orders fulfillment 1609459200000-0

# Xem pending (message đã deliver chưa ACK)
XPENDING orders fulfillment

# Claim message từ consumer chết (idle quá lâu)
XCLAIM orders fulfillment consumer-2 3600000 1609459200000-0
# 3600000 ms = 1 giờ idle
```

### Luồng Xử Lý Chuẩn

```
1. XREADGROUP → nhận message (vào Pending Entries List — PEL)
2. Xử lý business logic
3. XACK → xóa khỏi PEL
4. Nếu crash trước XACK → message pending, consumer khác XCLAIM
```

```javascript
// Node.js — consumer loop
async function consume() {
  while (true) {
    const results = await redis.xreadgroup(
      'GROUP', 'fulfillment', 'worker-1',
      'COUNT', 1, 'BLOCK', 5000,
      'STREAMS', 'orders', '>'
    );
    if (!results) continue;

    for (const [stream, messages] of results) {
      for (const [id, fields] of messages) {
        try {
          await processOrder(fields);
          await redis.xack('orders', 'fulfillment', id);
        } catch (err) {
          // Không ACK — message pending, retry sau
          console.error('Failed:', id, err);
        }
      }
    }
  }
}
```

---

## Commands Thực Tế

### Stream Management

| Command | Mô Tả |
| ------- | ----- |
| **XLEN** | Số entry trong stream |
| **XRANGE** | Đọc range theo ID |
| **XTRIM** | Cắt stream (MAXLEN) — retention |
| **XINFO STREAM** | Metadata stream |
| **XINFO GROUPS** | Danh sách consumer groups |
| **XDEL** | Xóa entry cụ thể (hiếm dùng) |

### Retention — XTRIM

```bash
# Giữ tối đa 10000 entries
XADD orders MAXLEN ~ 10000 * orderId 126 action created

# ~ = approximate trim (hiệu năng tốt hơn)
```

### So Sánh Commands

| Mục Đích | Pub/Sub | Streams |
| -------- | ------- | ------- |
| Publish | PUBLISH | XADD |
| Subscribe | SUBSCRIBE | XREAD / XREADGROUP |
| Ack | ❌ | XACK |
| Pending/retry | ❌ | XPENDING, XCLAIM |
| History | ❌ | XRANGE |

---

## Persistence & Durability

Redis persistence phụ thuộc cấu hình:

| Mode | Mô Tả | Durability |
| ---- | ----- | ---------- |
| **RDB** | Snapshot định kỳ | Có thể mất data giữa snapshots |
| **AOF** | Append-only file mỗi write | Tốt hơn, configurable fsync |
| **No persistence** | Chỉ RAM | Mất hết khi restart |

```
Streams data nằm trong Redis memory (+ disk nếu AOF/RDB)
→ Không phải durable như Kafka (replicated log nhiều broker)
→ Phù hợp messaging ngắn hạn, không phải system of record dài hạn
```

### Redis Cluster & Streams

- Stream keys hash đến **slot** — cùng stream trên một shard
- Consumer group gắn với stream key — không cross-shard
- **Hot key** risk nếu một stream quá lớn

### High Availability

```
Redis Sentinel — failover master/replica
Redis Cluster — sharding + HA

Lưu ý: Failover có thể mất vài message nếu chưa replicate (tùy config)
```

---

## So Sánh Với Kafka

| Tiêu Chí | Redis Streams | Kafka |
| -------- | ------------- | ----- |
| **Latency** | sub-ms | vài ms |
| **Throughput** | Cao (in-memory) | Rất cao (disk-optimized) |
| **Retention** | MAXLEN / memory-bound | TB, days/months |
| **Durability** | AOF/RDB, single-node risk | Replicated partitions |
| **Consumer groups** | ✅ | ✅ |
| **Replay** | ✅ (trong retention) | ✅ (full history) |
| **Ecosystem** | Hạn chế | Connect, Streams, Schema Registry |
| **Ops** | Thấp (nếu đã có Redis) | Cao |

```
Chọn Redis Streams khi:
  - Đã có Redis, latency cực thấp
  - Retention ngắn (hours/days)
  - Volume vừa phải

Chọn Kafka khi:
  - Event log dài hạn, replay, audit
  - Throughput hàng triệu/s
  - Stream processing ecosystem
```

---

## Thiết Kế Thực Tế

### Pattern: Real-Time Dashboard

```
Microservices ──XADD──► Stream "metrics"
                              │
                    Consumer Group "dashboard"
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              WebSocket Server    Aggregation Worker
              (push to clients)   (rollup stats)
```

### Pattern: Lightweight Task Queue

```
API ──XADD──► Stream "tasks"
                    │
          Consumer Group "workers"
                    │
          Worker 1, 2, 3 (competing consumers)
                    │
          XACK sau khi xử lý
          XPENDING + XCLAIM cho retry
```

### Anti-Patterns

```
❌ Dùng Redis Streams làm primary event store 7 năm
❌ Pub/Sub cho payment processing (mất message)
❌ Một stream khổng lồ không trim — OOM risk
❌ Không XACK — pending list phình, message stuck
```

### Checklist Production

```
□ AOF enabled với fsync policy phù hợp
□ XTRIM / MAXLEN để giới hạn memory
□ Monitor: stream length, pending count, memory usage
□ Consumer XACK trong finally block
□ XCLAIM timeout cho poison message recovery
□ Idempotent processing — XCLAIM có thể duplicate
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Redis Pub/Sub vs Streams — khi nào dùng cái nào?

**Đáp án mẫu:** **Pub/Sub** — real-time broadcast, subscriber phải online, không cần lưu (cache invalidation, live notifications). **Streams** — cần persistence, consumer groups, ACK, replay trong retention, work queue pattern. Streams là lựa chọn messaging; Pub/Sub là notification channel.

### Câu 2: Consumer group hoạt động thế nào trong Redis Streams?

**Đáp án mẫu:** Group track **last delivered ID** cho mỗi consumer. `XREADGROUP` với `>` đọc message mới chưa assign. Message vào **PEL (Pending Entries List)** cho đến `XACK`. Consumer crash → message pending → consumer khác `XCLAIM` sau min-idle-time. Tương tự Kafka consumer group nhưng đơn giản hơn, single-node.

### Câu 3: Redis làm message broker — rủi ro gì?

**Đáp án mẫu:** (1) **Durability** — memory-first, failover có thể mất data. (2) **Scale** — single stream hot key, không scale như Kafka partitions. (3) **Retention** — memory-bound, không phù hợp long-term log. (4) **No built-in DLQ** — tự implement với separate stream hoặc XCLAIM logic.

### Câu 4: XACK quan trọng thế nào? Không ACK thì sao?

**Đáp án mẫu:** `XACK` báo message đã xử lý xong, xóa khỏi PEL. Không ACK → message **pending** mãi, không deliver cho consumer khác (trừ XCLAIM). PEL phình → memory leak logic, message không được xử lý lại tự động. Luôn ACK trong `finally` hoặc sau success; fail thì để pending cho retry.

### Câu 5: So sánh Redis Streams với RabbitMQ queue?

**Đáp án mẫu:** **RabbitMQ** — mature AMQP, routing (exchanges), DLQ native, push-based, durable queue design. **Redis Streams** — đơn giản, latency thấp, tận dụng Redis có sẵn, pull-based với consumer groups. RabbitMQ cho task queue production đầy đủ tính năng; Redis Streams cho lightweight, real-time, đã có Redis.

---

**Xem tiếp:** [3-nats-jetstream.md](./3-nats-jetstream.md) — NATS Core vs JetStream.
