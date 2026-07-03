# Cloud Managed Messaging — Dịch Vụ Messaging Quản Lý Trên Cloud

> So sánh và học chuyên sâu các dịch vụ messaging được cloud provider vận hành: Amazon MSK (Managed Streaming for Apache Kafka), Confluent Cloud, Azure Event Hubs, GCP Pub/Sub — khi nào chọn managed vs self-hosted, migration strategy, và cost optimization.

## Mục Lục

1. [Tại Sao Học Cloud Managed Messaging](#tại-sao-học-cloud-managed-messaging)
2. [Managed vs Self-Hosted](#managed-vs-self-hosted)
3. [Ma Trận So Sánh Nhanh](#ma-trận-so-sánh-nhanh)
4. [Kiến Trúc Lựa Chọn Theo Cloud](#kiến-trúc-lựa-chọn-theo-cloud)
5. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
6. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
7. [Migration & Cost Checklist](#migration--cost-checklist)
8. [Bài Tập Thực Hành](#bài-tập-thực-hành)
9. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học Cloud Managed Messaging

Sau khi nắm [Apache Kafka](../03-apache-kafka/README.md) và [RabbitMQ](../04-rabbitmq/README.md) self-hosted, bạn cần hiểu **lựa chọn managed trên cloud** vì:

- Hầu hết doanh nghiệp triển khai messaging trên AWS, Azure, hoặc GCP — không tự vận hành broker
- Phỏng vấn system design thường hỏi: "Tại sao MSK thay vì self-hosted Kafka?"
- Migration từ on-premise hoặc Docker local lên production cloud là bước bắt buộc
- Cost model managed khác hoàn toàn self-hosted — cần tính đúng TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu)

| Dịch Vụ | Cloud | Engine Cốt Lõi | Điểm Mạnh Chính |
| ------- | ----- | -------------- | --------------- |
| **Amazon MSK** | AWS | Apache Kafka | Kafka native trên AWS, MSK Connect, Serverless |
| **Confluent Cloud** | Multi-cloud | Apache Kafka + platform | Schema Registry, ksqlDB, Flink, enterprise support |
| **Azure Event Hubs** | Azure | Kafka-compatible / native | Capture to Blob, Azure ecosystem, tier linh hoạt |
| **GCP Pub/Sub** | GCP | Native Pub/Sub | Serverless, global, push/pull, ordering keys |

> **Điều kiện tiên quyết:** Đã học [03-apache-kafka/](../03-apache-kafka/README.md), [05-other-brokers/1-amazon-sqs-sns.md](../05-other-brokers/1-amazon-sqs-sns.md), [01-fundamentals/6-broker-selection-guide.md](../01-fundamentals/6-broker-selection-guide.md).

---

## Managed vs Self-Hosted

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              MANAGED vs SELF-HOSTED (Quản Lý vs Tự Vận Hành)                 │
│                                                                             │
│  Self-Hosted Kafka                    Managed (MSK / Confluent Cloud)         │
│  ┌─────────────────────┐            ┌─────────────────────┐                 │
│  │ Bạn quản lý:        │            │ Provider quản lý:   │                 │
│  │ • Broker patching   │            │ • Broker patching   │                 │
│  │ • OS, JVM tuning    │            │ • OS, JVM tuning    │                 │
│  │ • Disk, network     │            │ • Disk, network     │                 │
│  │ • KRaft/ZooKeeper   │            │ • KRaft/ZooKeeper   │                 │
│  │ • Monitoring stack  │            │ • Built-in metrics  │                 │
│  │ • HA, failover      │            │ • HA, failover      │                 │
│  └─────────────────────┘            └─────────────────────┘                 │
│                                                                             │
│  Bạn vẫn quản lý (cả hai): topic design, consumer groups, schema, ACL,     │
│  application code, idempotency, DLQ strategy, SLO/alerting                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Tiêu Chí | Self-Hosted | Managed |
| -------- | ----------- | ------- |
| **Ops burden (Gánh vận hành)** | Cao — team cần Kafka expertise | Thấp — provider handle infra |
| **Time to production** | Tuần–tháng (cluster setup, tuning) | Giờ–ngày |
| **Cost model** | EC2/VM + disk + ops headcount | Per-hour broker + storage + throughput |
| **Customization** | Full control (JVM, configs) | Giới hạn theo platform |
| **Compliance** | Bạn chứng minh hardening | Provider có SOC2, ISO certs |
| **Vendor lock-in** | Thấp (open source Kafka) | Trung bình–cao (tùy dịch vụ) |

**Khi nào chọn managed:**

```
✅ Team nhỏ, không có dedicated Kafka SRE
✅ Cần go-live nhanh trên cloud đã chọn
✅ Muốn SLA và support từ vendor
✅ Throughput ổn định, không cần tuning cực đoan

Khi nào cân nhắc self-hosted:
⚠️ Cost managed quá cao ở scale lớn (hàng trăm TB/ngày)
⚠️ Cần config broker không supported trên managed
⚠️ On-premise requirement (data residency nghiêm ngặt)
⚠️ Đã có team SRE mạnh và tooling sẵn
```

---

## Ma Trận So Sánh Nhanh

| Tiêu Chí | MSK | Confluent Cloud | Event Hubs | GCP Pub/Sub |
| -------- | --- | --------------- | ---------- | ----------- |
| **Kafka API** | ✅ Native | ✅ Native | ✅ Compatible (Kafka endpoint) | ❌ (native API) |
| **Protocol chính** | Kafka | Kafka | Kafka + AMQP (Premium) | gRPC/REST |
| **Serverless option** | ✅ MSK Serverless | ✅ Serverless clusters | ✅ (Consumption tier) | ✅ Native |
| **Schema Registry** | ⚠️ (MSK + Glue/Confluent) | ✅ Built-in | ⚠️ (Schema Registry riêng) | ✅ Schema validation |
| **Stream processing** | MSK Connect, Lambda | ksqlDB, Flink | Stream Analytics, Spark | Dataflow |
| **Replay** | ✅ (Kafka log) | ✅ | ✅ (retention-based) | ✅ (retention 7d–31d) |
| **Multi-cloud** | ❌ AWS only | ✅ AWS, Azure, GCP | ❌ Azure only | ❌ GCP only |
| **Pricing driver** | Broker hours + storage | CKU + storage + egress | TU (Throughput Units) + capture | Message volume + egress |

**Phân loại theo use case:**

| Use Case | Gợi Ý |
| -------- | ----- |
| Event streaming + replay trên AWS | MSK hoặc Confluent Cloud on AWS |
| Kafka ecosystem đầy đủ (ksqlDB, Connect) | Confluent Cloud |
| Azure-native, analytics pipeline | Event Hubs + Capture |
| Serverless fan-out, không cần Kafka API | GCP Pub/Sub hoặc SQS/SNS |
| Hybrid: task queue + event bus | SQS + MSK (xem [05-other-brokers](../05-other-brokers/README.md)) |

---

## Kiến Trúc Lựa Chọn Theo Cloud

### AWS Ecosystem

```
┌──────────────┐     ┌─────────────┐     ┌──────────────────┐
│ Application  │────►│ Amazon MSK  │────►│ MSK Connect      │
│ (ECS/EKS)    │     │ (Kafka)     │     │ → S3, RDS, ES    │
└──────────────┘     └──────┬──────┘     └──────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Lambda   │  │ SQS/SNS  │  │ Glue     │
        │ (events) │  │ (tasks)  │  │ Schema   │
        └──────────┘  └──────────┘  └──────────┘
```

### Azure Ecosystem

```
Producers ──► Event Hubs ──┬──► Stream Analytics / Spark
                           ├──► Capture ──► Blob Storage (data lake)
                           └──► Kafka consumers (compatible endpoint)
```

### GCP Ecosystem

```
Publishers ──► Pub/Sub Topic ──┬──► Push subscription ──► Cloud Run / GKE
                               ├──► Pull subscription ──► Dataflow
                               └──► BigQuery subscription (analytics)
```

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-aws-msk.md](./1-aws-msk.md) | MSK Provisioned, Serverless, MSK Connect | 1.5 giờ |
| 2 | [2-confluent-cloud.md](./2-confluent-cloud.md) | Confluent Cloud, Schema Registry, ksqlDB | 1.5 giờ |
| 3 | [3-azure-event-hubs.md](./3-azure-event-hubs.md) | Event Hubs, Kafka endpoint, Capture | 1.5 giờ |
| 4 | [4-gcp-pubsub.md](./4-gcp-pubsub.md) | Pub/Sub push/pull, ordering keys | 1.5 giờ |

**Thứ tự đề xuất:**

```
1. Đọc README này → nắm ma trận so sánh
2. Học dịch vụ trên cloud bạn đang dùng (hoặc mục tiêu phỏng vấn)
3. Đọc thêm 1 dịch vụ cross-cloud để so sánh phỏng vấn
4. Làm lab migration checklist
```

---

## Các Tài Liệu Chi Tiết

| File | Mô Tả | Độ Ưu Tiên |
| ---- | ----- | ---------- |
| [1-aws-msk.md](./1-aws-msk.md) | Amazon MSK — cluster types, security, MSK Connect, cost | ⭐⭐ AWS shops |
| [2-confluent-cloud.md](./2-confluent-cloud.md) | Confluent platform trên cloud — ksqlDB, Flink, governance | ⭐⭐ Kafka-heavy |
| [3-azure-event-hubs.md](./3-azure-event-hubs.md) | Event Hubs tiers, Capture, Kafka compatibility | ⭐⭐ Azure shops |
| [4-gcp-pubsub.md](./4-gcp-pubsub.md) | Pub/Sub model, push vs pull, ordering, BigQuery sink | ⭐⭐ GCP shops |

---

## Migration & Cost Checklist

### Pre-Migration Checklist

```
□ Inventory topics, partitions, retention policies
□ Document consumer groups, offset commit strategy
□ Map ACL/IAM permissions
□ Estimate throughput (MB/s) và message rate
□ Identify MSK Connect / MirrorMaker needs cho dual-write
□ Plan schema migration (Schema Registry compatibility)
□ Define rollback plan (keep old cluster running parallel)
```

### Cost Optimization Tips

| Chiến Lược | Áp Dụng |
| ---------- | ------- |
| **Right-size partitions** | Không over-partition — mỗi partition = overhead |
| **Retention tuning** | Giảm retention nếu không cần replay dài |
| **Tier selection** | Event Hubs Basic vs Standard; MSK Serverless cho variable load |
| **Compression** | lz4/snappy giảm storage và egress cost |
| **Serverless vs Provisioned** | Variable traffic → serverless; steady high → provisioned |
| **Cross-AZ egress** | Thiết kế producer/consumer cùng region/AZ khi có thể |

---

## Bài Tập Thực Hành

### Lab 1: So Sánh Pricing (60 phút)

```
Kịch bản: 50 MB/s ingest, 7 ngày retention, 3 AZ HA.

Tính rough cost cho:
1. MSK Provisioned (kafka.m5.large x 3 brokers)
2. MSK Serverless
3. Confluent Cloud (Basic cluster)
4. Event Hubs Standard (TU calculation)

Deliverable: Bảng so sánh + recommendation cho startup vs enterprise.
```

### Lab 2: MSK Connect Pipeline (90 phút)

```
Mục tiêu: Source từ MSK topic → S3 sink (JSON, partitioned by date).

Bước:
1. Tạo MSK cluster (hoặc dùng existing)
2. Deploy MSK Connect connector (S3 Sink)
3. Produce test events
4. Verify files trong S3 prefix structure
5. Monitor connector lag qua CloudWatch
```

### Lab 3: Migration Planning Document (45 phút)

```
Viết migration plan 1 trang:
- Source: self-hosted Kafka 3 brokers
- Target: MSK hoặc Confluent Cloud
- Phases: dual-write → consumer cutover → producer cutover → decommission
- Risk matrix và rollback triggers
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| MSK khác self-hosted Kafka thế nào? | AWS quản lý broker infra; bạn vẫn quản lý topics, ACL, apps; giới hạn config |
| Khi nào MSK Serverless vs Provisioned? | Serverless: variable/unpredictable load; Provisioned: steady throughput, cần tuning |
| Confluent Cloud value-add là gì? | Schema Registry, ksqlDB, Flink, governance, support — trên Kafka core |
| Event Hubs có phải Kafka không? | Kafka-compatible endpoint; không phải Kafka 100%; một số feature khác biệt |
| GCP Pub/Sub vs Kafka? | Pub/Sub: serverless, no partition management; Kafka: log, replay control, ecosystem |
| Migrate Kafka lên cloud không downtime? | MirrorMaker 2 / MSK Replicator dual-write, consumer lag = 0 rồi cutover |
| Cost driver lớn nhất managed Kafka? | Broker instance hours + storage (EBS) + cross-AZ data transfer |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| SQS/SNS (AWS task queue) | [05-other-brokers/1-amazon-sqs-sns.md](../05-other-brokers/1-amazon-sqs-sns.md) |
| Kafka architecture | [03-apache-kafka/1-kafka-architecture.md](../03-apache-kafka/1-kafka-architecture.md) |
| Schema Registry | [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) |
| Security (IAM, TLS) | [08-security/README.md](../08-security/README.md) |
| Production checklist | [09-monitoring/7-production-checklist.md](../09-monitoring/7-production-checklist.md) |
| Broker selection | [01-fundamentals/6-broker-selection-guide.md](../01-fundamentals/6-broker-selection-guide.md) |

---

**Cập Nhật:** 2026-07-03
