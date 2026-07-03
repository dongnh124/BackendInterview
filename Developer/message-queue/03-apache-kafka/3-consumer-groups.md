# Consumer Groups — Nhóm Consumer Và Rebalancing

> Hiểu Consumer Group (Nhóm Consumer), partition assignment (gán phân vùng), rebalancing (cân bằng lại), cooperative sticky assignor, static membership, và cách scale consumer an toàn trong production.

## Mục Lục

1. [Consumer Group Là Gì?](#consumer-group-là-gì)
2. [Partition Assignment](#partition-assignment)
3. [Rebalancing](#rebalancing)
4. [Rebalance Protocols](#rebalance-protocols)
5. [Static Membership](#static-membership)
6. [Scale Consumer](#scale-consumer)
7. [Multiple Consumer Groups](#multiple-consumer-groups)
8. [Consumer Config Quan Trọng](#consumer-config-quan-trọng)
9. [Troubleshooting](#troubleshooting)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Consumer Group Là Gì?

**Consumer Group** là tập hợp consumers **cùng `group.id`** — chia sẻ workload của topic partitions.

```
Topic "orders" (3 partitions)
Consumer Group "inventory-service"

┌─────────────┐     P0     ┌─────────────┐
│ Consumer 1  │◄──────────►│             │
└─────────────┘            │   Kafka     │
┌─────────────┐     P1     │   Broker    │
│ Consumer 2  │◄──────────►│             │
└─────────────┘            │             │
┌─────────────┐     P2     │             │
│ Consumer 3  │◄──────────►│             │
└─────────────┘            └─────────────┘

Mỗi partition → tối đa 1 consumer trong group
```

| Quy Tắc | Mô Tả |
| ------- | ----- |
| **1 partition : 1 consumer** | Trong cùng group, partition không chia cho 2 consumer |
| **1 consumer : N partitions** | Consumer có thể đọc nhiều partitions |
| **Nhiều groups** | Mỗi group đọc toàn bộ topic độc lập |

```properties
# Consumer config
group.id=inventory-service
```

---

## Partition Assignment

### Group Coordinator

Một broker đóng vai **Group Coordinator (Điều Phối Nhóm)** cho mỗi consumer group:

1. Nhận **JoinGroup** request từ consumers
2. Chọn **Group Leader** (một consumer trong group)
3. Leader chạy **Partition Assignor** — quyết định ai đọc partition nào
4. Coordinator broadcast assignment

```
Consumer A ──JoinGroup──► Coordinator
Consumer B ──JoinGroup──► Coordinator
Consumer C ──JoinGroup──► Coordinator
                │
                ▼
         Elect Group Leader (ví dụ: B)
                │
                ▼
         Leader runs assignor → {A: P0, B: P1, C: P2}
                │
                ▼
         SyncGroup → all consumers start fetching
```

### Assignor Strategies

| Assignor | Hành Vi |
| -------- | ------- |
| **RangeAssignor** | Chia theo range — có thể skew nếu topics nhiều |
| **RoundRobinAssignor** | Round-robin partitions across consumers |
| **StickyAssignor** | Giữ assignment cũ tối đa khi rebalance |
| **CooperativeStickyAssignor** | Incremental rebalance — không revoke tất cả |

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

> **Production khuyến nghị:** `CooperativeStickyAssignor` — giảm stop-the-world rebalance.

---

## Rebalancing

### Khi Nào Rebalance Xảy Ra?

| Trigger | Ví Dụ |
| ------- | ----- |
| Consumer join | Deploy instance mới, scale up |
| Consumer leave | Crash, graceful shutdown, `session.timeout` |
| Topic partition thay đổi | Tăng partition count |
| Subscription thay đổi | Subscribe topic mới |

### Rebalance Impact

```
Rebalance lifecycle:
1. All consumers STOP processing (eager protocol)
2. Revoke all partitions
3. Re-assign partitions
4. Consumers seek to committed offset
5. Resume processing

→ Processing pause — "rebalance storm" nếu deploy liên tục
```

### Rebalance Storm (Bão Cân Bằng Lại)

```
Nguyên nhân:
- Rolling deploy → mỗi pod restart trigger rebalance
- session.timeout.ms quá ngắn → GC pause bị coi là dead
- max.poll.interval.ms vượt quá → consumer bị kick

Giải pháp:
- CooperativeStickyAssignor
- Static membership (group.instance.id)
- Tăng session.timeout / max.poll.interval hợp lý
- Scale during low traffic
```

---

## Rebalance Protocols

### Eager (Classic — Cổ Điển)

Tất cả partitions **revoke** trước khi assign lại — downtime trong rebalance.

### Cooperative (Incremental — Tăng Dần)

Chỉ revoke partitions **cần chuyển** — consumer tiếp tục xử lý partitions không đổi.

```
Eager:     [P0,P1,P2] → revoke ALL → reassign
Cooperative: [P0,P1] giữ nguyên, chỉ chuyển P2 từ C1 → C2
```

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

---

## Static Membership

**Static Membership (Thành Viên Tĩnh)** — consumer có `group.instance.id` cố định:

```properties
group.id=payment-service
group.instance.id=payment-pod-1
```

Khi consumer restart nhanh (trong `session.timeout`), coordinator **giữ assignment** — không rebalance toàn group.

```
Không static:
  Pod-1 die → rebalance toàn group → Pod-2, Pod-3 pause

Có static:
  Pod-1 die → slot reserved → Pod-1 rejoin → resume same partitions
```

> **Kubernetes:** Dùng StatefulSet name hoặc stable pod identity làm `group.instance.id`.

---

## Scale Consumer

### Scale Up

```
Trước: 3 partitions, 1 consumer (đọc P0,P1,P2) — lag cao
Sau:  3 partitions, 3 consumers (mỗi người 1 partition) — 3x throughput
```

**Giới hạn:** Số consumer active ≤ số partitions.

### Scale Beyond Partitions

```
6 consumers, 3 partitions → 3 consumers idle
→ Cần tăng partitions TRƯỚC khi scale consumer thêm
```

### Quy Trình Scale An Toàn

```
1. Monitor consumer lag per partition
2. Nếu processing chậm → optimize code trước
3. Nếu cần scale → đảm bảo partitions đủ
4. Thêm consumer từ từ — tránh rebalance storm
5. Verify lag giảm, không có duplicate spike
```

---

## Multiple Consumer Groups

Mỗi consumer group là **độc lập** — đọc cùng topic mà không ảnh hưởng nhau:

```
Topic: order.created

Group "inventory"  → trừ kho
Group "notification" → gửi email
Group "analytics"  → warehouse ETL

Mỗi group có offset riêng trên __consumer_offsets
```

Đây là điểm khác biệt lớn với **competing consumers** trong RabbitMQ queue.

---

## Consumer Config Quan Trọng

| Config | Mô Tả | Gợi Ý |
| ------ | ----- | ----- |
| `group.id` | Tên consumer group | Unique per application |
| `enable.auto.commit` | Tự commit offset | `false` cho critical processing |
| `auto.offset.reset` | Hành vi khi không có offset | `earliest` / `latest` |
| `session.timeout.ms` | Heartbeat timeout | 45000 (default) — tăng nếu GC dài |
| `heartbeat.interval.ms` | Tần suất heartbeat | ~1/3 session.timeout |
| `max.poll.interval.ms` | Max thời gian giữa poll() | Tăng nếu xử lý message lâu |
| `max.poll.records` | Records mỗi poll | Giảm nếu xử lý chậm |
| `fetch.min.bytes` | Batch fetch size | Tăng cho throughput |

```java
Properties props = new Properties();
props.put("group.id", "order-processor");
props.put("enable.auto.commit", "false");
props.put("max.poll.records", "100");
props.put("session.timeout.ms", "30000");
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

### Poll Loop

```java
while (true) {
  ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(1000));
  for (ConsumerRecord<String, Order> record : records) {
    process(record); // phải hoàn thành trước max.poll.interval.ms
  }
  consumer.commitSync(); // manual commit sau batch
}
```

---

## Troubleshooting

| Triệu Chứng | Nguyên Nhân | Fix |
| ----------- | ----------- | --- |
| Lag tăng liên tục | Consumer chậm / ít instance | Scale, optimize, tăng partitions |
| Rebalance liên tục | Timeout, deploy rolling | Static membership, cooperative assignor |
| Duplicate processing | Commit sau crash | Idempotent consumer |
| Consumer không nhận message | Sai group.id / chưa subscribe | Verify config |
| Một partition lag cao hơn | Hot partition | Redesign partition key |

### Useful Commands

```bash
# Mô tả consumer group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-service

# Output: PARTITION, CURRENT-OFFSET, LOG-END-OFFSET, LAG, CONSUMER-ID, HOST
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Consumer group hoạt động thế nào?

**Gợi ý trả lời:** Consumers cùng `group.id` chia partitions — mỗi partition chỉ 1 consumer trong group. Coordinator quản lý membership và assignment. Khi member thay đổi → rebalance.

### Câu 2: Rebalance là gì? Tại sao nguy hiểm?

**Gợi ý trả lời:** **Rebalance** phân phối lại partitions khi group thay đổi. Trong eager mode, **tất cả consumer pause** — lag spike, duplicate nếu commit chưa kịp. Giảm bằng cooperative assignor và static membership.

### Câu 3: 12 partitions, 4 consumers — phân bổ thế nào?

**Gợi ý trả lời:** Mỗi consumer ~3 partitions (tùy assignor). Tất cả 4 consumer active. Thêm consumer thứ 5 → rebalance, mỗi người ~2–3 partitions.

### Câu 4: Hai microservices cùng đọc topic — cùng group hay khác group?

**Gợi ý trả lời:** **Khác group** — mỗi service cần đọc toàn bộ events. Cùng group → messages chia đôi, mỗi service chỉ nhận subset.

### Câu 5: max.poll.interval.ms vs session.timeout.ms?

**Gợi ý trả lời:** **session.timeout.ms** — consumer phải gửi heartbeat, không thì coi là dead. **max.poll.interval.ms** — thời gian tối đa giữa hai lần `poll()` — nếu xử lý message quá lâu không poll → bị kick khỏi group dù heartbeat vẫn chạy.

---

**Xem tiếp:** [4-producers-serialization.md](./4-producers-serialization.md) — producer configuration và serialization.
