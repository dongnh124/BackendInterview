# Kế Hoạch Học 90 Ngày — Message Queue & Event Broker

> Lộ trình học có cấu trúc 90 ngày từ Beginner đến sẵn sàng phỏng vấn messaging, kết hợp theory (lý thuyết) và practice (thực hành) hands-on với Kafka và RabbitMQ.

## Mục Lục

1. [Tổng Quan 90 Ngày](#tổng-quan-90-ngày)
2. [Giai Đoạn 1: Nền Tảng (Ngày 1–14)](#giai-đoạn-1-nền-tảng-ngày-114)
3. [Giai Đoạn 2: Kafka & RabbitMQ (Ngày 15–35)](#giai-đoạn-2-kafka--rabbitmq-ngày-1535)
4. [Giai Đoạn 3: Patterns & Reliability (Ngày 36–56)](#giai-đoạn-3-patterns--reliability-ngày-3656)
5. [Giai Đoạn 4: Production Skills (Ngày 57–77)](#giai-đoạn-4-production-skills-ngày-5777)
6. [Giai Đoạn 5: Interview Prep (Ngày 78–90)](#giai-đoạn-5-interview-prep-ngày-7890)
7. [Portfolio Project](#portfolio-project)
8. [Theo Dõi Tiến Độ](#theo-dõi-tiến-độ)

---

## Tổng Quan 90 Ngày

```
Tuần 1–2   ████████░░░░░░░░░░░░  Nền tảng messaging
Tuần 3–5  ░░░░████████░░░░░░░░  Kafka & RabbitMQ hands-on
Tuần 6–8  ░░░░░░░░████████░░░░  EDA patterns & reliability
Tuần 9–11 ░░░░░░░░░░░░████████  Monitoring, security, cloud
Tuần 12–13░░░░░░░░░░░░░░░░████  Interview prep & mock
```

| Giai Đoạn | Ngày | Giờ/Tuần | Mục Tiêu |
| --------- | ---- | -------- | -------- |
| 1. Nền Tảng | 1–14 | 8–10h | MQ fundamentals, delivery semantics |
| 2. Brokers | 15–35 | 12–15h | Kafka + RabbitMQ producer/consumer |
| 3. Patterns | 36–56 | 10–12h | Outbox, Saga, DLQ, idempotency |
| 4. Production | 57–77 | 10–12h | Monitoring, performance, security |
| 5. Interview | 78–90 | 15–20h | System design, mock interviews |

**Tổng ước tính:** 90–120 giờ (~1–1.5h/ngày trung bình)

---

## Giai Đoạn 1: Nền Tảng (Ngày 1–14)

### Tuần 1: Messaging Fundamentals

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 1 | Setup Docker lab: Kafka + RabbitMQ + UI | README.md | `docker compose up -d` |
| 2 | MQ vs Event Broker, terminology | 01-fundamentals/1-message-queue-basics.md | Vẽ diagram use cases |
| 3 | Point-to-Point vs Pub/Sub | 01-fundamentals/2-pub-sub-vs-point-to-point.md | So sánh 3 scenarios |
| 4 | Delivery guarantees — **bắt buộc** | 01-fundamentals/3-delivery-guarantees.md | Quiz 10 câu tự kiểm tra |
| 5 | Message ordering, partition keys | 01-fundamentals/4-ordering-and-sequencing.md | Thiết kế key cho order events |
| 6 | Backpressure, flow control | 01-fundamentals/5-backpressure-flow-control.md | Ghi chú 3 triệu chứng |
| 7 | **Review tuần 1** | INTERVIEW_GUIDE Câu 1–7 | Giải thích aloud 3 delivery semantics |

### Tuần 2: Broker Selection & EDA Intro

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 8 | Broker selection decision tree | 01-fundamentals/6-broker-selection-guide.md | Chọn broker cho 2 use cases |
| 9 | Event-Driven Architecture basics | 02-architecture-patterns/1-event-driven-architecture.md | Vẽ EDA cho order flow |
| 10 | Idempotency & dedup | 02-architecture-patterns/5-idempotency-dedup.md | Implement idempotency table schema |
| 11 | Kafka architecture overview | 03-apache-kafka/1-kafka-architecture.md | Explore Kafka UI topics |
| 12 | RabbitMQ exchanges overview | 04-rabbitmq/1-exchanges-queues-bindings.md | Tạo direct exchange + queue |
| 13 | Ôn tập + mini quiz | — | 20 câu fundamentals |
| 14 | **Checkpoint 1** | INTERVIEW_GUIDE Câu 1–8 | Mock 15 phút: delivery guarantees |

**Milestone Giai Đoạn 1:** Giải thích delivery semantics, so sánh Kafka vs RabbitMQ, chạy được Docker lab.

---

## Giai Đoạn 2: Kafka & RabbitMQ (Ngày 15–35)

### Tuần 3: Apache Kafka Deep Dive

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 15 | Topics & partitions | 03-apache-kafka/2-topics-partitions.md | Tạo topic 3 partitions |
| 16 | Producer: acks, batching | 03-apache-kafka/4-producers-serialization.md | Producer với acks=all |
| 17 | Consumer groups | 03-apache-kafka/3-consumer-groups.md | 3 consumers cùng group |
| 18 | Offset management | 03-apache-kafka/5-offset-management.md | Manual commit consumer |
| 19 | Schema Registry basics | 03-apache-kafka/6-schema-registry.md | Đọc Avro schema concepts |
| 20 | Gây lỗi: kill consumer | — | Observe rebalance + lag |
| 21 | **Review Kafka** | 2-kafka-questions.md Câu 1–10 | Trả lời 10 câu |

### Tuần 4: RabbitMQ Deep Dive

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 22 | Routing patterns | 04-rabbitmq/2-routing-patterns.md | Work queue + 2 workers |
| 23 | Topic exchange | 04-rabbitmq/1-exchanges-queues-bindings.md | Fan-out notification |
| 24 | Publisher confirms | 04-rabbitmq/4-publisher-confirms.md | Confirm mode producer |
| 25 | DLQ design | 04-rabbitmq/3-dead-letter-queue.md | Setup DLX + DLQ |
| 26 | Consumer ack modes | 04-rabbitmq/4-publisher-confirms.md | Manual ack consumer |
| 27 | Gây lỗi: poison message | 06-reliability/4-poison-message.md | Message vào DLQ |
| 28 | **Review RabbitMQ** | 3-rabbitmq-questions.md | Trả lời 10 câu |

### Tuần 5: Hands-on Integration

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 29–31 | **Lab: Order Event Pipeline** | 1-system-design-scenarios.md | Kafka producer + 2 consumers |
| 32 | Kafka Connect intro | 03-apache-kafka/7-kafka-connect.md | Đọc Debezium architecture |
| 33 | Other brokers overview | 05-other-brokers/README.md | So sánh SQS, Redis Streams |
| 34 | Performance tuning intro | 07-performance-scaling/1-throughput-tuning.md | Tune batch.size, linger.ms |
| 35 | **Checkpoint 2** | INTERVIEW_GUIDE Câu 15–24 | Mock 20 phút: Kafka vs RabbitMQ |

**Milestone Giai Đoạn 2:** Viết producer/consumer cho cả Kafka và RabbitMQ; setup DLQ; hiểu consumer groups.

---

## Giai Đoạn 3: Patterns & Reliability (Ngày 36–56)

### Tuần 6: Architecture Patterns

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 36 | Outbox Pattern | 02-architecture-patterns/4-outbox-inbox-pattern.md | Thiết kế outbox table |
| 37 | Saga Pattern | 02-architecture-patterns/3-saga-pattern.md | Vẽ choreography flow |
| 38 | CQRS & Event Sourcing | 02-architecture-patterns/2-cqrs-event-sourcing.md | So sánh với CRUD |
| 39 | At-least-once thực tế | 06-reliability/1-at-least-once-exactly-once.md | Effective exactly-once diagram |
| 40 | Retry & backoff | 06-reliability/2-retry-backoff.md | Implement exponential backoff |
| 41 | DLQ operations | 06-reliability/3-dead-letter-handling.md | DLQ replay procedure |
| 42 | **Review patterns** | INTERVIEW_GUIDE Câu 9–14 | Giải thích Outbox aloud |

### Tuần 7: Reliability & Failure Testing

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 43 | Poison message handling | 06-reliability/4-poison-message.md | Quarantine workflow |
| 44 | Circuit breaker | 06-reliability/5-circuit-breaker-consumers.md | Design circuit breaker states |
| 45 | Partitioning strategies | 07-performance-scaling/2-partitioning-strategies.md | Fix hot partition scenario |
| 46 | Consumer scaling | 07-performance-scaling/3-consumer-scaling.md | Scale lab consumers |
| 47 | Failure testing | — | Kill broker, network delay |
| 48 | **Lab: Outbox implementation** | — | Outbox table + poller |
| 49 | **Review reliability** | INTERVIEW_GUIDE Câu 25–29 | STAR story draft #1 |

### Tuần 8: Advanced Topics Intro

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 50 | CDC với Debezium | 11-advanced/1-change-data-capture.md | Đọc CDC architecture |
| 51 | Schema evolution | 11-advanced/3-schema-evolution.md | Backward compatibility rules |
| 52 | Kafka Streams intro | 03-apache-kafka/8-kafka-streams.md | Word count example |
| 53 | Multi-DC overview | 11-advanced/2-multi-datacenter.md | Active-passive diagram |
| 54 | Serverless processing | 11-advanced/4-serverless-processing.md | Lambda + SQS pattern |
| 55–56 | **Checkpoint 3** | 1-system-design-scenarios.md | Thiết kế notification system 45 phút |

**Milestone Giai Đoạn 3:** Thiết kế Outbox + Saga; implement DLQ + retry; 1 STAR story draft.

---

## Giai Đoạn 4: Production Skills (Ngày 57–77)

### Tuần 9: Monitoring & Observability

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 57 | Key metrics | 09-monitoring/1-key-metrics.md | List golden signals |
| 58 | Lag monitoring | 09-monitoring/2-lag-monitoring.md | Design alert thresholds |
| 59 | Alerting strategy | 09-monitoring/3-alerting-strategy.md | Write runbook snippet |
| 60 | Prometheus & Grafana | 09-monitoring/4-prometheus-grafana.md | Setup Kafka exporter |
| 61 | Distributed tracing | 09-monitoring/5-distributed-tracing.md | Correlation ID propagation |
| 62 | Troubleshooting playbook | 09-monitoring/6-troubleshooting-playbook.md | Walk through lag scenario |
| 63 | Production checklist | 09-monitoring/7-production-checklist.md | Review checklist |

### Tuần 10: Security & Performance

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 64 | Authentication | 08-security/1-authentication.md | SASL/SCRAM concepts |
| 65 | ACL & authorization | 08-security/2-authorization-acl.md | Topic-level permissions |
| 66 | Encryption TLS | 08-security/3-encryption.md | TLS in-transit diagram |
| 67 | Throughput tuning | 07-performance-scaling/1-throughput-tuning.md | Benchmark producer |
| 68 | Broker sizing | 07-performance-scaling/5-broker-sizing.md | Capacity estimate exercise |
| 69 | Backpressure handling | 07-performance-scaling/4-backpressure-handling.md | Rate limit design |
| 70 | **Review ops** | INTERVIEW_GUIDE Câu 28–30 | STAR story draft #2 |

### Tuần 11: Cloud Managed

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 71 | AWS MSK | 10-cloud-managed/1-aws-msk.md | So sánh Serverless vs Provisioned |
| 72 | Confluent Cloud | 10-cloud-managed/2-confluent-cloud.md | Schema Registry hosted |
| 73 | Azure Event Hubs | 10-cloud-managed/3-azure-event-hubs.md | Kafka endpoint notes |
| 74 | GCP Pub/Sub | 10-cloud-managed/4-gcp-pubsub.md | Push vs pull subscriptions |
| 75–77 | **Checkpoint 4** | 1-system-design-scenarios.md | Thiết kế CDC pipeline 45 phút |

**Milestone Giai Đoạn 4:** Setup monitoring mental model; security checklist; cloud broker comparison.

---

## Giai Đoạn 5: Interview Prep (Ngày 78–90)

### Tuần 12: Intensive Review

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 78 | INTERVIEW_GUIDE Phần 1–2 | INTERVIEW_GUIDE.md | Trả lời 14 câu aloud |
| 79 | INTERVIEW_GUIDE Phần 3–4 | INTERVIEW_GUIDE.md | Trả lời 10 câu Kafka/RabbitMQ |
| 80 | INTERVIEW_GUIDE Phần 5 | INTERVIEW_GUIDE.md | Trả lời 6 câu ops |
| 81 | Kafka deep questions | 2-kafka-questions.md | 20 câu review |
| 82 | RabbitMQ deep questions | 3-rabbitmq-questions.md | 15 câu review |
| 83 | System design scenarios | 1-system-design-scenarios.md | Order processing 45 phút |
| 84 | STAR stories finalize | 4-star-stories.md | Practice 3 stories |

### Tuần 13: Mock Interviews

| Ngày | Nội Dung | Thực Hành |
| ---- | -------- | --------- |
| 85 | Mock interview #1 — fundamentals | 30 phút với đồng nghiệp |
| 86 | Mock interview #2 — system design | 45 phút notification system |
| 87 | Mock interview #3 — Kafka deep dive | 30 phút technical |
| 88 | Review weak areas | Focus gaps từ mock |
| 89 | Final STAR practice | 3 stories × 3 phút |
| 90 | **Sẵn sàng phỏng vấn** | Checklist hoàn thành |

**Milestone Giai Đoạn 5:** Tự tin trả lời top 30 câu; 3 STAR stories; 2 system design scenarios.

---

## Portfolio Project

### Đề Xuất: Order Event Pipeline

```
Components:
├── Order API (REST) + PostgreSQL + Outbox table
├── Outbox Relay → Kafka topic "orders" (partition by orderId)
├── Consumers:
│   ├── Inventory Service (reserve stock)
│   ├── Payment Service (charge — idempotent)
│   └── Notification Service (email via queue)
├── DLQ topic/queue per consumer
└── Monitoring: lag dashboard, DLQ alerts

Skills demonstrated:
✓ Outbox Pattern
✓ At-least-once + idempotency
✓ Partition key design
✓ DLQ + retry
✓ Consumer lag monitoring
```

**Thời gian:** 2–3 tuần (song song Giai Đoạn 3–4)

---

## Theo Dõi Tiến Độ

Copy checklist và đánh dấu:

```markdown
## Message Queue 90-Day Progress

### Giai Đoạn 1 (Ngày 1–14)
- [ ] Docker lab running
- [ ] Delivery semantics mastered
- [ ] Broker selection understood
- [ ] Checkpoint 1 passed

### Giai Đoạn 2 (Ngày 15–35)
- [ ] Kafka producer/consumer
- [ ] RabbitMQ exchanges + DLQ
- [ ] Order pipeline lab
- [ ] Checkpoint 2 passed

### Giai Đoạn 3 (Ngày 36–56)
- [ ] Outbox Pattern understood
- [ ] Saga + idempotency
- [ ] Failure testing done
- [ ] Checkpoint 3 passed

### Giai Đoạn 4 (Ngày 57–77)
- [ ] Monitoring golden signals
- [ ] Security basics
- [ ] Cloud brokers compared
- [ ] Checkpoint 4 passed

### Giai Đoạn 5 (Ngày 78–90)
- [ ] Top 30 questions confident
- [ ] 3 STAR stories ready
- [ ] 2 system designs practiced
- [ ] 3 mock interviews done
```

---

**Cập Nhật Lần Cuối:** 2026-07-03
