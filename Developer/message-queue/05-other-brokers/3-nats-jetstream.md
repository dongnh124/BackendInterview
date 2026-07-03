# NATS & JetStream — Messaging Cloud-Native Nhẹ

> NATS (Neural Autonomic Transport System) — messaging system nhẹ, cloud-native: NATS Core (in-memory, ultra-fast) vs JetStream (persistence, at-least-once), subject hierarchy, request-reply pattern, và use case edge/IoT.

## Mục Lục

1. [Tổng Quan NATS](#tổng-quan-nats)
2. [NATS Core Messaging](#nats-core-messaging)
3. [Subject Hierarchy](#subject-hierarchy)
4. [Request-Reply Pattern](#request-reply-pattern)
5. [JetStream — Persistence Layer](#jetstream--persistence-layer)
6. [Streams & Consumers](#streams--consumers)
7. [So Sánh Core vs JetStream](#so-sánh-core-vs-jetstream)
8. [Deployment & HA](#deployment--ha)
9. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan NATS

**NATS** là message broker **nhẹ, hiệu năng cao**, thiết kế cho cloud-native và distributed systems.

```
┌─────────────────────────────────────────────────────────────┐
│                      NATS ECOSYSTEM                          │
│                                                             │
│  NATS Core          JetStream          NATS Services        │
│  (pub/sub, RR)      (persistence)      (microservices)      │
│                                                             │
│  Đặc điểm:                                                  │
│  - Binary protocol, minimal overhead                        │
│  - Subject-based routing                                    │
│  - Clustering, super-cluster (gateway)                      │
│  - CNCF project, open source                                │
└─────────────────────────────────────────────────────────────┘
```

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **NATS Server (nats-server)** | Broker — routing messages |
| **NATS Core** | Pub/Sub, Request-Reply — không persistence |
| **JetStream** | Stream storage, consumer ack, retention |
| **NATS CLI** | `nats` command-line tool |

**Điểm mạnh:**

- **Ultra-low latency** — triệu message/giây trên single node
- **Đơn giản** — binary protocol, client libraries đa ngôn ngữ
- **Cloud-native** — container-friendly, footprint nhỏ
- **Unified API** — pub/sub + queue + request-reply cùng model

---

## NATS Core Messaging

### Publish-Subscribe

```
Publisher ──► Subject "orders.created" ──► Subscribers (tất cả nhận)
```

```bash
# Terminal 1 — Subscribe
nats sub "orders.>"

# Terminal 2 — Publish
nats pub orders.created '{"orderId": "123"}'
```

```go
// Go — nats.go
nc, _ := nats.Connect(nats.DefaultURL)
defer nc.Close()

// Subscribe
nc.Subscribe("orders.created", func(m *nats.Msg) {
    fmt.Printf("Received: %s\n", string(m.Data))
})

// Publish
nc.Publish("orders.created", []byte(`{"orderId":"123"}`))
```

### Queue Groups — Load Balancing

**Queue Group (Nhóm Hàng Đợi)** — nhiều subscriber cùng queue name, mỗi message chỉ đến **một** subscriber (competing consumers).

```go
// Worker 1 và Worker 2 cùng queue "processors"
nc.QueueSubscribe("tasks.process", "processors", handleTask)
```

```
                    Subject "tasks.process"
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Queue Group "processors"
              │
         Worker A    Worker B    Worker C
         (1 msg → 1 worker only)
```

**Khác Pub/Sub thuần:** `Subscribe` — broadcast; `QueueSubscribe` — work distribution.

### Core Limitations

```
❌ Không persistence — subscriber offline = mất message
❌ Không ACK semantics — fire-and-forget
❌ Không replay
✅ Cực nhanh, đơn giản — phù hợp control plane, signaling
```

---

## Subject Hierarchy

NATS dùng **subject (chủ đề)** thay vì topic/queue — hierarchical với `.` separator.

```
orders.created
orders.updated
orders.cancelled
payments.processed
```

### Wildcards

| Wildcard | Ý Nghĩa | Ví Dụ |
| -------- | ------- | ----- |
| `*` | Một token | `orders.*` match `orders.created`, không match `orders.item.created` |
| `>` | Một hoặc nhiều token (cuối subject) | `orders.>` match tất cả bắt đầu `orders.` |

```bash
# Subscribe tất cả order events
nats sub "orders.>"

# Subscribe chỉ created/updated
nats sub "orders.created"
nats sub "orders.updated"
```

**Best practice:** Thiết kế hierarchy có ý nghĩa — `service.entity.action` (ví dụ: `inventory.stock.reserved`).

---

## Request-Reply Pattern

**Request-Reply (Yêu Cầu-Phản Hồi)** — built-in, không cần reply queue riêng như RabbitMQ.

```
Client                              Server
   │── Request (subject: "help") ──►│
   │    Reply subject: _INBOX.abc    │
   │◄── Response ────────────────────│
```

```go
// Server — respond to requests
nc.Subscribe("help", func(m *nats.Msg) {
    response := []byte("here to help")
    m.Respond(response)
})

// Client — request with timeout
msg, err := nc.Request("help", []byte("need help"), 2*time.Second)
```

| So Với RabbitMQ RPC | NATS |
| ------------------- | ---- |
| Reply queue | Inbox subject tự động |
| correlationId | Built-in request pairing |
| Setup | Đơn giản hơn |

**Phù hợp:** Service discovery, health check, sync RPC over messaging, control commands.

---

## JetStream — Persistence Layer

**JetStream** — persistence layer trên NATS Core, thêm streams, consumers, ack.

```
┌─────────────────────────────────────────────────────────────┐
│  NATS Server + JetStream (-js flag)                          │
│                                                             │
│  Core: pub/sub, request-reply (ephemeral)                   │
│  JetStream: Stream (storage) ← publish subject              │
│             Consumer (delivery, ack) → applications           │
└─────────────────────────────────────────────────────────────┘
```

### Enable JetStream

```bash
# Docker
docker run -d -p 4222:4222 -p 8222:8222 nats:latest -js

# Config file
jetstream {
  store_dir: /data/jetstream
  max_memory_store: 1GB
  max_file_store: 10GB
}
```

### Stream — Lưu Trữ Messages

**Stream** capture messages từ subject(s) và lưu theo retention policy.

```bash
# Tạo stream
nats stream add ORDERS \
  --subjects "orders.>" \
  --storage file \
  --retention limits \
  --max-msgs 1000000 \
  --max-age 7d
```

| Retention | Mô Tả |
| --------- | ----- |
| **limits** | Xóa khi đạt max-msgs, max-bytes, max-age |
| **interest** | Xóa khi tất cả consumers đã ack |
| **workqueue** | Message xóa sau khi một consumer ack (task queue) |

| Storage | Mô Tả |
| ------- | ----- |
| **memory** | Nhanh, mất khi restart (có replicate) |
| **file** | Disk-backed, durable |

---

## Streams & Consumers

### Consumer Types

| Loại | Mô Tả |
| ---- | ----- |
| **Push** | Server đẩy message đến subscriber |
| **Pull** | Client kéo batch message (recommended cho scale) |
| **Durable** | State survive restart — track ack position |
| **Ephemeral** | Tạm thời, xóa khi disconnect |

```bash
# Tạo durable pull consumer
nats consumer add ORDERS fulfillment \
  --pull \
  --deliver all \
  --ack explicit \
  --max-deliver 3 \
  --filter "orders.created"
```

### Acknowledgment

```go
// JetStream pull consumer
js, _ := nc.JetStream()

sub, _ := js.PullSubscribe("orders.created", "fulfillment")
msgs, _ := sub.Fetch(10)

for _, msg := range msgs {
    err := processOrder(msg.Data)
    if err != nil {
        msg.Nak()        // Negative ack — redeliver
        // msg.Term()    // Terminate — không redeliver (poison)
        continue
    }
    msg.Ack()
}
```

| Ack Type | Ý Nghĩa |
| -------- | ------- |
| **Ack** | Xử lý thành công |
| **Nak** | Fail — redeliver (có backoff) |
| **Term** | Poison message — dừng retry |
| **InProgress** | Extend ack wait (long processing) |

### Max Deliver & Backoff

```
max-deliver: 3
ack-wait: 30s

Message fail → Nak → redeliver (tối đa 3 lần)
Sau max-deliver → có thể route sang dead letter stream (manual config)
```

---

## So Sánh Core vs JetStream

| Tiêu Chí | NATS Core | JetStream |
| -------- | --------- | --------- |
| **Persistence** | ❌ | ✅ |
| **Delivery** | At-most-once | At-least-once (với ack) |
| **Replay** | ❌ | ✅ |
| **Latency** | Cực thấp | Thấp (có disk I/O) |
| **Use case** | Signaling, RPC, live | Events, tasks, audit |
| **Consumer state** | ❌ | Durable consumers |

```
Quy tắc chọn:
  Core    → realtime, không cần lưu, request-reply
  JetStream → cần durability, retry, replay, work queue
```

### So Với Kafka

| | NATS JetStream | Kafka |
| - | -------------- | ----- |
| **Complexity** | Thấp | Cao |
| **Throughput** | Rất cao | Rất cao |
| **Ecosystem** | Nhỏ hơn | Lớn (Connect, Streams) |
| **Multi-tenancy** | Account-based | Manual |
| **Ops footprint** | Nhỏ | Lớn |
| **Replay** | ✅ | ✅ |

---

## Deployment & HA

### Clustering

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ NATS-1  │──│ NATS-2  │──│ NATS-3  │
└─────────┘  └─────────┘  └─────────┘
     Cluster — subject routing across nodes
```

### JetStream HA — R3 Replication

```bash
# Stream với 3 replicas
nats stream add ORDERS --replicas 3
```

| Replicas | Fault Tolerance |
| -------- | --------------- |
| 1 | Không HA |
| 3 | Chịu 1 node fail (khuyến nghị production) |
| 5 | Chịu 2 node fail |

### Super Cluster & Leaf Nodes

- **Super Cluster** — kết nối nhiều NATS cluster (geo)
- **Leaf Nodes** — edge/IoT connect về hub — phù hợp distributed edge

---

## Thiết Kế Thực Tế

### Pattern: Microservices với NATS

```
                    ┌──────────────┐
API Gateway ───────►│ NATS         │
                    │              │
                    ├─ Core: RPC health checks
                    ├─ JetStream: order events
                    └─ Queue groups: background workers
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    Order Service    Payment Service    Notification
```

### Pattern: IoT Telemetry

```
Sensors ──► Leaf Node (edge) ──► Hub Cluster ──► JetStream
                                                    │
                                              Analytics Pipeline
```

### Khi Nào Chọn NATS

```
✅ Cloud-native microservices, footprint nhỏ
✅ Request-reply + pub/sub unified
✅ Edge computing, IoT
✅ Team muốn đơn giản hơn Kafka
✅ Latency-critical signaling

❌ Cần ecosystem lớn (Kafka Connect, ksqlDB)
❌ Team đã invest nặng Kafka
❌ Complex stream processing SQL
```

### Checklist Production

```
□ JetStream enabled với file storage
□ Stream replicas = 3 cho HA
□ Durable consumers với explicit ack
□ max-deliver + monitoring undelivered
□ Subject naming convention documented
□ TLS + authentication (NKeys, JWT)
□ Monitor: stream bytes, consumer lag, slow consumers
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: NATS Core vs JetStream — khi nào dùng cái nào?

**Đáp án mẫu:** **Core** — pub/sub realtime, request-reply, không cần lưu message (control plane, health checks, live updates). **JetStream** — cần persistence, at-least-once với ack, replay, work queue retention. Nhiều hệ thống dùng cả hai: Core cho RPC, JetStream cho events.

### Câu 2: Queue group trong NATS hoạt động thế nào?

**Đáp án mẫu:** Nhiều subscriber subscribe cùng subject với **cùng queue group name** — NATS deliver mỗi message cho **một** subscriber trong group (round-robin). Khác broadcast `Subscribe` (tất cả nhận). Tương tự competing consumers trong RabbitMQ/SQS.

### Câu 3: JetStream retention policies?

**Đáp án mẫu:** **limits** — giữ theo max-msgs/bytes/age. **interest** — xóa khi all consumers ack (tiết kiệm storage). **workqueue** — xóa sau khi một consumer ack — pattern task queue. Chọn theo use case: event log → limits; task → workqueue.

### Câu 4: NATS so với Kafka — trade-offs?

**Đáp án mẫu:** **NATS** — đơn giản, ops nhẹ, latency thấp, unified pub/sub+queue+RPC, ecosystem nhỏ. **Kafka** — throughput scale lớn, ecosystem phong phú (Connect, Streams), phù hợp event backbone doanh nghiệp. NATS cho greenfield cloud-native nhẹ; Kafka cho data platform, analytics pipeline.

### Câu 5: Xử lý poison message trong JetStream?

**Đáp án mẫu:** Set `max-deliver` (ví dụ 3). Consumer `Nak()` để retry, `Term()` để dừng retry. Sau max-deliver, message không redeliver — cần monitor và manual intervention. Pattern: separate **dead letter stream** + consumer subscribe để alert/inspect. Kết hợp idempotent processing.

---

**Xem tiếp:** [4-apache-pulsar.md](./4-apache-pulsar.md) — Apache Pulsar multi-tenancy và geo-replication.
