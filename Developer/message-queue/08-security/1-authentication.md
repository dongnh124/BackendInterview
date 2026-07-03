# Authentication — Xác Thực Client Với Message Broker

> Authentication (Xác Thực) trả lời câu hỏi **"Bạn là ai?"** trước khi broker cho phép produce, consume, hoặc admin. Production broker **phải** yêu cầu auth — không có anonymous access.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [SASL — Simple Authentication and Security Layer](#sasl--simple-authentication-and-security-layer)
3. [SCRAM — Salted Challenge Response Authentication Mechanism](#scram--salted-challenge-response-authentication-mechanism)
4. [mTLS — Mutual TLS](#mtls--mutual-tls)
5. [API Keys & Token-Based Auth](#api-keys--token-based-auth)
6. [OAuth 2.0 / OIDC Integration](#oauth-20--oidc-integration)
7. [So Sánh Theo Broker](#so-sánh-theo-broker)
8. [Secret Management Best Practices](#secret-management-best-practices)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│           AUTHENTICATION MECHANISMS — QUICK REFERENCE            │
├─────────────────────────────────────────────────────────────────┤
│  SASL/PLAIN      → Username/password (CHỈ dùng với TLS!)        │
│  SASL/SCRAM      → Challenge-response, khuyến nghị Kafka        │
│  mTLS            → Certificate-based, zero-trust, service mesh  │
│  API Keys        → Cloud managed (MSK IAM, Confluent API key)   │
│  OAuth/OIDC      → Enterprise SSO, Confluent Cloud RBAC         │
└─────────────────────────────────────────────────────────────────┘
```

| Mechanism | Bảo Mật | Độ Phức Tạp | Use Case |
| --------- | ------- | ----------- | -------- |
| **SASL/PLAIN** | Thấp (plaintext password) | Thấp | Chỉ lab — **bắt buộc TLS** |
| **SASL/SCRAM-SHA-256/512** | Cao | Trung bình | **Kafka production default** |
| **mTLS** | Rất cao | Cao | K8s, service mesh, zero-trust |
| **API Key + Secret** | Trung bình–Cao | Thấp | Cloud managed services |
| **OAuth 2.0 / OIDC** | Cao | Cao | Enterprise, Confluent Cloud |

---

## SASL — Simple Authentication and Security Layer

**SASL (Simple Authentication and Security Layer — Lớp Xác Thực và Bảo Mật Đơn Giản)** là framework chuẩn cho authentication trong Kafka và nhiều protocol khác.

### Kafka SASL Listeners

```
# server.properties
listeners=SASL_SSL://0.0.0.0:9093,CONTROLLER://0.0.0.0:9094
advertised.listeners=SASL_SSL://broker1.example.com:9093

security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
```

**Hai loại listener quan trọng:**

| Listener | Mục Đích |
| -------- | -------- |
| **Client listener** | Producer/Consumer kết nối |
| **Inter-broker listener** | Broker ↔ Broker replication |

> **Lưu ý:** Inter-broker cũng cần auth — attacker có thể giả mạo broker nếu chỉ client auth mà broker-to-broker plaintext.

### SASL/PLAIN

```
Client ──► username + password (plaintext trong TLS tunnel)
Broker ──► verify against JAAS config hoặc credential store
```

```properties
# JAAS config (Kafka broker)
KafkaServer {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="admin-secret"
  user_admin="admin-secret"
  user_order_svc="order-svc-secret";
};
```

**Cảnh báo:** Password truyền plaintext trong SASL handshake — **chỉ chấp nhận được khi có TLS bọc ngoài**. Không dùng SASL_PLAINTEXT (không TLS) trên production.

---

## SCRAM — Salted Challenge Response Authentication Mechanism

**SCRAM (Salted Challenge Response Authentication Mechanism — Cơ Chế Xác Thực Phản Hồi Thách Thức Có Muối)** là cơ chế **khuyến nghị cho Kafka production**.

### Tại Sao SCRAM Tốt Hơn PLAIN

```
SASL/PLAIN:
  Client ──► password ──► Broker (password có thể bị log nếu misconfig)

SCRAM-SHA-256:
  Client ◄── salt + iteration count ──► Broker
  Client ──► ClientProof (hash, KHÔNG gửi password) ──► Broker
  Broker ──► ServerSignature ──► Client (mutual verify)
```

| Đặc Điểm | SCRAM-SHA-256 | SCRAM-SHA-512 |
| -------- | ------------- | ------------- |
| Hash algorithm | SHA-256 | SHA-512 |
| Bảo mật | Đủ cho hầu hết use case | Cao hơn, compliance strict |
| Performance | Nhanh hơn | Chậm hơn chút |
| Khuyến nghị | **Default** | Financial, government |

### Tạo SCRAM User trên Kafka

```bash
# Tạo SCRAM credential cho user order-service
kafka-configs.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --alter --add-config 'SCRAM-SHA-256=[password=StrongP@ssw0rd!]' \
  --entity-type users --entity-name order-service

# List users
kafka-configs.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --describe --entity-type users
```

### Client Configuration

```javascript
// Node.js — kafkajs với SCRAM
const kafka = new Kafka({
  clientId: 'order-producer',
  brokers: ['broker1:9093'],
  ssl: true,
  sasl: {
    mechanism: 'scram-sha-256',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD,
  },
});
```

```properties
            // JAAS client config (Java)
// KafkaClient {
//   org.apache.kafka.common.security.scram.ScramLoginModule required
//   username="order-service"
//   password="StrongP@ssw0rd!";
// };
```

### Credential Rotation

```
1. Tạo password mới cho user (SCRAM hỗ trợ 2 credential cùng lúc)
2. Deploy client với password mới (rolling)
3. Verify tất cả client đã migrate
4. Xóa credential cũ
5. Monitor auth failure metrics trong quá trình rotation
```

---

## mTLS — Mutual TLS

**mTLS (Mutual TLS — TLS Hai Chiều)** — cả client **và** server đều present certificate để xác thực lẫn nhau.

```
Standard TLS (one-way):
  Client ── verify server cert ──► Broker

mTLS (mutual):
  Client ── verify server cert ──► Broker
  Client ◄── verify client cert ── Broker
         (client phải có cert signed bởi trusted CA)
```

### Khi Nào Dùng mTLS

| Scenario | Khuyến Nghị |
| -------- | ----------- |
| Kubernetes + service mesh (Istio, Linkerd) | **mTLS** — mesh tự rotate cert |
| Traditional VM deployment | SCRAM + TLS one-way (đơn giản hơn) |
| Zero-trust architecture | **mTLS** bắt buộc |
| High compliance (PCI-DSS, HIPAA) | **mTLS** + SCRAM (defense in depth) |

### Kafka mTLS Configuration

```properties
# Broker server.properties
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/etc/kafka/secrets/kafka.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/etc/kafka/secrets/kafka.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}

# Yêu cầu client certificate
ssl.client.auth=required

# TLS version
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.protocol=TLSv1.3
```

```javascript
// Client — present cert + verify broker
const kafka = new Kafka({
  brokers: ['broker1:9093'],
  ssl: {
    rejectUnauthorized: true,
    ca: [fs.readFileSync('/certs/ca.pem', 'utf-8')],
    cert: fs.readFileSync('/certs/client.pem', 'utf-8'),
    key: fs.readFileSync('/certs/client-key.pem', 'utf-8'),
  },
});
```

### Certificate Hierarchy

```
                    ┌─────────────┐
                    │  Root CA    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Broker 1 │ │ Broker 2 │ │ Client   │
        │   cert   │ │   cert   │ │   cert   │
        └──────────┘ └──────────┘ └──────────┘
```

**Best practices:**

- Root CA offline, intermediate CA cho signing hàng ngày
- Cert validity **≤ 90 ngày** (automated rotation)
- SAN (Subject Alternative Name — Tên Thay Thế Chủ Thể) phải match hostname
- Không dùng self-signed trên production (trừ lab)

---

## API Keys & Token-Based Auth

Cloud managed messaging services thường dùng **API Key** thay vì tự quản lý SASL.

### Amazon MSK — IAM Authentication

```javascript
// AWS MSK với IAM auth (không cần username/password)
const { generateAuthToken } = require('aws-msk-iam-sasl-signer-js');

const authToken = await generateAuthToken({
  region: 'ap-southeast-1',
});

const kafka = new Kafka({
  brokers: [process.env.MSK_BOOTSTRAP],
  ssl: true,
  sasl: {
    mechanism: 'oauthbearer',
    oauthBearerProvider: async () => ({
      value: authToken.token,
    }),
  },
});
```

**IAM policy ví dụ:**

```json
{
  "Effect": "Allow",
  "Action": [
    "kafka-cluster:Connect",
    "kafka-cluster:WriteData",
    "kafka-cluster:ReadData"
  ],
  "Resource": [
    "arn:aws:kafka:ap-southeast-1:123456789:cluster/my-cluster/*",
    "arn:aws:kafka:ap-southeast-1:123456789:topic/my-cluster/orders/*"
  ]
}
```

### Confluent Cloud — API Key + Secret

```
API Key:    L47XXXXX
API Secret: cfltXXXXX (chỉ hiện 1 lần khi tạo)

Gán API Key vào RBAC role:
  - DeveloperRead  → READ topics
  - DeveloperWrite → READ + WRITE
  - ResourceOwner  → full control trên resource scope
```

### RabbitMQ — Username/Password + TLS

```javascript
const connection = await amqp.connect({
  protocol: 'amqps',
  hostname: 'rabbitmq.internal',
  port: 5671,
  username: process.env.RABBITMQ_USER,
  password: process.env.RABBITMQ_PASSWORD,
  vhost: '/orders',
});
```

RabbitMQ dùng **internal user database** hoặc **LDAP/OAuth2 plugin** cho enterprise.

---

## OAuth 2.0 / OIDC Integration

**OAuth 2.0 (Open Authorization — Ủy Quyền Mở)** và **OIDC (OpenID Connect — Kết Nối Mở Nhận Dạng)** cho phép tích hợp SSO (Single Sign-On — Đăng Nhập Một Lần) enterprise.

```
┌──────────┐    1. Request token     ┌──────────┐
│  Client  │ ───────────────────────►│   IdP    │
│ (Kafka)  │◄─────────────────────── │ (Okta,   │
└────┬─────┘    2. JWT access token   │  Azure)  │
     │                                └──────────┘
     │ 3. Present JWT
     ▼
┌──────────┐
│  Broker  │ ── validate JWT signature, claims, expiry
│ (Kafka)  │
└──────────┘
```

| Platform | OAuth Support |
| -------- | ------------- |
| **Confluent Platform** | OAuthBearerLoginModule + OIDC |
| **Confluent Cloud** | SSO + RBAC native |
| **Apache Kafka (OSS)** | Cần custom authorizer hoặc plugin |
| **RabbitMQ** | rabbitmq-auth-mechanism-oauth2 plugin |

---

## So Sánh Theo Broker

| Broker | Auth Mechanisms | Khuyến Nghị Production |
| ------ | --------------- | ---------------------- |
| **Apache Kafka** | SASL/PLAIN, SCRAM, GSSAPI (Kerberos), OAuth, mTLS | SCRAM-SHA-256 + TLS |
| **RabbitMQ** | PLAIN, AMQPLAIN, EXTERNAL (cert), OAuth2 | Username/password + TLS hoặc cert |
| **Amazon MSK** | TLS, IAM, SCRAM | IAM (AWS-native) hoặc SCRAM |
| **Confluent Cloud** | API Key, OAuth/SSO | API Key scoped + RBAC |
| **Azure Event Hubs** | SAS (Shared Access Signature), AAD | Azure AD (Managed Identity) |
| **GCP Pub/Sub** | Service Account, OAuth2 | Workload Identity |

---

## Secret Management Best Practices

```
❌ KHÔNG:
  - Hardcode password trong source code
  - Commit credentials vào git
  - Dùng chung 1 credential cho tất cả services
  - Log username/password

✅ NÊN:
  - AWS Secrets Manager / Azure Key Vault / HashiCorp Vault
  - Kubernetes Secrets + external-secrets operator
  - Service account riêng per microservice
  - Rotate credentials định kỳ (90 ngày)
  - Inject qua environment variable tại runtime
```

### Service Account Pattern

```
Microservice          Kafka User           Scope
─────────────────────────────────────────────────
order-service    →    order-svc-prod    →  WRITE orders, READ order-updates
payment-service  →    payment-svc-prod  →  READ orders, WRITE payment-events
analytics-job    →    analytics-prod    →  READ * (chỉ READ topics cần thiết)
admin-tool       →    kafka-admin       →  ALL (human, MFA required)
```

---

## Anti-Patterns

| Anti-Pattern | Rủi Ro | Fix |
| ------------ | ------ | --- |
| `SASL_PLAINTEXT` không TLS | Credential sniffing | Dùng `SASL_SSL` |
| Shared `admin` credential cho app | Over-privilege, không audit được | Service account riêng |
| Password trong docker-compose.yml | Leak qua git | Secret manager |
| Self-signed cert không rotate | Expiry outage | cert-manager, automated rotation |
| Auth chỉ client, không inter-broker | Broker impersonation | Enable inter-broker SASL/SSL |
| Disable auth "tạm" để debug | Attack window | Dùng auth-enabled staging env |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: SCRAM khác PLAIN thế nào?

**Trả lời:** **PLAIN** gửi password trực tiếp (cần TLS bảo vệ). **SCRAM** dùng challenge-response — client chứng minh biết password mà **không gửi password**. SCRAM chống replay attack tốt hơn và là best practice Kafka.

### Câu 2: Khi nào dùng mTLS thay vì SCRAM?

**Trả lời:** **mTLS** khi: (1) service mesh tự quản cert, (2) zero-trust yêu cầu cert-based identity, (3) compliance strict. **SCRAM** khi: VM/traditional deploy, team quen username/password, cần đơn giản. Có thể kết hợp cả hai.

### Câu 3: MSK IAM auth hoạt động ra sao?

**Trả lời:** Client dùng AWS credentials (IAM role/instance profile) generate **signed auth token** → present cho broker qua SASL OAUTHBEARER → broker validate signature với AWS → map IAM identity sang Kafka ACL. Không cần quản lý password riêng.

### Câu 4: Làm sao rotate SCRAM password không downtime?

**Trả lời:** Kafka SCRAM hỗ trợ **2 credentials đồng thời** cho 1 user. Tạo password mới → rolling deploy clients → verify → xóa password cũ. Monitor `authentication failures` metric trong quá trình migration.

---

**Tiếp theo:** [2-authorization-acl.md](./2-authorization-acl.md) — Authorization sau khi đã xác thực identity.
