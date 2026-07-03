# Kafka Connect — Tích Hợp Source Và Sink

> Kafka Connect framework: Source Connector (Bộ Kết Nối Nguồn), Sink Connector (Bộ Kết Nối Đích), distributed mode (chế độ phân tán), Single Message Transform (SMT — Biến Đổi Tin Nhắn Đơn), và Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu) với Debezium.

## Mục Lục

1. [Kafka Connect Là Gì?](#kafka-connect-là-gì)
2. [Architecture](#architecture)
3. [Source vs Sink Connectors](#source-vs-sink-connectors)
4. [Standalone vs Distributed Mode](#standalone-vs-distributed-mode)
5. [Debezium CDC](#debezium-cdc)
6. [Single Message Transforms](#single-message-transforms)
7. [Error Handling & DLQ](#error-handling--dlq)
8. [Popular Connectors](#popular-connectors)
9. [Operations](#operations)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kafka Connect Là Gì?

**Kafka Connect** là framework **scalable, reliable (mở rộng, tin cậy)** để stream data giữa Apache Kafka và external systems — **không cần viết custom producer/consumer** cho mỗi integration.

```
External System          Kafka Connect           Kafka
┌─────────────┐         ┌─────────────┐        ┌─────────┐
│ PostgreSQL  │──CDC───►│   Source    │───────►│  Topic  │
│ MongoDB     │         │  Connector  │        │         │
│ S3          │         └─────────────┘        └────┬────┘
└─────────────┘                                     │
┌─────────────┐         ┌─────────────┐               │
│ Elasticsearch│◄───────│    Sink     │◄──────────────┘
│ JDBC DB     │         │  Connector  │
│ S3          │         └─────────────┘
└─────────────┘
```

| Lợi Ích | Mô Tả |
| ------- | ----- |
| **No custom code** | Config-driven integration |
| **Scalability** | Distributed workers, task parallelism |
| **Fault tolerance** | Offset tracking, task rebalance |
| **Ecosystem** | Hàng trăm connectors (Confluent Hub) |

---

## Architecture

### Components

| Component | Vai Trò |
| --------- | ------- |
| **Connector** | High-level job — ví dụ `JdbcSourceConnector` |
| **Task** | Unit of work — parallel instances của connector |
| **Worker** | JVM process chạy connectors/tasks |
| **Converter** | Serialize/deserialize — JsonConverter, AvroConverter |
| **Transform (SMT)** | Lightweight message transformation |

```
Connector "debezium-postgres" (1 instance)
    ├── Task 0 → partition table A, B
    ├── Task 1 → partition table C, D
    └── Task 2 → partition table E

Workers (distributed):
    Worker 1: Task 0, Task 2
    Worker 2: Task 1
```

### Offset Storage

Source connectors lưu offset (vị trí đọc) trong topic `connect-offsets`:

```
JDBC: last processed primary key / timestamp
Debezium: binlog position (LSN — Log Sequence Number)
File: byte offset
```

---

## Source vs Sink Connectors

### Source Connector

Đọc từ external system → ghi vào Kafka topics.

```json
{
  "name": "jdbc-source-orders",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://localhost:5432/shop",
    "table.whitelist": "orders",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "db.",
    "tasks.max": "3"
  }
}
```

→ Topic: `db.orders`

### Sink Connector

Đọc từ Kafka topics → ghi vào external system.

```json
{
  "name": "es-sink-orders",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "topics": "db.orders",
    "connection.url": "http://localhost:9200",
    "key.ignore": "false",
    "schema.ignore": "true",
    "tasks.max": "2"
  }
}
```

---

## Standalone vs Distributed Mode

| Mode | Mô Tả | Use Case |
| ---- | ----- | -------- |
| **Standalone** | 1 process, local config files | Dev, testing |
| **Distributed** | Nhiều workers, REST API, HA | Production |

### Distributed Mode

```
REST API: POST /connectors
    │
    ▼
Connect Cluster (3 workers)
    │
    ├── Worker 1: Connector A, Task 0
    ├── Worker 2: Connector A, Task 1
    └── Worker 3: Connector B, Task 0

Worker die → tasks rebalance sang worker khác
```

```bash
# Tạo connector qua REST
curl -X POST -H "Content-Type: application/json" \
  http://localhost:8083/connectors \
  -d @debezium-postgres.json

# Status
curl http://localhost:8083/connectors/debezium-postgres/status
```

---

## Debezium CDC

**Debezium** là source connector phổ biến nhất cho **Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu)** — capture row-level changes từ database transaction log.

```
PostgreSQL WAL (Write-Ahead Log)
        │
        ▼
Debezium Connector
        │
        ▼
Kafka Topic: dbserver1.shop.orders
        │
        ├── Key: { "id": 123 }
        └── Value: {
              "before": null,
              "after": { "id": 123, "status": "CREATED", ... },
              "op": "c",           // create, update, delete
              "ts_ms": 1710000000
            }
```

### Supported Databases

| Database | Log Source |
| -------- | ---------- |
| PostgreSQL | Logical decoding / WAL |
| MySQL | Binlog |
| SQL Server | CDC tables |
| MongoDB | Oplog |
| Oracle | LogMiner |

### CDC Event Structure

```json
{
  "op": "u",
  "before": { "id": 1, "status": "PENDING", "amount": 100 },
  "after":  { "id": 1, "status": "PAID", "amount": 100 },
  "source": { "db": "shop", "table": "orders", "lsn": 12345678 }
}
```

| op | Ý Nghĩa |
| -- | ------- |
| `c` | Create (insert) |
| `u` | Update |
| `d` | Delete |
| `r` | Read (snapshot) |

### CDC Best Practices

| Practice | Mô Tả |
| -------- | ----- |
| **Compacted topic cho state** | Key = primary key |
| **Schema Registry** | Avro schema cho CDC events |
| **Snapshot mode** | `initial` — full snapshot rồi streaming |
| **Monitor lag** | Debezium lag = DB changes chưa publish |

---

## Single Message Transforms

**SMT (Single Message Transform)** — biến đổi nhẹ message trong pipeline, không cần Kafka Streams:

```json
{
  "transforms": "extractKey,renameField",
  "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
  "transforms.extractKey.field": "id",
  "transforms.renameField.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.renameField.renames": "status:order_status"
}
```

| Transform | Mô Tả |
| --------- | ----- |
| `ExtractField` | Lấy nested field làm key |
| `ReplaceField` | Rename/remove fields |
| `Flatten` | Flatten nested struct |
| `TimestampRouter` | Route theo timestamp |
| `Filter` | Drop message theo condition |

> **Giới hạn:** SMT cho transform đơn giản — logic phức tạp dùng Kafka Streams hoặc custom SMT.

---

## Error Handling & DLQ

```json
{
  "errors.tolerance": "all",
  "errors.deadletterqueue.topic.name": "connect-dlq",
  "errors.deadletterqueue.context.headers.enable": "true",
  "errors.log.enable": "true",
  "errors.log.include.messages": "true"
}
```

| errors.tolerance | Hành Vi |
| ---------------- | ------- |
| `none` (default) | Task fail khi lỗi |
| `all` | Skip lỗi, gửi DLQ |

```
Sink fail (bad record)
    │
    ▼
errors.tolerance=all
    │
    ├──► DLQ topic (connect-dlq) + error headers
    └──► Continue processing other messages
```

---

## Popular Connectors

| Connector | Loại | Use Case |
| --------- | ---- | -------- |
| **Debezium** | Source | CDC từ databases |
| **JdbcSource/Sink** | Both | JDBC databases |
| **S3 Sink** | Sink | Data lake archival |
| **Elasticsearch Sink** | Sink | Search index sync |
| **MongoDB Connector** | Both | Mongo ↔ Kafka |
| **HTTP Source** | Source | Poll REST APIs |
| **Iceberg Sink** | Sink | Table format data lake |

---

## Operations

### Monitoring

| Metric | Ý Nghĩa |
| ------ | ------- |
| `connector-status` | RUNNING, FAILED, PAUSED |
| `task-count` | Active tasks |
| `source-record-poll-rate` | Throughput |
| `sink-record-send-rate` | Write rate |
| Consumer lag (sink) | Sink chậm hơn topic |

### Common Operations

```bash
# Pause connector (maintenance)
curl -X PUT http://localhost:8083/connectors/my-connector/pause

# Restart failed task
curl -X POST http://localhost:8083/connectors/my-connector/tasks/0/restart

# Update config
curl -X PUT -H "Content-Type: application/json" \
  http://localhost:8083/connectors/my-connector/config \
  -d '{ ... new config ... }'
```

### Scaling

```
Tăng tasks.max → more parallelism
Điều kiện: source có thể partition (tables, shards)
Sink: đảm bảo target system chịu được parallel writes
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka Connect khác custom producer/consumer thế nào?

**Gợi ý trả lời:** Connect là **framework config-driven** — connector có sẵn cho JDBC, S3, Debezium. Không viết code poll/push. Có distributed mode, offset management, REST API. Custom code chỉ khi logic đặc biệt.

### Câu 2: Debezium CDC hoạt động thế nào?

**Gợi ý trả lời:** Đọc **database transaction log** (WAL, binlog) — capture insert/update/delete realtime. Publish lên Kafka với before/after payload. Không dùng polling `SELECT *` — low latency, ít load DB.

### Câu 3: Source connector scale thế nào?

**Gợi ý trả lời:** Tăng `tasks.max` — mỗi task xử lý subset (tables, partitions). Distributed workers rebalance tasks. Giới hạn bởi khả năng partition của source.

### Câu 4: Sink connector lag — xử lý thế nào?

**Gợi ý trả lời:** Sink là consumer group — lag = chậm hơn topic. Scale `tasks.max`, optimize target write (batch), check errors.tolerance không skip quá nhiều. Monitor DLQ.

### Câu 5: SMT vs Kafka Streams?

**Gợi ý trả lời:** **SMT** — lightweight per-record transform trong Connect pipeline (rename, extract key). **Kafka Streams** — full stream processing (join, aggregate, window). SMT cho ETL đơn giản; Streams cho logic phức tạp.

---

**Xem tiếp:** [8-kafka-streams.md](./8-kafka-streams.md) — stream processing với Kafka Streams.
