# Dead Letter Handling — Vận Hành DLQ

> Dead Letter Queue (DLQ — Hàng Đợi Thư Chết) là **safety net (Lưới An Toàn)** cuối cùng khi message không xử lý được sau retries. Thiết kế và vận hành DLQ đúng cách quyết định hệ thống **fail gracefully (Thất Bại Có Kiểm Soát)** hay **fail silently (Thất Bại Im Lặng)**.

## Mục Lục

1. [DLQ Là Gì và Tại Sao Cần](#dlq-là-gì-và-tại-sao-cần)
2. [Thiết Kế DLQ Cross-Broker](#thiết-kế-dlq-cross-broker)
3. [Khi Nào Message Vào DLQ](#khi-nào-message-vào-dlq)
4. [DLQ Operations](#dlq-operations)
5. [Replay Procedure](#replay-procedure)
6. [Monitoring & Alerting](#monitoring--alerting)
7. [Runbook Template](#runbook-template)
8. [Thiết Kế Production](#thiết-kế-production)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DLQ Là Gì và Tại Sao Cần

**Dead Letter Queue (DLQ — Hàng Đợi Thư Chết)** — queue/topic riêng lưu message **không xử lý thành công** sau retries, để phân tích, replay, hoặc discard thay vì block main queue.

```
Main Queue ──fail (max retries)──► DLQ
                                      │
                                      ├── Manual review
                                      ├── Replay (sau fix)
                                      └── Discard (poison)
```

| Không Có DLQ | Có DLQ |
| ------------ | ------ |
| Poison message block queue vô thời hạn | Message cô lập, main queue tiếp tục |
| Message fail im lặng | Có audit trail, có thể replay |
| Khó debug production issues | Tập trung failed messages để phân tích |
| Không có recovery path | Replay sau khi fix root cause |

---

## Thiết Kế DLQ Cross-Broker

### RabbitMQ — Dead Letter Exchange (DLX)

```
Main Queue ──nack(requeue=false)──► DLX (Dead Letter Exchange)
                                        │
                                        └──► DLQ
```

```javascript
// Cấu hình DLX trên main queue
await channel.assertQueue('orders.process', {
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'orders.dlx',
    'x-dead-letter-routing-key': 'orders.failed',
  },
});

await channel.assertExchange('orders.dlx', 'direct', { durable: true });
await channel.assertQueue('orders.dlq', { durable: true });
await channel.bindQueue('orders.dlq', 'orders.dlx', 'orders.failed');
```

Xem chi tiết: [04-rabbitmq/3-dead-letter-queue.md](../04-rabbitmq/3-dead-letter-queue.md)

### Kafka — DLQ Topic

Kafka không có native DLQ — dùng **dedicated topic**:

```javascript
// Consumer publish failed message sang DLQ topic
async function handleFailure(message, error) {
  await producer.send({
    topic: 'orders.dlq',
    messages: [{
      key: message.key,
      value: message.value,
      headers: {
        'x-original-topic': 'orders',
        'x-error-message': error.message,
        'x-failed-at': new Date().toISOString(),
        'x-retry-count': String(retryCount),
      },
    }],
  });
  await commitOffset(message);  // Skip message trên main topic
}
```

**Best practice:** Một DLQ topic per business domain, hoặc `{main-topic}.dlq`.

### Amazon SQS — Redrive Policy

```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:region:account:orders-dlq",
    "maxReceiveCount": 5
  }
}
```

SQS tự động chuyển message sang DLQ sau `maxReceiveCount` lần receive mà không delete.

### Redis Streams — Không Có Native DLQ

```javascript
// Manual: XADD vào dlq stream khi fail
await redis.xadd('orders:dlq', '*',
  'original_id', messageId,
  'payload', JSON.stringify(payload),
  'error', error.message
);
await redis.xack('orders', groupName, messageId);  // Ack trên main stream
```

### So Sánh DLQ Theo Broker

| Broker | DLQ Mechanism | Auto DLQ | Headers/Metadata |
| ------ | ------------- | -------- | ---------------- |
| **RabbitMQ** | DLX + DLQ | Có (khi cấu hình) | x-death, x-first-death-* |
| **Kafka** | DLQ topic | Không (app logic) | Custom headers |
| **SQS** | Redrive policy | Có | Receive count |
| **Redis Streams** | Manual stream | Không | Custom fields |

---

## Khi Nào Message Vào DLQ

### Trigger Conditions

| Trigger | Mô Tả | Broker |
| ------- | ----- | ------ |
| **Max retries exceeded** | Retry count >= max | Tất cả |
| **Permanent error** | Validation, 400, schema mismatch | App logic |
| **Consumer reject** | nack(requeue=false) | RabbitMQ |
| **TTL expired** | Message/queue TTL hết | RabbitMQ |
| **Delivery limit** | x-delivery-limit exceeded | RabbitMQ quorum |
| **maxReceiveCount** | SQS receive count | SQS |

### Decision Flow

```
Message processing failed
    │
    ├── Permanent error? ──YES──► DLQ ngay
    │
    ├── Transient error?
    │       │
    │       ├── retryCount < max? ──NO──► DLQ
    │       │
    │       └── YES ──► Retry (không vào DLQ)
    │
    └── Unknown ──► Retry limited ──► DLQ
```

---

## DLQ Operations

### 1. Inspect (Kiểm Tra)

```bash
# RabbitMQ — xem DLQ depth
rabbitmqctl list_queues name messages | grep dlq

# Kafka — consume từ DLQ topic (không commit nếu chỉ inspect)
kafka-console-consumer --topic orders.dlq --from-beginning --max-messages 10

# SQS — receive từ DLQ
aws sqs receive-message --queue-url https://sqs.../orders-dlq
```

### 2. Classify (Phân Loại)

| Category | Mô Tả | Hành Động |
| -------- | ----- | --------- |
| **Fixable** | Bug đã fix, data có thể sửa | Replay sau deploy |
| **Upstream issue** | Producer gửi sai format | Notify upstream, replay sau fix |
| **Poison** | Message luôn fail | Quarantine, discard |
| **One-off** | Lỗi tạm thời đã hết | Replay ngay |

### 3. Replay (Phát Lại)

Xem [Replay Procedure](#replay-procedure) bên dưới.

### 4. Purge (Xóa)

```bash
# RabbitMQ — purge DLQ (cẩn thận!)
rabbitmqctl purge_queue orders.dlq

# Chỉ purge sau khi: (1) đã replay thành công, hoặc (2) quyết định discard
```

### 5. Archive (Lưu Trữ)

```
DLQ message → Export to S3/Blob → Purge DLQ
  (compliance, audit, post-mortem)
```

---

## Replay Procedure

**Replay (Phát Lại)** — gửi message từ DLQ về main queue/topic để xử lý lại.

### Pre-Replay Checklist

```
□ Root cause đã fix? (deploy, config, upstream)
□ Consumer idempotent? (replay an toàn)
□ Replay batch nhỏ trước (10-100 messages)
□ Monitor error rate trong replay
□ Có rollback plan nếu replay gây vấn đề
```

### Replay Implementation

```javascript
// RabbitMQ — replay từ DLQ về main queue
async function replayFromDLQ(dlqName, mainQueueName, limit = 100) {
  let replayed = 0;

  await channel.consume(dlqName, async (msg) => {
    if (replayed >= limit) {
      channel.nack(msg, false, true);  // Requeue, stop
      return;
    }

    const headers = {
      ...msg.properties.headers,
      'x-retry-count': 0,           // Reset retry counter
      'x-replayed-at': new Date().toISOString(),
      'x-replay-source': 'dlq',
    };

    channel.sendToQueue(mainQueueName, msg.content, {
      headers,
      persistent: true,
    });
    channel.ack(msg);  // Remove from DLQ
    replayed++;
  }, { noAck: false });
}
```

```javascript
// Kafka — replay từ DLQ topic về main topic
async function replayFromDLQTopic(dlqTopic, mainTopic, limit) {
  const consumer = kafka.consumer({ groupId: 'dlq-replay-worker' });
  await consumer.subscribe({ topic: dlqTopic });

  let count = 0;
  await consumer.run({
    eachMessage: async ({ message }) => {
      if (count >= limit) return;

      await producer.send({
        topic: mainTopic,
        messages: [{
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            'x-retry-count': '0',
            'x-replayed-at': new Date().toISOString(),
          },
        }],
      });
      count++;
    },
  });
}
```

### Replay Strategies

| Strategy | Mô Tả | Khi Dùng |
| -------- | ----- | -------- |
| **Full replay** | Replay tất cả DLQ | Root cause fixed, batch nhỏ |
| **Selective replay** | Replay theo filter (time, error type) | Một phần fixable |
| **Staged replay** | Batch 100 → monitor → tiếp | DLQ lớn, cần cẩn thận |
| **No replay** | Discard sau analysis | Poison, không fixable |

---

## Monitoring & Alerting

### Metrics Cần Theo Dõi

| Metric | Ngưỡng Alert | Ý Nghĩa |
| ------ | ------------ | ------- |
| **DLQ depth** | > 0 sustained (5+ phút) | Có message fail cần xử lý |
| **DLQ growth rate** | > N/phút | Bug mới hoặc upstream issue |
| **DLQ age** | Message > 24h | Chưa được xử lý |
| **Replay success rate** | < 95% | Replay có vấn đề |
| **Death reason breakdown** | Spike permanent errors | Schema/validation issue |

### Dashboard Widgets

```
┌─────────────────────────────────────────────────────────┐
│  DLQ Dashboard                                           │
│                                                          │
│  orders.dlq depth:     42  ▲ +12 (1h)                   │
│  payments.dlq depth:    0  ✓                             │
│                                                          │
│  Death reasons (24h):                                    │
│    validation_error:  28                                 │
│    timeout:           10                                 │
│    unknown:            4                                 │
│                                                          │
│  Oldest message: 2h 15m ago                              │
└─────────────────────────────────────────────────────────┘
```

### Alert Rules

```yaml
alerts:
  - name: DLQDepthNonZero
    condition: dlq_messages > 0 for 5m
    severity: warning
    runbook: /runbooks/dlq-investigation

  - name: DLQDepthCritical
    condition: dlq_messages > 100
    severity: critical
    runbook: /runbooks/dlq-incident

  - name: DLQGrowthSpike
    condition: rate(dlq_messages[5m]) > 10
    severity: warning
```

---

## Runbook Template

### DLQ Investigation Runbook

```markdown
## DLQ Alert: {queue-name}.dlq depth > 0

### 1. Triage (5 phút)
- [ ] Check DLQ depth và growth rate
- [ ] Identify time range message bắt đầu vào DLQ
- [ ] Check recent deploys, config changes

### 2. Sample & Classify (15 phút)
- [ ] Peek 5-10 messages từ DLQ
- [ ] Đọc error headers (x-death, x-error-message)
- [ ] Phân loại: fixable / poison / upstream

### 3. Action
- **Fixable:** Fix root cause → staged replay
- **Poison:** Quarantine → Jira ticket → discard
- **Upstream:** Notify team → wait for fix → replay

### 4. Post-Incident
- [ ] Update runbook nếu cần
- [ ] Post-mortem nếu impact lớn
```

---

## Thiết Kế Production

### Naming Convention

```
Main queue:    orders.process
DLQ:           orders.process.dlq
Retry queue:   orders.process.retry.30s
```

### Checklist

```
□ Mỗi business-critical queue có DLQ riêng
□ DLQ naming: {main-queue}.dlq
□ Alert khi DLQ depth > 0
□ Runbook: ai xử lý, SLA, replay procedure
□ Log death headers khi message vào DLQ
□ Max retry + exponential backoff trước DLQ
□ Idempotent consumer — replay an toàn
□ Dashboard: DLQ depth, death reason
□ Archive policy cho compliance
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề |
| ------------ | ------ |
| Không có DLQ | Poison block queue, silent loss |
| DLQ chung cho tất cả | Khó debug, khó prioritize |
| Không monitor DLQ | Message fail im lặng |
| Replay không idempotent | Duplicate side effects |
| Purge DLQ thường xuyên | Che giấu recurring issues |
| Replay hàng loạt không test | Gây incident mới |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: DLQ depth = 0 có nghĩa hệ thống healthy không?

**Trả lời:** Không hoàn toàn. DLQ=0 có thể do: (1) Thực sự không có lỗi. (2) Message bị **drop** thay vì DLQ. (3) DLQ bị purge che giấu vấn đề. Cần monitor cả **consumer error rate** và **retry count**.

### Câu 2: Replay từ DLQ an toàn cần điều kiện gì?

**Trả lời:** (1) **Idempotent consumer**. (2) **Root cause đã fix**. (3) Reset retry counter. (4) Staged replay — batch nhỏ, monitor. (5) Audit log messageId.

### Câu 3: Kafka DLQ khác RabbitMQ thế nào?

**Trả lời:** RabbitMQ có **native DLX** — broker tự chuyển message. Kafka dùng **DLQ topic** — application publish failed message. Kafka cần implement logic và metadata headers tự quản lý.

### Câu 4: Ai chịu trách nhiệm xử lý DLQ?

**Trả lời:** Tùy tổ chức. Thường: **On-call engineer** investigate alert, **Service owner** quyết định replay/discard, **Upstream team** nếu data issue. Cần **runbook** và **SLA** rõ ràng (ví dụ: DLQ xử lý trong 4h).

### Câu 5: DLQ message lưu bao lâu?

**Trả lời:** Tùy retention policy. Thường **7–30 ngày** trước archive/purge. Compliance có thể yêu cầu lưu lâu hơn — export sang cold storage (S3) trước purge.

---

**Xem tiếp:** [4-poison-message.md](./4-poison-message.md) — Phát hiện và xử lý poison message.
