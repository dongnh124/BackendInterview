# Lag Monitoring — Giám Sát Độ Trễ Consumer

> Consumer Lag (Độ Trễ Consumer) là metric quan trọng nhất trong messaging observability. Bài này đi sâu cách đo lag, thiết kế threshold (ngưỡng), và alerting cho Kafka và RabbitMQ.

## Mục Lục

1. [Consumer Lag Là Gì](#consumer-lag-là-gì)
2. [Cách Đo Lag — Kafka](#cách-đo-lag--kafka)
3. [Cách Đo Lag — RabbitMQ](#cách-đo-lag--rabbitmq)
4. [Lag vs Queue Depth vs Processing Delay](#lag-vs-queue-depth-vs-processing-delay)
5. [Thiết Kế Threshold](#thiết-kế-threshold)
6. [Per-Partition Lag Analysis](#per-partition-lag-analysis)
7. [Công Cụ Lag Monitoring](#công-cụ-lag-monitoring)
8. [Alert Rules Mẫu](#alert-rules-mẫu)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Consumer Lag Là Gì

**Consumer Lag (Độ Trễ Consumer)** là số lượng message (hoặc khoảng offset) mà consumer **chưa xử lý** so với message mới nhất trên broker.

```
Partition timeline:
─────────────────────────────────────────────────────────────►
     committed          current              log end
     offset             position             offset
        │                   │                    │
        ▼                   ▼                    ▼
        [████████████████████│░░░░░░░░░░░░░░░░░░░]
        │    đã xử lý        │      LAG           │
        │                    │  (chưa commit)     │
```

**Hai loại lag:**

| Loại | Định Nghĩa | Dùng Khi |
| ---- | ---------- | -------- |
| **Offset lag** | `log_end_offset - committed_offset` | Alerting, capacity planning |
| **Time lag** | Thời gian từ message produce → consume | SLA-based alerting |

```
Time lag ≈ offset lag × average_processing_time_per_message

Ví dụ: lag = 10,000 messages, processing = 50ms/message
       → time lag ≈ 500 giây (~8.3 phút)
```

---

## Cách Đo Lag — Kafka

### Công Thức

```python
# Per partition
lag[partition] = log_end_offset(partition) - committed_offset(partition, consumer_group)

# Total lag cho consumer group
total_lag = sum(lag[p] for p in assigned_partitions)
```

### Nguồn Dữ Liệu

| Nguồn | Cách Lấy | Ưu/Nhược |
| ----- | -------- | -------- |
| **kafka_exporter** | Prometheus metric `kafka_consumergroup_lag` | Dễ setup, phổ biến |
| **Burrow** | HTTP API, lag evaluation | Smart lag analysis, status OK/WARN/ERR |
| **Consumer JMX** | `records-lag-max` từ client | Chỉ lag của consumer instance đó |
| **kafka-consumer-groups CLI** | `kafka-consumer-groups --describe` | Manual debug, không real-time |
| **Cruise Control** | LinkedIn tool, lag + auto rebalance | Enterprise, phức tạp |

### kafka_exporter Metrics

```promql
# Total lag per consumer group
sum(kafka_consumergroup_lag{consumergroup="order-processor"}) by (consumergroup)

# Lag per partition — phát hiện hot partition
kafka_consumergroup_lag{consumergroup="order-processor", topic="orders", partition="3"}

# Lag growth rate — quan trọng hơn absolute value
rate(kafka_consumergroup_lag{consumergroup="order-processor"}[5m]) > 0
```

### Burrow Lag Evaluation

**Burrow** không chỉ đo lag mà **đánh giá trạng thái**:

```
Status OK:     Lag stable hoặc decreasing
Status WARN:   Lag increasing nhưng chưa critical
Status ERR:    Lag increasing sustained, consumer có vấn đề
Status STOP:   Không có offset commit mới (consumer dead?)
```

```json
// Burrow API response example
{
  "status": {
    "status": "ERR",
    "maxlag": {
      "topic": "orders",
      "partition": 3,
      "end": { "offset": 1000000, "timestamp": 1719900000000 },
      "current_lag": 50000
    }
  }
}
```

---

## Cách Đo Lag — RabbitMQ

RabbitMQ không có concept "offset" như Kafka — thay vào đó dùng **queue depth (độ sâu hàng đợi)**:

```
┌─────────────────────────────────────────────────────────────────┐
│              RABBITMQ QUEUE MESSAGE STATES                         │
│                                                                  │
│  Published ──► [Ready Queue] ──► Delivered ──► [Unacked] ──► Ack │
│                    │                              │              │
│              messages_ready              messages_unacknowledged │
│              (= "lag" chính)            (= đang xử lý)          │
└─────────────────────────────────────────────────────────────────┘
```

| Metric | Ý Nghĩa | Tương Đương Kafka |
| ------ | ------- | ----------------- |
| **messages_ready** | Chờ consumer lấy | Consumer lag |
| **messages_unacknowledged** | Đã deliver, chưa ack | In-flight messages |
| **messages** | ready + unacknowledged | Total backlog |

```promql
# Queue depth alert
rabbitmq_queue_messages_ready{queue="order-processing"} > 1000

# Consumer không hoạt động
rabbitmq_queue_consumers{queue="order-processing"} == 0
and rabbitmq_queue_messages_ready{queue="order-processing"} > 0
```

### Time-Based Lag Cho RabbitMQ

```python
# Ước tính time lag từ queue depth
time_lag_seconds = messages_ready / consume_rate_per_second

# Ví dụ: 5000 messages ready, consume 100/sec → 50 giây lag
```

---

## Lag vs Queue Depth vs Processing Delay

| Khái Niệm | Đo Gì | Broker |
| --------- | ----- | ------ |
| **Offset lag** | Message chưa commit offset | Kafka |
| **Queue depth** | Message chờ trong queue | RabbitMQ, SQS |
| **Processing delay** | Thời gian xử lý 1 message | Application |
| **End-to-end latency** | Produce timestamp → processed | Cross-broker |
| **Age of oldest message** | Message cũ nhất trong queue | RabbitMQ, SQS |

```
End-to-end latency = queue_wait_time + processing_time + commit_time

Trong đó:
  queue_wait_time ≈ lag / consume_rate (Kafka)
  queue_wait_time ≈ messages_ready / consume_rate (RabbitMQ)
```

> **Lưu ý phỏng vấn:** Lag cao không luôn = SLA breach. Nếu processing nhanh (10ms/msg) thì lag 10,000 chỉ = 100 giây. Luôn convert lag sang time-based khi thiết kế SLA alert.

---

## Thiết Kế Threshold

### Phương Pháp 1: Absolute Lag Threshold

```
Warning:  lag > 10,000 messages
Critical: lag > 50,000 messages
```

**Ưu:** Đơn giản  
**Nhược:** False positive khi traffic spike hợp lệ

### Phương Pháp 2: Time-Based Threshold (Khuyến Nghị)

```
Warning:  estimated_time_lag > 2 minutes
Critical: estimated_time_lag > 5 minutes

estimated_time_lag = current_lag / avg_consume_rate_5m
```

```promql
# Prometheus alert rule
(
  sum(kafka_consumergroup_lag{consumergroup="order-processor"})
  /
  sum(rate(kafka_consumergroup_current_offset{consumergroup="order-processor"}[5m])
      - kafka_consumergroup_lag{consumergroup="order-processor"})
) > 300  # 5 phút
```

### Phương Pháp 3: Lag Growth Rate

```
Alert khi: lag tăng liên tục 15 phút (dù absolute value thấp)

rate(kafka_consumergroup_lag[15m]) > 0
AND kafka_consumergroup_lag > 1000
```

**Ưu:** Phát hiện sớm consumer chậm dần  
**Nhược:** Cần baseline period

### Phương Pháp 4: SLO-Based (Production Grade)

```
SLI: % thời gian lag tương đương < 5 phút processing
SLO: 99.9% trong 30 ngày

Error budget: 0.1% × 30 days = 43.2 phút lag breach được phép/tháng
```

| Tier | Time Lag Threshold | Use Case |
| ---- | ------------------ | -------- |
| **Real-time** | < 30 giây | Notification, fraud detection |
| **Near real-time** | < 5 phút | Order processing, payment |
| **Batch-like** | < 1 giờ | Analytics, reporting |
| **Daily batch** | < 24 giờ | ETL, data warehouse sync |

---

## Per-Partition Lag Analysis

Hot partition (phân vùng nóng) là nguyên nhân phổ biến lag không đều:

```
Consumer Group: order-processor (3 consumers, 6 partitions)

Partition │ Lag    │ Consumer    │ Status
──────────┼────────┼─────────────┼────────
    0     │  100   │ consumer-1  │ OK
    1     │  150   │ consumer-2  │ OK
    2     │ 50,000 │ consumer-3  │ HOT ⚠️
    3     │  120   │ consumer-1  │ OK
    4     │  130   │ consumer-2  │ OK
    5     │  110   │ consumer-3  │ OK
```

**Diagnosis steps:**

1. Check partition key distribution — key skew?
2. Check message size partition 2 — oversized messages?
3. Check consumer-3 processing time — slow handler?
4. Check downstream dependency — DB timeout cho partition 2 keys?

```promql
# Alert: một partition lag > 10x median
kafka_consumergroup_lag{consumergroup="order-processor"}
> 10 * avg(kafka_consumergroup_lag{consumergroup="order-processor"})
```

---

## Công Cụ Lag Monitoring

| Công Cụ | Broker | Đặc Điểm |
| ------- | ------ | -------- |
| **kafka_exporter + Grafana** | Kafka | Open source, Prometheus native |
| **Burrow** | Kafka | Lag evaluation, status API |
| **Confluent Control Center** | Kafka | Commercial, full observability |
| **AWS CloudWatch** | MSK | Managed, tích hợp AWS |
| **rabbitmq_exporter** | RabbitMQ | Prometheus metrics |
| **RabbitMQ Management UI** | RabbitMQ | Built-in, manual |
| **Datadog / New Relic** | Both | SaaS, APM integration |

### Grafana Dashboard Panels Khuyến Nghị

```
Row 1: Overview
  - Total lag (gauge)
  - Lag trend (7 days)
  - Consume rate vs produce rate

Row 2: Per-Partition
  - Lag heatmap by partition
  - Hot partition highlight

Row 3: Consumer Health
  - Active consumers count
  - Rebalance events
  - Processing latency P95
```

---

## Alert Rules Mẫu

### Kafka — Prometheus Alertmanager

```yaml
groups:
  - name: kafka_lag_alerts
    rules:
      - alert: KafkaConsumerLagHigh
        expr: |
          sum(kafka_consumergroup_lag{consumergroup="order-processor"}) > 50000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag cao cho {{ $labels.consumergroup }}"
          runbook: "https://wiki.example.com/runbooks/kafka-lag-high"

      - alert: KafkaConsumerLagGrowing
        expr: |
          deriv(kafka_consumergroup_lag{consumergroup="order-processor"}[10m]) > 0
          and sum(kafka_consumergroup_lag{consumergroup="order-processor"}) > 1000
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Lag đang tăng liên tục — consumer không theo kịp"

      - alert: KafkaNoActiveConsumers
        expr: |
          kafka_consumergroup_members{consumergroup="order-processor"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Không có active consumer — messages không được xử lý"

      - alert: KafkaHotPartition
        expr: |
          max(kafka_consumergroup_lag) by (topic, partition)
          > 10 * avg(kafka_consumergroup_lag) by (topic)
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Hot partition detected: {{ $labels.topic }}-{{ $labels.partition }}"
```

### RabbitMQ — Prometheus Alertmanager

```yaml
groups:
  - name: rabbitmq_lag_alerts
    rules:
      - alert: RabbitMQQueueDepthHigh
        expr: |
          rabbitmq_queue_messages_ready{queue="order-processing"} > 5000
        for: 5m
        labels:
          severity: warning

      - alert: RabbitMQNoConsumers
        expr: |
          rabbitmq_queue_consumers{queue="order-processing"} == 0
          and rabbitmq_queue_messages_ready{queue="order-processing"} > 0
        for: 2m
        labels:
          severity: critical
```

---

## Câu Hỏi Phỏng Vấn

**Q: Consumer lag spike đột ngột — troubleshoot như thế nào?**

> 1. Check active consumers (có consumer die không?)  
> 2. Check rebalance events (rebalance storm?)  
> 3. Check produce rate spike (traffic burst hợp lệ?)  
> 4. Check processing latency (downstream slow?)  
> 5. Check per-partition lag (hot partition?)  
> 6. Check recent deployment (code regression?)

**Q: Lag = 0 nhưng business báo thiếu data?**

> Consumer có thể commit offset mà không xử lý đúng (bug skip logic), hoặc consume từ wrong topic/partition. Cần business metrics và audit log, không chỉ dựa lag.

**Q: Nên alert trên absolute lag hay growth rate?**

> Cả hai. Absolute cho SLA breach rõ ràng; growth rate phát hiện sớm degradation. Production thường dùng time-based threshold kết hợp growth rate.

---

**Tiếp theo:** [3-alerting-strategy.md](./3-alerting-strategy.md) — Thiết kế alerting strategy và tránh alert fatigue.
