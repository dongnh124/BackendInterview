# Kafka Architecture — Kiến Trúc Cluster

> Hiểu kiến trúc Apache Kafka: Broker (Máy Chủ Broker), Cluster (Cụm), replication (nhân bản), ISR (In-Sync Replicas — Bản Sao Đồng Bộ), leader election (bầu leader), và sự chuyển đổi từ ZooKeeper sang KRaft (Kafka Raft — Giao Thức Đồng Thuận Raft Trong Kafka).

## Mục Lục

1. [Kafka Là Gì?](#kafka-là-gì)
2. [Broker & Cluster](#broker--cluster)
3. [Partition & Replication](#partition--replication)
4. [ISR & Leader Election](#isr--leader-election)
5. [ZooKeeper vs KRaft](#zookeeper-vs-kraft)
6. [Request Flow](#request-flow)
7. [Storage Model](#storage-model)
8. [High Availability Design](#high-availability-design)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kafka Là Gì?

**Apache Kafka** là **distributed event streaming platform (nền tảng streaming sự kiện phân tán)** — hệ thống publish-subscribe được thiết kế như **distributed commit log (nhật ký commit phân tán)** có khả năng:

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **High Throughput (Thông Lượng Cao)** | Hàng triệu message/giây trên cluster |
| **Durability (Bền Vững)** | Ghi disk, replication across brokers |
| **Scalability (Mở Rộng)** | Thêm broker, partition để scale |
| **Fault Tolerance (Chịu Lỗi)** | Tự động failover khi broker/partition leader chết |
| **Replay (Phát Lại)** | Consumer đọc lại từ offset bất kỳ trong retention |

```
Kafka mental model:
  Topic = logical stream
  Partition = ordered, immutable sequence of records
  Offset = position trong partition (0, 1, 2, ...)
```

---

## Broker & Cluster

### Broker (Máy Chủ Broker)

Mỗi **Kafka Broker** là một server trong cluster, chịu trách nhiệm:

- Nhận message từ producers
- Lưu trữ message trên **local disk (đĩa cục bộ)**
- Phục vụ fetch request từ consumers
- Tham gia replication — làm leader hoặc follower cho partitions

```
Cluster 3 brokers:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Broker 1   │  │  Broker 2   │  │  Broker 3   │
│  ID: 1      │  │  ID: 2      │  │  ID: 3      │
│             │  │             │  │             │
│  P0 leader  │  │  P1 leader  │  │  P2 leader  │
│  P1 follower│  │  P2 follower│  │  P0 follower│
│  P2 follower│  │  P0 follower│  │  P1 follower│
└─────────────┘  └─────────────┘  └─────────────┘
```

### Cluster Controller

Một broker được bầu làm **Controller (Bộ Điều Khiển)** — quản lý:

- Leader election cho partitions
- Broker membership (thành viên cluster)
- Topic creation/deletion
- Partition reassignment

> **KRaft mode:** Controller logic chạy trên **KRaft quorum (nhóm đồng thuận KRaft)** thay vì ZooKeeper.

### Bootstrap Servers

Client kết nối qua **`bootstrap.servers`** — chỉ cần 1–3 broker addresses để discover toàn bộ cluster metadata.

```properties
# Producer/Consumer config
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
```

---

## Partition & Replication

### Partition (Phân Vùng)

Mỗi topic chia thành **partitions** — đơn vị **parallelism (song song hóa)** và **ordering (thứ tự)**:

- Thứ tự **FIFO** được đảm bảo **trong** partition
- **Không** đảm bảo thứ tự **giữa** các partitions
- Mỗi partition có **một leader** và **n−1 followers** (với replication factor = n)

### Replication Factor (Hệ Số Nhân Bản)

```bash
# Tạo topic với RF=3 — mỗi partition có 3 bản sao
kafka-topics.sh --create --topic payments \
  --partitions 6 --replication-factor 3 \
  --bootstrap-server localhost:9092
```

| RF | Ý Nghĩa | Production |
| -- | ------- | ---------- |
| 1 | Không replication — mất data khi broker chết | Chỉ dev/test |
| 2 | Chịu được 1 broker fail | Tối thiểu staging |
| 3 | Industry standard — chịu 1 broker fail an toàn | Khuyến nghị production |

**Quy tắc:** `replication-factor` ≤ số brokers trong cluster.

---

## ISR & Leader Election

### ISR (In-Sync Replicas — Bản Sao Đồng Bộ)

**ISR** là tập hợp replicas **đã bắt kịp (caught up)** với leader — follower lag trong ngưỡng `replica.lag.time.max.ms`.

```
Partition P0:
  Leader: Broker 1  ◄── producers write here
  ISR:    [Broker 1, Broker 2, Broker 3]
  Out-of-sync: (none)

Khi Broker 3 lag quá lâu:
  ISR: [Broker 1, Broker 2]  — Broker 3 bị loại khỏi ISR
```

### Leader Election (Bầu Leader)

Khi **leader chết**, controller trigger election từ ISR:

1. Chọn follower trong ISR làm leader mới
2. Các follower khác replicate từ leader mới
3. Producers/consumers redirect qua metadata update

```
Timeline:
  T0: P0 leader = Broker 1
  T1: Broker 1 crash
  T2: Controller elect Broker 2 làm leader (từ ISR)
  T3: Clients refresh metadata → connect Broker 2
```

> **Unclean leader election (bầu leader không sạch):** Cho phép replica **ngoài ISR** làm leader → có thể **mất message**. Production nên set `unclean.leader.election.enable=false`.

### min.insync.replicas

```properties
# Topic-level hoặc broker default
min.insync.replicas=2
```

Kết hợp với producer `acks=all`:

- ISR ≥ min.insync.replicas → write success
- ISR < min.insync.replicas → `NotEnoughReplicasException` — producer fail thay vì mất durability

---

## ZooKeeper vs KRaft

### ZooKeeper (Legacy — Cũ)

Trước Kafka 3.x, cluster dùng **Apache ZooKeeper** cho:

- Broker registration
- Controller election
- Topic/partition metadata
- ACL storage (một số version)

```
┌──────────────┐     metadata     ┌──────────────┐
│  ZooKeeper   │◄────────────────►│ Kafka Brokers│
│  Ensemble    │                  │              │
│  (3 or 5)    │                  └──────────────┘
└──────────────┘
```

**Nhược điểm:** Vận hành thêm ZooKeeper ensemble, metadata bottleneck khi có hàng nghìn partitions.

### KRaft (Kafka Raft — Mới)

**KRaft mode** loại bỏ ZooKeeper — metadata lưu trong **internal topic** `__cluster_metadata`, quản lý bởi Raft quorum.

| Tiêu Chí | ZooKeeper | KRaft |
| -------- | --------- | ----- |
| **Ops complexity** | 2 hệ thống riêng | Chỉ Kafka |
| **Metadata scalability** | Giới hạn | Hàng triệu partitions |
| **Failover time** | Vài giây | Nhanh hơn (ms–s) |
| **Status** | Deprecated | Default từ Kafka 4.0 |

```properties
# KRaft broker config (rút gọn)
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
```

> **Phỏng vấn:** Biết KRaft thay ZooKeeper; production mới nên dùng KRaft. MSK và Confluent Cloud đã hỗ trợ KRaft.

---

## Request Flow

### Producer Write Path

```
Producer                    Leader Broker              Followers
    │                            │                        │
    │── Produce request ────────►│                        │
    │    (partition, records)    │── replicate ──────────►│
    │                            │◄── ack ────────────────│
    │◄── ack (per acks config) ──│                        │
```

### Consumer Read Path

```
Consumer                    Leader Broker
    │                            │
    │── Fetch request ──────────►│
    │   (partition, offset)      │
    │◄── records ────────────────│
    │                            │
    │── Commit offset ──────────►│ (hoặc __consumer_offsets topic)
```

Consumer **luôn đọc từ leader** — followers chỉ dùng cho replication và failover.

---

## Storage Model

### Log Segments

Mỗi partition lưu dưới dạng **log segments** trên disk:

```
/var/kafka/data/topic-orders-0/
  00000000000000000000.log   ← segment file
  00000000000000000000.index ← offset index (tìm nhanh)
  00000000000000000000.timeindex
  00000000000123456789.log   ← segment mới khi đủ size/time
```

| Config | Mô Tả | Default |
| ------ | ----- | ------- |
| `log.segment.bytes` | Kích thước tối đa mỗi segment | 1 GB |
| `log.retention.hours` | Giữ message bao lâu | 168h (7 ngày) |
| `log.retention.bytes` | Giữ tối đa bao nhiêu bytes/partition | unlimited |

### Zero-Copy (Không Sao Chép)

Kafka dùng **sendfile()** system call — transfer data từ disk → network socket mà không copy qua user space → throughput cao.

---

## High Availability Design

### Production Checklist

| Item | Khuyến Nghị |
| ---- | ----------- |
| **Brokers** | ≥ 3 brokers across AZs (Availability Zones — Vùng Sẵn Sàng) |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Producer acks** | `all` (hoặc `-1`) |
| **unclean.leader.election** | `false` |
| **Monitoring** | Under-replicated partitions, ISR shrink, controller changes |

### Failure Scenarios

| Scenario | Hành Vi | Action |
| -------- | ------- | ------ |
| 1 follower chết | ISR giảm, vẫn serve | Replace broker, wait re-sync |
| Leader chết | Election từ ISR | Auto — monitor election time |
| 2/3 brokers chết (RF=3) | Một số partitions unavailable | Critical outage — restore brokers |
| Network partition | Split-brain risk với misconfig | Proper quorum, min ISR |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka lưu message ở đâu?

**Gợi ý trả lời:** Trên **local disk** của broker dưới dạng append-only log segments — không phải in-memory queue. Đây là lý do Kafka có durability và replay. Memory dùng cho **page cache (bộ nhớ đệm trang)** OS để tăng tốc đọc.

### Câu 2: ISR là gì? Tại sao quan trọng?

**Gợi ý trả lời:** **ISR** là replicas đồng bộ với leader. Producer `acks=all` chỉ chờ ISR ack. Khi leader fail, chỉ replica trong ISR mới được bầu leader — tránh mất data. Follower lag quá `replica.lag.time.max.ms` bị loại khỏi ISR.

### Câu 3: Replication factor 3 nghĩa là gì?

**Gợi ý trả lời:** Mỗi partition có **3 bản sao** trên 3 broker khác nhau. Chịu được mất 1 broker mà không mất data (với min.insync.replicas=2 và acks=all). Không phải "3 copies của toàn topic trên 1 broker".

### Câu 4: ZooKeeper dùng để làm gì? KRaft thay thế ra sao?

**Gợi ý trả lời:** ZooKeeper lưu cluster metadata và điều phối controller election. **KRaft** dùng Raft consensus nội bộ Kafka — metadata topic + quorum controllers — loại bỏ dependency ZooKeeper, scale metadata tốt hơn.

### Câu 5: Consumer đọc từ leader hay follower?

**Gợi ý trả lời:** **Leader only** — đảm bảo đọc data mới nhất. Follower chỉ replicate. (Kafka có thêm **follower fetching** cho một số use case nhưng default consumer đọc leader.)

---

**Xem tiếp:** [2-topics-partitions.md](./2-topics-partitions.md) — thiết kế topic và partitioning strategy.
