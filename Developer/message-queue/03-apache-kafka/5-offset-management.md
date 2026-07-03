# Offset Management — Quản Lý Offset

> Quản lý Consumer Offset (Vị Trí Đọc Consumer): auto commit vs manual commit (tự động vs thủ công), `__consumer_offsets` topic, offset reset (đặt lại offset), seek (nhảy offset), và commit strategy (chiến lược commit) an toàn trong production.

## Mục Lục

1. [Offset Là Gì?](#offset-là-gì)
2. [__consumer_offsets Topic](#__consumer_offsets-topic)
3. [Auto Commit vs Manual Commit](#auto-commit-vs-manual-commit)
4. [Commit Strategies](#commit-strategies)
5. [Offset Reset](#offset-reset)
6. [Seek & Replay](#seek--replay)
7. [Offset và Delivery Semantics](#offset-và-delivery-semantics)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Offset Là Gì?

**Offset** là **vị trí tuần tự (sequential position)** của consumer trong một partition — pointer đến message tiếp theo sẽ đọc.

```
Partition 0:
Offset:  0      1      2      3      4      5
        [msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
                              ▲
                         committed offset = 3
                         → next read: offset 3 (msg3)
```

| Khái Niệm | Mô Tả |
| -------- | ----- |
| **Current position** | Offset consumer đang đọc / vừa đọc |
| **Committed offset** | Offset đã lưu — resume point khi restart |
| **Log end offset** | Offset mới nhất + 1 trong partition |
| **Lag** | log_end_offset − committed_offset |

---

## __consumer_offsets Topic

Kafka lưu committed offsets trong **internal compacted topic** `__consumer_offsets`:

```
Key:   (group.id, topic, partition)
Value: offset + metadata + commit timestamp
```

```
Consumer commit offset
        │
        ▼
Coordinator broker
        │
        ▼
Write to __consumer_offsets (compacted)
        │
        ▼
Restart → read committed offset → resume
```

> **Lưu ý:** Không chỉnh sửa topic này thủ công. Dùng `kafka-consumer-groups.sh` để inspect/reset.

---

## Auto Commit vs Manual Commit

### Auto Commit (Mặc Định)

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

```
Timeline auto-commit:
  poll() → nhận messages
  xử lý msg1, msg2
  [crash trước khi commit interval]
  → restart → đọc lại từ offset cũ → at-least-once (OK)
  → HOẶC commit đã chạy trước khi xử lý xong → at-most-once (MẤT)
```

**Vấn đề:** Commit theo **thời gian**, không theo **xử lý xong** — có thể commit offset trước khi process success → **message loss**.

### Manual Commit (Khuyến Nghị Production)

```properties
enable.auto.commit=false
```

```java
ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(1000));
for (ConsumerRecord<String, Order> record : records) {
  processOrder(record);           // xử lý trước
}
consumer.commitSync();            // commit sau khi batch xong
```

| Mode | Ưu | Nhược |
| ---- | -- | ----- |
| **Auto** | Đơn giản | Message loss risk |
| **Manual sync** | Rõ ràng, durable commit | Chậm hơn nếu commit mỗi message |
| **Manual async** | Nhanh hơn | Cần handle callback errors |

---

## Commit Strategies

### Commit Per Batch (Sau Mỗi Poll)

```java
while (true) {
  var records = consumer.poll(Duration.ofMillis(1000));
  for (var record : records) {
    processWithIdempotency(record);
  }
  consumer.commitSync(); // commit offset cuối batch
}
```

**Trade-off:** Crash giữa batch → reprocess cả batch → cần **idempotent consumer**.

### Commit Per Message

```java
for (var record : records) {
  process(record);
  consumer.commitSync(Map.of(
    new TopicPartition(record.topic(), record.partition()),
    new OffsetAndMetadata(record.offset() + 1)
  ));
}
```

**Trade-off:** Ít duplicate hơn nhưng **commit overhead** cao — latency tăng.

### Async Commit

```java
consumer.commitAsync((offsets, exception) -> {
  if (exception != null) {
    log.error("Commit failed", exception);
    // retry hoặc alert
  }
});
```

> **Lưu ý:** Trước rebalance/shutdown, gọi `commitSync()` để đảm bảo commit hoàn tất.

### commitSync vs commitAsync

| | commitSync | commitAsync |
| --- | ---------- | ----------- |
| Blocking | Có | Không |
| Guaranteed | Có trước khi return | Callback — có thể fail silent |
| Use case | Shutdown, sau critical batch | High throughput |

---

## Offset Reset

Khi **không có committed offset** (group mới hoặc offset expired):

```properties
auto.offset.reset=earliest   # đọc từ đầu partition
auto.offset.reset=latest     # chỉ đọc message mới (default behavior khi có offset)
auto.offset.reset=none       # throw exception — an toàn production
```

| Giá Trị | Hành Vi | Khi Dùng |
| ------- | ------- | -------- |
| `earliest` | Đọc từ offset 0 | Reprocess toàn bộ, new consumer group |
| `latest` | Bỏ qua message cũ | Chỉ care real-time events |
| `none` | Fail nếu không có offset | Production strict — tránh surprise replay |

### Reset Offset Thủ Công

```bash
# Shift offset về 1 giờ trước (by timestamp)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-datetime 2026-07-03T08:00:00.000 \
  --execute

# Reset về earliest
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --reset-offsets --to-earliest --all-topics --execute
```

> **Cảnh báo:** Reset offset → **reprocess** — đảm bảo idempotency và downstream chịu được load.

---

## Seek & Replay

### seek() — Nhảy Offset Trong Code

```java
TopicPartition tp = new TopicPartition("orders", 0);
consumer.seek(tp, 1000);        // nhảy đến offset 1000
consumer.seekToBeginning(List.of(tp));
consumer.seekToEnd(List.of(tp));
```

### Replay Use Cases

| Use Case | Cách Làm |
| -------- | -------- |
| **Bug fix reprocess** | Reset offset hoặc tạo consumer group mới + earliest |
| **New downstream** | Consumer group mới tự đọc từ earliest |
| **Point-in-time** | `--reset-offsets --to-datetime` |
| **Testing** | seekToBeginning trong integration test |

```
Replay flow:
1. Stop consumer group (hoặc scale to 0)
2. Reset offsets to target position
3. Verify idempotent handlers ready
4. Start consumers — monitor lag & error rate
```

---

## Offset và Delivery Semantics

```
                    COMMIT TIMING
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   Commit TRƯỚC    Commit SAU       Exactly-once
   khi process     khi process      (transactional)
         │               │               │
         ▼               ▼               ▼
   At-most-once    At-least-once    EOS (phức tạp)
   (có thể mất)    (có thể dup)     (Kafka txn)
```

| Pattern | Commit | Semantics |
| ------- | ------ | --------- |
| Auto-commit default | Trước/song song process | Không đảm bảo |
| Process → commitSync | Sau process | At-least-once |
| Transactional consume-produce | Atomic offset + produce | Exactly-once (EOS) |

> Xem thêm: [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md)

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| `enable.auto.commit=false` | Commit sau khi xử lý xong |
| Idempotent consumer | At-least-once là mặc định |
| `auto.offset.reset=none` production | Tránh accidental full replay |
| Commit trước rebalance listener | Tránh duplicate khi partition revoke |
| Monitor lag per partition | Phát hiện consumer chậm sớm |
| Document reset procedures | Runbook cho replay incidents |

### ConsumerRebalanceListener

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
  @Override
  public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    consumer.commitSync(); // commit trước khi mất partition
  }
  @Override
  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
    // optional: seek logic
  }
});
```

### Offset Expiration

```properties
offsets.retention.minutes=10080  # 7 ngày — offset bị xóa nếu group inactive
```

Consumer group không active quá lâu → committed offset expire → `auto.offset.reset` áp dụng khi rejoin.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Auto-commit có an toàn không?

**Gợi ý trả lời:** **Không** cho critical processing — commit theo interval, có thể commit trước khi xử lý xong → message loss. Production dùng **manual commit sau process**.

### Câu 2: Consumer lag là gì?

**Gợi ý trả lời:** **Lag** = `log_end_offset − current_offset` — số message consumer chưa xử lý kịp. Lag tăng = consumer chậm hơn producer hoặc processing bottleneck.

### Câu 3: Làm sao replay message từ 1 tuần trước?

**Gợi ý trả lời:** Nếu trong retention: tạo consumer group mới với `auto.offset.reset=earliest` hoặc `kafka-consumer-groups --reset-offsets --to-datetime`. Đảm bảo idempotent downstream.

### Câu 4: Commit sync sau mỗi poll — duplicate khi nào?

**Gợi ý trả lời:** Crash **sau** process nhưng **trước** commit → restart đọc lại batch → duplicate. Đây là **at-least-once** — cần idempotent handler.

### Câu 5: __consumer_offsets lưu gì?

**Gợi ý trả lời:** Internal compacted topic lưu **committed offset** per (group, topic, partition). Coordinator ghi khi consumer commit. Cho phép consumer resume sau restart.

---

**Xem tiếp:** [6-schema-registry.md](./6-schema-registry.md) — Schema Registry và schema evolution.
