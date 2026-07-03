# Producers & Serialization — Cấu Hình Producer

> Cấu hình Kafka Producer: acknowledgment (acks — xác nhận), batching (gom lô), compression (nén), idempotence (tính bất biến gửi), transactional producer (producer giao dịch), và serialization (tuần tự hóa) key/value.

## Mục Lục

1. [Producer Overview](#producer-overview)
2. [Acknowledgment (acks)](#acknowledgment-acks)
3. [Batching & Linger](#batching--linger)
4. [Compression](#compression)
5. [Idempotent Producer](#idempotent-producer)
6. [Transactional Producer](#transactional-producer)
7. [Serialization](#serialization)
8. [Error Handling & Retries](#error-handling--retries)
9. [Producer Config Reference](#producer-config-reference)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Producer Overview

**Kafka Producer** gửi records tới topic partitions — client tự handle partitioning, batching, compression, và retry.

```
Application
    │
    ▼
Producer API
    │ serialize key/value
    │ partition (key hash / custom)
    │ accumulate batch
    │ compress
    ▼
Kafka Broker (leader partition)
```

```javascript
// kafkajs example
const producer = kafka.producer({
  allowAutoTopicCreation: false,
  idempotent: true,
});

await producer.send({
  topic: 'orders',
  messages: [{ key: 'ord-1', value: JSON.stringify(payload) }],
  acks: -1, // all
});
```

---

## Acknowledgment (acks)

**acks** xác định producer chờ bao nhiêu replica xác nhận trước khi coi send thành công.

| acks | Hành Vi | Durability | Latency |
| ---- | ------- | ---------- | ------- |
| `0` | Fire-and-forget — không chờ | Thấp nhất — có thể mất | Thấp nhất |
| `1` | Chờ leader ack | Mất nếu leader die trước replicate | Trung bình |
| `all` / `-1` | Chờ tất cả ISR ack | Cao nhất (với min.insync.replicas) | Cao nhất |

```
acks=0:
  Producer ──► Leader ──► (không chờ) → success
  Leader crash ngay sau → message mất

acks=1:
  Producer ──► Leader ──► ack → success
  Leader crash trước khi follower replicate → mất

acks=all, min.insync.replicas=2:
  Producer ──► Leader ──► ISR [L, F1] ack → success
  Chịu leader fail nếu còn replica trong ISR
```

### Production Khuyến Nghị

```properties
acks=all
min.insync.replicas=2
replication.factor=3
```

> **Trade-off:** `acks=all` + network issue → latency tăng, timeout. Metrics: `record-error-rate`, `request-latency-avg`.

---

## Batching & Linger

Producer **gom nhiều records** thành batch trước khi gửi — tăng throughput.

| Config | Mô Tả | Default |
| ------ | ----- | ------- |
| `batch.size` | Kích thước batch tối đa (bytes) | 16384 |
| `linger.ms` | Chờ thêm để gom batch | 0 |
| `buffer.memory` | Memory buffer tổng | 32 MB |

```
linger.ms=0:  gửi ngay khi có record (low latency)
linger.ms=5:  chờ 5ms gom thêm → batch lớn hơn → higher throughput
```

```properties
# Throughput tuning
batch.size=65536
linger.ms=10
```

> **Lưu ý:** `linger.ms` tăng → latency tăng nhẹ. Balance theo SLA.

---

## Compression

| Codec | Tỷ Lệ Nén | CPU | Khuyến Nghị |
| ----- | --------- | --- | ----------- |
| `none` | 1x | Thấp | Dev, message nhỏ |
| `lz4` | ~2–3x | Thấp | **Default production** |
| `snappy` | ~2x | Thấp | Alternative |
| `zstd` | ~3–4x | Trung bình | Bandwidth constrained |
| `gzip` | ~3–4x | Cao | Legacy |

```properties
compression.type=lz4
```

Compression áp dụng trên **batch** — hiệu quả hơn nén từng message.

---

## Idempotent Producer

**Idempotent Producer (Producer Bất Biến)** — Kafka tự dedup trong phạm vi producer session:

```properties
enable.idempotence=true
# Tự set: acks=all, retries>0, max.in.flight.requests.per.connection=5
```

```
Cơ chế:
- Producer ID (PID) + Sequence Number per partition
- Broker reject duplicate sequence → không ghi trùng

Phạm vi: retry do network — KHÔNG cover app gửi 2 lần logic
```

| Scenario | Idempotent Producer |
| -------- | ------------------- |
| Network retry duplicate | ✅ Dedup |
| App gọi send() 2 lần | ❌ 2 messages |
| Producer restart (new PID) | ❌ Có thể duplicate |

> **Production:** Luôn bật `enable.idempotence=true` — overhead thấp, an toàn hơn.

---

## Transactional Producer

**Transactional Producer** — exactly-once **write** across multiple partitions/topics trong một transaction:

```java
producer.initTransactions();
try {
  producer.beginTransaction();
  producer.send(record1);
  producer.send(record2);
  producer.commitTransaction();
} catch (Exception e) {
  producer.abortTransaction();
}
```

```properties
transactional.id=my-app-txn-1  # unique per producer instance
enable.idempotence=true
```

### Use Cases

| Use Case | Mô Tả |
| -------- | ----- |
| **Read-process-write** | Consume → transform → produce atomically |
| **Multi-topic write** | Ghi nhiều topic cùng lúc — all or nothing |
| **EOS pipeline** | Kết hợp transactional consumer |

### Isolation Level

Consumer cần:

```properties
isolation.level=read_committed  # chỉ đọc committed messages
# default read_uncommitted — thấy cả aborted txn
```

> **Thực tế:** Transactional API phức tạp — nhiều team dùng at-least-once + idempotent consumer thay thế.

---

## Serialization

### Serializers

| Format | Serializer | Use Case |
| ------ | ---------- | -------- |
| String | `StringSerializer` | JSON string, simple text |
| Bytes | `ByteArraySerializer` | Raw binary |
| Avro | `KafkaAvroSerializer` | Schema Registry integration |
| JSON | `KafkaJsonSerializer` | Schema-less JSON |
| Protobuf | `KafkaProtobufSerializer` | Strong typing, compact |

```java
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
    StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
    KafkaAvroSerializer.class.getName());
props.put("schema.registry.url", "http://localhost:8081");
```

### JSON vs Avro

```
JSON:
  ✅ Human readable, dễ debug
  ❌ Không schema enforcement, payload lớn

Avro + Schema Registry:
  ✅ Compact binary, schema evolution
  ✅ Compatibility rules enforced
  ❌ Cần Schema Registry infrastructure
```

---

## Error Handling & Retries

```properties
retries=2147483647           # retry indefinitely (với delivery.timeout.ms)
delivery.timeout.ms=120000     # tổng timeout cho send
retry.backoff.ms=100
request.timeout.ms=30000
```

### Exception Types

| Exception | Hành Động |
| --------- | --------- |
| `RetriableException` | Retry tự động |
| `NotEnoughReplicasException` | ISR < min — check cluster health |
| `RecordTooLargeException` | Giảm message size hoặc tăng `max.message.bytes` |
| `SerializationException` | Fix serializer/schema — không retry |

```java
producer.send(record, (metadata, exception) -> {
  if (exception != null) {
    if (exception instanceof RetriableException) {
      // đã retry — log và alert nếu vẫn fail
    } else {
      // non-retriable — dead letter hoặc alert
    }
  }
});
```

---

## Producer Config Reference

| Config | Production Value | Ghi Chú |
| ------ | ---------------- | ------- |
| `acks` | `all` | Durability |
| `enable.idempotence` | `true` | Dedup retry |
| `compression.type` | `lz4` | Throughput |
| `linger.ms` | `5–20` | Batch tuning |
| `batch.size` | `32KB–64KB` | Theo message size |
| `max.in.flight.requests.per.connection` | `5` (với idempotence) | Ordering vs throughput |
| `delivery.timeout.ms` | `120000` | Fail sau 2 phút |

### max.in.flight.requests.per.connection

```
= 1: strict ordering khi retry (chậm hơn)
= 5: higher throughput, idempotence vẫn đảm bảo no duplicate
> 5: không compatible với idempotence
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: acks=1 vs acks=all?

**Gợi ý trả lời:** **acks=1** chỉ chờ leader — nhanh hơn nhưng mất data nếu leader crash trước replicate. **acks=all** chờ ISR — an toàn với `min.insync.replicas=2` và RF=3. Production dùng `acks=all`.

### Câu 2: Idempotent producer giải quyết gì?

**Gợi ý trả lời:** Loại duplicate do **network retry** — broker track PID + sequence number. Không ngăn app logic gửi 2 lần. Kết hợp at-least-once consumer vẫn cần idempotent handler.

### Câu 3: Làm sao tăng producer throughput?

**Gợi ý trả lời:** Tăng `batch.size`, `linger.ms`, bật `compression.type=lz4`, tăng partitions, `max.in.flight.requests=5` với idempotence. Monitor broker disk và network.

### Câu 4: Transactional producer khi nào cần?

**Gợi ý trả lời:** Khi cần **atomic write** nhiều partitions hoặc read-process-write exactly-once. Phức tạp — chỉ dùng khi business yêu cầu EOS và team có kinh nghiệm vận hành.

### Câu 5: Message quá lớn — xử lý thế nào?

**Gợi ý trả lời:** Default `max.message.bytes` ~1MB. Options: (1) Tăng broker/producer limit — không khuyến nghị quá lớn. (2) **Claim check pattern** — lưu payload S3/DB, gửi reference qua Kafka. (3) Chia nhỏ message.

---

**Xem tiếp:** [5-offset-management.md](./5-offset-management.md) — quản lý offset và commit strategy.
