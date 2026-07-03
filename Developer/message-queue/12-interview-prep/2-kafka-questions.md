# Câu Hỏi Sâu Apache Kafka — Deep Dive Questions

> 20 câu hỏi phỏng vấn chuyên sâu về Apache Kafka, kèm đáp án chi tiết và liên kết tài liệu.

## Mục Lục

1. [Architecture & Internals](#architecture--internals)
2. [Producers](#producers)
3. [Consumers & Consumer Groups](#consumers--consumer-groups)
4. [Operations & Tuning](#operations--tuning)
5. [Advanced Topics](#advanced-topics)

---

## Architecture & Internals

### Câu 1: Kafka broker architecture — các thành phần chính?

**Đáp án:**

- **Broker:** Server lưu trữ partitions, serve produce/consume requests
- **Cluster:** Nhiều brokers; mỗi partition có leader + replicas
- **ZooKeeper / KRaft:** Metadata coordination (KRaft thay ZooKeeper từ Kafka 3.x)
- **ISR (In-Sync Replicas):** Replicas đã sync với leader — chỉ ISR được elect leader
- **Controller:** Broker quản lý partition leadership, rebalancing

📖 [03-apache-kafka/1-kafka-architecture.md](../03-apache-kafka/1-kafka-architecture.md)

---

### Câu 2: Leader và Follower replica hoạt động thế nào?

**Producer** ghi vào **leader partition**; followers replicate từ leader.

**Consumer** chỉ đọc từ leader (trước Kafka 2.4; sau có `read_from_followers` tùy config).

**Failover:** Leader chết → controller elect leader mới từ ISR → consumers/producers reconnect.

**Rủi ro:** `acks=1` + leader fail trước replicate → **message loss**. Fix: `acks=all`.

---

### Câu 3: Log segment và retention hoạt động ra sao?

Partition = **append-only log** chia thành **segments** (files trên disk).

- `log.segment.bytes` — kích thước mỗi segment
- `retention.ms` / `retention.bytes` — xóa segment cũ
- **Compaction (nén log):** Giữ latest value per key — dùng cho changelog topics

---

## Producers

### Câu 4: Producer batching — `batch.size` và `linger.ms`?

**Batching (Gom Lô):** Producer gom nhiều records trước khi gửi — tăng throughput, tăng latency nhẹ.

| Config | Ý Nghĩa | Trade-off |
| ------ | ------- | --------- |
| `batch.size` | Max bytes per batch | Lớn hơn → throughput cao |
| `linger.ms` | Chờ thêm records | > 0 → latency tăng, throughput tăng |
| `compression.type` | lz4, snappy, zstd | CPU vs network bandwidth |

**Production tip:** `linger.ms=5-20`, `compression.type=lz4` cho throughput cao.

📖 [03-apache-kafka/4-producers-serialization.md](../03-apache-kafka/4-producers-serialization.md)

---

### Câu 5: Idempotent Producer — cơ chế hoạt động?

`enable.idempotence=true` → broker deduplicate dựa trên **Producer ID (PID)** + **Sequence Number** per partition.

**Giải quyết:** Retry do network timeout gây duplicate trên broker — không phải end-to-end exactly-once.

**Yêu cầu:** `acks=all`, `retries>0`, `max.in.flight.requests.per.connection≤5`.

---

### Câu 6: Transactional Producer — khi nào dùng?

**Use case:** Atomic write nhiều partitions/topics; consume-process-produce exactly-once trong Kafka.

```java
producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.commitTransaction(); // hoặc abortTransaction()
```

**Overhead:** Higher latency; dùng khi thực sự cần — không default cho mọi producer.

---

## Consumers & Consumer Groups

### Câu 7: Offset commit — auto vs manual?

| Mode | Behavior | Risk |
| ---- | -------- | ---- |
| **Auto commit** | Commit định kỳ (`auto.commit.interval.ms`) | At-least-once hoặc duplicate nếu crash giữa process và commit |
| **Manual sync** | `commitSync()` sau xử lý | Safer, chậm hơn |
| **Manual async** | `commitAsync()` | Nhanh, có thể mất commit order |

**Best practice:** Manual commit **sau** side effects (DB write) thành công.

📖 [03-apache-kafka/5-offset-management.md](../03-apache-kafka/5-offset-management.md)

---

### Câu 8: `max.poll.interval.ms` vs `session.timeout.ms`?

- **`session.timeout.ms`:** Heartbeat timeout — consumer considered dead → rebalance
- **`max.poll.interval.ms`:** Max time giữa hai `poll()` — handler quá chậm → rebalance

**Lỗi thường gặp:** Handler xử lý 5 phút nhưng `max.poll.interval.ms=300000` (5 phút) → rebalance loop.

**Fix:** Tăng `max.poll.interval.ms` HOẶC giảm `max.poll.records` HOẶC async processing với pause/resume.

---

### Câu 9: Consumer group rebalancing — các assignor?

| Assignor | Behavior |
| -------- | -------- |
| **Range** | Chia partition theo range — có thể imbalance |
| **RoundRobin** | Phân bố đều hơn |
| **Sticky** | Giữ assignment cũ khi có thể |
| **Cooperative Sticky** | Incremental rebalance — chỉ revoke partitions cần thiết |

**Kafka 2.4+:** Dùng `CooperativeStickyAssignor` — giảm stop-the-world rebalance.

---

### Câu 10: Tại sao số consumers > số partitions là lãng phí?

Mỗi partition chỉ assign cho **1 consumer** trong group. Consumer thừa **idle (nhàn rỗi)**.

```
3 partitions, 5 consumers → 3 active, 2 idle
Scale: tăng partition count TRƯỚC khi thêm consumers
```

**Lưu ý:** Tăng partition **không** reorder existing messages — cần plan từ đầu.

📖 [07-performance-scaling/3-consumer-scaling.md](../07-performance-scaling/3-consumer-scaling.md)

---

## Operations & Tuning

### Câu 11: Hot partition — detect và fix?

**Detect:** Per-partition lag metric — 1 partition lag cao hơn hẳn.

**Nguyên nhân:** Partition key skew (ví dụ: `country=US` = 80% traffic).

**Fix:**
- Salting key: `hash(userId + random(0,9))`
- Custom partitioner
- Tách topic theo traffic pattern

📖 [07-performance-scaling/2-partitioning-strategies.md](../07-performance-scaling/2-partitioning-strategies.md)

---

### Câu 12: Under-replicated partitions — ý nghĩa?

Partition có replicas chưa trong ISR — **risk mất data** nếu leader fail.

**Nguyên nhân:** Broker down, network partition, disk slow.

**Action:** Alert immediately; fix broker; không deploy producer `acks=all` nếu `min.insync.replicas` không đạt.

---

### Câu 13: Khi nào tăng partition count?

**Tăng khi:**
- Consumer lag không giảm dù đã optimize handler
- Cần parallelism > current partition count
- Throughput write vượt single partition limit (~10MB/s)

**Không tăng khi:**
- Chỉ để "có nhiều partitions" — overhead metadata
- Cần ordering cross-partition

**Quy tắc:** Bắt đầu `partitions = expected_peak_throughput / per_partition_throughput`.

---

### Câu 14: Kafka Connect — source vs sink connector?

**Source Connector:** External system → Kafka (Debezium CDC, JDBC source)

**Sink Connector:** Kafka → External system (Elasticsearch sink, S3 sink)

**Worker:** Chạy connectors; scale workers cho throughput.

📖 [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md)

---

## Advanced Topics

### Câu 15: Schema Registry — tại sao cần?

**Vấn đề:** Producer/consumer dùng schema khác nhau → deserialization fail.

**Schema Registry:** Central schema storage + compatibility check (backward/forward).

**Formats:** Avro (phổ biến), Protobuf, JSON Schema.

📖 [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md)

---

### Câu 16: Kafka Streams vs external stream processor (Flink)?

| | Kafka Streams | Flink |
| - | ------------- | ----- |
| **Deployment** | Library trong app | Cluster riêng |
| **State** | Local RocksDB + changelog | Managed state |
| **Scale** | Per application instance | Cluster scale |
| **Phù hợp** | Microservice embedding | Large-scale analytics |

📖 [03-apache-kafka/8-kafka-streams.md](../03-apache-kafka/8-kafka-streams.md)

---

### Câu 17: Log compaction vs delete retention?

**Delete:** Xóa segment cũ theo time/size — event stream thông thường.

**Compaction:** Giữ **latest record per key** — dùng cho changelog (KTables, config topics).

---

### Câu 18: Replay messages — cách thực hiện an toàn?

**Cách:**
1. Reset consumer group offset: `kafka-consumer-groups --reset-offsets`
2. Tạo consumer group mới đọc từ `earliest`
3. Clone topic → replay topic

**An toàn:**
- Replay vào **staging** trước
- Consumer phải **idempotent**
- Thông báo downstream về duplicate window

---

### Câu 19: KRaft vs ZooKeeper?

**KRaft (Kafka Raft):** Metadata quorum built-in — không cần ZooKeeper.

**Lợi ích:** Đơn giản ops, faster metadata operations, scalability.

**Migration:** Kafka 3.x hỗ trợ KRaft mode; ZooKeeper deprecated.

---

### Câu 20: MSK (Managed Streaming for Kafka) — điểm cần biết khi phỏng vấn?

- AWS quản lý broker patching, scaling
- MSK Serverless vs Provisioned — cost/throughput trade-off
- IAM authentication, VPC networking
- MSK Connect cho Debezium connectors

📖 [10-cloud-managed/1-aws-msk.md](../10-cloud-managed/1-aws-msk.md)

---

## Bảng Tóm Tắt Nhanh

| Chủ Đề | Config / Concept Quan Trọng |
| ------ | --------------------------- |
| Durability | `acks=all`, `min.insync.replicas=2` |
| Throughput | `batch.size`, `linger.ms`, `compression` |
| Consumer safety | Manual commit sau side effect |
| Scale | partitions ≥ consumers cần active |
| Ordering | Same partition key |
| Exactly-once | Idempotent producer + transactions + idempotent consumer |

---

**Cập Nhật Lần Cuối:** 2026-07-03
