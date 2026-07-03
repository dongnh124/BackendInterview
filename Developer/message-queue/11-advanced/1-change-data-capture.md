# Change Data Capture (CDC) — Bắt Thay Đổi Dữ Liệu

> Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu): capture row-level changes từ database transaction log, publish lên event bus (Kafka), và xây dựng event-driven sync pipeline với Debezium, Kafka Connect, và các pattern sink phổ biến.

## Mục Lục

1. [CDC Là Gì?](#cdc-là-gì)
2. [CDC vs Các Phương Pháp Khác](#cdc-vs-các-phương-pháp-pháp-khác)
3. [Kiến Trúc CDC Pipeline](#kiến-trúc-cdc-pipeline)
4. [Debezium Deep Dive](#debezium-deep-dive)
5. [Snapshot Modes & Initial Load](#snapshot-modes--initial-load)
6. [CDC Event Patterns](#cdc-event-patterns)
7. [Sink Patterns & Downstream Sync](#sink-patterns--downstream-sync)
8. [Operational Concerns](#operational-concerns)
9. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CDC Là Gì?

**Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu)** là kỹ thuật phát hiện và capture mọi thay đổi dữ liệu (INSERT, UPDATE, DELETE) từ source database **ngay khi commit transaction**, rồi publish dưới dạng events cho downstream systems.

```
Traditional Batch ETL:
  Cron job mỗi 15 phút: SELECT * FROM orders WHERE updated_at > ?
  → Stale data, DB load cao, miss deletes

CDC:
  Transaction commit → WAL/Binlog → Debezium → Kafka topic
  → Sub-second latency, đầy đủ create/update/delete
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Source of truth** | OLTP database (PostgreSQL, MySQL, ...) |
| **Capture mechanism** | Transaction log (WAL — Write-Ahead Log, Binlog, Oplog) |
| **Delivery** | Event stream (Kafka topic per table hoặc unified) |
| **Latency** | Millisecond đến vài giây |
| **Ordering** | Per-table hoặc per-partition key |

---

## CDC vs Các Phương Pháp Khác

### So Sánh Tổng Quan

| Phương Pháp | Latency | DB Load | Captures Deletes | Transactional |
| ----------- | ------- | ------- | ---------------- | ------------- |
| **Polling** (`updated_at`) | Phút–giờ | Cao (full scan) | ❌ Khó | ❌ |
| **Trigger-based** | Thấp | Trung bình | ✅ | ⚠️ |
| **Outbox Pattern** | Thấp | Thấp | ✅ (app-controlled) | ✅ |
| **Log-based CDC** | Rất thấp | Rất thấp | ✅ | ✅ (log order) |

### CDC vs Outbox Pattern

```
Outbox Pattern:
  App transaction: UPDATE orders + INSERT outbox_events (cùng TX)
  Relay process đọc outbox → publish Kafka
  ✅ Chỉ events app chủ đích
  ✅ Strong transactional guarantee
  ❌ Cần sửa application code

Log-based CDC (Debezium):
  App transaction: UPDATE orders (bình thường)
  Debezium đọc WAL → publish Kafka
  ✅ Không sửa app code
  ✅ Capture mọi change (kể cả ad-hoc SQL, migrations)
  ❌ Publish mọi column change — cần filter/transform
  ❌ Không biết "business intent" — chỉ biết row changed
```

**Khi nào chọn gì:**

| Scenario | Gợi Ý |
| -------- | ----- |
| Greenfield microservices, domain events rõ ràng | **Outbox** |
| Legacy monolith, nhiều tables, không sửa code | **CDC** |
| Analytics sync toàn bộ DB | **CDC** |
| Chỉ publish `OrderPlaced`, không publish mọi UPDATE | **Outbox** |
| Kết hợp | CDC cho read replicas/search; Outbox cho domain events |

---

## Kiến Trúc CDC Pipeline

### End-to-End Flow

```
┌─────────────┐     WAL/Binlog      ┌─────────────┐     ┌─────────────┐
│ PostgreSQL  │ ──────────────────► │  Debezium   │ ──► │   Kafka     │
│   (OLTP)    │   logical decoding  │  Connector  │     │   Topics    │
└─────────────┘                     └─────────────┘     └──────┬──────┘
                                                               │
                    ┌──────────────────────────────────────────┼──────────┐
                    ▼                    ▼                     ▼          ▼
              ┌──────────┐        ┌──────────┐          ┌──────────┐ ┌────────┐
              │Elastic-  │        │  Redis   │          │ Snowflake│ │ Lambda │
              │ search   │        │  Cache   │          │ / BQ     │ │ (alert)│
              └──────────┘        └──────────┘          └──────────┘ └────────┘
```

### Topic Naming Conventions

| Pattern | Ví Dụ | Use Case |
| ------- | ----- | -------- |
| `{server}.{db}.{table}` | `dbserver1.shop.orders` | Debezium default |
| `{env}.{domain}.{entity}` | `prod.commerce.orders` | Custom SMT routing |
| Single unified topic | `db.changes` (with table header) | Small deployments |

### Key Design

```
Key = primary key của row (e.g. { "id": 12345 })
→ Cùng key vào cùng Kafka partition → ordering per entity
→ Compacted topic: giữ latest state per key
```

---

## Debezium Deep Dive

**Debezium** là open-source CDC platform, chạy như **Kafka Connect Source Connector (Bộ Kết Nối Nguồn Kafka Connect)**.

### Supported Databases

| Database | Log Source | Connector Class |
| -------- | ---------- | --------------- |
| **PostgreSQL** | Logical decoding / WAL | `PostgresConnector` |
| **MySQL** | Binlog | `MySqlConnector` |
| **SQL Server** | CDC tables | `SqlServerConnector` |
| **MongoDB** | Oplog / Change Streams | `MongoDbConnector` |
| **Oracle** | LogMiner | `OracleConnector` |

### PostgreSQL Setup

```sql
-- postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4

-- Tạo replication user
CREATE ROLE debezium REPLICATION LOGIN PASSWORD 'secret';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;
```

### Connector Configuration

```json
{
  "name": "postgres-orders-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "shop",
    "database.server.name": "shop-db",
    "table.include.list": "public.orders,public.order_items",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_orders",
    "publication.name": "dbz_publication",
    "snapshot.mode": "initial",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

### CDC Event Envelope (Debezium Default)

```json
{
  "before": { "id": 1, "status": "PENDING", "amount": 100 },
  "after":  { "id": 1, "status": "PAID", "amount": 100 },
  "source": {
    "version": "2.5.0",
    "connector": "postgresql",
    "name": "shop-db",
    "ts_ms": 1710000000123,
    "snapshot": "false",
    "db": "shop",
    "schema": "public",
    "table": "orders",
    "txId": 987654,
    "lsn": 12345678
  },
  "op": "u",
  "ts_ms": 1710000000456
}
```

| Field | Ý Nghĩa |
| ----- | ------- |
| `op` | `c` create, `u` update, `d` delete, `r` read (snapshot) |
| `before` / `after` | Row state trước/sau change |
| `source.lsn` | Log Sequence Number — offset trong WAL |
| `source.txId` | Transaction ID — group changes cùng TX |

### ExtractNewRecordState SMT

**SMT (Single Message Transform — Biến Đổi Tin Nhắn Đơn)** `ExtractNewRecordState` flatten envelope:

```
Input:  { before, after, op, source }
Output: { id: 1, status: "PAID", amount: 100, __deleted: false }

Delete: { id: 1, __deleted: true }  (delete.handling.mode=rewrite)
```

---

## Snapshot Modes & Initial Load

Khi connector start lần đầu, cần **initial snapshot (ảnh chụp ban đầu)** — đọc toàn bộ existing data trước khi streaming.

| snapshot.mode | Hành Vi | Use Case |
| ------------- | ------- | -------- |
| `initial` (default) | Snapshot rồi streaming | Production mới |
| `initial_only` | Chỉ snapshot, không streaming | One-time export |
| `never` | Chỉ streaming từ bây giờ | DB đã có data elsewhere |
| `when_needed` | Snapshot nếu chưa có offset/signal | Flexible |
| `no_data` | Streaming, schema only | Schema sync |

### Snapshot Process

```
Phase 1: SNAPSHOT
  ├── Acquire global read lock (hoặc MVCC snapshot — Postgres)
  ├── SELECT * FROM each table (chunked)
  ├── Publish events với op="r"
  └── Release lock

Phase 2: STREAMING
  ├── Switch to WAL/Binlog reading
  └── Publish realtime changes (op c/u/d)
```

**Lưu ý production:**

```
⚠️ Snapshot trên bảng lớn (100M+ rows) — mất giờ, tăng DB load
✅ table.include.list — chỉ tables cần thiết
✅ snapshot.fetch.size — control batch size
✅ Chạy snapshot off-peak hours
✅ Monitor replication slot lag (Postgres) — slot không consume → WAL tích tụ
```

### Postgres Replication Slot Warning

```
Replication slot chưa ack → WAL không được recycle → disk full!

Monitor:
  SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
  FROM pg_replication_slots;
```

---

## CDC Event Patterns

### 1. Event Sourcing Bootstrap

```
CDC từ OLTP → compacted topic (key = entity ID)
→ Materialized view consumer build state
→ Không phải Event Sourcing "thuần" nhưng practical bootstrap
```

### 2. Cache Invalidation

```
orders table UPDATE → CDC event → consumer invalidate Redis key `order:{id}`
→ Cache-aside pattern tự động, không cần app publish event
```

### 3. Search Index Sync

```
products INSERT/UPDATE/DELETE → CDC → Elasticsearch Sink Connector
→ Search index luôn sync với DB (eventual consistency vài giây)
```

### 4. Data Warehouse Ingestion

```
All tables CDC → Kafka → S3 (Avro/Parquet) → Snowflake/BigQuery external table
→ Replace nightly batch ETL với near-realtime pipeline
```

### 5. Domain Event Derivation

```
CDC raw change → Kafka Streams / Flink:
  IF orders.status changed PENDING → PAID
  THEN emit OrderPaid domain event (richer semantics)
```

```
Raw CDC:     { op: "u", before: {status:PENDING}, after: {status:PAID} }
Derived:     { eventType: "OrderPaid", orderId: "123", paidAt: "..." }
```

---

## Sink Patterns & Downstream Sync

### Idempotent Consumer (Bắt Buộc)

CDC = **at-least-once delivery** — consumer phải **idempotent (bất biến khi lặp lại)**:

```java
void handleOrderEvent(OrderChange event) {
    // Upsert by primary key — safe to replay
    orderRepository.upsert(event.getId(), event.toEntity());
}
```

Xem thêm: [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md)

### Compacted Topics cho Latest State

```
Topic config: cleanup.policy=compact
Key = primary key
→ Kafka giữ latest record per key
→ New consumer có thể rebuild state từ compacted log
```

### Sink Connector Options

| Sink | Connector | Pattern |
| ---- | --------- | ------- |
| Elasticsearch | `ElasticsearchSinkConnector` | Upsert by document ID |
| JDBC | `JdbcSinkConnector` | Upsert (`insert.mode=upsert`) |
| S3 | `S3SinkConnector` | Time-partitioned Parquet |
| Redis | Custom consumer | Cache set/delete |
| MongoDB | `MongoDbSinkConnector` | ReplaceOne upsert |

### Handling Deletes

```
Option 1: Tombstone (null value) trên compacted topic → consumer delete downstream
Option 2: ExtractNewRecordState với __deleted=true → consumer soft-delete
Option 3: Hard delete downstream khi op="d"
```

---

## Operational Concerns

### Monitoring Checklist

| Metric | Ý Nghĩa | Alert Threshold |
| ------ | ------- | --------------- |
| **Connector status** | RUNNING vs FAILED | FAILED |
| **MilliSecondsBehindSource** | Debezium lag vs DB | > 30s warning, > 5min critical |
| **Replication slot lag** (Postgres) | WAL chưa consume | Disk usage trend |
| **Snapshot progress** | % tables completed | Stuck > expected time |
| **Error rate / DLQ** | Bad records | > 0 sustained |

### Failure Scenarios

```
Connector crash:
  → Offset stored in connect-offsets topic
  → Restart → resume từ last LSN/binlog position
  → Gap nếu slot bị drop — cần re-snapshot

Schema change (ALTER TABLE):
  → Debezium detect schema change → emit schema change event
  → Downstream consumers cần handle new columns (BACKWARD compatible)
  → DROP COLUMN — consumers ignore (Avro) hoặc fail (strict)

DB failover:
  → Connector cần reconnect new primary
  → Verify replication slot / binlog position preserved
  → HA: Debezium on Connect cluster với multiple workers
```

### Performance Tuning

| Parameter | Mô Tả |
| --------- | ----- |
| `max.batch.size` | Records per poll batch |
| `max.queue.size` | Internal queue buffer |
| `poll.interval.ms` | Poll frequency |
| `snapshot.fetch.size` | Rows per snapshot chunk |
| `tasks.max` | Parallel tasks (per table partition) |

---

## Thiết Kế Thực Tế

### Reference Architecture: E-Commerce CDC

```
┌─────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE CDC PLATFORM                       │
│                                                                 │
│  PostgreSQL (shop DB)                                           │
│    ├── orders          ──► shop-db.shop.orders                  │
│    ├── order_items     ──► shop-db.shop.order_items             │
│    ├── products        ──► shop-db.shop.products                │
│    └── customers       ──► shop-db.shop.customers               │
│                                                                 │
│  Downstream:                                                    │
│    orders + order_items ──► Kafka Streams ──► order-summary     │
│    products             ──► ES Sink ──► product search          │
│    customers            ──► Redis consumer ──► customer cache   │
│    ALL tables           ──► S3 Sink ──► data lake (analytics)     │
└─────────────────────────────────────────────────────────────────┘
```

### Production Checklist

```
□ Logical replication / binlog enabled và tested
□ Replication slot monitoring + alerting
□ table.include.list — không capture tables không cần
□ Schema Registry + Avro cho CDC events
□ Idempotent sinks với upsert semantics
□ DLQ cho poison records (errors.tolerance=all)
□ Snapshot scheduled off-peak (lần đầu)
□ Document schema change process với downstream teams
□ Security: connector credentials least privilege (SELECT + REPLICATION only)
□ Network: connector gần DB (same AZ) giảm latency
```

### Anti-Patterns

```
❌ Capture toàn bộ database (100+ tables) không filter — noise và cost
❌ Consumer assume exactly-once — CDC là at-least-once
❌ Ignore replication slot lag — risk disk full trên DB
❌ Không handle DELETE — downstream stale data
❌ Snapshot production DB peak hours
❌ Dùng CDC thay Outbox khi cần domain events có business semantics
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Log-based CDC hoạt động thế nào?

**Gợi ý trả lời:** Database ghi mọi change vào **transaction log** (WAL, binlog) trước khi commit. CDC connector đọc log này như một replication subscriber — capture INSERT/UPDATE/DELETE **không query bảng trực tiếp**. Low latency, minimal DB load, capture deletes. Khác polling `SELECT WHERE updated_at`.

### Câu 2: Debezium snapshot mode `initial` vs `never`?

**Gợi ý trả lời:** **`initial`**: connector mới chạy full table scan (snapshot) publish existing rows (op=`r`), rồi chuyển streaming. **`never`**: bỏ snapshot, chỉ capture changes từ thời điểm start — dùng khi data đã có ở downstream hoặc chỉ cần changes mới.

### Câu 3: CDC vs Outbox — khi nào dùng cái nào?

**Gợi ý trả lời:** **Outbox** khi app control events, cần transactional guarantee với business domain events, greenfield. **CDC** khi sync từ legacy DB không sửa code, analytics toàn bộ tables, search index sync. Có thể kết hợp: CDC cho data sync, Outbox cho domain events.

### Câu 4: Làm sao handle DELETE events downstream?

**Gợi ý trả lời:** Debezium emit delete với `before` row và `op=d`. Options: **tombstone** (null value) trên compacted topic; **ExtractNewRecordState** với `__deleted=true`; consumer **hard delete** hoặc **soft delete** (set deleted_at). Sink connectors cần `delete.enabled=true` (ES, JDBC).

### Câu 5: Postgres replication slot lag nguy hiểm thế nào?

**Gợi ý trả lời:** Slot giữ WAL position — nếu connector stop lâu, WAL không recycle → **disk full** trên Postgres → DB crash. Monitor `pg_replication_slots`, alert lag, có runbook drop slot + re-snapshot nếu cần.

### Câu 6: Thiết kế CDC pipeline cho 50 tables?

**Gợi ý trả lời:** `table.include.list` chỉ tables cần; **Schema Registry** Avro; **topic per table** hoặc route SMT; **idempotent sinks**; compacted topics cho state; **Kafka Streams** derive domain events nếu cần; monitor Debezium lag và slot; **DLQ** cho bad records; document schema change process.

---

**Xem tiếp:** [2-multi-datacenter.md](./2-multi-datacenter.md) — Geo-replication và active-active patterns.

**Liên quan:** [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md) — Kafka Connect fundamentals.
