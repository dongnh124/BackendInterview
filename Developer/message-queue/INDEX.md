# Message Queue & Event Broker — Chỉ Mục Toàn Diện

> Bản đồ điều hướng toàn bộ kiến thức về Message Queue (Hàng Đợi Tin Nhắn), Event Broker (Broker Sự Kiện) và Event-Driven Architecture (Kiến Trúc Hướng Sự Kiện)

## 📁 Cấu Trúc Thư Mục

```
Developer/message-queue/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/                            Nền tảng messaging
│   ├── README.md                               Tổng quan chủ đề nền tảng
│   ├── 1-message-queue-basics.md                 MQ vs Event Broker, use cases, terminology
│   ├── 2-pub-sub-vs-point-to-point.md            Publish/Subscribe vs Point-to-Point patterns
│   ├── 3-delivery-guarantees.md                  At-most-once, At-least-once, Exactly-once
│   ├── 4-ordering-and-sequencing.md              Message ordering, partition keys, FIFO
│   ├── 5-backpressure-flow-control.md            Backpressure, rate limiting, flow control
│   └── 6-broker-selection-guide.md               Decision tree chọn broker phù hợp
│
├── 02-architecture-patterns/                   Mẫu kiến trúc hướng sự kiện
│   ├── README.md                               Tổng quan EDA patterns
│   ├── 1-event-driven-architecture.md            EDA fundamentals, event types, boundaries
│   ├── 2-cqrs-event-sourcing.md                  CQRS, Event Sourcing, projections
│   ├── 3-saga-pattern.md                         Choreography vs Orchestration, compensation
│   ├── 4-outbox-inbox-pattern.md                 Transactional messaging, dual-write problem
│   └── 5-idempotency-dedup.md                    Idempotent consumers, deduplication keys
│
├── 03-apache-kafka/                            Apache Kafka chuyên sâu
│   ├── README.md                               Tổng quan Kafka ecosystem
│   ├── 1-kafka-architecture.md                   Broker, cluster, ZooKeeper/KRaft, ISR
│   ├── 2-topics-partitions.md                    Topic design, partitioning, key routing
│   ├── 3-consumer-groups.md                      Consumer groups, rebalancing, scale-out
│   ├── 4-producers-serialization.md              Producer config, acks, batching, compression
│   ├── 5-offset-management.md                    Auto vs manual commit, offset reset
│   ├── 6-schema-registry.md                      Avro, Protobuf, schema evolution
│   ├── 7-kafka-connect.md                        Source/sink connectors, CDC integration
│   └── 8-kafka-streams.md                        Stream processing, state stores, windowing
│
├── 04-rabbitmq/                                RabbitMQ chuyên sâu
│   ├── README.md                               Tổng quan RabbitMQ & AMQP
│   ├── 1-exchanges-queues-bindings.md            Direct, Fanout, Topic, Headers exchanges
│   ├── 2-routing-patterns.md                     Work queue, pub/sub, routing, RPC pattern
│   ├── 3-dead-letter-queue.md                    DLX, DLQ design, poison message handling
│   ├── 4-publisher-confirms.md                   Publisher confirms, consumer ack modes
│   └── 5-clustering-ha.md                        Mirrored queues, quorum queues, HA setup
│
├── 05-other-brokers/                           Các broker khác
│   ├── README.md                               So sánh & lựa chọn broker
│   ├── 1-amazon-sqs-sns.md                       SQS FIFO, SNS fan-out, visibility timeout
│   ├── 2-redis-streams.md                        Redis Pub/Sub vs Streams, consumer groups
│   ├── 3-nats-jetstream.md                       NATS core vs JetStream, persistence
│   └── 4-apache-pulsar.md                        Multi-tenancy, geo-replication, functions
│
├── 06-reliability/                             Độ tin cậy & xử lý lỗi
│   ├── README.md                               Tổng quan reliability patterns
│   ├── 1-at-least-once-exactly-once.md           Delivery semantics thực tế, trade-offs
│   ├── 2-retry-backoff.md                        Retry strategy, exponential backoff, jitter
│   ├── 3-dead-letter-handling.md                 DLQ operations, replay, monitoring
│   ├── 4-poison-message.md                       Detection, quarantine, root cause analysis
│   └── 5-circuit-breaker-consumers.md            Circuit breaker cho message consumers
│
├── 07-performance-scaling/                     Hiệu năng & mở rộng
│   ├── README.md                               Tổng quan performance tuning
│   ├── 1-throughput-tuning.md                    Batch size, linger, compression, pipelining
│   ├── 2-partitioning-strategies.md              Key design, hot partition, rebalancing
│   ├── 3-consumer-scaling.md                     Parallelism, partition-to-consumer ratio
│   ├── 4-backpressure-handling.md                Rate limiting, throttling, queue depth
│   └── 5-broker-sizing.md                        Capacity planning, disk, network, memory
│
├── 08-security/                                Bảo mật messaging
│   ├── README.md                               Tổng quan security cho message brokers
│   ├── 1-authentication.md                       SASL/SCRAM, mTLS, API keys
│   ├── 2-authorization-acl.md                    ACL, RBAC, topic-level permissions
│   ├── 3-encryption.md                           TLS in-transit, encryption at-rest
│   └── 4-audit-logging.md                        Audit trails, compliance requirements
│
├── 09-monitoring/                              Giám sát & observability
│   ├── README.md                               Tổng quan monitoring messaging layer
│   ├── 1-key-metrics.md                          Lag, throughput, error rate, rebalance
│   ├── 2-lag-monitoring.md                       Consumer lag, alerting thresholds
│   ├── 3-alerting-strategy.md                    SLOs, alert fatigue, runbook integration
│   ├── 4-prometheus-grafana.md                   Metrics setup cho Kafka/RabbitMQ
│   ├── 5-distributed-tracing.md                  OpenTelemetry, correlation ID, trace context
│   ├── 6-troubleshooting-playbook.md             Common issues & diagnosis steps
│   └── 7-production-checklist.md                Pre-deployment & go-live checklist
│
├── 10-cloud-managed/                           Dịch vụ messaging trên cloud
│   ├── README.md                               So sánh cloud managed services
│   ├── 1-aws-msk.md                              Amazon MSK, MSK Connect, Serverless
│   ├── 2-confluent-cloud.md                      Confluent Cloud, Schema Registry, ksqlDB
│   ├── 3-azure-event-hubs.md                     Event Hubs, Kafka endpoint, capture
│   └── 4-gcp-pubsub.md                           Pub/Sub, push/pull, ordering keys
│
├── 11-advanced/                                Chủ đề nâng cao
│   ├── README.md                               Tổng quan advanced topics
│   ├── 1-change-data-capture.md                  Debezium, CDC patterns, event-driven sync
│   ├── 2-multi-datacenter.md                     Geo-replication, active-active, conflict
│   ├── 3-schema-evolution.md                     Backward/forward compatibility, versioning
│   └── 4-serverless-processing.md                Lambda, Cloud Functions, event triggers
│
├── 12-interview-prep/                          Chuẩn bị phỏng vấn
│   ├── README.md                               Tổng quan ôn thi messaging
│   ├── INTERVIEW_GUIDE.md                      Top 30 câu hỏi + đáp án chi tiết
│   ├── 1-system-design-scenarios.md              Order processing, notification, CDC pipeline
│   ├── 2-kafka-questions.md                      Câu hỏi sâu về Kafka
│   ├── 3-rabbitmq-questions.md                   Câu hỏi sâu về RabbitMQ
│   ├── 4-star-stories.md                         Template câu chuyện incident STAR
│   └── 5-90-day-study-plan.md                    Kế hoạch học 90 ngày
├── GLOSSARY.md
```

---

## ✅ Nội Dung Đã Tạo

| Chủ Đề | File | Trạng Thái | Chất Lượng |
| ------ | ---- | ---------- | ---------- |
| **Tổng Quan & Lộ Trình** | README.md | ✅ | Toàn diện |
| **Chỉ Mục** | INDEX.md | ✅ | Toàn diện |
| **01-fundamentals/** | README + 6 bài | ✅ | Toàn diện |
| 01-fundamentals/README.md | Tổng quan chủ đề nền tảng | ✅ | Toàn diện |
| 01-fundamentals/1-message-queue-basics.md | MQ vs Event Broker, terminology | ✅ | Toàn diện |
| 01-fundamentals/2-pub-sub-vs-point-to-point.md | Pub/Sub vs Point-to-Point | ✅ | Toàn diện |
| 01-fundamentals/3-delivery-guarantees.md | At-most/least/exactly-once | ✅ | Toàn diện |
| 01-fundamentals/4-ordering-and-sequencing.md | Ordering, partition key, FIFO | ✅ | Toàn diện |
| 01-fundamentals/5-backpressure-flow-control.md | Backpressure, flow control | ✅ | Toàn diện |
| 01-fundamentals/6-broker-selection-guide.md | Decision tree chọn broker | ✅ | Toàn diện |
| **02-architecture-patterns/** | README + 5 bài | ✅ | Toàn diện |
| 02-architecture-patterns/README.md | Tổng quan EDA patterns | ✅ | Toàn diện |
| 02-architecture-patterns/1-event-driven-architecture.md | EDA fundamentals, event types, boundaries | ✅ | Toàn diện |
| 02-architecture-patterns/2-cqrs-event-sourcing.md | CQRS, Event Sourcing, projections | ✅ | Toàn diện |
| 02-architecture-patterns/3-saga-pattern.md | Choreography vs Orchestration, compensation | ✅ | Toàn diện |
| 02-architecture-patterns/4-outbox-inbox-pattern.md | Transactional messaging, dual-write | ✅ | Toàn diện |
| 02-architecture-patterns/5-idempotency-dedup.md | Idempotent consumers, dedup keys | ✅ | Toàn diện |
| **03-apache-kafka/** | README + 8 bài | ✅ | Toàn diện |
| 03-apache-kafka/README.md | Tổng quan Kafka ecosystem | ✅ | Toàn diện |
| 03-apache-kafka/1-kafka-architecture.md | Broker, cluster, KRaft, ISR | ✅ | Toàn diện |
| 03-apache-kafka/2-topics-partitions.md | Topic design, partitioning, key routing | ✅ | Toàn diện |
| 03-apache-kafka/3-consumer-groups.md | Consumer groups, rebalancing, scale-out | ✅ | Toàn diện |
| 03-apache-kafka/4-producers-serialization.md | Producer config, acks, batching, compression | ✅ | Toàn diện |
| 03-apache-kafka/5-offset-management.md | Auto vs manual commit, offset reset | ✅ | Toàn diện |
| 03-apache-kafka/6-schema-registry.md | Avro, Protobuf, schema evolution | ✅ | Toàn diện |
| 03-apache-kafka/7-kafka-connect.md | Source/sink connectors, CDC integration | ✅ | Toàn diện |
| 03-apache-kafka/8-kafka-streams.md | Stream processing, state stores, windowing | ✅ | Toàn diện |
| **04-rabbitmq/** | README + 5 bài | ✅ | Toàn diện |
| 04-rabbitmq/README.md | Tổng quan RabbitMQ & AMQP | ✅ | Toàn diện |
| 04-rabbitmq/1-exchanges-queues-bindings.md | Direct, Fanout, Topic, Headers exchanges | ✅ | Toàn diện |
| 04-rabbitmq/2-routing-patterns.md | Work queue, pub/sub, routing, RPC | ✅ | Toàn diện |
| 04-rabbitmq/3-dead-letter-queue.md | DLX, DLQ design, poison message | ✅ | Toàn diện |
| 04-rabbitmq/4-publisher-confirms.md | Publisher confirms, consumer ack modes | ✅ | Toàn diện |
| 04-rabbitmq/5-clustering-ha.md | Mirrored queues, quorum queues, HA | ✅ | Toàn diện |
| **05-other-brokers/** | README + 4 bài | ✅ | Toàn diện |
| 05-other-brokers/README.md | So sánh & lựa chọn broker | ✅ | Toàn diện |
| 05-other-brokers/1-amazon-sqs-sns.md | SQS FIFO, SNS fan-out, visibility timeout | ✅ | Toàn diện |
| 05-other-brokers/2-redis-streams.md | Pub/Sub vs Streams, consumer groups | ✅ | Toàn diện |
| 05-other-brokers/3-nats-jetstream.md | NATS Core vs JetStream, persistence | ✅ | Toàn diện |
| 05-other-brokers/4-apache-pulsar.md | Multi-tenancy, geo-replication, functions | ✅ | Toàn diện |
| **06-reliability/** | README + 5 bài | ✅ | Toàn diện |
| 06-reliability/README.md | Tổng quan reliability patterns | ✅ | Toàn diện |
| 06-reliability/1-at-least-once-exactly-once.md | Delivery semantics thực tế, trade-offs | ✅ | Toàn diện |
| 06-reliability/2-retry-backoff.md | Retry strategy, exponential backoff, jitter | ✅ | Toàn diện |
| 06-reliability/3-dead-letter-handling.md | DLQ operations, replay, monitoring | ✅ | Toàn diện |
| 06-reliability/4-poison-message.md | Detection, quarantine, root cause analysis | ✅ | Toàn diện |
| 06-reliability/5-circuit-breaker-consumers.md | Circuit breaker cho message consumers | ✅ | Toàn diện |
| **07-performance-scaling/** | README + 5 bài | ✅ | Toàn diện |
| 07-performance-scaling/README.md | Tổng quan performance tuning | ✅ | Toàn diện |
| 07-performance-scaling/1-throughput-tuning.md | Batch size, linger, compression, pipelining | ✅ | Toàn diện |
| 07-performance-scaling/2-partitioning-strategies.md | Key design, hot partition, rebalancing | ✅ | Toàn diện |
| 07-performance-scaling/3-consumer-scaling.md | Parallelism, partition-to-consumer ratio, HPA | ✅ | Toàn diện |
| 07-performance-scaling/4-backpressure-handling.md | Rate limiting, throttling, queue depth | ✅ | Toàn diện |
| 07-performance-scaling/5-broker-sizing.md | Capacity planning, disk, network, memory | ✅ | Toàn diện |
| **08-security/** | README + 4 bài | ✅ | Toàn diện |
| 08-security/README.md | Tổng quan security cho message brokers | ✅ | Toàn diện |
| 08-security/1-authentication.md | SASL/SCRAM, mTLS, API keys | ✅ | Toàn diện |
| 08-security/2-authorization-acl.md | ACL, RBAC, topic-level permissions | ✅ | Toàn diện |
| 08-security/3-encryption.md | TLS in-transit, encryption at-rest | ✅ | Toàn diện |
| 08-security/4-audit-logging.md | Audit trails, compliance requirements | ✅ | Toàn diện |
| **09-monitoring/** | README + 7 bài | ✅ | Toàn diện |
| 09-monitoring/README.md | Tổng quan monitoring messaging layer | ✅ | Toàn diện |
| 09-monitoring/1-key-metrics.md | Lag, throughput, error rate, rebalance | ✅ | Toàn diện |
| 09-monitoring/2-lag-monitoring.md | Consumer lag, alerting thresholds | ✅ | Toàn diện |
| 09-monitoring/3-alerting-strategy.md | SLOs, alert fatigue, runbook integration | ✅ | Toàn diện |
| 09-monitoring/4-prometheus-grafana.md | Metrics setup cho Kafka/RabbitMQ | ✅ | Toàn diện |
| 09-monitoring/5-distributed-tracing.md | OpenTelemetry, correlation ID, trace context | ✅ | Toàn diện |
| 09-monitoring/6-troubleshooting-playbook.md | Common issues & diagnosis steps | ✅ | Toàn diện |
| 09-monitoring/7-production-checklist.md | Pre-deployment & go-live checklist | ✅ | Toàn diện |
| **10-cloud-managed/** | README + 4 bài | ✅ | Toàn diện |
| 10-cloud-managed/README.md | So sánh cloud managed services | ✅ | Toàn diện |
| 10-cloud-managed/1-aws-msk.md | MSK Provisioned, Serverless, MSK Connect | ✅ | Toàn diện |
| 10-cloud-managed/2-confluent-cloud.md | Confluent Cloud, Schema Registry, ksqlDB | ✅ | Toàn diện |
| 10-cloud-managed/3-azure-event-hubs.md | Event Hubs, Kafka endpoint, Capture | ✅ | Toàn diện |
| 10-cloud-managed/4-gcp-pubsub.md | Pub/Sub push/pull, ordering keys | ✅ | Toàn diện |
| **11-advanced/** | README + 4 bài | ✅ | Toàn diện |
| 11-advanced/README.md | Tổng quan advanced topics | ✅ | Toàn diện |
| 11-advanced/1-change-data-capture.md | Debezium, CDC patterns, event-driven sync | ✅ | Toàn diện |
| 11-advanced/2-multi-datacenter.md | Geo-replication, active-active, conflict | ✅ | Toàn diện |
| 11-advanced/3-schema-evolution.md | Backward/forward compatibility, governance | ✅ | Toàn diện |
| 11-advanced/4-serverless-processing.md | Lambda, Cloud Functions, event triggers | ✅ | Toàn diện |
| **12-interview-prep/** | README + 6 bài | ✅ | Toàn diện |
| 12-interview-prep/README.md | Tổng quan ôn thi messaging | ✅ | Toàn diện |
| 12-interview-prep/INTERVIEW_GUIDE.md | Top 30 câu hỏi + đáp án chi tiết | ✅ | Toàn diện |
| 12-interview-prep/1-system-design-scenarios.md | Order processing, notification, CDC | ✅ | Toàn diện |
| 12-interview-prep/2-kafka-questions.md | 20 câu hỏi sâu Kafka | ✅ | Toàn diện |
| 12-interview-prep/3-rabbitmq-questions.md | 15 câu hỏi sâu RabbitMQ | ✅ | Toàn diện |
| 12-interview-prep/4-star-stories.md | Template incident STAR | ✅ | Toàn diện |
| 12-interview-prep/5-90-day-study-plan.md | Kế hoạch học 90 ngày | ✅ | Toàn diện |
| **Từ Điển Thuật Ngữ** | GLOSSARY.md | ✅ | Toàn diện |

---

## 🎯 Cần Tạo Tiếp (Thứ Tự Ưu Tiên)

### Ưu Tiên Cao (Kỹ năng cốt lõi)

- [x] 01-fundamentals/README.md — Nền tảng messaging, terminology
- [x] 01-fundamentals/3-delivery-guarantees.md — At-least-once, Exactly-once (luôn được hỏi phỏng vấn)
- [x] 01-fundamentals/1-message-queue-basics.md — MQ vs Event Broker
- [x] 01-fundamentals/2-pub-sub-vs-point-to-point.md — Pub/Sub vs Point-to-Point
- [x] 01-fundamentals/4-ordering-and-sequencing.md — Message ordering, FIFO
- [x] 01-fundamentals/5-backpressure-flow-control.md — Backpressure, flow control
- [x] 01-fundamentals/6-broker-selection-guide.md — Decision tree chọn broker
- [x] 02-architecture-patterns/README.md — Tổng quan EDA patterns
- [x] 02-architecture-patterns/1-event-driven-architecture.md — EDA fundamentals
- [x] 02-architecture-patterns/4-outbox-inbox-pattern.md — Transactional messaging
- [x] 02-architecture-patterns/3-saga-pattern.md — Distributed transactions
- [x] 02-architecture-patterns/5-idempotency-dedup.md — Idempotent consumers
- [x] 02-architecture-patterns/2-cqrs-event-sourcing.md — CQRS & Event Sourcing
- [x] 03-apache-kafka/README.md — Tổng quan Kafka ecosystem
- [x] 03-apache-kafka/1-kafka-architecture.md — Broker, cluster, KRaft, ISR
- [x] 03-apache-kafka/2-topics-partitions.md — Topic design, partitioning
- [x] 03-apache-kafka/3-consumer-groups.md — Consumer groups, scaling
- [x] 03-apache-kafka/4-producers-serialization.md — Producer config, acks, batching
- [x] 03-apache-kafka/5-offset-management.md — Offset commit, reset, replay
- [x] 03-apache-kafka/6-schema-registry.md — Avro, Protobuf, schema evolution
- [x] 03-apache-kafka/7-kafka-connect.md — Source/sink connectors, CDC
- [x] 03-apache-kafka/8-kafka-streams.md — Stream processing, windowing
- [x] 04-rabbitmq/README.md — Tổng quan RabbitMQ
- [x] 04-rabbitmq/1-exchanges-queues-bindings.md — Exchanges, queues, bindings
- [x] 04-rabbitmq/2-routing-patterns.md — Work queue, pub/sub, RPC
- [x] 04-rabbitmq/3-dead-letter-queue.md — DLX, DLQ, poison message
- [x] 04-rabbitmq/4-publisher-confirms.md — Publisher confirms, consumer ack
- [x] 04-rabbitmq/5-clustering-ha.md — Quorum queues, clustering, HA
- [x] 06-reliability/README.md — DLQ, retry, poison message
- [x] 06-reliability/1-at-least-once-exactly-once.md — Delivery semantics thực tế
- [x] 06-reliability/2-retry-backoff.md — Retry, exponential backoff, jitter
- [x] 06-reliability/3-dead-letter-handling.md — DLQ operations, replay
- [x] 06-reliability/4-poison-message.md — Poison message detection, quarantine
- [x] 06-reliability/5-circuit-breaker-consumers.md — Circuit breaker cho consumers
- [x] 12-interview-prep/INTERVIEW_GUIDE.md — Top 30 câu hỏi phỏng vấn
- [x] 12-interview-prep/README.md — Tổng quan ôn thi messaging
- [x] 12-interview-prep/1-system-design-scenarios.md — Bài toán thiết kế
- [x] 12-interview-prep/2-kafka-questions.md — Câu hỏi sâu Kafka
- [x] 12-interview-prep/3-rabbitmq-questions.md — Câu hỏi sâu RabbitMQ
- [x] 12-interview-prep/4-star-stories.md — Template STAR incident
- [x] 12-interview-prep/5-90-day-study-plan.md — Kế hoạch học 90 ngày
- [ ] ROADMAP.md — Lộ trình học 90 ngày chi tiết

### Ưu Tiên Trung Bình (Kỹ năng nâng cao)

- [x] 07-performance-scaling/README.md — Throughput tuning
- [x] 07-performance-scaling/1-throughput-tuning.md — Batch size, linger, compression
- [x] 07-performance-scaling/2-partitioning-strategies.md — Key design, hot partition
- [x] 07-performance-scaling/3-consumer-scaling.md — Parallelism, HPA
- [x] 07-performance-scaling/4-backpressure-handling.md — Rate limiting, throttling
- [x] 07-performance-scaling/5-broker-sizing.md — Capacity planning
- [x] 08-security/README.md — Authentication, ACL, TLS, audit
- [x] 08-security/1-authentication.md — SASL/SCRAM, mTLS, API keys
- [x] 08-security/2-authorization-acl.md — ACL, RBAC, least privilege
- [x] 08-security/3-encryption.md — TLS in-transit, encryption at-rest
- [x] 08-security/4-audit-logging.md — Audit trails, compliance
- [x] 09-monitoring/README.md — Lag monitoring, alerting
- [x] 09-monitoring/1-key-metrics.md — Golden signals, broker/producer/consumer metrics
- [x] 09-monitoring/2-lag-monitoring.md — Consumer lag, threshold design
- [x] 09-monitoring/3-alerting-strategy.md — SLOs, alert fatigue, runbook
- [x] 09-monitoring/4-prometheus-grafana.md — Prometheus & Grafana setup
- [x] 09-monitoring/5-distributed-tracing.md — OpenTelemetry, correlation ID
- [x] 09-monitoring/6-troubleshooting-playbook.md — Incident diagnosis steps
- [x] 09-monitoring/7-production-checklist.md — Pre-deployment checklist
- [x] 10-cloud-managed/README.md — So sánh cloud managed services
- [x] 10-cloud-managed/1-aws-msk.md — AWS MSK guide
- [x] 10-cloud-managed/2-confluent-cloud.md — Confluent Cloud, ksqlDB
- [x] 10-cloud-managed/3-azure-event-hubs.md — Event Hubs, Capture
- [x] 10-cloud-managed/4-gcp-pubsub.md — GCP Pub/Sub push/pull

### Ưu Tiên Thấp (Tham khảo)

- [x] 05-other-brokers/README.md — So sánh & lựa chọn broker
- [x] 05-other-brokers/1-amazon-sqs-sns.md — SQS FIFO, SNS fan-out
- [x] 05-other-brokers/2-redis-streams.md — Redis Pub/Sub vs Streams
- [x] 05-other-brokers/3-nats-jetstream.md — NATS Core vs JetStream
- [x] 05-other-brokers/4-apache-pulsar.md — Multi-tenancy, geo-replication
- [x] 11-advanced/README.md — Tổng quan advanced topics
- [x] 11-advanced/1-change-data-capture.md — Debezium, CDC patterns
- [x] 11-advanced/2-multi-datacenter.md — Geo-replication, active-active
- [x] 11-advanced/3-schema-evolution.md — Schema governance, breaking changes
- [x] 11-advanced/4-serverless-processing.md — Lambda, Cloud Functions
- [x] 12-interview-prep/1-system-design-scenarios.md — Bài toán thiết kế
- [x] GLOSSARY.md — Thuật ngữ
- [ ] RESOURCES.md — Tài liệu học thêm

---

## 🚀 Cách Sử Dụng Knowledge Base

### Tự Học

```
1. Bắt đầu với README.md
2. Chọn lộ trình (Beginner/Intermediate/Advanced)
3. Học tuần tự từng section
4. Thực hành với Docker lab (Kafka + RabbitMQ)
5. Xây dựng project portfolio (order/event pipeline)
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md
2. Học sâu broker mục tiêu (Kafka hoặc RabbitMQ)
3. Ôn 01-fundamentals/delivery-guarantees.md (luôn được hỏi)
4. Ôn 02-architecture-patterns/ (Outbox, Saga — hay hỏi system design)
5. Chuẩn bị câu chuyện incident từ kinh nghiệm thực tế
6. Luyện mock interview với đồng nghiệp
```

### Trên Công Việc

```
Dùng làm tài liệu tham khảo:
- Tích hợp mới: 03-apache-kafka/ hoặc 04-rabbitmq/
- Xử lý lỗi: 06-reliability/ (DLQ, retry, poison message)
- Go-live: 09-monitoring/7-production-checklist.md
- Scale: 07-performance-scaling/ (partitioning, consumer scaling)
- Bảo mật: 08-security/ (ACL, TLS, authentication)
```

### Thiết Kế Hệ Thống

```
1. Đọc 01-fundamentals/broker-selection-guide.md
2. Áp dụng 02-architecture-patterns/ cho EDA design
3. Thiết kế reliability với 06-reliability/
4. Lập kế hoạch monitoring từ 09-monitoring/
5. So sánh cloud options tại 10-cloud-managed/
```

---

## 📊 Ước Tính Thời Gian Học

| Phần | Thời Gian | Độ Khó | Ưu Tiên |
| ---- | --------- | ------ | ------- |
| Fundamentals (Nền Tảng) | 4–6 giờ | ⭐ | Bắt buộc |
| Architecture Patterns (Mẫu Kiến Trúc) | 6–8 giờ | ⭐⭐ | Bắt buộc |
| Apache Kafka | 12–16 giờ | ⭐⭐⭐ | Bắt buộc |
| RabbitMQ | 8–10 giờ | ⭐⭐ | Bắt buộc |
| Reliability (Độ Tin Cậy) | 4–6 giờ | ⭐⭐ | Bắt buộc |
| Performance & Scaling | 6–8 giờ | ⭐⭐⭐ | Nên học |
| Security (Bảo Mật) | 4–6 giờ | ⭐⭐ | Nên học |
| Monitoring | 4–6 giờ | ⭐⭐ | Nên học |
| Cloud Managed | 6–8 giờ | ⭐⭐ | Tùy chọn |
| Advanced Topics | 10–15 giờ | ⭐⭐⭐ | Tùy chọn |

**Tổng: 65–90 giờ cho kiến thức messaging toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Beginner (0–1 năm kinh nghiệm)

- [ ] Message Queue vs Event Broker
- [ ] Point-to-Point vs Pub/Sub
- [ ] At-least-once delivery
- [ ] Cài đặt Kafka/RabbitMQ local
- [ ] Producer/Consumer cơ bản

**Thời gian nắm vững:** 1–2 tháng

### Intermediate (1–3 năm kinh nghiệm)

- [ ] Topic/Queue design
- [ ] Consumer Groups & scaling
- [ ] DLQ & retry strategy
- [ ] Outbox Pattern
- [ ] Consumer lag troubleshooting

**Thời gian nắm vững:** 2–3 tháng để đi sâu

### Advanced (3–5+ năm kinh nghiệm)

- [ ] Event-Driven Architecture design
- [ ] Saga Pattern implementation
- [ ] Kafka cluster tuning
- [ ] Exactly-once in production
- [ ] Multi-datacenter replication

**Thời gian nắm vững:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | Vị Trí |
| ------- | ------ |
| Tổng quan | [README.md](README.md) |
| Nền tảng messaging | [01-fundamentals/README.md](01-fundamentals/README.md) |
| Mẫu kiến trúc EDA | [02-architecture-patterns/README.md](02-architecture-patterns/README.md) |
| Kafka deep dive | [03-apache-kafka/README.md](03-apache-kafka/README.md) |
| RabbitMQ guide | [04-rabbitmq/README.md](04-rabbitmq/README.md) |
| Các broker khác | [05-other-brokers/README.md](05-other-brokers/README.md) |
| Độ tin cậy & DLQ | [06-reliability/README.md](06-reliability/README.md) |
| Performance & scaling | [07-performance-scaling/README.md](07-performance-scaling/README.md) |
| Security (ACL, TLS) | [08-security/README.md](08-security/README.md) |
| Monitoring & lag | [09-monitoring/README.md](09-monitoring/README.md) |
| Cloud managed (MSK, Pub/Sub) | [10-cloud-managed/README.md](10-cloud-managed/README.md) |
| Advanced (CDC, multi-DC) | [11-advanced/README.md](11-advanced/README.md) |
| Câu hỏi phỏng vấn | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md) |
| Từ điển thuật ngữ | [GLOSSARY.md](GLOSSARY.md) |

---

## 📈 Theo Dõi Tiến Độ Học

Tạo bản copy và theo dõi tiến độ:

```markdown
## Message Queue Knowledge Completion

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] Message Queue vs Event Broker
- [ ] Point-to-Point vs Pub/Sub
- [ ] Delivery guarantees (3/3)
- [ ] Message ordering
- [ ] Backpressure concept

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)

- [ ] Kafka topics & partitions
- [ ] Consumer groups & rebalancing
- [ ] RabbitMQ exchanges & routing
- [ ] DLQ design & implementation
- [ ] Outbox Pattern
- [ ] Idempotency & dedup

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)

- [ ] Saga Pattern
- [ ] CQRS & Event Sourcing basics
- [ ] Performance tuning
- [ ] Monitoring & alerting setup
- [ ] Security (ACL, TLS)

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Kafka Streams
- [ ] CDC với Debezium
- [ ] Cloud managed services
- [ ] System design scenarios
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích delivery semantics không cần nhìn tài liệu
- [ ] So sánh Kafka vs RabbitMQ cho use case cụ thể
- [ ] Thiết kế topic/queue structure hợp lý
- [ ] Hiểu trade-offs của event-driven vs sync

### ✅ Năng Lực Vận Hành

- [ ] Troubleshoot consumer lag systematically
- [ ] Implement DLQ + retry strategy
- [ ] Cấu hình monitoring & alerting
- [ ] Áp dụng Outbox Pattern trong production
- [ ] Respond to messaging incidents

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời top 30 câu hỏi messaging tự tin
- [ ] Kể 2–3 câu chuyện incident (STAR format)
- [ ] Thiết kế event-driven system cho bài toán cho trước
- [ ] Thảo luận trade-offs và constraints rõ ràng
- [ ] Nắm vững ít nhất một broker (Kafka hoặc RabbitMQ)

---

## 🚀 Bước Tiếp Theo

### Ngay (Tuần Này)

1. Đọc README.md kỹ lưỡng
2. Chọn lộ trình học phù hợp
3. Dựng Docker lab (Kafka + RabbitMQ)
4. Bắt đầu 01-fundamentals/

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành 01-fundamentals/
2. Học sâu 03-apache-kafka/ hoặc 04-rabbitmq/
3. Thực hành producer/consumer hands-on
4. Làm lab cho mỗi chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành core topics (01–06)
2. Setup monitoring cơ bản (09-monitoring/)
3. Chuẩn bị 2–3 câu chuyện incident
4. Mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Nắm vững một broker hoàn toàn
2. Hiểu trade-offs tất cả brokers chính
3. Xây dựng portfolio project (event pipeline)
4. Sẵn sàng phỏng vấn hoặc đảm nhận vai trò messaging

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng thực hành:** Không chỉ đọc — chạy Kafka/RabbitMQ local, gây lỗi rồi sửa
2. **Hiểu delivery semantics:** Đây là chủ đề được hỏi nhiều nhất — nắm thật chắc
3. **Xây mental model:** Hiểu TẠI SAO, không chỉ CÁI GÌ — trade-offs quan trọng hơn syntax
4. **Monitor consumer lag:** Thói quen hàng ngày khi làm việc với messaging
5. **Thiết kế idempotent:** Mọi consumer nên idempotent — đây là best practice số 1
6. **Test failure scenarios:** Kill consumer, restart broker, network partition — học từ lỗi
7. **Document incidents:** Post-mortem là cơ hội học tập tuyệt vời cho phỏng vấn

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống, hoan nghênh đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa có
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn cho khái niệm phức tạp
- [x] Hướng dẫn platform-specific (MSK, Confluent Cloud)

---

## 📄 Giấy Phép

Knowledge base này mở cho mục đích học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-07-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ README & INDEX hoàn thành | ✅ 01-fundamentals hoàn thành | ✅ 02-architecture-patterns hoàn thành | ✅ 03-apache-kafka hoàn thành | ✅ 04-rabbitmq hoàn thành | ✅ 05-other-brokers hoàn thành | ✅ 06-reliability hoàn thành | ✅ 07-performance-scaling hoàn thành | ✅ 08-security hoàn thành | ✅ 09-monitoring hoàn thành | ✅ 10-cloud-managed hoàn thành | ✅ 11-advanced hoàn thành | ✅ 12-interview-prep hoàn thành | ✅ GLOSSARY hoàn thành
