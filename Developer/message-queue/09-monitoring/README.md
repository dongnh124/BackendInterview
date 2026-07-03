# Giám Sát & Observability — Tổng Quan

> Chủ đề bắt buộc trước và sau go-live production: Key Metrics (Chỉ Số Quan Trọng), Consumer Lag (Độ Trễ Consumer), Alerting Strategy (Chiến Lược Cảnh Báo), Prometheus & Grafana (Stack Giám Sát), Distributed Tracing (Truy Vết Phân Tán), và Troubleshooting Playbook (Sổ Tay Xử Lý Sự Cố).

## Mục Lục

1. [Tại Sao Monitoring Quan Trọng Với Messaging](#tại-sao-monitoring-quan-trọng-với-messaging)
2. [Ba Trụ Cột Observability](#ba-trụ-cột-observability)
3. [Kiến Trúc Monitoring Tổng Quan](#kiến-trúc-monitoring-tổng-quan)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Monitoring Checklist Nhanh](#monitoring-checklist-nhanh)
7. [Bài Tập Thực Hành](#bài-tập-thực-hành)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Monitoring Quan Trọng Với Messaging

Messaging system là **async pipeline (đường ống bất đồng bộ)** — lỗi không hiện ngay trên UI như HTTP 500. Consumer lag tích lũy âm thầm, DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) phình to mà không ai biết, rebalance storm (bão tái cân bằng) làm gián đoạn processing — tất cả đều cần **observability (khả năng quan sát)** để phát hiện sớm.

| Triệu Chứng Không Monitor | Hậu Quả | Metric Cần Theo Dõi |
| ------------------------- | ------- | ------------------- |
| Consumer chậm dần | Message stale, SLA breach | Consumer lag, processing latency |
| Broker disk đầy | Broker crash, produce fail | Disk usage, log retention |
| Rebalance liên tục | Throughput giảm đột ngột | Rebalance rate, group stability |
| DLQ tăng âm thầm | Mất data business-critical | DLQ depth, error rate |
| Hot partition | Uneven load, một consumer quá tải | Per-partition lag |

> **Quy tắc vàng:** Nếu bạn không đo được **consumer lag** và **error rate**, bạn không biết messaging system có healthy hay không. Đây là metric số 1 mọi team messaging phải có dashboard.

---

## Ba Trụ Cột Observability

Observability (Khả Năng Quan Sát) cho messaging layer dựa trên ba pillar (trụ cột) chuẩn:

```
┌──────────────────────────────────────────────────────────────────────────┐
│              OBSERVABILITY PILLARS (Ba Trụ Cột Quan Sát)                  │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │    METRICS      │  │      LOGS       │  │        TRACES           │  │
│  │  (Chỉ Số Số)    │  │  (Nhật Ký)      │  │  (Dấu Vết Phân Tán)     │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────────────┤  │
│  │ Lag, throughput │  │ Consumer errors │  │ correlation ID          │  │
│  │ Error rate      │  │ Rebalance events│  │ end-to-end latency      │  │
│  │ Queue depth     │  │ DLQ messages    │  │ produce → consume flow  │  │
│  │ Broker CPU/disk │  │ Broker warnings │  │ cross-service debug     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │
│         │                      │                        │              │
│         └──────────────────────┼────────────────────────┘              │
│                                ▼                                        │
│                    ALERTING + DASHBOARD + RUNBOOK                       │
└──────────────────────────────────────────────────────────────────────────┘
```

| Pillar | Công Cụ Phổ Biến | Câu Hỏi Trả Lời |
| ------ | ---------------- | --------------- |
| **Metrics (Chỉ Số)** | Prometheus, Grafana, Datadog, CloudWatch | "Hệ thống có healthy không? Lag bao nhiêu?" |
| **Logs (Nhật Ký)** | ELK, Loki, CloudWatch Logs | "Consumer fail vì lý do gì? Exception gì?" |
| **Traces (Dấu Vết)** | OpenTelemetry, Jaeger, Zipkin | "Message đi từ đâu đến đâu? Bottleneck ở đâu?" |

---

## Kiến Trúc Monitoring Tổng Quan

```
┌──────────────────────────────────────────────────────────────────────────┐
│              MESSAGING MONITORING STACK (Stack Giám Sát Messaging)        │
│                                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Producer │  │  Broker  │  │ Consumer │  │   DLQ    │  │ Downstream│  │
│  │  metrics │  │  JMX/    │  │  custom  │  │  depth   │  │  health  │  │
│  │          │  │  exporter│  │  metrics │  │          │  │          │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │             │             │             │             │        │
│       └─────────────┴─────────────┴─────────────┴─────────────┘        │
│                                   │                                      │
│                          ┌────────▼────────┐                             │
│                          │   Prometheus    │                             │
│                          │  (Time Series)  │                             │
│                          └────────┬────────┘                             │
│                                   │                                      │
│              ┌────────────────────┼────────────────────┐                 │
│              ▼                    ▼                    ▼                 │
│       ┌──────────┐        ┌──────────┐        ┌──────────┐              │
│       │ Grafana  │        │ Alertmgr │        │  Runbook │              │
│       │Dashboard │        │  Pager   │        │  Links   │              │
│       └──────────┘        └──────────┘        └──────────┘              │
└──────────────────────────────────────────────────────────────────────────┘
```

**Năm lớp cần monitor:**

1. **Broker Layer** — CPU, memory, disk, network, under-replicated partitions
2. **Producer Layer** — produce rate, error rate, request latency, batch size
3. **Consumer Layer** — lag, processing time, commit rate, rebalance events
4. **DLQ Layer** — depth, ingress rate, age of oldest message
5. **Business Layer** — orders processed/min, payment events lag, SLA compliance

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 4–6 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-key-metrics.md](./1-key-metrics.md) | Lag, throughput, error rate, rebalance | 45 phút |
| 2 | [2-lag-monitoring.md](./2-lag-monitoring.md) | Consumer lag, alerting thresholds | 45 phút |
| 3 | [3-alerting-strategy.md](./3-alerting-strategy.md) | SLOs, alert fatigue, runbook integration | 45 phút |
| 4 | [4-prometheus-grafana.md](./4-prometheus-grafana.md) | Metrics setup cho Kafka/RabbitMQ | 1 giờ |
| 5 | [5-distributed-tracing.md](./5-distributed-tracing.md) | OpenTelemetry, correlation ID | 45 phút |
| 6 | [6-troubleshooting-playbook.md](./6-troubleshooting-playbook.md) | Common issues & diagnosis steps | 45 phút |
| 7 | [7-production-checklist.md](./7-production-checklist.md) | Pre-deployment & go-live checklist | 30 phút |

**Điều kiện tiên quyết:**

- [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md) — hiểu consumer groups, offset, rebalance
- [06-reliability/README.md](../06-reliability/README.md) — hiểu DLQ, retry, error handling
- [07-performance-scaling/README.md](../07-performance-scaling/README.md) — hiểu throughput, scaling

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4 → 5 → 6 → 7. Nắm metrics trước, lag monitoring (quan trọng nhất), alerting strategy, setup tooling, tracing, troubleshooting, cuối cùng checklist go-live.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-key-metrics.md](./1-key-metrics.md) | Golden signals cho messaging, broker/producer/consumer metrics |
| [2-lag-monitoring.md](./2-lag-monitoring.md) | Kafka consumer lag, RabbitMQ queue depth, threshold design |
| [3-alerting-strategy.md](./3-alerting-strategy.md) | SLO/SLI, alert routing, on-call, runbook links |
| [4-prometheus-grafana.md](./4-prometheus-grafana.md) | JMX Exporter, kafka_exporter, rabbitmq_exporter, dashboard |
| [5-distributed-tracing.md](./5-distributed-tracing.md) | Correlation ID, trace context propagation, OpenTelemetry |
| [6-troubleshooting-playbook.md](./6-troubleshooting-playbook.md) | Lag spike, rebalance storm, disk full, DLQ flood |
| [7-production-checklist.md](./7-production-checklist.md) | Pre-deployment monitoring checklist |

---

## Monitoring Checklist Nhanh

```markdown
## Messaging Monitoring — Minimum Viable Setup

### Metrics (Bắt Buộc)
- [ ] Consumer lag per topic/partition/consumer group
- [ ] Produce/consume rate (messages/sec)
- [ ] Error rate (produce fail, consume fail, DLQ ingress)
- [ ] Broker disk usage & retention headroom
- [ ] DLQ depth

### Alerting (Bắt Buộc)
- [ ] Lag > threshold (warning + critical)
- [ ] Consumer group có 0 active members
- [ ] Broker disk > 80%
- [ ] DLQ ingress rate > 0 sustained
- [ ] Under-replicated partitions > 0 (Kafka)

### Dashboard (Khuyến Nghị)
- [ ] Overview: lag, throughput, error rate
- [ ] Per-service consumer health
- [ ] Broker cluster health
- [ ] DLQ trend (7 ngày)

### Runbook (Khuyến Nghị)
- [ ] Mỗi alert có link runbook
- [ ] Escalation path rõ ràng
- [ ] Post-mortem template sẵn sàng
```

---

## Bài Tập Thực Hành

### Lab 1: Setup Prometheus + Grafana Cho Kafka (90 phút)

```
Mục tiêu: Dashboard hiển thị consumer lag, produce rate, broker disk.

Bước:
1. Deploy kafka_exporter + JMX Exporter trong Docker Compose
2. Cấu hình Prometheus scrape targets
3. Import Grafana dashboard (Kafka overview)
4. Tạo consumer chậm cố ý → quan sát lag tăng
5. Scale consumer → quan sát lag giảm
```

### Lab 2: Thiết Kế Alert Rules (45 phút)

```
Kịch bản: Order processing pipeline — SLA xử lý order trong 5 phút.

Thiết kế:
1. SLI (Service Level Indicator — Chỉ Số Mức Dịch Vụ): consumer lag tương đương < 5 phút processing time
2. SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ): 99.9% thời gian lag < threshold
3. Warning alert: lag > 2 phút equivalent
4. Critical alert: lag > 5 phút equivalent
5. Runbook link cho mỗi alert
```

### Lab 3: Troubleshooting Simulation (60 phút)

```
Gây lỗi có chủ đích:
1. Kill consumer pod → lag spike → diagnose từ dashboard
2. Tăng message size → produce latency tăng
3. Fill DLQ → alert fire
4. Ghi lại diagnosis steps → cập nhật playbook
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Consumer lag là gì? Làm sao đo? | Offset difference giữa log end và committed offset; kafka_exporter, Burrow, custom metrics |
| Metric nào quan trọng nhất cho messaging? | Consumer lag, error rate, DLQ depth — giải thích tại sao |
| Làm sao tránh alert fatigue? | SLO-based alerting, grouping, runbook, alert on symptoms not causes |
| Lag cao — troubleshoot như thế nào? | Check consumer alive → processing time → downstream → partition skew |
| Prometheus vs CloudWatch cho Kafka? | Trade-offs: self-hosted flexibility vs managed simplicity |
| Correlation ID dùng để làm gì? | Trace message qua producer → broker → consumer → downstream |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Consumer scaling khi lag cao | [07-performance-scaling/3-consumer-scaling.md](../07-performance-scaling/3-consumer-scaling.md) |
| DLQ monitoring | [06-reliability/3-dead-letter-handling.md](../06-reliability/3-dead-letter-handling.md) |
| Backpressure signals | [07-performance-scaling/4-backpressure-handling.md](../07-performance-scaling/4-backpressure-handling.md) |
| Security audit logs | [08-security/4-audit-logging.md](../08-security/4-audit-logging.md) |

---

**Cập Nhật:** 2026-07-03
