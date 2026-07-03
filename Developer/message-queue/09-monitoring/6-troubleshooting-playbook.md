# Troubleshooting Playbook — Sổ Tay Xử Lý Sự Cố Messaging

> Quy trình chẩn đoán và xử lý các sự cố messaging phổ biến: Consumer lag spike (đột biến độ trễ), rebalance storm (bão tái cân bằng), disk full (đĩa đầy), DLQ flood (tràn DLQ), và message loss (mất tin).

## Mục Lục

1. [Quy Trình Troubleshooting Chung](#quy-trình-troubleshooting-chung)
2. [Incident 1: Consumer Lag Spike](#incident-1-consumer-lag-spike)
3. [Incident 2: Rebalance Storm](#incident-2-rebalance-storm)
4. [Incident 3: Broker Disk Full](#incident-3-broker-disk-full)
5. [Incident 4: DLQ Flood](#incident-4-dlq-flood)
6. [Incident 5: Message Loss Suspected](#incident-5-message-loss-suspected)
7. [Incident 6: Produce Failures](#incident-6-produce-failures)
8. [Incident 7: Consumer Stuck / Not Processing](#incident-7-consumer-stuck--not-processing)
9. [Diagnostic Commands Cheat Sheet](#diagnostic-commands-cheat-sheet)
10. [Post-Incident Template](#post-incident-template)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Quy Trình Troubleshooting Chung

```
┌─────────────────────────────────────────────────────────────────┐
│           MESSAGING INCIDENT RESPONSE FLOW                         │
│                                                                  │
│  1. DETECT    → Alert fires / user report / dashboard anomaly   │
│  2. TRIAGE    → Severity? Business impact? Scope?               │
│  3. STABILIZE → Stop bleeding (scale, pause, rollback)          │
│  4. DIAGNOSE  → Root cause analysis (metrics, logs, traces)     │
│  5. RESOLVE   → Fix root cause                                  │
│  6. VERIFY    → Metrics back to normal? SLA restored?           │
│  7. DOCUMENT  → Post-mortem, update runbook                     │
└─────────────────────────────────────────────────────────────────┘
```

### Triage Questions (Câu Hỏi Phân Loại)

| Câu Hỏi | Mục Đích |
| ------- | -------- |
| Alert nào fire? | Xác định symptom |
| Bao nhiêu consumer groups bị ảnh hưởng? | Scope: isolated vs systemic |
| Pipeline nào? Payment? Analytics? | Business impact |
| Khi nào bắt đầu? | Correlate với deployment/event |
| Có deployment gần đây không? | Regression vs infrastructure |
| Auto-recovering hay sustained? | Urgency level |

### Diagnostic Toolkit

```
Layer 1: Dashboards     → Grafana — lag, throughput, error rate
Layer 2: Metrics        → Prometheus queries — drill down
Layer 3: Logs           → ELK/Loki — consumer errors, broker warnings
Layer 4: Traces         → Jaeger — end-to-end latency breakdown
Layer 5: CLI            → kafka-consumer-groups, rabbitmqctl
Layer 6: Broker UI      → Kafka UI, RabbitMQ Management
```

---

## Incident 1: Consumer Lag Spike

### Symptoms (Triệu Chứng)

- Alert: `KafkaConsumerLagHigh` hoặc `RabbitMQQueueDepthHigh`
- Dashboard: lag tăng đột ngột hoặc tăng liên tục
- Business: orders/notifications delayed

### Diagnosis Flow

```
Lag spike detected
    │
    ├── Q1: Active consumers = 0?
    │   YES → Incident 7 (Consumer Stuck)
    │   NO ↓
    │
    ├── Q2: Produce rate spike?
    │   YES → Traffic burst (có thể bình thường)
    │         → Scale consumers nếu lag không tự giảm
    │   NO ↓
    │
    ├── Q3: Consume rate drop?
    │   YES → Consumer slow
    │         ├── Processing latency tăng? → Downstream issue
    │         ├── Rebalance events? → Incident 2
    │         └── Recent deployment? → Rollback
    │   NO ↓
    │
    ├── Q4: Hot partition?
    │   YES → Key skew — một partition lag cao hơn hẳn
    │         → Investigate partition key distribution
    │   NO ↓
    │
    └── Q5: All partitions lag đều?
        YES → Systemic issue — broker slow, network, downstream
```

### Resolution Steps

| Root Cause | Action | Verify |
| ---------- | ------ | ------ |
| Consumer dead | Restart pods/processes | `kafka_consumergroup_members > 0` |
| Consumer slow | Scale out consumers | Lag decreasing |
| Downstream timeout | Fix DB/API, circuit breaker | Processing latency normal |
| Traffic burst | Scale consumers temporarily | Lag stabilizes |
| Hot partition | Revisit partition key strategy | Per-partition lag even |
| Code regression | Rollback deployment | Error rate drops |

### Prometheus Queries

```promql
# Consume rate drop?
rate(kafka_consumergroup_current_offset{consumergroup="order-processor"}[5m])

# Produce rate spike?
sum(rate(kafka_server_BrokerTopicMetrics_MessagesInPerSec{topic="orders"}[5m]))

# Processing latency?
histogram_quantile(0.95, rate(messaging_consume_duration_seconds_bucket[5m]))
```

---

## Incident 2: Rebalance Storm

### Symptoms

- Frequent rebalance events trong logs
- Throughput drop đột ngột, lag spike tạm thời
- Consumer logs: `Revoke previously assigned partitions`, `Rebalance in progress`

### Root Causes

| Nguyên Nhân | Dấu Hiệu |
| ----------- | -------- |
| **Consumer join/leave liên tục** | K8s pod restart loop, health check fail |
| **session.timeout.ms quá thấp** | Consumer processing > session timeout |
| **max.poll.interval.ms exceeded** | Long-running message processing |
| **Coordinator issue** | Broker coordinator unavailable |
| **Rolling deployment** | Expected — nhưng nên minimize impact |

### Diagnosis

```bash
# Kafka — check consumer group state
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --describe --group order-processor --state

# Check rebalance frequency in logs
grep -i "rebalance" /var/log/consumer.log | tail -50

# Metrics
rate(kafka_consumer_rebalance_total[1h])
```

### Resolution

```properties
# Tăng timeouts nếu processing lâu
session.timeout.ms=45000          # default 10000
max.poll.interval.ms=300000       # default 300000 (5 min)
heartbeat.interval.ms=15000       # < session.timeout/3

# Cooperative rebalancing (ít disruptive hơn)
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

```
Deployment best practices:
1. Scale gradually — không add/remove nhiều consumers cùng lúc
2. Dùng static membership (group.instance.id) nếu có thể
3. Rolling deploy với maxUnavailable=1
4. Pause consumption trước khi deploy nếu critical pipeline
```

---

## Incident 3: Broker Disk Full

### Symptoms

- Alert: `KafkaBrokerDiskHigh` hoặc RabbitMQ disk alarm
- Produce failures: `NOT_ENOUGH_REPLICAS`, `disk full`
- RabbitMQ: publishers blocked

### Diagnosis

```bash
# Kafka — disk usage per broker
df -h /var/kafka-logs

# Kafka — largest topics/partitions
du -sh /var/kafka-logs/*/ | sort -rh | head -20

# RabbitMQ
rabbitmqctl status | grep -A5 "Disk"
```

```promql
# Prometheus
(node_filesystem_size_bytes - node_filesystem_avail_bytes) 
/ node_filesystem_size_bytes{mountpoint="/kafka"}

kafka_log_Log_Size
```

### Resolution (Ưu Tiên)

| Action | Impact | Khi Nào |
| ------ | ------ | ------- |
| **Giảm retention** | Mất data cũ hơn retention mới | Emergency — cần space ngay |
| **Xóa unused topics** | Mất data topic đó | Topic không còn dùng |
| **Expand disk** | Không mất data | Có thể expand volume |
| **Add broker + rebalance** | Không mất data | Long-term capacity |
| **Enable compression** | Giảm size tương lai | Chưa enable compression |

```bash
# Emergency — giảm retention (Kafka)
kafka-configs --alter --entity-type topics --entity-name old-topic \
  --add-config retention.ms=3600000  # 1 giờ tạm thời

# Cleanup log segments
kafka-log-dirs --bootstrap-server kafka:9092 --describe | grep -v "size: 0"
```

### Prevention

- Monitor disk với alert ở 70% (warning) và 85% (critical)
- Retention policy phù hợp với use case
- Capacity planning: disk growth rate × retention period

---

## Incident 4: DLQ Flood

### Symptoms

- Alert: `MessagingDLQIngress` sustained
- DLQ depth tăng liên tục
- Business: orders/events không được xử lý

### Diagnosis Flow

```
DLQ ingress detected
    │
    ├── Q1: Tất cả messages hay subset?
    │   ALL → Systemic (schema change, downstream down)
    │   SUBSET → Specific message type/format issue
    │
    ├── Q2: Error type trong logs?
    │   Deserialization → Schema mismatch
    │   Timeout → Downstream slow/down
    │   Business exception → Logic bug, bad data
    │
    ├── Q3: Bắt đầu khi nào?
    │   Correlate với deployment/schema change
    │
    └── Q4: DLQ message sample?
        Inspect 5-10 messages — pattern?
```

### Resolution

| Root Cause | Action |
| ---------- | ------ |
| Schema mismatch | Fix schema, redeploy consumer, replay DLQ |
| Downstream down | Fix downstream, circuit breaker recovery, replay |
| Bad data từ producer | Fix producer, quarantine bad messages |
| Logic bug | Fix code, deploy, replay DLQ |
| Poison message | Quarantine specific messages — xem [06-reliability/4-poison-message.md](../06-reliability/4-poison-message.md) |

```bash
# Inspect DLQ messages (Kafka)
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic orders-dlq --from-beginning --max-messages 10

# Replay DLQ sau khi fix
# 1. Verify fix với sample messages
# 2. Replay tool hoặc custom reprocessor
# 3. Monitor DLQ ingress = 0
```

---

## Incident 5: Message Loss Suspected

### Symptoms

- Business report thiếu events
- Audit gap trong event log
- Producer success nhưng consumer không nhận

### Investigation Checklist

```markdown
- [ ] Producer acks setting? (acks=0 có thể mất message)
- [ ] Consumer auto-commit trước khi process xong?
- [ ] Retention expired trước khi consumer đọc?
- [ ] Consumer group reset offset nhầm?
- [ ] Topic deleted/recreated?
- [ ] Replication factor = 1 và broker crash?
- [ ] Network partition → unclean leader election?
```

### Diagnostic Commands

```bash
# Verify message exists on broker
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic orders --partition 0 --offset 12345 --max-messages 1

# Check consumer committed offset vs log end
kafka-consumer-groups --describe --group order-processor

# Check topic retention
kafka-configs --describe --entity-type topics --entity-name orders
```

### Common Root Causes

| Scenario | Mechanism | Prevention |
| -------- | --------- | ---------- |
| `acks=0` produce | Fire-and-forget | `acks=all` cho critical data |
| Auto-commit before process | Offset committed, crash before process | Manual commit after process |
| Retention < consumer lag | Old messages deleted | Monitor lag vs retention |
| RF=1 broker crash | No replica | RF >= 3 |
| Offset reset to latest | Missed messages during downtime | `auto.offset.reset=earliest` + alert |

---

## Incident 6: Produce Failures

### Symptoms

- Producer error rate tăng
- Application logs: `NOT_ENOUGH_REPLICAS`, `TOPIC_AUTHORIZATION_FAILED`, `RECORD_TOO_LARGE`

### Quick Reference

| Error | Nguyên Nhân | Fix |
| ----- | ----------- | --- |
| `NOT_ENOUGH_REPLICAS` | ISR < min.insync.replicas | Check broker health, disk |
| `TOPIC_AUTHORIZATION_FAILED` | ACL deny | Fix ACL permissions |
| `RECORD_TOO_LARGE` | Message > max.message.bytes | Increase limit hoặc chunk message |
| `UNKNOWN_TOPIC_OR_PARTITION` | Topic chưa tạo | Create topic |
| `REQUEST_TIMED_OUT` | Broker overload/network | Check broker metrics |
| `BROKER_NOT_AVAILABLE` | Broker down | Check broker health |

```bash
# Check ISR
kafka-topics --describe --topic orders

# Check ACL
kafka-acls --list --principal User:order-service
```

---

## Incident 7: Consumer Stuck / Not Processing

### Symptoms

- `kafka_consumergroup_members == 0` nhưng lag > 0
- Messages trong queue nhưng không ai consume
- Consumer process running nhưng không poll

### Diagnosis

```bash
# Kafka — consumer group members
kafka-consumer-groups --describe --group order-processor --members

# Check consumer process
kubectl get pods -l app=order-processor
kubectl logs order-processor-xxx --tail=100

# RabbitMQ — consumers per queue
rabbitmqctl list_queues name messages consumers
```

### Common Causes

| Cause | Fix |
| ----- | --- |
| Pod crash loop | Check logs, fix startup error |
| OOM killed | Increase memory limit |
| Stuck in rebalance | Restart consumer, check coordinator |
| Wrong consumer group | Fix group ID config |
| Subscription mismatch | Consumer subscribe wrong topic |
| Thread blocked | Deadlock, infinite loop in handler |
| K8s HPA scaled to 0 | Check HPA min replicas |

---

## Diagnostic Commands Cheat Sheet

### Kafka

```bash
# Consumer group lag
kafka-consumer-groups --bootstrap-server $BROKER \
  --describe --group $GROUP

# Topic details
kafka-topics --bootstrap-server $BROKER --describe --topic $TOPIC

# List consumer groups
kafka-consumer-groups --bootstrap-server $BROKER --list

# Reset offset (CAREFUL!)
kafka-consumer-groups --bootstrap-server $BROKER \
  --group $GROUP --topic $TOPIC --reset-offsets --to-datetime 2024-01-01T00:00:00.000 \
  --execute

# Broker configs
kafka-configs --bootstrap-server $BROKER \
  --describe --entity-type brokers --entity-name 0
```

### RabbitMQ

```bash
# Queue status
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers

# Connections
rabbitmqctl list_connections name state channels

# Node health
rabbitmqctl status

# Purge queue (CAREFUL!)
rabbitmqctl purge_queue order-processing
```

### Prometheus Quick Queries

```promql
# Total lag
sum(kafka_consumergroup_lag{consumergroup="order-processor"})

# Error rate
rate(messaging_consume_errors_total[5m]) / rate(messaging_consume_total[5m])

# DLQ depth
messaging_dlq_messages_total

# Under-replicated
kafka_server_ReplicaManager_UnderReplicatedPartitions
```

---

## Post-Incident Template

```markdown
# Post-Mortem: [Incident Title]

**Date:** YYYY-MM-DD
**Duration:** X hours
**Severity:** P1/P2/P3
**Author:** [Name]

## Summary
[1-2 câu mô tả sự cố và impact]

## Impact
- [ ] Business impact: X orders delayed
- [ ] SLA breach: Yes/No
- [ ] Data loss: Yes/No
- [ ] Users affected: ~N

## Timeline (UTC)
| Time | Event |
| ---- | ----- |
| HH:MM | Alert fired: KafkaConsumerLagHigh |
| HH:MM | On-call acknowledged |
| HH:MM | Root cause identified |
| HH:MM | Fix applied |
| HH:MM | Metrics normalized |

## Root Cause
[Technical root cause — 5 Whys]

## Resolution
[What was done to fix]

## What Went Well
- [ ] Alert detected quickly
- [ ] Runbook helpful

## What Went Wrong
- [ ] Missing alert for X
- [ ] Runbook outdated

## Action Items
| Action | Owner | Due Date |
| ------ | ----- | -------- |
| Add alert for X | @team | YYYY-MM-DD |
| Update runbook | @team | YYYY-MM-DD |
| Capacity review | @team | YYYY-MM-DD |
```

---

## Câu Hỏi Phỏng Vấn

**Q: Consumer lag spike 3AM — bạn làm gì đầu tiên?**

> 1. Acknowledge alert, check dashboard scope (một group hay tất cả?)  
> 2. Check active consumers — có consumer alive không?  
> 3. Check produce vs consume rate — traffic burst hay consumer slow?  
> 4. Check recent deployments  
> 5. Stabilize (restart/scale) trước, root cause sau nếu P1

**Q: Làm sao phân biệt consumer slow vs producer burst?**

> So sánh produce rate và consume rate trên cùng time window. Producer burst: produce rate spike, consume rate bình thường, lag tăng rồi tự giảm. Consumer slow: consume rate drop, processing latency tăng, lag tăng sustained.

**Q: Kể về incident messaging bạn từng xử lý (STAR)?**

> Chuẩn bị câu chuyện thực tế: Situation (lag spike/DLQ flood), Task (restore SLA), Action (diagnosis steps cụ thể), Result (MTTR, prevention measures added).

---

**Tiếp theo:** [7-production-checklist.md](./7-production-checklist.md) — Checklist trước go-live.
