# Other Brokers — Các Broker Khác

> So sánh và học chuyên sâu các message broker bổ sung ngoài Kafka và RabbitMQ: Amazon SQS/SNS, Redis Streams, NATS JetStream, Apache Pulsar — khi nào chọn, trade-offs, và pattern thực tế trên production.

## Mục Lục

1. [Tại Sao Học Các Broker Khác](#tại-sao-học-các-broker-khác)
2. [Vị Trí Trong Ecosystem](#vị-trí-trong-ecosystem)
3. [Ma Trận So Sánh Nhanh](#ma-trận-so-sánh-nhanh)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học Các Broker Khác

Sau khi nắm [Apache Kafka](../03-apache-kafka/README.md) và [RabbitMQ](../04-rabbitmq/README.md), bạn cần hiểu **các lựa chọn thay thế** để:

- Trả lời phỏng vấn system design: "Tại sao không dùng Kafka?"
- Thiết kế đúng trên cloud (AWS SQS/SNS, GCP Pub/Sub)
- Tận dụng infrastructure có sẵn (Redis đã trong stack)
- Đánh giá broker mới (Pulsar, NATS) cho greenfield project

| Broker | Điểm Mạnh Chính | Khi Nào Xem Xét |
| ------ | --------------- | --------------- |
| **Amazon SQS/SNS** | Serverless, zero ops, AWS native | Ứng dụng trên AWS, Lambda trigger |
| **Redis Streams** | Ultra-low latency, đơn giản | Đã có Redis, real-time features |
| **NATS JetStream** | Lightweight, cloud-native, unified API | Edge/IoT, microservices nhẹ |
| **Apache Pulsar** | Multi-tenancy, geo-replication | SaaS multi-tenant, Kafka alternative |

> **Điều kiện tiên quyết:** Đã học [broker selection guide](../01-fundamentals/6-broker-selection-guide.md), [delivery guarantees](../01-fundamentals/3-delivery-guarantees.md), và ít nhất một trong Kafka hoặc RabbitMQ.

---

## Vị Trí Trong Ecosystem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGING LANDSCAPE (BẢN ĐỒ BROKER)                       │
│                                                                             │
│  High Throughput + Replay          Flexible Routing        Managed/Serverless│
│  ┌─────────────────┐              ┌──────────────┐        ┌──────────────┐  │
│  │ Apache Kafka    │              │ RabbitMQ     │        │ Amazon SQS   │  │
│  │ Apache Pulsar   │              │              │        │ SNS          │  │
│  └─────────────────┘              └──────────────┘        └──────────────┘  │
│                                                                             │
│  Low Latency + Simple              Cloud-Native Lightweight                 │
│  ┌─────────────────┐              ┌──────────────┐                          │
│  │ Redis Streams   │              │ NATS         │                          │
│  │ Redis Pub/Sub   │              │ JetStream    │                          │
│  └─────────────────┘              └──────────────┘                          │
│                                                                             │
│  Hybrid thực tế: Kafka (event bus) + SQS (Lambda) + Redis (real-time)       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Nguyên tắc chọn broker:**

```
1. Fit use case trước — không chọn theo hype
2. Tận dụng stack hiện có (Redis, AWS)
3. Đánh giá ops burden vs managed cost
4. Hybrid architecture là bình thường — không cần "một broker cho tất cả"
```

---

## Ma Trận So Sánh Nhanh

| Tiêu Chí | SQS/SNS | Redis Streams | NATS JetStream | Apache Pulsar |
| -------- | ------- | ------------- | -------------- | ------------- |
| **Model** | Queue + Pub/Sub | Stream log | Pub/Sub + Queue | Unified stream/queue |
| **Throughput** | Cao (managed) | Cao (in-memory) | Rất cao | Rất cao |
| **Latency** | ms–s | sub-ms | sub-ms–ms | Vài ms |
| **Replay** | ❌ | ✅ (trong retention) | ✅ | ✅ |
| **Ordering** | FIFO option | Per stream | Per subject | Per key |
| **Ops** | Zero (managed) | Thấp (nếu có Redis) | Thấp–Trung bình | Cao |
| **Multi-tenant** | ⚠️ (account-level) | ❌ | ⚠️ | ✅ Native |
| **Geo-replication** | ❌ (region-bound) | ❌ | ⚠️ | ✅ Built-in |
| **Protocol** | AWS API | Redis | NATS binary | Binary + Kafka API |

**So với Kafka/RabbitMQ (tóm tắt):**

| | Kafka | RabbitMQ | SQS | Redis Streams | NATS | Pulsar |
| - | ----- | -------- | --- | ------------- | ---- | ------ |
| Event log dài hạn | ✅ | ⚠️ | ❌ | ⚠️ | ✅ (JetStream) | ✅ |
| Task queue đơn giản | ⚠️ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Complex routing | ❌ | ✅ | ❌ | ❌ | ⚠️ | ⚠️ |
| Zero ops | ❌ | ❌ | ✅ | ⚠️ | ⚠️ | ❌ |

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-amazon-sqs-sns.md](./1-amazon-sqs-sns.md) | SQS Standard/FIFO, SNS fan-out, visibility timeout | 2 giờ |
| 2 | [2-redis-streams.md](./2-redis-streams.md) | Pub/Sub vs Streams, consumer groups, XREADGROUP | 1.5 giờ |
| 3 | [3-nats-jetstream.md](./3-nats-jetstream.md) | NATS Core vs JetStream, persistence, request-reply | 1.5 giờ |
| 4 | [4-apache-pulsar.md](./4-apache-pulsar.md) | Multi-tenancy, geo-replication, Pulsar Functions | 2 giờ |

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4. Bắt đầu với **SQS/SNS** (phổ biến trên AWS), sau đó **Redis** (dễ lab local), rồi **NATS** và **Pulsar** (broker nâng cao).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-amazon-sqs-sns.md](./1-amazon-sqs-sns.md) | Standard vs FIFO, visibility timeout, DLQ, SNS fan-out pattern |
| [2-redis-streams.md](./2-redis-streams.md) | Pub/Sub fire-and-forget vs Streams persistence, consumer groups |
| [3-nats-jetstream.md](./3-nats-jetstream.md) | Core messaging, JetStream durability, subject hierarchy |
| [4-apache-pulsar.md](./4-apache-pulsar.md) | Tenant/namespace, geo-replication, tiered storage, Functions |

---

## Bài Tập Thực Hành

### Lab 1: Redis Streams Local (30 phút)

```bash
# Docker Redis
docker run -d --name redis -p 6379:6379 redis:7

# redis-cli
XADD mystream * field1 value1
XREAD COUNT 10 STREAMS mystream 0
```

### Lab 2: NATS + JetStream (45 phút)

```bash
# Docker NATS với JetStream enabled
docker run -d --name nats -p 4222:4222 -p 8222:8222 nats:latest -js

# nats CLI: pub/sub và stream create
nats stream add ORDERS --subjects "orders.>" --storage file --retention limits
```

### Lab 3: AWS SQS/SNS (LocalStack hoặc AWS Free Tier)

```
1. Tạo SNS topic + subscribe SQS queue
2. Publish message → verify fan-out đến queue
3. Tạo FIFO queue — test MessageGroupId ordering
4. Cấu hình DLQ + maxReceiveCount
5. Test visibility timeout — message invisible khi đang xử lý
```

### Lab 4: So Sánh Broker Cho Use Case

```
Scenario: Notification system — 10K events/s, AWS-native, Lambda consumers

1. Thiết kế với SNS → SQS → Lambda
2. So sánh với Kafka MSK — trade-offs ops/cost
3. Document decision: tại sao chọn broker nào
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: SQS Standard vs FIFO — khi nào dùng FIFO?

**Gợi ý trả lời:** **FIFO (First-In-First-Out — Vào Trước Ra Trước)** khi cần strict ordering per MessageGroupId và deduplication (khử trùng lặp). Giới hạn ~300 msg/s (có thể tăng với batching). **Standard** cho throughput cao hơn, at-least-once, không đảm bảo ordering — phù hợp hầu hết use case. Consumer phải idempotent với Standard.

### Câu 2: Redis Pub/Sub vs Redis Streams?

**Gợi ý trả lời:** **Pub/Sub** — fire-and-forget, không persistence, subscriber offline mất message. **Streams** — append-only log, consumer groups, ACK, replay trong retention window — giống lightweight Kafka. Chọn Streams cho messaging cần durability; Pub/Sub cho real-time broadcast không cần lưu.

### Câu 3: NATS Core vs JetStream?

**Gợi ý trả lời:** **NATS Core** — in-memory, ultra-fast, không persistence (mất message khi subscriber offline). **JetStream** — persistence, at-least-once, stream retention, consumer ack — khi cần durability mà vẫn giữ NATS simplicity. Pattern: Core cho request-reply realtime, JetStream cho event storage.

### Câu 4: Pulsar khác Kafka ở điểm nào?

**Gợi ý trả lời:** **Pulsar** tách storage (BookKeeper) và compute (broker) — scale độc lập, multi-tenancy native (tenant/namespace), geo-replication built-in, unified queue + streaming model. **Kafka** có ecosystem lớn hơn, community rộng, nhiều managed options. Pulsar hỗ trợ Kafka protocol compatibility cho migration.

### Câu 5: Khi nào dùng hybrid architecture (nhiều broker)?

**Gợi ý trả lời:** Khi use cases khác nhau trong cùng hệ thống: **Kafka** làm event backbone/audit log, **SQS** cho Lambda serverless workers, **Redis** cho real-time state/notifications. Tránh over-engineering — mỗi broker thêm operational complexity. Document rõ boundary và data flow.

---

**Xem tiếp:** [1-amazon-sqs-sns.md](./1-amazon-sqs-sns.md) — bắt đầu với Amazon SQS và SNS trên AWS.
