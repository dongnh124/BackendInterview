# 📨 Message Queue & Event Broker — Lộ Trình Kiến Thức Toàn Diện

> Hướng dẫn toàn diện về Message Queue (Hàng Đợi Tin Nhắn), Event Broker (Broker Sự Kiện) và Event-Driven Architecture (Kiến Trúc Hướng Sự Kiện) — từ nền tảng lý thuyết đến vận hành production với Apache Kafka, RabbitMQ và các nền tảng cloud.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Nền Tảng & Công Nghệ](#nền-tảng--công-nghệ)
4. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
5. [Liên Kết Nhanh](#liên-kết-nhanh)
6. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
7. [Bắt Đầu Học](#bắt-đầu-học)
8. [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)
9. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)
10. [Tự Đánh Giá](#tự-đánh-giá)
11. [Cách Sử Dụng Tài Liệu](#cách-sử-dụng-tài-liệu)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Message Queue (Hàng Đợi Tin Nhắn) vs Event Broker (Broker Sự Kiện) — Khái niệm, use case
- [ ] Point-to-Point (Điểm-Điểm) vs Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký)
- [ ] Delivery Semantics (Ngữ Nghĩa Giao Hàng) — At-most-once, At-least-once, Exactly-once
- [ ] Message Ordering (Thứ Tự Tin Nhắn) & Idempotency (Tính Bất Biến Khi Lặp Lại)
- [ ] Backpressure (Áp Lực Ngược) & Flow Control (Điều Khiển Luồng)

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)**

- [ ] Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện) — Producer, Consumer, Broker
- [ ] Apache Kafka — Topics, Partitions (Phân Vùng), Consumer Groups (Nhóm Consumer)
- [ ] RabbitMQ — Exchange (Bộ Trao Đổi), Queue (Hàng Đợi), Binding (Liên Kết)
- [ ] Dead Letter Queue (DLQ — Hàng Đợi Thư Chết) & Retry Strategy (Chiến Lược Thử Lại)
- [ ] Outbox Pattern (Mẫu Hộp Thoại Ra) & Transactional Messaging (Nhắn Tin Giao Dịch)
- [ ] Schema Registry (Đăng Ký Schema) — Avro, Protobuf, JSON Schema

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Saga Pattern (Mẫu Saga) — Choreography vs Orchestration
- [ ] CQRS (Command Query Responsibility Segregation — Tách Trách Nhiệm Đọc/Ghi) & Event Sourcing (Lưu Trữ Sự Kiện)
- [ ] Performance Tuning (Tối Ưu Hiệu Năng) — Throughput (Thông Lượng), Latency (Độ Trễ), Partitioning
- [ ] Monitoring (Giám Sát) — Consumer Lag (Độ Trễ Consumer), Broker Metrics (Chỉ Số Broker)
- [ ] Security (Bảo Mật) — SASL, ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập), TLS
- [ ] Cloud Managed Services (Dịch Vụ Quản Lý Trên Cloud) — MSK, Confluent Cloud, Event Hubs

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Kafka Streams & ksqlDB — Stream Processing (Xử Lý Luồng)
- [ ] Multi-datacenter Replication (Nhân Bản Đa Trung Tâm Dữ Liệu)
- [ ] Exactly-once Semantics (Ngữ Nghĩa Đúng Một Lần) trong Kafka
- [ ] Platform Comparison (So Sánh Nền Tảng) — Kafka vs RabbitMQ vs Pulsar vs NATS
- [ ] System Design (Thiết Kế Hệ Thống) với Event-Driven Architecture

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| -------- | ---------- | --------- | ---------- |
| **Messaging Fundamentals (Nền Tảng Nhắn Tin)** | ⭐⭐⭐ | 1 tuần | - |
| **Delivery Guarantees (Đảm Bảo Giao Hàng)** | ⭐⭐⭐ | 1 tuần | - |
| **Apache Kafka** | ⭐⭐⭐ | 3 tuần | - |
| **RabbitMQ** | ⭐⭐⭐ | 2 tuần | - |
| **Event-Driven Patterns (Mẫu Hướng Sự Kiện)** | ⭐⭐⭐ | 2 tuần | - |
| **Reliability & Error Handling (Độ Tin Cậy & Xử Lý Lỗi)** | ⭐⭐⭐ | 1.5 tuần | - |
| **Monitoring & Troubleshooting (Giám Sát & Xử Lý Sự Cố)** | ⭐⭐⭐ | 1 tuần | - |
| **Performance & Scaling (Hiệu Năng & Mở Rộng)** | ⭐⭐ | 2 tuần | - |
| **Security & Compliance (Bảo Mật & Tuân Thủ)** | ⭐⭐ | 1 tuần | - |
| **Cloud Managed Messaging (Nhắn Tin Quản Lý Trên Cloud)** | ⭐⭐ | 2 tuần | - |

---

## 🗂️ Nền Tảng & Công Nghệ

### **Apache Kafka**

```
Điểm mạnh: Throughput cao, Event Log bền vững, Replay (Phát Lại), Stream Processing
Phù hợp: Event streaming, log aggregation, analytics pipeline, CDC (Change Data Capture)
Nội dung: 01-fundamentals, 03-apache-kafka, 07-performance-scaling
```

### **RabbitMQ**

```
Điểm mạnh: Routing linh hoạt, AMQP protocol, dễ triển khai, DLQ tích hợp
Phù hợp: Task queue, RPC (Remote Procedure Call), workflow, microservices decoupling
Nội dung: 01-fundamentals, 04-rabbitmq, 06-reliability
```

### **Amazon SQS / SNS**

```
Điểm mạnh: Serverless, managed, tích hợp AWS ecosystem
Phù hợp: AWS-native apps, fan-out notifications, decoupled services
Nội dung: 05-other-brokers, 10-cloud-managed
```

### **Redis Streams / Pub/Sub**

```
Điểm mạnh: Ultra-low latency, đơn giản, đã có sẵn trong stack caching
Phù hợp: Real-time notifications, lightweight messaging, session events
Nội dung: 05-other-brokers
```

### **Apache Pulsar / NATS JetStream**

```
Điểm mạnh: Multi-tenancy, geo-replication, unified queue + stream model
Phù hợp: Multi-tenant SaaS, cloud-native messaging, edge computing
Nội dung: 05-other-brokers, 11-advanced
```

---

## 📁 Tổng Quan Chủ Đề

### 📁 **1. Fundamentals (Nền Tảng)** (`01-fundamentals/`)

- Message Queue vs Event Broker — Sự khác biệt và khi nào dùng
- Point-to-Point vs Pub/Sub — Mô hình giao tiếp
- Delivery Semantics — At-most-once, At-least-once, Exactly-once
- Message Ordering & Sequencing (Thứ Tự & Tuần Tự Hóa)
- Backpressure & Flow Control

### 📁 **2. Architecture Patterns (Mẫu Kiến Trúc)** (`02-architecture-patterns/`)

- Event-Driven Architecture (EDA)
- CQRS & Event Sourcing
- Saga Pattern — Choreography vs Orchestration
- Outbox Pattern & Inbox Pattern
- Idempotency (Tính Bất Biến) & Deduplication (Khử Trùng Lặp)

### 📁 **3. Apache Kafka** (`03-apache-kafka/`)

- **Kafka Architecture** — Broker, ZooKeeper/KRaft, Cluster
- **Topics & Partitions** — Partitioning strategy, key-based routing
- **Consumer Groups** — Rebalancing, scale-out consumers
- **Producers** — Acknowledgment (acks), compression, batching
- **Offset Management** — Commit strategy, auto vs manual
- **Kafka Streams & Connect** — Stream processing, CDC integration

### 📁 **4. RabbitMQ** (`04-rabbitmq/`)

- **Exchanges & Queues** — Direct, Fanout, Topic, Headers
- **Routing Patterns** — Work queue, pub/sub, routing, topics
- **Dead Letter Exchange (DLX)** — Poison message handling
- **Clustering & HA** — Mirrored queues, quorum queues
- **Publisher Confirms & Consumer Ack** — Reliable delivery

### 📁 **5. Other Brokers (Broker Khác)** (`05-other-brokers/`)

- Amazon SQS & SNS — FIFO queue, fan-out pattern
- Redis Streams & Pub/Sub — Lightweight messaging
- NATS & JetStream — Cloud-native messaging
- Apache Pulsar — Multi-tenancy, geo-replication

### 📁 **6. Reliability (Độ Tin Cậy)** (`06-reliability/`)

- At-least-once vs Exactly-once — Trade-offs thực tế
- Retry & Exponential Backoff (Lùi Lũy Tiến)
- Dead Letter Queue (DLQ) — Design & operations
- Poison Message (Tin Nhắn Độc) Detection
- Circuit Breaker (Cầu Dao Bảo Vệ) cho consumers

### 📁 **7. Performance & Scaling (Hiệu Năng & Mở Rộng)** (`07-performance-scaling/`)

- Throughput Tuning — Batch size, linger.ms, compression
- Partitioning Strategies — Key design, hot partition
- Consumer Scaling — Parallelism, partition count
- Backpressure Handling — Rate limiting, throttling
- Broker Sizing & Capacity Planning (Lập Kế Hoạch Dung Lượng)

### 📁 **8. Security (Bảo Mật)** (`08-security/`)

- Authentication — SASL/SCRAM, mTLS (Mutual TLS — TLS Hai Chiều)
- Authorization — ACL, RBAC (Role-Based Access Control)
- Encryption — In-transit (Truyền Tải) & At-rest (Lưu Trữ)
- Schema Validation & Schema Registry security
- Audit Logging (Ghi Nhật Ký Kiểm Toán)

### 📁 **9. Monitoring & Observability (Giám Sát & Quan Sát)** (`09-monitoring/`)

- Key Metrics — Lag, throughput, error rate, rebalance
- Alerting Thresholds & SLOs (Service Level Objectives — Mục Tiêu Mức Dịch Vụ)
- Prometheus & Grafana cho Kafka/RabbitMQ
- Distributed Tracing (Truy Vết Phân Tán) — OpenTelemetry, correlation ID
- Troubleshooting Playbook (Sổ Tay Xử Lý Sự Cố)

### 📁 **10. Cloud Managed (Dịch Vụ Quản Lý Trên Cloud)** (`10-cloud-managed/`)

- **AWS MSK** (Managed Streaming for Apache Kafka)
- **Confluent Cloud** — Schema Registry, ksqlDB
- **Azure Event Hubs** — Kafka-compatible endpoint
- **GCP Pub/Sub** — Push/Pull subscriptions
- Cost Optimization (Tối Ưu Chi Phí) & Migration (Di Chuyển)

### 📁 **11. Advanced Topics (Chủ Đề Nâng Cao)** (`11-advanced/`)

- Kafka Streams & ksqlDB
- Change Data Capture (CDC) — Debezium, Kafka Connect
- Multi-datacenter & Geo-replication
- Event Schema Evolution (Tiến Hóa Schema Sự Kiện)
- Serverless Event Processing — Lambda, Cloud Functions

### 📁 **12. Interview Prep (Chuẩn Bị Phỏng Vấn)** (`12-interview-prep/`)

- Top 30 Message Queue Interview Questions
- System Design Scenarios — Order processing, notification system
- Incident Response Stories (STAR method)
- Broker Selection Decision Tree (Cây Quyết Định Chọn Broker)
- 90-day Study Plan (Kế Hoạch Học 90 Ngày)

---

## 🔗 Liên Kết Nhanh

| Chủ Đề | Thư Mục | Độ Ưu Tiên |
| ------ | ------- | ---------- |
| Bắt đầu tại đây | [INDEX.md](./INDEX.md) | Đọc trước |
| Lộ trình chi tiết | [ROADMAP.md](./ROADMAP.md) | Lập kế hoạch |
| Câu hỏi phỏng vấn | [12-interview-prep](./12-interview-prep/) | Trước phỏng vấn |
| Delivery Guarantees | [01-fundamentals/delivery-guarantees.md](./01-fundamentals/delivery-guarantees.md) | Bắt buộc |
| Kafka Deep Dive | [03-apache-kafka/README.md](./03-apache-kafka/README.md) | Nền tảng chính |
| Production Checklist | [09-monitoring/7-production-checklist.md](./09-monitoring/7-production-checklist.md) | Trước go-live |

---

## 📊 Ma Trận Kỹ Năng

### Beginner (0–1 năm)

- [ ] Hiểu Message Queue vs Event Broker
- [ ] Biết Point-to-Point vs Pub/Sub
- [ ] Giải thích được At-least-once delivery
- [ ] Cài đặt Kafka/RabbitMQ local với Docker
- [ ] Viết producer/consumer đơn giản

### Intermediate (1–3 năm)

- [ ] Thiết kế topic/queue structure hợp lý
- [ ] Cấu hình Consumer Groups & scaling
- [ ] Implement DLQ & retry strategy
- [ ] Áp dụng Outbox Pattern
- [ ] Monitor consumer lag & troubleshoot
- [ ] So sánh trade-offs Kafka vs RabbitMQ

### Advanced (3–5+ năm)

- [ ] Thiết kế Event-Driven Architecture cho microservices
- [ ] Implement Saga Pattern (choreography/orchestration)
- [ ] Tune Kafka cluster cho throughput cao
- [ ] Exactly-once semantics trong production
- [ ] Multi-datacenter replication strategy
- [ ] Incident command & post-mortem cho messaging outages

---

## 🚀 Bắt Đầu Học

### Bước 1: Chọn Lộ Trình

```
Chọn hướng học:
- Generalist (Tổng Quát): Kafka + RabbitMQ + patterns
- Kafka Specialist: Deep dive Kafka ecosystem
- Cloud-focused: MSK, Event Hubs, Pub/Sub
- Integration-focused: CDC, event sourcing, saga
```

### Bước 2: Dựng Môi Trường Lab

```bash
# Docker Compose stack cho Kafka + RabbitMQ
docker-compose up -d

# Bao gồm: ZooKeeper/KRaft, Kafka, RabbitMQ, Kafka UI, Redis
```

### Bước 3: Học + Thực Hành

```
1. Đọc một module (30 phút)
2. Chạy lab local (30 phút)
3. Viết producer/consumer thực tế (45–60 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation (Tình Huống)
- Task (Nhiệm Vụ)
- Action (Hành Động)
- Result (Kết Quả)
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Designing Data-Intensive Applications"** by Martin Kleppmann — Chương về messaging, logs, streams
- **"Kafka: The Definitive Guide"** by Neha Narkhede et al. — Kafka chuyên sâu
- **"Enterprise Integration Patterns"** by Hohpe & Woolf — Messaging patterns kinh điển
- **"Building Event-Driven Microservices"** by Adam Bellemare — EDA thực chiến

### Tài Liệu Chính Thức

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Confluent Developer Guides](https://developer.confluent.io/)
- [AWS MSK Documentation](https://docs.aws.amazon.com/msk/)

### Blog & Bài Viết Quan Trọng

- Confluent Blog — Kafka best practices
- CloudAMQP Blog — RabbitMQ tutorials
- Martin Kleppmann's blog — Stream processing, exactly-once
- InfoQ — Event-driven architecture articles

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### Fundamentals (Nền Tảng)

- [ ] Message Queue khác Event Broker như thế nào?
- [ ] Giải thích At-least-once, At-most-once, Exactly-once
- [ ] Khi nào dùng sync API vs async messaging?
- [ ] Idempotency là gì? Tại sao quan trọng?

#### Kafka

- [ ] Topic, Partition, Consumer Group hoạt động thế nào?
- [ ] Cách scale consumer khi lag tăng?
- [ ] acks=all vs acks=1 — trade-offs?
- [ ] Exactly-once semantics trong Kafka hoạt động ra sao?

#### RabbitMQ

- [ ] Các loại Exchange và khi nào dùng?
- [ ] Dead Letter Queue thiết kế như thế nào?
- [ ] Publisher Confirm vs Consumer Ack?

#### Architecture (Kiến Trúc)

- [ ] Outbox Pattern giải quyết vấn đề gì?
- [ ] Saga Choreography vs Orchestration?
- [ ] Event Sourcing vs CRUD — trade-offs?

#### Sự Cố Thực Tế

- [ ] Kể về incident liên quan messaging (STAR)
- [ ] Consumer lag spike — cách diagnose & fix
- [ ] Message loss — root cause & prevention

Xem `12-interview-prep/` để có hướng dẫn Q&A đầy đủ.

---

## ✅ Tự Đánh Giá

Trước phỏng vấn hoặc khi đảm nhận vai trò mới, kiểm tra:

- [ ] Giải thích được delivery semantics không cần nhìn tài liệu
- [ ] Thiết kế được messaging flow cho order processing
- [ ] Cấu hình Kafka consumer group & partition strategy
- [ ] Implement DLQ + retry cho RabbitMQ
- [ ] Troubleshoot consumer lag systematically
- [ ] So sánh Kafka vs RabbitMQ cho use case cụ thể
- [ ] Áp dụng Outbox Pattern trong transactional system
- [ ] Thiết kế monitoring & alerting cho messaging layer
- [ ] Giải thích trade-offs của event-driven vs request-response
- [ ] Kể được 2–3 câu chuyện incident thực tế (STAR)

---

## 📋 Cách Sử Dụng Tài Liệu

### Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi tuần tự qua từng giai đoạn
3. Làm bài thực hành với Docker lab
4. Xây dựng project portfolio (ví dụ: order processing pipeline)

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào [12-interview-prep](./12-interview-prep/)
2. Học sâu broker mục tiêu (Kafka hoặc RabbitMQ)
3. Chuẩn bị câu chuyện incident (STAR)
4. Luyện giải thích trade-offs rõ ràng

### Trên Công Việc

1. Tham khảo [03-apache-kafka](./03-apache-kafka/) hoặc [04-rabbitmq](./04-rabbitmq/) khi tích hợp
2. Dùng [09-monitoring](./09-monitoring/) để setup observability
3. Dùng [06-reliability](./06-reliability/) khi xử lý lỗi & retry
4. Kiểm tra [7-production-checklist](./09-monitoring/7-production-checklist.md) trước go-live

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Xem INDEX.md để nắm cấu trúc toàn bộ
├─ 3️⃣  Chọn lộ trình (Beginner/Intermediate/Advanced)
├─ 4️⃣  Bắt đầu với 01-fundamentals/
├─ 5️⃣  Dựng lab Docker (Kafka + RabbitMQ)
├─ 6️⃣  Hoàn thành bài tập cho từng chủ đề
├─ 7️⃣  Xây dựng project thực tế (order/event pipeline)
└─ 8️⃣  Ôn phỏng vấn với 12-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-07-03
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
