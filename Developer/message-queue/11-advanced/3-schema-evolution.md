# Event Schema Evolution — Tiến Hóa Schema Sự Kiện

> Schema evolution (Tiến Hóa Schema) ở quy mô enterprise: backward/forward compatibility (Tương Thích Ngược/Xuôi), multi-team governance (Quản Trị Đa Nhóm), breaking change migration, contract testing, và chiến lược versioning cho event-driven systems.

## Mục Lục

1. [Tại Sao Schema Evolution Quan Trọng](#tại-sao-schema-evolution-quan-trọng)
2. [Compatibility Modes Chi Tiết](#compatibility-modes-chi-tiết)
3. [Safe vs Breaking Changes](#safe-vs-breaking-changes)
4. [Multi-Team Governance Model](#multi-team-governance-model)
5. [Breaking Change Migration Strategies](#breaking-change-migration-strategies)
6. [Contract Testing & CI/CD](#contract-testing--cicd)
7. [Schema Formats Comparison](#schema-formats-comparison)
8. [CDC & Schema Evolution](#cdc--schema-evolution)
9. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Schema Evolution Quan Trọng

Trong event-driven architecture, **event schema là API contract (Hợp Đồng API)** giữa producers và consumers — thường **nhiều team deploy độc lập**.

```
Team A (Orders):     deploy producer v3 — thêm field loyaltyPoints
Team B (Analytics):  vẫn chạy consumer v1 — expect schema cũ
Team C (Billing):  consumer v2 — đã handle optional fields

Không có evolution rules → runtime SerializationException → production incident
```

| Vấn Đề Không Governance | Hậu Quả |
| ----------------------- | ------- |
| Producer deploy schema mới không tương thích | Consumer crash |
| Không version schema | Không rollback được |
| JSON tự do không schema | Silent data corruption |
| Nhiều team share topic | Ai được đổi schema? |

> **Nền tảng:** Xem [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) cho Schema Registry basics. File này đi sâu **enterprise patterns**.

---

## Compatibility Modes Chi Tiết

**Schema Registry** enforce compatibility khi register schema version mới.

### Visual: Reader-Writer Compatibility

```
BACKWARD (Consumer upgrade trước):
  Writer (Producer) v1 ──► data ──► Reader (Consumer) v2  ✅
  Writer v2 ──► data ──► Reader v1  ❌ (until consumer upgraded)

FORWARD (Producer upgrade trước):
  Writer v2 ──► data ──► Reader v1  ✅
  Writer v1 ──► data ──► Reader v2  ❌

FULL (Both directions):
  Any version reader ↔ any version writer (within range)  ✅
```

### Compatibility Matrix

| Mode | Rule | Deploy Order | Phổ Biến |
| ---- | ---- | ------------ | -------- |
| **BACKWARD** | New schema reads old data | Consumer first → Producer | ⭐⭐⭐ Default |
| **BACKWARD_TRANSITIVE** | Backward vs ALL old versions | Consumer first | Strict |
| **FORWARD** | Old schema reads new data | Producer first → Consumer | Ít hơn |
| **FORWARD_TRANSITIVE** | Forward vs ALL old versions | Producer first | Strict |
| **FULL** | Backward + Forward | Flexible | Maximum safety |
| **FULL_TRANSITIVE** | Full vs ALL versions | Flexible | Enterprise |
| **NONE** | No check | Any | Dev only ⚠️ |

### Transitive vs Non-Transitive

```
Non-transitive BACKWARD:
  v3 compatible with v2 ✅
  v3 compatible with v1 ❓ (not checked)

Transitive BACKWARD:
  v3 compatible with v2 ✅
  v3 compatible with v1 ✅ (also checked)
  → Safer for long-lived topics with many versions
```

**Khuyến nghị production:** `BACKWARD` hoặc `FULL` cho hầu hết topics; `BACKWARD_TRANSITIVE` cho critical shared events.

---

## Safe vs Breaking Changes

### Avro Safe Changes (Typically BACKWARD Compatible)

| Change | Compatible | Ghi Chú |
| ------ | ---------- | ------- |
| Thêm field **optional** với **default** | ✅ BACKWARD | Consumer cũ ignore |
| Xóa field (có default implicit) | ✅ BACKWARD | Consumer mới dùng default |
| Thêm field required **không default** | ❌ BREAKING | Old data thiếu field |
| Đổi type `int` → `long` | ✅ (Avro promotion) | Avro type promotion rules |
| Đổi type `int` → `string` | ❌ BREAKING | |
| Đổi tên field (không alias) | ❌ BREAKING | Dùng Avro **aliases** |
| Enum: thêm symbol | ⚠️ | BACKWARD yes, FORWARD no |

### Avro Aliases cho Rename

```json
{
  "name": "customerId",
  "type": "string",
  "aliases": ["clientId", "user_id"]
}
```

Consumer đọc `customerId`, data cũ có `clientId` → Avro resolve qua alias.

### Protobuf Field Numbers

```
Protobuf dùng field NUMBER (không phải name) trên wire:
  Field 1: orderId
  Field 2: amount
  Thêm field 3: currency  ← safe (new number)

KHÔNG reuse field number — dù đổi tên
  reserve deprecated fields: reserved 3;
```

### Decision Tree: Safe hay Breaking?

```
Thay đổi schema mới
    │
    ├── Thêm optional field + default? ──► Safe (BACKWARD)
    │
    ├── Xóa field? ──► Safe nếu consumer mới không cần
    │
    ├── Đổi type? ──► Check Avro promotion rules
    │
    ├── Rename field? ──► Dùng alias hoặc Breaking
    │
    ├── Restructure nested object? ──► Likely Breaking
    │
    └── Semantic change (amount: cents → dollars)? ──► Breaking (business logic)
```

---

## Multi-Team Governance Model

### Schema Ownership

```
┌─────────────────────────────────────────────────────────────┐
│                  SCHEMA GOVERNANCE MODEL                     │
│                                                             │
│  Schema Owner (Producer Team)                               │
│    ├── Define schema                                        │
│    ├── Propose changes (PR to schema repo)                  │
│    ├── Compatibility check in CI                            │
│    └── Approve register to Schema Registry                  │
│                                                             │
│  Schema Consumers (Downstream Teams)                        │
│    ├── Review breaking changes (notification)               │
│    ├── Contract test against new schema                     │
│    └── SLA: respond within X days for breaking proposals    │
│                                                             │
│  Platform Team                                              │
│    ├── Schema Registry ops                                  │
│    ├── Global compatibility defaults                        │
│    └── Audit trail & compliance                             │
└─────────────────────────────────────────────────────────────┘
```

### Schema Registry Subject Strategy

| Strategy | Pattern | Use Case |
| -------- | ------- | -------- |
| **Topic-RecordName** | `{topic}-{record-name}` | Multiple event types per topic |
| **Topic-Name** | `{topic}-value` | One schema per topic (phổ biến) |
| **Record-Name** | `{record-name}` | Schema shared across topics |

```bash
# Topic-Name (default Confluent)
orders-value    → schema for orders topic value
orders-key      → schema for orders topic key
```

### Role-Based Access

```
Producer team:  WRITE subject orders-value
Consumer teams: READ subject orders-value
Platform:       ADMIN Schema Registry
CI/CD pipeline:  WRITE (register on deploy)
Developers:     READ only (no auto.register in prod)
```

---

## Breaking Change Migration Strategies

Khi thay đổi **không thể evolve** (restructure lớn, semantic change), cần migration strategy.

### Strategy 1: New Topic Version

```
orders.v1  (existing, deprecated)
orders.v2  (new schema)

Timeline:
  Week 1-2: Create orders.v2, dual-write (producer ghi cả v1 và v2)
  Week 3-4: Migrate consumers v1 → v2
  Week 5:   Stop writing v1
  Week 6+:  Archive/delete orders.v1
```

```
Producer dual-write:
  send(orders.v1, legacyFormat(event))
  send(orders.v2, newFormat(event))
```

### Strategy 2: New Subject / Schema Version (Same Topic)

```
Chỉ khi compatibility mode = NONE (không khuyến khích)
Hoặc dùng topic mới — safer
```

### Strategy 3: Updater/Translator Service

```
                    ┌─────────────────┐
orders.v2 topic ──► │ Schema          │ ──► orders.v1 format
(new consumers)     │ Translator      │     (legacy consumers)
                    │ Service         │
                    └─────────────────┘
```

Temporary bridge trong migration period — remove sau khi all consumers upgraded.

### Strategy 4: Event Type Versioning (Envelope Pattern)

```json
{
  "eventType": "OrderCreated",
  "schemaVersion": 2,
  "payload": { "orderId": "123", "amount": { "value": 100, "currency": "VND" } }
}
```

Consumer switch on `schemaVersion` — multiple versions trong cùng topic.

| Strategy | Downtime | Complexity | Dual-write Period |
| -------- | -------- | ---------- | ----------------- |
| New topic | Zero | Medium | Yes |
| Translator service | Zero | High | Yes |
| Envelope versioning | Zero | Low-Medium | Optional |
| Big bang (stop all) | Yes | Low | No |

---

## Contract Testing & CI/CD

### Schema-First Workflow

```
1. Author schema change (Avro .avsc file) in Git
2. PR → CI runs compatibility check against Registry
3. Consumer teams review PR (auto-notify via CODEOWNERS)
4. Merge → CI registers schema to Registry (staging)
5. Deploy producer (uses new schema ID)
6. Deploy consumers (already compatible from step 2)
```

### Compatibility Check in CI

```bash
# Confluent Schema Registry Maven Plugin / CLI
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data @orders-v3.avsc \
  "http://schema-registry:8081/compatibility/subjects/orders-value/versions/latest"

# Response: {"is_compatible": true} or false
```

```yaml
# GitHub Actions example
- name: Schema Compatibility Check
  run: |
    for schema in schemas/*.avsc; do
      subject=$(basename $schema .avsc)-value
      curl -sf -X POST \
        --data @$schema \
        "$SCHEMA_REGISTRY/compatibility/subjects/$subject/versions/latest" \
        | jq -e '.is_compatible == true'
    done
```

### Consumer Contract Tests

```java
@Test
void consumerHandlesAllSchemaVersions() {
    for (SchemaVersion version : registry.getVersions("orders-value")) {
        GenericRecord record = generateSampleRecord(version);
        assertDoesNotThrow(() -> orderHandler.process(record));
    }
}
```

### Production Settings

| Setting | Dev | Production |
| ------- | --- | ---------- |
| `auto.register.schemas` | `true` | **`false`** |
| Schema registration | Manual / CI | **CI/CD only** |
| Compatibility mode | BACKWARD | BACKWARD or FULL |
| `use.latest.version` | `true` | **`false`** (pin version) |

---

## Schema Formats Comparison

| Tiêu Chí | Avro | Protobuf | JSON Schema |
| -------- | ---- | -------- | ----------- |
| **Wire size** | Nhỏ | Nhỏ nhất | Lớn |
| **Evolution rules** | Rõ ràng, Registry native | Field numbers, reserved | Phức tạp hơn |
| **Human readable** | ❌ (binary) | ❌ (binary) | ✅ |
| **Kafka ecosystem** | ⭐⭐⭐ | ⭐⭐ (improving) | ⭐ |
| **Code generation** | avro-maven-plugin | protoc | jsonschema2pojo |
| **Schema in message** | ID only (4 bytes) | Unknown fields skipped | Often embedded |
| **Cross-language** | Good | Excellent | Good |

### Khi Nào Chọn Format

```
Avro:       Kafka-native, Schema Registry, analytics pipelines
Protobuf:   gRPC + Kafka, polyglot microservices, performance critical
JSON Schema: Human debug priority, small teams, low volume
```

---

## CDC & Schema Evolution

CDC events có th challenges riêng khi database schema thay đổi.

### Debezium Schema Change Events

```
ALTER TABLE orders ADD COLUMN discount DECIMAL;
  → Debezium emit schema change event
  → New Avro schema version auto-registered (if configured)
  → Downstream consumers receive new field (nullable, no default for existing rows)
```

### CDC Schema Evolution Rules

| DB Change | CDC Impact | Downstream Action |
| --------- | ---------- | ----------------- |
| ADD COLUMN nullable | New field in events | Consumer ignore (BACKWARD) |
| ADD COLUMN NOT NULL | Existing rows null in snapshot | Need default or backfill |
| DROP COLUMN | Field missing in events | Consumer must not require field |
| RENAME COLUMN | Appears as drop + add | Breaking — use SMT alias |
| CHANGE TYPE | Incompatible | Breaking — new topic or transform |

### SMT cho Schema Mapping

```json
{
  "transforms": "unwrap,renameField",
  "transforms.renameField.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.renameField.renames": "customer_id:customerId"
}
```

---

## Thiết Kế Thực Tế

### Enterprise Event Catalog

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT CATALOG                             │
│                                                             │
│  Event: OrderCreated                                        │
│  Owner: commerce-team@company.com                           │
│  Topic: orders (subject: orders-value)                      │
│  Schema versions: v1, v2, v3 (current: v3)                  │
│  Compatibility: BACKWARD_TRANSITIVE                         │
│  Consumers: analytics, billing, notification (5 teams)      │
│  Changelog:                                                 │
│    v2 (2025-01): +currency field (default VND)              │
│    v3 (2025-06): +lineItems array (default [])              │
│  Breaking change policy: 30-day notice, new topic required  │
└─────────────────────────────────────────────────────────────┘
```

### Production Checklist

```
□ Compatibility mode set per subject (not global NONE)
□ auto.register.schemas=false in production
□ Schema changes via CI/CD with compatibility gate
□ Schema ownership documented (CODEOWNERS / event catalog)
□ Consumer contract tests in CI
□ Breaking change → new topic version workflow documented
□ Default values for all new optional fields
□ Avro aliases for field renames
□ Monitor Schema Registry availability (single point of failure)
□ Schema Registry HA (multi-node) in production
□ Deprecation policy: N versions retained, sunset timeline
```

### Anti-Patterns

```
❌ NONE compatibility in production
❌ auto.register.schemas=true — rogue producer breaks consumers
❌ Breaking change on shared topic without migration plan
❌ JSON without schema enforcement — "Chúng ta sẽ parse linh hoạt"
❌ Không notify consumer teams trước schema change
❌ Reuse Protobuf field numbers
❌ Semantic change (units, meaning) without version bump
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: BACKWARD vs FORWARD compatibility?

**Gợi ý trả lời:** **BACKWARD**: consumer schema **mới** đọc data từ producer schema **cũ** — deploy consumer trước. **FORWARD**: consumer schema **cũ** đọc data từ producer schema **mới** — deploy producer trước. Kafka ecosystem default **BACKWARD** — thêm optional fields với default.

### Câu 2: Handle breaking schema change trong production?

**Gợi ý trả lời:** Không force incompatible evolution. Tạo **topic mới** (`orders.v2`), **dual-write period**, migrate consumers, deprecate v1. Hoặc **envelope versioning** với consumer switch on version. Communicate 30-day notice cho downstream teams.

### Câu 3: Schema governance multi-team?

**Gợi ý trả lời:** **Schema owner** (producer team) propose changes via Git PR. **CI compatibility check** against Registry. **Consumer teams review** breaking changes. **Platform team** ops Registry. `auto.register.schemas=false` — chỉ CI register. **Event catalog** document ownership và changelog.

### Câu 4: Avro vs Protobuf cho Kafka?

**Gợi ý trả lời:** **Avro** — native Schema Registry integration, compact, evolution rules rõ. **Protobuf** — better polyglot/gRPC, field numbers evolution, slightly better performance. Chọn Avro cho Kafka-heavy; Protobuf khi đã standardize Protobuf across services.

### Câu 5: Transitive compatibility khi nào cần?

**Gợi ý trả lời:** Topic có **nhiều schema versions** tích lũy (v1–v10). Non-transitive chỉ check latest vs new; **transitive** check new vs **all** old versions — prevent issue khi v10 compatible v9 nhưng không compatible v3 còn data on topic.

### Câu 6: CDC schema change ảnh hưởng downstream?

**Gợi ý trả lời:** `ALTER TABLE ADD COLUMN` → Debezium emit new field — OK nếu BACKWARD (nullable/default). `DROP/RENAME/CHANGE TYPE` → potentially breaking. Downstream consumers cần ignore unknown fields, không require dropped fields. Document DB migration coordination với CDC consumers.

---

**Xem tiếp:** [4-serverless-processing.md](./4-serverless-processing.md) — Lambda và Cloud Functions.

**Liên quan:** [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) — Schema Registry fundamentals.
