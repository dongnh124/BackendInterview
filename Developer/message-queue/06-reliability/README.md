# Độ Tin Cậy & Xử Lý Lỗi — Tổng Quan

> Chủ đề cốt lõi cho vận hành messaging trong production: Delivery Semantics (Ngữ Nghĩa Giao Hàng) thực tế, Retry Strategy (Chiến Lược Thử Lại), Dead Letter Queue (DLQ — Hàng Đợi Thư Chết), Poison Message (Tin Nhắn Độc), và Circuit Breaker (Cầu Dao Bảo Vệ) cho consumers.

## Mục Lục

1. [Tại Sao Reliability Quan Trọng](#tại-sao-reliability-quan-trọng)
2. [Kiến Trúc Xử Lý Lỗi Tổng Quan](#kiến-trúc-xử-lý-lỗi-tổng-quan)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Bài Tập Thực Hành](#bài-tập-thực-hành)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Reliability Quan Trọng

Messaging trong production **không bao giờ fail-free**. Network partition (Phân Vùng Mạng), consumer crash, downstream timeout, schema mismatch — tất cả đều xảy ra. Khác biệt giữa hệ thống amateur và production-grade nằm ở **cách thiết kế xử lý lỗi**.

| Vấn Đề | Hậu Quả Nếu Không Xử Lý | Giải Pháp |
| ------ | ----------------------- | --------- |
| Transient failure (Lỗi tạm thời) | Mất message hoặc không retry | Retry + Exponential Backoff (Lùi Lũy Tiến) |
| Permanent failure (Lỗi vĩnh viễn) | Infinite retry loop | DLQ + Poison Message handling |
| Duplicate delivery (Giao trùng) | Double charge, oversell | Idempotency (Tính Bất Biến) — xem [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md) |
| Downstream overload (Quá tải downstream) | Cascade failure (Lỗi dây chuyền) | Circuit Breaker + Backpressure |
| Silent message loss | Data inconsistency | At-least-once + monitoring |

> **Quy tắc vàng:** Production messaging mặc định là **At-least-once (Ít Nhất Một Lần)** + **Idempotent consumers** + **DLQ** + **Retry có giới hạn**. Đây là "effective exactly-once" thực tế.

---

## Kiến Trúc Xử Lý Lỗi Tổng Quan

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    RELIABILITY PIPELINE (Luồng Độ Tin Cậy)                │
│                                                                          │
│  Producer ──► Broker ──► Consumer ──► Downstream (DB, API, Service)      │
│                  │            │                                          │
│                  │            ├── SUCCESS ──► Ack (Xác Nhận)             │
│                  │            │                                          │
│                  │            ├── TRANSIENT ERROR ──► Retry Queue        │
│                  │            │         (backoff + jitter)               │
│                  │            │              │                           │
│                  │            │              └── max retries ──► DLQ   │
│                  │            │                                          │
│                  │            ├── PERMANENT ERROR ──► DLQ (ngay)         │
│                  │            │                                          │
│                  │            └── DOWNSTREAM DOWN ──► Circuit Breaker    │
│                  │                     (pause consume, alert)            │
│                  │                                                       │
│                  └── DLQ ──► Quarantine ──► Fix ──► Replay              │
└──────────────────────────────────────────────────────────────────────────┘
```

**Ba lớp bảo vệ:**

1. **Retry Layer (Lớp Thử Lại)** — xử lý lỗi tạm thời (network blip, DB connection timeout)
2. **DLQ Layer (Lớp Hàng Đợi Thư Chết)** — cô lập message không xử lý được, tránh block main queue
3. **Circuit Breaker Layer (Lớp Cầu Dao)** — bảo vệ downstream và consumer khi dependency fail liên tục

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 4–6 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-at-least-once-exactly-once.md](./1-at-least-once-exactly-once.md) | Delivery semantics thực tế, effective exactly-once | 1 giờ |
| 2 | [2-retry-backoff.md](./2-retry-backoff.md) | Retry strategy, exponential backoff, jitter | 1 giờ |
| 3 | [3-dead-letter-handling.md](./3-dead-letter-handling.md) | DLQ design, replay, monitoring | 1 giờ |
| 4 | [4-poison-message.md](./4-poison-message.md) | Detection, quarantine, root cause analysis | 45 phút |
| 5 | [5-circuit-breaker-consumers.md](./5-circuit-breaker-consumers.md) | Circuit breaker cho message consumers | 45 phút |

**Điều kiện tiên quyết:** Đọc [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md) và [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md) trước.

**Thứ tự khuyến nghị:** 1 → 2 → 3 → 4 → 5. Học delivery semantics trước, sau đó retry (nền tảng), rồi DLQ và poison message (vận hành), cuối cùng circuit breaker (bảo vệ hệ thống).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-at-least-once-exactly-once.md](./1-at-least-once-exactly-once.md) | At-least-once vs Exactly-once trong production, effective exactly-once pattern |
| [2-retry-backoff.md](./2-retry-backoff.md) | Transient vs permanent error, exponential backoff, jitter, retry budget |
| [3-dead-letter-handling.md](./3-dead-letter-handling.md) | DLQ design cross-broker, replay procedure, monitoring, runbook |
| [4-poison-message.md](./4-poison-message.md) | Phát hiện poison message, quarantine pattern, RCA (Root Cause Analysis) |
| [5-circuit-breaker-consumers.md](./5-circuit-breaker-consumers.md) | Circuit breaker states, integration với consumer, bulkhead pattern |

---

## Bài Tập Thực Hành

### Lab 1: Thiết Kế Error Handling Pipeline (45 phút)

```
Kịch bản: Order processing consumer — gọi Payment API + ghi DB.

Bài tập:
1. Phân loại lỗi: transient vs permanent (ít nhất 5 loại mỗi nhóm)
2. Thiết kế retry policy: max retries, backoff intervals, jitter
3. Vẽ DLQ flow: khi nào vào DLQ, ai xử lý, SLA replay
4. Liệt kê metrics cần monitor
```

### Lab 2: Gây Lỗi Cố Ý (60 phút)

```bash
# Docker lab với Kafka hoặc RabbitMQ
# 1. Publish message với payload invalid → verify vào DLQ
# 2. Kill consumer giữa chừng → verify redelivery + idempotency
# 3. Block downstream API → verify circuit breaker mở
# 4. Replay từ DLQ → verify không duplicate side effect
```

### Lab 3: Viết Runbook DLQ (30 phút)

```
Tạo runbook cho team on-call:
- Alert trigger: DLQ depth > 0
- Bước diagnose: đọc headers, check error logs
- Quyết định: replay / fix upstream / discard
- Escalation path
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Production nên dùng At-least-once hay Exactly-once?

**Trả lời ngắn:** **At-least-once** + **idempotent consumers** là default production. Exactly-once end-to-end rất khó và tốn kém — chỉ cần cho financial/billing critical. Xem chi tiết tại [1-at-least-once-exactly-once.md](./1-at-least-once-exactly-once.md).

### Câu 2: Retry bao nhiêu lần là đủ?

**Trả lời:** Phụ thuộc use case. Thông thường **3–5 lần** với exponential backoff. Quan trọng hơn số lần là **phân loại lỗi** — permanent error không nên retry. Xem [2-retry-backoff.md](./2-retry-backoff.md).

### Câu 3: DLQ depth > 0 có phải lúc nào cũng bad?

**Trả lời:** DLQ depth > 0 **cần investigate** nhưng không phải lúc nào cũng incident. Có thể do upstream gửi batch data lỗi, schema migration, hoặc planned maintenance. Quan trọng là có **runbook** và **SLA xử lý**.

### Câu 4: Circuit breaker khác gì retry?

**Trả lời:** **Retry** xử lý lỗi từng message. **Circuit breaker** dừng consume khi downstream fail liên tục — tránh waste resources và cascade failure. Bổ sung cho nhau, không thay thế.

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Delivery guarantees cơ bản | [01-fundamentals/3-delivery-guarantees.md](../01-fundamentals/3-delivery-guarantees.md) |
| Idempotency & dedup | [02-architecture-patterns/5-idempotency-dedup.md](../02-architecture-patterns/5-idempotency-dedup.md) |
| RabbitMQ DLQ chi tiết | [04-rabbitmq/3-dead-letter-queue.md](../04-rabbitmq/3-dead-letter-queue.md) |
| Monitoring & alerting | [09-monitoring/](../09-monitoring/) (sắp có) |

---

**Cập Nhật:** 2026-07-03
