# Broker Sizing — Lập Kế Hoạch Dung Lượng Broker

> **Capacity Planning (Lập Kế Hoạch Dung Lượng)** cho message broker: ước tính disk, network, memory, CPU để cluster handle traffic hiện tại và growth 12–18 tháng — tránh incident disk full hoặc over-provision lãng phí.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Capacity Planning Framework](#capacity-planning-framework)
3. [Disk Sizing](#disk-sizing)
4. [Network Sizing](#network-sizing)
5. [Memory Sizing](#memory-sizing)
6. [CPU Sizing](#cpu-sizing)
7. [Kafka Cluster Sizing](#kafka-cluster-sizing)
8. [RabbitMQ Cluster Sizing](#rabbitmq-cluster-sizing)
9. [Growth Projection](#growth-projection)
10. [Cost vs Performance Trade-offs](#cost-vs-performance-trade-offs)
11. [Pre-Production Checklist](#pre-production-checklist)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              BROKER SIZING — QUICK FORMULAS                        │
├─────────────────────────────────────────────────────────────────┤
│  Disk  = daily_ingress × retention_days × RF × 1.2 overhead       │
│  Net   = ingress × RF (replication) × 1.1 overhead              │
│  RAM   = heap + page cache (OS cache cho disk I/O)                │
│  CPU   = benchmark-driven — thường 8-16 vCPU/broker production  │
└─────────────────────────────────────────────────────────────────┘
```

| Resource | Bottleneck Symptom | First Check |
| -------- | ------------------ | ----------- |
| **Disk** | Retention fail, broker crash | `df -h`, log size per topic |
| **Network** | Replication lag, slow fetch | `iftop`, broker network metrics |
| **Memory** | OOM, GC pause, page cache miss | Heap usage, OS memory |
| **CPU** | Request queue, high `%util` | JMX thread metrics |

---

## Capacity Planning Framework

### Bước 1: Thu Thập Requirements

| Input | Ví Dụ | Nguồn |
| ----- | ----- | ----- |
| Peak messages/sec | 50,000 | Load test / production metrics |
| Average message size | 2 KB | Sample messages |
| Retention period | 7 days | Business requirement |
| Replication factor | 3 | HA policy |
| Peak/Average ratio | 3x | Traffic pattern |
| Growth rate | 50%/year | Business forecast |

### Bước 2: Tính Resource

```
Peak throughput = 50,000 msg/s × 2 KB = 100 MB/s ingress
Daily ingress = 100 MB/s × 86,400s × (avg/peak factor)
```

### Bước 3: Benchmark Validate

**Không tin công thức alone** — chạy load test trên hardware tương tự production.

### Bước 4: Headroom

Luôn cộng **20–30% headroom** cho spike, rebalancing, compaction.

---

## Disk Sizing

Disk thường là **bottleneck đầu tiên** của Kafka cluster.

### Công Thức Kafka

```
Disk per broker = (daily_ingress / broker_count)
                  × retention_days
                  × replication_factor_local   # Mỗi broker giữ ~1/N partitions
                  × (1 + overhead)

overhead ≈ 20% (indexes, segments chưa compact, free space)
```

### Ví Dụ Tính Toán

```
Input:
  Peak ingress:     200 MB/s
  Average factor: 40% of peak → 80 MB/s average
  Daily ingress:  80 × 86400 ≈ 6.9 TB/day
  Retention:      7 days
  Replication:    3
  Brokers:        6

Mỗi broker giữ ~1/6 partitions (uniform):
  Per broker = 6.9 TB × 7 × (3/6) × 1.2
             = 6.9 × 7 × 0.5 × 1.2
             ≈ 29 TB per broker

→ Chọn 32 TB NVMe per broker (hoặc 6 brokers × 6 TB nếu retention giảm)
```

### Disk Type Khuyến Nghị

| Type | Throughput | Latency | Kafka |
| ---- | ---------- | ------- | ----- |
| **NVMe SSD** | Rất cao | Thấp | **Khuyến nghị production** |
| **SSD (SATA)** | Cao | Trung bình | Acceptable |
| **HDD** | Thấp | Cao | Không khuyến nghị Kafka |
| **EBS gp3** | Configurable IOPS | Trung bình | AWS — tune IOPS |

### Kafka Disk Best Practices

```properties
# Tách log dirs trên nhiều disk (JBOD)
log.dirs=/data1/kafka,/data2/kafka,/data3/kafka

# Retention safety
log.retention.hours=168
log.retention.check.interval.ms=300000

# Segment size — balance file count vs recovery
log.segment.bytes=1073741824   # 1 GB
```

### RabbitMQ Disk

RabbitMQ disk cho persistent messages:

```
Disk = queue_depth_max × avg_message_size × queue_count × 1.2

+ metadata overhead (nhỏ)
+ Quorum queue Raft log
```

---

## Network Sizing

### Kafka Replication Traffic

```
Network per broker ≈ ingress × replication_factor

Ví dụ:
  Ingress 100 MB/s, RF=3
  Leader nhận 100 MB/s, replicate tới 2 followers
  → Network out ≈ 200 MB/s per leader broker
  → Chọn 1 Gbps minimum, 10 Gbps cho high throughput
```

### Bandwidth Checklist

| Traffic Type | Direction | Factor |
| ------------ | --------- | ------ |
| Producer → Leader | In | 1× ingress |
| Leader → Followers | Out | (RF-1) × ingress |
| Consumer fetch | Out | Depends on consumer count |
| Cross-AZ replication | Both | +latency, +cost (cloud) |

### Cloud Network

```
AWS: Same-AZ traffic free, cross-AZ $0.01/GB
→ Đặt broker và producer/consumer cùng AZ khi có thể
→ replication.factor=3 across AZs cho HA (chấp nhận cross-AZ cost)
```

---

## Memory Sizing

### Kafka Memory Model

```
Total RAM = JVM Heap + OS Page Cache + OS overhead

JVM Heap:     6-8 GB typical (G1GC), KHÔNG quá 50% total RAM
Page Cache:   Phần còn lại — CRITICAL cho disk read performance
```

```properties
# Kafka JVM — KAFKA_HEAP_OPTS
KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"

# Total broker RAM: 32 GB
# Heap: 6 GB
# Page cache: ~24 GB cho hot data
```

> **Quy tắc:** Kafka dựa vào **OS page cache** nhiều hơn JVM heap. Under-provision RAM → disk I/O tăng → latency spike.

### RabbitMQ Memory

```
RabbitMQ dùng RAM cho:
  - Queue index
  - Message body (transient queues)
  - Connection state

Memory watermark default: 40% total RAM
  → Vượt → block publishers

Khuyến nghị: 8-16 GB RAM/node cho production moderate load
```

```bash
# Tính memory per queue (ước tính)
# ~1 KB metadata + message_size × depth per queue
```

---

## CPU Sizing

### Kafka CPU Drivers

| Operation | CPU Impact |
| --------- | ---------- |
| Compression/decompression | Cao (lz4 thấp, gzip cao) |
| SSL/TLS encryption | Trung bình–cao |
| Replication | Trung bình |
| Request handling | Thấp–trung bình |
| Log compaction | Cao (periodic) |

### Benchmark-Driven

```bash
# Chạy perf test, monitor CPU
kafka-producer-perf-test ... # Target 80% CPU sustained

# Rule of thumb nếu chưa benchmark:
# 50 MB/s per broker → 8 vCPU
# 100 MB/s per broker → 16 vCPU
```

### RabbitMQ CPU

- Mostly single-threaded per queue (Erlang scheduler)
- More queues → more scheduler utilization
- **16 vCPU** node handle hàng trăm queues moderate load

---

## Kafka Cluster Sizing

### Broker Count

```
brokers ≥ max(replication_factor, partition_distribution)

Typical production:
  - Minimum 3 brokers (RF=3, tolerate 1 failure)
  - 6-12 brokers cho medium cluster
  - 12+ cho high throughput / isolation
```

### Partitions Per Broker

```
partitions_per_broker = total_partitions / broker_count

Khuyến nghị: < 4,000 partitions/broker (Confluent guideline)
> 10,000 → metadata overhead, slow leader election
```

### Ví Dụ Cluster Design

```
Requirements:
  500 MB/s peak ingress
  7-day retention
  RF=3

Benchmark: 1 broker handle 80 MB/s (NVMe, 16 vCPU)

Brokers needed (throughput): 500/80 ≈ 7 → 9 brokers (headroom)
Disk per broker: ~25 TB (tính từ công thức)
Instance: i3en.2xlarge (8 vCPU, 64 GB, 2.5 TB NVMe × 2)
  → Cần nhiều disk hoặc instance lớn hơn

Alternative: 6× i3en.6xlarge (24 NVMe disks) — right-size sau benchmark
```

### KRaft vs ZooKeeper

**KRaft (Kafka Raft Metadata)** — metadata trong Kafka brokers, không cần ZooKeeper riêng. Giảm operational overhead, metadata scale tốt hơn cho cluster lớn.

---

## RabbitMQ Cluster Sizing

### Node Count

```
Minimum 3 nodes (quorum queues cần odd number cho Raft)
Scale: thêm nodes khi CPU/RAM/disk pressure
```

### Queue Distribution

```
Classic mirrored queues: deprecated → dùng Quorum Queues
Quorum queue: replicate across nodes (RF configurable)

1 queue = 1 leader + N-1 replicas on other nodes
Nhiều queues → spread across cluster
```

### Sizing Table

| Load | Nodes | RAM/node | Disk/node | vCPU |
| ---- | ----- | -------- | --------- | ---- |
| Dev | 1 | 4 GB | 50 GB | 2 |
| Small prod | 3 | 8 GB | 200 GB SSD | 4 |
| Medium prod | 3-5 | 16 GB | 500 GB SSD | 8 |
| Large prod | 5+ | 32 GB | 1 TB NVMe | 16 |

---

## Growth Projection

### 12-Month Forecast Template

```
Month 0 (now):  50 MB/s, 6 brokers, 15 TB total disk
Month 6:        75 MB/s (+50%), 9 brokers, 25 TB
Month 12:       100 MB/s (+100%), 12 brokers, 40 TB

Trigger points:
  Disk > 70%:     Plan expansion Q+1
  CPU > 70% sustained: Add brokers
  Lag frequent:   Consumer scale OR broker scale
```

### Elastic vs Fixed Capacity

| Model | Ưu | Nhược |
| ----- | -- | ----- |
| **Fixed (on-prem)** | Predictable cost | Plan ahead, lead time |
| **Cloud auto-scale** | Elastic | Cost surprise, complexity |
| **Managed (MSK, CloudAMQP)** | Ops offload | Less control, $$$ |

---

## Cost vs Performance Trade-offs

| Decision | Save Cost | Pay Performance |
| -------- | --------- | --------------- |
| RF=2 thay vì 3 | 33% disk/network | Mất 1 broker tolerance |
| Retention 3 ngày thay vì 7 | ~57% disk | Ít replay window |
| HDD thay vì NVMe | $$$ | Latency, throughput |
| acks=1 | Lower latency cost | Durability risk |
| Single AZ | No cross-AZ traffic | Mất AZ = outage |
| Compression off | CPU | Network/disk 2-5x |

---

## Pre-Production Checklist

```
□ Load test đạt target throughput với RF và acks production
□ Disk usage projection 12 tháng documented
□ Network bandwidth verified (including replication)
□ JVM heap tuned, page cache adequate
□ Retention policy set, disk alert at 70%/80%
□ Broker count tolerate N-1 failure (N = RF)
□ Partition count right-sized (không quá 4000/broker)
□ Monitoring: disk, network, CPU, under-replicated partitions
□ Runbook disk full scenario
□ Growth review calendar (quarterly)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka cluster 3 brokers, disk 80% — xử lý gì trước?

**Trả lời:** (1) **Immediate:** Giảm retention tạm thời hoặc xóa topic không cần. (2) **Short-term:** Thêm disk (expand volume) hoặc thêm brokers + rebalance. (3) **Long-term:** Review retention policy, compression, topic cleanup. **Không** chờ 100% — broker có thể crash.

### Câu 2: 32 GB RAM broker — set Kafka heap bao nhiêu?

**Trả lời:** **6–8 GB heap** (G1GC). Phần còn lại (~24 GB) cho **OS page cache** — critical cho read performance. Heap quá lớn → GC pause dài, ít page cache → disk I/O tăng.

### Câu 3: Làm sao biết cần thêm broker hay tune config?

**Trả lời:** **Benchmark trước.** Nếu CPU/disk/network < 60% mà throughput chưa đạt → tune (batching, compression, partition). Nếu resources > 70% sustained → thêm broker. Nếu 1 partition hot → fix key design, không thêm broker.

---

**Cập Nhật:** 2026-07-03
