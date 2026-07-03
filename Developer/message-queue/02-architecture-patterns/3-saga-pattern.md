# Saga Pattern — Mẫu Saga Cho Giao Dịch Phân Tán

> **Saga Pattern (Mẫu Saga)** quản lý **distributed transaction (giao dịch phân tán)** qua nhiều microservices bằng chuỗi **local transactions (giao dịch cục bộ)** và **compensating transactions (giao dịch bù trừ)** — thay thế 2PC (Two-Phase Commit — Cam Kết Hai Pha) trong hệ thống event-driven.

## Mục Lục

1. [Vấn Đề Distributed Transaction](#vấn-đề-distributed-transaction)
2. [Saga Là Gì?](#saga-là-gì)
3. [Choreography vs Orchestration](#choreography-vs-orchestration)
4. [Compensation — Giao Dịch Bù Trừ](#compensation--giao-dịch-bù-trừ)
5. [Saga State Machine](#saga-state-machine)
6. [Failure Handling](#failure-handling)
7. [Implementation Example](#implementation-example)
8. [Saga vs 2PC vs TCC](#saga-vs-2pc-vs-tcc)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Distributed Transaction

Order flow điển hình cần cập nhật **3 services**:

```
Create Order → Reserve Inventory → Process Payment → Confirm Order
     │                │                  │
  Order DB        Inventory DB        Payment DB
```

**Không thể** dùng single ACID transaction xuyên 3 database:

```
❌ BEGIN GLOBAL TRANSACTION  -- không tồn tại thực tế
     INSERT orders ...
     UPDATE inventory ...
     CHARGE payment ...
   COMMIT
```

**Tại sao không 2PC?**

| Vấn Đề 2PC | Hậu Quả |
| ---------- | ------- |
| **Blocking** | Coordinator lock resources đến khi all commit |
| **Single point of failure** | Coordinator crash → stuck |
| **Latency** | Round-trip nhiều phase |
| **Not supported** | Kafka, hầu hết message brokers không support XA |

---

## Saga Là Gì?

**Saga** — chuỗi **local transactions**, mỗi bước publish event trigger bước tiếp theo. Nếu bước nào **fail** → chạy **compensating transactions** undo các bước trước.

```
Happy path:
  T1: CreateOrder     → OrderCreated
  T2: ReserveStock    → StockReserved
  T3: ChargePayment   → PaymentCompleted
  T4: ConfirmOrder    → OrderConfirmed

Failure at T3 (payment fail):
  C2: ReleaseStock    ← compensate T2
  C1: CancelOrder     ← compensate T1
```

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Step 1  │───►│ Step 2  │───►│ Step 3  │───►│ Step 4  │
│ Order   │    │Inventory│    │ Payment │    │ Confirm │
└─────────┘    └─────────┘    └────┬────┘    └─────────┘
                                   │ FAIL
                              ┌────▼────┐
                              │ Compensate
                              │ C2 → C1
                              └─────────┘
```

**Đặc điểm:**

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **No global lock** | Mỗi service own transaction |
| **Eventual consistency** | Không atomic toàn flow |
| **Compensation** | Semantic undo, không phải rollback DB |
| **Visible intermediate state** | Order có thể "PENDING" trong lúc saga chạy |

---

## Choreography vs Orchestration

Hai cách implement saga:

### Choreography (Điệu Múa — Phi Tập Trung)

Mỗi service **listen event** và **publish event** tiếp theo — decentralized, không có coordinator.

```
Order Service:
  CreateOrder → publish OrderCreated

Inventory Service:
  listen OrderCreated → ReserveStock → publish StockReserved
  listen PaymentFailed → ReleaseStock (compensate)

Payment Service:
  listen StockReserved → ChargePayment → publish PaymentCompleted / PaymentFailed

Order Service:
  listen PaymentCompleted → ConfirmOrder
  listen PaymentFailed → CancelOrder (compensate)
```

```
OrderSvc ──OrderCreated──► InventorySvc ──StockReserved──► PaymentSvc
    ▲                            │                              │
    │                            │ PaymentFailed                │
    └──────── CancelOrder ◄──────┴──────────────────────────────┘
              ReleaseStock ◄─────┘
```

| Ưu | Nhược |
| --- | ----- |
| Simple khi ít services | Khó theo dõi flow khi phức tạp |
| No single point of failure | Cyclic dependencies risk |
| Truly decoupled | Khó debug — event chain dài |
| Phù hợp 2–4 bước | Thay đổi flow = sửa nhiều services |

### Orchestration (Điều Phối — Tập Trung)

**Saga Orchestrator (Bộ Điều Phối Saga)** điều khiển flow — gửi command cho từng service, nhận response/event.

```
                    ┌─────────────────────┐
                    │  Saga Orchestrator  │
                    │  (state machine)    │
                    └──────────┬──────────┘
           command           │           command
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  Order   │  │Inventory │  │ Payment  │
        │ Service  │  │ Service  │  │ Service  │
        └──────────┘  └──────────┘  └──────────┘
              │              │              │
              └──────────────┴──────────────┘
                        reply events
```

```javascript
// Orchestrator state machine (pseudo)
const sagaSteps = {
  START: { action: 'CreateOrder', onSuccess: 'RESERVE_STOCK', onFail: 'END' },
  RESERVE_STOCK: { action: 'ReserveStock', onSuccess: 'CHARGE_PAYMENT', onFail: 'CANCEL_ORDER' },
  CHARGE_PAYMENT: { action: 'ChargePayment', onSuccess: 'CONFIRM_ORDER', onFail: 'RELEASE_STOCK' },
  RELEASE_STOCK: { action: 'ReleaseStock', onSuccess: 'CANCEL_ORDER', onFail: 'MANUAL_INTERVENTION' },
  CANCEL_ORDER: { action: 'CancelOrder', onSuccess: 'END', onFail: 'MANUAL_INTERVENTION' },
  CONFIRM_ORDER: { action: 'ConfirmOrder', onSuccess: 'END', onFail: 'MANUAL_INTERVENTION' },
};
```

| Ưu | Nhược |
| --- | ----- |
| Flow rõ ràng, dễ visualize | Orchestrator = dependency |
| Dễ timeout, retry, monitoring | Cần HA cho orchestrator |
| Thay đổi flow tập trung | Thêm component maintain |
| Phù hợp flow phức tạp | Risk over-centralization |

### Decision Guide

```
                    ┌─────────────────────────┐
                    │ Bao nhiêu bước?         │
                    └───────────┬─────────────┘
                          ≤3    │    >3 hoặc phức tạp
                    ┌───────────┴───────────┐
                    ▼                       ▼
              Choreography            Orchestration
              (event chain)           (state machine)
                    │
                    ▼
              Flow có thay đổi thường xuyên?
                    │
              Có ───┴─── Không
              ▼           ▼
        Orchestration   Choreography OK
```

---

## Compensation — Giao Dịch Bù Trừ

**Compensating Transaction (Giao Dịch Bù Trừ)** — semantic undo của business operation đã commit.

| Forward Action | Compensation |
| -------------- | ------------ |
| CreateOrder | CancelOrder |
| ReserveStock | ReleaseStock |
| ChargePayment | RefundPayment |
| SendNotification | SendCancellationNotice (không "undo" được — chỉ inform) |

**Quan trọng:** Compensation **không phải** database ROLLBACK — forward transaction đã commit. Compensation là **business operation mới** revert effect.

```javascript
// Forward
async function reserveStock(orderId, items) {
  await db.query(`
    UPDATE inventory SET reserved = reserved + $1
    WHERE sku = $2 AND available >= $1
  `, [qty, sku]);
}

// Compensation
async function releaseStock(orderId, items) {
  await db.query(`
    UPDATE inventory SET reserved = reserved - $1
    WHERE sku = $2
  `, [qty, sku]);
}
```

### Compensation Challenges

| Challenge | Giải Pháp |
| --------- | ---------- |
| **Non-reversible actions** | Email đã gửi — gửi correction email |
| **Compensation fail** | Retry + alert + manual intervention queue |
| **Partial compensation** | Idempotent compensate — check state trước |
| **Nested saga** | Mỗi sub-saga own compensation chain |

---

## Saga State Machine

Orchestrator lưu **saga instance state** để recover sau crash:

```sql
CREATE TABLE saga_instances (
  saga_id        UUID PRIMARY KEY,
  saga_type      VARCHAR(100) NOT NULL,     -- 'CreateOrderSaga'
  current_step   VARCHAR(50) NOT NULL,
  status         VARCHAR(20) NOT NULL,      -- RUNNING, COMPLETED, COMPENSATING, FAILED
  payload        JSONB NOT NULL,
  created_at     TIMESTAMPTZ DEFAULT NOW(),
  updated_at     TIMESTAMPTZ DEFAULT NOW()
);
```

```
States:
  RUNNING      → đang execute forward steps
  COMPENSATING → đang chạy compensation chain
  COMPLETED    → saga thành công
  FAILED       → compensation cũng fail — cần manual
```

---

## Failure Handling

### Timeout

```
Orchestrator gửi ReserveStock → không response trong 30s
→ Mark step FAILED
→ Trigger compensation chain
```

### Duplicate Events

```
PaymentCompleted arrive 2 lần
→ Orchestrator check saga state — nếu đã COMPLETED → ignore (idempotent)
→ Xem 5-idempotency-dedup.md
```

### Out-of-Order Events

```
StockReserved arrive trước OrderCreated (unlikely với partition key)
→ Dùng correlationId + saga state validation
→ Reject hoặc buffer until prerequisite met
```

### Monitoring

| Metric | Ý Nghĩa |
| ------ | ------- |
| `saga_completed_total` | Success rate |
| `saga_compensating_total` | Compensation frequency |
| `saga_failed_total` | Need manual intervention |
| `saga_duration_seconds` | End-to-end latency |

---

## Implementation Example

### Choreography — Order Flow

```javascript
// Inventory Service consumer
async function onOrderCreated(event) {
  try {
    await reserveStock(event.orderId, event.items);
    await publish('StockReserved', { orderId: event.orderId, ... });
  } catch (err) {
    await publish('StockReservationFailed', { orderId: event.orderId });
  }
}

async function onPaymentFailed(event) {
  await releaseStock(event.orderId); // compensate
}
```

### Orchestration — With State Persistence

```javascript
async function handleSagaEvent(sagaId, eventType, payload) {
  const saga = await loadSaga(sagaId);
  const step = sagaSteps[saga.current_step];

  if (eventType === step.successEvent) {
    const nextStep = step.onSuccess;
    if (nextStep === 'END') {
      await completeSaga(sagaId);
    } else {
      await updateSagaStep(sagaId, nextStep);
      await sendCommand(sagaSteps[nextStep].action, payload);
    }
  } else if (eventType === step.failEvent) {
    await startCompensation(sagaId, step.onFail);
  }
}
```

---

## Saga vs 2PC vs TCC

| | Saga | 2PC | TCC (Try-Confirm-Cancel) |
| --- | --- | --- | --- |
| **Consistency** | Eventual | Strong | Strong (business level) |
| **Blocking** | No | Yes | No (reserve phase) |
| **Complexity** | Medium | High | High |
| **Compensation** | Semantic undo | Automatic rollback | Cancel reserved resources |
| **Phù hợp** | Long-running business flows | Short DB transactions | Payment, inventory reserve |
| **Messaging fit** | Excellent | Poor | Good |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Saga giải quyết vấn đề gì?

**Đáp án mẫu:** **Distributed transaction** across microservices — mỗi service có DB riêng, không global ACID. Saga dùng chuỗi local transactions + compensation khi fail — trade eventual consistency cho availability và scalability.

### Câu 2: Choreography vs Orchestration — khi nào dùng?

**Đáp án mẫu:** **Choreography:** 2–4 bước, team nhỏ, flow ổn định — mỗi service react event. **Orchestration:** flow phức tạp, nhiều bước, cần timeout/retry/monitoring tập trung, flow thay đổi thường xuyên. Production phức tạp thường chọn orchestration.

### Câu 3: Compensation khác rollback?

**Đáp án mẫu:** **Rollback** undo trong cùng DB transaction — chưa commit. **Compensation** là business operation **sau khi đã commit** — semantic undo (CancelOrder, RefundPayment). Không phải lúc nào cũng perfect inverse — email đã gửi không "unsend" được.

### Câu 4: Saga fail giữa chừng — user thấy gì?

**Đáp án mẫu:** **Intermediate states** visible — order status "PENDING", "PAYMENT_PROCESSING". UX cần reflect: polling status, WebSocket update, email khi fail. Saga orchestrator nên expose status API. Compensation xong → status "CANCELLED" + notification.

### Câu 5: Làm sao đảm bảo saga step không chạy 2 lần?

**Đáp án mẫu:** **Idempotency** ở mỗi service step — dedup key per sagaId + step. Orchestrator check saga state trước transition. **Inbox pattern** cho incoming commands. Local transaction conditional update (`WHERE status = 'PENDING'`).

---

**Xem tiếp:** [2-cqrs-event-sourcing.md](./2-cqrs-event-sourcing.md) — advanced pattern cho read/write separation và audit trail.
