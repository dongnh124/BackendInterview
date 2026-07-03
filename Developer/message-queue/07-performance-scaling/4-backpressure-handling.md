# Backpressure Handling — Xử Lý Áp Lực Ngược

> Backpressure (Áp Lực Ngược) trong performance context tập trung vào **cơ chế chủ đích** để hệ thống tự điều chỉnh tốc độ — rate limiting (Giới Hạn Tốc Độ), throttling (Điều Tiết), adaptive consume (Tiêu Thụ Thích Ứng) — trước khi queue overflow gây incident.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Backpressure vs Performance](#backpressure-vs-performance)
3. [Queue Depth Management](#queue-depth-management)
4. [Rate Limiting Strategies](#rate-limiting-strategies)
5. [Throttling Producers](#throttling-producers)
6. [Adaptive Consumer](#adaptive-consumer)
7. [Broker-Level Flow Control](#broker-level-flow-control)
8. [Circuit Breaker Integration](#circuit-breaker-integration)
9. [Monitoring & Alerting](#monitoring--alerting)
10. [Runbook: Backpressure Incident](#runbook-backpressure-incident)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│         BACKPRESSURE HANDLING — DEFENSE IN DEPTH                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: Producer rate limit (token bucket / leaky bucket)       │
│  Layer 2: Broker retention + max queue length                     │
│  Layer 3: Consumer prefetch / poll rate adaptive                  │
│  Layer 4: Circuit breaker pause consume khi downstream down       │
│  Layer 5: Autoscale consumer khi lag sustained                    │
└─────────────────────────────────────────────────────────────────┘
```

| Signal | Action |
| ------ | ------ |
| Queue depth tăng liên tục | Throttle producer hoặc scale consumer |
| Consumer lag > SLA | Alert + investigate bottleneck |
| Broker disk > 80% | Tăng retention cleanup, scale disk, throttle |
| Downstream timeout spike | Circuit breaker open, pause consume |

> **Đọc thêm lý thuyết:** [01-fundamentals/5-backpressure-flow-control.md](../01-fundamentals/5-backpressure-flow-control.md)

---

## Backpressure vs Performance

**Performance tuning** tối đa hóa throughput khi hệ thống healthy. **Backpressure handling** bảo vệ hệ thống khi capacity không đủ.

```
Healthy state:          Overload state:
Producer ──► Broker     Producer ──► Broker (FULL)
              │                        │
Consumer ◄────┘          Consumer ◄────┘ (can't keep up)
                                              │
                                    BACKPRESSURE propagates ↑
```

### Ba Phản Ứng Khi Overload

| Phản Ứng | Mô Tả | Risk |
| -------- | ----- | ---- |
| **Buffer (Đệm)** | Message tích lũy trong queue | Disk/memory full |
| **Drop (Bỏ)** | Reject message mới | Data loss |
| **Slow down (Chậm Lại)** | Producer/consumer throttle | Latency tăng, nhưng stable |

> **Production khuyến nghị:** Slow down có chủ đích > buffer vô hạn > drop.

---

## Queue Depth Management

**Queue Depth (Độ Sâu Hàng Đợi)** — số message đang chờ xử lý.

### Ngưỡng Tham Khảo

| Depth | Trạng Thái | Hành Động |
| ----- | ---------- | --------- |
| Normal | < 1,000 | Monitor |
| Elevated | 1,000 – 10,000 | Investigate, prepare scale |
| High | 10,000 – 100,000 | Scale consumer, throttle producer |
| Critical | > 100,000 hoặc tăng không dừng | Incident — emergency throttle |

### Kafka — Consumer Lag Là Queue Depth

```
Lag = log_end_offset - current_offset (per partition)

Total lag = sum(lag per partition)

Age of oldest unprocessed = lag / consume_rate
```

### RabbitMQ — Queue Messages Ready

```bash
# rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
# messages_ready = chờ consumer
# messages_unacknowledged = đang xử lý (prefetch)
```

### Max Length Policy (RabbitMQ)

```javascript
// Giới hạn queue depth — overflow strategy
await channel.assertQueue('orders', {
  durable: true,
  arguments: {
    'x-max-length': 100000,
    'x-overflow': 'reject-publish',  // Hoặc 'drop-head' (cẩn thận!)
  },
});
```

| Overflow Strategy | Hành Vi | Khi Nào Dùng |
| ----------------- | ------- | ------------ |
| `reject-publish` | Producer nhận error | Bảo vệ data, producer retry |
| `drop-head` | Xóa message cũ nhất | Metrics only — không critical data |

---

## Rate Limiting Strategies

### Token Bucket (Thùng Token)

```
Bucket capacity: 1000 tokens
Refill rate: 100 tokens/second

Mỗi message = 1 token
Không đủ token → reject hoặc wait
```

```javascript
class TokenBucket {
  constructor(capacity, refillPerSec) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillPerSec;
    this.lastRefill = Date.now();
  }

  tryConsume(n = 1) {
    this.refill();
    if (this.tokens >= n) {
      this.tokens -= n;
      return true;
    }
    return false;
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
}

const limiter = new TokenBucket(1000, 100);  // burst 1000, sustained 100/s

async function publish(msg) {
  while (!limiter.tryConsume()) {
    await sleep(10);  // Backpressure: chờ token
  }
  await producer.send(msg);
}
```

### Leaky Bucket (Thùng Rò)

Message vào bucket, ra với rate cố định — smooth traffic spikes.

```
Burst 5000 msg trong 1 giây
Leaky bucket 500 msg/s
→ Output smooth 500/s trong 10 giây
```

### Per-Tenant Rate Limit

```javascript
const tenantLimiters = new Map();

function getLimiter(tenantId) {
  if (!tenantLimiters.has(tenantId)) {
    const tier = getTenantTier(tenantId);
    const rate = tier === 'enterprise' ? 1000 : 100;
    tenantLimiters.set(tenantId, new TokenBucket(rate * 10, rate));
  }
  return tenantLimiters.get(tenantId);
}
```

---

## Throttling Producers

### Kafka — Buffer Backpressure

Khi `buffer.memory` full, `send()` block hoặc throw `BufferExhaustedException`:

```properties
# Producer tự backpressure khi buffer đầy
buffer.memory=67108864        # 64 MB
max.block.ms=60000            # Block tối đa 60s trước khi throw
```

```javascript
try {
  await producer.send({ topic, messages });
} catch (err) {
  if (err.name === 'KafkaJSBufferExhausted') {
    // Backpressure — slow down or shed load
    await sleep(100);
    return retrySend(topic, messages);
  }
}
```

### Application-Level Produce Gate

```javascript
// Chỉ produce khi lag dưới ngưỡng
async function gatedProduce(topic, message) {
  const lag = await getConsumerLag(topic);
  if (lag > LAG_THRESHOLD) {
    metrics.increment('produce.throttled');
    throw new ThrottleError('Consumer lag too high');
  }
  await producer.send({ topic, messages: [message] });
}
```

### HTTP 429 cho API → MQ Pipeline

```
API nhận request → check queue depth
  depth < threshold → publish message → 202 Accepted
  depth > threshold → 429 Too Many Requests (client retry)
```

---

## Adaptive Consumer

Consumer tự điều chỉnh tốc độ poll/consume dựa trên downstream health.

### Dynamic Prefetch

```javascript
let prefetch = 50;

async function adjustPrefetch() {
  const dbLatencyP99 = await metrics.get('db.latency.p99');
  if (dbLatencyP99 > 500) {
    prefetch = Math.max(1, prefetch - 10);   // Slow down
  } else if (dbLatencyP99 < 100) {
    prefetch = Math.min(100, prefetch + 10); // Speed up
  }
  await channel.prefetch(prefetch);
}

setInterval(adjustPrefetch, 30000);
```

### Pause/Resume Partition (Kafka)

```javascript
// Pause partitions khi downstream overload
const paused = new Set();

async function eachMessage({ topic, partition, message }) {
  const dbHealthy = await healthCheck();
  if (!dbHealthy) {
    if (!paused.has(partition)) {
      consumer.pause([{ topic, partitions: [partition] }]);
      paused.add(partition);
    }
    return;
  }
  if (paused.has(partition)) {
    consumer.resume([{ topic, partitions: [partition] }]);
    paused.delete(partition);
  }
  await processMessage(message);
}
```

### Bounded Worker Pool

```javascript
const semaphore = new Semaphore(MAX_CONCURRENT);  // e.g. 50

async function handleMessage(msg) {
  await semaphore.acquire();
  try {
    await process(msg);
    channel.ack(msg);
  } finally {
    semaphore.release();
  }
}
// Khi 50 tasks đang chạy → acquire block → prefetch backpressure
```

---

## Broker-Level Flow Control

### Kafka Retention & Disk

```properties
# Tự động xóa log cũ — tránh disk full
log.retention.hours=168          # 7 ngày
log.retention.bytes=-1           # Hoặc giới hạn theo size
log.segment.bytes=1073741824     # 1 GB per segment
```

### RabbitMQ Memory Alarm

```
RabbitMQ khi memory > watermark (default 40% RAM):
  → Block ALL publishers cluster-wide
  → Publishers block until memory freed

Fix: TTL, max-length queue, scale consumers, tăng RAM
```

```bash
# rabbitmqctl set_vm_memory_high_watermark 0.6
# rabbitmqctl set_policy TTL ".*" '{"message-ttl":3600000}' --apply-to queues
```

### SQS — Visibility Timeout Backpressure

Message invisible trong visibility timeout. Consumer chậm → message reappear → duplicate processing. Tune visibility timeout ≥ P99 processing time.

---

## Circuit Breaker Integration

Khi downstream fail liên tục, **Circuit Breaker (Cầu Dao Bảo Vệ)** pause consume thay vì retry vô hạn — giảm backpressure ngược lên broker.

```
Consumer ──► DB (down)
    │
    ├── Retry 1000 msg/s → all fail → lag tăng
    │
    └── Circuit OPEN → pause poll → lag tăng chậm
              → downstream recover → HALF-OPEN → resume
```

> Xem chi tiết: [06-reliability/5-circuit-breaker-consumers.md](../06-reliability/5-circuit-breaker-consumers.md)

---

## Monitoring & Alerting

### Alert Rules

| Alert | Condition | Severity |
| ----- | --------- | -------- |
| `LagHigh` | lag > 10,000 for 5m | Warning |
| `LagCritical` | lag > 100,000 or growing 30m | Critical |
| `QueueDepthHigh` | depth > threshold | Warning |
| `BrokerDiskHigh` | disk > 80% | Warning |
| `ProduceThrottled` | throttle rate > 0 sustained | Info |
| `MemoryAlarm` | RabbitMQ memory alarm | Critical |

### Dashboard Panels

```
Row 1: Produce rate | Consume rate | Lag (total + per-partition)
Row 2: Queue depth | Oldest message age | P99 handler latency
Row 3: Broker disk | CPU | Network throughput
Row 4: Throttle events | Circuit breaker state | Error rate
```

---

## Runbook: Backpressure Incident

```
1. ACKNOWLEDGE — confirm lag/depth alert
2. ASSESS severity:
   - Lag growth rate (messages/min)
   - Oldest message age vs SLA
   - Disk usage trend
3. IMMEDIATE mitigation (chọn 1+):
   □ Scale consumer (+2 instances)
   □ Throttle non-critical producers
   □ Pause low-priority consumer groups
   □ Circuit breaker nếu downstream down
4. IDENTIFY root cause:
   □ Consumer slow? → profile handler, check DB
   □ Producer spike? → check upstream, rate limit
   □ Hot partition? → per-partition lag
   □ Broker issue? → disk, CPU, network
5. LONG-TERM fix:
   □ Optimize handler, add cache
   □ Increase partitions
   □ Fix key skew
   □ Capacity plan broker
6. POST-INCIDENT: update thresholds, runbook, load test
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Queue depth tăng — ưu tiên scale consumer hay throttle producer?

**Trả lời:** **Cả hai tùy ngữ cảnh.** Nếu traffic hợp lệ và consumer under-provisioned → scale consumer. Nếu traffic spike bất thường (DDoS, bug loop) → throttle producer trước để bảo vệ hệ thống. Trong incident, thường làm **cả hai song song**: scale + throttle non-critical.

### Câu 2: `x-overflow: drop-head` có nên dùng production không?

**Trả lời:** **Hầu như không** cho business-critical data — mất message cũ silently. Chỉ acceptable cho metrics/telemetry nơi mất vài data points OK. Production critical: `reject-publish` + producer handle backpressure.

### Câu 3: Kafka `max.block.ms` nên set bao nhiêu?

**Trả lời:** Phụ thuộc SLA. `60000` (60s) là common — producer block tối đa 60s khi buffer full. Quá thấp → nhiều false failures. Quá cao → thread block lâu. Alternative: async send + callback handle backpressure explicitly.

---

**Cập Nhật:** 2026-07-03
