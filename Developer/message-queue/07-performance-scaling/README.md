# Hiệu Năng & Mở Rộng — Tổng Quan

> Chủ đề cốt lõi khi messaging system cần xử lý hàng triệu message/ngày: Throughput Tuning (Tối Ưu Thông Lượng), Partitioning Strategies (Chiến Lược Phân Vùng), Consumer Scaling (Mở Rộng Consumer), Backpressure Handling (Xử Lý Áp Lực Ngược), và Broker Sizing (Lập Kế Hoạch Dung Lượng Broker).

## Mục Lục

1. [Tại Sao Performance & Scaling Quan Trọng](#tại-sao-performance--scaling-quan-trọng)
2. [Throughput vs Latency Trade-off](#throughput-vs-latency-trade-off)
3. [Kiến Trúc Scaling Tổng Quan](#kiến-trúc-scaling-tổng-quan)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Performance & Scaling Quan Trọng

Messaging system thường bắt đầu nhỏ — vài trăm message/phút — nhưng traffic tăng theo cấp số nhân khi business scale. **Không tune sớm** dẫn đến consumer lag tích lũy, hot partition (phân vùng nóng), disk full, và incident production.

| Giai Đoạn | Triệu Chứng | Hậu Quả |
| --------- | ----------- | ------- |
| **Early growth** | Lag tăng nhẹ, P99 latency cao | User experience giảm |
| **Traffic spike** | Queue depth tăng nhanh | Message stale, SLA breach |
| **Sustained overload** | Hot partition, broker CPU 100% | Uneven load, rebalance storm |
| **Capacity limit** | Disk full, OOM (Out of Memory — Hết Bộ Nhớ) | Broker crash, message loss risk |

> **Quy tắc vàng:** Scale theo **bottleneck thực tế** — không phải lúc nào cũng thêm consumer. Đôi khi cần tune producer batching, tăng partition count, hoặc resize broker trước.

---

## Throughput vs Latency Trade-off

Hai metric quan trọng nhất trong messaging performance thường **mâu thuẫn**:

```
                    HIGH THROUGHPUT
                          ▲
                          │
         Kafka batching   │   RabbitMQ prefetch cao
         linger.ms > 0    │   Consumer pool lớn
                          │
    ◄─────────────────────┼─────────────────────►
    LOW LATENCY           │              HIGH LATENCY
                          │
         linger.ms = 0    │   acks=0 fire-and-forget
         sync produce     │
                          ▼
                    LOW THROUGHPUT
```

| Mục Tiêu | Ưu Tiên | Cấu Hình Điển Hình |
| -------- | ------- | ------------------ |
| **Real-time notification** | Latency thấp | `linger.ms=0`, prefetch=1, push consumer |
| **Log ingestion / analytics** | Throughput cao | `batch.size` lớn, `linger.ms=10-50`, compression |
| **Order processing** | Cân bằng + ordering | Partition theo `orderId`, manual commit |
| **CDC pipeline** | Throughput + durability | `acks=all`, batching, parallel consumers |

---

## Kiến Trúc Scaling Tổng Quan

```
┌──────────────────────────────────────────────────────────────────────────┐
│              PERFORMANCE & SCALING LAYERS (Các Lớp Tối Ưu)               │
│                                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌───────────┐ │
│  │  PRODUCER   │───►│   BROKER    │───►│  CONSUMER   │───►│ DOWNSTREAM│ │
│  │  Tuning     │    │   Sizing    │    │  Scaling    │    │  (DB/API) │ │
│  └─────────────┘    └─────────────┘    └─────────────┘    └───────────┘ │
│        │                  │                  │                  │       │
│   batch/linger       partitions          parallelism         connection │
│   compression        replication           prefetch            pool size  │
│   pipelining         disk I/O            HPA/K8s             query tune  │
│                                                                          │
│  ═══════════════════ CROSS-CUTTING ═══════════════════════════════════  │
│  Partitioning Strategy │ Backpressure │ Monitoring (lag, depth, CPU)    │
└──────────────────────────────────────────────────────────────────────────┘
```

**Bốn điểm tune chính:**

1. **Producer Layer** — batch size, linger, compression, pipelining
2. **Broker Layer** — partition count, disk, network, memory, replication
3. **Consumer Layer** — parallelism, partition-to-consumer ratio, prefetch
4. **Application Layer** — downstream connection pool, idempotent batch writes

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-throughput-tuning.md](./1-throughput-tuning.md) | Batch size, linger, compression, pipelining | 1.5 giờ |
| 2 | [2-partitioning-strategies.md](./2-partitioning-strategies.md) | Key design, hot partition, rebalancing | 1.5 giờ |
| 3 | [3-consumer-scaling.md](./3-consumer-scaling.md) | Parallelism, partition-to-consumer ratio, HPA | 1.5 giờ |
| 4 | [4-backpressure-handling.md](./4-backpressure-handling.md) | Rate limiting, throttling, queue depth | 1 giờ |
| 5 | [5-broker-sizing.md](./5-broker-sizing.md) | Capacity planning, disk, network, memory | 1.5 giờ |

**Điều kiện tiên quyết:**

- [01-fundamentals/5-backpressure-flow-control.md](../01-fundamentals/5-backpressure-flow-control.md)
- [03-apache-kafka/2-topics-partitions.md](../03-apache-kafka/2-topics-partitions.md)
- [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md)
- [03-apache-kafka/4-producers-serialization.md](../03-apache-kafka/4-producers-serialization.md)

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4 → 5. Tune producer trước (dễ đo), thiết kế partition (ảnh hưởng lâu dài), scale consumer (phổ biến nhất), xử lý backpressure (khi quá tải), cuối cùng broker sizing (infrastructure).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-throughput-tuning.md](./1-throughput-tuning.md) | Producer/consumer throughput, batching, compression, pipelining cross-broker |
| [2-partitioning-strategies.md](./2-partitioning-strategies.md) | Partition key design, hot partition detection & mitigation, rebalancing |
| [3-consumer-scaling.md](./3-consumer-scaling.md) | Kafka consumer groups, RabbitMQ prefetch, K8s HPA, scaling limits |
| [4-backpressure-handling.md](./4-backpressure-handling.md) | Rate limiting, throttling, adaptive consume, queue depth management |
| [5-broker-sizing.md](./5-broker-sizing.md) | Capacity planning formula, disk/network/memory sizing, growth projection |

---

## Bài Tập Thực Hành

### Lab 1: Benchmark Throughput (60 phút)

```bash
# Kafka — dùng kafka-producer-perf-test / kafka-consumer-perf-test
kafka-producer-perf-test --topic perf-test \
  --num-records 1000000 --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092 \
    batch.size=65536 linger.ms=10 compression.type=lz4

# So sánh: linger.ms=0 vs linger.ms=50, gzip vs lz4 vs snappy
# Ghi lại: records/sec, MB/sec, P99 latency
```

### Lab 2: Phát Hiện Hot Partition (45 phút)

```
Kịch bản: Topic orders với key = userId, 80% traffic từ 5% users.

Bài tập:
1. Publish skewed workload (Zipf distribution)
2. Monitor per-partition message rate (Kafka UI / metrics)
3. Xác định partition nào hot
4. Đề xuất fix: salt key, composite key, hoặc tách topic
```

### Lab 3: Consumer Scaling Limit (45 phút)

```
1. Topic 6 partitions, consumer group — thêm consumer từ 1 → 10
2. Quan sát: khi nào throughput không tăng thêm?
3. Ghi nhận rebalance impact khi scale
4. Tính partition-to-consumer ratio tối ưu
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Consumer lag cao — scale consumer hay tune producer?

**Trả lời:** **Diagnose trước.** Nếu consumer count < partition count và CPU consumer thấp → scale consumer. Nếu consumer đã max partition và CPU cao → optimize handler hoặc scale downstream. Nếu producer burst quá lớn → rate limit producer hoặc tăng buffer. Xem [3-consumer-scaling.md](./3-consumer-scaling.md).

### Câu 2: Tăng partition count có ảnh hưởng ordering không?

**Trả lời:** Ordering chỉ đảm bảo **trong cùng partition**. Tăng partition → key space phân tán hơn → có thể **phá vỡ ordering** nếu key design không đúng. Cần re-evaluate partition key strategy. Xem [2-partitioning-strategies.md](./2-partitioning-strategies.md).

### Câu 3: Kafka broker cần bao nhiêu disk?

**Trả lời:** `disk = daily_ingress × retention_days × replication_factor × (1 + overhead)`. Ví dụ: 500 GB/ngày × 7 ngày × 3 replicas × 1.2 overhead ≈ **12.6 TB**. Xem [5-broker-sizing.md](./5-broker-sizing.md).

### Câu 4: Backpressure khác rate limiting thế nào?

**Trả lời:** **Backpressure** là hiện tượng/hệ quả khi downstream không theo kịp. **Rate limiting** là cơ chế **chủ đích** giới hạn tốc độ để tránh backpressure. Xem [4-backpressure-handling.md](./4-backpressure-handling.md) và [01-fundamentals/5-backpressure-flow-control.md](../01-fundamentals/5-backpressure-flow-control.md).

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Backpressure cơ bản | [01-fundamentals/5-backpressure-flow-control.md](../01-fundamentals/5-backpressure-flow-control.md) |
| Kafka partitions | [03-apache-kafka/2-topics-partitions.md](../03-apache-kafka/2-topics-partitions.md) |
| Consumer groups | [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md) |
| Producer config | [03-apache-kafka/4-producers-serialization.md](../03-apache-kafka/4-producers-serialization.md) |
| Circuit breaker | [06-reliability/5-circuit-breaker-consumers.md](../06-reliability/5-circuit-breaker-consumers.md) |
| Monitoring lag | [09-monitoring/](../09-monitoring/) (sắp có) |

---

**Cập Nhật:** 2026-07-03
