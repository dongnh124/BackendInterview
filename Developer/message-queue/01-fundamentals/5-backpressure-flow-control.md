# Backpressure & Flow Control — Áp Lực Ngược & Điều Khiển Luồng

> Backpressure (Áp Lực Ngược) xảy ra khi producer gửi message nhanh hơn consumer xử lý. Hiểu cơ chế, dấu hiệu, và chiến lược flow control (điều khiển luồng) để tránh sập hệ thống.

## Mục Lục

1. [Backpressure Là Gì?](#backpressure-là-gì)
2. [Dấu Hiệu Backpressure](#dấu-hiệu-backpressure)
3. [Nguyên Nhân](#nguyên-nhân)
4. [Chiến Lược Xử Lý](#chiến-lược-xử-lý)
5. [Rate Limiting & Throttling](#rate-limiting--throttling)
6. [Prefetch & Batch Size](#prefetch--batch-size)
7. [Monitoring Queue Depth](#monitoring-queue-depth)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Backpressure Là Gì?

**Backpressure (Áp Lực Ngược)** là cơ chế — tự nhiên hoặc có chủ đích — khi downstream (bên nhận) **không theo kịp** upstream (bên gửi), áp lực "dồn ngược" lên toàn pipeline.

```
Producer: 10,000 msg/s
                │
                ▼
         [ Broker Queue ]
         depth: 500,000   ◄── Tích lũy!
                │
                ▼
Consumer: 2,000 msg/s    ◄── Chậm hơn 5x

Kết quả: Latency tăng → Memory/disk đầy → OOM → Crash
```

### Tại Sao Quan Trọng?

| Hậu Quả | Mô Tả |
| ------- | ----- |
| **Latency spike** | Message chờ hàng giờ trong queue |
| **Memory pressure** | Broker buffer đầy RAM |
| **Disk full** | Kafka log retention chiếm hết disk |
| **Cascade failure** | Consumer timeout → retry storm → worse |
| **Message expiry** | TTL hết hạn trước khi xử lý |

---

## Dấu Hiệu Backpressure

### Metrics Cần Theo Dõi

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
| ------ | --------------- | ------- |
| **Queue Depth (Độ Sâu Hàng Đợi)** | Tăng liên tục | Consumer không theo kịp |
| **Consumer Lag** | > 10,000 hoặc tăng 1h | Kafka — offset behind |
| **Age of Oldest Message** | > SLA (ví dụ 5 phút) | Message "stale" |
| **Processing Time P99** | Tăng đột biến | Consumer chậm |
| **Error Rate** | Tăng | Retry làm trầm trọng thêm |
| **Broker Disk Usage** | > 80% | Retention chưa kịp xóa |

### Biểu Đồ Điển Hình

```
Queue Depth
    │
    │                              ╱── Crash / throttle
    │                           ╱
    │                        ╱
    │                     ╱
    │                  ╱
    │───────────────╱
    └──────────────────────────────────► Time
         Normal    Spike    Backpressure
```

---

## Nguyên Nhân

### 1. Consumer Chậm

```
- Bug trong handler (N+1 query, blocking I/O)
- Downstream dependency chậm (DB, external API)
- Insufficient instances (thiếu pod/container)
- GC pause (Java/Node.js)
```

### 2. Producer Spike

```
- Flash sale, marketing campaign
- Batch job gửi hàng loạt
- Retry storm từ DLQ replay
```

### 3. Broker Bottleneck

```
- Disk I/O saturated
- Network bandwidth
- Under-provisioned cluster
```

### 4. Thiết Kế Sai

```
- Không có max queue size
- Không có consumer auto-scaling
- Sync processing trong async pipeline
```

---

## Chiến Lược Xử Lý

### Tầng 1: Scale Consumer (Mở Rộng Consumer)

```
Cách nhanh nhất khi consumer CPU-bound hoặc I/O-bound:

Kubernetes HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang)
  metric: consumer_lag > 5000
  action: scale 3 → 10 replicas

Lưu ý: Kafka — không scale quá số partition
```

### Tầng 2: Throttle Producer (Giới Hạn Producer)

```
Khi broker/consumer không scale kịp — BẮT BUỘC slow down producer:

- Rate limiter ở API gateway
- Kafka: max.in.flight.requests.per.connection
- Pause producer khi queue depth > threshold
```

```javascript
// Pseudo-code — adaptive throttling
async function publish(event) {
  const depth = await getQueueDepth();
  if (depth > MAX_DEPTH) {
    await sleep(calculateBackoff(depth));
  }
  await broker.send(event);
}
```

### Tầng 3: Drop / Sample (Bỏ Qua Có Chủ Đích)

```
Chỉ cho non-critical data:
- Metrics: sample 10% khi overload
- Logs: drop debug level
- KHÔNG drop orders, payments
```

### Tầng 4: Circuit Breaker (Cầu Dao Bảo Vệ)

```
Consumer detect downstream fail liên tục
  → Open circuit: stop consuming, message accumulate in broker
  → Half-open: thử vài message
  → Close: resume normal

Tránh waste resource retry message sẽ fail
```

### Tầng 5: Load Shedding (Giảm Tải Có Chủ Đích)

```
API trả 503 khi system overloaded
  → Producer không gửi thêm message
  → Bảo vệ consumer và broker
```

---

## Rate Limiting & Throttling

### Producer-Side Rate Limit

```javascript
const limiter = new RateLimiter({ tokensPerInterval: 100, interval: 'second' });

async function sendMessage(msg) {
  await limiter.removeTokens(1);
  await producer.send(msg);
}
```

### Broker-Side Quota (Kafka)

```properties
# Giới hạn produce rate per client
producer_byte_rate=10485760  # 10 MB/s
```

### Consumer Prefetch Limit (RabbitMQ)

```javascript
// Chỉ prefetch 10 message — tránh consumer nhận quá nhiều
await channel.prefetch(10);
```

**Tại sao prefetch quan trọng:**

```
Prefetch = 1000, Consumer chậm:
  → 1000 message "in-flight" unacked trên 1 consumer
  → Các consumer khác idle (message đã bị claim)
  → Backpressure không lan truyền đúng
```

---

## Prefetch & Batch Size

| Cấu Hình | Tác Dụng | Trade-off |
| -------- | -------- | --------- |
| **Low prefetch (1-10)** | Fair distribution, backpressure nhanh | Throughput thấp hơn |
| **High prefetch (100+)** | Throughput cao | 1 consumer hog messages |
| **Small batch** | Low latency | Nhiều round-trip |
| **Large batch** | High throughput | Latency cao hơn |

### Kafka Consumer Tuning

```properties
# Fetch nhiều data mỗi poll — throughput cao
fetch.min.bytes=1048576
max.poll.records=500

# Cân bằng: max.poll.interval.ms
# Nếu xử lý 500 records > interval → rebalance!
max.poll.interval.ms=300000
```

---

## Monitoring Queue Depth

### Alerting Rules Mẫu

```yaml
# Prometheus alert
- alert: HighConsumerLag
  expr: kafka_consumer_lag > 10000
  for: 15m
  labels:
    severity: warning

- alert: QueueDepthCritical
  expr: rabbitmq_queue_messages > 100000
  for: 5m
  labels:
    severity: critical
```

### Runbook Khi Backpressure

```
1. Xác nhận: lag/depth tăng hay spike tạm?
2. Check consumer health: error rate, processing time
3. Check downstream: DB slow query? API timeout?
4. Short-term: scale consumer (trong giới hạn partition)
5. Medium-term: throttle producer nếu cần
6. Long-term: optimize handler, tăng partition, redesign
```

### Auto-Scaling Formula (Kafka)

```
Cần consumer instances ≈ total_lag / (target_lag_per_consumer)
Giới hạn: ≤ số partition trong topic

Ví dụ: lag=50,000, target=5,000/consumer → cần 10 consumers
       topic có 6 partitions → max 6 active consumers
       → Cần tăng partition HOẶC optimize processing
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Backpressure khác gì consumer lag?

**Đáp án mẫu:** **Consumer lag** là metric (độ trễ offset) — triệu chứng. **Backpressure** là hiện tượng/hiện tượng hệ thống khi pressure từ downstream lan ngược lên — queue đầy, producer bị block. Lag cao thường báo hiệu backpressure đang xảy ra.

### Câu 2: Làm gì khi consumer lag tăng liên tục?

**Đáp án mẫu:** (1) Diagnose root cause — consumer chậm hay producer spike? (2) Scale consumer trong giới hạn partition. (3) Optimize handler — profiling, fix N+1. (4) Tăng partition nếu cần parallelism. (5) Throttle producer tạm thời. (6) Không replay DLQ khi đang lag.

### Câu 3: Prefetch cao gây vấn đề gì?

**Đáp án mẫu:** Consumer nhận nhiều message trước khi xử lý xong — nếu consumer chậm/crash, message unacked bị "stuck". Các consumer khác idle vì message đã prefetch. Giảm fairness và che giấu backpressure. Production thường prefetch 10-50, tune theo processing time.

### Câu 4: Thiết kế hệ thống chống backpressure từ đầu?

**Đáp án mẫu:** (1) **Monitoring** lag/depth từ ngày 1. (2) **Auto-scaling** consumer theo lag. (3) **Rate limit** producer. (4) **Idempotent** consumer cho safe retry. (5) **DLQ** cho poison message — tránh block queue. (6) **Capacity planning** — biết max throughput consumer. (7) **Load test** với spike scenario.

---

**Xem tiếp:** [6-broker-selection-guide.md](./6-broker-selection-guide.md) — chọn broker phù hợp use case.
