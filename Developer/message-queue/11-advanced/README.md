# Advanced Topics — Chủ Đề Nâng Cao

> Kiến thức chuyên sâu cho messaging ở quy mô enterprise: Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu), Multi-datacenter Replication (Nhân Bản Đa Trung Tâm Dữ Liệu), Event Schema Evolution (Tiến Hóa Schema Sự Kiện), và Serverless Event Processing (Xử Lý Sự Kiện Serverless).

## Mục Lục

1. [Tại Sao Học Advanced Topics](#tại-sao-học-advanced-topics)
2. [Điều Kiện Tiên Quyết](#điều-kiện-tiên-quyết)
3. [Tổng Quan 4 Chủ Đề](#tổng-quan-4-chủ-đề)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Kiến Trúc Tổng Hợp](#kiến-trúc-tổng-hợp)
7. [Bài Tập Thực Hành](#bài-tập-thực-hành)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học Advanced Topics

Sau khi nắm [Kafka](../03-apache-kafka/README.md), [RabbitMQ](../04-rabbitmq/README.md), [Reliability](../06-reliability/README.md), và [Cloud Run deployments](../09-monitoring/README.md), bạn cần advanced topics vì:

- **CDC** là nền tảng của event-driven sync — thay thế batch ETL và polling database
- **Multi-datacenter** là yêu cầu thực tế cho global SaaS, disaster recovery (DR — Phục Hồi Thảm Họa), và data residency (yêu cầu dữ liệu theo vùng địa lý)
- **Schema evolution** ở scale lớn cần governance — nhiều team, nhiều consumer, không thể "đổi schema tùy ý"
- **Serverless processing** phổ biến trên cloud — Lambda, Cloud Functions kết hợp với event bus

| Chủ Đề | Khi Nào Cần | Độ Khó |
| ------ | ----------- | ------ |
| **CDC** | Sync DB → search/analytics/cache, event sourcing bootstrap | ⭐⭐⭐ |
| **Multi-DC** | Global users, DR, compliance multi-region | ⭐⭐⭐⭐ |
| **Schema Evolution** | Nhiều microservices share events, long-lived topics | ⭐⭐⭐ |
| **Serverless** | Spiky traffic, cost optimization, cloud-native | ⭐⭐ |

---

## Điều Kiện Tiên Quyết

Hoàn thành hoặc nắm vững các chủ đề sau trước khi học module này:

| Chủ Đề | File | Lý Do |
| ------ | ---- | ----- |
| Kafka Connect basics | [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md) | CDC thường dùng Debezium qua Connect |
| Schema Registry | [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) | CDC events cần schema contract |
| Outbox Pattern | [02-architecture-patterns/4-outbox-inbox-pattern.md](../02-architecture-patterns/4-outbox-inbox-pattern.md) | So sánh CDC vs Outbox |
| Idempotency | [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md) | CDC consumer bắt buộc idempotent |
| Cloud managed | [10-cloud-managed/README.md](../10-cloud-managed/README.md) | Serverless thường trên MSK, Pub/Sub, Event Hubs |

---

## Tổng Quan 4 Chủ Đề

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ADVANCED MESSAGING LANDSCAPE                              │
│                                                                             │
│  ┌─────────────┐    CDC (Debezium)     ┌─────────────┐                     │
│  │ PostgreSQL  │ ─────────────────────►│ Kafka Topic │                     │
│  │ MySQL       │                       └──────┬──────┘                     │
│  └─────────────┘                              │                             │
│                                               ├──► Search (Elasticsearch)   │
│                                               ├──► Cache (Redis)            │
│                                               └──► Serverless (Lambda)      │
│                                                                             │
│  Multi-DC:  US Cluster ◄──MirrorMaker/Cluster Linking──► EU Cluster        │
│                                                                             │
│  Schema:    Producer v3 ──► Schema Registry ──► Consumer v2 (BACKWARD)   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Change Data Capture (CDC)

Capture thay đổi row-level từ database transaction log → publish events realtime. Thay thế:

```
❌ Polling: SELECT * FROM orders WHERE updated_at > ?  (mỗi 5 phút)
✅ CDC:     INSERT/UPDATE/DELETE → event ngay lập tức
```

### 2. Multi-Datacenter Replication

Replicate topics/events across regions cho DR, latency, và compliance. Patterns:

| Pattern | Mô Tả | Phù Hợp |
| ------- | ----- | ------- |
| **Active-Passive** | Một region primary, region khác standby | DR, failover |
| **Active-Active** | Cả hai region nhận traffic | Global low latency |
| **Hub-and-Spoke** | Central hub replicate ra edge regions | Retail, CDN-like |

### 3. Schema Evolution

Quản lý thay đổi event schema khi nhiều team deploy độc lập — compatibility modes, governance, breaking change strategy.

### 4. Serverless Event Processing

Event-driven compute không quản lý server — AWS Lambda, GCP Cloud Functions, Azure Functions trigger từ Kafka, SQS, Pub/Sub.

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 10–15 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-change-data-capture.md](./1-change-data-capture.md) | Debezium, CDC patterns, event-driven sync | 3–4 giờ |
| 2 | [2-multi-datacenter.md](./2-multi-datacenter.md) | Geo-replication, active-active, conflict resolution | 3–4 giờ |
| 3 | [3-schema-evolution.md](./3-schema-evolution.md) | Compatibility, governance, breaking changes | 2–3 giờ |
| 4 | [4-serverless-processing.md](./4-serverless-processing.md) | Lambda, Cloud Functions, event triggers | 2–3 giờ |

**Thứ tự đề xuất:**

```
1. CDC trước — nền tảng cho nhiều pipeline thực tế
2. Schema evolution — cần khi CDC events có contract
3. Multi-DC — khi scale global hoặc phỏng vấn system design
4. Serverless — áp dụng ngay trên cloud stack hiện tại
```

---

## Các Tài Liệu Chi Tiết

| File | Mô Tả | Độ Ưu Tiên |
| ---- | ----- | ---------- |
| [1-change-data-capture.md](./1-change-data-capture.md) | Debezium, CDC vs Outbox, snapshot, SMT, sink patterns | ⭐⭐⭐ Integration teams |
| [2-multi-datacenter.md](./2-multi-datacenter.md) | MirrorMaker 2, Cluster Linking, active-active conflicts | ⭐⭐⭐ Global systems |
| [3-schema-evolution.md](./3-schema-evolution.md) | Enterprise governance, multi-team contracts, migration | ⭐⭐ Platform teams |
| [4-serverless-processing.md](./4-serverless-processing.md) | Lambda + MSK, Pub/Sub push, cold start, cost | ⭐⭐ Cloud-native |

---

## Kiến Trúc Tổng Hợp

### Reference Architecture: Event-Driven Data Platform

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         EVENT-DRIVEN DATA PLATFORM                        │
│                                                                          │
│  ┌──────────┐   CDC    ┌─────────┐   Stream   ┌──────────────┐          │
│  │ OLTP DB  │ ────────►│  Kafka  │ ─────────►│ Kafka Streams│          │
│  │ (Postgres)│         │ Cluster │           │ / Flink      │          │
│  └──────────┘          └────┬────┘           └──────┬───────┘          │
│                             │                       │                  │
│         Schema Registry ◄───┤                       │                  │
│                             │                       ▼                  │
│                    ┌────────┼────────┐         ┌──────────────┐          │
│                    ▼        ▼        ▼         │ Data Warehouse│         │
│              ┌─────────┐ ┌────┐ ┌────────┐   │ (Snowflake)  │          │
│              │ Lambda  │ │ ES │ │ Redis  │   └──────────────┘          │
│              │(serverless)│ │    │ │ Cache  │                            │
│              └─────────┘ └────┘ └────────┘                              │
│                                                                          │
│  Multi-DC: Kafka US ◄── MM2 / Cluster Linking ──► Kafka EU              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Bài Tập Thực Hành

### Lab 1: CDC Pipeline End-to-End (2 giờ)

```
Mục tiêu: PostgreSQL → Debezium → Kafka → consumer update Elasticsearch

Bước:
1. Docker Compose: Postgres + Kafka + Debezium Connect + Elasticsearch
2. Enable logical replication trên Postgres
3. Deploy Debezium connector (snapshot.mode=initial)
4. INSERT/UPDATE rows → verify events trên topic
5. Consumer index vào ES — search realtime
6. Simulate connector restart — verify offset resume
```

### Lab 2: Schema Evolution Migration (90 phút)

```
Mục tiêu: Thêm field bắt buộc vào event schema không downtime

Bước:
1. Schema v1: { orderId, amount }
2. Deploy consumer v2 (đọc v2 với default)
3. Register schema v2 (currency optional + default)
4. Upgrade producer gửi v2
5. Thử breaking change (đổi type) → observe Registry reject
6. Document dual-topic migration plan
```

### Lab 3: Multi-Region Failover Drill (2 giờ)

```
Mục tiêu: Active-passive failover simulation

Bước:
1. 2 Kafka clusters (hoặc 2 Docker networks) + MirrorMaker 2
2. Producer ghi US cluster
3. Consumer EU đọc replicated topic
4. Kill US cluster → promote EU (manual runbook)
5. Measure RPO (Recovery Point Objective) và RTO (Recovery Time Objective)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| CDC vs Outbox Pattern? | CDC: capture mọi DB change tự động; Outbox: chỉ events app chủ đích ghi, transactional guarantee chặt hơn |
| Debezium snapshot mode? | `initial` full snapshot rồi streaming; `never` chỉ streaming; `when_needed` adaptive |
| Active-active messaging conflicts? | Per-key ordering, CRDT (Conflict-free Replicated Data Type), last-write-wins có rủi ro, design bounded context |
| Schema breaking change strategy? | Topic versioning, dual-write migration, không force incompatible evolution |
| Lambda consume Kafka challenges? | Batch size, offset commit, cold start, VPC networking, không long-running |
| MirrorMaker 2 vs Confluent Cluster Linking? | MM2 open source, offset mapping manual; Cluster Linking managed, bi-directional, enterprise |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Kafka Connect & Debezium intro | [03-apache-kafka/7-kafka-connect.md](../03-apache-kafka/7-kafka-connect.md) |
| Schema Registry basics | [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) |
| Pulsar geo-replication | [05-other-brokers/4-apache-pulsar.md](../05-other-brokers/4-apache-pulsar.md) |
| Outbox Pattern | [02-architecture-patterns/4-outbox-inbox-pattern.md](../02-architecture-patterns/4-outbox-inbox-pattern.md) |
| AWS MSK & Lambda | [10-cloud-managed/1-aws-msk.md](../10-cloud-managed/1-aws-msk.md) |
| GCP Pub/Sub push | [10-cloud-managed/4-gcp-pubsub.md](../10-cloud-managed/4-gcp-pubsub.md) |

---

**Cập Nhật:** 2026-07-03
