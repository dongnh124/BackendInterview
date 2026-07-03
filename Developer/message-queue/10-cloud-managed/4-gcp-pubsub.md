# GCP Pub/Sub — Messaging Serverless Trên Google Cloud

> Google Cloud Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký): fully managed, serverless messaging, push vs pull subscriptions, ordering keys, dead letter topics, và tích hợp GCP ecosystem (Cloud Run, Dataflow, BigQuery).

## Mục Lục

1. [Tổng Quan GCP Pub/Sub](#tổng-quan-gcp-pubsub)
2. [Topics, Subscriptions & Message Flow](#topics-subscriptions--message-flow)
3. [Push vs Pull Subscriptions](#push-vs-pull-subscriptions)
4. [Ordering Keys & Message Ordering](#ordering-keys--message-ordering)
5. [Delivery Guarantees & Acknowledgment](#delivery-guarantees--acknowledgment)
6. [Dead Letter Topics](#dead-letter-topics)
7. [Schema Validation](#schema-validation)
8. [Security & IAM](#security--iam)
9. [Tích Hợp GCP Ecosystem](#tích-hợp-gcp-ecosystem)
10. [So Sánh Với Kafka & SQS](#so-sánh-với-kafka--sqs)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan GCP Pub/Sub

**Google Cloud Pub/Sub** là fully managed, serverless **messaging middleware (lớp trung gian nhắn tin)** — không cần provision brokers, partitions, hay clusters.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     GCP PUB/SUB ARCHITECTURE                                 │
│                                                                             │
│  ┌──────────────┐     ┌─────────────┐     ┌──────────────────────────┐      │
│  │  Publishers  │────►│   Topic     │────►│  Subscriptions           │      │
│  │  (any client │     │  (logical   │     │  ┌────────┐ ┌────────┐  │      │
│  │   /service)  │     │   channel)  │     │  │ Pull   │ │ Push   │  │      │
│  └──────────────┘     └─────────────┘     │  │ sub    │ │ sub    │  │      │
│                                              │  └───┬────┘ └───┬────┘  │      │
│                                              └──────┼──────────┼───────┘      │
│                                                     │          │              │
│                                              ┌──────▼──┐  ┌────▼─────┐        │
│                                              │ Worker  │  │ Cloud Run│        │
│                                              │ (pull)  │  │ (push)   │        │
│                                              └─────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Serverless** | Auto-scale, không quản lý infrastructure |
| **Global** | Topics/subscriptions trong project, multi-region |
| **At-least-once** | Default delivery — cần idempotent consumers |
| **Retention** | 7 ngày (default), max 31 ngày |
| **Throughput** | Auto-scale — không cần provision TU/partitions |
| **Pricing** | Per message + egress |

**Khác Kafka fundamentally:** Pub/Sub không có partition concept — Google quản lý scaling internally. Bạn không chọn partition count.

---

## Topics, Subscriptions & Message Flow

### Core Concepts

| Concept | Mô Tả |
| ------- | ----- |
| **Topic** | Named resource publishers gửi messages đến |
| **Subscription** | Named resource consumers nhận messages từ topic |
| **Message** | Data + optional attributes + ordering key |
| **Ack (Acknowledgment — Xác Nhận)** | Consumer báo đã xử lý xong — message bị xóa |
| **Nack (Negative Acknowledgment — Từ Chối Xác Nhận)** | Consumer báo fail — message redeliver |

### Fan-Out Pattern

```
                    ┌──► Subscription A ──► Service A (analytics)
Publisher ──► Topic ─┼──► Subscription B ──► Service B (notifications)
                    └──► Subscription C ──► BigQuery (warehouse)

Mỗi subscription nhận COPY of every message — independent delivery
```

### Message Attributes

```json
{
  "data": "base64-encoded-payload",
  "attributes": {
    "eventType": "OrderCreated",
    "correlationId": "abc-123",
    "source": "order-service"
  },
  "orderingKey": "customer-456",
  "messageId": "1234567890",
  "publishTime": "2026-07-03T10:00:00Z"
}
```

**Attributes** dùng cho routing/filtering ở application layer — Pub/Sub không có built-in content-based routing như RabbitMQ exchanges.

---

## Push vs Pull Subscriptions

### Pull Subscription

Consumer **chủ động request** messages từ Pub/Sub.

```
┌─────────────┐     pull()      ┌─────────────┐
│  Consumer   │◄───────────────►│  Pub/Sub    │
│  (worker,   │     ack/nack    │  Subscription│
│   Dataflow) │                 └─────────────┘
└─────────────┘

✅ Consumer control pacing — backpressure natural
✅ Batch pull — up to 1000 messages
✅ Phù hợp: long-running workers, Dataflow, custom apps
```

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_id)

def callback(message):
    print(f"Received: {message.data}")
    process(message.data)
    message.ack()  # Xác nhận xử lý thành công

streaming_pull_future = subscriber.subscribe(subscription_path, callback=callback)
streaming_pull_future.result()
```

### Push Subscription

Pub/Sub **gửi messages đến endpoint** consumer (HTTP POST).

```
┌─────────────┐     HTTP POST   ┌─────────────┐
│  Pub/Sub    │────────────────►│  Cloud Run  │
│  Subscription│◄────────────────│  / App Engine│
└─────────────┘     200 OK ack  └─────────────┘

✅ Không cần polling code — event-driven
✅ Tích hợp Cloud Run, App Engine, Cloud Functions
⚠️ Endpoint phải respond nhanh (< 10 min ack deadline)
⚠️ Cần handle duplicate deliveries
```

```yaml
# Push subscription config
pushConfig:
  pushEndpoint: https://my-service-abc.run.app/process
  oidcToken:
    serviceAccountEmail: pubsub-invoker@project.iam.gserviceaccount.com
    audience: https://my-service-abc.run.app
```

### Push vs Pull Decision

| Tiêu Chí | Pull | Push |
| -------- | ---- | ---- |
| **Hosting** | Any (GCE, GKE, on-prem) | HTTP endpoint required |
| **Scaling** | Consumer-managed | Pub/Sub pushes → auto with Cloud Run |
| **Latency** | Polling interval | Immediate push |
| **Backpressure** | Consumer controls pull rate | Ack deadline + flow control |
| **Use case** | Batch processing, Dataflow | Serverless (Cloud Run, Functions) |

---

## Ordering Keys & Message Ordering

### Message Ordering

Mặc định Pub/Sub **không guarantee ordering**. Enable ordering với **ordering key**:

```
Publisher gửi messages cùng orderingKey → delivered in order per key
Khác orderingKey → không guarantee order relative to each other
```

```python
# Publish với ordering key
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_id)

future = publisher.publish(
    topic_path,
    data=b'{"orderId": "123", "status": "created"}',
    ordering_key="order-123"  # Tất cả events order-123 → ordered
)
future.result()
```

### Enable Ordering on Subscription

```
Subscription setting: enableMessageOrdering = true

⚠️ Ordering reduces throughput — single thread per ordering key
⚠️ Nếu message nack — subsequent messages cùng key bị hold (head-of-line blocking)
```

### Head-of-Line Blocking

```
Ordering key "order-123":
  msg1 → processing... (slow)
  msg2 → WAIT (blocked behind msg1)
  msg3 → WAIT

Mitigation:
• Keep processing fast
• Short ack deadline with extend
• Split hot keys nếu ordering không cần strict per entity
```

---

## Delivery Guarantees & Acknowledgment

### At-Least-Once Delivery

```
Pub/Sub guarantee: message delivered ít nhất 1 lần
→ Duplicate possible → consumer PHẢI idempotent
```

### Ack Deadline

**Ack deadline (Thời Hạn Xác Nhận)** — thời gian consumer phải ack trước khi message redeliver:

| Setting | Default | Range |
| ------- | ------- | ----- |
| **Ack deadline** | 10 giây | 10 giây – 600 giây (10 phút) |

```python
# Extend ack deadline khi processing lâu
subscriber.modify_ack_deadline(
    request={
        "subscription": subscription_path,
        "ack_ids": [ack_id],
        "ack_deadline_seconds": 300  # extend thêm 5 phút
    }
)
```

### Flow Control (Pull)

```python
flow_control = pubsub_v1.types.FlowControl(
    max_messages=100,        # Max unacked messages
    max_bytes=10 * 1024 * 1024  # Max 10 MB unacked
)

subscriber.subscribe(
    subscription_path,
    callback=callback,
    flow_control=flow_control
)
```

### Retry Policy

```
Subscription retry policy:
• Minimum backoff: 10s
• Maximum backoff: 600s
• Exponential backoff between redeliveries

Sau max delivery attempts → Dead Letter Topic
```

---

## Dead Letter Topics

**Dead Letter Topic (DLT — Chủ Đề Thư Chết)** — messages fail repeatedly được chuyển sang topic riêng để investigate.

```
Main Topic ──► Subscription ──► Consumer (fail x5)
                    │
                    └──► Dead Letter Topic ──► Manual review / replay tool
```

### Configuration

```yaml
deadLetterPolicy:
  deadLetterTopic: projects/my-project/topics/orders-dlt
  maxDeliveryAttempts: 5
```

```python
# Subscribe to DLT để inspect failed messages
def dlt_callback(message):
    log.error(f"DLT message: {message.data}, delivery_attempt={message.delivery_attempt}")
    # Alert, store for manual replay, or auto-fix
    message.ack()
```

**Best practice:** Monitor DLT subscription message count — alert on any ingress.

Chi tiết DLQ patterns: [06-reliability/3-dead-letter-handling.md](../06-reliability/3-dead-letter-handling.md)

---

## Schema Validation

Pub/Sub hỗ trợ **schema validation** — reject messages không conform schema.

### Schema Types

| Format | Support |
| ------ | ------- |
| **Avro** | ✅ |
| **Protocol Buffers** | ✅ |
| **JSON** | ✅ (limited) |

```
1. Create schema in Pub/Sub Schema Service
2. Attach schema to topic
3. Publisher encode per schema
4. Pub/Sub validate on publish — reject invalid
5. Consumer decode with same schema
```

```python
# Publish binary-encoded Avro
from google.cloud.pubsub import SchemaServiceClient

# Schema defined in Pub/Sub console or API
# encoding: JSON hoặc BINARY
publisher.publish(topic_path, data=avro_encoded_bytes)
```

---

## Security & IAM

### IAM Roles

| Role | Permission |
| ---- | ---------- |
| **roles/pubsub.publisher** | Publish to topics |
| **roles/pubsub.subscriber** | Consume from subscriptions |
| **roles/pubsub.admin** | Full management |
| **roles/pubsub.viewer** | Read-only |

### Least Privilege Example

```
order-service SA:
  • pubsub.publisher on topic "orders"
  • KHÔNG có subscriber permission

notification-worker SA:
  • pubsub.subscriber on subscription "orders-notifications"
  • KHÔNG có publisher permission
```

### Encryption

| Layer | Option |
| ----- | ------ |
| **In-transit** | TLS (automatic) |
| **At-rest** | Google-managed keys (default) hoặc CMEK (Customer-Managed Encryption Keys — Khóa Mã Hóa Do Khách Quản Lý) |

### VPC Service Controls

```
Perimeter security — restrict Pub/Sub access to resources within VPC Service Controls boundary
Phù hợp: regulated data, prevent data exfiltration
```

---

## Tích Hợp GCP Ecosystem

### Cloud Run (Push)

```
Pub/Sub Push ──► Cloud Run service
• Auto-scale instances theo message rate
• OIDC authentication — Pub/Sub calls as service account
• Idempotent handler required
```

### Cloud Functions (Eventarc)

```python
# Cloud Functions Gen 2 — Pub/Sub trigger
import functions_framework

@functions_framework.cloud_event
def process_order(cloud_event):
    data = base64.b64decode(cloud_event.data["message"]["data"])
    order = json.loads(data)
    handle_order(order)
```

### Dataflow (Streaming ETL)

```
Pub/Sub ──► Dataflow pipeline ──► BigQuery / Cloud Storage / another Pub/Sub topic

Use case: real-time ETL, windowed aggregations, complex transforms
```

### BigQuery Subscription

**BigQuery Subscription** — write messages trực tiếp vào BigQuery table, không cần consumer code:

```
Topic ──► BigQuery Subscription ──► BigQuery Table
• Auto-create table schema from message
• At-least-once delivery to BigQuery
• Use case: analytics, audit log, event warehouse
```

### Pub/Sub Lite

**Pub/Sub Lite (Nhẹ)** — lower cost alternative cho high-volume:

| | Pub/Sub | Pub/Sub Lite |
| - | ------- | ------------ |
| **Cost** | Higher | Lower |
| **Ops** | Fully serverless | Zonal resources (partitions) |
| **Throughput** | Unlimited auto | Partition-based |
| **Use case** | General purpose | High volume, cost-sensitive, ordering per partition |

---

## So Sánh Với Kafka & SQS

| Tiêu Chí | GCP Pub/Sub | Amazon SQS | Apache Kafka |
| -------- | ----------- | ---------- | ------------ |
| **Model** | Pub/Sub native | Queue | Event log |
| **Partitions** | Hidden (managed) | N/A (FIFO has groups) | Explicit |
| **Replay** | ✅ (retention window) | ❌ | ✅ (full log) |
| **Ordering** | Ordering keys | FIFO queues | Partition key |
| **Serverless** | ✅ Native | ✅ Native | ❌ (managed options) |
| **Stream processing** | Dataflow | Lambda | Kafka Streams, Flink |
| **Protocol** | gRPC/REST | AWS API | Kafka protocol |

### Khi Chọn Pub/Sub

```
✅ GCP-native application stack
✅ Serverless-first (Cloud Run, Functions)
✅ Fan-out to multiple services
✅ BigQuery analytics pipeline
✅ Không cần Kafka ecosystem (Connect, ksqlDB)

Chọn Kafka/MSK thay Pub/Sub khi:
⚠️ Cần Kafka API compatibility
⚠️ Complex stream processing với Kafka Streams
⚠️ Long retention replay với partition control
⚠️ Multi-cloud portability via Kafka protocol
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Pub/Sub khác Kafka? | Serverless, no partitions to manage; at-least-once; GCP-native |
| Push vs Pull? | Push = HTTP to endpoint; Pull = consumer polls — trade-offs |
| Ordering keys hoạt động thế nào? | Same key → ordered delivery; enable on subscription |
| Head-of-line blocking? | Slow/failed message blocks subsequent same-key messages |
| Ack deadline là gì? | Time to ack before redelivery — extend for long processing |
| Dead letter topic? | Failed messages after max attempts → separate topic for investigation |
| Pub/Sub vs Pub/Sub Lite? | Lite = lower cost, partition-based, zonal — high volume |
| Idempotency tại sao bắt buộc? | At-least-once delivery → duplicates on retry |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Delivery guarantees | [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md) |
| Idempotency | [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md) |
| DLQ handling | [06-reliability/3-dead-letter-handling.md](../06-reliability/3-dead-letter-handling.md) |
| Amazon SQS/SNS | [05-other-brokers/1-amazon-sqs-sns.md](../05-other-brokers/1-amazon-sqs-sns.md) |
| Azure Event Hubs | [3-azure-event-hubs.md](./3-azure-event-hubs.md) |
| Cloud overview | [README.md](./README.md) |

---

**Cập Nhật:** 2026-07-03
