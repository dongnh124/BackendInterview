# Bảo Mật Messaging — Tổng Quan

> Chủ đề bắt buộc trước go-live production: Authentication (Xác Thực), Authorization (Phân Quyền), Encryption (Mã Hóa) in-transit (truyền tải) & at-rest (lưu trữ), và Audit Logging (Ghi Nhật Ký Kiểm Toán) cho message brokers.

## Mục Lục

1. [Tại Sao Security Quan Trọng Với Messaging](#tại-sao-security-quan-trọng-với-messaging)
2. [Mô Hình Bảo Mật Defense-in-Depth](#mô-hình-bảo-mật-defense-in-depth)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Security Checklist Trước Go-Live](#security-checklist-trước-go-live)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Security Quan Trọng Với Messaging

Message broker thường là **backbone (xương sống)** của hệ thống phân tán — chứa dữ liệu nhạy cảm: order events, payment notifications, PII (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân), audit trails. Broker mặc định **không an toàn** — Kafka, RabbitMQ local thường chạy không auth, plaintext.

| Rủi Ro | Hậu Quả | Giải Pháp |
| ------ | ------- | --------- |
| Broker exposed (lộ ra internet) | Data breach, message injection | Network isolation, TLS, auth |
| Over-privileged client | Consumer đọc topic không thuộc quyền | ACL / RBAC least privilege |
| Plaintext traffic | Sniffing credentials, message content | TLS in-transit |
| Unencrypted disk | Disk theft → đọc message | Encryption at-rest |
| Không audit trail | Không truy vết incident, fail compliance | Audit logging |

> **Quy tắc vàng:** Production messaging mặc định là **TLS everywhere** + **Authentication bắt buộc** + **Least privilege ACL** + **Audit log** cho admin operations. Không có "dev broker" trên production network.

---

## Mô Hình Bảo Mật Defense-in-Depth

```
┌──────────────────────────────────────────────────────────────────────────┐
│              MESSAGING SECURITY LAYERS (Các Lớp Bảo Mật)                  │
│                                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌───────────┐ │
│  │   NETWORK   │    │    AUTHN    │    │    AUTHZ    │    │ ENCRYPTION│ │
│  │  Isolation  │───►│  Who are    │───►│  What can   │───►│  Protect  │ │
│  │  VPC/Firewall│    │  you?       │    │  you do?    │    │  data     │ │
│  └─────────────┘    └─────────────┘    └─────────────┘    └───────────┘ │
│        │                  │                  │                  │       │
│   Private subnet     SASL/SCRAM          ACL/RBAC           TLS 1.2+    │
│   Security groups    mTLS                Topic-level        At-rest KMS │
│   No public port     API keys            Consumer group     Schema ACL  │
│                                                                          │
│  ═══════════════════ CROSS-CUTTING ═══════════════════════════════════  │
│  Audit Logging │ Secret Management │ Compliance (GDPR, PCI-DSS, SOC2)   │
└──────────────────────────────────────────────────────────────────────────┘
```

**Bốn trụ cột bảo mật:**

1. **Authentication (Xác Thực)** — xác minh identity của producer/consumer/admin
2. **Authorization (Phân Quyền)** — giới hạn hành động trên topic/queue/exchange
3. **Encryption (Mã Hóa)** — bảo vệ data khi truyền và khi lưu
4. **Audit & Compliance (Kiểm Toán & Tuân Thủ)** — ghi nhận ai làm gì, khi nào

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 4–6 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-authentication.md](./1-authentication.md) | SASL/SCRAM, mTLS, API keys, OAuth | 1.5 giờ |
| 2 | [2-authorization-acl.md](./2-authorization-acl.md) | ACL, RBAC, topic-level permissions | 1.5 giờ |
| 3 | [3-encryption.md](./3-encryption.md) | TLS in-transit, encryption at-rest | 1 giờ |
| 4 | [4-audit-logging.md](./4-audit-logging.md) | Audit trails, compliance requirements | 1 giờ |

**Điều kiện tiên quyết:**

- [03-apache-kafka/1-kafka-architecture.md](../03-apache-kafka/1-kafka-architecture.md) — hiểu broker, client roles
- [04-rabbitmq/README.md](../04-rabbitmq/README.md) — hiểu exchange, queue, vhost

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4. Auth trước (ai được vào), authz tiếp (làm được gì), encryption (bảo vệ data), audit (truy vết & compliance).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-authentication.md](./1-authentication.md) | SASL mechanisms, SCRAM, mTLS, API keys cross-broker |
| [2-authorization-acl.md](./2-authorization-acl.md) | Kafka ACL, RabbitMQ permissions, RBAC, least privilege |
| [3-encryption.md](./3-encryption.md) | TLS config, certificate management, at-rest encryption |
| [4-audit-logging.md](./4-audit-logging.md) | Audit events, SIEM integration, GDPR/PCI compliance |

---

## Security Checklist Trước Go-Live

```markdown
## Messaging Security Go-Live Checklist

### Authentication
- [ ] Broker yêu cầu auth — không anonymous access
- [ ] Credentials lưu trong Secret Manager (không hardcode)
- [ ] Service account riêng cho mỗi producer/consumer
- [ ] Certificate rotation schedule đã định

### Authorization
- [ ] ACL/RBAC theo least privilege
- [ ] Admin account tách biệt application account
- [ ] Topic/queue naming convention phản ánh ownership

### Encryption
- [ ] TLS 1.2+ cho client ↔ broker
- [ ] TLS cho broker ↔ broker (inter-broker)
- [ ] Encryption at-rest enabled (disk/KMS)
- [ ] Không có plaintext port exposed

### Network
- [ ] Broker trong private subnet
- [ ] Security group chỉ allow app subnets
- [ ] Không expose broker port ra internet

### Audit & Compliance
- [ ] Admin operations được log
- [ ] ACL changes được alert
- [ ] Retention policy cho audit logs
- [ ] PII trong message đã được đánh giá
```

---

## Bài Tập Thực Hành

### Lab 1: Enable SASL/SCRAM trên Kafka (60 phút)

```bash
# 1. Tạo JAAS config với SCRAM user
# 2. Enable SASL_PLAINTEXT hoặc SASL_SSL listener
# 3. Tạo user: kafka-configs --alter --add-config SCRAM-SHA-256=[password=...]
# 4. Test: producer không auth → bị reject
# 5. Test: producer có auth nhưng không ACL → AuthorizationException
```

### Lab 2: Thiết Kế ACL Matrix (45 phút)

```
Kịch bản: Microservices order system
- order-service: produce orders, consume order-updates
- payment-service: consume orders, produce payment-events
- notification-service: consume payment-events only

Bài tập:
1. Liệt kê principal (user/service account) cho mỗi service
2. Viết ACL matrix: READ/WRITE/CREATE trên từng topic
3. Xác định ACL nào thừa quyền (over-privileged)
4. Thiết kế admin ACL tách biệt
```

### Lab 3: TLS End-to-End (45 phút)

```
1. Generate CA + broker cert + client cert
2. Configure Kafka/RabbitMQ với TLS
3. Verify: openssl s_client connect → xem certificate chain
4. Test: client không cert → connection refused
5. Document cert expiry date và rotation procedure
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: SASL/SCRAM khác mTLS thế nào?

**Trả lời:** **SASL/SCRAM** xác thực bằng username/password qua challenge-response — phù hợp app-to-broker. **mTLS (Mutual TLS — TLS Hai Chiều)** xác thực bằng certificate — mạnh hơn, phù hợp service mesh và zero-trust. Có thể kết hợp cả hai. Xem [1-authentication.md](./1-authentication.md).

### Câu 2: Kafka ACL hoạt động ở mức nào?

**Trả lời:** ACL gắn với **principal** (user) + **resource** (topic, group, cluster) + **operation** (READ, WRITE, CREATE, DESCRIBE, DELETE, ALTER). Ví dụ: user `payment-svc` chỉ READ topic `orders`, WRITE topic `payment-events`. Xem [2-authorization-acl.md](./2-authorization-acl.md).

### Câu 3: TLS có ảnh hưởng performance không?

**Trả lời:** Có overhead **5–15%** throughput tùy CPU và cipher suite. Dùng **TLS 1.3**, hardware AES-NI, và session resumption để giảm impact. Trade-off bảo mật luôn đáng giá trên production. Xem [3-encryption.md](./3-encryption.md).

### Câu 4: Message chứa PII cần làm gì?

**Trả lời:** (1) Minimize PII trong message — chỉ gửi ID reference. (2) Encrypt at-rest + TLS in-transit. (3) ACL restrict ai đọc được. (4) Retention policy ngắn. (5) Audit log ai access. (6) Tuân thủ GDPR — right to erasure khó với immutable log. Xem [4-audit-logging.md](./4-audit-logging.md).

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Kafka architecture | [03-apache-kafka/1-kafka-architecture.md](../03-apache-kafka/1-kafka-architecture.md) |
| Schema Registry security | [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) |
| RabbitMQ clustering | [04-rabbitmq/5-clustering-ha.md](../04-rabbitmq/5-clustering-ha.md) |
| Cloud managed (MSK security) | [10-cloud-managed/](../10-cloud-managed/) (sắp có) |
| Production checklist | [09-monitoring/](../09-monitoring/) (sắp có) |

---

**Cập Nhật:** 2026-07-03
