# Throughput Tuning — Tối Ưu Thông Lượng

> Throughput (Thông Lượng) đo bằng messages/second hoặc MB/second. Tuning đúng ở producer và consumer có thể tăng throughput **5–20x** mà không cần thêm hardware — nhưng luôn trade-off với latency (độ trễ).

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Đo Lường Throughput](#đo-lường-throughput)
3. [Producer Tuning — Kafka](#producer-tuning--kafka)
4. [Producer Tuning — RabbitMQ](#producer-tuning--rabbitmq)
5. [Consumer Tuning](#consumer-tuning)
6. [Compression (Nén)](#compression-nén)
7. [Pipelining (Xử Lý Pipeline)](#pipelining-xử-lý-pipeline)
8. [Serialization Impact](#serialization-impact)
9. [Benchmark Checklist](#benchmark-checklist)
10. [Anti-Patterns](#anti-patterns)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              THROUGHPUT TUNING — QUICK WINS                        │
├─────────────────────────────────────────────────────────────────┤
│  Kafka Producer:  batch.size ↑, linger.ms 10-50, compression   │
│  Kafka Consumer:   fetch.min.bytes ↑, max.poll.records ↑         │
│  RabbitMQ:         publisher confirms async, prefetch tune       │
│  Universal:        binary serialization (Avro/Protobuf) > JSON   │
└─────────────────────────────────────────────────────────────────┘
```

| Lever (Đòn Bẩy) | Throughput Impact | Latency Impact |
| --------------- | ----------------- | -------------- |
| **Batching (Gom Lô)** | ⬆⬆⬆ Cao | ⬆ Tăng nhẹ |
| **Compression (Nén)** | ⬆⬆ Trung bình–cao | ⬇ Giảm (ít data transfer) |
| **Async produce** | ⬆⬆⬆ Cao | ⬆ Tăng (buffer) |
| **Parallel consumers** | ⬆⬆ (có giới hạn) | ➡ Ổn định |
| **acks=all** | ⬇ Giảm | ⬆ Tăng |

---

## Đo Lường Throughput

**Không tune mù** — đo baseline trước khi thay đổi.

### Metrics Cần Thu Thập

| Metric | Công Thức / Nguồn | Mục Đích |
| ------ | ----------------- | -------- |
| **Messages/sec** | `delta(messages) / delta(time)` | Throughput tổng |
| **Bytes/sec** | `messages/sec × avg_message_size` | Network/disk load |
| **P50/P99 latency** | Histogram end-to-end | User-facing SLA |
| **Producer batch rate** | Kafka `batch-size-avg` | Hiệu quả batching |
| **Consumer poll time** | `poll idle ratio` | Consumer bottleneck |

### Kafka Benchmark Tools

```bash
# Producer benchmark
kafka-producer-perf-test \
  --topic throughput-test \
  --num-records 5000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:9092 \
    acks=1 \
    batch.size=65536 \
    linger.ms=20 \
    compression.type=lz4

# Consumer benchmark
kafka-consumer-perf-test \
  --topic throughput-test \
  --messages 5000000 \
  --bootstrap-server localhost:9092 \
  --fetch-size 1048576
```

### RabbitMQ Benchmark

```bash
# rabbitmq-perf-test (official tool)
java -jar perf-test.jar \
  --uri amqp://localhost \
  --queue perf-queue \
  --producers 4 \
  --consumers 4 \
  --size 1024 \
  --rate 10000
```

---

## Producer Tuning — Kafka

Xem chi tiết tại [03-apache-kafka/4-producers-serialization.md](../03-apache-kafka/4-producers-serialization.md).

### Batching — Gom Lô

Producer gom nhiều records thành một request HTTP tới broker.

| Config | Mô Tả | Khuyến Nghị High Throughput |
| ------ | ----- | --------------------------- |
| `batch.size` | Kích thước batch tối đa (bytes) | `65536` (64 KB) – `131072` (128 KB) |
| `linger.ms` | Chờ thêm để gom batch | `10` – `50` ms |
| `buffer.memory` | Buffer memory tổng | Tăng nếu `RecordAccumulator` full |

```
linger.ms = 0:
  Record 1 ──► send ngay
  Record 2 ──► send ngay     → Nhiều request nhỏ, latency thấp

linger.ms = 20:
  Record 1 ──┐
  Record 2 ──┼──► batch send  → Ít request, throughput cao
  Record 3 ──┘
```

```properties
# High throughput profile
batch.size=65536
linger.ms=20
compression.type=lz4
acks=1                    # Trade durability cho speed (non-critical)
# acks=all                # Production critical data
```

### acks Trade-off

| acks | Throughput | Durability | Khi Nào Dùng |
| ---- | ---------- | ---------- | ------------ |
| `0` | Cao nhất | Thấp nhất | Metrics, telemetry |
| `1` | Cao | Trung bình | Log ingestion |
| `all` | Thấp hơn | Cao nhất | Financial, orders |

### Idempotent Producer

`enable.idempotence=true` (mặc định khi `acks=all`) — **không giảm throughput đáng kể** nhưng tránh duplicate từ producer retry. Luôn bật cho production.

---

## Producer Tuning — RabbitMQ

### Publisher Confirms Async

```javascript
// Async confirms — không block mỗi message
const confirmChannel = await connection.createConfirmChannel();
const pending = new Map();
let seq = 0;

function publish(msg) {
  const id = seq++;
  pending.set(id, msg);
  confirmChannel.publish('exchange', 'routing.key', Buffer.from(msg));
  confirmChannel.once('drain', () => { /* buffer trống, tiếp tục */ });
}

confirmChannel.on('ack', (msg) => {
  pending.delete(msg.properties.messageId);
});
```

### Connection & Channel Pool

| Pattern | Throughput | Ghi Chú |
| ------- | ---------- | ------- |
| 1 connection, 1 channel | Thấp | Đơn giản, dev only |
| 1 connection, N channels | Trung bình | Khuyến nghị |
| N connections (pool) | Cao | Multi-threaded apps |

> **Lưu ý:** Mỗi connection tốn ~100 KB RAM trên broker. Không tạo quá nhiều connection.

### Message Persistence Trade-off

```javascript
// Persistent — ghi disk, chậm hơn
channel.publish('ex', 'key', buf, { persistent: true });

// Non-persistent — memory only, nhanh hơn 2-10x
channel.publish('ex', 'key', buf, { persistent: false });
```

---

## Consumer Tuning

### Kafka Consumer Fetch

| Config | Mô Tả | High Throughput |
| ------ | ----- | --------------- |
| `fetch.min.bytes` | Chờ đủ data trước khi trả về | `1048576` (1 MB) |
| `fetch.max.wait.ms` | Max chờ nếu chưa đủ `fetch.min.bytes` | `500` |
| `max.poll.records` | Records mỗi poll | `500` – `1000` |
| `max.partition.fetch.bytes` | Max bytes/partition/fetch | `1048576` |

```javascript
const consumer = kafka.consumer({
  'fetch.min.bytes': 1048576,
  'max.poll.records': 500,
  'max.partition.fetch.bytes': 1048576,
});
```

### RabbitMQ Prefetch

**Prefetch (QoS — Quality of Service)** — số message unacked broker gửi trước cho consumer.

```javascript
// High throughput — prefetch cao (nếu handler nhanh)
channel.prefetch(100);

// Low latency / fair dispatch — prefetch thấp
channel.prefetch(1);  // Work queue pattern
```

| Prefetch | Throughput | Fairness | Risk |
| -------- | ---------- | -------- | ---- |
| `1` | Thấp | Cao (round-robin đều) | Không |
| `10-50` | Trung bình | Trung bình | Một consumer giữ nhiều msg |
| `100+` | Cao | Thấp | Memory, uneven nếu handler chậm |

### Batch Processing Trong Handler

```javascript
// Thay vì xử lý từng message → DB
for (const msg of batch) {
  await db.insert(msg);  // N round-trips
}

// Batch insert — 1 round-trip
await db.insertMany(batch.map(parseMessage));
```

---

## Compression (Nén)

| Codec | Compression Ratio | CPU | Throughput | Khuyến Nghị |
| ----- | ----------------- | --- | ---------- | ----------- |
| **none** | 1x | Thấp | Network-bound | Message nhỏ (<1 KB) |
| **lz4** | 2-3x | Thấp | **Cao nhất** | **Default production** |
| **snappy** | 2-3x | Thấp | Cao | Legacy, tương đương lz4 |
| **zstd** | 3-5x | Trung bình | Cao | Large messages, Kafka 2.1+ |
| **gzip** | 3-5x | Cao | Thấp | Archival, không real-time |

```
Uncompressed JSON 1 KB × 100,000 msg/s = 100 MB/s network
lz4 compressed ~400 B × 100,000 msg/s = 40 MB/s network
→ Throughput tăng vì ít I/O, CPU lz4 rất nhẹ
```

**Rule of thumb:** Bật compression khi message > 1 KB hoặc network/disk là bottleneck.

---

## Pipelining (Xử Lý Pipeline)

**Pipelining** — xử lý song song nhiều stage mà không chờ stage trước hoàn thành hoàn toàn.

```
Sequential (Tuần Tự):
  [Parse]──►[Validate]──►[DB Write]──►[Publish]
  Total: 10 + 5 + 50 + 10 = 75ms/msg → 13 msg/s

Pipelined:
  Msg1: [Parse]──►[Validate]──►[DB]──►[Publish]
  Msg2:      [Parse]──►[Validate]──►[DB]──►[Publish]
  Msg3:           [Parse]──►[Validate]──►[DB]──►[Publish]
  Throughput ≈ 1000/50 = 20 msg/s (bottleneck = DB 50ms)
```

### Async Handoff Pattern

```javascript
// Producer thread: parse + enqueue
// Worker pool: DB writes (bounded queue = backpressure)
const queue = new BoundedQueue(1000);

async function consumeLoop() {
  for await (const msg of consumer) {
    await queue.put(msg);  // Block nếu queue đầy → backpressure
  }
}

// N workers
for (let i = 0; i < 8; i++) {
  processWorker(queue);
}
```

---

## Serialization Impact

| Format | Size (typical) | Serialize Speed | Schema Evolution |
| ------ | -------------- | --------------- | ---------------- |
| **JSON** | Lớn (verbose) | Chậm | Linh hoạt |
| **Avro** | Nhỏ | Nhanh | Schema Registry |
| **Protobuf** | Nhỏ nhất | Rất nhanh | `.proto` files |
| **MessagePack** | Trung bình | Nhanh | Ít dùng trong MQ |

```
JSON order event:     ~800 bytes
Avro order event:     ~200 bytes  → 4x ít network I/O
Protobuf order event: ~150 bytes
```

> Xem [03-apache-kafka/6-schema-registry.md](../03-apache-kafka/6-schema-registry.md) cho schema evolution.

---

## Benchmark Checklist

```
□ Baseline: ghi messages/sec, MB/sec, P99 latency
□ Thay đổi MỘT biến mỗi lần (batch.size, linger, compression...)
□ Chạy đủ lâu (≥5 phút) để warm-up JVM/GC ổn định
□ Monitor broker CPU, disk I/O, network — không chỉ client
□ Test với message size thực tế (không chỉ 1 KB synthetic)
□ Test với replication factor production
□ Document kết quả và config cuối cùng
```

---

## Anti-Patterns

| Anti-Pattern | Vấn Đề | Fix |
| ------------ | ------ | --- |
| `linger.ms=0` + high volume | Quá nhiều small requests | `linger.ms=10-20` |
| JSON + no compression | Network bottleneck | Avro + lz4 |
| `prefetch=1000` + slow handler | Message "stuck" trên 1 consumer | Giảm prefetch hoặc tăng workers |
| Sync DB write per message | Handler bottleneck | Batch insert, async write |
| Tune không đo | Không biết cải thiện bao nhiêu | Benchmark trước/sau |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `linger.ms` và `batch.size` hoạt động cùng nhau thế nào?

**Trả lời:** Producer gom records vào batch cho đến khi **đạt `batch.size`** HOẶC **hết `linger.ms`** — điều kiện nào đến trước thì gửi. `linger.ms=0` gửi ngay (low latency). `linger.ms=20` chờ tối đa 20ms để gom thêm records → batch lớn hơn → throughput cao hơn.

### Câu 2: Tại sao lz4 được khuyến nghị hơn gzip?

**Trả lời:** **lz4** nén/decompress **rất nhanh** với CPU thấp — phù hợp real-time. **gzip** ratio tốt hơn nhưng CPU cao, có thể trở thành bottleneck. Trong messaging, throughput thường quan trọng hơn compression ratio tuyệt đối.

### Câu 3: `acks=1` có an toàn cho production không?

**Trả lời:** Phụ thuộc use case. `acks=1` chấp nhận mất message nếu leader die trước khi replicate. Cho **critical data** (orders, payments) dùng `acks=all` + `min.insync.replicas=2`. Cho **logs, metrics** `acks=1` hoặc `acks=0` acceptable.

---

**Cập Nhật:** 2026-07-03
