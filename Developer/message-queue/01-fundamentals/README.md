# Nền Tảng Message Queue & Event Broker — Tổng Quan

> Chủ đề nền tảng bắt buộc trước khi đi sâu vào Apache Kafka, RabbitMQ hay bất kỳ broker nào: khái niệm messaging, mô hình giao tiếp, delivery semantics (ngữ nghĩa giao hàng), ordering (thứ tự), backpressure (áp lực ngược), và cách chọn broker phù hợp.

## Mục Lục

1. [Tại Sao Cần Học Nền Tảng Trước](#tại-sao-cần-học-nền-tảng-trước)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Bài Tập Thực Hành](#bài-tập-thực-hành)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Cần Học Nền Tảng Trước

Message Queue (Hàng Đợi Tin Nhắn) và Event Broker (Broker Sự Kiện) là **hạ tầng giao tiếp bất đồng bộ (asynchronous communication — Giao Tiếp Bất Đồng Bộ)** giữa các service trong hệ thống phân tán. Trước khi học cấu hình Kafka hay RabbitMQ, bạn cần nắm vững:

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **MQ vs Event Broker** | Chọn đúng loại hệ thống — task queue hay event log |
| **Point-to-Point vs Pub/Sub** | Quyết định ai nhận message, bao nhiêu consumer |
| **Delivery Guarantees** | At-least-once (ít nhất một lần) là mặc định production — phải thiết kế idempotency |
| **Message Ordering** | Partition key, FIFO — ảnh hưởng thiết kế topic/queue |
| **Backpressure** | Tránh sập downstream khi producer nhanh hơn consumer |
| **Broker Selection** | Kafka, RabbitMQ, SQS — mỗi loại có trade-offs riêng |

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN SYSTEM (Hệ Thống Hướng Sự Kiện)      │
│                                                                     │
│  ┌──────────────┐         ┌─────────────────────┐    ┌───────────┐ │
│  │  Producer    │ ──────► │  Message Broker     │───►│ Consumer  │ │
│  │  (Nhà SX)    │ publish │  (Kafka/RabbitMQ/   │pull│ (Bên Nhận)│ │
│  │              │         │   SQS...)           │    │           │ │
│  └──────────────┘         │                     │    └───────────┘ │
│                           │  ┌─────┐ ┌─────┐   │    ┌───────────┐ │
│  ┌──────────────┐         │  │Topic│ │Queue│   │───►│ Consumer  │ │
│  │  Producer    │ ──────► │  │/Part│ │     │   │    │ (Nhóm 2)  │ │
│  └──────────────┘         │  └─────┘ └─────┘   │    └───────────┘ │
│                           └─────────────────────┘                  │
│                                                                     │
│  Vai trò broker: Decoupling (Tách Rời), Buffering (Đệm),           │
│                  Load Leveling (Cân Bằng Tải), Durability (Bền Vững) │
└─────────────────────────────────────────────────────────────────────┘
```

**Luồng cơ bản:**

1. **Producer** gửi message vào broker (publish/send)
2. **Broker** lưu trữ, định tuyến (route) message
3. **Consumer** đọc và xử lý message (consume)
4. Consumer **acknowledge (ack — xác nhận)** khi xử lý xong — hoặc **nack/reject** để retry/DLQ

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 4–6 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-message-queue-basics.md](./1-message-queue-basics.md) | MQ vs Event Broker, terminology, use cases | 1 giờ |
| 2 | [2-pub-sub-vs-point-to-point.md](./2-pub-sub-vs-point-to-point.md) | Publish/Subscribe vs Point-to-Point patterns | 45 phút |
| 3 | [3-delivery-guarantees.md](./3-delivery-guarantees.md) | At-most-once, At-least-once, Exactly-once | 1 giờ |
| 4 | [4-ordering-and-sequencing.md](./4-ordering-and-sequencing.md) | Ordering, partition keys, FIFO | 45 phút |
| 5 | [5-backpressure-flow-control.md](./5-backpressure-flow-control.md) | Backpressure, rate limiting, flow control | 45 phút |
| 6 | [6-broker-selection-guide.md](./6-broker-selection-guide.md) | Decision tree chọn broker | 1 giờ |

**Thứ tự học được khuyến nghị:** 1 → 2 → 3 → 4 → 5 → 6. File 3 (delivery guarantees) là chủ đề **được hỏi nhiều nhất** trong phỏng vấn — nên học kỹ.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-message-queue-basics.md](./1-message-queue-basics.md) | Khái niệm MQ, Event Broker, Producer/Consumer, sync vs async |
| [2-pub-sub-vs-point-to-point.md](./2-pub-sub-vs-point-to-point.md) | Hai mô hình messaging cốt lõi và khi nào dùng |
| [3-delivery-guarantees.md](./3-delivery-guarantees.md) | Ba mức đảm bảo giao hàng, trade-offs, idempotency |
| [4-ordering-and-sequencing.md](./4-ordering-and-sequencing.md) | Thứ tự message, partition key, global vs partial ordering |
| [5-backpressure-flow-control.md](./5-backpressure-flow-control.md) | Xử lý khi producer vượt khả năng consumer |
| [6-broker-selection-guide.md](./6-broker-selection-guide.md) | So sánh Kafka, RabbitMQ, SQS, Redis — decision tree |

---

## Bài Tập Thực Hành

### Lab 1: Sync vs Async — So Sánh Trực Tiếp (30 phút)

```
Kịch bản: Service A gọi Service B để gửi email.

Bài tập:
1. Vẽ sequence diagram cho HTTP sync call
2. Vẽ sequence diagram cho async qua message queue
3. Liệt kê 3 failure scenario cho mỗi cách
4. Kết luận: khi nào chọn async?
```

### Lab 2: Docker — Producer/Consumer Đầu Tiên (45 phút)

```bash
# Dùng docker-compose từ repo hoặc chạy RabbitMQ standalone
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

# Viết producer gửi 10 message vào queue "hello"
# Viết consumer in ra message và ack
# Kill consumer giữa chừng — quan sát message còn trong queue
```

### Lab 3: Delivery Semantics Thực Nghiệm (45 phút)

```
1. Cấu hình consumer auto-ack → quan sát message loss khi crash
2. Cấu hình manual ack sau khi xử lý → message được redeliver
3. Implement idempotent handler với dedup key
```

### Lab 4: Broker Selection (30 phút)

```
Cho 3 use case:
- Order processing (10K orders/giây, cần replay)
- Background image resize (task queue, ít message)
- Real-time notification fan-out (1 event → 100K users)

Chọn broker cho mỗi case, giải thích trade-offs
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Message Queue khác Event Broker như thế nào?

**Gợi ý trả lời:** Message Queue tập trung vào **task distribution (phân phối tác vụ)** — message thường bị xóa sau khi consumer xử lý xong. Event Broker (như Kafka) lưu **event log (nhật ký sự kiện)** bền vững — nhiều consumer có thể đọc cùng event, hỗ trợ **replay (phát lại)**. MQ phù hợp work queue; Event Broker phù hợp event streaming và analytics.

### Câu 2: Giải thích At-least-once, At-most-once, Exactly-once?

**Gợi ý trả lời:** **At-most-once:** gửi tối đa 1 lần, có thể mất message. **At-least-once:** không mất nhưng có thể duplicate (trùng lặp) — cần idempotent consumer. **Exactly-once:** mỗi message được xử lý đúng 1 lần — khó đạt, thường là "effective exactly-once" qua idempotency + transactional outbox. Production phổ biến nhất: **at-least-once + idempotency**.

### Câu 3: Khi nào dùng sync API vs async messaging?

**Gợi ý trả lời:** Dùng **sync (HTTP/gRPC)** khi cần response ngay, logic đơn giản, latency thấp. Dùng **async messaging** khi: decouple services, xử lý peak load, fire-and-forget, cần durability/retry, hoặc nhiều subscriber cùng nhận event. Trade-off: async phức tạp hơn (ordering, duplicate, monitoring).

### Câu 4: Idempotency là gì? Tại sao quan trọng với messaging?

**Gợi ý trả lời:** **Idempotency (Tính Bất Biến Khi Lặp Lại)** — gọi cùng operation nhiều lần cho cùng kết quả. Với at-least-once delivery, consumer có thể nhận message trùng. Idempotent handler dùng **deduplication key (khóa khử trùng)** hoặc kiểm tra trạng thái trước khi thực hiện side effect.

### Câu 5: Backpressure là gì?

**Gợi ý trả lời:** **Backpressure (Áp Lực Ngược)** xảy ra khi producer gửi nhanh hơn consumer xử lý — queue depth tăng, memory/disk đầy, latency tăng. Giải pháp: rate limiting, throttle producer, scale consumer, hoặc reject/slow down ở broker level.

---

**Xem tiếp:** [1-message-queue-basics.md](./1-message-queue-basics.md) — bắt đầu với khái niệm Message Queue và Event Broker.
