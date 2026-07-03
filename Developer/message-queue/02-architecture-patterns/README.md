# Mẫu Kiến Trúc Hướng Sự Kiện — Tổng Quan

> Chủ đề cốt lõi cho thiết kế hệ thống phân tán: Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện), CQRS (Command Query Responsibility Segregation — Tách Trách Nhiệm Đọc/Ghi), Event Sourcing (Lưu Trữ Sự Kiện), Saga Pattern (Mẫu Saga), Outbox/Inbox Pattern, và Idempotency (Tính Bất Biến Khi Lặp Lại).

## Mục Lục

1. [Tại Sao Cần Học Architecture Patterns](#tại-sao-cần-học-architecture-patterns)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Bài Tập Thực Hành](#bài-tập-thực-hành)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Cần Học Architecture Patterns

Sau khi nắm [01-fundamentals](../01-fundamentals/README.md) (delivery guarantees, pub/sub, ordering), bước tiếp theo là **thiết kế hệ thống** dùng messaging đúng cách — không chỉ "gửi message qua Kafka".

| Pattern | Vấn Đề Giải Quyết | Được Hỏi Phỏng Vấn |
| ------- | ----------------- | ------------------- |
| **Event-Driven Architecture** | Decouple services, scale độc lập | ⭐⭐⭐ System design |
| **CQRS & Event Sourcing** | Tách read/write, audit trail | ⭐⭐ Senior/Architect |
| **Saga Pattern** | Distributed transaction không dùng 2PC | ⭐⭐⭐ System design |
| **Outbox/Inbox Pattern** | Dual-write problem, atomic publish | ⭐⭐⭐ Production |
| **Idempotency & Dedup** | At-least-once + duplicate handling | ⭐⭐⭐ Luôn được hỏi |

> **Lưu ý:** Các pattern này **bổ sung cho nhau**, không thay thế lẫn nhau. Production thường kết hợp: EDA + Outbox + Idempotent consumers + Saga cho multi-service flows.

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN MICROSERVICES (Microservices Hướng Sự Kiện)    │
│                                                                         │
│  ┌──────────┐    OrderCreated     ┌──────────┐    PaymentProcessed    │
│  │  Order   │ ──────────────────► │  Kafka   │ ──────────────────►    │
│  │  Service │    (Outbox relay)   │  Broker  │                        │
│  └────┬─────┘                     └────┬─────┘                        │
│       │ DB + Outbox table              │                               │
│       │ (atomic write)                 ├──► Inventory Service           │
│                                        ├──► Notification Service       │
│                                        └──► Analytics (Event Sourcing)   │
│                                                                         │
│  Cross-cutting: Idempotency keys, Correlation ID, Saga orchestrator      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Luồng điển hình:**

1. Service nhận **Command (Lệnh)** → ghi DB + Outbox trong **cùng transaction**
2. **Outbox Relay (Bộ Chuyển Tiếp Outbox)** publish event lên broker
3. Consumer xử lý **Event (Sự Kiện)** với **idempotent handler**
4. Multi-service flow dùng **Saga** với compensation (bù trừ) khi lỗi

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-event-driven-architecture.md](./1-event-driven-architecture.md) | EDA fundamentals, event types, bounded context | 1.5 giờ |
| 2 | [5-idempotency-dedup.md](./5-idempotency-dedup.md) | Idempotent consumers, dedup keys | 1 giờ |
| 3 | [4-outbox-inbox-pattern.md](./4-outbox-inbox-pattern.md) | Dual-write, transactional messaging | 1.5 giờ |
| 4 | [3-saga-pattern.md](./3-saga-pattern.md) | Choreography vs Orchestration | 1.5 giờ |
| 5 | [2-cqrs-event-sourcing.md](./2-cqrs-event-sourcing.md) | CQRS, projections, event store | 1.5 giờ |

**Thứ tự khuyến nghị:** 1 → 5 → 4 → 3 → 2. Học **Idempotency** và **Outbox** trước Saga vì chúng là nền tảng production. CQRS/Event Sourcing là nâng cao — học sau khi đã hiểu EDA cơ bản.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-event-driven-architecture.md](./1-event-driven-architecture.md) | EDA vs request-response, domain events, event notification vs event-carried state transfer |
| [2-cqrs-event-sourcing.md](./2-cqrs-event-sourcing.md) | Command/Query separation, event store, projections, snapshots |
| [3-saga-pattern.md](./3-saga-pattern.md) | Distributed transactions, choreography, orchestration, compensation |
| [4-outbox-inbox-pattern.md](./4-outbox-inbox-pattern.md) | Outbox table, relay process, inbox dedup, dual-write problem |
| [5-idempotency-dedup.md](./5-idempotency-dedup.md) | Idempotency keys, dedup table, natural idempotency, TTL |

---

## Bài Tập Thực Hành

### Lab 1: Thiết Kế EDA Cho Order Flow (45 phút)

```
Kịch bản: User đặt hàng → trừ kho → thanh toán → gửi email.

Bài tập:
1. Vẽ bounded context và các domain event
2. Xác định sync vs async cho từng bước
3. Liệt kê failure scenario và cách xử lý
4. Chọn choreography hay orchestration — giải thích
```

### Lab 2: Implement Outbox Pattern (60 phút)

```
1. Tạo bảng outbox trong PostgreSQL
2. Trong transaction: INSERT order + INSERT outbox row
3. Viết relay process poll outbox → publish Kafka
4. Test: kill relay giữa chừng — verify không mất event
```

### Lab 3: Idempotent Consumer (45 phút)

```
1. Consumer nhận message với idempotency_key
2. Check dedup table trước khi xử lý
3. Gửi cùng message 3 lần — verify chỉ xử lý 1 lần
4. Thêm TTL cleanup cho dedup records
```

### Lab 4: Saga Compensation (60 phút)

```
Kịch bản: Order → Reserve inventory → Charge payment
- Payment fail → compensate: release inventory, cancel order
Implement orchestration saga với state machine đơn giản
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Event-Driven Architecture khác Request-Response như thế nào?

**Gợi ý trả lời:** **Request-Response (Yêu Cầu-Phản Hồi):** caller chờ response đồng bộ, tight coupling về availability và latency. **EDA:** producer publish event, không biết consumer nào xử lý — **loose coupling (liên kết lỏng)**, scale độc lập, chịu được peak load. Trade-off: eventual consistency (nhất quán cuối cùng), khó debug hơn, cần idempotency và monitoring.

### Câu 2: Outbox Pattern giải quyết vấn đề gì?

**Gợi ý trả lời:** **Dual-write problem (Vấn Đề Ghi Kép)** — cần ghi DB và publish message atomically nhưng không thể transaction xuyên DB và Kafka. Outbox: ghi event vào bảng outbox **trong cùng DB transaction**, relay process publish sau — đảm bảo **at-least-once publish** không mất event.

### Câu 3: Saga Choreography vs Orchestration?

**Gợi ý trả lời:** **Choreography (Điệu Múa):** mỗi service listen event và publish event tiếp theo — decentralized, đơn giản khi ít bước. **Orchestration (Điều Phối):** central orchestrator điều khiển flow — dễ theo dõi, debug, timeout; phù hợp flow phức tạp. Trade-off: orchestrator là single point (cần HA).

### Câu 4: CQRS và Event Sourcing có bắt buộc dùng cùng nhau không?

**Gợi ý trả lời:** **Không.** CQRS tách read model và write model — có thể dùng CRUD write side. Event Sourcing lưu state dưới dạng event sequence — thường kết hợp CQRS vì read model được build từ projections. Cả hai đều tăng complexity — chỉ dùng khi có lý do rõ (audit, temporal queries, high read scale).

### Câu 5: Làm sao đảm bảo consumer idempotent?

**Gợi ý trả lời:** (1) **Idempotency key** trong message header/payload. (2) **Dedup table** — check-before-process, insert key sau success. (3) **Natural idempotency** — UPSERT, conditional update. (4) **Inbox pattern** cho exactly-once processing semantics. Luôn thiết kế idempotent **by default** vì at-least-once là mặc định production.

---

**Xem tiếp:** [1-event-driven-architecture.md](./1-event-driven-architecture.md) — bắt đầu với Event-Driven Architecture fundamentals.
