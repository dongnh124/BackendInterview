# STAR Stories — Template Câu Chuyện Incident Messaging

> Hướng dẫn chuẩn bị câu chuyện behavioral interview theo mô hình STAR (Situation — Tình Huống, Task — Nhiệm Vụ, Action — Hành Động, Result — Kết Quả) cho Backend developer làm việc với Message Queue và Event Broker.

## Mục Lục

1. [STAR Framework](#star-framework)
2. [Các Loại Câu Hỏi Behavioral Messaging](#các-loại-câu-hỏi-behavioral-messaging)
3. [Template Stories](#template-stories)
4. [Câu Chuyện Mẫu Theo Chủ Đề](#câu-chuyện-mẫu-theo-chủ-đề)
5. [Cách Viết STAR Story Của Bạn](#cách-viết-star-story-của-bạn)
6. [Câu Hỏi Ngược Cho Interviewer](#câu-hỏi-ngược-cho-interviewer)

---

## STAR Framework

```
┌─────────────────────────────────────────────────────────────┐
│  S — SITUATION (15%)                                        │
│  Bối cảnh: hệ thống messaging, scale, broker đang dùng     │
├─────────────────────────────────────────────────────────────┤
│  T — TASK (15%)                                             │
│  Nhiệm vụ cụ thể của BẠN trong incident                    │
├─────────────────────────────────────────────────────────────┤
│  A — ACTION (50%)                                           │
│  Chi tiết kỹ thuật: metrics, logs, root cause, fix          │
├─────────────────────────────────────────────────────────────┤
│  R — RESULT (20%)                                           │
│  Số liệu: lag giảm, downtime, prevention measures           │
└─────────────────────────────────────────────────────────────┘
```

**Thời gian trả lời:** 2–3 phút mỗi story. Practice để không quá dài.

**Nguyên tắc:** Dùng "tôi", không "chúng tôi" — interviewer muốn biết **đóng góp cá nhân**.

---

## Các Loại Câu Hỏi Behavioral Messaging

| Loại | Câu Hỏi Mẫu | Story Cần Chuẩn Bị |
| ---- | ----------- | ------------------ |
| **Consumer lag** | "Kể về lần consumer lag spike" | Lag troubleshooting |
| **Message loss/duplicate** | "Kể về bug liên quan message" | Delivery semantics fix |
| **Poison message** | "Kể về message gây hệ thống fail" | DLQ implementation |
| **Broker outage** | "Kể khi Kafka/RabbitMQ down" | HA, failover |
| **Migration** | "Kể về migrate messaging system" | Broker migration |
| **Design decision** | "Kể khi bạn chọn Kafka vs RabbitMQ" | Architecture choice |
| **Pressure** | "Kể khi fix production dưới áp lực" | Hotfix incident |
| **Post-mortem** | "Kể về bài học từ incident" | Process improvement |

**Chuẩn bị tối thiểu:** 5 stories cover nhiều loại câu hỏi.

---

## Template Stories

### Template 1: Consumer Lag Spike

```markdown
## [Tên Story — e.g., "Kafka Lag Spike Sau Black Friday"]

**S — Situation:**
Hệ thống order processing dùng Kafka với topic `orders` (12 partitions).
Vào [sự kiện], consumer lag tăng từ [X] messages lên [Y] messages trong [Z] giờ.
Alert PagerDuty fire lúc [thời điểm].

**T — Task:**
Tôi là [role] on-call cho fulfillment consumer group.
Nhiệm vụ: restore lag về normal trong [timeframe] trước khi SLA breach.

**A — Action:**
1. Tôi check Grafana — lag tập trung ở partition [N] (hot partition)
2. Phân tích: partition key = `sellerId` — top seller chiếm 40% traffic
3. Short-term: scale consumers từ 6 → 12 (match partition count)
4. Tôi tăng `max.poll.records` và batch DB writes — throughput tăng 3x
5. Long-term: đề xuất salt partition key `sellerId + bucket(0-9)`

**R — Result:**
- Lag giảm từ [Y] về [X] trong [time]
- Thêm alert: lag per partition > 10K messages
- Deploy partition key fix sprint tiếp theo
- Bài học: monitor per-partition lag, không chỉ aggregate
```

---

### Template 2: Poison Message / DLQ

```markdown
## [Tên Story — e.g., "Poison Message Block Payment Queue"]

**S — Situation:**
RabbitMQ queue `payment-tasks` depth tăng từ 0 lên 50K trong 2 giờ.
Payment processing stop — orders không được charge.

**T — Task:**
Tôi lead investigation và restore payment flow.

**A — Action:**
1. Tôi inspect queue — message đầu tiên redeliver liên tục (delivery count = 100+)
2. Root cause: schema change — field `amount` đổi từ string sang number, consumer parse fail
3. Short-term: manual move message sang DLQ qua shovel plugin
4. Tôi deploy consumer fix với backward-compatible parsing
5. Long-term: implement `x-delivery-limit=3` + DLX routing + schema validation

**R — Result:**
- Queue drain trong 30 phút sau fix
- DLQ monitoring dashboard + alert
- Thêm contract test giữa producer/consumer schemas
```

---

### Template 3: Message Duplicate / Idempotency

```markdown
## [Tên Story — e.g., "Duplicate Charge từ At-Least-Once Delivery"]

**S — Situation:**
Sau deploy consumer mới, support nhận 200+ complaints về double charge trong 4 giờ.
Payment service consume từ Kafka topic `payment-events`.

**T — Task:**
Tôi investigate root cause và prevent recurrence.

**A — Action:**
1. Tôi trace correlation ID — cùng `eventId` xử lý 2 lần
2. Root cause: consumer commit offset TRƯỚC khi charge DB → crash → reprocess
3. Short-term: manual refund affected users (list từ audit log)
4. Fix: reorder — charge DB trong transaction, commit offset SAU success
5. Thêm `processed_events` table với unique constraint trên `eventId`

**R — Result:**
- Zero duplicate sau fix deploy
- Idempotency check thành standard cho mọi consumer
- Post-mortem shared với team — updated consumer checklist
```

---

### Template 4: Broker Outage / Failover

```markdown
## [Tên Story — e.g., "Kafka Broker Failure During Peak"]

**S — Situation:**
3-broker Kafka cluster, 1 broker disk failure lúc peak traffic.
Under-replicated partitions alert + produce latency spike.

**T — Task:**
Tôi on-call — ensure no data loss và restore cluster health.

**A — Action:**
1. Tôi verify `acks=all` + `min.insync.replicas=2` — producers không mất data
2. Controller elect new leaders từ ISR — automatic failover
3. Tôi replace failed broker node, trigger partition rebalance
4. Verify all partitions back to RF=3, ISR complete
5. Post-incident: disk monitoring alert threshold 70% → 60%

**R — Result:**
- Zero message loss (confirmed via audit)
- Downtime produce: ~2 phút latency spike, không outage hoàn toàn
- Runbook updated cho broker replacement
```

---

### Template 5: Outbox Pattern Implementation

```markdown
## [Tên Story — e.g., "Implement Outbox cho Order Service"]

**S — Situation:**
Order service gặp inconsistency — order saved nhưng event không publish (5–10 cases/tuần).
Dual-write problem giữa PostgreSQL và Kafka.

**T — Task:**
Tôi được giao design và implement reliable event publishing.

**A — Action:**
1. Tôi phân tích options: Outbox vs CDC vs Kafka transactions
2. Chọn Outbox + Debezium relay — phù hợp team skill và infra hiện có
3. Implement `outbox` table + migration existing publish logic
4. Setup Debezium connector với monitoring lag
5. Integration test: kill relay mid-flight — verify no loss, no duplicate

**R — Result:**
- Inconsistency cases: 0 sau 3 tháng
- Event publish latency P99: < 2s
- Pattern adopted làm standard cho 4 services khác
```

---

## Câu Chuyện Mẫu Theo Chủ Đề

### Mẫu Hoàn Chỉnh: Consumer Lag (Có Thể Adapt)

**S — Situation:**
Tại fintech startup, payment notification service consume từ Kafka topic `payment.completed` với 8 partitions, throughput ~500 msg/s peak. Thứ Hai sau long weekend, Datadog alert consumer lag vượt 500K messages — cao gấp 50 lần bình thường.

**T — Task:**
Tôi là senior backend engineer, primary owner của notification consumer. SLA yêu cầu 95% notifications gửi trong 5 phút — đang breach.

**A — Action:**
1. Tôi mở Grafana dashboard — lag đồng đều tất cả partitions, không phải hot partition
2. Check consumer logs — `TimeoutException` khi gọi SendGrid API (rate limit 429)
3. Root cause: marketing campaign gửi 10x traffic, email provider throttle
4. Short-term: tôi implement exponential backoff + circuit breaker pause consumption 5 phút
5. Tôi scale consumer từ 4 → 8 instances và negotiate temporary rate limit increase với SendGrid
6. Long-term: tách queue `transactional-email` và `marketing-email` với rate limit riêng

**R — Result:**
- Lag clear trong 4 giờ (từ 500K → 0)
- Circuit breaker + per-channel rate limiting deployed tuần sau
- SLA recovery: P95 notification latency về 2 phút
- Bài học: downstream dependency limits là bottleneck phổ biến — monitor external API errors cùng consumer lag

---

## Cách Viết STAR Story Của Bạn

### Bước 1: Brainstorm Incidents

Liệt kê 5–10 incidents từ kinh nghiệm:
- Production outages
- Performance issues
- Bugs bạn fix
- Architecture decisions
- Migrations

### Bước 2: Chọn 5 Stories Mạnh Nhất

Tiêu chí:
- Có **số liệu** (lag, downtime, throughput)
- Bạn có **vai trò rõ ràng** (không chỉ "tham gia")
- Có **technical depth** (root cause cụ thể)
- Có **bài học** và **prevention**

### Bước 3: Viết Theo Template

```markdown
## Story [N]: [Tên Ngắn]

**Keywords:** consumer lag, Kafka, idempotency (dùng để map câu hỏi)

**S:** [2–3 câu bối cảnh]
**T:** [1–2 câu nhiệm vụ của bạn]
**A:** [4–6 bullet points kỹ thuật]
**R:** [2–3 câu kết quả + số liệu + bài học]
```

### Bước 4: Practice

- Đọc to aloud 2–3 phút mỗi story
- Record và nghe lại — có lan man không?
- Mock interview với đồng nghiệp

### Điều Tránh

| Tránh | Nên |
| ----- | --- |
| "Chúng tôi đã fix" | "Tôi đã identify root cause và deploy fix" |
| Không có số liệu | "Lag giảm từ 500K xuống 0 trong 4 giờ" |
| Đổ lỗi team khác | Focus vào solution và collaboration |
| Quá dài (> 5 phút) | 2–3 phút, interviewer sẽ hỏi thêm nếu cần |

---

## Câu Hỏi Ngược Cho Interviewer

Sau phần behavioral, có thể hỏi:

1. "Team đang dùng Kafka hay RabbitMQ? Consumer lag monitoring setup thế nào?"
2. "Có incident response runbook cho messaging outages không?"
3. "Team có standard pattern cho idempotency và outbox không?"
4. "Làm thế nào team handle schema evolution cho events?"

---

**Cập Nhật Lần Cuối:** 2026-07-03
