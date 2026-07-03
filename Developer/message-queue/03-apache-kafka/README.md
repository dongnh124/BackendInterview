# Apache Kafka — Tổng Quan Ecosystem

> Chủ đề chuyên sâu về Apache Kafka — distributed event streaming platform (nền tảng streaming sự kiện phân tán) phổ biến nhất trong production: kiến trúc cluster, topics/partitions, consumer groups, producers, offset management, Schema Registry, Kafka Connect, và Kafka Streams.

## Mục Lục

1. [Tại Sao Học Kafka](#tại-sao-học-kafka)
2. [Kafka Trong Ecosystem](#kafka-trong-ecosystem)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học Kafka

Apache Kafka là **Event Broker (Broker Sự Kiện)** được dùng rộng rãi nhất cho event streaming, log aggregation, và Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu). Sau khi nắm [01-fundamentals](../01-fundamentals/README.md), Kafka là broker **bắt buộc** cho backend engineer và system design interview.

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **Architecture (Kiến Trúc)** | Broker, partition, ISR — nền tảng troubleshoot |
| **Topics & Partitions** | Thiết kế sai → hot partition, mất ordering |
| **Consumer Groups** | Scale consumer, rebalancing — hay gặp production issue |
| **Producer Config** | acks, idempotence — trade-off durability vs latency |
| **Offset Management** | Commit sai → duplicate hoặc message loss |
| **Schema Registry** | Schema evolution trong microservices |
| **Kafka Connect & Streams** | CDC pipeline, stream processing |

> **Điều kiện tiên quyết:** Đã học [delivery guarantees](../01-fundamentals/3-delivery-guarantees.md), [ordering](../01-fundamentals/4-ordering-and-sequencing.md), và [Outbox Pattern](../02-architecture-patterns/4-outbox-inbox-pattern.md).

---

## Kafka Trong Ecosystem

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    APACHE KAFKA ECOSYSTEM                                │
│                                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │  Producers  │   │   Kafka     │   │  Consumers  │   │ Kafka       │  │
│  │  (Apps,     │──►│   Cluster   │──►│  (Apps,     │   │ Streams     │  │
│  │   Connect)  │   │  (Brokers)  │   │   Connect)  │   │ (Processing)│  │
│  └─────────────┘   └──────┬──────┘   └─────────────┘   └─────────────┘  │
│                           │                                              │
│                    ┌──────┴──────┐                                       │
│                    │ Schema      │   Kafka Connect: Source/Sink          │
│                    │ Registry    │   (Debezium CDC, S3, JDBC...)         │
│                    └─────────────┘                                       │
│                                                                         │
│  Managed: AWS MSK, Confluent Cloud, Azure Event Hubs (Kafka endpoint)   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Thành phần chính:**

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Kafka Broker** | Lưu trữ và phục vụ messages theo topic/partition |
| **ZooKeeper / KRaft** | Metadata coordination (điều phối metadata) — KRaft thay ZooKeeper từ Kafka 3.x |
| **Schema Registry** | Quản lý Avro/Protobuf/JSON Schema, compatibility rules |
| **Kafka Connect** | Framework tích hợp source/sink không cần viết producer/consumer |
| **Kafka Streams** | Stream processing library nhúng trong application |

---

## Kiến Trúc Tổng Quan

```
                    ┌──────────────────────────────────────┐
                    │         KAFKA CLUSTER                 │
                    │                                      │
  Producer ────────►│  Broker 1    Broker 2    Broker 3   │
  (key=order-123)   │  ┌──────┐   ┌──────┐   ┌──────┐    │
                    │  │ P0   │   │ P1   │   │ P2   │    │◄── Topic "orders"
                    │  │leader│   │leader│   │leader│    │    (3 partitions)
                    │  └──────┘   └──────┘   └──────┘    │
                    │     ▲ replicas (bản sao) trên broker khác
                    │                                      │
                    │  Consumer Group "inventory-svc"        │
                    │  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
                    │  │Consumer1│ │Consumer2│ │Consumer3│ │
                    │  │  → P0   │ │  → P1   │ │  → P2   │ │
                    │  └─────────┘ └─────────┘ └─────────┘ │
                    └──────────────────────────────────────┘
```

**Luồng cơ bản:**

1. Producer gửi record vào **topic** — broker route theo **partition key**
2. Mỗi partition là **append-only log (nhật ký chỉ ghi thêm)** có thứ tự FIFO
3. Consumer thuộc **consumer group** — mỗi partition chỉ được 1 consumer trong group đọc
4. Consumer **commit offset** để đánh dấu vị trí đã xử lý

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 12–16 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-kafka-architecture.md](./1-kafka-architecture.md) | Broker, cluster, KRaft, ISR | 2 giờ |
| 2 | [2-topics-partitions.md](./2-topics-partitions.md) | Topic design, partitioning, key routing | 2 giờ |
| 3 | [3-consumer-groups.md](./3-consumer-groups.md) | Consumer groups, rebalancing, scale-out | 2 giờ |
| 4 | [4-producers-serialization.md](./4-producers-serialization.md) | acks, batching, compression, idempotence | 1.5 giờ |
| 5 | [5-offset-management.md](./5-offset-management.md) | Auto vs manual commit, offset reset | 1.5 giờ |
| 6 | [6-schema-registry.md](./6-schema-registry.md) | Avro, Protobuf, schema evolution | 1.5 giờ |
| 7 | [7-kafka-connect.md](./7-kafka-connect.md) | Source/sink connectors, CDC | 1.5 giờ |
| 8 | [8-kafka-streams.md](./8-kafka-streams.md) | Stream processing, windowing | 2 giờ |

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4 → 5. Học **architecture + partitions + consumer groups** trước vì đây là nền tảng cho mọi câu hỏi phỏng vấn Kafka. Schema Registry, Connect, Streams học sau khi đã chạy producer/consumer thực tế.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-kafka-architecture.md](./1-kafka-architecture.md) | Broker internals, replication, ISR, leader election, KRaft vs ZooKeeper |
| [2-topics-partitions.md](./2-topics-partitions.md) | Partition count, key strategy, compaction, retention |
| [3-consumer-groups.md](./3-consumer-groups.md) | Group coordinator, rebalance protocols, static membership |
| [4-producers-serialization.md](./4-producers-serialization.md) | Producer configs, serializers, exactly-once producer |
| [5-offset-management.md](./5-offset-management.md) | `__consumer_offsets`, commit strategies, seek |
| [6-schema-registry.md](./6-schema-registry.md) | Schema compatibility, Avro vs JSON, Confluent Schema Registry |
| [7-kafka-connect.md](./7-kafka-connect.md) | Connect architecture, Debezium CDC, error handling |
| [8-kafka-streams.md](./8-kafka-streams.md) | Topology, state stores, joins, windowing |

---

## Bài Tập Thực Hành

### Lab 1: Dựng Kafka Local (30 phút)

```bash
# Docker Compose — KRaft mode (không cần ZooKeeper)
docker compose -f docker-compose-kafka.yml up -d

# Tạo topic
kafka-topics.sh --create --topic orders \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1

# Producer/Consumer CLI
kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092
kafka-console-consumer.sh --topic orders --bootstrap-server localhost:9092 --from-beginning
```

### Lab 2: Partition Key & Ordering (45 phút)

```
1. Gửi 10 message cùng key "user-1" — verify thứ tự trong 1 partition
2. Gửi message không có key — quan sát round-robin partition
3. Tăng partition count — hiểu message cũ không di chuyển partition
```

### Lab 3: Consumer Group Scaling (45 phút)

```
1. Chạy 1 consumer trong group — đọc tất cả partitions
2. Thêm consumer thứ 2 — quan sát rebalance, partition reassignment
3. Thêm consumer thứ 4 (nhiều hơn partitions) — 1 consumer idle
4. Kill 1 consumer — quan sát rebalance và message redistribution
```

### Lab 4: acks & Durability (30 phút)

```
1. acks=0 — kill broker, quan sát message loss
2. acks=all, min.insync.replicas=2 — kill follower, producer vẫn success
3. Kill leader partition — quan sát leader election và producer retry
```

### Lab 5: Schema Registry + Avro (60 phút)

```
1. Start Confluent Schema Registry
2. Register Avro schema cho Order event
3. Producer dùng AvroSerializer — consumer dùng AvroDeserializer
4. Thử thêm field optional — verify backward compatibility
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Kafka khác RabbitMQ ở điểm nào?

**Gợi ý trả lời:** Kafka là **distributed commit log** — message persist theo retention, hỗ trợ replay và nhiều consumer group độc lập. RabbitMQ là **message broker** truyền thống — routing linh hoạt (exchange types), message thường xóa sau ack, phù hợp task queue và RPC. Kafka: throughput cao, event streaming. RabbitMQ: routing phức tạp, latency thấp hơn cho queue nhỏ.

### Câu 2: Topic, Partition, Consumer Group hoạt động thế nào?

**Gợi ý trả lời:** **Topic** là category/logical stream. **Partition** là đơn vị song song và ordering — thứ tự FIFO trong partition. **Consumer Group** chia partitions cho các consumer — mỗi partition chỉ 1 consumer trong group; nhiều group đọc cùng topic độc lập.

### Câu 3: Làm sao scale consumer khi lag tăng?

**Gợi ý trả lời:** (1) Thêm consumer instance trong cùng group — tối đa bằng số partitions. (2) Nếu đã đủ consumer mà vẫn lag → tăng partition count (cần plan key strategy). (3) Tối ưu consumer processing time. (4) Kiểm tra rebalance storm không làm chậm thêm.

### Câu 4: acks=all nghĩa là gì?

**Gợi ý trả lời:** Producer chờ **tất cả in-sync replicas (ISR — Bản Sao Đồng Bộ)** ack trước khi coi message đã ghi thành công. Kết hợp `min.insync.replicas=2` để đảm bảo durability — nếu ISR < min, producer nhận lỗi thay vì ghi vào 1 replica duy nhất.

### Câu 5: Exactly-once trong Kafka hoạt động ra sao?

**Gợi ý trả lời:** Cần **idempotent producer** (`enable.idempotence=true`) + **transactional producer** cho read-process-write. Consumer dùng `isolation.level=read_committed`. Thực tế nhiều team dùng **at-least-once + idempotent consumer** vì transactional API phức tạp hơn.

---

**Xem tiếp:** [1-kafka-architecture.md](./1-kafka-architecture.md) — bắt đầu với kiến trúc Kafka cluster.
