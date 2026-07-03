# Multi-Datacenter Replication — Nhân Bản Đa Trung Tâm Dữ Liệu

> Multi-datacenter và Geo-replication (Nhân Bản Địa Lý) cho message brokers: active-passive vs active-active, MirrorMaker 2, Confluent Cluster Linking, conflict resolution, và thiết kế DR (Disaster Recovery — Phục Hồi Thảm Họa) cho Kafka và các broker khác.

## Mục Lục

1. [Tại Sao Cần Multi-DC Messaging](#tại-sao-cần-multi-dc-messaging)
2. [Replication Patterns](#replication-patterns)
3. [Kafka Multi-DC Options](#kafka-multi-dc-options)
4. [MirrorMaker 2 Deep Dive](#mirrormaker-2-deep-dive)
5. [Confluent Cluster Linking](#confluent-cluster-linking)
6. [Active-Active & Conflict Resolution](#active-active--conflict-resolution)
7. [Other Brokers Geo-Replication](#other-brokers-geo-replication)
8. [Disaster Recovery Planning](#disaster-recovery-planning)
9. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Multi-DC Messaging

Hệ thống messaging single-region có **single point of failure (điểm lỗi đơn)** và latency cao cho users xa region.

| Driver | Mô Tả | Ví Dụ |
| ------ | ----- | ----- |
| **Disaster Recovery (DR)** | Region outage không dừng business | AWS us-east-1 incident |
| **Low Latency** | Users global — produce/consume gần region | SaaS EU + US + APAC |
| **Data Residency** | Compliance yêu cầu data ở EU | GDPR |
| **Read Locality** | Analytics region-local | EU team query EU data |

```
Single Region Risk:
  us-east-1 outage → toàn bộ event pipeline down
  → Orders không process, notifications stop, analytics gap

Multi-Region:
  us-east-1 outage → failover us-west-2
  → RTO (Recovery Time Objective — Thời Gian Phục Hồi Mục Tiêu) phút thay vì giờ
```

---

## Replication Patterns

### Active-Passive (Primary-Secondary)

```
┌─────────────────┐         replicate         ┌─────────────────┐
│  US-EAST        │ ────────────────────────► │  US-WEST        │
│  (PRIMARY)      │         (async)           │  (STANDBY)      │
│                 │                           │                 │
│  Producers ✓    │                           │  Producers ✗    │
│  Consumers ✓    │                           │  Consumers ✗    │
│                 │                           │  (standby only) │
└─────────────────┘                           └─────────────────┘

Failover: Promote US-WEST → redirect DNS/producers → consumers start
```

| Ưu Điểm | Nhược Điểm |
| ------- | ---------- |
| Đơn giản, ít conflict | Standby region idle (cost) |
| Single writer — ordering rõ | Failover manual/semi-auto |
| DR proven pattern | RPO > 0 (async lag) |

### Active-Active

```
┌─────────────────┐         bi-directional      ┌─────────────────┐
│  US-EAST        │ ◄────────────────────────► │  EU-WEST        │
│                 │         replicate           │                 │
│  Producers ✓    │                           │  Producers ✓    │
│  Consumers ✓    │                           │  Consumers ✓    │
└─────────────────┘                           └─────────────────┘

Users US → US cluster; Users EU → EU cluster
Cross-region sync cho global view
```

| Ưu Điểm | Nhược Điểm |
| ------- | ---------- |
| Full utilization cả 2 regions | **Conflict resolution** phức tạp |
| Low latency local | Split-brain risk nếu partition |
| No idle standby cost | Duplicate processing risk |

### Hub-and-Spoke

```
                    ┌─────────────┐
                    │  CENTRAL    │
                    │  (Hub)      │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ Edge US    │  │ Edge EU    │  │ Edge APAC  │
    │ (Spoke)    │  │ (Spoke)    │  │ (Spoke)    │
    └────────────┘  └────────────┘  └────────────┘
```

Phù hợp retail, IoT edge — events aggregate về central, commands distribute ra edge.

---

## Kafka Multi-DC Options

| Tool | Loại | Bi-directional | Managed |
| ---- | ---- | -------------- | ------- |
| **MirrorMaker 2 (MM2)** | Open source | ✅ | Self-hosted |
| **Confluent Cluster Linking** | Confluent platform | ✅ | Confluent Cloud / Self |
| **Confluent Replicator** | Legacy | ⚠️ | Deprecated → Cluster Linking |
|  | **MSK Replicator** | AWS managed | ✅ One-way | AWS MSK |
| **UReplicator** | Uber open source | ✅ | Self-hosted |

### Architecture Overview

```
Source Cluster (US)                    Target Cluster (EU)
┌─────────────────────┐               ┌─────────────────────┐
│ Topic: orders       │               │ Topic: orders       │
│ (12 partitions)     │               │ (12 partitions)     │
└──────────┬──────────┘               └──────────▲──────────┘
           │                                       │
           └──────────► MirrorMaker 2 ──────────────┘
                        (source connector +
                         checkpoint connector +
                         heartbeat connector)
```

**Quan trọng:** Replicated topic là **bản copy độc lập** — offsets, consumer groups **không share** giữa clusters.

---

## MirrorMaker 2 Deep Dive

**MirrorMaker 2 (MM2 — MirrorMaker Phiên Bản 2)** là Kafka tool replicate topics giữa clusters, built trên Kafka Connect framework.

### Components

| Connector | Vai Trò |
| --------- | ------- |
| **MirrorSourceConnector** | Copy records từ source → target topics |
| **MirrorCheckpointConnector** | Sync consumer group offsets source → target |
| **MirrorHeartbeatConnector** | Monitor replication lag (heartbeat topics) |

### Configuration Example

```properties
# mm2.properties
clusters = us, eu
us.bootstrap.servers = kafka-us:9092
eu.bootstrap.servers = kafka-eu:9092

us->eu.enabled = true
us->eu.topics = orders.*, payments.*
us->eu.replication.policy.class = org.apache.kafka.connect.mirror.DefaultReplicationPolicy

# Rename: orders → us.orders (tránh conflict bi-directional)
us->eu.replication.policy.separator = .

us->eu.sync.topic.configs.enabled = true
us->eu.sync.topic.acls.enabled = false

tasks.max = 4
replication.factor = 3
```

### Topic Naming với Replication Policy

```
Source topic:     orders
Target topic:     us.orders     (prefix = source cluster alias)

Bi-directional:
  US produces → us.orders on EU cluster
  EU produces → eu.orders on US cluster
  → Tránh loop: MM2 không replicate topics đã có prefix
```

### Offset Translation

```
Consumer ở US đọc orders offset 5000
Failover EU: Checkpoint connector map → eu.orders offset tương đương
→ Consumer EU resume gần đúng vị trí (không exact 1:1)
```

**Giới hạn:** Offset mapping là **approximate** — active-passive failover OK; active-active cần design cẩn thận.

### Monitoring MM2

| Metric | Ý Nghĩa |
| ------ | ------- |
| `record-rate` | Replication throughput |
| `byte-rate` | Bandwidth usage |
| `replication-latency-ms` | End-to-end lag |
| `checkpoint-latency-ms` | Offset sync delay |

Topics nội bộ: `us.checkpoints.internal`, `us.heartbeats`

---

## Confluent Cluster Linking

**Cluster Linking** là giải pháp Confluent cho bi-directional replication với **offset preservation (bảo toàn offset)** tốt hơn MM2.

```
Source Cluster ──► Cluster Link ──► Destination Cluster
                   (managed link)

Features:
  ✅ Bi-directional links
  ✅ Offset preservation (exact consumer failover)
  ✅ Topic auto-creation trên destination
  ✅ Schema Linking (Schema Registry sync)
  ✅ Confluent Cloud native
```

### So Sánh MM2 vs Cluster Linking

| Tiêu Chí | MirrorMaker 2 | Cluster Linking |
| -------- | ------------- | --------------- |
| **License** | Open source | Confluent (commercial) |
| **Offset mapping** | Approximate (checkpoints) | Exact preservation |
| **Bi-directional** | Manual config | Native link pairs |
| **Schema sync** | Manual | Schema Linking |
| **Ops complexity** | Cao (3 connectors) | Thấp hơn (managed) |
| **Cloud** | Self-hosted | Confluent Cloud |

---

## Active-Active & Conflict Resolution

Active-active là pattern khó nhất — **cùng entity có thể được modify ở 2 regions**.

### Vấn Đề Cơ Bản

```
T0: Order #123 status=PENDING (cả US và EU đều có)
T1: US user cancel  → status=CANCELLED (US cluster)
T2: EU warehouse ship → status=SHIPPED (EU cluster)
T3: Replicate cả 2 → CONFLICT: CANCELLED vs SHIPPED?
```

### Chiến Lược Giải Quyết

| Strategy | Mô Tả | Trade-off |
| -------- | ----- | --------- |
| **Avoid conflicts (design)** | Mỗi entity owned by 1 region (shard by user region) | Best — prevent thay vì resolve |
| **Last-Write-Wins (LWW)** | Timestamp cao nhất thắng | Đơn giản nhưng mất updates |
| **Version vectors** | Track version per region, detect conflict | Complex, explicit merge |
| **CRDT (Conflict-free Replicated Data Type — Kiểu Dữ Liệu Nhân Bản Không Xung Đột)** | Data structure merge tự động | Limited data types |
| **Saga compensation** | Detect conflict → trigger compensating action | Business logic heavy |

### Recommended: Entity Ownership (Sharding)

```
Rule: Order owned by region where customer registered
  customer.region = "EU" → all writes for that order ONLY in EU cluster
  US cluster has read replica (via replication) — read-only

→ No write conflicts by design
→ Active-active for READS, single-writer per entity for WRITES
```

### Kafka-Specific Active-Active Rules

```
1. Partition key = entity ID → per-entity ordering trong 1 cluster
2. KHÔNG assume global ordering across clusters
3. Idempotent consumers — duplicate events từ replication
4. Deduplication by (entityId, version) hoặc event ID
5. Avoid bi-directional replication cùng topic name — dùng prefix (us.orders, eu.orders)
6. Consumer ở US chỉ process us.* topics hoặc locally-produced events
```

---

## Other Brokers Geo-Replication

### Apache Pulsar

Geo-replication **built-in** ở namespace level:

```bash
pulsar-admin namespaces set-clusters production/acme/orders \
  --clusters us-east, eu-west, ap-southeast
```

Xem chi tiết: [05-other-brokers/4-apache-pulsar.md](../05-other-brokers/4-apache-pulsar.md)

### RabbitMQ Federation & Shovel

```
Federation:  upstream exchange/queue → downstream exchange (one-way)
Shovel:      move messages giữa brokers (more control)

Không có active-active native — thường dùng active-passive
Quorum queues: HA trong cluster, không cross-DC replication built-in
```

### AWS Multi-Region

```
MSK (per region) + MSK Replicator (one-way) → DR pattern
SQS/SNS: Global không — per-region, fan-out cross-region manual
EventBridge: Cross-region event bus routing
```

### GCP Pub/Sub

```
Pub/Sub: Regional service — không native geo-replication
Pattern: Publish cùng message vào 2 regional topics (dual-write)
         hoặc Dataflow replicate
```

---

## Disaster Recovery Planning

### RPO & RTO Definitions

| Metric | Ý Nghĩa | Messaging Context |
| ------ | ------- | ----------------- |
| **RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi)** | Max data loss chấp nhận | Replication lag = potential lost messages |
| **RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi)** | Max downtime chấp nhận | Thời gian failover producers + consumers |

```
Async replication lag 30 giây → RPO ≈ 30 giây messages
Failover runbook 15 phút → RTO ≈ 15 phút
```

### Failover Runbook (Active-Passive Kafka)

```
Pre-requisites:
  □ MM2/Cluster Linking running, lag < SLA
  □ Consumer apps support cluster switch (config/bootstrap change)
  □ DNS/config management ready

Failover Steps:
  1. Confirm primary region down (not transient)
  2. Stop MM2 (prevent stale writes if primary recovers)
  3. Verify secondary cluster health
  4. Update producer bootstrap.servers → secondary
  5. Start consumers on secondary (offset from checkpoint)
  6. Monitor lag, error rate
  7. Communicate RPO gap to stakeholders

Failback Steps:
  1. Primary recovered and verified
  2. Reverse replicate secondary → primary
  3. Dual-write period
  4. Cutover back to primary
  5. Re-enable normal replication
```

### Dual-Write Migration Pattern

```
Phase 1: Primary only (US)
Phase 2: Primary + replicate → DR (EU standby)
Phase 3: Dual-write producers (US + EU) — migration period
Phase 4: Active-active or cutover
```

---

## Thiết Kế Thực Tế

### Global Order Processing

```
┌──────────────────────────────────────────────────────────────────┐
│                  GLOBAL ORDER SYSTEM (Active-Passive DR)            │
│                                                                  │
│  US-EAST (Primary)                    US-WEST (DR)               │
│  ┌─────────────┐   MM2 replicate   ┌─────────────┐              │
│  │ MSK Cluster │ ────────────────► │ MSK Cluster │              │
│  │ orders (12p)│                   │ us.orders   │              │
│  └──────┬──────┘                   └──────┬──────┘              │
│         │                                  │ (standby)          │
│    Producers US                       Failover consumers         │
│    Consumers US                                                  │
│                                                                  │
│  RPO: 60s (replication lag SLA)                                  │
│  RTO: 15 min (automated failover script)                         │
└──────────────────────────────────────────────────────────────────┘
```

### Production Checklist

```
□ Chọn pattern: active-passive (default) vs active-active (justified)
□ Replication tool deployed HA (MM2 Connect cluster)
□ Monitor replication lag — alert < 60s warning, < 5min critical
□ Topic naming policy (prefix per region cho bi-directional)
□ Consumer idempotent + dedup
□ Failover runbook tested quarterly (game day)
□ Network: dedicated cross-region link (AWS Direct Connect / ExpressRoute)
□ Cost model: double cluster + cross-region egress
□ Schema Registry: sync strategy (Schema Linking hoặc manual)
□ Document RPO/RTO cho stakeholders
```

### Anti-Patterns

```
❌ Active-active without conflict strategy — data corruption
❌ Assume global ordering across regions
❌ Failover không test — discover offset mapping issues khi incident
❌ Replicate ALL topics — cost và noise (chỉ business-critical)
❌ Ignore cross-region egress cost ($0.02–0.09/GB adds up)
❌ Bi-directional replicate same topic name — infinite loop
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Active-passive vs active-active messaging?

**Gợi ý trả lời:** **Active-passive**: một region primary xử lý traffic, region khác standby replicate async — đơn giản, DR pattern. **Active-active**: cả 2 regions xử lý traffic — low latency global nhưng cần **conflict resolution**, entity ownership, idempotent consumers. Default active-passive trừ khi có requirement rõ ràng.

### Câu 2: MirrorMaker 2 hoạt động thế nào?

**Gợi ý trả lời:** MM2 dùng **3 Kafka Connect connectors**: Source (copy records), Checkpoint (sync consumer offsets), Heartbeat (monitor lag). Topics replicated với prefix (`us.orders`). Async, at-least-once. Offset mapping approximate qua checkpoints — đủ cho DR failover.

### Câu 3: Làm sao tránh conflict trong active-active?

**Gợi ý trả lời:** **Thiết kế tránh conflict** tốt nhất: **entity ownership** — mỗi entity chỉ write ở 1 region (shard by user/tenant region). Nếu không tránh được: version vectors, LWW (cẩn thận), hoặc CRDT. Kafka: prefix topics per region, idempotent consumers, dedup by event ID.

### Câu 4: RPO/RTO trong messaging DR?

**Gợi ý trả lời:** **RPO** = max data loss = replication lag (async). **RTO** = thời gian restore service = failover script + producer/consumer cutover. Ví dụ: lag 30s → RPO 30s; runbook 15 phút → RTO 15 phút. Sync replication → RPO ≈ 0 nhưng latency cao.

### Câu 5: Kafka vs Pulsar geo-replication?

**Gợi ý trả lời:** **Pulsar** — built-in namespace-level geo-replication, config đơn giản. **Kafka** — cần MM2 hoặc Confluent Cluster Linking, ops phức tạp hơn nhưng ecosystem lớn hơn. Pulsar advantage multi-region by design; Kafka mature tooling và managed options (MSK Replicator, Cluster Linking).

### Câu 6: Thiết kế DR cho Kafka production?

**Gợi ý trả lời:** Active-passive 2 regions, **MM2/MSK Replicator** async replicate critical topics, monitor lag SLA, **failover runbook** tested quarterly, consumer **offset checkpoint**, producer bootstrap configurable, document **RPO/RTO**, game day drill. Replicate chỉ topics cần — không all topics.

---

**Xem tiếp:** [3-schema-evolution.md](./3-schema-evolution.md) — Enterprise schema governance.

**Liên quan:** [05-other-brokers/4-apache-pulsar.md](../05-other-brokers/4-apache-pulsar.md) — Pulsar geo-replication.
