# Broker Selection Guide — Hướng Dẫn Chọn Broker

> Decision tree (cây quyết định) và ma trận so sánh để chọn Message Broker phù hợp: Apache Kafka, RabbitMQ, Amazon SQS/SNS, Redis Streams, NATS JetStream, Apache Pulsar.

## Mục Lục

1. [Framework Quyết Định](#framework-quyết-định)
2. [Decision Tree](#decision-tree)
3. [So Sánh Chi Tiết](#so-sánh-chi-tiết)
4. [Kafka — Khi Nào Chọn](#kafka--khi-nào-chọn)
5. [RabbitMQ — Khi Nào Chọn](#rabbitmq--khi-nào-chọn)
6. [Cloud Managed — Khi Nào Chọn](#cloud-managed--khi-nào-chọn)
7. [Các Broker Khác](#các-broker-khác)
8. [Migration & Hybrid](#migration--hybrid)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Framework Quyết Định

Trước khi chọn broker, trả lời 5 câu hỏi:

| # | Câu Hỏi | Ảnh Hưởng |
| - | ------- | --------- |
| 1 | **Throughput** cần bao nhiêu? | Kafka/Pulsar vs RabbitMQ/SQS |
| 2 | Cần **replay (phát lại)** event không? | Kafka, Pulsar vs SQS, RabbitMQ queue |
| 3 | **Routing** phức tạp hay đơn giản? | RabbitMQ exchanges vs Kafka topics |
| 4 | **Ops burden** — tự host hay managed? | Self-hosted vs MSK, Confluent Cloud |
| 5 | **Ordering** và **latency** requirement? | FIFO SQS, partition key, Redis |

---

## Decision Tree

```
                        ┌─────────────────────────┐
                        │ Bắt đầu chọn broker      │
                        └────────────┬────────────┘
                                     │
                        ┌────────────▼────────────┐
                        │ Cần event replay/log?    │
                        └────────────┬────────────┘
                              Có   │   Không
                    ┌──────────────┴──────────────┐
                    ▼                              ▼
           ┌────────────────┐          ┌─────────────────┐
           │ Throughput >     │          │ Task queue /    │
           │ 100K msg/s?      │          │ simple routing? │
           └───────┬────────┘          └────────┬────────┘
              Có   │   Không                Có   │   Không
         ┌─────────┴─────────┐          ┌─────────┴─────────┐
         ▼                   ▼          ▼                   ▼
    ┌─────────┐        ┌─────────┐  ┌──────────┐      ┌──────────┐
    │ Kafka   │        │ Pulsar  │  │ RabbitMQ │      │ SQS/SNS  │
    │ hoặc    │        │         │  │          │      │ (AWS)    │
    │ Redpanda│        └─────────┘  └──────────┘      └──────────┘
    └─────────┘
                                     │
                        ┌────────────▼────────────┐
                        │ Latency < 10ms,         │
                        │ đã có Redis?            │
                        └────────────┬────────────┘
                              Có     │
                                     ▼
                              ┌──────────────┐
                              │ Redis Streams│
                              └──────────────┘
```

---

## So Sánh Chi Tiết

| Tiêu Chí | Kafka | RabbitMQ | SQS/SNS | Redis Streams | Pulsar |
| -------- | ----- | -------- | ------- | ------------- | ------ |
| **Throughput** | Rất cao (triệu/s) | Trung bình (50K/s) | Cao (managed) | Cao (in-memory) | Rất cao |
| **Latency** | Vài ms | Vài ms | ms–s | sub-ms | Vài ms |
| **Replay** | ✅ Native | ⚠️ Streams plugin | ❌ | ✅ | ✅ |
| **Routing** | Topic + key | Exchange (linh hoạt) | Đơn giản | Đơn giản | Topic + subscription |
| **Ordering** | Per partition | Per queue | FIFO option | Per stream | Per key |
| **Ops** | Cao | Trung bình | Không (managed) | Thấp (nếu có Redis) | Cao |
| **Protocol** | Binary custom | AMQP | AWS API | Redis | Binary custom |
| **Multi-tenant** | ⚠️ | ⚠️ | ✅ | ❌ | ✅ Native |
| **Learning curve** | Cao | Trung bình | Thấp | Thấp | Cao |

---

## Kafka — Khi Nào Chọn

### Điểm Mạnh

- **Event log bền vững** — retention theo thời gian/dung lượng
- **Replay** — consumer đọc lại từ offset bất kỳ
- **Throughput cực cao** — hàng triệu message/giây
- **Ecosystem** — Kafka Connect, Streams, Schema Registry
- **Multiple consumer groups** — Pub/Sub native

### Use Cases Phù Hợp

```
✅ Event streaming pipeline
✅ Log aggregation
✅ CDC (Change Data Capture)
✅ Real-time analytics
✅ Event sourcing backbone
✅ Microservices event bus (high volume)
```

### Không Phù Hợp Khi

```
❌ Task queue đơn giản, volume thấp
❌ Complex routing (topic-per-event-type có thể quá nhiều)
❌ Team nhỏ, không muốn ops Kafka cluster
❌ RPC pattern với reply queue
❌ Message TTL ngắn, fire-and-forget only
```

### Managed Options

| Service | Mô Tả |
| ------- | ----- |
| **Amazon MSK** | Managed Kafka trên AWS |
| **Confluent Cloud** | Full ecosystem, Schema Registry |
| **Azure Event Hubs** | Kafka-compatible endpoint |
| **Redpanda** | Kafka API, không cần ZooKeeper |

---

## RabbitMQ — Khi Nào Chọn

### Điểm Mạnh

- **Routing linh hoạt** — Direct, Fanout, Topic, Headers exchange
- **AMQP standard** — interoperability
- **Dễ triển khai** — single node cho dev/small prod
- **DLQ native** — dead letter exchange
- **RPC pattern** — request/reply với correlationId
- **Quorum queues** — HA consensus-based

### Use Cases Phù Hợp

```
✅ Task/work queue
✅ Background job processing
✅ Microservices với routing phức tạp
✅ RPC over messaging
✅ Medium throughput (< 50K msg/s)
✅ Priority queue
```

### Không Phù Hợp Khi

```
❌ Cần replay toàn bộ event history
❌ Throughput cực cao (triệu/s)
❌ Long-term event storage
❌ Stream processing native
```

### So Sánh Nhanh: Kafka vs RabbitMQ

| Câu Hỏi | Chọn Kafka | Chọn RabbitMQ |
| ------- | ---------- | ------------- |
| "Ai đã consume message?" | Consumer track offset | Message xóa sau ack |
| "Cần đọc lại data cũ?" | ✅ | ❌ |
| "1 event → nhiều service" | Consumer groups | Fanout exchange |
| "Gửi task cho 1 worker" | Consumer group | Queue competing consumers |
| "RPC request/reply" | Khó | ✅ Built-in |

---

## Cloud Managed — Khi Nào Chọn

### Amazon SQS + SNS

```
Chọn khi:
- AWS-native application
- Không muốn manage broker
- Serverless (Lambda trigger)
- Fan-out: SNS → nhiều SQS

SQS Standard: high throughput, no ordering
SQS FIFO: ordering + dedup, 300 msg/s limit
```

### Google Cloud Pub/Sub

```
- Push hoặc Pull subscription
- Ordering keys cho partial ordering
- Tích hợp GCP ecosystem
```

### Azure Service Bus

```
- Queue và Topic
- Sessions cho ordering
- Dead letter queue built-in
```

### Trade-off Managed vs Self-hosted

| | Managed | Self-hosted |
| - | ------- | ----------- |
| **Ops** | Thấp | Cao |
| **Cost** | Pay per use (có thể đắt ở scale) | Infra + engineer time |
| **Control** | Giới hạn config | Full control |
| **Compliance** | Phụ thuộc cloud region | On-prem possible |

---

## Các Broker Khác

### Redis Streams

```
Chọn khi:
- Đã có Redis trong stack
- Ultra-low latency (< 1ms)
- Volume vừa phải, có thể chấp nhận in-memory risk
- Real-time features (chat, live dashboard)

Không chọn khi:
- Cần durability cao nhất (dù có AOF/RDB)
- Long retention terabytes
```

### NATS / JetStream

```
Chọn khi:
- Cloud-native, lightweight
- Request-reply + pub/sub unified
- Edge computing, IoT
- JetStream: persistence khi cần

Không chọn khi:
- Cần ecosystem lớn (Connect, Streams)
- Team đã invest Kafka
```

### Apache Pulsar

```
Chọn khi:
- Multi-tenancy native
- Geo-replication built-in
- Unified queue + streaming model
- Kafka migration path (Pulsar protocol)

Không chọn khi:
- Team chưa có Pulsar experience
- Ecosystem nhỏ hơn Kafka
```

---

## Migration & Hybrid

### Hybrid Architecture Phổ Biến

```
                    ┌─────────────┐
API Gateway ───────►│ Kafka       │──► Analytics, CDC, Audit
                    │ (event bus) │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ RabbitMQ │ │ SQS      │ │ Redis    │
        │ (tasks)  │ │ (lambda) │ │ (realtime)│
        └──────────┘ └──────────┘ └──────────┘

Kafka = system of record cho events
RabbitMQ/SQS = task execution
Redis = real-time state
```

### Migration Path

```
RabbitMQ → Kafka:
  1. Dual-write period
  2. Migrate consumer từng service
  3. Outbox pattern cho consistency
  4. Deprecate RabbitMQ khi all consumers migrated

SQS → Kafka:
  1. MSK/Kafka làm event backbone
  2. SQS giữ cho Lambda triggers nếu cần
```

---

## Ma Trận Use Case → Broker

| Use Case | Broker Đề Xuất | Lý Do |
| -------- | -------------- | ----- |
| Order event bus (100K/s) | Kafka | Throughput, replay, retention |
| Email background job | RabbitMQ / SQS | Task queue, DLQ |
| Payment processing | Kafka + idempotency | Audit trail, exactly-once effort |
| Real-time dashboard | Redis Streams / Kafka | Low latency vs durability |
| AWS Lambda trigger | SQS / SNS | Native integration |
| Multi-tenant SaaS | Pulsar / Kafka | Isolation, quotas |
| IoT telemetry | NATS / Kafka | Volume, edge |
| Image resize workers | RabbitMQ / SQS | Simple queue, competing consumers |
| Audit log 7 năm | Kafka (long retention) | Immutable log |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka vs RabbitMQ — chọn cái nào cho microservices?

**Đáp án mẫu:** **Kafka** nếu event bus trung tâm, cần replay, high throughput, nhiều service consume cùng event stream. **RabbitMQ** nếu task queue, routing phức tạp, RPC pattern, volume vừa phải, team muốn ops đơn giản hơn. Nhiều hệ thống dùng **cả hai** — Kafka cho events, RabbitMQ cho tasks.

### Câu 2: Khi nào dùng SQS thay vì self-hosted?

**Đáp án mẫu:** Khi trên AWS, team nhỏ, không muốn ops broker, volume unpredictable (auto-scale), Lambda integration. Trade-off: vendor lock-in, cost ở scale cao, không replay, giới hạn FIFO throughput. Self-hosted khi cần control, cost optimization ở scale lớn, hoặc multi-cloud.

### Câu 3: Redis làm message broker được không?

**Đáp án mẫu:** **Redis Pub/Sub** — fire-and-forget, không persistence. **Redis Streams** — persistence, consumer groups, ACK — phù hợp lightweight messaging. Phù hợp khi đã có Redis, latency cực thấp. Không thay Kafka cho event log dài hạn hoặc throughput hàng triệu/s với durability cao.

### Câu 4: Tiêu chí quan trọng nhất khi chọn broker?

**Đáp án mẫu:** (1) **Use case fit** — queue vs log vs stream. (2) **Team expertise** — ops capability. (3) **Operational model** — managed vs self-hosted. (4) **Ecosystem** — monitoring, connectors, client libraries. (5) **Non-functional requirements** — throughput, latency, ordering, retention. Không chọn theo hype — chọn theo constraints thực tế.

---

**Hoàn thành chủ đề 01-fundamentals.** Tiếp theo: [02-architecture-patterns/README.md](../02-architecture-patterns/README.md) — Event-Driven Architecture, Outbox, Saga.
