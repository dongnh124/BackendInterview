# Serverless Event Processing — Xử Lý Sự Kiện Serverless

> Serverless event processing (Xử Lý Sự Kiện Serverless): AWS Lambda, GCP Cloud Functions, Azure Functions kết hợp với message brokers — event triggers, scaling, cold start (Khởi Động Lạnh), cost model, và production patterns.

## Mục Lục

1. [Serverless Messaging Overview](#serverless-messaging-overview)
2. [AWS: Lambda + Messaging Services](#aws-lambda--messaging-services)
3. [GCP: Cloud Functions + Pub/Sub](#gcp-cloud-functions--pubsub)
4. [Azure: Functions + Event Hubs](#azure-functions--event-hubs)
5. [Kafka + Serverless Patterns](#kafka--serverless-patterns)
6. [Scaling & Concurrency](#scaling--concurrency)
7. [Cold Start & Performance](#cold-start--performance)
8. [Reliability & Error Handling](#reliability--error-handling)
9. [Cost Optimization](#cost-optimization)
10. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Serverless Messaging Overview

**Serverless (Không Máy Chủ)** trong messaging context: function được **trigger (kích hoạt)** bởi messages/events, cloud provider quản lý infrastructure, scale tự động, pay-per-invocation.

```
Traditional Consumer:
  Long-running pod/VM → poll Kafka 24/7 → pay for idle time

Serverless Consumer:
  Message arrives → Lambda invoked → process → terminate
  → Pay only for execution time
  → Auto-scale 0 → N concurrent executions
```

| Ưu Điểm | Nhược Điểm |
| ------- | ---------- |
| Zero ops (no servers) | Cold start latency |
| Auto-scale to zero | Max execution time limit (15 min Lambda) |
| Pay per use | Không phù hợp long-running stream processing |
| Fast deploy | VPC networking complexity (Kafka) |
| Event-native integration | Debugging/monitoring khó hơn |

### Khi Nào Dùng Serverless cho Messaging

```
✅ Spiky, unpredictable traffic (notifications, webhooks)
✅ Simple transform/enrichment per message
✅ Fan-out to multiple actions (SNS → Lambda)
✅ Low-volume event handlers
✅ Prototype / MVP nhanh

❌ High sustained throughput (100K+ msg/s) — dedicated consumers rẻ hơn
❌ Long-running aggregation/windowing — dùng Kafka Streams/Flink
❌ Strict sub-10ms latency — cold start risk
❌ Complex stateful processing
```

---

## AWS: Lambda + Messaging Services

### Integration Matrix

| AWS Service | Trigger Type | Pattern |
| ----------- | ------------ | ------- |
| **SQS** | Poll (Lambda polls queue) | Task processing |
| **SNS** | Push | Fan-out notifications |
| **Kinesis** | Poll (batch) | Stream processing |
| **MSK (Kafka)** | Poll via event source mapping | Kafka consumer |
| **EventBridge** | Push | Event routing |
| **DynamoDB Streams** | Push | CDC-like triggers |

### SQS → Lambda (Phổ Biến Nhất)

```
SQS Queue ──► Lambda Event Source Mapping ──► Lambda Function
                    │
                    ├── Batch size: 1-10 messages
                    ├── Batching window: 0-300 seconds
                    └── Partial batch failure reporting
```

```python
# lambda_function.py
import json

def handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        process_order(body)
    # Success → Lambda auto-deletes messages from SQS
    # Exception → messages return to queue (visibility timeout)
```

| Config | Mô Tả |
| ------ | ----- |
| `BatchSize` | Messages per invocation (1–10 standard, 10K FIFO) |
| `MaximumBatchingWindowInSeconds` | Wait to fill batch |
| `FunctionResponseTypes` | `ReportBatchItemFailures` — partial retry |

### SNS → Lambda Fan-Out

```
                    ┌──► Lambda: sendEmail
SNS Topic ──fan-out──┼──► Lambda: updateAnalytics
                    └──► Lambda: pushNotification
                    └──► SQS (buffer) ──► Lambda: heavyProcessing
```

**Pattern:** SNS direct cho lightweight; SQS buffer cho backpressure (Lambda concurrency limit).

### MSK (Kafka) → Lambda

```
MSK Cluster ──► Lambda Event Source Mapping ──► Lambda
                    │
                    ├── Bootstrap servers (VPC)
                    ├── Topic: orders
                    ├── Consumer group: lambda-orders-processor
                    ├── Batch size + window
                    └── Starting position: LATEST / TRIM_HORIZON
```

**Yêu cầu:**

```
□ Lambda trong VPC (same as MSK)
□ Security groups: Lambda → MSK broker ports (9092/9094/9098)
□ IAM auth hoặc SASL/SCRAM credentials in Secrets Manager
□ ENI (Elastic Network Interface) — cold start tăng do VPC
```

```yaml
# SAM / CloudFormation snippet
OrdersProcessor:
  Type: AWS::Lambda::Function
  Properties:
    VpcConfig:
      SecurityGroupIds: [!Ref LambdaSG]
      SubnetIds: [!Ref PrivateSubnets]
    Events:
      MSKEvent:
        Type: MSK
        Properties:
          Stream: !Ref MSKClusterArn
          Topics:
            - orders
          StartingPosition: LATEST
          BatchSize: 100
          MaximumBatchingWindowInSeconds: 5
```

### Lambda + SQS Best Practices

```
✅ Idempotent handler — SQS at-least-once
✅ ReportBatchItemFailures — chỉ retry failed messages
✅ DLQ on source queue — poison messages
✅ Reserved concurrency — prevent runaway scale
✅ Visibility timeout > Lambda timeout + buffer
```

---

## GCP: Cloud Functions + Pub/Sub

### Pub/Sub Push vs Pull với Cloud Functions

```
Push (Serverless native):
  Pub/Sub Topic ──push HTTP──► Cloud Function (Gen 2)
  → Pub/Sub retries on 5xx
  → Ack deadline configurable

Pull (Cloud Run / GKE):
  Pub/Sub Subscription ──pull──► Long-running consumer
  → More control, no push timeout limit
```

### Cloud Functions Gen 2 + Pub/Sub

```python
# main.py — Cloud Functions Gen 2
import functions_framework
import base64
import json

@functions_framework.cloud_event
def process_order(cloud_event):
    data = base64.b64decode(cloud_event.data["message"]["data"])
    order = json.loads(data)
    handle_order(order)
```

```yaml
# deployment
gcloud functions deploy process-order \
  --gen2 \
  --trigger-topic=orders \
  --runtime=python312 \
  --memory=256MB \
  --max-instances=100 \
  --min-instances=1
```

| Config | Mô Tả |
| ------ | ----- |
| `--min-instances` | Warm instances — giảm cold start |
| `--max-instances` | Concurrency cap |
| `--memory` | Ảnh hưởng CPU allocation |
| `--timeout` | Max 3600s (Gen 2) |

### Eventarc (Unified Event Routing)

```
Cloud Storage ──► Eventarc ──► Cloud Run / Cloud Functions
Firestore       ──► Eventarc ──► ...
Pub/Sub         ──► Eventarc ──► ...
Custom          ──► Eventarc ──► ...
```

Xem thêm: [10-cloud-managed/4-gcp-pubsub.md](../10-cloud-managed/4-gcp-pubsub.md)

---

## Azure: Functions + Event Hubs

### Event Hubs Trigger

```csharp
// Azure Function — Event Hubs trigger
[FunctionName("ProcessOrders")]
public static async Task Run(
    [EventHubTrigger("orders", Connection = "EventHubConnection",
     ConsumerGroup = "lambda-equivalent")] EventData[] events,
    ILogger log)
{
    foreach (var eventData in events)
    {
        var order = JsonSerializer.Deserialize<Order>(eventData.Body.ToString());
        await ProcessOrder(order);
    }
}
```

### Azure Functions + Service Bus

```
Service Bus Queue/Topic ──► Azure Function trigger
  ├── Auto-complete on success
  ├── Retry policy (exponential backoff)
  └── DLQ after max delivery count
```

| Feature | Event Hubs | Service Bus |
| ------- | ---------- | ----------- |
| **Throughput** | Very high (millions/s) | Moderate |
| **Pattern** | Stream / log | Queue / pub-sub |
| **Serverless trigger** | ✅ Native | ✅ Native |
| **Ordering** | Partition-based | Sessions (FIFO) |

Xem thêm: [10-cloud-managed/3-azure-event-hubs.md](../10-cloud-managed/3-azure-event-hubs.md)

---

## Kafka + Serverless Patterns

Kafka không designed cho serverless natively — cần patterns đặc biệt.

### Pattern 1: Lambda as Kafka Consumer (MSK)

```
Pros: No consumer infrastructure
Cons: VPC cold start, 15-min limit, offset commit per batch
```

**Offset management:**

```
Lambda batch process → success → commit offset
Lambda fail → không commit → reprocess batch (at-least-once)
→ Handler MUST be idempotent
```

### Pattern 2: Kafka → Kinesis → Lambda

```
MSK/K rows of Kafka Connect / custom bridge → Kinesis Data Streams → Lambda
→ Tránh VPC Lambda complexity
→ Additional hop, latency tăng
```

### Pattern 3: Kafka → SQS → Lambda

```
Custom consumer (Fargate/ECS minimal) → SQS → Lambda
→ SQS buffer decouple Kafka polling from Lambda scaling
→ Fargate consumer lightweight, Lambda scale processing
```

```
┌─────────┐    ┌──────────────┐    ┌─────────┐    ┌─────────┐
│  Kafka  │───►│ Fargate      │───►│   SQS   │───►│ Lambda  │
│  Topic  │    │ (poll+route) │    │  Queue  │    │(process)│
└─────────┘    └──────────────┘    └─────────┘    └─────────┘
```

### Pattern 4: EventBridge Pipe (AWS)

```
MSK / SQS / DynamoDB ──► EventBridge Pipes ──► Lambda / Step Functions
→ Managed integration, filtering, enrichment
→ No custom poll code
```

---

## Scaling & Concurrency

### Lambda Concurrency Model

```
Account limit: 1000 concurrent (default, tăng được)
Reserved concurrency: Guarantee/min cap per function
Provisioned concurrency: Pre-warmed instances (no cold start)

Unreserved pool: Shared across all functions
```

```
SQS queue depth 10,000 messages
Lambda concurrency 50 (reserved)
→ 50 parallel executions
→ Remaining messages wait in queue
→ Scale: tăng concurrency hoặc optimize handler duration
```

### Pub/Sub Scaling

```
Push subscription:
  Max concurrent push requests = min(max_instances, Pub/Sub limits)
  Exponential backoff on 503/429

Pull (Cloud Run):
  Autoscale based on undelivered message count (HPA — Horizontal Pod Autoscaler)
```

### Anti-Pattern: Unbounded Scale

```
❌ Lambda max concurrency = account limit
   → One function consumes all quota
   → Other functions starved

✅ Reserved concurrency per function
✅ SQS maxReceiveCount + DLQ
✅ Rate limiting ở producer hoặc SQS
```

---

## Cold Start & Performance

### Cold Start Anatomy (Lambda)

```
Request arrives (no warm instance)
  → Init runtime (Python/Node/Java)
  → Init handler code
  → VPC: create ENI (Elastic Network Interface — thêm 1-10s!)
  → Execute handler
  → Total cold start: 100ms (simple) → 10s+ (VPC + Java)
```

### Mitigation Strategies

| Strategy | Effect | Cost |
| -------- | ------ | ---- |
| **Provisioned concurrency** | Eliminate cold start | $$$ always-on |
| **Min instances** (Cloud Functions Gen 2) | Keep warm | $$ |
| **Avoid VPC** when possible | Faster cold start | — |
| **Smaller runtime** (Python/Node vs Java) | Faster init | — |
| **Lambda SnapStart** (Java) | Faster Java init | Free |
| **ARM Graviton** | Cheaper + often faster | Savings |

### Latency Budget

```
Use case: Order confirmation email
  Acceptable latency: 30 seconds
  Cold start: 2 seconds → OK ✅

Use case: Fraud detection block transaction
  Acceptable latency: 100ms
  Cold start: 2 seconds → NOT OK ❌ → use always-on consumer
```

---

## Reliability & Error Handling

### SQS + Lambda Error Flow

```
Lambda throws exception
  → SQS message NOT deleted
  → Visibility timeout expires
  → Message reappears → retry
  → After maxReceiveCount → DLQ

Partial batch failure (ReportBatchItemFailures):
  → Only failed message IDs retried
  → Successful messages deleted
```

```python
def handler(event, context):
    batch_failures = []
    for record in event['Records']:
        try:
            process(record)
        except Exception:
            batch_failures.append({"itemIdentifier": record['messageId']})
    return {"batchItemFailures": batch_failures}
```

### Pub/Sub Push Retry

```
Cloud Function returns 5xx → Pub/Sub retries với exponential backoff
Returns 2xx → Ack
Returns 4xx (non-retryable) → Drop or DLQ (dead letter topic)
```

```python
# Configure dead letter topic
gcloud pubsub subscriptions update orders-sub \
  --dead-letter-topic=orders-dlq \
  --max-delivery-attempts=5
```

### Idempotency (Bắt Buộc)

```
Serverless = at-least-once delivery
→ Duplicate invocations common (retry, timeout, rebalance)

Solution:
  dedup_key = message_id or business_id
  if cache.exists(dedup_key): return  # already processed
  process(message)
  cache.set(dedup_key, ttl=24h)
```

---

## Cost Optimization

### Cost Comparison (Rough)

```
Scenario: 1M messages/day, 200ms processing, 256MB

Lambda (SQS trigger):
  1M invocations × 200ms × 256MB ≈ $3-8/month (compute)
  + SQS requests ≈ $0.40
  Total: ~$5-10/month

Always-on Fargate (0.25 vCPU, 512MB):
  24/7 running ≈ $10-15/month

Break-even: ~2-5M messages/day depending on processing time
Below that → serverless cheaper
Above that → dedicated consumer cheaper
```

### Cost Tips

| Tip | Mô Tả |
| --- | ----- |
| **Right-size memory** | Memory = CPU — profile trước khi tăng |
| **Batch processing** | 10 SQS messages per invocation vs 10 invocations |
| **Avoid provisioned concurrency** unless SLA requires | Expensive |
| **SQS long polling** | Reduce empty receives |
| **Filter at source** | SNS filter policy, EventBridge rules — không invoke Lambda không cần |
| **ARM architecture** | 20% cheaper on AWS |

---

## Thiết Kế Thực Tế

### Reference: Order Notification Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              ORDER NOTIFICATION (Serverless)                     │
│                                                                 │
│  API Gateway ──► Lambda (validate) ──► SNS Topic "order-events" │
│                                              │                  │
│                         ┌────────────────────┼──────────────┐   │
│                         ▼                    ▼              ▼   │
│                   Lambda:email      Lambda:SMS    SQS ──►     │
│                   (lightweight)     (lightweight)  Lambda:    │
│                                                    analytics  │
│                                                    (heavy)    │
│                                                                 │
│  DLQ on SQS ──► Lambda: dlq-handler ──► alert on-call          │
└─────────────────────────────────────────────────────────────────┘
```

### Reference: MSK Event Processor

```
MSK Topic "orders"
  → Lambda (VPC, batch 100, 5s window)
  → DynamoDB (idempotency table)
  → SNS (downstream notifications)

Config:
  Reserved concurrency: 20
  Timeout: 60s
  Memory: 512MB
  DLQ: SQS orders-lambda-dlq
  Alarm: DLQ depth > 0
```

### Production Checklist

```
□ Handler idempotent (dedup table/cache)
□ DLQ configured (SQS DLQ, Pub/Sub dead letter topic)
□ Timeout < visibility timeout (SQS) / ack deadline (Pub/Sub)
□ Reserved concurrency set (prevent starvation)
□ CloudWatch/Monitoring: errors, duration, throttles, DLQ depth
□ VPC design optimized (minimal subnets, Hyperplane ENI for Lambda)
□ Partial batch failure enabled (SQS)
□ Secrets in Secrets Manager (not env vars plaintext)
□ Load test cold start impact on P99 latency
□ Cost alarm (Lambda invocations spike)
```

### Anti-Patterns

```
❌ Lambda process 100K msg/s sustained — use Kinesis/Fargate consumer
❌ Không idempotent — duplicate charges, duplicate emails
❌ Java Lambda in VPC without SnapStart — 10s cold starts
❌ No DLQ — poison messages retry forever
❌ Lambda timeout = visibility timeout — message reprocessed mid-flight
❌ Kafka complex stream join in Lambda — wrong tool
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào dùng Lambda vs long-running Kafka consumer?

**Gợi ý trả lời:** **Lambda** cho spiky/low-medium volume, simple per-message processing, SQS/SNS triggers, ops-free scaling. **Long-running consumer** cho high sustained throughput, complex stateful processing, strict low latency (no cold start), Kafka Streams. Break-even thường vài triệu messages/day.

### Câu 2: Lambda consume Kafka (MSK) challenges?

**Gợi ý trả lời:** **VPC required** → cold start tăng (ENI creation). **15-minute max** execution. **Offset commit** per batch — at-least-once, cần idempotent. **Not designed** for Kafka consumer groups natively — event source mapping limitations. Alternative: Fargate poll → SQS → Lambda.

### Câu 3: SQS partial batch failure?

**Gợi ý trả lời:** `ReportBatchItemFailures` — Lambda return failed message IDs only. SQS retry chỉ failed messages, successful deleted. Tránh reprocess entire batch khi 1 message lỗi. Requires `FunctionResponseTypes` config.

### Câu 4: Cold start mitigation?

**Gợi ý trả lời:** **Provisioned concurrency** (costly), **min instances** (Cloud Functions), **avoid VPC** if possible, **smaller runtime** (Python/Node), **SnapStart** (Java), **ARM Graviton**. Design: chấp nhận cold start nếu latency SLA > 5s.

### Câu 5: Serverless messaging reliability?

**Gợi ý trả lời:** At-least-once delivery — **idempotent handlers** bắt buộc. **DLQ** cho poison messages. **Retry** with backoff (SQS visibility, Pub/Sub push retry). **Timeout alignment** — Lambda timeout < SQS visibility timeout. Monitor DLQ depth và error rate.

### Câu 6: Thiết kế fan-out notification serverless?

**Gợi ý trả lời:** **SNS topic** fan-out → multiple Lambda (email, SMS, push). Heavy processing → **SQS buffer** → Lambda (decouple, backpressure). **Filter policies** on SNS — route by attribute. **DLQ** per queue. **Idempotency** key per notification ID.

---

**Hoàn thành chủ đề 11-advanced.** Tiếp theo: [12-interview-prep/](../12-interview-prep/) — Chuẩn bị phỏng vấn.

**Liên quan:** [10-cloud-managed/1-aws-msk.md](../10-cloud-managed/1-aws-msk.md) — MSK trên AWS.
