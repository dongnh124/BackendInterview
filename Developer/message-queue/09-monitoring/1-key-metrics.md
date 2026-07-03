# Key Metrics — Chỉ Số Quan Trọng Cho Messaging Layer

> Golden signals (tín hiệu vàng) và metrics (chỉ số) cần thu thập ở mọi tầng: Broker, Producer, Consumer, DLQ (Dead Letter Queue — Hàng Đợi Thư Chết), và Business layer.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Golden Signals Cho Messaging](#golden-signals-cho-messaging)
3. [Broker Metrics](#broker-metrics)
4. [Producer Metrics](#producer-metrics)
5. [Consumer Metrics](#consumer-metrics)
6. [DLQ & Error Metrics](#dlq--error-metrics)
7. [Business Metrics](#business-metrics)
8. [Metric Naming Convention](#metric-naming-convention)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│           MESSAGING KEY METRICS — QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│  CONSUMER LAG     → Metric #1 — độ trễ xử lý (messages/seconds) │
│  THROUGHPUT       → Messages in/out per second                  │
│  ERROR RATE       → Produce fail, consume fail, DLQ ingress     │
│  LATENCY          → End-to-end, processing time, commit latency │
│  SATURATION       → Disk, CPU, memory, queue depth, connections │
│  AVAILABILITY     → Active consumers, ISR (In-Sync Replicas)    │
└─────────────────────────────────────────────────────────────────┘
```

| Metric | Ý Nghĩa | Alert Khi |
| ------ | ------- | --------- |
| **Consumer Lag** | Số message chưa xử lý | Tăng liên tục hoặc > SLA threshold |
| **Throughput** | messages/sec produce/consume | Drop đột ngột > 50% |
| **Error Rate** | % message fail | > 0.1% sustained |
| **DLQ Depth** | Message trong DLQ | > 0 sustained hoặc tăng nhanh |
| **Disk Usage** | % disk broker đã dùng | > 80% warning, > 90% critical |
| **Rebalance Rate** | Tần suất consumer group rebalance | > 1/hour bất thường |

---

## Golden Signals Cho Messaging

Google SRE định nghĩa **four golden signals (bốn tín hiệu vàng)**: Latency (Độ Trễ), Traffic (Lưu Lượng), Errors (Lỗi), Saturation (Bão Hòa). Áp dụng cho messaging:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    GOLDEN SIGNALS — MESSAGING MAPPING                     │
│                                                                          │
│  LATENCY          TRAFFIC           ERRORS           SATURATION          │
│  ────────         ───────           ──────           ──────────          │
│  Processing time  Produce rate      Produce errors   Disk usage          │
│  Commit latency   Consume rate      Consume errors   CPU/Memory          │
│  End-to-end lag   Bytes/sec         DLQ ingress      Connection count    │
│  P50/P95/P99      Rebalance events  Retry count      Queue depth         │
└──────────────────────────────────────────────────────────────────────────┘
```

### Latency (Độ Trễ)

| Metric | Đo Ở Đâu | Ghi Chú |
| ------ | -------- | ------- |
| **End-to-end latency** | Producer timestamp → consumer processed | Cần clock sync hoặc correlation ID |
| **Processing latency** | Consumer nhận → xử lý xong | Metric quan trọng cho SLA |
| **Commit latency** | Xử lý xong → offset committed | Ảnh hưởng lag khi commit chậm |
| **Broker request latency** | Produce/fetch request time | P99 quan trọng hơn average |

### Traffic (Lưu Lượng)

| Metric | Kafka | RabbitMQ |
| ------ | ----- | -------- |
| **Ingress rate** | `kafka_server_BrokerTopicMetrics_MessagesInPerSec` | `rabbitmq_queue_messages_published_total` |
| **Egress rate** | `kafka_server_BrokerTopicMetrics_MessagesOutPerSec` | `rabbitmq_queue_messages_delivered_total` |
| **Bytes rate** | `BytesInPerSec`, `BytesOutPerSec` | `rabbitmq_queue_message_bytes` |

### Errors (Lỗi)

| Error Type | Metric | Nguyên Nhân Thường Gặp |
| ---------- | ------ | ---------------------- |
| **Produce error** | `record-error-rate` | Auth fail, topic not found, `NOT_ENOUGH_REPLICAS` |
| **Consume error** | Custom counter | Deserialization fail, business logic exception |
| **Commit error** | `commit-latency-avg` spike | Rebalance in progress, coordinator unavailable |
| **DLQ ingress** | Custom counter | Permanent failure sau max retries |

### Saturation (Bão Hòa)

| Resource | Kafka Metric | RabbitMQ Metric |
| -------- | ------------ | --------------- |
| **Disk** | `kafka_log_Log_Size` / disk total | `rabbitmq_disk_free` |
| **CPU** | Node exporter `node_cpu` | `rabbitmq_process_cpu_seconds_total` |
| **Memory** | `kafka_server_*_MemoryUsage` | `rabbitmq_process_resident_memory_bytes` |
| **Connections** | `connection-count` | `rabbitmq_connections` |
| **Queue depth** | N/A (log-based) | `rabbitmq_queue_messages` |

---

## Broker Metrics

### Kafka Broker — Metrics Quan Trọng

```yaml
# JMX / kafka_exporter metrics
kafka_server_ReplicaManager:
  UnderReplicatedPartitions: 0          # CRITICAL nếu > 0
  PartitionCount: per broker
  LeaderCount: per broker

kafka_log_Log:
  Size: bytes per partition             # Disk planning
  LogEndOffset: per partition

kafka_network_RequestMetrics:
  RequestsPerSec: produce, fetch, metadata
  TotalTimeMs: P99 request latency

kafka_controller_KafkaController:
  ActiveControllerCount: 1              # Phải = 1 trong cluster
  OfflinePartitionsCount: 0             # CRITICAL nếu > 0
```

| Metric | Ý Nghĩa | Threshold Gợi Ý |
| ------ | ------- | --------------- |
| **UnderReplicatedPartitions** | Replica chưa sync với leader | = 0 (bất kỳ > 0 cần investigate) |
| **OfflinePartitionsCount** | Partition không có leader | = 0 |
| **ActiveControllerCount** | Số broker đang là controller | = 1 |
| **RequestQueueSize** | Request đang chờ xử lý | < 100 (tùy cluster size) |
| **Log flush rate** | Tốc độ ghi disk | Monitor trend, không absolute threshold |

### RabbitMQ Broker — Metrics Quan Trọng

```yaml
# rabbitmq_exporter / built-in prometheus plugin
rabbitmq_queues:
  messages: queue depth                 # Tương đương "lag"
  messages_ready: chờ consumer
  messages_unacknowledged: đang xử lý

rabbitmq_connections: active connections
rabbitmq_channels: active channels
rabbitmq_consumers: consumers per queue

rabbitmq_node_mem_used / rabbitmq_node_mem_limit: memory alarm
rabbitmq_disk_free: disk space
```

| Metric | Ý Nghĩa | Alert |
| ------ | ------- | ----- |
| **messages_ready** | Message chờ consumer | Tăng liên tục = consumer chậm |
| **messages_unacknowledged** | Message đã deliver, chưa ack | Cao = consumer xử lý chậm hoặc stuck |
| **memory alarm** | Broker sắp block publishers | Critical — flow control sắp kick in |
| **disk alarm** | Disk sắp đầy | Critical — publishers bị block |

---

## Producer Metrics

### Kafka Producer

```java
// Micrometer / custom metrics từ producer client
kafka.producer.record-send-rate          // messages/sec
kafka.producer.record-error-rate         // errors/sec
kafka.producer.request-latency-avg       // ms
kafka.producer.batch-size-avg            // bytes
kafka.producer.compression-rate-avg      // ratio
kafka.producer.buffer-available-bytes    // buffer còn trống
```

| Metric | Healthy | Unhealthy |
| ------ | ------- | --------- |
| **record-error-rate** | ~0 | > 0 sustained |
| **buffer-available-bytes** | > 20% buffer | → 0 (backpressure) |
| **request-latency-avg** | Stable | Spike khi broker overload |
| **batch-size-avg** | Gần config `batch.size` | Quá nhỏ = không batching hiệu quả |

### RabbitMQ Publisher

```yaml
# Custom application metrics
messaging_publish_total{exchange, routing_key, status}
messaging_publish_duration_seconds{exchange}
messaging_publish_confirms_pending     # Nếu dùng publisher confirms
```

> **Best practice:** Instrument (Gắn Metric) producer ở application layer — broker metrics không cho biết **service nào** produce fail.

---

## Consumer Metrics

### Kafka Consumer — Metrics Cốt Lõi

```yaml
# Consumer lag — metric quan trọng nhất
kafka_consumergroup_lag{group, topic, partition}

# Từ consumer client
kafka.consumer.records-consumed-rate
kafka.consumer.records-lag-max           # Client-side lag estimate
kafka.consumer.commit-rate
kafka.consumer.commit-latency-avg
kafka.consumer.rebalance-rate-per-hour  # Custom — rebalance events
```

**Consumer Lag (Độ Trễ Consumer)** được tính:

```
lag = log_end_offset - committed_offset (per partition)

total_lag = sum(lag) across all partitions assigned to group
```

| Lag Pattern | Diagnosis |
| ----------- | --------- |
| Lag tăng đều, tất cả partition | Consumer chậm hoặc thiếu consumer |
| Lag cao 1 partition | Hot partition — key skew |
| Lag spike rồi giảm | Rebalance hoặc consumer restart |
| Lag = 0 nhưng business stale | Consumer commit nhưng không xử lý đúng (bug) |

### RabbitMQ Consumer

```yaml
rabbitmq_queue_messages_ready            # Queue depth
rabbitmq_queue_consumer_utilisation      # % thời gian consumer active
rabbitmq_queue_messages_unacknowledged   # In-flight messages

# Application metrics
messaging_consume_total{queue, status}
messaging_consume_duration_seconds{queue}
messaging_consume_errors_total{queue, error_type}
```

---

## DLQ & Error Metrics

```
┌─────────────────────────────────────────────────────────────────┐
│                    ERROR METRICS HIERARCHY                        │
│                                                                  │
│  Level 1: Transient (Tạm Thời)                                   │
│    → retry_count, retry_queue_depth                              │
│                                                                  │
│  Level 2: Permanent (Vĩnh Viễn)                                  │
│    → dlq_ingress_rate, dlq_depth, dlq_oldest_message_age         │
│                                                                  │
│  Level 3: System (Hệ Thống)                                      │
│    → circuit_breaker_state, consumer_down_count                  │
└─────────────────────────────────────────────────────────────────┘
```

| Metric | Mô Tả | Alert |
| ------ | ----- | ----- |
| **dlq_depth** | Tổng message trong DLQ | > 0 sustained (warning), > 100 (critical) |
| **dlq_ingress_rate** | Message vào DLQ/giây | > 0 sustained |
| **dlq_oldest_message_age** | Message cũ nhất trong DLQ | > 24h = cần replay hoặc investigate |
| **retry_count_histogram** | Phân bố số lần retry | P99 retry cao = transient issue |
| **circuit_breaker_open** | Circuit breaker state | = 1 → consumer paused |

---

## Business Metrics

Technical metrics cho biết **hệ thống** healthy; business metrics cho biết **nghiệp vụ** healthy:

| Business Metric | Ví Dụ | Liên Kết Technical |
| --------------- | ----- | ------------------ |
| **Orders processed/min** | 1,200 orders/min | `consume_rate` × success rate |
| **Payment events lag** | Max 30s behind | Consumer lag × avg processing time |
| **Notification delivery SLA** | 99% trong 1 phút | End-to-end latency P99 |
| **Stale events count** | Events > 1h chưa xử lý | Lag × message age |

```promql
# Ví dụ: Order processing SLA breach rate
sum(rate(orders_processed_total{status="success"}[5m]))
/
sum(rate(orders_received_total[5m]))
```

> **Tip phỏng vấn:** Luôn đề cập cả technical metrics (lag) VÀ business metrics (orders/min) — thể hiện tư duy production, không chỉ infrastructure.

---

## Metric Naming Convention

Tuân theo **Prometheus naming convention**:

```
# Format: <namespace>_<subsystem>_<name>_<unit>

messaging_producer_records_total{topic, service}
messaging_consumer_lag{group, topic, partition}
messaging_consumer_processing_duration_seconds{queue}
messaging_dlq_messages_total{source_queue}
```

**Labels (Nhãn) quan trọng:**

| Label | Dùng Cho |
| ----- | -------- |
| `service` | Tên microservice |
| `topic` / `queue` | Destination |
| `consumer_group` | Kafka consumer group |
| `partition` | Kafka partition (lag per partition) |
| `status` | success / error / retry |
| `error_type` | deserialization / timeout / business |

**Tránh high-cardinality labels:** Không dùng `message_id`, `user_id` làm label — explode metric count.

---

## Anti-Patterns

| Anti-Pattern | Vấn Đề | Giải Pháp |
| ------------ | ------ | --------- |
| Chỉ monitor broker, không monitor app | Không biết service nào fail | Instrument producer/consumer |
| Alert trên absolute lag | False positive khi traffic tăng | Alert trên lag growth rate hoặc time-based |
| Không có DLQ metrics | Mất message âm thầm | Monitor DLQ depth + ingress |
| Quá nhiều dashboards | Không ai xem | 1 overview + per-team drill-down |
| Metric không có runbook | Alert fire nhưng không ai biết làm gì | Link runbook trong alert annotation |

---

## Câu Hỏi Phỏng Vấn

**Q: Metric nào quan trọng nhất cho Kafka consumer?**

> Consumer lag per partition và per consumer group. Lag cho biết consumer có theo kịp producer không. Kết hợp với processing latency để biết bottleneck ở consumer hay downstream.

**Q: UnderReplicatedPartitions > 0 nghĩa là gì?**

> Có partition mà số replica in-sync (ISR — In-Sync Replicas) < replication factor. Nguy cơ mất data nếu leader fail. Cần investigate broker down, network issue, hoặc disk slow.

**Q: Làm sao phân biệt consumer chậm vs producer burst?**

> Xem produce rate vs consume rate. Nếu produce spike tạm thời → lag tăng rồi giảm (bình thường). Nếu consume rate < produce rate sustained → consumer bottleneck.

**Q: Tại sao cần business metrics ngoài technical metrics?**

> Lag = 0 không đảm bảo business đúng — consumer có thể commit offset mà skip logic. Business metrics (orders/min, error rate theo nghiệp vụ) validate end-to-end correctness.

---

**Tiếp theo:** [2-lag-monitoring.md](./2-lag-monitoring.md) — Chi tiết consumer lag monitoring và threshold design.
