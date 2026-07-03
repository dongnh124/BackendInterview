# Circuit Breaker cho Message Consumers — Cầu Dao Bảo Vệ

> **Circuit Breaker (Cầu Dao Bảo Vệ)** — pattern bảo vệ consumer và downstream khi dependency fail liên tục. Khác với retry (xử lý từng message), circuit breaker **dừng consume** để tránh cascade failure (Lỗi Dây Chuyền) và waste resources.

## Mục Lục

1. [Circuit Breaker Là Gì](#circuit-breaker-là-gì)
2. [Ba Trạng Thái](#ba-trạng-thái)
3. [Tích Hợp Với Message Consumer](#tích-hợp-với-message-consumer)
4. [Cấu Hình & Tuning](#cấu-hình--tuning)
5. [Bulkhead Pattern (Mẫu Vách Ngăn)](#bulkhead-pattern-mẫu-vách-ngăn)
6. [So Sánh Với Retry và DLQ](#so-sánh-với-retry-và-dlq)
7. [Implementation](#implementation)
8. [Thiết Kế Production](#thiết-kế-production)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Circuit Breaker Là Gì

**Circuit Breaker (Cầu Dao Bảo Vệ)** — pattern ngăn gọi dependency đang fail, tương tự cầu dao điện ngắt khi quá tải.

```
Consumer ──► Circuit Breaker ──► Downstream (DB, API)
                  │
                  ├── CLOSED: Gọi bình thường
                  ├── OPEN: Từ chối gọi, fail fast
                  └── HALF-OPEN: Thử 1 request để test recovery
```

| Không Có Circuit Breaker | Có Circuit Breaker |
| ------------------------ | ---------------- |
| Consumer tiếp tục gọi API down | Fail fast, không waste resources |
| Retry storm làm trầm trọng outage | Dừng consume, chờ recovery |
| Queue depth tăng vô hạn | Pause consume, message giữ trong broker |
| Downstream không có thời gian recovery | Thời gian recovery cho dependency |

---

## Ba Trạng Thái

### State Machine

```
                    ┌─────────────┐
                    │   CLOSED    │  ← Bình thường, gọi downstream
                    │  (Đóng)     │
                    └──────┬──────┘
                           │ failure threshold exceeded
                           ▼
                    ┌─────────────┐
         ┌─────────│    OPEN     │  ← Từ chối mọi request
         │         │   (Mở)      │
         │         └──────┬──────┘
         │                │ timeout elapsed
         │                ▼
         │         ┌─────────────┐
         │         │ HALF-OPEN   │  ← Cho phép 1 request thử
         │         │ (Nửa Mở)    │
         │         └──────┬──────┘
         │                │
         │    success     │ failure
         └────────────────┘
```

| State | Hành Vi | Chuyển Sang |
| ----- | ------- | ----------- |
| **CLOSED (Đóng)** | Gọi downstream bình thường | OPEN khi failure rate > threshold |
| **OPEN (Mở)** | Fail fast, không gọi downstream | HALF-OPEN sau timeout |
| **HALF-OPEN (Nửa Mở)** | Cho 1 request thử | CLOSED nếu success, OPEN nếu fail |

### Ví Dụ Timeline

```
T+0:   Downstream API healthy, circuit CLOSED
T+1m:  API bắt đầu trả 503
T+2m:  5/10 requests fail → circuit OPEN
T+2m+: Consumer fail fast, không gọi API
T+5m:  Timeout 3 phút → circuit HALF-OPEN
T+5m+: 1 test request → success → circuit CLOSED
T+6m:  Resume normal processing
```

---

## Tích Hợp Với Message Consumer

### Vấn Đề: Consumer + Circuit Breaker

Message consumer khác HTTP client — **message vẫn trong queue** khi circuit OPEN. Cần quyết định:

| Strategy | Hành Vi | Message Fate |
| -------- | ------- | ------------ |
| **Pause consume** | Dừng poll/consume | Giữ trong broker |
| **Nack + requeue** | Reject message | Quay lại queue (cẩn thận retry storm) |
| **Don't ack** | Không commit offset | Kafka redeliver sau |
| **Defer to retry queue** | Gửi sang retry topic | Xử lý sau khi circuit CLOSED |

### Recommended: Pause Consume

```javascript
const circuitBreaker = new CircuitBreaker(callDownstream, {
  timeout: 5000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
});

circuitBreaker.on('open', () => {
  logger.warn('Circuit OPEN — pausing consumer');
  consumer.pause();  // Dừng poll
});

circuitBreaker.on('halfOpen', () => {
  logger.info('Circuit HALF-OPEN — resuming consumer for test');
  consumer.resume();
});

circuitBreaker.on('close', () => {
  logger.info('Circuit CLOSED — resuming normal consume');
  consumer.resume();
});

// Trong message handler
async function handleMessage(msg) {
  try {
    await circuitBreaker.fire(processMessage, msg);
    await ack(msg);
  } catch (err) {
    if (circuitBreaker.opened) {
      // Circuit open — không ack, message sẽ redeliver
      return;
    }
    await handleFailure(msg, err);
  }
}
```

### Kafka: Pause Partitions

```javascript
consumer.on(circuitBreaker, 'open', () => {
  const assignment = consumer.assignment();
  consumer.pause(assignment.map(({ topic, partition }) => ({ topic, partitions: [partition] })));
});

consumer.on(circuitBreaker, 'close', () => {
  const assignment = consumer.assignment();
  consumer.resume(assignment.map(({ topic, partition }) => ({ topic, partitions: [partition] })));
});
```

### RabbitMQ: Cancel Consumer

```javascript
circuitBreaker.on('open', async () => {
  if (consumerTag) {
    await channel.cancel(consumerTag);
    consumerTag = null;
  }
});

circuitBreaker.on('close', async () => {
  const { consumerTag: tag } = await channel.consume('orders', handler);
  consumerTag = tag;
});
```

---

## Cấu Hình & Tuning

### Parameters

| Parameter | Mô Tả | Giá Trị Typical |
| --------- | ----- | --------------- |
| **failureThreshold** | % failures để OPEN | 50% |
| **volumeThreshold** | Min requests trước khi tính | 10 |
| **resetTimeout** | Thời gian OPEN trước HALF-OPEN | 30s – 5m |
| **timeout** | Request timeout | 5–30s |
| **halfOpenRequests** | Số request thử ở HALF-OPEN | 1–3 |

### Tuning Guidelines

```
resetTimeout:
  - Quá ngắn: Circuit flip OPEN/CLOSED liên tục (flapping)
  - Quá dài: Chậm recovery khi downstream đã healthy
  - Khuyến nghị: 2-3x expected recovery time của downstream

failureThreshold:
  - Quá thấp (20%): Circuit OPEN quá nhạy
  - Quá cao (80%): Chậm phát hiện outage
  - Khuyến nghị: 50% với volumeThreshold >= 10
```

### Per-Dependency Circuit

```javascript
// Mỗi downstream có circuit riêng
const circuits = {
  paymentApi: new CircuitBreaker(callPaymentApi, { resetTimeout: 60000 }),
  inventoryApi: new CircuitBreaker(callInventoryApi, { resetTimeout: 30000 }),
  emailService: new CircuitBreaker(sendEmail, { resetTimeout: 120000 }),
};

// Order consumer gọi nhiều dependencies
async function processOrder(msg) {
  await circuits.paymentApi.fire(charge, msg);
  await circuits.inventoryApi.fire(reserve, msg);
  await circuits.emailService.fire(notify, msg);
}
```

---

## Bulkhead Pattern (Mẫu Vách Ngăn)

**Bulkhead (Vách Ngăn)** — cô lập resources giữa các luồng xử lý, tránh một luồng fail làm ảnh hưởng toàn bộ.

```
┌─────────────────────────────────────────────────────────┐
│  Consumer Process                                        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Thread Pool  │  │ Thread Pool  │  │ Thread Pool  │  │
│  │ Orders       │  │ Payments     │  │ Notifications│  │
│  │ (10 threads) │  │ (5 threads)  │  │ (3 threads)  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │          │
│         ▼                 ▼                 ▼          │
│  Circuit: orders   Circuit: payments  Circuit: email   │
└─────────────────────────────────────────────────────────┘
```

| Pattern | Mục Đích |
| ------- | -------- |
| **Circuit breaker** | Bảo vệ downstream |
| **Bulkhead** | Bảo vệ consumer resources |
| **Kết hợp** | Orders queue fail không block Payments queue |

### Queue-Level Isolation

```
orders.queue     → orders-consumer     → payment-api circuit
payments.queue   → payments-consumer   → stripe-api circuit
notifications    → notify-consumer     → sendgrid circuit

Mỗi queue/consumer độc lập — outage payment không block notifications
```

---

## So Sánh Với Retry và DLQ

| Mechanism | Scope | Khi Dùng | Message Fate |
| --------- | ----- | -------- | ------------ |
| **Retry** | Từng message | Transient failure | Retry sau delay |
| **DLQ** | Từng message | Permanent failure, max retries | Cô lập vào DLQ |
| **Circuit Breaker** | Toàn consumer | Downstream outage | Pause consume, giữ trong broker |

```
Retry:       "Message này fail, thử lại"
DLQ:         "Message này không xử lý được, cô lập"
Circuit:     "Downstream down, dừng gửi request"
```

**Kết hợp:**

```
1. Transient error (1 message)     → Retry
2. Permanent error (1 message)     → DLQ
3. Downstream outage (nhiều msg)   → Circuit OPEN, pause consume
4. Circuit HALF-OPEN, test success → Resume consume, retry messages
```

---

## Implementation

### Với opossum (Node.js)

```javascript
const CircuitBreaker = require('opossum');

const options = {
  timeout: 10000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
  volumeThreshold: 10,
};

const breaker = new CircuitBreaker(callPaymentAPI, options);

breaker.fallback(() => {
  throw new Error('Payment service unavailable');
});

breaker.on('open', () => metrics.increment('circuit.payment.open'));
breaker.on('close', () => metrics.increment('circuit.payment.close'));

// Trong consumer
async function handleOrder(msg) {
  try {
    await breaker.fire(msg.orderId, msg.amount);
    await ack(msg);
  } catch (err) {
    if (breaker.opened) {
      // Không ack — message redeliver khi circuit close
      return;
    }
    await sendToDLQ(msg, err);
  }
}
```

### Với resilience4j (Java)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build();

CircuitBreaker breaker = CircuitBreaker.of("paymentService", config);

// Decorate consumer call
Supplier<OrderResult> decorated = CircuitBreaker
    .decorateSupplier(breaker, () -> paymentService.charge(order));
```

### Metrics

```javascript
// Expose circuit state cho monitoring
metrics.gauge('circuit_breaker_state', () => {
  return breaker.opened ? 1 : (breaker.halfOpen ? 0.5 : 0);
});

metrics.counter('circuit_breaker_trips', { dependency: 'payment-api' });
```

---

## Thiết Kế Production

### Checklist

```
□ Circuit breaker per critical downstream
□ Pause consume khi circuit OPEN (không nack storm)
□ resetTimeout phù hợp với downstream recovery time
□ Alert khi circuit OPEN
□ Metrics: state, trips, half-open attempts
□ Bulkhead: tách queue/consumer theo dependency
□ Runbook: circuit OPEN → check downstream, không panic
□ Test: chaos — kill downstream, verify circuit behavior
```

### Anti-Patterns

| Anti-Pattern | Vấn Đề |
| ------------ | ------ |
| Nack + requeue khi circuit OPEN | Retry storm |
| Circuit quá nhạy | Flapping, false positives |
| Một circuit cho tất cả dependencies | Payment down block email |
| Không pause consume | Waste resources, queue depth tăng |
| Không alert circuit OPEN | Không biết downstream down |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Circuit breaker khác retry thế nào?

**Trả lời:** **Retry** xử lý lỗi **từng message** — transient failure. **Circuit breaker** bảo vệ khi **downstream fail liên tục** — dừng gọi để tránh cascade. Retry cho 1 message fail; circuit cho cả dependency down.

### Câu 2: Circuit OPEN thì message đi đâu?

**Trả lời:** **Giữ trong broker** — không ack/commit offset. Consumer **pause** poll. Khi circuit CLOSED, resume consume, message được xử lý lại. Không nên nack/requeue liên tục — gây retry storm.

### Câu 3: resetTimeout chọn thế nào?

**Trả lời:** Thường **2–3 lần** thời gian recovery expected của downstream. Ví dụ: DB restart ~1 phút → resetTimeout 2–3 phút. Quá ngắn → flapping; quá dài → chậm recovery.

### Câu 4: Circuit breaker có cần cho mọi consumer không?

**Trả lời:** Không. Cần cho consumer gọi **external dependency** (API, DB) có thể fail. Consumer chỉ ghi local DB có thể không cần. Ưu tiên **critical path** (payment, inventory).

### Câu 5: Circuit OPEN + consumer lag tăng — có phải bad?

**Trả lời:** **Expected behavior** — message tích lũy trong broker khi pause. Lag tăng là **trade-off** để bảo vệ downstream. Khi circuit CLOSED, consumer catch up. Cần đảm bảo broker retention đủ lâu (Kafka retention, SQS visibility).

---

**Quay lại:** [README.md](./README.md) — Tổng quan chủ đề Reliability.
