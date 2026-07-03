# Audit Logging — Ghi Nhật Ký Kiểm Toán & Compliance

> Audit Logging (Ghi Nhật Ký Kiểm Toán) ghi lại **ai làm gì, khi nào, trên resource nào** — bắt buộc cho incident response (phản ứng sự cố), compliance (tuân thủ), và forensic analysis (phân tích pháp y).

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Tại Sao Audit Logging Quan Trọng](#tại-sao-audit-logging-quan-trọng)
3. [Events Cần Audit](#events-cần-audit)
4. [Kafka Audit Logging](#kafka-audit-logging)
5. [RabbitMQ Audit Logging](#rabbitmq-audit-logging)
6. [Cloud Managed Audit Trails](#cloud-managed-audit-trails)
7. [SIEM Integration](#siem-integration)
8. [Compliance Frameworks](#compliance-frameworks)
9. [PII Trong Messages](#pii-trong-messages)
10. [Audit Log Retention & Protection](#audit-log-retention--protection)
11. [Anti-Patterns](#anti-patterns)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              AUDIT LOGGING — QUICK REFERENCE                       │
├─────────────────────────────────────────────────────────────────┤
│  WHO      → Principal (user, service account, API key)          │
│  WHAT     → Action (CREATE topic, ALTER ACL, DELETE schema)     │
│  WHEN     → Timestamp (UTC, ISO 8601)                           │
│  WHERE    → Resource (topic name, cluster, vhost)               │
│  RESULT   → Success / Failure / Denied                          │
│  Retention→ 1–7 năm tùy compliance (PCI: 1yr, SOC2: theo policy)│
└─────────────────────────────────────────────────────────────────┘
```

| Mục Đích | Ví Dụ |
| -------- | ----- |
| **Security incident** | Ai tạo ACL cho phép READ topic PII? |
| **Compliance audit** | Chứng minh access control hoạt động |
| **Change tracking** | Ai xóa topic production lúc 2AM? |
| **Forensics** | Trace data breach timeline |

---

## Tại Sao Audit Logging Quan Trọng

Message broker là **central data hub** — mọi thay đổi cấu hình và access đều có impact rộng.

```
Scenario không có audit log:
  - Topic "customer-pii" bị expose → không biết ai thêm ACL
  - Schema breaking change → không trace ai register schema v2
  - Broker config thay đổi → không correlate với incident

Scenario có audit log:
  - Alert: ACL change on "customer-pii" at 03:42 UTC
  - Trace: User:contractor-temp added READ for User:analytics-ext
  - Action: Revoke ACL, rotate credentials, notify security team
```

### Audit vs Application Logging

| Loại | Nội Dung | Ví Dụ |
| ---- | -------- | ----- |
| **Application log** | Business logic, errors | "Order ord-123 processed" |
| **Broker operational log** | Broker health, replication | "Partition 3 under-replicated" |
| **Audit log** | Security-relevant actions | "User:admin ALTER ACL topic:orders" |

> **Quy tắc:** Audit log **immutable (bất biến)** — không cho phép sửa/xóa bởi admin thường. Ship sang SIEM (Security Information and Event Management — Quản Lý Thông Tin và Sự Kiện Bảo Mật) riêng.

---

## Events Cần Audit

### Tier 1 — Bắt Buộc (Critical)

| Event | Tại Sao |
| ----- | ------- |
| **Authentication success/failure** | Detect brute force, credential leak |
| **ACL create/alter/delete** | Permission change = security boundary change |
| **Topic/queue create/delete** | Data surface area change |
| **User/service account create/delete** | Identity lifecycle |
| **Schema register/delete** | Contract change, breaking change risk |
| **Admin/super-user actions** | Privileged access |

### Tier 2 — Nên Có (Important)

| Event | Tại Sao |
| ----- | ------- |
| **Consumer group join/leave** | Detect unauthorized consumers |
| **Config change (broker, topic)** | Retention, replication change |
| **Certificate upload/rotation** | TLS infrastructure change |
| **Quota/limit change** | DoS (Denial of Service — Từ Chối Dịch Vụ) risk |

### Tier 3 — Tùy Chọn (Nice to Have)

| Event | Tại Sao |
| ----- | ------- |
| **Message produce/consume (metadata only)** | Data access pattern — volume lớn |
| **Connection open/close** | Network anomaly detection |

```
Khuyến nghị: Tier 1 + Tier 2 cho production.
Tier 3 metadata-only (không log message body) nếu compliance yêu cầu.
```

---

## Kafka Audit Logging

### Authorizer Audit Log

Khi enable ACL authorizer, Kafka log mọi authorization decision:

```properties
# server.properties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false

# Super users — actions vẫn nên audit
super.users=User:admin
```

**Log format mẫu:**

```
INFO Principal User:payment-service is Denied operation Read
     from host 10.0.1.50 on resource Topic:customer-pii

INFO Principal User:admin is Allowed operation Alter
     from host 10.0.0.10 on resource Topic:orders
```

### Kafka Admin API Audit

Mọi thao tác qua Admin API nên được log:

```bash
# Các operations cần audit:
kafka-topics.sh --create / --delete / --alter
kafka-acls.sh --add / --remove / --list
kafka-configs.sh --alter
kafka-configs.sh --alter --entity-type users  # SCRAM credential change
```

### Open-source Kafka — Custom Audit

Apache Kafka OSS **không có built-in audit log đầy đủ**. Options:

| Approach | Mô Tả |
| -------- | ----- |
| **Log4j appenders** | Parse broker logs → ship to SIEM |
| **Confluent Audit Logs** | Enterprise feature — full audit |
| **Custom authorizer** | Wrap AclAuthorizer, log mọi decision |
| **Proxy sidecar** | API gateway log admin requests |

```java
// Custom authorizer pattern (conceptual)
public class AuditingAuthorizer extends AclAuthorizer {
  @Override
  public AuthorizationResult authorize(AuthorizableRequestContext context, Action action) {
    AuthorizationResult result = super.authorize(context, action);
    auditLogger.log(context.principal(), action, result);
    return result;
  }
}
```

### Confluent Platform Audit Logs

```
Confluent audit log categories:
  - ADMIN     → cluster/topic/user management
  - AUTHORIZE → ACL allow/deny decisions
  - DESCRIBE  → metadata read (optional)
  - AUTHENTICATE → login success/failure

Destination: Kafka topic _confluent-audit-log-*, HTTP endpoint, S3
Retention: configurable, default 90 days
```

---

## RabbitMQ Audit Logging

### Built-in Event Log

RabbitMQ log connection, channel, và management API events:

```ini
# rabbitmq.conf
log.console = true
log.console.level = info
log.file.level = info

# Management plugin audit
management.listener.ssl = true
```

**Events logged:**

```
=INFO REPORT==== Connection accepted: 10.0.1.50:52341 -> 10.0.0.5:5671
=INFO REPORT==== user 'order-service' authenticated and granted access to vhost '/orders'
=WARNING REPORT==== HTTP access denied: user 'guest' - invalid credentials
=INFO REPORT==== queue 'orders.processing' created on vhost '/orders'
```

### RabbitMQ Management API Audit

```
Management API (port 15672) actions cần audit:
  PUT  /api/users/{name}           → create/update user
  DELETE /api/users/{name}         → delete user
  PUT  /api/permissions/...        → set permissions
  PUT  /api/policies/...           → set policy
  DELETE /api/queues/...           → delete queue
```

### RabbitMQ Prometheus + External Audit

```
Pattern:
  1. Enable RabbitMQ JSON logging
  2. Filebeat/Fluentd ship logs → Elasticsearch
  3. Kibana dashboard + alert rules
  4. Alert: permission change, failed auth spike
```

---

## Cloud Managed Audit Trails

### AWS MSK — CloudTrail

```
CloudTrail events:
  - CreateCluster, DeleteCluster
  - UpdateBrokerCount, UpdateStorage
  - BatchAssociateScramSecret (SCRAM credential change)

MSK không log Kafka-level ACL changes qua CloudTrail —
cần enable Kafka authorizer logs riêng.
```

```json
{
  "eventName": "CreateCluster",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "platform-admin"
  },
  "eventTime": "2026-07-03T10:30:00Z",
  "requestParameters": {
    "ClusterName": "production-kafka"
  }
}
```

### Confluent Cloud — Audit Log

```
Confluent Cloud audit log (automatic):
  - Organization/environment/cluster changes
  - API key create/revoke
  - RBAC role assignment
  - Schema Registry changes
  - Connector deploy/delete

Export: Confluent Cloud → S3, Splunk, Datadog
```

### Azure Event Hubs — Activity Log

```
Azure Monitor Activity Log:
  - Namespace create/delete
  - Authorization rule changes
  - Network rule changes
  - Diagnostic settings (capture config)
```

### GCP Pub/Sub — Audit Logs

```
Cloud Audit Logs:
  - Admin Activity → always logged
  - Data Access → optional (message publish/subscribe metadata)
  
Enable data access logs nếu compliance yêu cầu track ai publish/subscribe.
```

---

## SIEM Integration

**SIEM (Security Information and Event Management — Quản Lý Thông Tin và Sự Kiện Bảo Mật)** tập trung audit logs từ nhiều nguồn.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Kafka   │  │ RabbitMQ │  │ CloudTrail│
│ audit log│  │ event log│  │          │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   ▼
          ┌────────────────┐
          │ Log Aggregator │  (Fluentd, Filebeat, Vector)
          └───────┬────────┘
                  ▼
          ┌────────────────┐
          │     SIEM       │  (Splunk, Elastic SIEM, Datadog)
          └───────┬────────┘
                  ▼
          ┌────────────────┐
          │ Alert Rules    │
          └────────────────┘
```

### Alert Rules Quan Trọng

| Rule | Trigger | Severity |
| ---- | ------- | -------- |
| **ACL change on PII topic** | ALTER ACL + topic matches `*-pii*` | CRITICAL |
| **Auth failure spike** | >10 failures/min per user | HIGH |
| **Admin action off-hours** | Admin API call outside 9–18 UTC | MEDIUM |
| **New super-user** | super.users config change | CRITICAL |
| **Topic delete production** | DELETE topic + env=prod tag | CRITICAL |
| **Schema compatibility break** | REGISTER schema + compatibility FAIL | HIGH |

### Correlation ID Trong Audit

```
Audit entry nên chứa:
  correlation_id  → trace xuyên suốt request
  source_ip       → client IP
  user_agent      → CLI tool / SDK version
  request_id      → unique per API call
  environment     → prod/staging/dev
```

---

## Compliance Frameworks

### GDPR (General Data Protection Regulation — Quy Định Bảo Vệ Dữ Liệu Chung)

| Yêu Cầu | Messaging Implication |
| ------- | --------------------- |
| **Right to access** | Audit ai đã đọc data subject's messages |
| **Right to erasure** | Kafka immutable log — khó xóa; cần retention policy + tombstone |
| **Data minimization** | Không gửi PII không cần thiết trong message |
| **Accountability** | Audit log chứng minh access control |

```
GDPR checklist cho messaging:
  □ PII inventory — topics nào chứa PII?
  □ Retention policy — auto-delete sau X ngày
  □ ACL restrict access to PII topics
  □ Audit log ai access PII topics
  □ DPA (Data Processing Agreement) với cloud provider
  □ Encryption in-transit + at-rest
```

### PCI-DSS (Payment Card Industry Data Security Standard)

```
Requirement 10 — Track and monitor all access:
  □ Log all access to cardholder data environment
  □ Audit trails immutable
  □ Retention minimum 1 year
  □ Daily log review (automated SIEM)

Messaging: KHÔNG gửi full PAN (Primary Account Number) trong message.
          Chỉ gửi tokenized reference.
```

### SOC 2 Type II

```
Trust Service Criteria — Logical Access:
  □ Unique user identification
  □ Access revoked upon termination
  □ Privileged access logged and reviewed
  □ Change management for ACL/config

Audit log retention: thường 1–3 năm theo policy.
```

### HIPAA (Health Insurance Portability and Accountability Act)

```
Nếu message chứa PHI (Protected Health Information):
  □ BAA (Business Associate Agreement) với broker provider
  □ Encryption bắt buộc (in-transit + at-rest)
  □ Access logs retained 6 years
  □ Minimum necessary — chỉ gửi PHI cần thiết
```

---

## PII Trong Messages

Message content audit là **vùng xám** — log body = thêm PII vào log system.

### Best Practices

```
❌ KHÔNG log message body trong audit log
❌ KHÔNG log full credit card, SSN, password

✅ Log metadata only:
   topic, partition, offset, message size, timestamp
   principal (ai produce/consume), correlation_id

✅ Nếu cần data access audit:
   log: "User:analytics-svc READ topic:customer-events offset:12345-67890"
   KHÔNG log: message content
```

### PII Minimization Pattern

```
❌ Message chứa PII:
  { orderId, customerName, email, phone, address, cardLast4 }

✅ Message chỉ reference:
  { orderId, customerId, amount, status }
  
  Consumer cần PII → fetch từ Customer Service API (có auth riêng)
```

---

## Audit Log Retention & Protection

### Retention Policy

| Compliance | Minimum Retention |
| ---------- | ----------------- |
| PCI-DSS | 1 year (3 months immediately available) |
| SOC 2 | Theo org policy (1–3 years) |
| HIPAA | 6 years |
| GDPR | Đủ cho accountability, không quy định cụ thể |
| Internal security | 90 days hot + 1 year cold archive |

### Immutability

```
Audit logs PHẢI:
  ✓ Ship sang storage riêng (S3 Object Lock, WORM storage)
  ✓ Không cho admin broker xóa audit logs
  ✓ Integrity checksum (hash chain nếu cần)
  ✓ Access to audit logs itself audited (meta-audit)

  ✗ Lưu cùng disk broker (mất khi broker compromise)
  ✗ Cho phép DELETE without break-glass procedure
```

### Architecture

```
┌─────────────┐     ship      ┌─────────────┐     archive    ┌──────────┐
│ Kafka audit │ ─────────────►│ S3 / GCS    │ ──────────────►│ Glacier  │
│ topic       │  (real-time)  │ (90 days)   │  (after 90d)   │ (1 year) │
└─────────────┘               └─────────────┘                └──────────┘
                                     │
                                     ▼
                              ┌─────────────┐
                              │ SIEM alerts │
                              └─────────────┘
```

---

## Anti-Patterns

| Anti-Pattern | Rủi Ro | Fix |
| ------------ | ------ | --- |
| Không audit ACL changes | Unauthorized access không detect | Enable authorizer audit + SIEM |
| Log message body (PII) | PII leak vào log system | Metadata-only audit |
| Audit log trên cùng broker disk | Attacker xóa logs | Ship to external immutable storage |
| Không alert auth failures | Brute force undetected | SIEM rule: auth failure spike |
| Retention quá ngắn (< 90 ngày) | Fail compliance audit | Policy theo framework |
| Admin có quyền xóa audit logs | Cover tracks after breach | WORM storage, separate access |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Audit log cần ghi những gì?

**Trả lời:** **WHO** (principal), **WHAT** (action/operation), **WHEN** (timestamp UTC), **WHERE** (resource — topic, group, cluster), **RESULT** (allowed/denied/success/failure). Tier 1: auth events, ACL changes, topic/user lifecycle, admin actions.

### Câu 2: Kafka OSS có built-in audit log không?

**Trả lời:** **Không đầy đủ.** Authorizer log allow/deny decisions qua log4j. Admin API operations cần custom solution hoặc Confluent Enterprise audit logs. Production OSS: parse broker logs + custom authorizer wrapper + ship to SIEM.

### Câu 3: GDPR và Kafka immutable log — mâu thuẫn?

**Trả lời:** Kafka log **append-only** — khó xóa data subject's data. Giải pháp: (1) **Không gửi PII** trong message — chỉ ID reference. (2) **Retention policy** ngắn — auto-delete sau X ngày. (3) **Compaction + tombstone** cho keyed data. (4) Legal basis và DPIA (Data Protection Impact Assessment) document trade-offs.

### Câu 4: Làm sao detect unauthorized ACL change?

**Trả lời:** (1) Enable authorizer audit logging. (2) Ship logs to SIEM real-time. (3) Alert rule: `event=ALTER AND resource_type=ACL AND environment=prod`. (4) Slack/PagerDuty notification. (5) Weekly ACL review report tự động so sánh với baseline.

---

**Quay lại:** [README.md](./README.md) — Tổng quan module Security.
