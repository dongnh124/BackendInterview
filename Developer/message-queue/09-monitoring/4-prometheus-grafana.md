# Prometheus & Grafana — Setup Metrics Cho Kafka/RabbitMQ

> Hướng dẫn thực chiến setup Prometheus (Hệ Thống Thu Thập Metrics), Grafana (Dashboard Trực Quan), và exporters cho Apache Kafka và RabbitMQ trong production.

## Mục Lục

1. [Kiến Trúc Monitoring Stack](#kiến-trúc-monitoring-stack)
2. [Kafka Metrics Collection](#kafka-metrics-collection)
3. [RabbitMQ Metrics Collection](#rabbitmq-metrics-collection)
4. [Prometheus Configuration](#prometheus-configuration)
5. [Grafana Dashboards](#grafana-dashboards)
6. [Application Metrics Instrumentation](#application-metrics-instrumentation)
7. [Docker Compose Lab Setup](#docker-compose-lab-setup)
8. [Production Best Practices](#production-best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Monitoring Stack

```
┌──────────────────────────────────────────────────────────────────────────┐
│              PROMETHEUS MONITORING ARCHITECTURE                           │
│                                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │ kafka_      │  │ JMX         │  │ rabbitmq_   │  │ Application   │  │
│  │ exporter    │  │ Exporter    │  │ exporter    │  │ /metrics      │  │
│  │ :9308       │  │ :9404       │  │ :9419       │  │ :8080         │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───────┬───────┘  │
│         │                │                │                  │          │
│         └────────────────┴────────────────┴──────────────────┘          │
│                                   │ scrape (pull)                      │
│                          ┌────────▼────────┐                           │
│                          │   Prometheus    │                           │
│                          │   :9090         │                           │
│                          └────────┬────────┘                           │
│                                   │                                    │
│              ┌────────────────────┼────────────────────┐               │
│              ▼                    ▼                    ▼               │
│       ┌──────────┐        ┌──────────┐        ┌──────────┐           │
│       │ Grafana  │        │Alertmgr  │        │ Thanos/  │           │
│       │ :3000    │        │ :9093    │        │ Cortex   │           │
│       └──────────┘        └──────────┘        └──────────┘           │
└──────────────────────────────────────────────────────────────────────────┘
```

| Component | Vai Trò | Port Mặc Định |
| --------- | ------- | --------------- |
| **Prometheus** | Time-series database, scrape metrics | 9090 |
| **Grafana** | Visualization, dashboards | 3000 |
| **Alertmanager** | Alert routing, grouping, silencing | 9093 |
| **kafka_exporter** | Kafka lag, consumer groups | 9308 |
| **JMX Exporter** | Kafka broker JMX metrics | 9404 |
| **rabbitmq_exporter** | RabbitMQ queue, node metrics | 9419 |

---

## Kafka Metrics Collection

### Option 1: kafka_exporter (Khuyến Nghị Cho Lag)

**kafka_exporter** (by danielqsj) expose consumer group lag — metric quan trọng nhất:

```yaml
# docker-compose snippet
kafka-exporter:
  image: danielqsj/kafka-exporter:latest
  command:
    - --kafka.server=kafka:9092
    - --sasl.enabled          # Nếu có auth
    - --sasl.username=admin
    - --sasl.password=${KAFKA_PASSWORD}
    - --sasl.mechanism=SCRAM-SHA-256
    - --tls.enabled
    - --tls.ca-file=/certs/ca.pem
  ports:
    - "9308:9308"
```

**Metrics chính:**

```promql
kafka_consumergroup_lag{consumergroup, topic, partition}
kafka_consumergroup_current_offset{consumergroup, topic, partition}
kafka_consumergroup_members{consumergroup}
kafka_topic_partitions{topic}
```

### Option 2: JMX Exporter (Broker Metrics)

**JMX Exporter** scrape Kafka broker JMX — CPU, disk, request latency, replication:

```yaml
# jmx_exporter config (kafka-2_0_0.yml)
rules:
  - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value
    name: kafka_server_$1_$2
    labels:
      clientId: "$3"
      topic: "$4"
      partition: "$5"
```

```properties
# Kafka broker — enable JMX Exporter as Java agent
KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx_exporter/kafka.yml"
```

**Metrics chính từ JMX:**

```promql
kafka_server_ReplicaManager_UnderReplicatedPartitions
kafka_server_BrokerTopicMetrics_MessagesInPerSec
kafka_server_BrokerTopicMetrics_BytesInPerSec
kafka_network_RequestMetrics_TotalTimeMs{request="Produce"}
kafka_log_Log_Size{topic, partition}
kafka_controller_KafkaController_OfflinePartitionsCount
```

### Option 3: Kết Hợp Cả Hai (Production)

| Exporter | Metrics | Use Case |
| -------- | ------- | -------- |
| **kafka_exporter** | Consumer lag, offsets | Alerting lag |
| **JMX Exporter** | Broker health, throughput | Infrastructure monitoring |
| **Node Exporter** | CPU, disk, network | Host-level metrics |

---

## RabbitMQ Metrics Collection

### Option 1: Built-in Prometheus Plugin (Khuyến Nghị)

RabbitMQ 3.8+ có **built-in Prometheus plugin**:

```bash
# Enable plugin
rabbitmq-plugins enable rabbitmq_prometheus

# Metrics available at
# http://rabbitmq:15692/metrics
```

```yaml
# docker-compose
rabbitmq:
  image: rabbitmq:3.13-management
  environment:
    - RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=-rabbitmq_prometheus
  ports:
    - "5672:5672"
    - "15672:15672"   # Management UI
    - "15692:15692"   # Prometheus metrics
```

**Metrics chính:**

```promql
rabbitmq_queue_messages{queue, vhost}
rabbitmq_queue_messages_ready{queue}
rabbitmq_queue_messages_unacknowledged{queue}
rabbitmq_queue_consumers{queue}
rabbitmq_connections
rabbitmq_channels
rabbitmq_process_resident_memory_bytes
rabbitmq_disk_space_available_bytes
```

### Option 2: rabbitmq_exporter (Community)

```yaml
rabbitmq-exporter:
  image: kbudde/rabbitmq-exporter:latest
  environment:
    - RABBIT_URL=http://rabbitmq:15672
    - RABBIT_USER=monitoring
    - RABBIT_PASSWORD=${RABBIT_PASSWORD}
    - PUBLISH_PORT=9419
  ports:
    - "9419:9419"
```

### Per-Queue Monitoring User

```bash
# Tạo user chỉ đọc cho monitoring — least privilege
rabbitmqctl add_user monitoring ${PASSWORD}
rabbitmqctl set_permissions -p / monitoring "" "" ".*"
rabbitmqctl set_user_tags monitoring monitoring
```

---

## Prometheus Configuration

### prometheus.yml Mẫu

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: messaging-prod
    env: production

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/kafka_alerts.yml
  - /etc/prometheus/rules/rabbitmq_alerts.yml

scrape_configs:
  # Kafka consumer lag
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']
    scrape_interval: 30s

  # Kafka broker JMX
  - job_name: 'kafka-jmx'
    static_configs:
      - targets: ['kafka-1:9404', 'kafka-2:9404', 'kafka-3:9404']

  # RabbitMQ built-in prometheus
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']

  # Application custom metrics
  - job_name: 'order-processor'
    static_configs:
      - targets: ['order-processor:8080']
    metrics_path: /actuator/prometheus  # Spring Boot
    # hoặc /metrics cho Node.js prom-client

  # Node exporter — host metrics
  - job_name: 'node'
    static_configs:
      - targets: ['kafka-1:9100', 'kafka-2:9100', 'kafka-3:9100']
```

### Recording Rules — Pre-compute Expensive Queries

```yaml
# rules/kafka_recording.yml
groups:
  - name: kafka_recording
    interval: 30s
    rules:
      - record: kafka:consumergroup_lag_total
        expr: sum(kafka_consumergroup_lag) by (consumergroup, topic)

      - record: kafka:consumergroup_lag_time_estimate_seconds
        expr: |
          kafka:consumergroup_lag_total
          /
          sum(rate(kafka_consumergroup_current_offset[5m])) by (consumergroup, topic)

      - record: kafka:produce_rate
        expr: sum(rate(kafka_server_BrokerTopicMetrics_MessagesInPerSec[5m])) by (topic)
```

---

## Grafana Dashboards

### Dashboard Structure Khuyến Nghị

```
📊 Messaging Overview Dashboard
├── Row 1: Health Summary
│   ├── Total Consumer Lag (stat)
│   ├── Active Consumers (stat)
│   ├── Error Rate (stat)
│   └── DLQ Depth (stat)
├── Row 2: Throughput
│   ├── Produce Rate (timeseries)
│   ├── Consume Rate (timeseries)
│   └── Lag Trend 24h (timeseries)
├── Row 3: Per Consumer Group
│   ├── Lag by Group (bar gauge)
│   └── Processing Latency P95 (timeseries)
└── Row 4: Broker Health
    ├── Disk Usage (gauge)
    ├── Under-Replicated Partitions (stat)
    └── Request Latency P99 (timeseries)
```

### Grafana Panel Examples

**Panel 1: Consumer Lag Gauge**

```promql
sum(kafka_consumergroup_lag{consumergroup="order-processor"})
```

**Panel 2: Lag vs Consume Rate**

```promql
# Lag
sum(kafka_consumergroup_lag{consumergroup="order-processor"})

# Consume rate
sum(rate(kafka_consumergroup_current_offset{consumergroup="order-processor"}[5m]))
```

**Panel 3: Hot Partition Heatmap**

```promql
kafka_consumergroup_lag{consumergroup="order-processor"}
```

**Panel 4: RabbitMQ Queue Depth**

```promql
rabbitmq_queue_messages_ready{queue=~"order.*"}
```

### Import Community Dashboards

| Dashboard | ID | Broker |
| --------- | -- | ------ |
| Kafka Overview | 7589 | Kafka JMX |
| Kafka Consumer Lag | 11159 | kafka_exporter |
| RabbitMQ Overview | 10991 | rabbitmq_prometheus |
| RabbitMQ Queues | 11340 | Per-queue detail |

```
Grafana → Import → Dashboard ID → Select Prometheus datasource
```

---

## Application Metrics Instrumentation

Broker metrics không đủ — cần **application-level metrics**:

### Node.js (prom-client)

```javascript
const { Counter, Histogram, register } = require('prom-client');

const messagesConsumed = new Counter({
  name: 'messaging_consume_total',
  help: 'Total messages consumed',
  labelNames: ['topic', 'status']  // success | error
});

const processingDuration = new Histogram({
  name: 'messaging_consume_duration_seconds',
  help: 'Message processing duration',
  labelNames: ['topic'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
});

async function processMessage(topic, message) {
  const end = processingDuration.startTimer({ topic });
  try {
    await handleBusinessLogic(message);
    messagesConsumed.inc({ topic, status: 'success' });
  } catch (err) {
    messagesConsumed.inc({ topic, status: 'error' });
    throw err;
  } finally {
    end();
  }
}

// Expose /metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Java/Spring Boot (Micrometer)

```java
@Timed(value = "messaging.consume", extraTags = {"topic", "orders"})
@Counted(value = "messaging.consume.total", extraTags = {"topic", "orders"})
public void processOrder(ConsumerRecord<String, OrderEvent> record) {
    // business logic
}

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    tags:
      application: order-processor
```

### .NET (prometheus-net)

```csharp
private static readonly Counter MessagesConsumed = Metrics
    .CreateCounter("messaging_consume_total", "Messages consumed",
        new CounterConfiguration { LabelNames = new[] { "topic", "status" } });

private static readonly Histogram ProcessingDuration = Metrics
    .CreateHistogram("messaging_consume_duration_seconds", "Processing time",
        new HistogramConfiguration { LabelNames = new[] { "topic" } });

using (ProcessingDuration.WithLabels("orders").NewTimer())
{
    await ProcessOrder(message);
    MessagesConsumed.WithLabels("orders", "success").Inc();
}
```

---

## Docker Compose Lab Setup

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    command: ["--kafka.server=kafka:9092"]
    ports:
      - "9308:9308"
    depends_on:
      - kafka

volumes:
  grafana-data:
```

```bash
# Khởi động stack
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

# Verify scrape targets
curl http://localhost:9090/api/v1/targets

# Verify kafka_exporter metrics
curl http://localhost:9308/metrics | grep consumergroup_lag
```

---

## Production Best Practices

| Practice | Chi Tiết |
| -------- | -------- |
| **Retention** | Prometheus 15–30 ngày local; Thanos/Cortex cho long-term |
| **High availability** | 2+ Prometheus instances, remote write |
| **Security** | TLS cho scrape, auth cho Grafana, network isolation |
| **Cardinality control** | Tránh high-cardinality labels (message_id, user_id) |
| **Recording rules** | Pre-compute expensive queries |
| **Dedicated monitoring user** | Least privilege cho exporter credentials |
| **Dashboard as code** | Grafana provisioning, version control JSON |
| **Alert testing** | `amtool alert add` test alerts trước go-live |

### Cardinality Warning

```promql
# ❌ BAD — millions of unique time series
messaging_message_processed{message_id="abc-123", user_id="user-456"}

# ✅ GOOD — bounded cardinality
messaging_message_processed{topic="orders", status="success", service="order-processor"}
```

---

## Câu Hỏi Phỏng Vấn

**Q: kafka_exporter vs JMX Exporter — khi nào dùng cái nào?**

> kafka_exporter cho consumer lag và consumer group health — metric quan trọng nhất cho alerting. JMX Exporter cho broker infrastructure (disk, replication, request latency). Production dùng cả hai.

**Q: Làm sao monitor consumer lag trong K8s?**

> Deploy kafka_exporter as sidecar hoặc standalone Deployment. Scrape qua Prometheus Operator ServiceMonitor. Grafana dashboard per namespace/team.

**Q: Prometheus scrape interval bao nhiêu là hợp lý?**

> 15–30 giây cho hầu hết metrics. Lag metrics có thể 30s (không cần real-time sub-second). Trade-off giữa resolution và Prometheus storage/load.

---

**Tiếp theo:** [5-distributed-tracing.md](./5-distributed-tracing.md) — Distributed tracing với OpenTelemetry.
