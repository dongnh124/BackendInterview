# Message Queue Basics — Nền Tảng Hàng Đợi Tin Nhắn

> Hiểu Message Queue (Hàng Đợi Tin Nhắn), Event Broker (Broker Sự Kiện), các thành phần cốt lõi, use case thực tế, và khi nào chọn async messaging thay vì synchronous API call.

## Mục Lục

1. [Message Queue Là Gì?](#message-queue-là-gì)
2. [Event Broker Là Gì?](#event-broker-là-gì)
3. [MQ vs Event Broker](#mq-vs-event-broker)
4. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
5. [Sync vs Async Communication](#sync-vs-async-communication)
6. [Use Cases Thực Tế](#use-cases-thực-tế)
7. [Thuật Ngữ Quan Trọng](#thuật-ngữ-quan-trọng)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Message Queue Là Gì?

**Message Queue (MQ — Hàng Đợi Tin Nhắn)** là middleware (phần mềm trung gian) cho phép các ứng dụng giao tiếp **bất đồng bộ (asynchronously — Không Đồng Thời)** thông qua việc gửi và nhận **message (tin nhắn)**.

```
Producer ──► [ Queue ] ──► Consumer
              ▲
              │ Message được lưu tạm
              │ cho đến khi consumer xử lý
```

**Đặc điểm chính:**

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Decoupling (Tách Rời)** | Producer không cần biết consumer đang chạy hay không |
| **Buffering (Đệm)** | Broker giữ message khi consumer offline hoặc chậm |
| **Load Leveling (Cân Bằng Tải)** | Hấp thụ traffic spike, consumer xử lý dần |
| **Durability (Bền Vững)** | Message persist (lưu bền) trên disk — không mất khi restart |

**Ví dụ đời thường:** Hàng đợi tại quầy — khách (producer) lấy số, nhân viên (consumer) gọi số xử lý. Khách không cần đứng chờ trực tiếp trước mặt nhân viên.

---

## Event Broker Là Gì?

**Event Broker (Broker Sự Kiện)** là hệ thống messaging lưu trữ **event (sự kiện)** dưới dạng **append-only log (nhật ký chỉ ghi thêm)** — message không bị xóa ngay sau khi consume.

```
Producer ──► [ Event Log: e1, e2, e3, e4, ... ]
                    │         │         │
                    ▼         ▼         ▼
              Consumer A  Consumer B  Consumer C
              (analytics) (audit)     (notification)
```

**Đặc điểm chính:**

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Event Log** | Lưu lịch sử sự kiện — có thể **replay (phát lại)** |
| **Multiple Consumers** | Nhiều consumer group đọc cùng stream độc lập |
| **High Throughput (Thông Lượng Cao)** | Tối ưu cho hàng triệu event/giây |
| **Retention (Giữ Lại)** | Event được giữ theo thời gian hoặc dung lượng |

**Ví dụ:** Apache Kafka, Amazon Kinesis, Apache Pulsar.

---

## MQ vs Event Broker

| Tiêu Chí | Message Queue | Event Broker |
| -------- | ------------- | ------------ |
| **Mục đích** | Phân phối task/job | Stream event, event sourcing |
| **Sau khi consume** | Message thường bị xóa | Event giữ trong log (theo retention) |
| **Số consumer** | Thường 1 consumer/queue (competing consumers) | Nhiều consumer group độc lập |
| **Replay** | Hạn chế hoặc không có | Hỗ trợ replay từ offset bất kỳ |
| **Ordering** | FIFO trong queue | Ordering trong partition |
| **Ví dụ** | RabbitMQ, Amazon SQS | Apache Kafka, Pulsar |
| **Phù hợp** | Task queue, RPC async, workflow | Analytics, CDC, event-driven microservices |

```
                    ┌─────────────────────────────────┐
                    │     BẠN CẦN GÌ?                  │
                    └────────────┬────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     "Xử lý job 1 lần"   "Nhiều service đọc    "Replay lịch sử
      rồi xóa"            cùng event"            event"
              │                  │                  │
              ▼                  ▼                  ▼
         Message Queue      Event Broker        Event Broker
         (RabbitMQ, SQS)    (Kafka)             (Kafka + retention)
```

> **Lưu ý thực tế:** Ranh giới ngày càng mờ — RabbitMQ có streams, Kafka có compacted topics. Quan trọng là hiểu **mental model (mô hình tư duy)** phù hợp use case.

---

## Các Thành Phần Cốt Lõi

### Producer (Nhà Sản Xuất)

Ứng dụng hoặc service **gửi message** vào broker.

```javascript
// Ví dụ pseudo-code — RabbitMQ
await channel.sendToQueue('orders', Buffer.from(JSON.stringify({
  orderId: 'ORD-001',
  amount: 150000,
})));
```

### Consumer (Bên Tiêu Thụ)

Ứng dụng **nhận và xử lý message** từ broker.

```javascript
await channel.consume('orders', async (msg) => {
  const order = JSON.parse(msg.content.toString());
  await processOrder(order);
  channel.ack(msg); // Acknowledge — xác nhận đã xử lý
});
```

### Broker (Trung Gian)

Phần mềm trung gian quản lý routing (định tuyến), persistence (lưu trữ), và delivery (giao hàng).

| Broker | Đặc Trưng |
| ------ | --------- |
| **Apache Kafka** | Distributed commit log, partition, consumer group |
| **RabbitMQ** | AMQP protocol, exchange + queue + binding |
| **Amazon SQS** | Managed queue, serverless, visibility timeout |
| **Redis Streams** | In-memory, ultra-low latency |

### Message (Tin Nhắn)

Đơn vị dữ liệu truyền qua broker, thường gồm:

```
┌────────────────────────────────────────┐
│ Message                                │
├────────────────────────────────────────┤
│ Header: metadata (correlationId, type) │
│ Body: payload (JSON, Avro, Protobuf)   │
│ Key: routing/partition key (optional)  │
│ Timestamp: thời điểm tạo               │
└────────────────────────────────────────┘
```

### Queue vs Topic

| Khái Niệm | Mô Tả | Broker Thường Dùng |
| --------- | ----- | ------------------ |
| **Queue (Hàng Đợi)** | Message đi đến **một** consumer trong nhóm | RabbitMQ, SQS |
| **Topic (Chủ Đề)** | Message broadcast đến **nhiều** subscriber | Kafka, SNS |

---

## Sync vs Async Communication

### Synchronous (Đồng Bộ) — Request/Response

```
Client ──HTTP POST──► Order Service ──HTTP──► Payment Service
        ◄── 200 OK ──              ◄── 200 ──
```

**Ưu điểm:** Đơn giản, response ngay, dễ debug  
**Nhược điểm:** Tight coupling (ghép chặt), cascade failure (lỗi dây chuyền), không hấp thụ spike

### Asynchronous (Bất Đồng Bộ) — Messaging

```
Order Service ──publish──► [ orders.created ] ──consume──► Payment Service
                         ──consume──► Inventory Service
                         ──consume──► Notification Service
```

**Ưu điểm:** Decoupling, resilience (khả năng phục hồi), scale độc lập  
**Nhược điểm:** Eventual consistency (nhất quán cuối cùng), phức tạp hơn, khó trace

### Khi Nào Chọn Async?

| Chọn Async | Chọn Sync |
| ---------- | --------- |
| Fire-and-forget (gửi xong không cần đợi) | Cần kết quả ngay (checkout, auth) |
| Nhiều service cùng phản ứng 1 event | Logic đơn giản, 2 service |
| Peak load cần buffer | Latency < 50ms bắt buộc |
| Cần retry/DLQ tự động | Strong consistency (nhất quán mạnh) bắt buộc |

---

## Use Cases Thực Tế

### 1. Order Processing (Xử Lý Đơn Hàng)

```
User đặt hàng → Order Service lưu DB → publish "OrderCreated"
  → Payment Service: charge
  → Inventory Service: reserve stock
  → Email Service: gửi xác nhận
```

### 2. Background Job / Task Queue

```
User upload ảnh → API trả 202 Accepted → Queue "image-resize"
  → Worker resize, watermark, upload S3
```

### 3. Log Aggregation (Tập Hợp Log)

```
100 app servers → Kafka topic "app-logs" → Elasticsearch / Datadog
```

### 4. Change Data Capture (CDC — Bắt Thay Đổi Dữ Liệu)

```
PostgreSQL WAL → Debezium → Kafka → Search index, Data warehouse
```

### 5. Notification Fan-out (Phân Phối Thông Báo)

```
1 event "MatchStarted" → SNS → 1M push notification subscribers
```

---

## Thuật Ngữ Quan Trọng

| Thuật Ngữ | Giải Thích |
| --------- | ---------- |
| **Producer** | Service gửi message/event |
| **Consumer** | Service nhận và xử lý message |
| **Broker** | Middleware lưu trữ và route message |
| **Queue** | Hàng đợi FIFO, competing consumers |
| **Topic** | Channel broadcast, nhiều subscriber |
| **Partition** | Shard (phân mảnh) của topic — Kafka |
| **Offset** | Vị trí đọc trong log — Kafka |
| **Ack (Acknowledge)** | Consumer xác nhận đã xử lý xong |
| **Nack / Reject** | Consumer từ chối message — trigger retry |
| **DLQ (Dead Letter Queue)** | Hàng đợi chứa message xử lý thất bại |
| **Poison Message** | Message gây lỗi liên tục, block queue |
| **Consumer Lag** | Độ trễ giữa message mới nhất và vị trí consumer |
| **Idempotency** | Xử lý trùng message không gây side effect |
| **Correlation ID** | ID theo dõi request xuyên suốt các service |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao cần Message Queue thay vì gọi API trực tiếp?

**Đáp án mẫu:** MQ giúp **decouple** producer và consumer — consumer down không làm producer fail. **Buffer** traffic spike khi consumer chậm. Hỗ trợ **retry** và **DLQ** khi xử lý lỗi. Cho phép **scale consumer** độc lập. Trade-off: thêm complexity, eventual consistency, cần monitoring lag.

### Câu 2: Message và Event khác nhau thế nào?

**Đáp án mẫu:** **Message** thường mang **command (lệnh)** — "hãy làm X" (ProcessPayment). **Event** mang **fact (sự kiện đã xảy ra)** — "PaymentCompleted". Trong thực tế ranh giới mờ; quan trọng là naming convention và consumer design phù hợp.

### Câu 3: Competing Consumers là gì?

**Đáp án mẫu:** Nhiều consumer instance cùng đọc **một queue** — mỗi message chỉ được **một** consumer xử lý (load balancing). Khác với Pub/Sub nơi **mỗi subscriber** nhận bản copy message.

### Câu 4: Message loss có thể xảy ra ở đâu?

**Đáp án mẫu:** (1) Producer gửi nhưng broker chưa persist — crash trước ack. (2) Consumer xử lý xong nhưng crash trước ack — message redeliver (duplicate, không mất). (3) Consumer auto-ack trước khi xử lý xong — crash = mất. (4) Broker disk full, retention xóa message. Giải pháp: publisher confirm, manual ack, replication.

---

**Xem tiếp:** [2-pub-sub-vs-point-to-point.md](./2-pub-sub-vs-point-to-point.md) — hai mô hình messaging cốt lõi.
