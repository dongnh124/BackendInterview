# Poison Message — Tin Nhắn Độc

> **Poison Message (Tin Nhắn Độc)** — message luôn fail khi xử lý, gây infinite retry loop (Vòng Lặp Thử Lại Vô Hạn) nếu không được cô lập. Đây là một trong những incident phổ biến nhất trong messaging production.

## Mục Lục

1. [Poison Message Là Gì](#poison-message-là-gì)
2. [Triệu Chứng Nhận Biết](#triệu-chứng-nhận-biết)
3. [Nguyên Nhân Gốc Rễ](#nguyên-nhân-gốc-rễ)
4. [Phòng Ngừa](#phòng-ngừa)
5. [Detection (Phát Hiện)](#detection-phát-hiện)
6. [Quarantine Pattern (Mẫu Cách Ly)](#quarantine-pattern-mẫu-cách-ly)
7. [Root Cause Analysis (RCA)](#root-cause-analysis-rca)
8. [Thiết Kế Production](#thiết-kế-production)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Poison Message Là Gì

**Poison Message (Tin Nhắn Độc)** — message **không thể xử lý thành công** dù retry bao nhiêu lần, thường do:
- Payload invalid (Dữ liệu không hợp lệ)
- Bug trong consumer code
- Missing dependency (resource không tồn tại)
- Schema mismatch (Không khớp schema)

```
Consumer nhận message A
  → Process fail
  → Requeue / Retry
  → Process fail (cùng message A)
  → Requeue / Retry
  → ... infinite loop
  → Block toàn bộ queue (head-of-line blocking)
```

### Hậu Quả

| Hậu Quả | Mô Tả |
| ------- | ----- |
| **Queue blocking** | Message poison ở đầu queue, các message sau không được xử lý |
| **Resource waste** | CPU, memory cho retry vô ích |
| **Consumer lag spike** | Lag tăng vì consumer bận retry cùng message |
| **Cascade failure** | Retry storm làm quá tải downstream |
| **Silent data loss** | Message khác timeout, expire |

---

## Triệu Chứng Nhận Biết

### Dấu Hiệu Trong Logs

```
# Cùng messageId xuất hiện liên tục
ERROR processing messageId=abc-123: ValidationError: invalid orderId
ERROR processing messageId=abc-123: ValidationError: invalid orderId
ERROR processing messageId=abc-123: ValidationError: invalid orderId
... (hàng trăm lần trong vài phút)
```

### Dấu Hiệu Trong Metrics

| Metric | Pattern | Ý Nghĩa |
| ------ | ------- | ------- |
| **Queue depth** | Không giảm dù consumer chạy | Poison ở đầu queue |
| **Consumer lag** | Tăng đột ngột | Blocked by poison |
| **Error rate** | 100% cho 1 message | Poison candidate |
| **Retry count** | x-retry-count > 10 | Đã retry nhiều |
| **CPU usage** | Spike | Tight retry loop |

### Head-of-Line Blocking

```
Queue: [POISON] [msg2] [msg3] [msg4] [msg5]
         │
         └── Consumer stuck retry POISON
             msg2, msg3, msg4, msg5 chờ vô thời hạn
```

**Giải pháp:** DLQ poison ngay, hoặc dùng **multiple queues** / **partitioning** để poison không block toàn bộ.

---

## Nguyên Nhân Gốc Rễ

### Phân Loại Nguyên Nhân

| Category | Ví Dụ | Fixable? |
| -------- | ----- | -------- |
| **Data corruption** | Invalid JSON, null required field | Có — fix upstream |
| **Schema drift** | Producer gửi v2, consumer expect v1 | Có — deploy consumer mới |
| **Missing reference** | orderId không tồn tại trong DB | Có — data fix hoặc skip |
| **Code bug** | NullPointerException trên edge case | Có — deploy fix |
| **Config error** | Wrong API endpoint, wrong credentials | Có — fix config |
| **Business rule** | Order đã cancelled, không xử lý được | Có — handle gracefully |

### Timeline Điển Hình

```
T+0:   Upstream deploy gửi payload mới (thiếu field X)
T+1m:  Consumer bắt đầu fail
T+5m:  Queue depth tăng, alert firing
T+15m: On-call investigate
T+30m: Root cause: schema change
T+45m: Rollback upstream HOẶC deploy consumer fix
T+1h:  Replay từ DLQ
```

---

## Phòng Ngừa

### 1. Max Retry Limit

```javascript
const MAX_RETRIES = 5;

if (retryCount >= MAX_RETRIES) {
  await sendToDLQ(msg, { reason: 'max_retries_exceeded' });
  return;  // Không retry nữa
}
```

### 2. Error Classification

```javascript
// Permanent error → DLQ ngay, không retry
if (isPermanentError(err)) {
  await sendToDLQ(msg, { reason: 'permanent_error' });
  return;
}
```

### 3. Broker-Level Delivery Limit

```javascript
// RabbitMQ quorum queue
await channel.assertQueue('orders.process', {
  arguments: {
    'x-queue-type': 'quorum',
    'x-delivery-limit': 5,  // Auto DLQ sau 5 deliveries
    'x-dead-letter-exchange': 'orders.dlx',
  },
});
```

### 4. Schema Validation Sớm

```javascript
// Validate TRƯỚC khi xử lý business logic
const parsed = schema.safeParse(msg.body);
if (!parsed.success) {
  await sendToDLQ(msg, { reason: 'schema_validation_failed' });
  return;  // Không retry — permanent
}
```

### 5. Idempotency + Graceful Degradation

```javascript
// Resource not found — có thể skip thay vì fail
const order = await db.orders.findById(orderId);
if (!order) {
  logger.warn(`Order ${orderId} not found, skipping`);
  return;  // Ack — không poison
}
```

---

## Detection (Phát Hiện)

### Automated Detection

```javascript
// Track retry count per messageId
const retryTracker = new Map();  // messageId → count

async function processWithPoisonDetection(msg) {
  const messageId = msg.headers['message-id'];
  const count = (retryTracker.get(messageId) ?? 0) + 1;
  retryTracker.set(messageId, count);

  if (count > POISON_THRESHOLD) {
    await alertPoisonCandidate(messageId, count);
    await sendToDLQ(msg, { reason: 'poison_detected' });
    retryTracker.delete(messageId);
    return;
  }

  try {
    await process(msg);
    retryTracker.delete(messageId);
  } catch (err) {
    throw err;
  }
}
```

### Alert Rules

```yaml
- name: PoisonMessageCandidate
  condition: |
    count by (message_id) (
      rate(consumer_errors[5m]) > 0
    ) > 5
  severity: critical
  message: "Message {{ $labels.message_id }} failed 5+ times in 5m"
```

### DLQ Headers Inspection

```javascript
// RabbitMQ x-death header
const deaths = msg.properties.headers['x-death'];
const deathCount = deaths?.[0]?.count ?? 0;
const reason = deaths?.[0]?.reason;  // 'rejected', 'expired', 'maxlen'

if (deathCount > 3) {
  // Poison candidate — escalate
}
```

---

## Quarantine Pattern (Mẫu Cách Ly)

**Quarantine (Cách Ly)** — tách poison message ra khỏi main flow, phân tích riêng, quyết định fix/replay/discard.

```
Main Queue ──fail──► DLQ ──► Quarantine Service
                                    │
                                    ├── Inspect payload
                                    ├── Classify: fixable / poison / unknown
                                    │
                                    ├── Fixable ──► Fix ──► Replay to main
                                    ├── Poison ──► Jira ticket ──► Discard
                                    └── Unknown ──► Manual review queue
```

### Quarantine Service

```javascript
async function quarantineProcessor(dlqMessage) {
  const classification = await classifyMessage(dlqMessage);

  switch (classification) {
    case 'fixable':
      // Có script/tool fix data
      const fixed = await fixPayload(dlqMessage);
      await replayToMain(fixed);
      break;

    case 'poison':
      await createJiraTicket(dlqMessage);
      await archiveToS3(dlqMessage);
      await purgeFromDLQ(dlqMessage);
      break;

    case 'unknown':
      await moveToManualReviewQueue(dlqMessage);
      await notifyOnCall(dlqMessage);
      break;
  }
}
```

### Manual Review Queue

```
DLQ ──► Quarantine ──► Manual Review Queue
                              │
                              ▼
                         Dashboard cho ops team
                         - View payload
                         - View error
                         - Actions: Replay / Discard / Escalate
```

---

## Root Cause Analysis (RCA)

**RCA (Root Cause Analysis — Phân Tích Nguyên Nhân Gốc)** — quy trình tìm và fix nguyên nhân poison message.

### RCA Checklist

```
□ Thu thập: message payload, error stack, headers (x-death, retry count)
□ Timeline: khi nào bắt đầu fail? deploy nào gần đó?
□ Reproduce: có thể replay 1 message và reproduce lỗi?
□ Scope: 1 message hay batch? 1 queue hay nhiều?
□ Upstream: producer có thay đổi? schema migration?
□ Downstream: dependency có outage?
```

### RCA Template (Post-Mortem)

```markdown
## Incident: Poison Message Blocked Order Queue

### Summary
- Duration: 45 minutes
- Impact: 1,200 orders delayed
- Root cause: Upstream deploy sent payload without required field `customerId`

### Timeline
- 14:00: Upstream deploy v2.1
- 14:05: First DLQ messages
- 14:10: Alert: queue depth spike
- 14:25: On-call identified poison pattern
- 14:35: Upstream rollback
- 14:45: Replay from DLQ completed

### Root Cause
Schema change in upstream without consumer compatibility check.

### Action Items
- [ ] Add schema validation at consumer entry
- [ ] Contract testing between producer/consumer
- [ ] DLQ alert threshold review
```

### Prevention Sau RCA

| Action | Mục Đích |
| ------ | -------- |
| Schema validation at ingress | Catch invalid payload sớm |
| Contract testing (Pact, etc.) | Đảm bảo producer/consumer compatible |
| Canary deploy | Phát hiện poison sớm trước full rollout |
| DLQ alert | Phát hiện nhanh khi message vào DLQ |
| Max retry + DLQ | Tránh infinite loop |

---

## Thiết Kế Production

### Checklist

```
□ Max retries (3-5) trước DLQ
□ Permanent error → DLQ ngay
□ Schema validation trước business logic
□ x-delivery-limit (RabbitMQ) hoặc maxReceiveCount (SQS)
□ Alert khi cùng messageId fail > 3 lần
□ Quarantine workflow documented
□ RCA template cho team
□ Idempotent consumer — replay an toàn
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề |
| ------------ | ------ |
| Requeue vô hạn | Infinite loop, block queue |
| Không phân loại lỗi | Retry permanent errors vô ích |
| Không có DLQ | Poison block forever |
| Ignore DLQ | Recurring poison không được fix |
| Replay poison without fix | Lặp lại incident |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Poison message khác failed message thế nào?

**Trả lời:** **Failed message** — fail một hoặc vài lần, có thể retry thành công. **Poison message** — **luôn fail**, retry vô ích. Poison cần **cô lập vào DLQ** ngay, không retry thêm.

### Câu 2: Làm sao tránh poison block toàn bộ queue?

**Trả lời:** (1) **Max retry + DLQ** — đưa poison ra khỏi main queue. (2) **Multiple queues/partitions** — poison 1 partition không block partition khác. (3) **Don't use single-threaded FIFO** cho mixed workloads nếu có thể.

### Câu 3: Message "order not found" có phải poison không?

**Trả lời:** Tùy context. Nếu **permanent** (order đã xóa) — handle gracefully, ack, không retry. Nếu **transient** (replication lag) — retry vài lần. Nếu **data issue** — DLQ để investigate. Phân loại lỗi quyết định hành động.

### Câu 4: Kể incident poison message (STAR)?

**Trả lời mẫu:** "Upstream deploy schema change, consumer fail validation. Queue depth tăng 10x trong 15 phút. Tôi identify cùng messageId trong logs, check DLQ headers, coordinate rollback upstream, replay 500 messages từ DLQ. Thêm schema validation và contract test để prevent recurrence."

### Câu 5: x-delivery-limit vs manual retry count?

**Trả lời:** **x-delivery-limit** — broker-enforced, đếm mọi delivery kể cả requeue. **Manual retry count** — app quản lý trong headers, linh hoạt hơn (phân loại lỗi, backoff). Có thể **kết hợp** — broker limit là safety net, app logic cho fine control.

---

**Xem tiếp:** [5-circuit-breaker-consumers.md](./5-circuit-breaker-consumers.md) — Circuit breaker cho message consumers.
