# Confluent Cloud — Nền Tảng Kafka Trên Cloud

> Confluent Cloud: managed Apache Kafka kèm Schema Registry (Đăng Ký Schema), ksqlDB (SQL cho Stream), Apache Flink, governance tools, và multi-cloud deployment — so sánh với MSK và self-hosted Confluent Platform.

## Mục Lục

1. [Tổng Quan Confluent Cloud](#tổng-quan-confluent-cloud)
2. [Cluster Types & Sizing](#cluster-types--sizing)
3. [Schema Registry Trên Confluent Cloud](#schema-registry-trên-confluent-cloud)
4. [ksqlDB & Stream Processing](#ksqldb--stream-processing)
5. [Connectors & Flink](#connectors--flink)
6. [Security & Governance](#security--governance)
7. [Multi-Cloud & Networking](#multi-cloud--networking)
8. [Confluent Cloud vs MSK vs Self-Hosted](#confluent-cloud-vs-msk-vs-self-hosted)
9. [Cost Model](#cost-model)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Confluent Cloud

**Confluent Cloud** là dịch vụ fully managed của **Confluent** — công ty founded bởi creators of Kafka. Không chỉ Kafka brokers mà cả **platform ecosystem** xung quanh streaming.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CONFLUENT CLOUD PLATFORM                                 │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Kafka     │  │   Schema    │  │   ksqlDB    │  │   Flink     │          │
│  │   Cluster   │  │   Registry  │  │  (Streams)  │  │ (Processing)│          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                │                 │
│         └────────────────┴────────────────┴────────────────┘                 │
│                                   │                                          │
│                    ┌──────────────▼──────────────┐                            │
│                    │  Confluent Cloud Console  │                            │
│                    │  • Cluster management     │                            │
│                    │  • Connectors (100+)      │                            │
│                    │  • Stream Governance      │                            │
│                    │  • Metrics & alerts       │                            │
│                    └───────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Thành Phần | Mô Tả |
| ---------- | ----- |
| **Kafka Cluster** | Managed brokers — Basic, Standard, Dedicated, Enterprise, Freight |
| **Schema Registry** | Centralized schema management — Avro, Protobuf, JSON Schema |
| **ksqlDB** | SQL interface cho stream processing |
| **Flink** | Advanced stream processing (stateful, windowing) |
| **Connectors** | Pre-built source/sink connectors (managed) |
| **Stream Governance** | Data catalog, lineage, quality rules |

**Deploy trên:** AWS, Azure, GCP — **multi-cloud** là điểm khác biệt lớn so với MSK (AWS-only).

---

## Cluster Types & Sizing

### Cluster Tiers

| Tier | Mô Tả | Use Case |
| ---- | ----- | -------- |
| **Basic** | Shared infrastructure, limited throughput | Dev, POC, learning |
| **Standard** | Dedicated resources, elastic scaling | Production SMB |
| **Dedicated** | Single-tenant, predictable performance | Production enterprise |
| **Enterprise** | Enhanced security, private networking | Regulated industries |
| **Freight** | Multi-zone resilience, highest SLA | Mission-critical |

### Elastic CKU (Confluent Kafka Unit — Đơn Vị Kafka Confluent)

Dedicated clusters scale bằng **CKU** — đơn vị compute + storage bundled:

```
1 CKU ≈ baseline throughput capacity
Scale up: thêm CKU khi cần throughput cao hơn
Auto-scaling: có thể bật elastic scaling theo load
```

### Serverless Clusters

Confluent cũng có **Serverless Kafka** — tương tự MSK Serverless:

```
✅ Không cần chọn CKU/instance
✅ Auto-scale theo demand
⚠️ Cost model per-GB — cần monitor usage
```

---

## Schema Registry Trên Confluent Cloud

**Schema Registry** là thành phần **built-in** — không cần deploy riêng như self-hosted.

### Workflow

```
1. Producer register schema (Avro/Protobuf/JSON Schema)
2. Schema Registry assign schema ID
3. Producer serialize message với schema ID embedded
4. Consumer fetch schema by ID → deserialize
5. Schema evolution theo compatibility mode
```

### Compatibility Modes

| Mode | Rule | Use Case |
| ---- | ---- | -------- |
| **BACKWARD** | New schema đọc được old data | Consumer upgrade trước producer |
| **FORWARD** | Old schema đọc được new data | Producer upgrade trước consumer |
| **FULL** | Cả backward và forward | Strict evolution |
| **NONE** | Không validate | Dev only — không production |

```json
// Avro schema example — OrderCreated event
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.example.orders",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "createdAt", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
```

### Schema Linking (Cross-Cluster)

Replicate schemas giữa clusters/regions — quan trọng cho multi-region và DR:

```
Primary Cluster Schema Registry ──► Schema Linking ──► DR Cluster Schema Registry
```

Chi tiết schema evolution: [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md)

---

## ksqlDB & Stream Processing

### ksqlDB Overview

**ksqlDB** — SQL engine chạy trên Kafka streams. Biến topics thành **streams (luồng)** và **tables (bảng)** queryable.

```
Kafka Topic: orders
       │
       ▼
┌──────────────────┐
│ CREATE STREAM    │
│ orders_stream    │
│ (orderId VARCHAR,│
│  amount DOUBLE)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│ CREATE TABLE     │     │ INSERT INTO      │
│ orders_by_customer│────►│ high_value_orders│
│ (aggregated)     │     │ WHERE amount>1000│
└──────────────────┘     └──────────────────┘
```

### ksqlDB Statements Phổ Biến

```sql
-- Tạo stream từ topic
CREATE STREAM orders (
  order_id VARCHAR,
  customer_id VARCHAR,
  amount DOUBLE,
  order_time BIGINT
) WITH (
  KAFKA_TOPIC = 'orders',
  VALUE_FORMAT = 'AVRO'
);

-- Aggregation — orders per customer per hour
CREATE TABLE orders_per_customer AS
  SELECT customer_id,
         COUNT(*) AS order_count,
         SUM(amount) AS total_amount
  FROM orders
  WINDOW TUMBLING (SIZE 1 HOUR)
  GROUP BY customer_id;

-- Filter và sink sang topic mới
CREATE STREAM high_value_orders AS
  SELECT * FROM orders
  WHERE amount > 1000;
```

### ksqlDB vs Kafka Streams vs Flink

| | ksqlDB | Kafka Streams | Flink (Confluent) |
| - | ------ | ------------- | ----------------- |
| **Interface** | SQL | Java/Scala API | SQL + DataStream API |
| **Learning curve** | Thấp | Trung bình | Cao |
| **Stateful processing** | ✅ | ✅ | ✅ Advanced |
| **Exactly-once** | ✅ | ✅ | ✅ |
| **Use case** | Ad-hoc analytics, filtering | Embedded in app | Complex CEP, large state |

---

## Connectors & Flink

### Managed Connectors

Confluent Cloud cung cấp **100+ pre-built connectors** — deploy qua Console, không cần Connect cluster riêng:

| Category | Examples |
| -------- | -------- |
| **Database CDC** | Debezium MySQL, PostgreSQL, MongoDB |
| **Cloud Storage** | S3, GCS, Azure Blob |
| **Data Warehouses** | Snowflake, BigQuery, Redshift |
| **Search** | Elasticsearch, OpenSearch |
| **SaaS** | Salesforce, HubSpot |

```
Source Connector (Debezium) ──► Kafka Topic ──► Sink Connector (Snowflake)
         Managed by Confluent Cloud — auto-scaling, monitoring included
```

### Apache Flink on Confluent

**Flink** cho use cases phức tạp hơn ksqlDB:

```
• Complex event processing (CEP — Xử Lý Sự Kiện Phức Tạp)
• Large stateful joins
• Event-time windowing với watermarks
• Low-latency analytics at scale
```

---

## Security & Governance

### Authentication

| Method | Mô Tả |
| ------ | ----- |
| **API Keys** | Service account keys — produce/consume |
| **OAuth/OIDC** | Enterprise SSO integration |
| **SASL/SCRAM** | Username/password |
| **mTLS** | Certificate-based |

### RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò)

Confluent RBAC granular hơn basic Kafka ACL:

```
Organization → Environment → Cluster → Topic/Subject/Connector
Roles: OrganizationAdmin, EnvironmentAdmin, CloudClusterAdmin, DeveloperRead, DeveloperWrite
```

### Stream Governance

| Feature | Mô Tả |
| ------- | ----- |
| **Stream Catalog** | Discover topics, schemas, connectors |
| **Data Lineage** | Trace data flow producer → topic → consumer |
| **Schema Rules** | Validation, migration enforcement |
| **Quality Rules** | Data quality checks on streams |

### Encryption

```
In-transit: TLS 1.2+ (bắt buộc)
At-rest: AES-256 encryption on storage
Private networking: PrivateLink (AWS), Private Service Connect (GCP), VNet peering (Azure)
```

---

## Multi-Cloud & Networking

### Deployment Options

```
Confluent Cloud regions:
• AWS: us-east-1, eu-west-1, ap-southeast-1, ...
• Azure: eastus, westeurope, ...
• GCP: us-central1, europe-west1, ...

Chọn region gần application để giảm latency và egress cost
```

### Private Networking

```
┌─────────────────────────────────────────────────────────────┐
│  Your VPC/VNet                                              │
│  ┌─────────────┐         PrivateLink / PSC                  │
│  │ Application │◄────────────────────────────────►          │
│  │ Consumers   │         Confluent Cloud (dedicated)         │
│  └─────────────┘                                             │
└─────────────────────────────────────────────────────────────┘

Không traffic qua public internet — required cho nhiều compliance
```

### Cluster Linking

Replicate data giữa Confluent clusters — thay thế MirrorMaker:

```
Source Cluster ──► Cluster Linking ──► Destination Cluster
(prod)                                  (analytics / DR)
```

---

## Confluent Cloud vs MSK vs Self-Hosted

| Tiêu Chí | Confluent Cloud | Amazon MSK | Self-Hosted |
| -------- | --------------- | ---------- | ----------- |
| **Kafka core** | ✅ | ✅ | ✅ |
| **Schema Registry** | ✅ Built-in | ⚠️ Glue/Confluent add-on | ⚠️ Deploy riêng |
| **ksqlDB** | ✅ | ❌ | ⚠️ License |
| **Flink** | ✅ Managed | ❌ | ⚠️ Self-manage |
| **Connectors** | ✅ 100+ managed | MSK Connect | Self-manage Connect |
| **Multi-cloud** | ✅ | ❌ AWS only | ✅ Anywhere |
| **Support** | Confluent SLA | AWS Support | Community/your team |
| **Cost** | CKU + egress | Broker hours + EBS | Lowest infra, highest ops |

**Khi chọn Confluent Cloud:**

```
✅ Cần full Kafka platform (Schema Registry + ksqlDB + Connectors)
✅ Multi-cloud hoặc có thể chuyển cloud
✅ Team muốn SQL stream processing nhanh
✅ Enterprise governance requirements
✅ Budget cho premium managed service
```

**Khi chọn MSK thay Confluent:**

```
✅ AWS-only, tight AWS integration (IAM, VPC)
✅ Chỉ cần Kafka core — không cần ksqlDB
✅ Cost optimization với AWS reserved capacity
✅ Đã có Glue Schema Registry trong stack
```

---

## Cost Model

### Billing Components

| Component | Mô Tả |
| --------- | ----- |
| **CKU hours** | Dedicated cluster compute |
| **CKU storage** | Data retention storage |
| **Connector tasks** | Per connector task-hour |
| **ksqlDB CSUs** | ksqlDB compute streaming units |
| **Flink CFUs** | Flink compute Flink units |
| **Egress** | Data transfer out — cost driver lớn |
| **Schema Registry** | Included hoặc tier-based |

### Cost Optimization

```
1. Chọn đúng tier — Basic cho dev, Dedicated cho prod
2. Retention tuning — giảm storage cost
3. Compression — lz4/zstd
4. Same-region deployment — tránh cross-region egress
5. Monitor egress — unexpected cost thường từ egress
6. Serverless cho variable workloads
7. Connector task sizing — không over-provision
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Confluent Cloud khác MSK? | Full platform (Schema Registry, ksqlDB, Flink); multi-cloud; Confluent support |
| ksqlDB vs Kafka Streams? | ksqlDB = SQL, server-side; Kafka Streams = embedded Java library |
| Schema Registry tại sao quan trọng? | Contract between producers/consumers; evolution without breaking |
| BACKWARD compatibility nghĩa là gì? | New consumer code đọc old messages — add fields with defaults |
| Cluster Linking vs MirrorMaker? | Managed, integrated với Schema Linking; simpler ops |
| Khi nào Flink thay ksqlDB? | Complex joins, large state, CEP — ksqlDB cho simpler SQL transforms |
| Confluent Cloud lock-in? | Kafka API portable; ksqlDB/Flink/governance less portable |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Schema Registry | [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) |
| Kafka Streams | [03-apache-kafka/8-kafka-streams.md](../03-apache-kafka/8-kafka-streams.md) |
| Kafka Connect | [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md) |
| Amazon MSK | [1-aws-msk.md](./1-aws-msk.md) |
| Cloud overview | [README.md](./README.md) |

---

**Cập Nhật:** 2026-07-03
