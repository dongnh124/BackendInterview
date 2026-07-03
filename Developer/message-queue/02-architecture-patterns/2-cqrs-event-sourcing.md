# CQRS & Event Sourcing — Tách Đọc/Ghi & Lưu Trữ Sự Kiện

> **CQRS (Command Query Responsibility Segregation — Tách Trách Nhiệm Đọc/Ghi)** và **Event Sourcing (Lưu Trữ Sự Kiện)** là pattern nâng cao thường dùng cùng Event-Driven Architecture — phù hợp khi cần scale read/write độc lập, audit trail, hoặc temporal queries (truy vấn theo thời gian).

## Mục Lục

1. [CQRS Là Gì?](#cqrs-là-gì)
2. [Event Sourcing Là Gì?](#event-sourcing-là-gì)
3. [CQRS + Event Sourcing Together](#cqrs--event-sourcing-together)
4. [Projections & Read Models](#projections--read-models)
5. [Snapshots](#snapshots)
6. [Event Store](#event-store)
7. [Trade-offs & Khi Nào Dùng](#trade-offs--khi-nào-dùng)
8. [CQRS/Event Sourcing vs CRUD](#cqrsevent-sourcing-vs-crud)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CQRS Là Gì?

**CQRS (Command Query Responsibility Segregation — Tách Trách Nhiệm Đọc/Ghi)** tách **write model (mô hình ghi)** và **read model (mô hình đọc)** thành hai path riêng biệt.

```
                    ┌─────────────────────────────────┐
                    │           CLIENT                 │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
            ┌───────────────┐               ┌───────────────┐
            │   COMMAND     │               │    QUERY      │
            │   (Write)     │               │    (Read)     │
            │               │               │               │
            │ CreateOrder   │               │ GetOrderList  │
            │ CancelOrder   │               │ GetOrderById  │
            └───────┬───────┘               └───────┬───────┘
                    │                               │
                    ▼                               ▼
            ┌───────────────┐               ┌───────────────┐
            │  Write DB     │    events     │   Read DB     │
            │  (normalized) │ ────────────► │  (denormalized│
            │               │  projection   │   optimized)  │
            └───────────────┘               └───────────────┘
```

### Command Side (Phía Lệnh)

- Nhận **commands** — `CreateOrder`, `UpdateInventory`
- Validate business rules
- Ghi vào write model / event store
- Publish domain events

### Query Side (Phía Truy Vấn)

- Nhận **queries** — `GetOrdersByCustomer`, `SearchProducts`
- Đọc từ **read model** tối ưu cho query pattern
- **Không** ghi trực tiếp — chỉ update qua projection từ events

| | Write Model | Read Model |
| --- | --- | --- |
| **Schema** | Normalized, domain-focused | Denormalized, query-focused |
| **Database** | PostgreSQL (OLTP) | Elasticsearch, Redis, read replica |
| **Optimize for** | Consistency, business rules | Query speed, aggregations |
| **Scale** | Vertical + sharding | Horizontal read replicas |

---

## Event Sourcing Là Gì?

**Event Sourcing (Lưu Trữ Sự Kiện)** — lưu state của aggregate dưới dạng **sequence of events (chuỗi sự kiện)**, không lưu current state trực tiếp.

```
CRUD truyền thống:
  orders table: { id: 123, status: "SHIPPED", total: 100000 }
  → chỉ biết state hiện tại, mất lịch sử

Event Sourcing:
  events cho order-123:
    1. OrderCreated      { items, total: 100000 }
    2. PaymentReceived   { amount: 100000 }
    3. OrderShipped      { trackingNumber: "VN123" }
  → rebuild state bằng cách replay events
  → current state = SHIPPED (derived)
```

### Aggregate & Event Stream

```
┌─────────────────────────────────────────┐
│  Order Aggregate (order-123)             │
│                                         │
│  Event Stream:                          │
│  ┌─────────────────────────────────┐   │
│  │ evt-1: OrderCreated             │   │
│  │ evt-2: ItemAdded                │   │
│  │ evt-3: PaymentReceived          │   │
│  │ evt-4: OrderShipped             │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Current State (rebuilt):               │
│  status=SHIPPED, total=100000           │
└─────────────────────────────────────────┘
```

### Rebuild State (Replay)

```javascript
function rebuildOrder(events) {
  let order = { status: null, items: [], total: 0 };

  for (const event of events) {
    switch (event.type) {
      case 'OrderCreated':
        order = { ...order, status: 'CREATED', items: event.items, total: event.total };
        break;
      case 'PaymentReceived':
        order = { ...order, status: 'PAID' };
        break;
      case 'OrderShipped':
        order = { ...order, status: 'SHIPPED', tracking: event.trackingNumber };
        break;
    }
  }
  return order;
}
```

---

## CQRS + Event Sourcing Together

Hai pattern **thường đi cùng nhau** nhưng **không bắt buộc**:

```
┌──────────┐  Command   ┌──────────────┐  Events   ┌──────────────┐
│  Client  │ ─────────► │ Write Side   │ ────────► │  Event Store │
└──────────┘            │ (aggregate)  │           │  (append-only│
       │                └──────────────┘           │   log)       │
       │ Query                                      └──────┬───────┘
       ▼                                                   │
┌──────────────┐                                    Projections
│  Read Side   │ ◄─────────────────────────────────────────┘
│  (optimized) │
└──────────────┘
```

| Combination | Mô Tả |
| ----------- | ----- |
| **CQRS only** | Tách read/write DB, write vẫn CRUD |
| **Event Sourcing only** | Event store làm source of truth, read từ replay |
| **CQRS + ES** | Write → event store, read → projections — pattern mạnh nhất, phức tạp nhất |

---

## Projections & Read Models

**Projection (Chiếu)** — process consume events và build **read model** tối ưu cho queries.

```
Event Store                    Projections
┌─────────────┐               ┌─────────────────────┐
│ OrderCreated│──────────────►│ OrderListView       │ → PostgreSQL read replica
│ OrderShipped│──────────────►│ OrderDetailView     │ → Redis cache
│ OrderCreated│──────────────►│ CustomerOrderStats  │ → Elasticsearch
└─────────────┘               └─────────────────────┘
```

### Projection Types

| Type | Mô Tả | Consistency |
| ---- | ----- | ----------- |
| **Synchronous** | Update read model trong cùng transaction | Strong (hiếm) |
| **Asynchronous** | Consumer event → update read DB | Eventual |
| **Live projection** | Query event store + cache | On-demand |

```javascript
// Async projection consumer
async function onOrderCreated(event) {
  await readDb.query(`
    INSERT INTO order_list_view (order_id, customer_id, status, total, created_at)
    VALUES ($1, $2, 'CREATED', $3, $4)
  `, [event.orderId, event.customerId, event.total, event.timestamp]);
}

async function onOrderShipped(event) {
  await readDb.query(`
    UPDATE order_list_view SET status = 'SHIPPED' WHERE order_id = $1
  `, [event.orderId]);
}
```

### Multiple Read Models

Một event stream → **nhiều projections** phục vụ use cases khác nhau:

```
OrderEvents ──► OrderListProjection (UI list page)
            ──► OrderAnalyticsProjection (BI dashboard)
            ──► InventoryProjection (stock levels)
            ──► NotificationProjection (email triggers)
```

---

## Snapshots

**Snapshot (Ảnh Chụp Trạng Thái)** — lưu current aggregate state định kỳ để tránh replay quá nhiều events.

```
Events: [1, 2, 3, ... 1000]  → replay 1000 events chậm!

With snapshot at event 900:
  Load snapshot(900) + replay events [901..1000]
  → chỉ replay 100 events
```

```sql
CREATE TABLE aggregate_snapshots (
  aggregate_id   VARCHAR(100) NOT NULL,
  aggregate_type VARCHAR(100) NOT NULL,
  version        INT NOT NULL,           -- event version at snapshot
  state          JSONB NOT NULL,
  created_at     TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (aggregate_id, version)
);
```

**Strategy:** snapshot mỗi N events (100–1000) hoặc theo schedule.

---

## Event Store

**Event Store (Kho Sự Kiện)** — append-only storage cho domain events.

### Requirements

| Requirement | Mô Tả |
| ----------- | ----- |
| **Append-only** | Events không sửa/xóa (chỉ soft delete/compensating event) |
| **Ordering** | Events per aggregate ordered by version |
| **Optimistic concurrency** | Version check khi append — chống lost update |
| **Stream per aggregate** | `order-123` → stream riêng |

```sql
CREATE TABLE events (
  event_id       UUID PRIMARY KEY,
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id   VARCHAR(100) NOT NULL,
  event_type     VARCHAR(100) NOT NULL,
  payload        JSONB NOT NULL,
  version        INT NOT NULL,  -- per aggregate sequence
  created_at     TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (aggregate_id, version)
);
```

```javascript
async function appendEvent(aggregateId, expectedVersion, event) {
  const result = await db.query(`
    INSERT INTO events (event_id, aggregate_id, event_type, payload, version)
    SELECT $1, $2, $3, $4, COALESCE(MAX(version), 0) + 1
    FROM events WHERE aggregate_id = $2
    HAVING COALESCE(MAX(version), 0) = $5
  `, [uuid(), aggregateId, event.type, event.payload, expectedVersion]);

  if (result.rowCount === 0) {
    throw new ConcurrencyError('Aggregate version mismatch');
  }
}
```

### Event Store Options

| Option | Mô Tả |
| ------ | ----- |
| **Custom PostgreSQL table** | Đơn giản, đủ cho nhiều cases |
| **EventStoreDB** | Purpose-built event store |
| **Apache Kafka** | Event log as store — retention-based |
| **Axon Server** | Java ecosystem |

---

## Trade-offs & Khi Nào Dùng

### Ưu Điểm

| Ưu Điểm | Mô Tả |
| ------- | ----- |
| **Audit trail** | Full history — ai làm gì, khi nào |
| **Temporal queries** | "Order trông thế nào lúc 10:00?" — replay đến thời điểm |
| **Debug & replay** | Rebuild read models, test projections |
| **Scale read/write** | Independent scaling |
| **Flexibility** | Thêm read model mới từ existing events |

### Nhược Điểm

| Nhược Điểm | Mô Tả |
| ---------- | ----- |
| **Complexity** | Steep learning curve, nhiều moving parts |
| **Eventual consistency** | Read model lag behind write |
| **Schema evolution** | Event schema changes khó hơn CRUD |
| **Storage growth** | Events accumulate — cần retention/archival |
| **Query complexity** | Ad-hoc queries khó — cần pre-built projections |

### Khi Nào Dùng

| Dùng | Không Dùng |
| ---- | ---------- |
| Audit/compliance bắt buộc | Simple CRUD app |
| Complex domain với rich history | Team nhỏ, deadline gấp |
| Read/write scale khác nhau nhiều | Không cần event history |
| Cần rebuild state / temporal query | Strong consistency everywhere |
| Event-driven đã mature trong org | Prototype/MVP |

---

## CQRS/Event Sourcing vs CRUD

| Tiêu Chí | CRUD | CQRS | Event Sourcing | CQRS + ES |
| -------- | ---- | ---- | -------------- | --------- |
| **Complexity** | Thấp | Trung bình | Cao | Rất cao |
| **History** | Mất (hoặc audit table) | Tùy | Full event log | Full event log |
| **Read scale** | Replica | Dedicated read DB | Projections | Optimized projections |
| **Consistency** | Strong | Eventual (read) | Eventual | Eventual |
| **Learning curve** | Thấp | Trung bình | Cao | Rất cao |
| **Phù hợp** | 80% apps | High read load | Audit, finance | Enterprise complex domain |

```
Decision tree:

Cần full audit trail + temporal queries?
  Có → Event Sourcing candidate
  Không ↓

Read load >> Write load?
  Có → CQRS candidate
  Không ↓

CRUD đủ — đừng over-engineer
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: CQRS giải quyết vấn đề gì?

**Đáp án mẫu:** Tách **read và write concerns** — write model optimize cho business rules và consistency; read model optimize cho query patterns (denormalized, indexed, cached). Scale read replicas độc lập không ảnh hưởng write path. Trade-off: **eventual consistency** giữa read và write.

### Câu 2: Event Sourcing khác audit log?

**Đáp án mẫu:** **Audit log** ghi lại changes nhưng **source of truth vẫn là current state** (CRUD table). **Event Sourcing** — events **là** source of truth, current state **derived** từ replay. Audit log optional; event store bắt buộc rebuild state.

### Câu 3: Làm sao xử lý schema evolution trong Event Sourcing?

**Đáp án mẫu:** **Upcasting** — transform old events sang new format khi replay. **Version trong event type** — `OrderCreated.v1`, `OrderCreated.v2`. **Additive changes only** — thêm field optional. Projections handle multiple versions. Không bao giờ sửa events đã store — chỉ append compensating events.

### Câu 4: Read model stale — user thấy data cũ?

**Đáp án mẫu:** **Eventual consistency** by design — projection lag thường ms–seconds. Mitigate: **read-your-writes** (route query về write side ngay sau command), **UI optimistic update**, **version number** cho client detect stale. Monitor projection lag metric.

### Câu 5: CQRS/Event Sourcing có bắt buộc dùng Kafka?

**Đáp án mẫu:** **Không.** Event store có thể là PostgreSQL table, EventStoreDB. Kafka phù hợp khi cần **high throughput event streaming** và nhiều consumers/projections. CQRS read models có thể update qua Kafka consumer hoặc direct DB projection. Chọn theo scale và infra hiện có.

---

**Quay lại:** [README.md](./README.md) — tổng quan Architecture Patterns.
