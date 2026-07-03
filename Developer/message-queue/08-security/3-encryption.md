# Encryption — Mã Hóa Dữ Liệu Messaging

> Encryption (Mã Hóa) bảo vệ message content và metadata khỏi bị đọc trái phép — cả khi **in-transit (truyền tải)** trên network và **at-rest (lưu trữ)** trên disk broker.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [TLS In-Transit — Mã Hóa Truyền Tải](#tls-in-transit--mã-hóa-truyền-tải)
3. [Cấu Hình TLS Cho Kafka](#cấu-hình-tls-cho-kafka)
4. [Cấu Hình TLS Cho RabbitMQ](#cấu-hình-tls-cho-rabbitmq)
5. [Encryption At-Rest — Mã Hóa Lưu Trữ](#encryption-at-rest--mã-hóa-lưu-trữ)
6. [End-to-End Encryption (E2EE)](#end-to-end-encryption-e2ee)
7. [Key Management](#key-management)
8. [Performance Impact](#performance-impact)
9. [Certificate Lifecycle](#certificate-lifecycle)
10. [Anti-Patterns](#anti-patterns)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              ENCRYPTION LAYERS — QUICK REFERENCE                │
├─────────────────────────────────────────────────────────────────┤
│  TLS in-transit     → Client↔Broker, Broker↔Broker             │
│  At-rest encryption → Disk/volume trên broker node             │
│  E2EE (End-to-End)  → App encrypt payload, broker không đọc   │
│  Field-level        → Encrypt PII fields trong message body     │
│  Minimum standard   → TLS 1.2+, strong cipher suites           │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | Bảo Vệ Khỏi | Không Bảo Vệ Khỏi |
| ----- | ----------- | ----------------- |
| **TLS in-transit** | Network sniffing, MITM (Man-in-the-Middle — Tấn Công Người Ở Giữa) | Compromised broker, unauthorized consumer |
| **At-rest encryption** | Disk theft, snapshot leak | In-memory access, authorized reader |
| **E2EE** | Broker admin, compromised broker | Client-side key leak |
| **Field-level** | PII exposure trong message | Non-PII fields vẫn plaintext |

---

## TLS In-Transit — Mã Hóa Truyền Tải

**TLS (Transport Layer Security — Bảo Mật Tầng Vận Chuyển)** mã hóa toàn bộ traffic giữa client và broker.

### Ba Kênh Cần TLS

```
┌──────────┐  TLS   ┌──────────┐  TLS   ┌──────────┐
│ Producer │◄──────►│  Broker  │◄──────►│ Consumer │
└──────────┘        └────┬─────┘        └──────────┘
                         │ TLS
                    ┌────▼─────┐
                    │  Broker  │  (inter-broker replication)
                    │ (replica)│
                    └──────────┘
```

| Kênh | Tại Sao Quan Trọng |
| ---- | ------------------ |
| **Client → Broker** | Bảo vệ message content + credentials trên wire |
| **Broker → Broker** | Replication traffic chứa full message |
| **Admin → Broker** | Admin API, JMX metrics |

### TLS vs SSL

```
SSL (Secure Sockets Layer) — deprecated, không dùng
TLS 1.0, 1.1 — deprecated (2020+)
TLS 1.2     — minimum production
TLS 1.3     — khuyến nghị (faster handshake, stronger ciphers)
```

### Cipher Suites Khuyến Nghị

```
TLS 1.3 (preferred):
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256

TLS 1.2 (fallback):
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256

❌ Tránh:
  NULL, EXPORT, DES, 3DES, RC4, MD5
```

---

## Cấu Hình TLS Cho Kafka

### Listener Security Protocol

| Protocol | Mô Tả | Production |
| -------- | ----- | ---------- |
| **PLAINTEXT** | Không mã hóa | ❌ Không bao giờ |
| **SSL** | TLS only | ✅ |
| **SASL_PLAINTEXT** | SASL không TLS | ❌ |
| **SASL_SSL** | SASL + TLS | ✅ **Khuyến nghị** |

```properties
# server.properties
listeners=SASL_SSL://0.0.0.0:9093,CONTROLLER://0.0.0.0:9094
advertised.listeners=SASL_SSL://broker1.internal:9093

# SSL config
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}

# Inter-broker
security.inter.broker.protocol=SASL_SSL

# TLS settings
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.protocol=TLSv1.3
ssl.endpoint.identification.algorithm=https
```

### Client Certificate Verification

```properties
# Broker verify client hostname
ssl.endpoint.identification.algorithm=https

# Client verify broker hostname match cert SAN
# bootstrap.servers=broker1.internal:9093
# Cert SAN phải chứa broker1.internal
```

### Kafka Connect & Schema Registry TLS

```
Mọi component kết nối broker đều cần TLS:
  - Kafka Connect workers
  - Schema Registry
  - ksqlDB / Kafka Streams
  - Monitoring agents (Burrow, Kafka Exporter)
  - Admin CLI tools
```

---

## Cấu Hình TLS Cho RabbitMQ

### Enable TLS Listener

```ini
# rabbitmq.conf
listeners.ssl.default = 5671

ssl_options.cacertfile = /etc/rabbitmq/ssl/ca_certificate.pem
ssl_options.certfile   = /etc/rabbitmq/ssl/server_certificate.pem
ssl_options.keyfile    = /etc/rabbitmq/ssl/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false

# TLS versions
ssl_options.versions.1 = tlsv1.3
ssl_options.versions.2 = tlsv1.2
```

### Management UI TLS

```ini
management.ssl.port = 15671
management.ssl.cacertfile = /etc/rabbitmq/ssl/ca_certificate.pem
management.ssl.certfile   = /etc/rabbitmq/ssl/server_certificate.pem
management.ssl.keyfile    = /etc/rabbitmq/ssl/server_key.pem
```

### Client Connection

```javascript
const connection = await amqp.connect({
  protocol: 'amqps',          // amqps = AMQP over TLS
  hostname: 'rabbitmq.internal',
  port: 5671,
  ca: [fs.readFileSync('ca.pem')],
});
```

### Inter-Node TLS (Cluster)

```
RabbitMQ cluster replication cũng cần TLS:
  cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
  cluster_name = production

# Erlang distribution TLS cho node-to-node
# rabbitmq.conf: cluster_partition_handling = autoheal
```

---

## Encryption At-Rest — Mã Hóa Lưu Trữ

**At-rest encryption** bảo vệ data trên disk khi broker lưu message (log segments, queue files).

### Tại Sao Cần At-Rest

```
Scenario không có at-rest:
  - Attacker steal EBS/disk snapshot → đọc message plaintext
  - Decommissioned server disk chưa wipe → data leak
  - Backup tape/cloud snapshot exposed
```

### Kafka At-Rest Options

| Approach | Mô Tả |
| -------- | ----- |
| **Volume encryption** | AWS EBS encryption, Azure Disk Encryption, LUKS (Linux Unified Key Setup — Thiết Lập Khóa Linux) |
| **Broker-level** | Confluent Platform encryption at rest |
| **Cloud managed** | MSK, Confluent Cloud — enabled by default |

```bash
# AWS MSK — encryption at rest via KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)
aws kafka create-cluster \
  --encryption-info '{
    "EncryptionAtRest": {
      "DataVolumeKMSKeyId": "arn:aws:kms:region:account:key/key-id"
    },
    "EncryptionInTransit": {
      "ClientBroker": "TLS",
      "InCluster": true
    }
  }'
```

### RabbitMQ At-Rest

RabbitMQ lưu message trên disk (persistent queues):

```
Approach:
  1. Volume-level encryption (EBS, encrypted PVC in K8s)
  2. RabbitMQ không có native message-level at-rest encryption
  3. E2EE nếu cần message-level protection
```

### Kubernetes — Encrypted PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data
spec:
  storageClassName: encrypted-gp3  # EBS encrypted
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 500Gi
```

---

## End-to-End Encryption (E2EE)

**E2EE (End-to-End Encryption — Mã Hóa Đầu Cuối)** — producer encrypt payload, chỉ consumer có key mới decrypt. Broker **không đọc được** message content.

```
Producer                    Broker                     Consumer
    │                          │                           │
    ├── encrypt(payload) ─────►│ (stores ciphertext)       │
    │   with shared key        │                           │
    │                          ├── deliver ciphertext ────►│
    │                          │                           ├── decrypt
    │                          │                           │   with shared key
```

### Khi Nào Cần E2EE

| Scenario | Cần E2EE? |
| -------- | --------- |
| Internal microservices, trust broker | Không — TLS đủ |
| Multi-tenant SaaS, untrusted broker admin | **Có** |
| PII/financial data, compliance strict | **Có** (field-level minimum) |
| Cross-organization data sharing | **Có** |

### Implementation Patterns

```javascript
// Field-level encryption — chỉ encrypt PII fields
const event = {
  orderId: 'ord-123',           // plaintext — OK
  customerEmail: encrypt(email), // encrypted PII
  amount: 99.99,                // plaintext — OK
};

// Envelope encryption
// 1. Generate DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu) per message
// 2. Encrypt payload with DEK
// 3. Encrypt DEK with KEK (Key Encryption Key — Khóa Mã Hóa Khóa) from KMS
// 4. Send: { encryptedPayload, encryptedDEK, keyId }
```

### Trade-offs E2EE

| Ưu Điểm | Nhược Điểm |
| ------- | ---------- |
| Broker không đọc được content | Không search/filter trên broker |
| Compliance (HIPAA, PCI) | Key distribution phức tạp |
| Zero-trust broker | Schema validation khó hơn |
| | Không dùng được Kafka Streams filter |

---

## Key Management

**KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)** quản lý lifecycle của encryption keys.

```
┌─────────────────────────────────────────────────────────┐
│                  KEY HIERARCHY                           │
│                                                          │
│  CMK (Customer Master Key — Khóa Chủ Khách Hàng)        │
│       │                                                  │
│       ├── DEK per topic/tenant (data keys)              │
│       │       │                                          │
│       │       └── Encrypt message/log segments          │
│       │                                                  │
│       └── Auto-rotation (annual)                        │
└─────────────────────────────────────────────────────────┘
```

| Platform | KMS Service |
| -------- | ----------- |
| **AWS** | AWS KMS |
| **Azure** | Azure Key Vault |
| **GCP** | Cloud KMS |
| **Self-hosted** | HashiCorp Vault |

**Best practices:**

- Không embed encryption keys trong code
- Rotate keys định kỳ (90–365 ngày)
- Separate keys per environment (dev/staging/prod)
- Audit key usage via CloudTrail / audit logs

---

## Performance Impact

| Factor | Impact | Mitigation |
| ------ | ------ | ---------- |
| **TLS handshake** | +1–5ms per new connection | Connection pooling, session resumption |
| **TLS encryption CPU** | 5–15% throughput reduction | AES-NI hardware, TLS 1.3 |
| **At-rest encryption** | 2–5% I/O overhead | Transparent volume encryption |
| **E2EE app-level** | 10–30% depending on payload | Encrypt only sensitive fields |

### Benchmark Reference

```
Kafka (no TLS):     ~500 MB/s produce
Kafka (TLS 1.3):    ~430 MB/s produce  (~14% overhead)
Kafka (TLS + at-rest): ~410 MB/s       (~18% total)

→ Overhead chấp nhận được cho production security
```

---

## Certificate Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Generate │───►│ Deploy   │───►│ Monitor  │───►│ Rotate   │
│ (CA/cert)│    │ to nodes │    │ expiry   │    │ before   │
└──────────┘    └──────────┘    └──────────┘    │ expiry   │
                                                 └──────────┘
```

### Automated Rotation với cert-manager (Kubernetes)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-broker-tls
spec:
  secretName: kafka-broker-tls
  duration: 2160h    # 90 days
  renewBefore: 360h # renew 15 days before expiry
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  dnsNames:
    - broker1.internal
    - broker2.internal
    - kafka.internal
```

### Expiry Monitoring

```
Alert rules:
  - cert_expiry_days < 30  → WARNING
  - cert_expiry_days < 7   → CRITICAL
  - cert_expired           → PAGE on-call

Tools: Prometheus ssl_exporter, cert-manager metrics
```

---

## Anti-Patterns

| Anti-Pattern | Rủi Ro | Fix |
| ------------ | ------ | --- |
| PLAINTEXT listener on production | Full traffic readable | SASL_SSL only |
| TLS chỉ client, không inter-broker | Replication sniffable | inter.broker.protocol=SASL_SSL |
| Self-signed cert không monitor expiry | Outage khi cert hết hạn | cert-manager + alerting |
| `ssl.endpoint.identification.algorithm=` (empty) | Hostname verification disabled | Set to `https` |
| Disable TLS "vì chậm" | Data breach | Tune cipher, use AES-NI |
| Encrypt everything E2EE không cần | Complexity, no broker filtering | Field-level cho PII only |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: TLS in-transit và at-rest khác nhau thế nào?

**Trả lời:** **In-transit** mã hóa data khi truyền trên network (client↔broker, broker↔broker). **At-rest** mã hóa data khi lưu trên disk/volume. Cần **cả hai** — in-transit chống sniffing, at-rest chống disk theft/snapshot leak.

### Câu 2: SASL_SSL vs SSL listener khác gì?

**Trả lời:** **SSL** chỉ có TLS encryption + cert auth (mTLS). **SASL_SSL** = TLS + SASL authentication (SCRAM username/password). Production Kafka thường dùng **SASL_SSL** — kết hợp encryption và credential-based auth.

### Câu 3: Khi nào cần E2EE thay vì chỉ TLS?

**Trả lời:** Khi **broker admin không được trust** (multi-tenant, cross-org), hoặc compliance yêu cầu broker không đọc PII. Internal microservices trust broker → TLS đủ. Thêm field-level encryption cho PII fields là middle ground tốt.

### Câu 4: TLS ảnh hưởng performance bao nhiêu?

**Trả lời:** Thường **5–15%** throughput reduction. Giảm bằng: TLS 1.3, AES-NI hardware, connection pooling, session resumption. Trade-off bảo mật luôn đáng giá — không disable TLS vì performance.

---

**Tiếp theo:** [4-audit-logging.md](./4-audit-logging.md) — Audit trails và compliance requirements.
