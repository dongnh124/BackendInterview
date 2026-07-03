# RabbitMQ — Tổng Quan & AMQP

> Chủ đề chuyên sâu về RabbitMQ — message broker (broker tin nhắn) phổ biến dựa trên AMQP (Advanced Message Queuing Protocol — Giao Thức Hàng Đợi Tin Nhắn Nâng Cao): exchanges, queues, bindings, routing patterns, DLQ, publisher confirms, và clustering/HA.

## Mục Lục

1. [Tại Sao Học RabbitMQ](#tại-sao-học-rabbitmq)
2. [RabbitMQ Trong Ecosystem](#rabbitmq-trong-ecosystem)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học RabbitMQ

RabbitMQ là **Message Broker (Broker Tin Nhắn)** được dùng rộng rãi cho task queue, RPC (Remote Procedure Call — Gọi Thủ Tục Từ Xa), workflow orchestration, và decoupling microservices. Sau khi nắm [01-fundamentals](../01-fundamentals/README.md), RabbitMQ là broker **bắt buộc** để so sánh với Kafka và trả lời system design interview.

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **Exchanges & Bindings** | Routing linh hoạt — điểm khác biệt cốt lõi so với Kafka |
| **Routing Patterns** | Work queue, pub/sub, topic routing — pattern thực tế |
| **Dead Letter Queue (DLQ)** | Xử lý poison message, retry — hay gặp production |
| **Publisher Confirms & Ack** | Đảm bảo delivery semantics |
| **Clustering & HA** | Mirrored queues vs quorum queues — quyết định production |

> **Điều kiện tiên quyết:** Đã học [delivery guarantees](../01-fundamentals/3-delivery-guarantees.md), [pub/sub vs point-to-point](../01-fundamentals/2-pub-sub-vs-point-to-point.md), và [idempotency](../02-architecture-patterns/5-idempotency-dedup.md).

---

## RabbitMQ Trong Ecosystem

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RABBITMQ ECOSYSTEM                                    │
│                                                                         │
│  ┌─────────────┐   ┌─────────────────────────────────┐   ┌────────────┐ │
│  │  Publishers │──►│         RabbitMQ Broker          │──►│ Consumers  │ │
│  │  (Producers)│   │  ┌──────────┐    ┌──────────┐   │   │ (Workers)  │ │
│  └─────────────┘   │  │ Exchange │───►│  Queue   │   │   └────────────┘ │
│                    │  └──────────┘    └──────────┘   │                  │
│                    │       ▲ binding (routing key)     │                  │
│                    └─────────────────────────────────┘                  │
│                                                                         │
│  Plugins: Management UI, Shovel, Federation, Stream                      │
│  Managed: CloudAMQP, Amazon MQ, Azure Service Bus (AMQP)                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Thành phần chính:**

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Exchange (Bộ Trao Đổi)** | Nhận message từ publisher, route đến queue(s) theo rules |
| **Queue (Hàng Đợi)** | Lưu message cho đến khi consumer xử lý và ack |
| **Binding (Liên Kết)** | Rule kết nối exchange với queue (routing key, headers) |
| **Channel (Kênh)** | Virtual connection trong TCP connection — lightweight |
| **Virtual Host (vhost)** | Namespace logic — tách môi trường/tenant |

---

## Kiến Trúc Tổng Quan

```
                    ┌──────────────────────────────────────┐
                    │         RABBITMQ BROKER               │
                    │                                      │
  Publisher ────────►│  Exchange "orders.topic"             │
  routing_key=      │       │                              │
  "order.created"   │       ├──binding──► Queue "email"    │──► Consumer A
                    │       ├──binding──► Queue "inventory"│──► Consumer B
                    │       └──binding──► Queue "audit"   │──► Consumer C
                    │                                      │
                    │  DLX (Dead Letter Exchange)          │
                    │       └──► Queue "orders.dlq"        │
                    └──────────────────────────────────────┘
```

**Luồng cơ bản (AMQP 0-9-1):**

1. Publisher gửi message đến **exchange** kèm **routing key**
2. Exchange áp dụng **binding rules** → message vào một hoặc nhiều **queue(s)**
3. Consumer **subscribe** queue, nhận message theo **prefetch (QoS)**
4. Consumer **ack** (hoặc **nack/reject**) → broker xóa hoặc requeue message

```
So sánh nhanh với Kafka:
  RabbitMQ: push-based (broker đẩy đến consumer), routing linh hoạt
  Kafka:    pull-based (consumer kéo), log-based, replay theo offset
```

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 8–10 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-exchanges-queues-bindings.md](./1-exchanges-queues-bindings.md) | Direct, Fanout, Topic, Headers exchanges | 2 giờ |
| 2 | [2-routing-patterns.md](./2-routing-patterns.md) | Work queue, pub/sub, routing, RPC | 1.5 giờ |
| 3 | [3-dead-letter-queue.md](./3-dead-letter-queue.md) | DLX, DLQ design, poison message | 1.5 giờ |
| 4 | [4-publisher-confirms.md](./4-publisher-confirms.md) | Publisher confirms, consumer ack modes | 1.5 giờ |
| 5 | [5-clustering-ha.md](./5-clustering-ha.md) | Mirrored queues, quorum queues, HA | 2 giờ |

**Thứ tự khuyến nghị:** 1 → 2 → 4 → 3 → 5. Nắm **exchange + routing** trước, sau đó **confirms/ack** (delivery semantics), rồi **DLQ** và **HA** cho production.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-exchanges-queues-bindings.md](./1-exchanges-queues-bindings.md) | 4 loại exchange, binding, routing key, AMQP model |
| [2-routing-patterns.md](./2-routing-patterns.md) | Enterprise Integration Patterns với RabbitMQ |
| [3-dead-letter-queue.md](./3-dead-letter-queue.md) | DLX configuration, TTL, max-length, retry pattern |
| [4-publisher-confirms.md](./4-publisher-confirms.md) | Confirm mode, mandatory flag, ack/nack/reject |
| [5-clustering-ha.md](./5-clustering-ha.md) | Classic mirrored vs quorum queues, cluster setup |

---

## Bài Tập Thực Hành

### Lab 1: Dựng RabbitMQ Local (20 phút)

```bash
# Docker
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management

# Management UI: http://localhost:15672 (guest/guest)
```

### Lab 2: Exchange Types (45 phút)

```
1. Tạo Direct exchange — route theo routing key chính xác
2. Tạo Fanout exchange — broadcast đến tất cả bound queues
3. Tạo Topic exchange — pattern matching (*.error, order.#)
4. Quan sát message flow trên Management UI
```

### Lab 3: Work Queue & Fair Dispatch (30 phút)

```
1. Tạo queue durable, 2 consumers cùng queue
2. Set prefetch_count=1 — verify fair dispatch
3. Consumer chậm không block consumer nhanh
```

### Lab 4: Publisher Confirms & Consumer Ack (45 phút)

```
1. Bật publisher confirms — verify ack từ broker
2. Consumer manual ack — kill consumer giữa chừng, message requeue
3. Consumer nack(requeue=false) — message vào DLQ
```

### Lab 5: RPC Pattern (45 phút)

```
1. Client gửi request kèm reply_to queue + correlation_id
2. Server xử lý, gửi response về reply_to
3. Client match correlation_id — verify request/response pairing
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: RabbitMQ khác Kafka ở điểm nào?

**Gợi ý trả lời:** RabbitMQ là **smart broker, dumb consumer** — routing phức tạp tại exchange, message xóa sau ack, push-based. Kafka là **dumb broker, smart consumer** — append-only log, consumer tự quản offset, pull-based, replay dễ. RabbitMQ phù hợp task queue, RPC, routing linh hoạt. Kafka phù hợp event streaming, high throughput, replay.

### Câu 2: Các loại Exchange và khi nào dùng?

**Gợi ý trả lời:** **Direct** — routing key khớp chính xác (task routing). **Fanout** — broadcast tất cả bound queues (notifications). **Topic** — pattern matching với `*` và `#` (event categorization). **Headers** — route theo header attributes (ít dùng hơn).

### Câu 3: Dead Letter Queue thiết kế như thế nào?

**Gợi ý trả lời:** Cấu hình **DLX (Dead Letter Exchange)** trên queue chính — message bị reject/nack, TTL hết hạn, hoặc queue đầy sẽ route sang DLQ. DLQ cần monitoring, alerting, và quy trình replay/manual fix. Kết hợp retry queue với TTL + DLX cho exponential backoff.

### Câu 4: Publisher Confirm vs Consumer Ack?

**Gợi ý trả lời:** **Publisher confirm** — broker ack đã nhận và persist message (at-least-once từ phía producer). **Consumer ack** — consumer báo đã xử lý xong, broker mới xóa message. Cần cả hai cho end-to-end reliability; thiếu confirm → message loss, thiếu ack → duplicate khi consumer crash.

### Câu 5: Mirrored Queue vs Quorum Queue?

**Gợi ý trả lời:** **Classic mirrored queues** (deprecated) — replicate toàn bộ queue sang node khác, dễ split-brain. **Quorum queues** (khuyến nghị) — dùng Raft consensus, durability và consistency tốt hơn, là default từ RabbitMQ 4.x. Stream queues cho use case log-like.

---

**Xem tiếp:** [1-exchanges-queues-bindings.md](./1-exchanges-queues-bindings.md) — bắt đầu với AMQP model và exchange types.
