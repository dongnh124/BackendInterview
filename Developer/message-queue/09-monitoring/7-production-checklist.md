# Production Checklist — Checklist Trước Go-Live & Vận Hành

> Checklist toàn diện trước khi đưa messaging pipeline lên production: monitoring setup, alerting, runbooks, capacity, và operational readiness (sẵn sàng vận hành).

## Mục Lục

1. [Pre-Deployment Checklist](#pre-deployment-checklist)
2. [Monitoring & Observability](#monitoring--observability)
3. [Alerting & On-Call](#alerting--on-call)
4. [Reliability & Error Handling](#reliability--error-handling)
5. [Performance & Capacity](#performance--capacity)
6. [Security](#security)
7. [Documentation & Runbooks](#documentation--runbooks)
8. [Go-Live Day Checklist](#go-live-day-checklist)
9. [Post-Go-Live (Tuần 1)](#post-go-live-tuần-1)
10. [Ongoing Operations](#ongoing-operations)

---

## Pre-Deployment Checklist

### Tổng Quan Pipeline

```markdown
## Pipeline: [Tên Pipeline — ví dụ: Order Processing]

### Architecture Review
- [ ] Topic/queue naming convention documented
- [ ] Partition/queue count justified (throughput, ordering needs)
- [ ] Consumer group naming convention
- [ ] Message schema versioned (Schema Registry nếu dùng Avro/Protobuf)
- [ ] Idempotency key defined cho mọi consumer
- [ ] DLQ topic/queue configured
- [ ] Retry strategy documented (max retries, backoff)
- [ ] Delivery semantics explicit (at-least-once expected)
```

---

## Monitoring & Observability

### Metrics — Bắt Buộc

```markdown
## Broker Metrics
- [ ] kafka_exporter hoặc JMX Exporter deployed (Kafka)
- [ ] rabbitmq_prometheus plugin enabled (RabbitMQ)
- [ ] Prometheus scrape targets healthy
- [ ] Node exporter cho broker hosts

## Consumer Metrics
- [ ] Consumer lag per group/topic/partition
- [ ] Consume rate (messages/sec)
- [ ] Processing latency (P50, P95, P99)
- [ ] Error rate (consume failures)
- [ ] Active consumer count

## Producer Metrics
- [ ] Produce rate (messages/sec)
- [ ] Produce error rate
- [ ] Produce latency P99

## DLQ Metrics
- [ ] DLQ depth (current count)
- [ ] DLQ ingress rate
- [ ] DLQ oldest message age

## Business Metrics
- [ ] Business KPI instrumented (orders/min, events processed)
- [ ] SLA compliance metric
```

### Dashboards

```markdown
## Grafana Dashboards
- [ ] Messaging overview dashboard created
- [ ] Per-pipeline dashboard (producer + consumer + DLQ)
- [ ] Broker health dashboard
- [ ] Dashboard accessible cho on-call team
- [ ] Dashboard links trong runbook
- [ ] Tested với real traffic (không empty panels)
```

### Logging

```markdown
## Structured Logging
- [ ] Correlation ID trong mọi log entry
- [ ] Log level appropriate (INFO prod, DEBUG staging)
- [ ] Consumer errors logged với message metadata (không log PII)
- [ ] Log aggregation setup (ELK/Loki/CloudWatch)
- [ ] Log search by correlation ID tested
```

### Tracing (Khuyến Nghị)

```markdown
## Distributed Tracing
- [ ] Correlation ID propagate qua message headers
- [ ] OpenTelemetry instrumentation (nếu có)
- [ ] Trace backend accessible (Jaeger/Tempo)
- [ ] Sample rate configured (1-10% prod)
```

---

## Alerting & On-Call

### Alert Rules — Bắt Buộc

```markdown
## Critical Alerts (Page On-Call)
- [ ] Zero active consumers + lag > 0
- [ ] Consumer lag > SLA critical threshold (sustained 5m+)
- [ ] UnderReplicatedPartitions > 0 (Kafka, sustained)
- [ ] Broker disk > 90%
- [ ] DLQ ingress sustained > 15m (critical pipelines)

## Warning Alerts (Slack)
- [ ] Consumer lag > warning threshold
- [ ] Lag growth rate positive sustained 15m+
- [ ] Broker disk > 80%
- [ ] Error rate > 0.1%
- [ ] Rebalance events > threshold/hour
- [ ] Hot partition detected

## Alert Quality
- [ ] Mỗi alert có `for:` duration (minimum 5m)
- [ ] Mỗi alert có runbook link trong annotation
- [ ] Alert routing configured (critical → PagerDuty, warning → Slack)
- [ ] Alert grouping configured (Alertmanager)
- [ ] Test alert fired successfully (amtool test)
- [ ] On-call rotation configured
- [ ] Escalation path documented
```

### SLO Definition

```markdown
## Service Level Objectives
- [ ] SLI defined: [metric đo chất lượng]
- [ ] SLO target: [ví dụ: 99.9% messages processed < 5 min]
- [ ] Error budget calculated
- [ ] SLO dashboard created
- [ ] Burn rate alerts configured (optional, advanced)
```

---

## Reliability & Error Handling

```markdown
## Delivery & Error Handling
- [ ] At-least-once delivery confirmed
- [ ] Idempotent consumer implemented & tested
- [ ] Manual offset commit (after successful processing)
- [ ] Retry với exponential backoff + jitter
- [ ] Max retry count defined
- [ ] DLQ configured và tested
- [ ] DLQ replay procedure documented
- [ ] Poison message handling plan
- [ ] Circuit breaker cho downstream calls (nếu applicable)

## Failure Testing
- [ ] Tested: kill consumer → lag spike → recovery
- [ ] Tested: downstream timeout → retry → DLQ
- [ ] Tested: duplicate message → idempotent handling
- [ ] Tested: broker restart → consumer reconnect
- [ ] Tested: message schema mismatch → DLQ
```

---

## Performance & Capacity

```markdown
## Capacity Planning
- [ ] Expected throughput documented (messages/sec peak)
- [ ] Partition count >= expected max consumers
- [ ] Broker disk sized: daily_volume × retention × replication_factor × 1.3
- [ ] Consumer resources sized (CPU, memory per pod)
- [ ] Load test completed với 2x expected peak
- [ ] HPA configured (nếu K8s) với appropriate metrics

## Performance Baseline
- [ ] Baseline metrics recorded (lag, throughput, latency)
- [ ] P99 processing latency documented
- [ ] Produce/consume rate at normal load documented
```

---

## Security

```markdown
## Security Checklist (chi tiết: 08-security/)
- [ ] Authentication enabled (SASL/SCRAM, mTLS)
- [ ] ACL/RBAC configured — least privilege
- [ ] TLS in-transit enabled
- [ ] Credentials in Secret Manager (không hardcode)
- [ ] Monitoring user có read-only permissions
- [ ] Network isolation (private subnet)
- [ ] Audit logging enabled
```

---

## Documentation & Runbooks

```markdown
## Documentation
- [ ] Architecture diagram (producer → broker → consumer → downstream)
- [ ] Topic/queue inventory với owner team
- [ ] Message schema documentation
- [ ] Consumer group ownership documented
- [ ] On-call runbook cho mỗi critical alert
- [ ] DLQ replay runbook
- [ ] Rollback procedure
- [ ] Contact list (service owners, platform team)

## Runbook Minimum Content
Mỗi runbook phải có:
- [ ] Symptoms (alert name, dashboard link)
- [ ] Business impact description
- [ ] Diagnosis steps (numbered, checkable)
- [ ] Resolution steps
- [ ] Escalation path
- [ ] Post-incident actions
```

---

## Go-Live Day Checklist

```markdown
## Go-Live Day — [DATE]

### Pre-Launch (T-2 hours)
- [ ] All checklist items above completed
- [ ] Staging tested end-to-end với production-like load
- [ ] On-call engineer identified và available
- [ ] Rollback plan ready
- [ ] Communication sent to stakeholders

### Launch (T-0)
- [ ] Deploy producer (canary nếu có thể)
- [ ] Verify produce metrics flowing
- [ ] Deploy consumer
- [ ] Verify consume metrics, lag = 0 or decreasing
- [ ] Verify DLQ empty
- [ ] Smoke test: send test message → verify processed

### Post-Launch (T+1 hour)
- [ ] Monitor dashboard continuously
- [ ] Lag stable?
- [ ] Error rate = 0?
- [ ] No unexpected alerts?
- [ ] Business team confirms data flowing

### Post-Launch (T+24 hours)
- [ ] Review metrics vs baseline
- [ ] Any alerts fired? False positives?
- [ ] DLQ status?
- [ ] Team retrospective (quick)
```

---

## Post-Go-Live (Tuần 1)

```markdown
## Week 1 Review

### Day 3
- [ ] Review alert history — any false positives?
- [ ] Adjust thresholds nếu cần
- [ ] Verify SLO compliance

### Day 7
- [ ] Capacity review: actual vs projected throughput
- [ ] Lag patterns during peak hours documented
- [ ] Update runbooks based on learnings
- [ ] Team knowledge share session
- [ ] Mark pipeline as "production stable" hoặc note issues
```

---

## Ongoing Operations

### Hàng Ngày

```markdown
- [ ] Glance messaging overview dashboard
- [ ] Check DLQ depth (should be 0 or stable low)
- [ ] Review any overnight alerts
```

### Hàng Tuần

```markdown
- [ ] Review consumer lag trends
- [ ] Review error rate trends
- [ ] Check broker disk growth rate
- [ ] Review DLQ messages (if any) — root cause
```

### Hàng Tháng

```markdown
- [ ] Alert review — remove/fix noisy alerts
- [ ] SLO/error budget review
- [ ] Capacity planning update
- [ ] Runbook review và update
- [ ] Credential rotation (nếu scheduled)
- [ ] Disaster recovery drill (quarterly)
```

### Hàng Quý

```markdown
- [ ] Full incident retrospective review
- [ ] Load test với updated traffic projections
- [ ] Security audit (ACL review)
- [ ] Architecture review — tech debt
- [ ] Update post-mortem learnings vào runbooks
```

---

## Quick Reference Card

In và dán lên desk on-call:

```
╔══════════════════════════════════════════════════════════════╗
║           MESSAGING ON-CALL QUICK REFERENCE                   ║
╠══════════════════════════════════════════════════════════════╣
║ DASHBOARD:  https://grafana.example.com/d/messaging-overview ║
║ RUNBOOKS:   https://wiki.example.com/runbooks/messaging       ║
║ ESCALATE:   #platform-oncall → PagerDuty platform-team        ║
╠══════════════════════════════════════════════════════════════╣
║ LAG SPIKE:                                                    ║
║   1. Active consumers? → restart if 0                       ║
║   2. Produce spike? → scale consumers                         ║
║   3. Processing slow? → check downstream                      ║
║   4. Hot partition? → check key skew                          ║
╠══════════════════════════════════════════════════════════════╣
║ DLQ FLOOD:                                                    ║
║   1. Sample DLQ messages                                      ║
║   2. Check error logs                                         ║
║   3. Schema/downstream/deployment change?                     ║
║   4. Fix → replay DLQ                                         ║
╠══════════════════════════════════════════════════════════════╣
║ DISK FULL:                                                    ║
║   1. Check largest topics                                     ║
║   2. Reduce retention (emergency)                               ║
║   3. Expand disk / add broker                                 ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Sign-Off Template

```markdown
# Production Readiness Sign-Off

**Pipeline:** _______________________
**Team:** _______________________
**Target Go-Live:** _______________________

| Area | Owner | Sign-Off | Date |
| ---- | ----- | -------- | ---- |
| Monitoring & Dashboards | | [ ] | |
| Alerting & On-Call | | [ ] | |
| Reliability & DLQ | | [ ] | |
| Performance & Load Test | | [ ] | |
| Security | | [ ] | |
| Documentation & Runbooks | | [ ] | |
| Go-Live Plan | | [ ] | |

**Approved by Engineering Lead:** _________________ Date: _______
```

---

**Liên quan:**
- [README.md](./README.md) — Tổng quan monitoring
- [6-troubleshooting-playbook.md](./6-troubleshooting-playbook.md) — Xử lý sự cố
- [08-security/README.md](../08-security/README.md) — Security checklist
- [06-reliability/README.md](../06-reliability/README.md) — Reliability patterns
