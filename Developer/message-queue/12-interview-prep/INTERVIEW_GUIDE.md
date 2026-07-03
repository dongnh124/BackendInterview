# Top 30 Câu Hỏi Phỏng Vấn Message Queue — Đáp Án Chi Tiết

> Bộ 30 câu hỏi phỏng vấn Message Queue & Event Broker được hỏi nhiều nhất, kèm đáp án chi tiết và liên kết tài liệu sâu hơn.

## Mục Lục

- [Phần 1: Fundamentals (Câu 1–8)](#phần-1-fundamentals-câu-18)
- [Phần 2: Architecture Patterns (Câu 9–14)](#phần-2-architecture-patterns-câu-914)
- [Phần 3: Apache Kafka (Câu 15–20)](#phần-3-apache-kafka-câu-1520)
- [Phần 4: RabbitMQ (Câu 21–24)](#phần-4-rabbitmq-câu-2124)
- [Phần 5: Reliability & Operations (Câu 25–30)](#phần-5-reliability--operations-câu-2530)

---

## Danh Sách Nhanh

| # | Câu Hỏi | Chủ Đề | Mức Độ |
|---|---------|--------|--------|
| 1 | Message Queue vs Event Broker | Fundamentals | ⭐ Cơ bản |
| 2 | Point-to-Point vs Pub/Sub | Fundamentals | ⭐ Cơ bản |
| 3 | At-most-once, At-least-once, Exactly-once | Fundamentals | ⭐⭐⭐ Rất quan trọng |
| 4 | Khi nào dùng sync API vs async messaging? | Fundamentals | ⭐⭐ Trung cấp |
| 5 | Idempotency là gì? Tại sao quan trọng? | Fundamentals | ⭐⭐⭐ Rất quan trọng |
| 6 | Message ordering — đảm bảo thứ tự thế nào? | Fundamentals | ⭐⭐ Trung cấp |
| 7 | Backpressure là gì? | Fundamentals | ⭐⭐ Trung cấp |
| 8 | Kafka vs RabbitMQ — khi nào chọn cái nào? | Fundamentals | ⭐⭐⭐ Rất quan trọng |
| 9 | Outbox Pattern giải quyết vấn đề gì? | Architecture | ⭐⭐⭐ Rất quan trọng |
| 10 | Saga Choreography vs Orchestration | Architecture | ⭐⭐⭐ Nâng cao |
| 11 | Event Sourcing vs CRUD | Architecture | ⭐⭐⭐ Nâng cao |
| 12 | CQRS là gì? Khi nào dùng? | Architecture | ⭐⭐ Trung cấp |
| 13 | Inbox Pattern — khi nào cần? | Architecture | ⭐⭐ Trung cấp |
| 14 | Dual-write problem | Architecture | ⭐⭐⭐ Rất quan trọng |
| 15 | Topic, Partition, Consumer Group | Kafka | ⭐⭐ Trung cấp |
| 16 | acks=0 vs acks=1 vs acks=all | Kafka | ⭐⭐ Trung cấp |
| 17 | Consumer lag — diagnose và fix | Kafka | ⭐⭐⭐ Rất quan trọng |
| 18 | Exactly-once trong Kafka | Kafka | ⭐⭐⭐ Nâng cao |
| 19 | Partition key design | Kafka | ⭐⭐ Trung cấp |
| 20 | Rebalancing — vấn đề gì xảy ra? | Kafka | ⭐⭐ Trung cấp |
| 21 | Các loại Exchange trong RabbitMQ | RabbitMQ | ⭐⭐ Trung cấp |
| 22 | DLQ thiết kế như thế nào? | RabbitMQ | ⭐⭐⭐ Rất quan trọng |
| 23 | Publisher Confirm vs Consumer Ack | RabbitMQ | ⭐⭐ Trung cấp |
| 24 | Quorum Queue vs Classic Mirrored Queue | RabbitMQ | ⭐⭐ Trung cấp |
| 25 | Poison message — xử lý thế nào? | Reliability | ⭐⭐⭐ Rất quan trọng |
| 26 | Retry strategy — exponential backoff | Reliability | ⭐⭐ Trung cấp |
| 27 | Circuit Breaker cho consumers | Reliability | ⭐⭐ Trung cấp |
| 28 | Consumer lag spike — root cause | Operations | ⭐⭐⭐ Rất quan trọng |
| 29 | Message loss — nguyên nhân và phòng ngừa | Operations | ⭐⭐⭐ Rất quan trọng |
| 30 | Monitoring messaging layer — metrics nào? | Operations | ⭐⭐ Trung cấp |

---

## Phần 1: Fundamentals (Câu 1–8)

### Câu 1: Message Queue vs Event Broker — khác nhau thế nào?

**Đáp án ngắn:** Message Queue (Hàng Đợi Tin Nhắn) tập trung **task distribution (phân phối tác vụ)** — một consumer xử lý mỗi message. Event Broker (Broker Sự Kiện) tập trung **event streaming (luồng sự kiện)** — nhiều consumer độc lập đọc cùng event stream.

| | Message Queue | Event Broker |
| - | ------------- | ------------ |
| **Mô hình** | Point-to-Point (Điểm-Điểm) | Pub/Sub (Publish/Subscribe) |
| **Consumer** | Competing consumers (cạnh tranh) | Mỗi subscriber nhận bản sao |
| **Retention** | Xóa sau khi ack | Giữ log (replay được) |
| **Ví dụ** | RabbitMQ work queue, SQS | Kafka, Event Hubs |

📖 Xem thêm: [01-fundamentals/1-message-queue-basics.md](../01-fundamentals/1-message-queue-basics.md)

---

### Câu 2: Point-to-Point vs Pub/Sub — khi nào dùng?

**Point-to-Point (P2P):** Mỗi message chỉ một consumer xử lý — phù hợp **task queue**, **job processing**, **load balancing (cân bằng tải)** giữa workers.

**Pub/Sub (Publish/Subscribe — Xuất Bản/Đăng Ký):** Mỗi subscriber nhận message — phù hợp **event notification**, **fan-out (phân phối đa hướng)**, **decoupling (tách rời) services**.

```
P2P:     Producer → Queue → [Worker A HOẶC Worker B]
Pub/Sub: Producer → Topic → [Service A, Service B, Service C]
```

📖 Xem thêm: [01-fundamentals/2-pub-sub-vs-point-to-point.md](../01-fundamentals/2-pub-sub-vs-point-to-point.md)

---

### Câu 3: Giải thích At-most-once, At-least-once, Exactly-once

**At-most-once (Tối Đa Một Lần):** Message deliver 0 hoặc 1 lần — có thể **mất**, không trùng. Dùng cho metrics, telemetry không critical.

**At-least-once (Ít Nhất Một Lần):** Message deliver ≥1 lần — **không mất** (lý tưởng), có thể **duplicate**. Production default — consumer phải **idempotent (bất biến khi lặp lại)**.

**Exactly-once (Đúng Một Lần):** Deliver đúng 1 lần — không mất, không trùng. Khó đạt end-to-end; thường dùng **effective exactly-once** = at-least-once + idempotent consumer + transactional producer.

```
Production thực tế:
  Broker: at-least-once
  + Idempotent consumer
  + Dedup key (messageId / eventId)
  = Effective exactly-once
```

📖 Xem thêm: [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md)

---

### Câu 4: Khi nào dùng sync API vs async messaging?

**Dùng sync API (Request-Response):**
- Cần response ngay (query, validation)
- Luồng đơn giản, ít services
- Strong consistency (nhất quán mạnh) trong một transaction

**Dùng async messaging:**
- Decouple producer và consumer về thời gian
- Spike traffic (đỉnh tải) — buffer qua queue
- Fan-out nhiều downstream services
- Long-running tasks (email, report generation)
- Event-driven microservices

**Trade-off:** Async tăng complexity (ordering, idempotency, monitoring) nhưng tăng resilience (khả năng phục hồi) và scalability (khả năng mở rộng).

---

### Câu 5: Idempotency là gì? Tại sao quan trọng?

**Idempotency (Tính Bất Biến Khi Lặp Lại):** Gọi cùng operation nhiều lần cho cùng kết quả như gọi một lần.

**Tại sao quan trọng:** At-least-once delivery → message có thể duplicate → consumer xử lý 2 lần không được gây side effect kép (charge 2 lần, gửi 2 email).

**Cách implement:**
1. **Natural idempotency:** `SET status = 'shipped'` — idempotent by nature
2. **Idempotency key:** Lưu `messageId`/`eventId` vào DB, skip nếu đã xử lý
3. **Upsert:** `INSERT ... ON CONFLICT DO NOTHING`

📖 Xem thêm: [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md)

---

### Câu 6: Message ordering — đảm bảo thứ tự thế nào?

**Nguyên tắc:** Ordering (thứ tự) chỉ đảm bảo **trong phạm vi hẹp**:

| Broker | Ordering Scope |
| ------ | -------------- |
| Kafka | Trong **một partition** (cùng partition key) |
| RabbitMQ | Trong **một queue** (single consumer) |
| SQS FIFO | Trong **message group** |

**Chiến lược:**
- Dùng **partition key** = `orderId` để events cùng order vào cùng partition
- Trade-off: hot partition nếu key phân bố lệch
- Không cần global ordering → tăng partition count

📖 Xem thêm: [01-fundamentals/4-ordering-and-sequencing.md](../01-fundamentals/4-ordering-and-sequencing.md)

---

### Câu 7: Backpressure là gì?

**Backpressure (Áp Lực Ngược):** Cơ chế **làm chậm producer** khi consumer không theo kịp, tránh broker/consumer bị quá tải.

**Triệu chứng:** Queue depth (độ sâu hàng đợi) tăng, consumer lag tăng, memory pressure trên broker.

**Giải pháp:**
- Rate limiting (giới hạn tốc độ) trên producer
- Scale consumer (tăng partition/consumer count)
- `max.poll.records` giảm batch size
- Pause consumption khi downstream overload

📖 Xem thêm: [01-fundamentals/5-backpressure-flow-control.md](../01-fundamentals/5-backpressure-flow-control.md)

---

### Câu 8: Kafka vs RabbitMQ — khi nào chọn cái nào?

| Tiêu Chí | Kafka | RabbitMQ |
| -------- | ----- | -------- |
| **Throughput** | Rất cao (hàng triệu msg/s) | Trung bình–cao |
| **Retention** | Event log, replay | Xóa sau consume |
| **Routing** | Topic + partition key | Exchange linh hoạt (direct, topic, fanout) |
| **Use case** | Event streaming, CDC, analytics | Task queue, RPC, workflow |
| **Complexity** | Cao hơn (ops, tuning) | Dễ triển khai hơn |

**Chọn Kafka khi:** Cần replay, high throughput, nhiều consumer groups đọc cùng stream, stream processing.

**Chọn RabbitMQ khi:** Task distribution, routing phức tạp, RPC pattern, team nhỏ cần setup nhanh.

📖 Xem thêm: [01-fundamentals/6-broker-selection-guide.md](../01-fundamentals/6-broker-selection-guide.md)

---

## Phần 2: Architecture Patterns (Câu 9–14)

### Câu 9: Outbox Pattern giải quyết vấn đề gì?

**Vấn đề — Dual-write (Ghi Kép):** Ghi DB và publish message là 2 operations độc lập — một thành công, một fail → inconsistency (không nhất quán).

**Outbox Pattern (Mẫu Hộp Thoại Ra):**
1. Ghi business data + event vào bảng `outbox` trong **cùng DB transaction**
2. Background poller/CDC đọc outbox và publish lên broker
3. Mark outbox row là `published`

```
BEGIN TRANSACTION
  INSERT INTO orders ...
  INSERT INTO outbox (event_type, payload) ...
COMMIT
→ Outbox Relay → Kafka/RabbitMQ
```

📖 Xem thêm: [02-architecture-patterns/4-outbox-inbox-pattern.md](../02-architecture-patterns/4-outbox-inbox-pattern.md)

---

### Câu 10: Saga Choreography vs Orchestration

**Saga Pattern (Mẫu Saga):** Quản lý **distributed transaction (giao dịch phân tán)** qua chuỗi local transactions + compensation (bù trừ) khi fail.

| | Choreography (Điệu Múa) | Orchestration (Điều Phối) |
| - | ----------------------- | ------------------------- |
| **Điều phối** | Services tự publish/subscribe events | Central orchestrator điều khiển |
| **Coupling** | Loose (lỏng) | Tighter với orchestrator |
| **Visibility** | Khó trace flow | Dễ theo dõi state |
| **Phù hợp** | Ít steps, team autonomous | Nhiều steps, logic phức tạp |

**Compensation ví dụ:** `OrderCreated` → `PaymentFailed` → publish `OrderCancelled` + refund.

📖 Xem thêm: [02-architecture-patterns/3-saga-pattern.md](../02-architecture-patterns/3-saga-pattern.md)

---

### Câu 11: Event Sourcing vs CRUD — trade-offs?

**Event Sourcing (Lưu Trữ Sự Kiện):** Lưu **chuỗi events** thay vì state hiện tại. State = replay events.

**CRUD:** Lưu state hiện tại, update trực tiếp.

| | Event Sourcing | CRUD |
| - | -------------- | ---- |
| **Audit trail** | Tự nhiên (full history) | Cần thêm audit log |
| **Replay** | Có | Không |
| **Complexity** | Cao (projections, schema evolution) | Thấp |
| **Query** | Cần read model / CQRS | Trực tiếp |

📖 Xem thêm: [02-architecture-patterns/2-cqrs-event-sourcing.md](../02-architecture-patterns/2-cqrs-event-sourcing.md)

---

### Câu 12: CQRS là gì? Khi nào dùng?

**CQRS (Command Query Responsibility Segregation — Tách Trách Nhiệm Đọc/Ghi):** Tách **write model** (commands → events) và **read model** (queries → optimized views).

**Khi nào dùng:**
- Read/write pattern khác nhau (write ít, read nhiều)
- Kết hợp Event Sourcing
- Cần scale read và write độc lập

**Không cần khi:** CRUD đơn giản, team nhỏ, over-engineering risk cao.

---

### Câu 13: Inbox Pattern — khi nào cần?

**Inbox Pattern (Mẫu Hộp Thoại Vào):** Lưu incoming message vào bảng `inbox` trước khi xử lý — đảm bảo **exactly-once processing** phía consumer khi kết hợp với DB transaction.

**Flow:**
1. Nhận message → INSERT inbox (dedup by messageId)
2. Xử lý business logic trong transaction
3. Mark inbox processed

**Khi cần:** Consumer ghi DB + cần tránh duplicate processing từ at-least-once delivery.

---

### Câu 14: Dual-write problem — giải pháp nào?

**Vấn đề:** `saveToDB()` + `publishEvent()` — không atomic (nguyên tử).

**Giải pháp (theo độ phổ biến):**

1. **Outbox Pattern** — recommended, transactional outbox + relay
2. **CDC (Change Data Capture)** — Debezium đọc WAL (Write-Ahead Log) → Kafka
3. **Transactional messaging** — Kafka transactions (phức tạp, giới hạn)
4. **Saga + compensation** — chấp nhận eventual consistency

**Tránh:** Publish trước rồi save DB (message orphan) hoặc save DB rồi publish không transaction (message loss).

---

## Phần 3: Apache Kafka (Câu 15–20)

### Câu 15: Topic, Partition, Consumer Group hoạt động thế nào?

**Topic:** Category/logical channel cho messages.

**Partition (Phân Vùng):** Topic chia thành partitions — đơn vị parallelism (song song hóa) và ordering scope.

**Consumer Group (Nhóm Consumer):** Nhóm consumers chia partition — mỗi partition chỉ 1 consumer trong group tại một thời điểm.

```
Topic "orders" (3 partitions)
Consumer Group "fulfillment":
  Consumer A → Partition 0
  Consumer B → Partition 1
  Consumer C → Partition 2

→ Max parallelism = số partitions
```

📖 Xem thêm: [03-apache-kafka/2-topics-partitions.md](../03-apache-kafka/2-topics-partitions.md), [3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md)

---

### Câu 16: acks=0 vs acks=1 vs acks=all

| acks | Ý Nghĩa | Durability | Latency |
| ---- | ------- | ---------- | ------- |
| **0** | Fire-and-forget | Thấp nhất | Thấp nhất |
| **1** | Leader ack | Trung bình | Trung bình |
| **all** | Tất cả ISR ack | Cao nhất | Cao nhất |

**ISR (In-Sync Replicas — Bản Sao Đồng Bộ):** Replicas đã catch-up với leader.

**Production:** `acks=all` + `min.insync.replicas=2` cho data quan trọng.

📖 Xem thêm: [03-apache-kafka/4-producers-serialization.md](../03-apache-kafka/4-producers-serialization.md)

---

### Câu 17: Consumer lag — diagnose và fix

**Consumer Lag (Độ Trễ Consumer):** Khoảng cách giữa offset mới nhất trên partition và offset consumer đã commit.

**Diagnose:**
1. Check lag per partition — hot partition?
2. Consumer processing time — slow handler?
3. Rebalancing liên tục? — `max.poll.interval.ms` exceeded
4. Downstream dependency slow? — DB, external API

**Fix:**
- Scale consumers (≤ partition count)
- Optimize handler (batch DB writes)
- Tăng partition count (cần re-key strategy)
- Fix poison message blocking partition

📖 Xem thêm: [09-monitoring/2-lag-monitoring.md](../09-monitoring/2-lag-monitoring.md), [2-kafka-questions.md](./2-kafka-questions.md)

---

### Câu 18: Exactly-once trong Kafka hoạt động ra sao?

**Ba thành phần:**
1. **Idempotent Producer** — `enable.idempotence=true`, dedup trong broker
2. **Transactional Producer** — `transactional.id`, atomic write nhiều partitions
3. **Read-Process-Write** — `consume-transform-produce` trong transaction

**Giới hạn:** Chỉ exactly-once **trong Kafka ecosystem**; end-to-end cần idempotent consumer + outbox.

```
Effective exactly-once = Kafka transactions
  + Idempotent consumer (DB dedup)
  + Outbox for external systems
```

📖 Xem thêm: [06-reliability/1-at-least-once-exactly-once.md](../06-reliability/1-at-least-once-exactly-once.md)

---

### Câu 19: Partition key design — best practices

**Mục tiêu:** Phân bố đều + đảm bảo ordering khi cần.

| Key | Ordering | Risk |
| --- | -------- | ---- |
| `orderId` | Events cùng order ordered | Tốt nếu phân bố đều |
| `userId` | Per-user ordering | Hot partition nếu power users |
| `null` (round-robin) | Không ordering | Phân bố đều nhất |

**Hot partition fix:** Salting — `hash(userId + salt)`, composite keys.

📖 Xem thêm: [07-performance-scaling/2-partitioning-strategies.md](../07-performance-scaling/2-partitioning-strategies.md)

---

### Câu 20: Rebalancing — vấn đề gì xảy ra?

**Rebalancing (Cân Bằng Lại):** Consumer group phân chia lại partition khi consumer join/leave/crash.

**Vấn đề:**
- **Stop-the-world** — consumers pause processing trong rebalance
- Duplicate processing nếu commit offset sau rebalance
- **Rebalance storm** — consumer crash loop

**Giảm impact:**
- `cooperative-sticky` assignor (incremental rebalance)
- Tăng `session.timeout.ms` cẩn thận
- Static membership (`group.instance.id`)
- Process nhanh hơn `max.poll.interval.ms`

📖 Xem thêm: [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md)

---

## Phần 4: RabbitMQ (Câu 21–24)

### Câu 21: Các loại Exchange trong RabbitMQ

| Exchange | Routing | Use Case |
| -------- | ------- | -------- |
| **Direct** | Routing key khớp chính xác | Task queue theo loại |
| **Fanout** | Broadcast tất cả bound queues | Notifications |
| **Topic** | Routing key pattern (`order.*`) | Multi-subscriber routing |
| **Headers** | Match headers | Ít dùng, phức tạp |

```
Producer → Exchange → Binding (routing key) → Queue → Consumer
```

📖 Xem thêm: [04-rabbitmq/1-exchanges-queues-bindings.md](../04-rabbitmq/1-exchanges-queues-bindings.md)

---

### Câu 22: DLQ thiết kế như thế nào?

**DLQ (Dead Letter Queue — Hàng Đợi Thư Chết):** Queue chứa messages không xử lý được sau N lần retry.

**Thiết kế:**
1. Main queue → `x-dead-letter-exchange` → DLX → DLQ
2. `x-message-ttl` hoặc retry count header
3. DLQ có monitoring + alerting riêng
4. Replay tool với manual review (tránh replay poison message)

```
Main Queue ──(N retries fail)──► DLX ──► DLQ
                                           │
                                    Alert + Manual replay
```

📖 Xem thêm: [04-rabbitmq/3-dead-letter-queue.md](../04-rabbitmq/3-dead-letter-queue.md), [06-reliability/3-dead-letter-handling.md](../06-reliability/3-dead-letter-handling.md)

---

### Câu 23: Publisher Confirm vs Consumer Ack

**Publisher Confirm:** Broker xác nhận đã **nhận và persist** message — at-least-once phía producer.

**Consumer Ack (Manual Acknowledgment — Xác Nhận Thủ Công):** Consumer xác nhận đã **xử lý xong** — broker mới xóa message.

```
Reliable flow:
  Producer ──[confirm]──► Broker ──[deliver]──► Consumer
                                                      │
                                              [ack sau xử lý]
```

**Lỗi thường gặp:** Ack trước khi xử lý xong → at-most-once semantics.

📖 Xem thêm: [04-rabbitmq/4-publisher-confirms.md](../04-rabbitmq/4-publisher-confirms.md)

---

### Câu 24: Quorum Queue vs Classic Mirrored Queue

**Quorum Queue (Hàng Đợi Đồng Thuận):** Raft-based replication — **recommended** cho HA (High Availability — Tính Sẵn Sàng Cao) từ RabbitMQ 3.8+.

**Classic Mirrored Queue:** Legacy HA — phức tạp, split-brain risk.

**Production:** Dùng Quorum Queues cho data quan trọng; classic queues cho throughput cao, data ephemeral.

📖 Xem thêm: [04-rabbitmq/5-clustering-ha.md](../04-rabbitmq/5-clustering-ha.md)

---

## Phần 5: Reliability & Operations (Câu 25–30)

### Câu 25: Poison message — xử lý thế nào?

**Poison Message (Tin Nhắn Độc):** Message gây consumer fail liên tục — block queue/partition.

**Xử lý:**
1. **Retry limit** — sau N lần → DLQ
2. **Quarantine** — isolate để phân tích
3. **Circuit breaker** — stop processing khi error rate cao
4. **Root cause** — schema mismatch, bad data, bug trong handler

📖 Xem thêm: [06-reliability/4-poison-message.md](../06-reliability/4-poison-message.md)

---

### Câu 26: Retry strategy — exponential backoff

**Exponential Backoff (Lùi Lũy Tiến):** Delay tăng dần giữa các lần retry — `1s → 2s → 4s → 8s`.

**Thêm Jitter (Nhiễu Ngẫu Nhiên):** Random hóa delay — tránh **thundering herd (bầy đàn)** khi nhiều consumer retry cùng lúc.

```
delay = min(maxDelay, baseDelay * 2^attempt) + random(0, jitter)
```

**Best practice:** Max retry count + DLQ; không retry vô hạn.

📖 Xem thêm: [06-reliability/2-retry-backoff.md](../06-reliability/2-retry-backoff.md)

---

### Câu 27: Circuit Breaker cho consumers

**Circuit Breaker (Cầu Dao Bảo Vệ):** Ngắt consumption khi downstream fail liên tục — tránh waste resources và message pile-up.

**States:** Closed (bình thường) → Open (ngắt) → Half-open (thử lại).

**Áp dụng:** Consumer gọi external API/DB — khi dependency down, pause consumption thay vì retry vô hạn.

📖 Xem thêm: [06-reliability/5-circuit-breaker-consumers.md](../06-reliability/5-circuit-breaker-consumers.md)

---

### Câu 28: Consumer lag spike — root cause phổ biến

| Root Cause | Triệu Chứng | Fix |
| ---------- | ----------- | --- |
| Deploy bug chậm handler | Lag tăng sau deploy | Rollback, optimize |
| Poison message | 1 partition lag cao | DLQ message |
| Consumer crash loop | Rebalance liên tục | Fix OOM, tăng memory |
| Traffic spike | Lag tất cả partitions | Scale consumers |
| Downstream DB slow | Handler timeout | Index, connection pool |

**Quy trình:** Metrics → Logs → Trace slow message → Partition-level analysis.

📖 Xem thêm: [09-monitoring/6-troubleshooting-playbook.md](../09-monitoring/6-troubleshooting-playbook.md)

---

### Câu 29: Message loss — nguyên nhân và phòng ngừa

**Nguyên nhân:**
- `acks=0` hoặc auto-ack trước xử lý
- Broker disk full, replication lag
- Consumer crash sau ack, trước side effect
- Misconfigured retention (message expired)

**Phòng ngừa:**
- `acks=all`, `min.insync.replicas≥2`
- Manual ack **sau** xử lý + side effect
- Monitoring: under-replicated partitions
- Outbox Pattern cho DB + event consistency

---

### Câu 30: Monitoring messaging layer — metrics nào?

**Golden Signals cho Messaging:**

| Metric | Ý Nghĩa | Alert |
| ------ | ------- | ----- |
| **Consumer Lag** | Độ trễ xử lý | Lag > threshold 5–15 phút |
| **Throughput** | msg/s in/out | Drop > 50% |
| **Error Rate** | DLQ rate, handler errors | Spike |
| **Rebalance Rate** | Instability signal | Frequent rebalance |
| **Broker Disk** | Retention risk | > 80% |

**Tools:** Prometheus + Grafana, Kafka Exporter, RabbitMQ Prometheus plugin, OpenTelemetry traces.

📖 Xem thêm: [09-monitoring/1-key-metrics.md](../09-monitoring/1-key-metrics.md), [7-production-checklist.md](../09-monitoring/7-production-checklist.md)

---

**Cập Nhật Lần Cuối:** 2026-07-03
