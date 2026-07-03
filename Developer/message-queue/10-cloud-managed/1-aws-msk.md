# Amazon MSK — Managed Streaming for Apache Kafka Trên AWS

> Amazon MSK (Managed Streaming for Apache Kafka — Kafka Quản Lý Trên AWS): MSK Provisioned, MSK Serverless, MSK Connect, security với IAM (Identity and Access Management — Quản Lý Danh Tính và Truy Cập), và chiến lược migration từ self-hosted Kafka lên AWS.

## Mục Lục

1. [Tổng Quan Amazon MSK](#tổng-quan-amazon-msk)
2. [MSK Provisioned vs Serverless](#msk-provisioned-vs-serverless)
3. [Cluster Sizing & Configuration](#cluster-sizing--configuration)
4. [Security: IAM, TLS, Private Connectivity](#security-iam-tls-private-connectivity)
5. [MSK Connect](#msk-connect)
6. [Monitoring & Operations](#monitoring--operations)
7. [Migration Strategy](#migration-strategy)
8. [Cost Model & Optimization](#cost-model--optimization)
9. [So Sánh Với Alternatives](#so-sánh-với-alternatives)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Amazon MSK

**Amazon MSK** là dịch vụ fully managed (quản lý hoàn toàn) chạy **Apache Kafka** trên AWS. AWS vận hành broker infrastructure; bạn quản lý topics, consumer groups, và application code.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AMAZON MSK ARCHITECTURE                               │
│                                                                             │
│  ┌──────────────┐     ┌─────────────────────────────────────────────┐      │
│  │  Producers   │────►│  MSK Cluster (3+ brokers across AZs)        │      │
│  │  Consumers   │◄────│  • Apache Kafka (version managed)           │      │
│  │  (ECS/EKS/   │     │  • EBS storage per broker                   │      │
│  │   EC2/Lambda)│     │  • KRaft mode (ZooKeeper deprecated)        │      │
│  └──────────────┘     └──────────────┬──────────────────────────────┘      │
│                                      │                                      │
│              ┌───────────────────────┼───────────────────────┐              │
│              ▼                       ▼                       ▼              │
│       ┌─────────────┐        ┌─────────────┐        ┌─────────────┐          │
│       │ MSK Connect │        │ CloudWatch  │        │ AWS Glue    │          │
│       │ (Connectors)│        │ Metrics     │        │ Schema Reg  │          │
│       └─────────────┘        └─────────────┘        └─────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Thành Phần | Mô Tả |
| ---------- | ----- |
| **MSK Cluster** | Kafka brokers trên EC2 (Provisioned) hoặc serverless compute |
| **Bootstrap servers** | Endpoint để producer/consumer kết nối |
| **Storage** | EBS volumes (Provisioned) hoặc managed storage (Serverless) |
| **MSK Connect** | Managed Kafka Connect workers — source/sink connectors |
| **MSK Replicator** | Cross-cluster replication (MSK ↔ MSK, MSK ↔ self-hosted) |

**Điểm quan trọng:** MSK chạy **open-source Apache Kafka** — client libraries, admin tools, và hầu hết Kafka concepts ([topics, partitions](../03-apache-kafka/2-topics-partitions.md), [consumer groups](../03-apache-kafka/3-consumer-groups.md)) áp dụng trực tiếp.

---

## MSK Provisioned vs Serverless

### MSK Provisioned

Broker chạy trên EC2 instances bạn chọn — full control throughput và sizing.

```
✅ Predictable performance — chọn instance type (kafka.m5.large, etc.)
✅ Custom broker configs (một số parameters)
✅ Dedicated resources — không noisy neighbor
✅ Phù hợp steady-state high throughput

❌ Phải right-size brokers
❌ Trả tiền 24/7 dù idle
❌ Scaling cần plan (add brokers, rebalance)
```

**Broker sizing phổ biến:**

| Instance Type | vCPU | Memory | Use Case |
| ------------- | ---- | ------ | -------- |
| kafka.t3.small | 2 | 2 GiB | Dev/test |
| kafka.m5.large | 2 | 8 GiB | Production nhỏ |
| kafka.m5.xlarge | 4 | 16 GiB | Production trung bình |
| kafka.m5.2xlarge | 8 | 32 GiB | High throughput |

### MSK Serverless

AWS tự động scale capacity theo workload — trả theo usage.

```
✅ Auto-scale — không cần chọn instance type
✅ Pay per data in/out + storage
✅ Nhanh setup — phút thay vì giờ
✅ Phù hợp variable/unpredictable traffic

❌ Giới hạn max throughput per cluster
❌ Ít broker config customization hơn
❌ Cost có thể cao hơn ở sustained high throughput
❌ Không phù hợp mọi Kafka feature (kiểm tra compatibility)
```

### Decision Tree

```
Throughput ổn định, cao, cần tuning?
  ├── YES → MSK Provisioned
  └── NO → Traffic biến động mạnh?
              ├── YES → MSK Serverless
              └── NO → Dev/test → Serverless hoặc Provisioned nhỏ
```

---

## Cluster Sizing & Configuration

### Số Brokers & AZs

```
Production minimum:
• 3 brokers across 3 AZs (Availability Zones — Vùng Sẵn Sàng)
• Replication factor = 3 (min.insync.replicas = 2)
• Partition count: plan trước — khó giảm sau
```

### Storage Planning

| Factor | Công Thức / Gợi Ý |
| ------ | ----------------- |
| **Daily ingest** | MB/s × 86400 = MB/day |
| **Retention** | daily × retention_days × replication_factor |
| **Headroom** | +20–30% cho compaction, spikes |
| **EBS type** | gp3 (default) — IOPS và throughput configurable |

```yaml
# Ví dụ: topic config trên MSK
# retention.ms = 604800000  (7 ngày)
# compression.type = lz4
# min.insync.replicas = 2
```

### Broker Properties

MSK cho phép custom **server.properties** qua AWS Console hoặc CLI — một số properties bị lock bởi AWS:

```
Configurable: log.retention.hours, compression.type, num.partitions (default)
Restricted: listeners, security configs (qua MSK API), một số internal settings
```

---

## Security: IAM, TLS, Private Connectivity

### Authentication Options

| Method | Mô Tả | Use Case |
| ------ | ----- | -------- |
| **TLS (mTLS — Mutual TLS)** | Client certificate authentication | Legacy, fine-grained cert per client |
| **SASL/SCRAM** | Username/password | App-level auth |
| **IAM Access Control** | AWS IAM policies | AWS-native apps (recommended) |

### IAM Access Control

MSK hỗ trợ **IAM-based authentication** — không cần manage Kafka users riêng cho AWS services.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:WriteData",
        "kafka-cluster:ReadData"
      ],
      "Resource": [
        "arn:aws:kafka:ap-southeast-1:123456789012:cluster/my-cluster/*",
        "arn:aws:kafka:ap-southeast-1:123456789012:topic/my-cluster/*"
      ]
    }
  ]
}
```

**IAM actions chính:**

| Action | Permission |
| ------ | ---------- |
| `kafka-cluster:Connect` | Kết nối broker |
| `kafka-cluster:WriteData` | Produce messages |
| `kafka-cluster:ReadData` | Consume messages |
| `kafka-cluster:CreateTopic` | Admin — tạo topic |
| `kafka-cluster:AlterGroup` | Consumer group management |

### Network Security

```
┌─────────────────────────────────────────────────────────────┐
│  VPC (Virtual Private Cloud — Mạng Riêng Ảo)                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Private Subnets (3 AZs)                            │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐               │    │
│  │  │ Broker  │  │ Broker  │  │ Broker  │               │    │
│  │  │  AZ-a   │  │  AZ-b   │  │  AZ-c   │               │    │
│  │  └─────────┘  └─────────┘  └─────────┘               │    │
│  └─────────────────────────────────────────────────────┘    │
│         ▲                                                    │
│         │ PrivateLink / VPC Peering / Same VPC               │
│  ┌──────┴──────┐                                             │
│  │ EKS / ECS   │                                             │
│  │ Consumers   │                                             │
│  └─────────────┘                                             │
└─────────────────────────────────────────────────────────────┘

❌ Không expose MSK publicly — luôn trong VPC
✅ Security groups: chỉ allow port 9092/9094/9096 từ app subnets
```

### Encryption

| Layer | Option |
| ----- | ------ |
| **In-transit (Truyền tải)** | TLS between clients and brokers (bắt buộc production) |
| **At-rest (Lưu trữ)** | AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) encryption on EBS |

---

## MSK Connect

**MSK Connect** là managed **Kafka Connect (Kết Nối Kafka)** — chạy connectors không cần self-manage Connect cluster.

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ Source       │────►│ MSK Topic   │────►│ Sink         │
│ (Debezium,   │     │             │     │ (S3, ES,     │
│  JDBC, etc.) │     │             │     │  OpenSearch) │
└──────────────┘     └─────────────┘     └──────────────┘
       ▲                                         │
       └────────── MSK Connect Workers ──────────┘
```

### Connector Types Phổ Biến

| Connector | Direction | Use Case |
| --------- | --------- | -------- |
| **Debezium CDC** | Source → Kafka | Database change capture |
| **S3 Sink** | Kafka → S3 | Data lake ingestion |
| **OpenSearch Sink** | Kafka → OpenSearch | Search indexing |
| **JDBC Source** | DB → Kafka | Batch/table sync |

### MSK Connect Configuration

```json
{
  "connectorConfiguration": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "topics": "orders",
    "s3.bucket.name": "my-data-lake",
    "s3.region": "ap-southeast-1",
    "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "partition.duration.ms": "86400000",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd",
    "locale": "en-US",
    "timezone": "UTC",
    "flush.size": "1000"
  },
  "capacity": {
    "mcuCount": 1,
    "workerCount": 1
  }
}
```

**MCU (MSK Connect Unit — Đơn Vị MSK Connect):** Đơn vị compute cho Connect workers — scale theo throughput connector.

### MSK Replicator

Replication giữa clusters — dùng cho migration và DR (Disaster Recovery — Phục Hồi Thảm Họa):

```
Source Cluster ──► MSK Replicator ──► Target Cluster
(self-hosted         (managed          (MSK production)
 or MSK dev)          MirrorMaker 2)
```

---

## Monitoring & Operations

### CloudWatch Metrics

| Metric | Ý Nghĩa | Alert Threshold |
| ------ | ------- | --------------- |
| **BytesInPerSec** | Ingest rate | Baseline + anomaly |
| **BytesOutPerSec** | Consumer read rate | |
| **CpuUser** | Broker CPU | > 70% sustained |
| **KafkaDataLogsDiskUsed** | Disk usage | > 80% |
| **UnderReplicatedPartitions** | Replication lag | > 0 |
| **OfflinePartitionsCount** | Unavailable partitions | > 0 critical |

### Consumer Lag trên MSK

MSK không có built-in consumer lag metric — options:

```
1. CloudWatch custom metrics từ app (expose lag)
2. Prometheus + kafka_exporter trong VPC
3. Open-source Burrow (deploy trong VPC)
4. Confluent Control Center (nếu dùng Confluent)
```

### Patching & Maintenance

```
AWS quản lý:
• Kafka version upgrades (maintenance window)
• OS patching on broker EC2
• Broker replacement khi hardware fail

Bạn quản lý:
• Topic/partition changes
• Client compatibility testing trước upgrade
• Maintenance window scheduling
```

---

## Migration Strategy

### Phased Migration (Zero-Downtime Target)

```
Phase 1: Setup MSK cluster + networking + IAM
Phase 2: MSK Replicator / MirrorMaker 2 — dual-write từ source
Phase 3: Migrate consumers → MSK (verify lag = 0)
Phase 4: Migrate producers → MSK
Phase 5: Decommission source cluster
```

### Dual-Write Pattern

```
                    ┌──────────────┐
Producer ──────────►│ Old Kafka    │──► Existing consumers
        │           └──────────────┘
        │
        └──────────►│ MSK Cluster  │──► New consumers (testing)
                    └──────────────┘

Cutover khi: new consumers stable, lag replicated = 0, rollback plan tested
```

### Compatibility Checklist

```
□ Kafka client version compatible với MSK Kafka version
□ ACL/IAM mapping hoàn tất
□ Schema Registry migration (Glue hoặc Confluent)
□ Topic configs (retention, compression) replicated
□ Consumer group offsets migrated (MirrorMaker preserves)
□ Integration tests passed on MSK
```

---

## Cost Model & Optimization

### Cost Components (Provisioned)

| Component | Billing |
| --------- | ------- |
| **Broker hours** | EC2 instance type × số brokers × giờ |
| **Storage** | EBS GB-month per broker |
| **Data transfer** | Cross-AZ, cross-region egress |
| **MSK Connect** | MCU-hours |

### Cost Components (Serverless)

| Component | Billing |
| --------- | ------- |
| **Cluster hours** | Per cluster active time |
| **Partition hours** | Per partition-hour |
| **Data in/out** | Per GB processed |
| **Storage** | Per GB-month |

### Optimization Tips

```
1. Compression (lz4) — giảm storage và transfer
2. Retention tuning — không giữ data không cần replay
3. Right-size partitions — tránh over-partitioning
4. Same-AZ consumers khi latency cho phép — giảm cross-AZ cost
5. MSK Serverless cho dev/staging — tắt khi không dùng
6. Reserved capacity (Provisioned) — discount cho 1-year commit
```

---

## So Sánh Với Alternatives

| | MSK Provisioned | MSK Serverless | Confluent Cloud | Self-Hosted EC2 |
| - | --------------- | -------------- | --------------- | --------------- |
| **Ops** | Low | Very low | Very low | High |
| **Kafka API** | ✅ | ✅ | ✅ | ✅ |
| **ksqlDB** | ❌ (cần Confluent) | ❌ | ✅ | ⚠️ |
| **Cost at scale** | Medium | Variable | Medium–High | Lowest (nếu có SRE) |
| **AWS integration** | Native | Native | Good | Manual |

**Khi dùng SQS thay MSK:** Task queue đơn giản, Lambda trigger, không cần replay/event log — xem [05-other-brokers/1-amazon-sqs-sns.md](../05-other-brokers/1-amazon-sqs-sns.md).

**Hybrid pattern phổ biến:**

```
MSK (event bus, replay, analytics) + SQS (task queue, Lambda) + SNS (fan-out notifications)
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| MSK khác Kafka trên EC2 tự cài? | AWS patch, monitor, replace brokers; bạn không SSH vào broker |
| MSK Serverless limitations? | Max throughput caps, ít config hơn, cost model khác |
| IAM auth hoạt động thế nào? | SigV4 signing; IAM policy grant topic/cluster actions |
| MSK Connect vs self-hosted Connect? | Managed workers (MCU), auto-scaling, AWS integration |
| Migrate Kafka không downtime? | Replicator dual-write, consumer cutover, offset sync |
| UnderReplicatedPartitions > 0 nghĩa là gì? | Broker down hoặc lag replication — risk data loss nếu thêm broker fail |
| Khi nào MSK không phù hợp? | Cần ksqlDB native, extreme cost optimization, config không supported |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Kafka architecture | [03-apache-kafka/1-kafka-architecture.md](../03-apache-kafka/1-kafka-architecture.md) |
| Kafka Connect | [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md) |
| SQS/SNS | [05-other-brokers/1-amazon-sqs-sns.md](../05-other-brokers/1-amazon-sqs-sns.md) |
| Confluent Cloud | [2-confluent-cloud.md](./2-confluent-cloud.md) |
| Security | [08-security/1-authentication.md](../08-security/1-authentication.md) |

---

**Cập Nhật:** 2026-07-03
