# Retry & Exponential Backoff — Chiến Lược Thử Lại

> Retry (Thử Lại) là lớp bảo vệ đầu tiên cho transient failure (Lỗi Tạm Thời). Thiết kế sai — retry vô hạn, không backoff, không phân loại lỗi — gây **retry storm (Bão Thử Lại)** và làm trầm trọng outage.

## Mục Lục

1. [Retry Là Gì và Khi Nào Dùng](#retry-là-gì-và-khi-nào-dùng)
2. [Phân Loại Lỗi: Transient vs Permanent](#phân-loại-lỗi-transient-vs-permanent)
3. [Exponential Backoff (Lùi Lũy Tiến)](#exponential-backoff-lùi-lũy-tiến)
4. [Jitter (Nhiễu Ngẫu Nhiên)](#jitter-nhiễu-ngẫu-nhiên)
5. [Retry Budget & Max Attempts](#retry-budget--max-attempts)
6. [Triển Khai Theo Broker](#triển-khai-theo-broker)
7. [Retry Storm Prevention](#retry-storm-prevention)
8. [Thiết Kế Production](#thiết-kế-production)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Retry Là Gì và Khi Nào Dùng

**Retry (Thử Lại)** — tự động gửi lại message hoặc thực hiện lại operation khi gặp lỗi **có khả năng thành công** ở lần sau.

```
Attempt 1 ──fail──► Wait ──► Attempt 2 ──fail──► Wait ──► Attempt 3 ──success ✓
```

| Khi Nên Retry | Khi KHÔNG Retry |
| ------------- | --------------- |
| Network timeout | Validation error (400) |
| DB connection refused | Schema mismatch |
| 503 Service Unavailable | Resource not found (404) |
| Rate limit (429) với Retry-After | Business rule violation |
| Temporary lock contention | Malformed payload |

---

## Phân Loại Lỗi: Transient vs Permanent

**Transient Error (Lỗi Tạm Thời)** — lỗi có thể tự hết, retry có cơ hội thành công.

**Permanent Error (Lỗi Vĩnh Viễn)** — lỗi không tự hết, retry vô ích → đưa thẳng vào DLQ.

```javascript
function classifyError(err) {
  // Permanent — không retry
  if (err instanceof ValidationError) return 'permanent';
  if (err.statusCode === 400 || err.statusCode === 404) return 'permanent';
  if (err.code === 'INVALID_PAYLOAD') return 'permanent';

  // Transient — retry
  if (err.code === 'ECONNREFUSED') return 'transient';
  if (err.code === 'ETIMEDOUT') return 'transient';
  if (err.statusCode === 503 || err.statusCode === 429) return 'transient';

  // Unknown — conservative: retry với limit, sau đó DLQ
  return 'unknown';
}
```

### Decision Tree

```
Error xảy ra
    │
    ├── Permanent? ──YES──► DLQ ngay (không retry)
    │
    ├── Transient? ──YES──► Retry với backoff
    │       │
    │       ├── retryCount < maxRetries? ──YES──► Schedule retry
    │       │
    │       └── NO ──► DLQ
    │
    └── Unknown? ──► Retry 1-2 lần → DLQ nếu vẫn fail
```

| Loại Lỗi | Ví Dụ | Hành Động |
| -------- | ----- | --------- |
| **Transient** | Connection timeout, 503 | Retry + backoff |
| **Permanent** | Invalid JSON, 400 Bad Request | DLQ ngay |
| **Unknown** | Unhandled exception | Retry limited → DLQ |

---

## Exponential Backoff (Lùi Lũy Tiến)

**Exponential Backoff (Lùi Lũy Tiến)** — tăng thời gian chờ giữa các lần retry theo cấp số nhân, giảm áp lực lên hệ thống đang gặp sự cố.

### Công Thức

```
delay = min(baseDelay × 2^attempt, maxDelay)

Ví dụ: baseDelay = 1s, maxDelay = 60s
  Attempt 0: 1s
  Attempt 1: 2s
  Attempt 2: 4s
  Attempt 3: 8s
  Attempt 4: 16s
  Attempt 5: 32s
  Attempt 6: 60s (capped)
```

### Implementation

```javascript
function calculateBackoff(attempt, options = {}) {
  const {
    baseDelayMs = 1000,
    maxDelayMs = 60000,
    multiplier = 2,
  } = options;

  const delay = Math.min(
    baseDelayMs * Math.pow(multiplier, attempt),
    maxDelayMs
  );
  return delay;
}

async function retryWithBackoff(fn, maxRetries = 5) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries || classifyError(err) === 'permanent') {
        throw err;
      }
      const delay = calculateBackoff(attempt);
      await sleep(delay);
    }
  }
}
```

### So Sánh Chiến Lược Backoff

| Chiến Lược | Công Thức | Ưu Điểm | Nhược Điểm |
| ---------- | --------- | ------- | ---------- |
| **Fixed (Cố Định)** | delay = 5s | Đơn giản | Không giảm load khi outage dài |
| **Linear (Tuyến Tính)** | delay = attempt × 5s | Dễ hiểu | Tăng chậm |
| **Exponential (Lũy Tiến)** | delay = 2^attempt × base | Giảm load nhanh | Có thể chờ lâu |
| **Exponential + Jitter** | delay + random | Tránh thundering herd | Phức tạp hơn chút |

---

## Jitter (Nhiễu Ngẫu Nhiên)

**Jitter (Nhiễu Ngẫu Nhiên)** — thêm randomness vào delay để tránh **thundering herd (Bầy Đàn)** — nhiều consumer retry cùng lúc.

### Vấn Đề Không Có Jitter

```
10 consumers fail cùng lúc (DB restart)
  → Tất cả retry sau đúng 4 giây
  → 10 requests đồng thời hit DB
  → DB overload → fail tiếp → retry storm
```

### Full Jitter

```javascript
// AWS recommended: random trong khoảng [0, delay]
function calculateBackoffWithJitter(attempt, options = {}) {
  const delay = calculateBackoff(attempt, options);
  return Math.floor(Math.random() * delay);  // Full jitter
}

// Ví dụ: attempt 3, delay = 8s → random 0-8s
```

### Equal Jitter

```javascript
// Giữ ít nhất half delay, random phần còn lại
function equalJitter(attempt, options = {}) {
  const delay = calculateBackoff(attempt, options);
  return delay / 2 + Math.random() * (delay / 2);
}
```

| Jitter Type | Công Thức | Khi Dùng |
| ----------- | --------- | -------- |
| **Full jitter** | random(0, delay) | Nhiều consumers, high concurrency |
| **Equal jitter** | delay/2 + random(0, delay/2) | Cần minimum wait time |
| **Decorrelated jitter** | random(base, prevDelay × 3) | AWS SDK style |

---

## Retry Budget & Max Attempts

**Retry Budget (Ngân Sách Thử Lại)** — giới hạn tổng số retry trong khoảng thời gian, tránh retry storm làm trầm trọng outage.

### Max Attempts Khuyến Nghị

| Use Case | Max Retries | Tổng Thời Gian Chờ (ước tính) |
| -------- | ----------- | ----------------------------- |
| Real-time notification | 2–3 | ~30s |
| Order processing | 3–5 | ~2–5 phút |
| Batch/ETL | 5–10 | ~30 phút – 1 giờ |
| Critical financial | 3 + DLQ + manual | Case by case |

### Retry Header Tracking

```javascript
// Track retry count trong message headers
const retryCount = msg.headers['x-retry-count'] ?? 0;
const maxRetries = 5;

if (retryCount >= maxRetries) {
  await sendToDLQ(msg);
  return;
}

// Schedule retry với incremented count
await scheduleRetry(msg, {
  headers: { 'x-retry-count': retryCount + 1 },
  delay: calculateBackoffWithJitter(retryCount),
});
```

### Retry Budget (Rate Limiting Retries)

```javascript
// Giới hạn: tối đa 100 retries/phút cho toàn service
const retryBudget = new TokenBucket({ capacity: 100, refillRate: 100, refillInterval: 60000 });

async function scheduleRetry(msg) {
  if (!retryBudget.tryConsume()) {
    // Budget exhausted — đưa vào DLQ hoặc delay lâu hơn
    await sendToDLQ(msg, { reason: 'retry_budget_exhausted' });
    return;
  }
  await enqueueRetry(msg);
}
```

---

## Triển Khai Theo Broker

### RabbitMQ — Retry Queue + TTL + DLX

```
Main Queue ──fail──► Retry Queue (TTL: 30s) ──expire──► Main Queue
       │ max retries
       ▼
      DLQ
```

```javascript
// Retry queue với TTL
await channel.assertQueue('orders.retry', {
  durable: true,
  arguments: {
    'x-message-ttl': 30000,
    'x-dead-letter-exchange': '',
    'x-dead-letter-routing-key': 'orders.process',
  },
});
```

Xem chi tiết: [04-rabbitmq/3-dead-letter-queue.md](../04-rabbitmq/3-dead-letter-queue.md)

### Kafka — Không Có Native Retry Queue

Kafka **không redeliver** message failed — consumer phải:
1. **Không commit offset** → message sẽ được đọc lại (blocking)
2. **Publish sang retry topic** → consumer riêng xử lý sau delay
3. **Dùng external scheduler** (SQS delay, Redis delayed queue)

```javascript
// Pattern: Retry topic
async function handleMessage(message) {
  try {
    await process(message);
    await commitOffset(message);
  } catch (err) {
    if (isTransient(err) && retryCount < MAX) {
      await producer.send({
        topic: 'orders.retry',
        messages: [{
          key: message.key,
          value: message.value,
          headers: { 'x-retry-count': String(retryCount + 1) },
        }],
      });
      await commitOffset(message);  // Skip main topic
    } else {
      await producer.send({ topic: 'orders.dlq', messages: [...] });
      await commitOffset(message);
    }
  }
}
```

### Amazon SQS — Visibility Timeout

```javascript
// SQS tự động retry khi không delete message
// Visibility timeout = thời gian message "ẩn" sau khi receive

// Tăng visibility timeout cho long-running task
await sqs.changeMessageVisibility({
  QueueUrl: queueUrl,
  ReceiptHandle: receiptHandle,
  VisibilityTimeout: 300,  // 5 phút
});

// Sau maxReceiveCount → tự động vào DLQ (cấu hình trên queue)
```

---

## Retry Storm Prevention

**Retry Storm (Bão Thử Lại)** — hàng loạt retry đồng thời làm quá tải hệ thống đang recovery.

### Nguyên Nhân

```
1. Không có backoff — retry ngay lập tức
2. Không có jitter — tất cả retry cùng lúc
3. Không giới hạn max retries
4. Retry permanent errors
5. Không có circuit breaker
```

### Biện Pháp Phòng Ngừa

| Biện Pháp | Mô Tả |
| --------- | ----- |
| **Exponential backoff** | Tăng delay giữa các lần retry |
| **Jitter** | Phân tán thời điểm retry |
| **Max retries** | Dừng sau N lần → DLQ |
| **Error classification** | Không retry permanent errors |
| **Circuit breaker** | Dừng consume khi downstream down |
| **Retry budget** | Rate limit tổng retries |
| **Retry queue isolation** | Tách retry khỏi main queue |

```
┌─────────────────────────────────────────────────────────┐
│  RETRY STORM PREVENTION CHECKLIST                        │
│                                                          │
│  □ Exponential backoff (không fixed delay)             │
│  □ Jitter (full hoặc equal)                            │
│  □ Max retries (3-5 typical)                           │
│  □ Phân loại transient vs permanent                    │
│  □ Circuit breaker cho downstream                      │
│  □ Monitor retry rate                                  │
└─────────────────────────────────────────────────────────┘
```

---

## Thiết Kế Production

### Retry Policy Template

```yaml
retry_policy:
  max_attempts: 5
  base_delay_ms: 1000
  max_delay_ms: 60000
  multiplier: 2
  jitter: full
  
  retryable_errors:
    - ECONNREFUSED
    - ETIMEDOUT
    - 503
    - 429
  
  non_retryable_errors:
    - 400
    - 404
    - ValidationError
```

### Checklist

```
□ Phân loại lỗi: transient vs permanent
□ Exponential backoff với jitter
□ Max retries (3-5) trước DLQ
□ Track retry count trong headers
□ Retry queue/topic riêng (không block main)
□ Alert khi retry rate spike
□ Idempotent consumer — retry an toàn
□ Document retry policy trong runbook
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề |
| ------------ | ------ |
| Requeue ngay (requeue=true) | Tight loop, CPU spike |
| Retry vô hạn | Block queue, waste resources |
| Retry permanent errors | Vô ích, delay DLQ |
| Không jitter | Thundering herd |
| Retry trong sync handler | Block consumer, không scale |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Exponential backoff tại sao cần jitter?

**Trả lời:** Nhiều consumers fail cùng lúc (ví dụ DB restart) sẽ retry sau **cùng delay** nếu không có jitter → **thundering herd** — đồng loạt hit DB khi nó vừa recovery → fail tiếp. Jitter phân tán retry trong thời gian.

### Câu 2: Retry bao nhiêu lần trước khi vào DLQ?

**Trả lời:** Thường **3–5 lần** với exponential backoff. Phụ thuộc SLA: real-time cần ít hơn (~3), batch có thể nhiều hơn (~10). Quan trọng: **permanent errors** vào DLQ ngay, không retry.

### Câu 3: Kafka retry khác RabbitMQ thế nào?

**Trả lời:** RabbitMQ có **requeue** và **retry queue + TTL + DLX** built-in. Kafka **không redeliver** failed message — phải không commit offset (block) hoặc publish sang **retry topic** riêng. Kafka cần design retry pattern ở application layer.

### Câu 4: Retry-After header (429) xử lý thế nào?

**Trả lời:** Khi API trả `429 Too Many Requests` với `Retry-After: 60`, nên **respect header** — delay ít nhất 60 giây trước retry. Không dùng backoff mặc định nếu API chỉ định thời gian cụ thể.

### Câu 5: Làm sao test retry logic?

**Trả lời:** (1) **Chaos testing** — kill downstream, inject latency. (2) **Unit test** classifyError, calculateBackoff. (3) **Integration test** — mock transient failure, verify retry count và DLQ. (4) Monitor retry metrics trong staging.

---

**Xem tiếp:** [3-dead-letter-handling.md](./3-dead-letter-handling.md) — DLQ design, replay, và vận hành.
