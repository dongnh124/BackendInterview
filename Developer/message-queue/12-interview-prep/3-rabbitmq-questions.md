# Câu Hỏi Sâu RabbitMQ — Deep Dive Questions

> 15 câu hỏi phỏng vấn chuyên sâu về RabbitMQ và AMQP (Advanced Message Queuing Protocol — Giao Thức Hàng Đợi Tin Nhắn Nâng Cao), kèm đáp án chi tiết.

## Mục Lục

1. [AMQP Fundamentals](#amqp-fundamentals)
2. [Routing & Patterns](#routing--patterns)
3. [Reliability](#reliability)
4. [Operations & HA](#operations--ha)

---

## AMQP Fundamentals

### Câu 1: RabbitMQ architecture — các thành phần?

```
Producer → Exchange → (Binding + Routing Key) → Queue → Consumer
```

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Exchange** | Nhận message từ producer, route đến queue(s) |
| **Queue** | Lưu message cho consumer |
| **Binding** | Rule liên kết exchange → queue |
| **Routing Key** | Key producer gắn kèm — exchange dùng để route |
| **Channel** | Virtual connection trong TCP connection — lightweight |

**Lưu ý:** Message **không** gửi trực tiếp vào queue — luôn qua exchange.

📖 [04-rabbitmq/1-exchanges-queues-bindings.md](../04-rabbitmq/1-exchanges-queues-bindings.md)

---

### Câu 2: Virtual Host (vhost) — tại sao quan trọng?

**vhost:** Logical isolation — exchanges, queues, permissions tách biệt per environment/tenant.

```
Production: vhost /prod
Staging:    vhost /staging
```

**Security:** User chỉ có quyền trên vhost được cấp — principle of least privilege.

---

### Câu 3: Prefetch count (`basic.qos`) — ảnh hưởng gì?

**Prefetch (Lấy Trước):** Số message broker gửi đến consumer chưa ack.

| Prefetch | Behavior |
| -------- | -------- |
| **Thấp (1–10)** | Fair distribution giữa consumers; latency thấp per message |
| **Cao (100+)** | Throughput cao; 1 consumer có thể hog messages |

**Best practice:** `prefetch=1` cho long-running tasks; `prefetch=10-50` cho fast handlers.

---

## Routing & Patterns

### Câu 4: Work Queue pattern — scale workers?

```
Producer → [task-queue] → Worker 1
                       → Worker 2
                       → Worker 3
```

**Direct Exchange** + single queue + competing consumers.

**Round-robin:** RabbitMQ distribute messages đều (không weighted mặc định).

**Fair dispatch:** `prefetch=1` — worker chỉ nhận message mới sau khi ack message hiện tại.

📖 [04-rabbitmq/2-routing-patterns.md](../04-rabbitmq/2-routing-patterns.md)

---

### Câu 5: Topic Exchange — routing key pattern?

**Pattern matching:**
- `*` — match exactly one word
- `#` — match zero or more words

```
Routing key: "order.created.us"
Pattern "order.*.us"     → ✅ match
Pattern "order.#"          → ✅ match
Pattern "payment.*.us"     → ❌ no match
```

**Use case:** Multi-tenant routing, event type filtering.

---

### Câu 6: RPC pattern với RabbitMQ — cách implement?

```
Client                          Server
  │ publish (reply_to, correlation_id)  │
  ├────────────────────────────────────►│
  │                               process│
  │◄────────────────────────────────────┤
  │ reply (same correlation_id)         │
```

**`reply_to`:** Queue name cho response
**`correlation_id`:** Match request-response

**Trade-off:** Sync-over-async — timeout handling, queue cleanup cần thiết.

---

### Câu 7: Priority Queue — khi nào dùng?

Queue argument `x-max-priority` (0–255) — message có `priority` property cao hơn được deliver trước.

**Use case:** VIP orders, critical alerts.

**Cảnh báo:** Priority queue **không** guarantee strict ordering; overhead cao hơn.

---

## Reliability

### Câu 8: Publisher Confirms — cơ chế hoạt động?

**Confirm mode:** Broker ack từng message (hoặc batch) đã persist.

```javascript
channel.confirmSelect();
channel.publish(exchange, routingKey, content, {}, (err) => {
  if (err) { /* retry or dead letter */ }
});
```

**vs Transaction:** Confirm nhẹ hơn; transaction atomic nhưng chậm — ít dùng.

📖 [04-rabbitmq/4-publisher-confirms.md](../04-rabbitmq/4-publisher-confirms.md)

---

### Câu 9: Consumer Ack modes — `ack`, `nack`, `reject`?

| Method | Behavior |
| ------ | -------- |
| **`basic.ack`** | Message processed — xóa khỏi queue |
| **`basic.nack`** | Fail — requeue hoặc dead letter |
| **`basic.reject`** | Fail single message (không batch) |

**`multiple=true`:** Ack/nack nhiều messages cùng lúc.

**Best practice:** Ack **sau** DB commit / side effect thành công.

---

### Câu 10: DLX (Dead Letter Exchange) — cấu hình?

Queue arguments:
```
x-dead-letter-exchange: dlx.exchange
x-dead-letter-routing-key: orders.failed
x-message-ttl: 86400000  (optional)
```

**Message vào DLQ khi:**
- Rejected/nacked với `requeue=false`
- TTL expired
- Queue length exceeded (`x-max-length`)

📖 [04-rabbitmq/3-dead-letter-queue.md](../04-rabbitmq/3-dead-letter-queue.md)

---

### Câu 11: Message persistence — durable queue vs persistent message?

| | Durable Queue | Persistent Message |
| - | ------------- | ------------------ |
| **Config** | `queue.durable=true` | `deliveryMode=2` |
| **Survive restart** | Queue structure | Message content |

**Cả hai cần** cho at-least-once qua broker restart. Performance hit do disk write.

---

### Câu 12: Poison message — RabbitMQ specific handling?

**Triệu chứng:** Queue depth không giảm; 1 message requeue liên tục.

**Giải pháp:**
1. `x-delivery-limit` (RabbitMQ 3.8+) — max redeliveries
2. DLX sau N rejections
3. Manual inspect: `rabbitmqadmin get queue=...`

📖 [06-reliability/4-poison-message.md](../06-reliability/4-poison-message.md)

---

## Operations & HA

### Câu 13: Quorum Queue vs Stream Queue?

| | Quorum Queue | Stream Queue |
| - | ------------ | ------------ |
| **Replication** | Raft consensus | Append-only log |
| **Use case** | Classic queue replacement, HA | High throughput, replay |
| **Consumer** | Competing consumers | Offset-based (giống Kafka) |

**Migration:** Classic mirrored queues → Quorum queues (RabbitMQ 4.x deprecate classic mirrors).

📖 [04-rabbitmq/5-clustering-ha.md](../04-rabbitmq/5-clustering-ha.md)

---

### Câu 14: Clustering — queue state across nodes?

**Quan trọng:** Queue **state** nằm trên **một node** (leader) — không phải distributed như Kafka partitions.

**HA:** Quorum queues replicate sang nodes khác.

**Client connect:** Nên dùng **load balancer** hoặc **multiple URIs** — client tự reconnect node khác.

---

### Câu 15: RabbitMQ vs Kafka — câu trả lời phỏng vấn ngắn?

| Tiêu Chí | RabbitMQ | Kafka |
| -------- | -------- | ----- |
| Model | Smart broker, dumb consumer | Dumb broker, smart consumer |
| Message lifecycle | Delete after ack | Retained log |
| Routing | Rich (exchanges) | Topic + partition key |
| Throughput | Good | Excellent |
| Replay | Không native | Native |
| Ops complexity | Lower | Higher |

**Chọn RabbitMQ:** Task queue, complex routing, RPC, team cần time-to-market nhanh.

**Chọn Kafka:** Event streaming, replay, high throughput, multiple consumer groups.

📖 [01-fundamentals/6-broker-selection-guide.md](../01-fundamentals/6-broker-selection-guide.md)

---

## Bảng Tóm Tắt Nhanh

| Chủ Đề | Best Practice |
| ------ | ------------- |
| Reliability | Publisher confirm + manual ack + durable |
| Scale | Thêm consumers + prefetch tuning |
| Failure | DLX + delivery limit + monitoring |
| HA | Quorum queues, 3+ node cluster |
| Security | vhost isolation, TLS, least privilege |

---

**Cập Nhật Lần Cuối:** 2026-07-03
