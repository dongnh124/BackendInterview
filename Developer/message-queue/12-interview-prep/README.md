# Chuẩn Bị Phỏng Vấn Message Queue & Event Broker — Tổng Quan

> Bộ tài liệu ôn thi phỏng vấn Backend về Message Queue (Hàng Đợi Tin Nhắn), Event Broker (Broker Sự Kiện) và Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện) — câu hỏi lý thuyết, system design scenarios (tình huống thiết kế hệ thống), STAR stories (câu chuyện theo mô hình Situation-Task-Action-Result), và kế hoạch học 90 ngày.

## Mục Lục

1. [Tại Sao Cần Chuẩn Bị Riêng](#tại-sao-cần-chuẩn-bị-riêng)
2. [Cấu Trúc Phỏng Vấn Messaging](#cấu-trúc-phỏng-vấn-messaging)
3. [Lộ Trình Ôn Thi](#lộ-trình-ôn-thi)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Chiến Lược Trả Lời Hiệu Quả](#chiến-lược-trả-lời-hiệu-quả)
6. [Checklist Trước Phỏng Vấn](#checklist-trước-phỏng-vấn)
7. [Liên Kết Knowledge Base](#liên-kết-knowledge-base)

---

## Tại Sao Cần Chuẩn Bị Riêng

Kiến thức trong `01-fundamentals` đến `11-advanced` là **nền tảng**. Phỏng vấn messaging yêu cầu thêm:

| Kỹ Năng Phỏng Vấn | Mô Tả |
| ----------------- | ----- |
| **Giải thích delivery semantics** | At-least-once, Exactly-once — luôn được hỏi, cần trả lời trong 2–3 phút |
| **Trade-off thinking** | Kafka vs RabbitMQ, sync API vs async messaging, choreography vs orchestration |
| **System design** | Thiết kế order pipeline, notification system, CDC (Change Data Capture — Bắt Thay Đổi Dữ Liệu) |
| **Operational experience** | Consumer lag, poison message, DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) replay |
| **Behavioral (hành vi)** | Kể câu chuyện STAR từ incident messaging thực tế |

**Nguyên tắc cốt lõi:** Interviewer (người phỏng vấn) không chỉ kiểm tra "biết hay không" — họ đánh giá **cách bạn suy nghĩ về reliability (độ tin cậy)**, **cách troubleshoot lag**, và **cách thiết kế idempotent consumers (consumer bất biến khi lặp lại)**.

---

## Cấu Trúc Phỏng Vấn Messaging

```
┌─────────────────────────────────────────────────────────────┐
│           TYPICAL MESSAGING INTERVIEW ROUNDS                │
├─────────────────────────────────────────────────────────────┤
│  Round 1: Screening (30–45 phút)                            │
│    → MQ vs Event Broker, delivery guarantees, Pub/Sub       │
│                                                             │
│  Round 2: Technical Deep Dive (60–90 phút)                  │
│    → Kafka partitions, consumer groups, RabbitMQ routing    │
│    → Outbox Pattern, idempotency, DLQ design                │
│                                                             │
│  Round 3: System Design (45–60 phút)                        │
│    → Order processing, notification fan-out, CDC pipeline   │
│                                                             │
│  Round 4: Behavioral + Operations (30–45 phút)              │
│    → STAR stories: lag spike, message loss, broker outage   │
└─────────────────────────────────────────────────────────────┘
```

| Vòng | Trọng Tâm | Tài Liệu Ôn |
| ---- | --------- | ----------- |
| Screening | Fundamentals, delivery semantics | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) Phần 1–2 |
| Technical Kafka | Partitions, offsets, exactly-once | [2-kafka-questions.md](./2-kafka-questions.md) |
| Technical RabbitMQ | Exchanges, DLQ, confirms | [3-rabbitmq-questions.md](./3-rabbitmq-questions.md) |
| System Design | EDA, saga, outbox | [1-system-design-scenarios.md](./1-system-design-scenarios.md) |
| Behavioral | Incident response | [4-star-stories.md](./4-star-stories.md) |

---

## Lộ Trình Ôn Thi

### 2 Tuần Trước Phỏng Vấn (Crash Course)

```
Tuần 1:
├── Ngày 1–2: Delivery guarantees + idempotency (bắt buộc)
├── Ngày 3–4: Kafka — topics, partitions, consumer groups
├── Ngày 5–6: RabbitMQ — exchanges, DLQ, publisher confirms
└── Ngày 7: Outbox Pattern + Saga Pattern

Tuần 2:
├── Ngày 8–9: System design scenarios (order, notification)
├── Ngày 10–11: INTERVIEW_GUIDE — 30 câu hỏi
├── Ngày 12: Chuẩn bị 3 STAR stories
├── Ngày 13: Mock interview với đồng nghiệp
└── Ngày 14: Review monitoring, lag troubleshooting
```

### 4 Tuần Trước Phỏng Vấn (Kỹ Lưỡng)

Xem [5-90-day-study-plan.md](./5-90-day-study-plan.md) — tập trung Giai Đoạn 4–5 (ngày 57–90).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung | Thời Gian Ôn |
| ---- | -------- | ------------ |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | Top 30 câu hỏi + đáp án chi tiết | 4–6 giờ |
| [1-system-design-scenarios.md](./1-system-design-scenarios.md) | Order processing, notification, CDC | 3–4 giờ |
| [2-kafka-questions.md](./2-kafka-questions.md) | 20 câu hỏi sâu Kafka | 2–3 giờ |
| [3-rabbitmq-questions.md](./3-rabbitmq-questions.md) | 15 câu hỏi sâu RabbitMQ | 2 giờ |
| [4-star-stories.md](./4-star-stories.md) | Template incident STAR | 1–2 giờ |
| [5-90-day-study-plan.md](./5-90-day-study-plan.md) | Lộ trình học 90 ngày | Tham khảo |

---

## Chiến Lược Trả Lời Hiệu Quả

### Framework WHAT–WHY–HOW–TRADE-OFF

```
WHAT:   Định nghĩa khái niệm (1–2 câu)
WHY:    Tại sao quan trọng / khi nào dùng
HOW:    Cơ chế hoạt động hoặc cách implement
TRADE-OFF: So sánh với alternative, rủi ro production
```

**Ví dụ — "At-least-once delivery là gì?"**

- **WHAT:** Message được deliver ít nhất một lần; có thể duplicate (trùng lặp).
- **WHY:** Production default — không mất message quan trọng hơn là xử lý trùng.
- **HOW:** Manual ack sau khi xử lý xong; producer `acks=all`; retry khi fail.
- **TRADE-OFF:** Consumer phải idempotent; exactly-once phức tạp hơn nhiều.

### System Design — Framework RESHADED

```
R — Requirements: Functional + Non-functional (throughput, latency, ordering)
E — Estimate: Messages/s, retention, storage
S — Storage: Topic/queue design, schema
H — High-level: Producer → Broker → Consumer diagram
A — APIs: Event schema, partition key strategy
D — Detailed: DLQ, retry, outbox, monitoring
E — Edge cases: Broker down, consumer crash, hot partition
D — Delivery: Idempotency, exactly-once strategy
```

---

## Checklist Trước Phỏng Vấn

### Kiến Thức Bắt Buộc

- [ ] Giải thích At-most-once, At-least-once, Exactly-once không cần nhìn tài liệu
- [ ] So sánh Kafka vs RabbitMQ cho 2 use case khác nhau
- [ ] Mô tả Outbox Pattern và dual-write problem (vấn đề ghi kép)
- [ ] Giải thích consumer lag và cách scale consumer
- [ ] Thiết kế DLQ + retry strategy

### Kỹ Năng Thực Hành

- [ ] Đã chạy Kafka/RabbitMQ local (Docker)
- [ ] Đã implement producer + consumer với manual ack
- [ ] Đã gây lỗi và troubleshoot (kill consumer, xem lag)

### Behavioral

- [ ] 3 STAR stories sẵn sàng (lag spike, message duplicate, broker incident)
- [ ] Mỗi story có số liệu đo lường (lag giảm từ X → Y, downtime Z phút)

---

## Liên Kết Knowledge Base

| Chủ Đề Phỏng Vấn | Tài Liệu Sâu |
| ---------------- | ------------ |
| Delivery guarantees | [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md) |
| Outbox, Saga | [02-architecture-patterns/](../02-architecture-patterns/) |
| Kafka deep dive | [03-apache-kafka/](../03-apache-kafka/) |
| RabbitMQ deep dive | [04-rabbitmq/](../04-rabbitmq/) |
| DLQ, retry, poison message | [06-reliability/](../06-reliability/) |
| Consumer lag, monitoring | [09-monitoring/](../09-monitoring/) |
| Broker selection | [01-fundamentals/6-broker-selection-guide.md](../01-fundamentals/6-broker-selection-guide.md) |

---

**Cập Nhật Lần Cuối:** 2026-07-03
