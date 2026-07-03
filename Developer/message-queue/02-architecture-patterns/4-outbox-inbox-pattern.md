# Outbox & Inbox Pattern — Mẫu Hộp Thoại Ra & Hộp Thư Đến

> **Outbox Pattern (Mẫu Hộp Thoại Ra)** và **Inbox Pattern (Mẫu Hộp Thư Đến)** giải quyết **dual-write problem (vấn đề ghi kép)** — đảm bảo ghi database và publish message **nhất quán**, không mất event khi crash.

## Mục Lục

1. [Dual-Write Problem](#dual-write-problem)
2. [Outbox Pattern](#outbox-pattern)
3. [Outbox Relay Process](#outbox-relay-process)
4. [Inbox Pattern](#inbox-pattern)
5. [Transactional Outbox vs Change Data Capture](#transactional-outbox-vs-change-data-capture)
6. [Implementation Guide](#implementation-guide)
7. [Failure Scenarios](#failure-scenarios)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Dual-Write Problem

Khi service cần **vừa ghi DB vừa publish message** — hai operation không thể nằm trong cùng distributed transaction:

```
❌ ANTI-PATTERN — Dual Write:

BEGIN transaction
  INSERT INTO orders ...
COMMIT

await kafka.publish('order.created', ...)  ← Nếu crash ở đây?
                                           → Order có trong DB
                                           → Event KHÔNG được publish
```

**Ba failure mode:**

| Scenario | DB | Broker | Hậu Quả |
| -------- | -- | ------ | ------- |
| Crash sau DB commit, trước publish | ✓ | ✗ | **Mất event** — downstream không biết |
| Publish OK, DB rollback | ✗ | ✓ | **Ghost event** — event không có data |
| Publish OK, DB fail | ✗ | ✓ | Inconsistency |

```
Service                    Kafka
   │                         │
   ├── 1. DB write ──────────┤
   │                         │
   ├── 2. Publish ──────────►│  ← Không atomic!
   │                         │
   └── Crash giữa 1 và 2 ────┘
```

> Không thể dùng **2PC (Two-Phase Commit — Cam Kết Hai Pha)** xuyên DB và Kafka trong hầu hết production — quá chậm, fragile, không được Kafka khuyến nghị.

---

## Outbox Pattern

**Outbox Pattern** — ghi event vào bảng **outbox** trong **cùng DB transaction** với business data. Process riêng (relay) đọc outbox và publish lên broker.

```
✅ OUTBOX PATTERN:

BEGIN transaction
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (id, aggregate_type, event_type, payload, created_at)
    VALUES (uuid, 'Order', 'OrderCreated', '{...}', now());
COMMIT  ← Atomic — cả order và event cùng commit hoặc cùng rollback

Relay process (separate):
  SELECT * FROM outbox WHERE published_at IS NULL
  → publish to Kafka
  → UPDATE outbox SET published_at = now()
```

```
┌─────────────┐     same TX      ┌─────────────┐
│   orders    │ ◄──────────────► │   outbox    │
│   table     │                  │   table     │
└─────────────┘                  └──────┬──────┘
                                        │
                                   Relay Process
                                        │
                                        ▼
                                 ┌─────────────┐
                                 │   Kafka     │
                                 └─────────────┘
```

### Outbox Table Schema

```sql
CREATE TABLE outbox (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type  VARCHAR(100) NOT NULL,  -- 'Order', 'Payment'
  aggregate_id    VARCHAR(100) NOT NULL,  -- 'ORD-12345'
  event_type      VARCHAR(100) NOT NULL,  -- 'OrderCreated'
  payload         JSONB NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at    TIMESTAMPTZ,            -- NULL = chưa publish
  retry_count     INT DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at)
  WHERE published_at IS NULL;
```

---

## Outbox Relay Process

**Outbox Relay (Bộ Chuyển Tiếp Outbox)** — background process poll outbox và publish.

### Polling Relay

```javascript
async function relayLoop() {
  while (true) {
    const rows = await db.query(`
      SELECT * FROM outbox
      WHERE published_at IS NULL
      ORDER BY created_at
      LIMIT 100
      FOR UPDATE SKIP LOCKED
    `);

    for (const row of rows.rows) {
      try {
        await kafka.send({
          topic: mapEventTypeToTopic(row.event_type),
          key: row.aggregate_id,
          value: JSON.stringify({
            eventId: row.id,
            eventType: row.event_type,
            payload: row.payload,
            idempotencyKey: `${row.aggregate_id}-${row.event_type}`,
          }),
        });

        await db.query(
          'UPDATE outbox SET published_at = NOW() WHERE id = $1',
          [row.id]
        );
      } catch (err) {
        await db.query(
          'UPDATE outbox SET retry_count = retry_count + 1 WHERE id = $1',
          [row.id]
        );
        log.error('Relay failed', { outboxId: row.id, err });
      }
    }

    await sleep(1000); // poll interval
  }
}
```

### Relay Strategies

| Strategy | Mô Tả | Trade-off |
| -------- | ----- | --------- |
| **Polling** | SELECT unpublished, publish, UPDATE | Đơn giản, có latency poll interval |
| **CDC (Change Data Capture)** | Debezium read WAL → Kafka | Real-time, không poll — phức tạp hơn |
| **Transactional Log Tailing** | Listen DB binlog/WAL | Low latency, infra dependency |

### Ordering Guarantee

```
Outbox rows cho cùng aggregate_id → publish theo created_at order
Kafka partition key = aggregate_id → ordering per entity
```

---

## Inbox Pattern

**Inbox Pattern** — consumer lưu incoming message vào **inbox table** trước khi xử lý, đảm bảo không process duplicate.

```
Broker ──► Consumer
              │
              ▼
         BEGIN transaction
           INSERT inbox (message_id UNIQUE)  ← dedup
           Process business logic
           UPDATE inbox SET processed_at = NOW()
         COMMIT
```

### Inbox Table Schema

```sql
CREATE TABLE inbox (
  message_id     VARCHAR(255) PRIMARY KEY,
  event_type     VARCHAR(100) NOT NULL,
  payload        JSONB NOT NULL,
  received_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at   TIMESTAMPTZ,
  status         VARCHAR(20) DEFAULT 'PENDING'  -- PENDING, PROCESSED, FAILED
);
```

```javascript
async function consumeMessage(msg) {
  const messageId = msg.headers.eventId;

  await db.transaction(async (tx) => {
    try {
      await tx.query(
        'INSERT INTO inbox (message_id, event_type, payload) VALUES ($1, $2, $3)',
        [messageId, msg.eventType, msg.payload]
      );
    } catch (err) {
      if (err.code === '23505') { // unique violation = duplicate
        return; // already processed or in progress
      }
      throw err;
    }

    await processBusinessLogic(msg, tx);

    await tx.query(
      'UPDATE inbox SET processed_at = NOW(), status = $1 WHERE message_id = $2',
      ['PROCESSED', messageId]
    );
  });
}
```

### Outbox + Inbox Together

```
Service A                          Service B
┌──────────────┐                   ┌──────────────┐
│ orders       │                   │ inbox        │
│ outbox ──────┼── Kafka ─────────►│ business     │
└──────────────┘                   └──────────────┘

A: atomic write (order + outbox)
Relay: at-least-once publish
B: inbox dedup + idempotent handler
→ End-to-end reliable delivery
```

---

## Transactional Outbox vs Change Data Capture

| | Transactional Outbox (Polling) | CDC (Debezium) |
| --- | --- | --- |
| **CATEGORY** | App write outbox table, relay poll | DB WAL → Kafka Connect |
| **Latency** | Poll interval (1–5s) | Near real-time (ms) |
| **Complexity** | Thấp — app code | Cao — Debezium, connectors |
| **Event shape** | App control payload | Raw row change |
| **Phù hợp** | Hầu hết microservices | Event sourcing, analytics pipeline |

```
CDC flow:
PostgreSQL WAL ──► Debezium ──► Kafka topic (outbox table changes)
                                    │
                                    └── Không cần relay app code
```

---

## Implementation Guide

### Checklist Outbox

| Bước | Action |
| ---- | ------ |
| 1 | Tạo outbox table với index unpublished |
| 2 | Wrap business write + outbox insert trong transaction |
| 3 | Deploy relay process (HA — ít nhất 1 instance, leader election nếu nhiều) |
| 4 | `FOR UPDATE SKIP LOCKED` tránh relay duplicate |
| 5 | Monitor: unpublished count, relay lag, retry_count |
| 6 | Dead letter: rows retry_count > N → alert + manual review |

### Checklist Inbox

| Bước | Action |
| ---- | ------ |
| 1 | Tạo inbox table với message_id UNIQUE |
| 2 | Insert inbox + process trong same transaction |
| 3 | Handle unique violation = skip duplicate |
| 4 | Cleanup processed inbox rows (retention policy) |
| 5 | Failed messages → status FAILED + DLQ/retry |

### Libraries & Tools

| Tool | Mô Tả |
| ---- | ----- |
| **Debezium** | CDC connector cho PostgreSQL, MySQL |
| **MassTransit** (.NET) | Outbox built-in |
| **Eventuate Tram** | Java — outbox + CDC |
| **Prisma + custom** | Node.js — manual outbox table |

---

## Failure Scenarios

### Relay Crash Sau Publish, Trước UPDATE published_at

```
→ Message published 2 lần (relay retry)
→ Consumer PHẢI idempotent (inbox/dedup key)
→ At-least-once publish là expected behavior
```

### Relay Crash Trước Publish

```
→ Row vẫn unpublished
→ Relay restart, poll lại → publish
→ No message loss ✓
```

### Multiple Relay Instances

```
→ Dùng FOR UPDATE SKIP LOCKED
→ Hoặc leader election (chỉ 1 active relay)
→ Không publish duplicate nếu mark published_at atomic
```

### Outbox Table Growth

```
→ Archive/delete rows published_at > 7 days ago
→ Partition by created_at
→ Monitor table size
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Outbox Pattern giải quyết vấn đề gì?

**Đáp án mẫu:** **Dual-write problem** — không thể atomic ghi DB và publish Kafka. Outbox ghi event vào DB trong cùng transaction với business data, relay publish sau — đảm bảo **không mất event** khi crash sau DB commit.

### Câu 2: Outbox đảm bảo exactly-once publish?

**Đáp án mẫu:** **Không** — relay có thể publish duplicate nếu crash sau Kafka ack nhưng trước UPDATE published_at. Outbox đảm bảo **at-least-once publish**. Exactly-once end-to-end cần **outbox + idempotent consumer/inbox** — "effective exactly-once".

### Câu 3: Polling relay vs CDC — chọn nào?

**Đáp án mẫu:** **Polling:** đơn giản, đủ cho hầu hết cases, latency vài giây chấp nhận được. **CDC (Debezium):** near real-time, ít app code relay, phù hợp high volume hoặc đã có Kafka Connect infrastructure. Trade-off: CDC phức tạp ops hơn.

### Câu 4: Inbox Pattern khác dedup table?

**Đáp án mẫu:** **Inbox** lưu full message payload + processing status — phù hợp audit, retry failed messages, transactional processing. **Dedup table** chỉ lưu idempotency key — nhẹ hơn, phù hợp simple skip-duplicate. Inbox là superset — có thể replay từ inbox nếu cần.

### Câu 5: Nhiều relay instance chạy cùng lúc — sao không duplicate publish?

**Đáp án mẫu:** Dùng **`SELECT ... FOR UPDATE SKIP LOCKED`** — mỗi instance lock rows khác nhau. Hoặc **single leader relay** với failover. Mark `published_at` **sau** Kafka ack thành công. Duplicate vẫn có thể xảy ra at edge cases — consumer phải idempotent.

---

**Xem tiếp:** [3-saga-pattern.md](./3-saga-pattern.md) — distributed transactions qua nhiều services.
