# Authorization & ACL — Phân Quyền Truy Cập Message Broker

> Authorization (Phân Quyền) trả lời **"Bạn được phép làm gì?"** sau khi Authentication (Xác Thực) xác minh identity. **ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập)** và **RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò)** là hai mô hình chính.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Nguyên Tắc Least Privilege](#nguyên-tắc-least-privilege)
3. [Kafka ACL](#kafka-acl)
4. [RabbitMQ Permissions](#rabbitmq-permissions)
5. [RBAC vs ACL](#rbac-vs-acl)
6. [Topic-Level Permission Design](#topic-level-permission-design)
7. [Consumer Group ACL](#consumer-group-acl)
8. [Schema Registry Authorization](#schema-registry-authorization)
9. [Cloud Managed Authorization](#cloud-managed-authorization)
10. [Anti-Patterns](#anti-patterns)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTHORIZATION MODEL — QUICK REFERENCE               │
├─────────────────────────────────────────────────────────────────┤
│  Kafka ACL       → Principal + Resource + Operation             │
│  RabbitMQ        → User + Vhost + Configure/Write/Read          │
│  RBAC            → Role gom nhiều permissions (Confluent Cloud)  │
│  IAM Policy      → AWS MSK resource-based permissions           │
│  Principle       → LEAST PRIVILEGE — chỉ cấp đủ dùng           │
└─────────────────────────────────────────────────────────────────┘
```

| Mô Hình | Granularity (Độ Chi Tiết) | Phù Hợp |
| ------- | ------------------------- | ------- |
| **ACL (flat)** | Per resource, per operation | Kafka OSS, RabbitMQ |
| **RBAC (role-based)** | Role → nhiều permissions | Confluent Cloud, enterprise |
| **IAM (policy-based)** | AWS resource ARN | MSK, SQS, SNS |
| **ABAC (attribute-based)** | Tag, label, attribute | Advanced, multi-tenant |

---

## Nguyên Tắc Least Privilege

**Least Privilege (Quyền Tối Thiểu)** — mỗi service account chỉ có quyền **đủ để làm việc**, không hơn.

```
❌ BAD — payment-service có ALL access:
  payment-svc: READ, WRITE trên * (tất cả topics)

✅ GOOD — payment-service scoped:
  payment-svc:
    READ  topic "orders"
    WRITE topic "payment-events"
    READ  group  "payment-processors"
```

### Permission Design Workflow

```
1. Liệt kê tất cả microservices và vai trò messaging
2. Với mỗi service: liệt kê topics produce + consume
3. Map sang operations tối thiểu (READ/WRITE/CREATE)
4. Tách admin account khỏi application account
5. Review định kỳ — xóa permissions không còn dùng
6. Alert khi ACL thay đổi
```

---

## Kafka ACL

**Kafka ACL** gắn **Principal** (identity từ SASL/mTLS) với **Resource** và **Operation**.

### Resource Types

| Resource | Ví Dụ | Operations |
| -------- | ----- | ---------- |
| **Topic** | `orders`, `payment-events` | READ, WRITE, CREATE, DELETE, ALTER, DESCRIBE |
| **Group** | `order-processors` | READ, DESCRIBE |
| **Cluster** | `kafka-cluster` | CREATE (topic), DESCRIBE, ALTER, IDEMPOTENT_WRITE |
| **TransactionalId** | `order-txn-producer` | WRITE, DESCRIBE |
| **DelegationToken** | token resource | DESCRIBE |

### ACL Syntax

```bash
# Cho phép order-service WRITE topic orders
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:order-service \
  --operation Write \
  --topic orders

# Cho phép payment-service READ topic orders
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:payment-service \
  --operation Read \
  --topic orders

# Consumer group permission
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:payment-service \
  --operation Read \
  --group payment-processors
```

### ACL Entry Format

```
Principal: User:payment-service
Resource:  Topic:orders
Operation: Read
Host:      * (hoặc IP cụ thể)
Permission: Allow
```

### Super User vs Regular User

```properties
# Broker config — super users bypass ACL
super.users=User:admin;User:kafka-admin
```

> **Cảnh báo:** Super user có **full access** — giới hạn số lượng, yêu cầu MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố), audit mọi action.

### ACL Matrix Ví Dụ — Order System

| Principal | Topic: orders | Topic: payment-events | Group: order-processors | Group: payment-processors |
| --------- | ------------- | --------------------- | ----------------------- | ------------------------- |
| order-service | WRITE | — | READ | — |
| payment-service | READ | WRITE | — | READ |
| notification-svc | — | READ | — | — |
| analytics-job | READ | READ | — | — |
| kafka-admin | ALL | ALL | ALL | ALL |

### Prefix & Literal Matching

```bash
# Literal — chỉ topic "orders" chính xác
--topic orders

# Prefixed — tất cả topic bắt đầu "orders."
--resource-pattern-type prefixed --topic orders.

# Ví dụ: orders.created, orders.updated, orders.cancelled
```

**Pattern type:**

| Type | Match |
| ---- | ----- |
| **LITERAL** | Tên chính xác |
| **PREFIXED** | Prefix match |
| **MATCH** | Wildcard (deprecated) |

---

## RabbitMQ Permissions

RabbitMQ phân quyền theo **Virtual Host (Vhost — Máy Chủ Ảo)**, **User**, và ba loại permission.

### Ba Loại Permission

| Permission | Ý Nghĩa | Ví Dụ |
| ---------- | ------- | ----- |
| **Configure (Cấu Hình)** | Tạo/xóa queue, exchange, binding | `orders.*` |
| **Write (Ghi)** | Publish message vào exchange | `orders.exchange` |
| **Read (Đọc)** | Consume từ queue | `orders.queue` |

### Permission Pattern

```
Permission pattern dùng regex trên resource name:
  orders.*     → match orders.created, orders.updated
  ^orders$     → chỉ orders chính xác
  .*           → tất cả (admin only)
```

### Cấu Hình qua rabbitmqctl

```bash
# Tạo user
rabbitmqctl add_user order-service StrongPassword123

# Set permissions trên vhost /orders
rabbitmqctl set_permissions -p /orders order-service \
  "^orders\\..*" "^orders\\.exchange$" "^orders\\.queue$"

# Configure  Write   Read
#    ↓        ↓       ↓
# ^orders\..* ^orders\.exchange$ ^orders\.queue$

# Tạo vhost riêng cho team
rabbitmqctl add_vhost /payments
rabbitmqctl set_permissions -p /payments payment-service ".*" ".*" ".*"
```

### Vhost Isolation

```
┌─────────────────────────────────────────────────────────┐
│                    RabbitMQ Cluster                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Vhost: /     │  │ Vhost: /orders│ │ Vhost: /pay  │   │
│  │ (default)    │  │ order-team   │  │ payment-team │   │
│  │              │  │              │  │              │   │
│  │ exchanges    │  │ orders.ex    │  │ payments.ex  │   │
│  │ queues       │  │ orders.q     │  │ payments.q   │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│         ↑                  ↑                  ↑           │
│    admin only        order-service      payment-service   │
└─────────────────────────────────────────────────────────┘
```

**Best practice:** Mỗi team/domain có **vhost riêng** — isolation tự nhiên, dễ audit.

### Topic Permissions (RabbitMQ 3.7+)

Cho **topic exchange** — permission theo routing key pattern:

```bash
rabbitmqctl set_topic_permissions -p /orders order-service \
  "orders.exchange" "^orders\\.(created|updated)$" "^orders\\.(created|updated)$"
#                              routing key write    routing key read
```

---

## RBAC vs ACL

| Tiêu Chí | ACL (Flat) | RBAC (Role-Based) |
| -------- | ---------- | ----------------- |
| **Mô hình** | User → Permission trực tiếp | User → Role → Permissions |
| **Quản lý** | Khó scale nhiều users | Dễ — gán role thay vì từng permission |
| **Audit** | Per-user ACL list | Per-role assignment |
| **Platform** | Kafka OSS, RabbitMQ | Confluent Cloud, enterprise tools |
| **Flexibility** | Rất chi tiết | Role template + override |

### Confluent Cloud RBAC Roles

```
OrganizationAdmin     → Full org control
EnvironmentAdmin      → Full env control
ClusterAdmin          → Cluster management
DeveloperRead         → READ topics, groups, schemas
DeveloperWrite       → READ + WRITE (không admin)
ResourceOwner         → Full trên specific resource
```

```
Gán role:
  API Key "analytics-key" → DeveloperRead → Resource: topic "orders"
  API Key "order-prod"    → DeveloperWrite → Resource: topic "orders", "order-events"
```

---

## Topic-Level Permission Design

### Naming Convention Hỗ Trợ ACL

```
Pattern: {domain}.{entity}.{event-type}

orders.order.created       → domain=orders
orders.order.updated
payments.transaction.completed
notifications.email.sent

ACL prefixed "orders." → tất cả order events
ACL literal "payments.transaction.completed" → chỉ event cụ thể
```

### Multi-Tenant ACL

```
Tenant A:
  User: tenant-a-producer → WRITE topic "tenant-a.*"
  User: tenant-a-consumer → READ  topic "tenant-a.*", group "tenant-a-*"

Tenant B:
  User: tenant-b-producer → WRITE topic "tenant-b.*"
  User: tenant-b-consumer → READ  topic "tenant-b.*", group "tenant-b-*"

Isolation: tenant-a KHÔNG đọc được tenant-b topics
```

### Deny vs Allow

Kafka ACL mặc định **deny all** — chỉ allow explicit:

```
Default: DENY all
Explicit ALLOW: User:order-service → WRITE orders
Result: order-service WRITE orders, mọi thứ khác DENY
```

RabbitMQ tương tự — user không có permission → access denied.

---

## Consumer Group ACL

Consumer **bắt buộc** có ACL trên **Group resource** — thiếu sẽ không join group được.

```bash
# Kafka — consumer group ACL
kafka-acls.sh --add \
  --allow-principal User:order-service \
  --operation Read \
  --group order-processors

# DESCRIBE cần cho monitoring tools
kafka-acls.sh --add \
  --allow-principal User:monitoring-agent \
  --operation Describe \
  --group '*'
```

**Lưu ý:** Group ID phải **consistent** — đổi group ID = cần ACL mới.

```
Consumer group naming:
  {service-name}-{purpose}
  order-service-processors
  payment-service-retry
  analytics-daily-batch
```

---

## Schema Registry Authorization

Schema Registry có ACL riêng — tách khỏi Kafka broker ACL.

| Operation | Ý Nghĩa |
| --------- | ------- |
| **READ** | Fetch schema by ID |
| **WRITE** | Register schema mới |
| **DELETE** | Xóa schema version |
| **COMPATIBILITY_READ/WRITE** | Đọc/sửa compatibility mode |

```bash
# Confluent — schema registry ACL
confluent iam acl create \
  --allow-principal User:order-service \
  --operation Write \
  --subject orders-value
```

**Best practice:** Producer cần WRITE subject `{topic}-value`. Consumer cần READ. Không cấp DELETE cho application accounts.

---

## Cloud Managed Authorization

### AWS MSK — IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:ReadData"
      ],
      "Resource": [
        "arn:aws:kafka:region:account:topic/cluster-name/orders",
        "arn:aws:kafka:region:account:group/cluster-name/order-processors"
      ]
    }
  ]
}
```

### Azure Event Hubs — RBAC

```
Azure Built-in Roles:
  Azure Event Hubs Data Owner    → Full data access
  Azure Event Hubs Data Sender   → Send only
  Azure Event Hubs Data Receiver → Receive only
```

### GCP Pub/Sub — IAM

```
roles/pubsub.publisher  → Publish to topic
roles/pubsub.subscriber → Subscribe, ack messages
roles/pubsub.viewer     → List topics/subscriptions (metadata only)
```

---

## Anti-Patterns

| Anti-Pattern | Rủi Ro | Fix |
| ------------ | ------ | --- |
| `User:*` wildcard ACL | Bất kỳ user nào access | Named principals only |
| Admin credential trong app | Full cluster control nếu leak | Service account scoped |
| Không ACL consumer group | Consumer fail join group | Group ACL cùng lúc topic ACL |
| `.*` permission RabbitMQ | Full vhost access | Regex scoped per resource |
| ACL copy từ staging sang prod | Over/under privilege | ACL matrix review per env |
| Không review ACL định kỳ | Orphan permissions tích lũy | Quarterly ACL audit |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka ACL gồm những thành phần nào?

**Trả lời:** **Principal** (User:service-name) + **Resource** (Topic/Group/Cluster) + **Operation** (Read/Write/Create/...) + **Permission** (Allow/Deny) + **Host** (IP filter, thường `*`). Default deny — phải explicit allow.

### Câu 2: RabbitMQ Configure/Write/Read khác nhau thế nào?

**Trả lời:** **Configure** — tạo/xóa queue, exchange, binding (admin-like). **Write** — publish message vào exchange. **Read** — consume từ queue. App thường chỉ cần Write + Read trên resources cụ thể, không cần Configure.

### Câu 3: RBAC tốt hơn ACL khi nào?

**Trả lời:** Khi có **nhiều users/services** cùng role — RBAC gom permissions vào role template, gán role thay vì copy ACL. Dễ onboard/offboard, audit theo role. ACL flat tốt hơn khi cần **granularity cực cao** per service.

### Câu 4: Thiết kế ACL cho microservices như thế nào?

**Trả lời:** (1) Service account riêng per service. (2) Topic naming convention hỗ trợ prefix ACL. (3) Chỉ READ topics consume, WRITE topics produce. (4) Consumer group ACL matching group ID. (5) Admin tách biệt. (6) Document ACL matrix, review quarterly.

---

**Tiếp theo:** [3-encryption.md](./3-encryption.md) — Bảo vệ data in-transit và at-rest.
