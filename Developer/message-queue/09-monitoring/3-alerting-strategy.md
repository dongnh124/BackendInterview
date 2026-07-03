# Alerting Strategy — Chiến Lược Cảnh Báo Cho Messaging

> Thiết kế alerting (cảnh báo) hiệu quả: SLOs (Service Level Objectives — Mục Tiêu Mức Dịch Vụ), SLIs (Service Level Indicators — Chỉ Số Mức Dịch Vụ), tránh alert fatigue (mệt mỏi vì cảnh báo), và tích hợp runbook (sổ tay vận hành).

## Mục Lục

1. [Nguyên Tắc Alerting](#nguyên-tắc-alerting)
2. [SLI, SLO, SLA Cho Messaging](#sli-slo-sla-cho-messaging)
3. [Phân Cấp Severity](#phân-cấp-severity)
4. [Alert Routing & On-Call](#alert-routing--on-call)
5. [Tránh Alert Fatigue](#tránh-alert-fatigue)
6. [Runbook Integration](#runbook-integration)
7. [Alert Templates Messaging](#alert-templates-messaging)
8. [Error Budget & Burn Rate](#error-budget--burn-rate)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Nguyên Tắc Alerting

```
┌─────────────────────────────────────────────────────────────────┐
│              ALERTING PRINCIPLES (Nguyên Tắc Cảnh Báo)             │
│                                                                  │
│  1. Alert on SYMPTOMS, not CAUSES                               │
│     ✅ Consumer lag > SLA threshold                              │
│     ❌ CPU > 80% (có thể bình thường)                            │
│                                                                  │
│  2. Every alert must be ACTIONABLE                               │
│     ✅ Có runbook, biết ai fix, biết fix gì                      │
│     ❌ "Something changed" — không ai biết làm gì                │
│                                                                  │
│  3. Page humans only when URGENT                                 │
│     Warning → Slack/dashboard                                    │
│     Critical → PagerDuty/on-call                                 │
│                                                                  │
│  4. Alerts should be RARE and MEANINGFUL                         │
│     Nếu alert fire hàng ngày → không phải alert, là noise      │
└─────────────────────────────────────────────────────────────────┘
```

| Nguyên Tắc | Giải Thích | Ví Dụ Messaging |
| ---------- | ---------- | --------------- |
| **Symptom-based** | Alert khi user/business bị ảnh hưởng | Lag > SLA, DLQ ingress, orders stale |
| **Actionable** | On-call biết bước tiếp theo | Runbook link trong annotation |
| **Urgent vs non-urgent** | Phân loại severity đúng | Warning = Slack; Critical = page |
| **Low noise** | `for:` duration, grouping, inhibition | Chờ 5m trước khi fire |

---

## SLI, SLO, SLA Cho Messaging

### Định Nghĩa

| Khái Niệm | Định Nghĩa | Ví Dụ Messaging |
| --------- | ---------- | --------------- |
| **SLI** | Metric đo chất lượng dịch vụ | % message xử lý trong 5 phút |
| **SLO** | Mục tiêu nội bộ cho SLI | 99.9% message xử lý < 5 phút |
| **SLA** | Cam kết với khách hàng (có penalty) | 99.5% uptime, credit nếu vi phạm |
| **Error Budget** | Phần được phép fail | 0.1% = 43.2 phút/tháng |

### SLI Examples Cho Messaging

```yaml
# SLI 1: Processing timeliness
sli_processing_timeliness:
  good_events: messages_processed_within_5min
  total_events: messages_received
  ratio: good / total

# SLI 2: Delivery success rate
sli_delivery_success:
  good_events: messages_consumed_successfully
  total_events: messages_delivered
  ratio: good / total

# SLI 3: DLQ rate
sli_dlq_rate:
  good_events: messages_not_in_dlq
  total_events: messages_processed
  ratio: good / total
```

### SLO Targets Theo Use Case

| Pipeline | SLO Target | SLI Chính |
| -------- | ---------- | --------- |
| **Payment processing** | 99.99% < 30s | Time lag |
| **Order fulfillment** | 99.9% < 5 min | Time lag |
| **Email notification** | 99% < 1 min | End-to-end latency |
| **Analytics ETL** | 99% < 4 hours | Daily completion |
| **Audit log** | 99.99% delivered | At-least-once success |

---

## Phân Cấp Severity

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    SEVERITY MATRIX (Ma Trận Mức Độ)                       │
│                                                                          │
│  SEVERITY    │ RESPONSE        │ CHANNEL        │ VÍ DỤ                 │
│  ────────────┼─────────────────┼────────────────┼────────────────────── │
│  P1 Critical │ < 15 min        │ PagerDuty      │ No consumers, data    │
│              │                 │ Phone call     │ loss risk, all lag    │
│  P2 High     │ < 1 hour        │ PagerDuty      │ Lag > SLA, DLQ flood  │
│              │                 │ Slack urgent   │                       │
│  P3 Warning  │ Next business   │ Slack channel  │ Lag growing, disk 75% │
│              │ day             │ Email          │                       │
│  P4 Info     │ No response     │ Dashboard only │ Metric anomaly        │
│              │ required        │                │                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### Messaging Alert Severity Guide

| Alert | Severity | Lý Do |
| ----- | -------- | ----- |
| Zero active consumers + lag > 0 | **P1** | Message không được xử lý |
| UnderReplicatedPartitions > 0 sustained | **P1** | Data loss risk |
| Consumer lag > SLA critical threshold | **P2** | Business impact |
| DLQ ingress rate > 0 sustained 15m | **P2** | Data quality issue |
| Lag growing 15m+ | **P3** | Early warning |
| Broker disk > 80% | **P3** | Capacity warning |
| Rebalance event | **P4** | Thường transient |
| Produce rate drop | **P3/P4** | Tùy business impact |

---

## Alert Routing & On-Call

### Routing Logic

```
Alert fires
    │
    ├── severity: critical + service: payment-pipeline
    │       └──► PagerDuty → Payment team on-call
    │
    ├── severity: warning + service: analytics-pipeline
    │       └──► Slack #analytics-alerts
    │
    └── severity: info
            └──► Grafana annotation only
```

### Alertmanager Configuration Mẫu

```yaml
route:
  receiver: default-slack
  group_by: ['alertname', 'consumergroup', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: pagerduty-messaging
      repeat_interval: 1h

    - match:
        severity: warning
        team: platform
      receiver: slack-platform-alerts

    - match:
        service: payment-pipeline
      receiver: pagerduty-payment-team

receivers:
  - name: pagerduty-messaging
    pagerduty_configs:
      - service_key: '<key>'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

  - name: slack-platform-alerts
    slack_configs:
      - channel: '#messaging-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.summary }}\nRunbook: {{ .CommonAnnotations.runbook }}'
```

### On-Call Expectations

| Role | Trách Nhiệm | Khi Nào Page |
| ---- | ----------- | ------------ |
| **Primary on-call** | Acknowledge trong 5 phút, triage | P1, P2 alerts |
| **Secondary on-call** | Backup nếu primary không respond | Escalation sau 15 phút |
| **Service owner** | Root cause fix, post-mortem | P1 sustained > 30 phút |
| **Platform team** | Broker infrastructure issues | Broker down, disk full |

---

## Tránh Alert Fatigue

**Alert fatigue (Mệt Mỏi Vì Cảnh Báo)** xảy ra khi quá nhiều alert không actionable → on-call ignore tất cả.

### Nguyên Nhân Phổ Biến

| Nguyên Nhân | Giải Pháp |
| ----------- | --------- |
| Alert trên mọi metric change | Chỉ alert symptom ảnh hưởng user |
| Threshold quá nhạy | Dùng `for: 5m`, `for: 15m` |
| Không group alerts | `group_by` trong Alertmanager |
| Repeat quá thường xuyên | `repeat_interval: 4h` |
| Warning cũng page | Warning → Slack only |
| Không có runbook | Mọi alert phải có action steps |

### Kỹ Thuật Giảm Noise

```yaml
# 1. Dùng "for" duration — chờ sustained trước khi fire
- alert: KafkaConsumerLagHigh
  expr: kafka_consumergroup_lag > 50000
  for: 10m          # Không fire nếu spike < 10 phút

# 2. Inhibition — suppress warning khi critical đã fire
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['consumergroup', 'topic']

# 3. Group related alerts
group_by: ['alertname', 'consumergroup']

# 4. Maintenance window — silence alerts khi deploy
# Dùng Alertmanager silences hoặc cron
```

### Alert Review Process

```
Hàng tháng:
1. Review alerts fired trong 30 ngày
2. Alert nào không ai action? → Xóa hoặc downgrade
3. Alert nào miss incident? → Thêm alert mới
4. Update runbook cho alerts thường xuyên
5. Track MTTR (Mean Time To Resolve — Thời Gian Giải Quyết Trung Bình)
```

---

## Runbook Integration

Mỗi alert **bắt buộc** có runbook link trong annotation:

```yaml
annotations:
  summary: "Consumer lag cao: {{ $labels.consumergroup }}"
  description: |
    Lag hiện tại: {{ $value }} messages
    Topic: {{ $labels.topic }}
  runbook: "https://wiki.example.com/runbooks/kafka-consumer-lag-high"
  dashboard: "https://grafana.example.com/d/kafka-lag"
  escalation: "Nếu không resolve trong 30 phút → page @platform-oncall"
```

### Runbook Template

```markdown
# Runbook: Kafka Consumer Lag High

## Symptoms
- Alert: KafkaConsumerLagHigh
- Consumer group lag > threshold sustained

## Impact
- Orders/notifications delayed
- SLA breach risk

## Diagnosis Steps
1. [ ] Check Grafana dashboard: [link]
2. [ ] Verify active consumers: `kafka-consumer-groups --describe --group <group>`
3. [ ] Check produce rate vs consume rate
4. [ ] Check per-partition lag (hot partition?)
5. [ ] Check recent deployments
6. [ ] Check downstream health (DB, API)

## Resolution Steps
### If consumer dead:
1. Restart consumer pods
2. Verify lag decreasing

### If consumer slow:
1. Check processing latency metrics
2. Scale consumers (if partitions available)
3. Check downstream bottleneck

### If hot partition:
1. Investigate key skew
2. Consider partition rebalancing

## Escalation
- 30 min no improvement → Platform team
- Data loss risk → P1 escalation

## Post-Incident
- [ ] Update post-mortem template
- [ ] Review alert threshold
```

---

## Alert Templates Messaging

### Template 1: Consumer Health

```yaml
- alert: MessagingConsumerUnhealthy
  expr: |
    (kafka_consumergroup_members == 0)
    or
    (rate(messaging_consume_errors_total[5m]) / rate(messaging_consume_total[5m]) > 0.01)
  for: 5m
  labels:
    severity: critical
    team: "{{ $labels.team }}"
  annotations:
    summary: "Consumer unhealthy: {{ $labels.consumergroup }}"
    runbook: "https://wiki.example.com/runbooks/consumer-unhealthy"
```

### Template 2: Data Pipeline SLA

```yaml
- alert: MessagingSLABreach
  expr: |
    (
      sum(kafka_consumergroup_lag{consumergroup="order-processor"})
      / sum(rate(kafka_consumergroup_current_offset{consumergroup="order-processor"}[5m]))
    ) > 300
  for: 5m
  labels:
    severity: critical
    slo: "order-processing-5min"
  annotations:
    summary: "Order processing SLA breach — estimated lag > 5 minutes"
```

### Template 3: DLQ Monitoring

```yaml
- alert: MessagingDLQIngress
  expr: |
    rate(messaging_dlq_messages_total[15m]) > 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Messages entering DLQ: {{ $labels.source_queue }}"
    runbook: "https://wiki.example.com/runbooks/dlq-investigation"
```

### Template 4: Broker Infrastructure

```yaml
- alert: KafkaBrokerDiskHigh
  expr: |
    (node_filesystem_avail_bytes{mountpoint="/kafka"} 
    / node_filesystem_size_bytes{mountpoint="/kafka"}) < 0.2
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Kafka broker disk > 80%: {{ $labels.instance }}"
```

---

## Error Budget & Burn Rate

**Error budget (Ngân Sách Lỗi)** = phần được phép vi phạm SLO:

```
SLO: 99.9% availability (30 ngày)
Error budget: 0.1% × 30 × 24 × 60 = 4,320 phút = 43.2 phút/tháng

Nếu đã dùng 80% error budget → freeze deployments, focus reliability
```

### Burn Rate Alerting

```yaml
# Fast burn — dùng 2% budget trong 1 giờ → page ngay
- alert: SLOFastBurn
  expr: |
    (
      1 - (sum(rate(messages_processed_within_sla[1h])) 
      / sum(rate(messages_total[1h])))
    ) > (0.001 * 14.4)  # 14.4x normal burn rate
  labels:
    severity: critical

# Slow burn — dùng 10% budget trong 6 giờ → warning
- alert: SLOSlowBurn
  expr: |
    (
      1 - (sum(rate(messages_processed_within_sla[6h])) 
      / sum(rate(messages_total[6h])))
    ) > (0.001 * 6)
  labels:
    severity: warning
```

---

## Anti-Patterns

| Anti-Pattern | Hậu Quả | Fix |
| ------------ | ------- | --- |
| Alert mọi metric > threshold | Alert fatigue | Alert symptom only |
| Không có `for:` duration | False positive spike | Thêm `for: 5m` minimum |
| Warning → PagerDuty | On-call burnout | Warning → Slack |
| Không review alerts định kỳ | Stale alerts tích lũy | Monthly alert review |
| Runbook thiếu escalation | Incident kéo dài | Rõ ràng escalation path |
| Một alert cho tất cả topics | Không biết impact scope | Per-pipeline alerts |

---

## Câu Hỏi Phỏng Vấn

**Q: Làm sao thiết kế alerting cho messaging mà không bị alert fatigue?**

> Alert trên symptoms (lag > SLA, DLQ ingress) không phải causes (CPU high). Dùng `for:` duration, group alerts, inhibition rules. Warning → Slack, Critical → page. Review alerts hàng tháng, xóa alerts không actionable.

**Q: SLO cho message processing pipeline thiết kế thế nào?**

> Xác định SLI (ví dụ % message xử lý trong 5 phút), đặt SLO (99.9%), tính error budget, alert trên burn rate. Align SLO với business SLA.

**Q: Khi nào page on-call vs chỉ Slack?**

> Page khi: data loss risk, zero consumers, SLA breach sustained, payment/financial pipeline affected. Slack khi: early warning, capacity approaching limit, transient issues self-healing.

---

**Tiếp theo:** [4-prometheus-grafana.md](./4-prometheus-grafana.md) — Setup Prometheus & Grafana cho Kafka/RabbitMQ.
