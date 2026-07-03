# Amazon SQS & SNS — Hàng Đợi và Thông Báo Trên AWS

> Amazon SQS (Simple Queue Service — Dịch Vụ Hàng Đợi Đơn Giản) và SNS (Simple Notification Service — Dịch Vụ Thông Báo Đơn Giản): Standard vs FIFO queue, visibility timeout, dead letter queue, và SNS fan-out pattern cho kiến trúc event-driven trên AWS.

## Mục Lục

1. [Tổng Quan SQS & SNS](#tổng-quan-sqs--sns)
2. [SQS Standard vs FIFO](#sqs-standard-vs-fifo)
3. [Visibility Timeout](#visibility-timeout)
4. [Dead Letter Queue (DLQ)](#dead-letter-queue-dlq)
5. [SNS Fan-Out Pattern](#sns-fan-out-pattern)
6. [Long Polling & Batching](#long-polling--batching)
7. [Tích Hợp Lambda & EventBridge](#tích-hợp-lambda--eventbridge)
8. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
9. [So Sánh Với Kafka/RabbitMQ](#so-sánh-với-kafkarabbitmq)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan SQS & SNS

### Amazon SQS

**Point-to-Point (Điểm-Điểm)** message queue — managed, serverless, không cần provision broker.

```
Producer ──► SQS Queue ──► Consumer(s)
                  │
                  └── Message lưu đến khi xử lý xong + delete
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Managed** | AWS vận hành toàn bộ — zero ops |
| **Durability** | Message replicate across AZ (Availability Zone — Vùng Sẵn Sàng) |
| **Scale** | Auto-scale, không giới hạn queue depth |
| **Delivery** | At-least-once (Standard), Exactly-once processing effort (FIFO + dedup) |
| **Retention** | 1 phút – 14 ngày (default 4 ngày) |

### Amazon SNS

**Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký)** — publish message đến topic, fan-out đến nhiều subscribers.

```
Publisher ──► SNS Topic ──┬──► SQS Queue A ──► Service A
                          ├──► SQS Queue B ──► Service B
                          ├──► Lambda
                          ├──► HTTP/HTTPS endpoint
                          └──► Email, SMS, Mobile push
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Fan-out** | 1 message → nhiều destination |
| **Protocols** | SQS, Lambda, HTTP, email, SMS |
| **FIFO Topic** | Ordering + dedup khi cần |
| **Filtering** | Subscription filter policy — route theo message attributes |

---

## SQS Standard vs FIFO

### SQS Standard Queue

```
✅ Throughput: gần unlimited
✅ At-least-once delivery
❌ Không đảm bảo ordering (best-effort)
❌ Có thể duplicate message
```

**Phù hợp:** Background jobs, decoupling services, volume cao, consumer idempotent.

### SQS FIFO Queue

```
✅ Strict ordering trong MessageGroupId
✅ Exactly-once processing (deduplication)
⚠️ Throughput: 300 msg/s per queue (3000 với high throughput mode + batching)
✅ MessageDeduplicationId — content-based hoặc explicit
```

**FIFO naming:** Queue name phải kết thúc `.fifo` (ví dụ: `orders.fifo`).

### MessageGroupId & Deduplication

```json
{
  "MessageBody": "{\"orderId\": \"123\", \"action\": \"created\"}",
  "MessageGroupId": "order-123",
  "MessageDeduplicationId": "order-123-created-v1"
}
```

| Field | Ý Nghĩa |
| ----- | ------- |
| **MessageGroupId** | Messages cùng group được xử lý tuần tự (ordering) |
| **MessageDeduplicationId** | Trong 5 phút, message trùng ID bị loại bỏ |
| **Content-based dedup** | FIFO queue có thể dedup theo body hash (bật khi tạo queue) |

### Khi Nào Chọn FIFO

```
Chọn FIFO khi:
- Cần strict ordering per entity (order per customer, per account)
- Cần deduplication native
- Throughput < 300–3000 msg/s per queue

Chọn Standard khi:
- Throughput cao
- Ordering không quan trọng hoặc xử lý ở application layer
- Consumer đã idempotent
```

---

## Visibility Timeout

**Visibility Timeout (Thời Gian Ẩn Hiện)** — sau khi consumer nhận message, message bị **ẩn** khỏi queue trong khoảng thời gian này. Nếu consumer không delete trước khi timeout hết → message **visible lại** cho consumer khác.

```
Timeline:
  T0: Consumer A receive message → message INVISIBLE
  T1: Consumer A đang xử lý...
  T2: Visibility timeout hết, Consumer A chưa delete
  T3: Message VISIBLE lại → Consumer B có thể nhận (DUPLICATE!)
```

### Cấu Hình

| Tham Số | Giá Trị | Ghi Chú |
| ------- | ------- | ------- |
| **Min** | 0 giây | |
| **Max** | 12 giờ | |
| **Default** | 30 giây | Thường cần tăng cho job dài |

```python
# AWS SDK — extend visibility timeout khi xử lý lâu
import boto3

sqs = boto3.client('sqs')
response = sqs.receive_message(QueueUrl=queue_url, MaxNumberOfMessages=1)
receipt_handle = response['Messages'][0]['ReceiptHandle']

# Xử lý job dài...
sqs.change_message_visibility(
    QueueUrl=queue_url,
    ReceiptHandle=receipt_handle,
    VisibilityTimeout=300  # extend thêm 5 phút
)

# Xong → delete
sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
```

### Best Practices

```
1. Visibility timeout > thời gian xử lý trung bình (có buffer)
2. Dùng heartbeat pattern — extend timeout định kỳ cho long-running job
3. Idempotent consumer — message có thể được xử lý 2 lần
4. Monitor ApproximateAgeOfOldestMessage — message stuck quá lâu
```

---

## Dead Letter Queue (DLQ)

**DLQ (Dead Letter Queue — Hàng Đợi Thư Chết)** — queue nhận message không xử lý được sau N lần thử.

```
Main Queue ──► Consumer (fail) ──► receive count++
     │
     └── maxReceiveCount exceeded ──► DLQ
```

### Cấu Hình Redrive Policy

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:region:account:orders-dlq",
  "maxReceiveCount": 3
}
```

| maxReceiveCount | Ý Nghĩa |
| --------------- | ------- |
| 1 | Message vào DLQ ngay lần fail đầu |
| 3–5 | Cho phép retry tự nhiên (visibility timeout) |
| 10+ | Cẩn thận — poison message retry nhiều lần |

### DLQ Operations

```
1. Monitor DLQ depth — alert khi > 0
2. Inspect message — xem body, attributes, lý do fail
3. Fix root cause
4. Redrive — chuyển message từ DLQ về main queue (AWS Console hoặc API)
5. Purge — xóa hàng loạt khi đã xử lý manual
```

> **Lưu ý:** SQS không có built-in exponential backoff như RabbitMQ TTL+DLX. Retry = visibility timeout + receive lại. Cần **idempotency** và **maxReceiveCount** hợp lý.

---

## SNS Fan-Out Pattern

### Pattern Cơ Bản

```
                    ┌─────────────────┐
Order Service ─────►│  SNS Topic      │
                    │  "order-events" │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ SQS: email  │  │ SQS: inventory│ │ SQS: analytics│
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           ▼                 ▼                 ▼
      Email Service    Inventory Svc    Analytics Svc
```

**Lợi ích:**

- **Decoupling (Tách Rời)** — publisher không biết subscribers
- **Independent scaling** — mỗi queue scale riêng
- **Failure isolation** — một service down không block others (message queue riêng)
- **Buffering** — SQS buffer khi consumer chậm

### Subscription Filter Policy

```json
{
  "eventType": ["order.created", "order.cancelled"],
  "region": ["ap-southeast-1"]
}
```

Chỉ message match filter mới đến subscription — giảm noise cho consumer.

### SNS FIFO + SQS FIFO

```
SNS FIFO Topic (.fifo) ──► SQS FIFO Queue (.fifo)
  - Ordering preserved end-to-end
  - MessageGroupId propagate từ SNS → SQS
  - Cần cả topic và queue đều FIFO
```

---

## Long Polling & Batching

### Long Polling (ReceiveMessageWaitTimeSeconds)

```
Short polling (0s): Trả về ngay, có thể empty — tốn API calls
Long polling (1–20s): Chờ message đến — giảm empty receives, tiết kiệm cost
```

**Khuyến nghị:** `WaitTimeSeconds=20` cho hầu hết consumers.

### Batching

| API | Batch Size | Lợi Ích |
| --- | ---------- | ------- |
| **SendMessageBatch** | Tối đa 10 messages | Giảm API calls khi publish |
| **ReceiveMessage** | MaxNumberOfMessages=10 | Xử lý batch, tăng throughput |
| **DeleteMessageBatch** | Tối đa 10 | Xóa hàng loạt sau xử lý |

```python
# Receive batch
messages = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20,
    VisibilityTimeout=60
)
```

---

## Tích Hợp Lambda & EventBridge

### SQS → Lambda

```
SQS Queue ──► Lambda Event Source Mapping ──► Lambda function
                    │
                    ├── Batch size: 1–10 (FIFO: 1–10, Standard: 1–10)
                    ├── Batching window
                    └── Partial batch failure (reportBatchItemFailures)
```

| Cấu Hình | Ghi Chú |
| -------- | ------- |
| **Batch size** | Tăng throughput, cần xử lý partial failure |
| **Reserved concurrency** | Giới hạn Lambda instances từ queue này |
| **DLQ on Lambda** | Lambda async có DLQ riêng — khác SQS DLQ |

### SNS → Lambda

Direct trigger — SNS gọi Lambda khi message publish. Không có buffering như SQS — cần Lambda scale đủ nhanh hoặc dùng SNS → SQS → Lambda.

### EventBridge

**EventBridge (Sự Kiện Cầu Nối)** — event bus với routing rules phức tạp hơn SNS:

```
EventBridge Rule: source=order.service AND detail-type=OrderCreated
  → Target: SQS, Lambda, Step Functions, ...
```

So sánh: **SNS** đơn giản fan-out; **EventBridge** schema registry, content-based routing, cross-account.

---

## Thiết Kế Thực Tế

### Pattern: Order Processing Pipeline

```
┌──────────────┐     ┌─────────────┐     ┌──────────────────┐
│ API Gateway  │────►│ SNS Topic   │────►│ SQS: fulfillment │
│ POST /orders │     │ order-events│     │ SQS: notification│
└──────────────┘     └─────────────┘     │ SQS: analytics   │
                                         └────────┬─────────┘
                                                  │
                              ┌───────────────────┼───────────────────┐
                              ▼                   ▼                   ▼
                         ECS/Fargate          Lambda              Kinesis
                         (fulfillment)        (email)            (analytics)
```

### Checklist Production

```
□ Queue encryption (SSE-SQS hoặc KMS)
□ DLQ configured với maxReceiveCount
□ Visibility timeout phù hợp job duration
□ IAM least privilege — producer/consumer roles riêng
□ CloudWatch alarms: ApproximateNumberOfMessagesVisible, DLQ depth
□ Idempotent consumers
□ FIFO chỉ khi thực sự cần ordering
□ Cost monitoring — API requests + data transfer
```

### Cost Considerations

| Factor | Impact |
| ------ | ------ |
| **API requests** | Mỗi send/receive/delete = 1 request — batch để giảm |
| **Data transfer** | Cross-region, cross-AZ |
| **Long polling** | Giảm empty receives → giảm cost |
| **FIFO premium** | Đắt hơn Standard |

---

## So Sánh Với Kafka/RabbitMQ

| Tiêu Chí | SQS/SNS | Kafka | RabbitMQ |
| -------- | ------- | ----- | -------- |
| **Ops** | Zero | Cao | Trung bình |
| **Replay** | ❌ | ✅ | ❌ (queue) |
| **Ordering** | FIFO (limited) | Per partition | Per queue |
| **Throughput** | Cao (managed) | Rất cao | Trung bình |
| **Routing** | SNS filter | Topic + key | Exchange |
| **Latency** | ms–s | ms | ms |
| **Multi-cloud** | AWS only | Any | Any |

**Chọn SQS/SNS khi:** AWS-native, serverless, không cần replay, team nhỏ.

**Không chọn khi:** Cần event log replay, multi-cloud, complex stream processing.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Visibility timeout hoạt động thế nào? Message duplicate khi nào?

**Đáp án mẫu:** Consumer receive message → message invisible trong visibility timeout. Nếu không delete trước timeout → visible lại, consumer khác nhận → **duplicate**. Xảy ra khi: xử lý lâu hơn timeout, consumer crash, không extend visibility. Fix: tăng timeout, heartbeat extend, idempotent consumer.

### Câu 2: SNS fan-out vs Kafka consumer groups?

**Đáp án mẫu:** **SNS fan-out** — mỗi subscriber (SQS queue) nhận **copy** message, độc lập scale. **Kafka consumer groups** — mỗi group đọc toàn bộ topic, partition chia trong group. SNS+SQS = push đến queue riêng; Kafka = pull từ shared log. SNS phù hợp AWS decoupling; Kafka phù hợp replay và high-throughput streaming.

### Câu 3: SQS FIFO throughput limit — xử lý thế nào?

**Đáp án mẫu:** 300 msg/s per queue (3000 high throughput). Giải pháp: (1) **Sharding** — nhiều FIFO queue với MessageGroupId phân tán (per customer/region). (2) **Standard queue** + ordering ở application nếu chấp nhận best-effort. (3) **Kafka MSK** nếu cần ordering + throughput cao.

### Câu 4: Thiết kế DLQ cho SQS?

**Đáp án mẫu:** Redrive policy `maxReceiveCount=3–5`. DLQ riêng per main queue. CloudWatch alarm DLQ depth > 0. Runbook: inspect → fix → redrive hoặc discard. Không retry vô hạn — poison message tốn resource. Kết hợp idempotency key trong message body.

### Câu 5: Khi nào dùng EventBridge thay vì SNS?

**Đáp án mẫu:** **EventBridge** khi cần: content-based routing phức tạp, schema registry, cross-account/cross-region event bus, integration nhiều AWS services (Step Functions, API Destinations). **SNS** khi: simple fan-out, ít routing rules, cost-sensitive, pattern SNS→SQS→Lambda đủ dùng.

---

**Xem tiếp:** [2-redis-streams.md](./2-redis-streams.md) — Redis Pub/Sub vs Streams.
