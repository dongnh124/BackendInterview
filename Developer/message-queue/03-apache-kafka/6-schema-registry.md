# Schema Registry — Đăng Ký Schema Và Tiến Hóa

> Confluent Schema Registry (Đăng Ký Schema), Avro/Protobuf/JSON Schema, schema evolution (tiến hóa schema), compatibility modes (chế độ tương thích), và tích hợp với Kafka producers/consumers.

## Mục Lục

1. [Tại Sao Cần Schema Registry](#tại-sao-cần-schema-registry)
2. [Schema Registry Architecture](#schema-registry-architecture)
3. [Avro, Protobuf, JSON Schema](#avro-protobuf-json-schema)
4. [Schema Evolution](#schema-evolution)
5. [Compatibility Modes](#compatibility-modes)
6. [Producer & Consumer Integration](#producer--consumer-integration)
7. [Schema Versioning Best Practices](#schema-versioning-best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Schema Registry

Trong microservices event-driven, producers và consumers **evolve độc lập** — không có schema enforcement dễ gây:

| Vấn Đề | Hậu Quả |
| ------ | ------- |
| Producer đổi field | Consumer deserialize fail |
| Không version schema | Breaking change silent |
| JSON tự do | Không contract, khó validate |

**Schema Registry** là **centralized schema store (kho schema tập trung)** — lưu versioned schemas, enforce compatibility rules trước khi register schema mới.

```
Without Schema Registry:
  Producer v2 { orderId, amount, currency }  →  Consumer v1 expect { orderId, amount }
  → DeserializationException at runtime

With Schema Registry:
  Register schema v2 → compatibility check → pass/fail before deploy
  Consumer dùng schema ID trong message → deserialize đúng version
```

---

## Schema Registry Architecture

```
┌─────────────┐     register schema      ┌──────────────────┐
│  Producer   │ ────────────────────────►│ Schema Registry  │
│             │◄── schema ID (e.g. 42) ──│  (REST API)      │
└──────┬──────┘                          └────────┬─────────┘
       │                                          │
       │ message = [magic byte][schema ID][avro data]
       ▼                                          │
┌─────────────┐     fetch schema by ID             │
│   Kafka     │                                    │
│   Broker    │                                    │
└──────┬──────┘                                    │
       │                                          │
       ▼                                          ▼
┌─────────────┐     lookup schema ID     ┌──────────────────┐
│  Consumer   │ ────────────────────────►│ Schema Registry  │
└─────────────┘                          └──────────────────┘
```

### Wire Format (Avro)

```
Byte 0:     Magic byte (0x0)
Bytes 1-4:  Schema ID (4 bytes)
Bytes 5+:   Avro serialized payload
```

Consumer đọc schema ID → fetch schema từ Registry → deserialize — **không cần embed full schema trong message**.

### Subject Naming

```
{topic-name}-value    → schema cho message value
{topic-name}-key      → schema cho message key

Ví dụ: orders-value, orders-key
```

---

## Avro, Protobuf, JSON Schema

| Format | Đặc Điểm | Phổ Biến |
| ------ | -------- | -------- |
| **Avro** | Compact binary, schema evolution mạnh | ⭐⭐⭐ Kafka ecosystem |
| **Protobuf** | Google, strong typing, gRPC friendly | ⭐⭐ Polyglot systems |
| **JSON Schema** | Human readable, validation rules | ⭐ JSON-heavy teams |

### Avro Schema Example

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.commerce.events",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "customerId", "type": "string" },
    { "name": "amount", "type": "long" },
    { "name": "currency", "type": "string", "default": "VND" }
  ]
}
```

### So Sánh

```
Avro:
  + Nhỏ nhất on wire
  + Schema evolution rules rõ ràng
  - Cần codegen hoặc GenericRecord

Protobuf:
  + Performance tốt, tooling mạnh
  + Backward/forward compat tốt
  - Ít native Kafka hơn Avro (đang cải thiện)

JSON Schema:
  + Dễ đọc, debug
  - Payload lớn hơn
  - Evolution phức tạp hơn Avro
```

---

## Schema Evolution

**Schema Evolution (Tiến Hóa Schema)** — thay đổi schema theo thời gian mà không break consumers/producers đang chạy.

### Safe Changes (Thường Backward Compatible)

| Thay Đổi | Mô Tả |
| -------- | ----- |
| Thêm field **optional** với default | Consumer cũ ignore field mới |
| Xóa field | Consumer mới không đọc field cũ |
| Đổi tên field (với alias) | Avro alias support |

### Breaking Changes

| Thay Đổi | Vấn Đề |
| -------- | ------ |
| Xóa required field | Consumer mới fail đọc data cũ |
| Đổi type `int` → `string` | Incompatible |
| Đổi tên field không alias | Consumer cũ không map được |

```
Evolution example:
  v1: { orderId, amount }
  v2: { orderId, amount, currency (default="VND") }  ← backward compatible
  v3: { orderId, amount, currency, lineItems[] }     ← cần check compatibility
```

---

## Compatibility Modes

Schema Registry kiểm tra compatibility khi register schema mới:

| Mode | Quy Tắc | Use Case |
| ---- | ------- | -------- |
| **BACKWARD** (default) | Schema mới đọc được data cũ | Consumer upgrade trước producer |
| **BACKWARD_TRANSITIVE** | Backward với tất cả versions | Strict backward |
| **FORWARD** | Schema cũ đọc được data mới | Producer upgrade trước consumer |
| **FORWARD_TRANSITIVE** | Forward với tất cả versions | Strict forward |
| **FULL** | Backward + Forward | Maximum flexibility |
| **NONE** | Không check | Dev only — nguy hiểm |

```
BACKWARD (phổ biến nhất):
  Deploy consumer mới (schema v2) TRƯỚC
  Producer vẫn gửi v1 → consumer v2 đọc OK (default values)
  Sau đó upgrade producer → gửi v2
```

```bash
# Set compatibility per subject
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://localhost:8081/config/orders-value
```

---

## Producer & Consumer Integration

### Java Producer (Avro)

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", KafkaAvroSerializer.class.getName());
props.put("schema.registry.url", "http://localhost:8081");
props.put("auto.register.schemas", "false"); // production: register qua CI/CD

KafkaProducer<String, OrderCreated> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("orders", orderId, orderEvent));
```

### Java Consumer (Avro)

```java
props.put("value.deserializer", KafkaAvroDeserializer.class.getName());
props.put("specific.avro.reader", "true"); // dùng generated class
// hoặc false → GenericRecord
```

### auto.register.schemas

| Setting | Production |
| ------- | ---------- |
| `true` | Dev — producer tự register |
| `false` | **Prod** — register qua pipeline, kiểm soát version |

### Schema ID Caching

Client cache schema locally — giảm Registry calls. Cache invalidate khi gặp schema ID mới.

---

## Schema Versioning Best Practices

| Practice | Mô Tả |
| -------- | ----- |
| **Schema as contract** | Review schema change như API change |
| **CI compatibility check** | Test register schema mới trước deploy |
| **Default values** | Mọi field mới nên có default |
| **Never break BACKWARD** | Trừ khi tạo topic/version mới |
| **Topic versioning** | `orders.v1`, `orders.v2` cho breaking change lớn |
| **Document changelog** | Schema migration notes cho consumers |

```
Breaking change workflow:
1. Không thể evolve → tạo topic mới orders.v2
2. Dual-write period: producer ghi cả v1 và v2
3. Migrate consumers sang v2
4. Deprecate v1 topic
```

---

## Troubleshooting

| Lỗi | Nguyên Nhân | Fix |
| --- | ----------- | --- |
| `SerializationException` | Schema mismatch | Check compatibility, consumer version |
| `Schema not found` | Wrong schema ID | Verify Registry connectivity |
| Register rejected | Compatibility fail | Fix schema hoặc đổi compatibility mode |
| `Unknown magic byte` | Non-Avro message on Avro topic | Mixed serializers — enforce convention |

```bash
# List schema versions
curl http://localhost:8081/subjects/orders-value/versions

# Get specific version
curl http://localhost:8081/subjects/orders-value/versions/3
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Schema Registry dùng để làm gì?

**Gợi ý trả lời:** Lưu **versioned schemas** tập trung, gắn **schema ID** vào message, enforce **compatibility** khi evolve. Cho phép producers/consumers deploy độc lập an toàn hơn.

### Câu 2: BACKWARD compatibility nghĩa là gì?

**Gợi ý trả lời:** Consumer dùng **schema mới** vẫn đọc được message serialized với **schema cũ**. Thường đạt bằng thêm optional fields có default. Deploy consumer trước, producer sau.

### Câu 3: Avro vs JSON trong Kafka?

**Gợi ý trả lời:** **Avro** compact, schema evolution tốt, tích hợp Schema Registry native. **JSON** dễ debug nhưng payload lớn, evolution khó. Production event streaming thường chọn Avro hoặc Protobuf.

### Câu 4: Làm sao handle breaking schema change?

**Gợi ý trả lời:** Không force incompatible evolution — tạo **topic mới** hoặc **subject version mới**, dual-write migration period, migrate consumers, deprecate cũ.

### Câu 5: Message chứa schema hay chỉ schema ID?

**Gợi ý trả lời:** Chỉ **schema ID** (4 bytes) + Avro payload — schema full fetch từ Registry. Tiết kiệm bandwidth so với embed JSON schema mỗi message.

---

**Xem tiếp:** [7-kafka-connect.md](./7-kafka-connect.md) — Kafka Connect và CDC integration.
