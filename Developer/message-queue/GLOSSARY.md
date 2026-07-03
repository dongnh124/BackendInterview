# 📖 Từ Điển Thuật Ngữ Message Queue & Event Broker

> Tập hợp thuật ngữ kỹ thuật quan trọng trong hệ sinh thái Message Queue (Hàng Đợi Tin Nhắn), Event Broker (Broker Sự Kiện) và Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện).  
> Thuật ngữ tiếng Anh được giữ nguyên, kèm giải thích tiếng Việt UTF-8.  
> Sắp xếp theo chủ đề; trong mỗi chủ đề sắp xếp theo bảng chữ cái.

---

## 📋 Tra Cứu Nhanh (A–Z)

| Thuật Ngữ | Ý Nghĩa Ngắn Gọn |
|-----------|-------------------|
| ACL | Access Control List — Danh sách kiểm soát truy cập topic/queue |
| acks | Producer acknowledgment level — Mức xác nhận ghi của producer |
| AMQP | Advanced Message Queuing Protocol — Giao thức hàng đợi tin nâng cao |
| At-least-once | Giao hàng ít nhất một lần — có thể trùng lặp |
| At-most-once | Giao hàng tối đa một lần — có thể mất message |
| Avro | Định dạng serialization nhị phân có schema, phổ biến với Kafka |
| Backpressure | Áp lực ngược — consumer chậm làm broker/queue đầy |
| Binding | Liên kết exchange với queue trong RabbitMQ |
| Broker | Máy chủ trung gian lưu trữ và định tuyến message |
| CDC | Change Data Capture — Bắt thay đổi database thành event |
| Circuit Breaker | Cầu dao bảo vệ — ngắt consumer khi downstream lỗi liên tục |
| Competing Consumers | Nhiều consumer cùng đọc một queue, mỗi message chỉ một consumer xử lý |
| Consumer Group | Nhóm consumer Kafka chia sẻ partition và offset |
| Consumer Lag | Độ trễ consumer — số message chưa xử lý so với log mới nhất |
| CQRS | Command Query Responsibility Segregation — Tách mô hình đọc/ghi |
| Dead Letter Queue (DLQ) | Hàng đợi thư chết — chứa message xử lý thất bại |
| Debezium | Connector CDC mã nguồn mở cho Kafka Connect |
| Deduplication | Khử trùng lặp message đã xử lý |
| DLX | Dead Letter Exchange — Exchange chuyển message sang DLQ |
| EDA | Event-Driven Architecture — Kiến trúc hướng sự kiện |
| Exactly-once | Giao hàng đúng một lần — khó đạt end-to-end |
| Exchange | Bộ trao đổi định tuyến message trong RabbitMQ |
| FIFO | First In, First Out — Vào trước ra trước |
| Fan-out | Phân phối một message đến nhiều consumer/queue |
| HPA | Horizontal Pod Autoscaler — Tự động mở rộng pod theo chiều ngang |
| Idempotency | Tính bất biến khi lặp lại — xử lý nhiều lần cho cùng kết quả |
| Inbox Pattern | Mẫu hộp thư vào — nhận message idempotent trong giao dịch DB |
| ISR | In-Sync Replicas — Bản sao đồng bộ trong Kafka |
| JetStream | Tầng persistence của NATS |
| Kafka Connect | Framework tích hợp source/sink với Kafka |
| Kafka Streams | Thư viện xử lý stream trên Kafka |
| KRaft | Kafka Raft — metadata quorum thay ZooKeeper |
| Lag | Độ trễ xử lý — backlog message chưa consume |
| mTLS | Mutual TLS — TLS hai chiều, xác thực cả client và server |
| Offset | Vị trí đọc trong partition Kafka |
| Outbox Pattern | Mẫu hộp thoại ra — gửi message đồng bộ với giao dịch DB |
| Partition | Phân vùng — đơn vị song song hóa trong Kafka topic |
| Poison Message | Tin nhắn độc — message gây lỗi lặp lại khi retry |
| Point-to-Point | Mô hình điểm-điểm — một producer, một consumer xử lý |
| Prefetch | Số message broker gửi trước cho consumer (RabbitMQ) |
| Producer | Thành phần gửi message/event lên broker |
| Pub/Sub | Publish/Subscribe — Xuất bản/Đăng ký |
| Pulsar | Apache Pulsar — broker đa tenant, geo-replication |
| Quorum Queue | Hàng đợi quorum RabbitMQ — replication dựa trên Raft |
| RabbitMQ | Message broker phổ biến, giao thức AMQP |
| Rebalance | Phân bổ lại partition cho consumer trong group |
| Replay | Phát lại message/event từ offset cũ |
| Retention | Thời gian hoặc dung lượng giữ message trên broker |
| Saga Pattern | Mẫu saga — giao dịch phân tán qua chuỗi local transaction |
| SASL | Simple Authentication and Security Layer — Lớp xác thực đơn giản |
| Schema Registry | Dịch vụ quản lý và version hóa schema message |
| SNS | Amazon Simple Notification Service — Pub/Sub managed trên AWS |
| SQS | Amazon Simple Queue Service — Hàng đợi managed trên AWS |
| Throughput | Thông lượng — số message xử lý trên đơn vị thời gian |
| Topic | Kênh log/event trong Kafka hoặc routing channel |
| Transactional Outbox | Outbox pattern trong transaction database |
| Visibility Timeout | Thời gian message ẩn khỏi queue sau khi consumer nhận (SQS) |
| ZooKeeper | Hệ thống coordination cũ cho Kafka metadata (đang thay bằng KRaft) |

---

## 🔵 01. Nền Tảng Messaging

### At-least-once Delivery (Giao Hàng Ít Nhất Một Lần)
Message được giao **ít nhất một lần** — consumer có thể nhận trùng nếu ack (xác nhận) thất bại hoặc consumer crash sau khi xử lý nhưng trước khi commit offset. **Mặc định phổ biến nhất** trong production; yêu cầu consumer **idempotent (bất biến khi lặp lại)**.

### At-most-once Delivery (Giao Hàng Tối Đa Một Lần)
Message có thể **bị mất** nhưng **không bao giờ trùng lặp**. Thường đạt bằng cách commit offset trước khi xử lý, hoặc fire-and-forget không ack. Phù hợp metric/logging không quan trọng.

### Backpressure (Áp Lực Ngược)
Khi consumer xử lý chậm hơn producer gửi, áp lực tích lũy ngược về upstream — queue depth tăng, memory/disk đầy, latency tăng. Giải pháp: rate limiting, throttling, scale consumer, hoặc từ chối message mới.

### Broker (Broker — Máy Chủ Trung Gian)
Thành phần trung tâm nhận message từ producer, lưu trữ (tùy cấu hình), và phân phối cho consumer. Ví dụ: Kafka broker, RabbitMQ node, Amazon SQS.

### Buffering (Đệm)
Broker giữ message tạm khi consumer offline hoặc chậm — hấp thụ traffic spike, consumer xử lý dần theo tốc độ của mình.

### Competing Consumers (Consumer Cạnh Tranh)
Nhiều consumer instance cùng đọc một queue; mỗi message chỉ được **một** consumer xử lý. Pattern phổ biến cho work queue và scale horizontal.

### Consumer (Consumer — Bên Tiêu Thụ)
Thành phần nhận và xử lý message từ broker. Có thể ack/nack (từ chối) message sau khi xử lý thành công hoặc thất bại.

### Decoupling (Tách Rời)
Producer không cần biết consumer nào đang chạy, ở đâu, hay có bao nhiêu instance — chỉ cần gửi message lên broker. Giảm coupling giữa microservices.

### Durability (Bền Vững)
Message được **persist (lưu bền)** trên disk hoặc storage replicated — không mất khi broker restart. Mức durability phụ thuộc cấu hình (acks, replication factor).

### Event Broker (Broker Sự Kiện)
Hệ thống messaging lưu event dưới dạng **append-only log (nhật ký chỉ ghi thêm)**. Message không bị xóa ngay sau consume; hỗ trợ replay, nhiều consumer group độc lập. Ví dụ: Apache Kafka, Apache Pulsar.

### Event (Sự Kiện)
Thông điệp mô tả **điều đã xảy ra** trong hệ thống (quá khứ): `OrderCreated`, `PaymentCompleted`. Khác với **Command (Lệnh)** — yêu cầu hành động tương lai.

### Exactly-once Delivery (Giao Hàng Đúng Một Lần)
Message được xử lý **đúng một lần** end-to-end — không mất, không trùng. **Rất khó** trong distributed system; thường chỉ đạt trong phạm vi broker + transactional consumer (ví dụ Kafka transactions), còn side effect bên ngoài vẫn cần idempotency.

### FIFO — First In, First Out (Vào Trước Ra Trước)
Thứ tự message được giữ nguyên: message gửi trước được consume trước. Amazon SQS FIFO, RabbitMQ single active consumer, Kafka partition đều đảm bảo ordering trong phạm vi nhất định.

### Flow Control (Điều Khiển Luồng)
Cơ chế giới hạn tốc độ gửi/nhận message giữa producer, broker và consumer — tránh overwhelm downstream. Bao gồm prefetch, credit-based flow control (AMQP), và rate limiting.

### Load Leveling (Cân Bằng Tải)
Broker hấp thụ burst traffic; consumer xử lý ở tốc độ ổn định — làm phẳng đỉnh tải, bảo vệ hệ thống downstream.

### Message (Tin Nhắn)
Đơn vị dữ liệu truyền qua messaging system — gồm payload (nội dung), headers/metadata (siêu dữ liệu), và đôi khi key (khóa phân vùng).

### Message Queue — MQ (Hàng Đợi Tin Nhắn)
Middleware cho phép ứng dụng giao tiếp **bất đồng bộ** qua queue. Message thường bị **xóa sau khi consume** (khác event log). Phù hợp task queue, job processing. Ví dụ: RabbitMQ, Amazon SQS.

### Middleware (Phần Mềm Trung Gian)
Lớp phần mềm đứng giữa các ứng dụng, xử lý messaging, routing, persistence — producer và consumer không giao tiếp trực tiếp.

### Point-to-Point (Điểm-Điểm)
Mô hình một producer gửi message vào queue; **một consumer** trong nhóm competing consumers xử lý. Khác Pub/Sub — mỗi subscriber nhận bản sao.

### Producer (Producer — Nhà Sản Xuất)
Thành phần tạo và gửi message/event lên broker. Chịu trách nhiệm serialization, chọn topic/queue, và (tùy cấu hình) đợi acknowledgment.

### Pub/Sub — Publish/Subscribe (Xuất Bản/Đăng Ký)
Producer **publish** message lên topic/channel; nhiều **subscriber** độc lập nhận bản sao. Phù hợp notification, event fan-out, decoupling nhiều service.

### Replay (Phát Lại)
Đọc lại message/event từ vị trí cũ trong log — hữu ích reprocess, recovery, hoặc thêm consumer group mới. Đặc trưng của event broker (Kafka), hạn chế trên traditional MQ.

### Retention (Giữ Lại)
Chính sách broker giữ message: theo thời gian (`retention.ms`), dung lượng (`retention.bytes`), hoặc log compaction. Quyết định khả năng replay và storage cost.

---

## 🟢 02. Mẫu Kiến Trúc & EDA

### Choreography (Điều Phối Nhảy Múa)
Trong Saga Pattern: mỗi service tự publish/subscribe event, **không có orchestrator trung tâm**. Linh hoạt, loose coupling; khó debug và theo dõi flow phức tạp.

### Command (Lệnh)
Message yêu cầu hành động: `CreateOrder`, `ProcessPayment`. Thường có **một consumer** xử lý; khác Event mô tả sự kiện đã xảy ra.

### Compensation (Bù Trừ)
Trong Saga: hành động **undo** khi bước sau thất bại — ví dụ `CancelReservation` bù cho `ReserveInventory`. Không phải rollback transaction DB truyền thống.

### CQRS — Command Query Responsibility Segregation (Tách Trách Nhiệm Đọc/Ghi)
Tách **write model** (command, event) và **read model** (query, projection). Write qua event stream; read qua database/materialized view được cập nhật bất đồng bộ.

### Deduplication — Dedup (Khử Trùng Lặp)
Cơ chế đảm bảo message trùng (do at-least-once) không gây side effect lặp. Dùng **deduplication key** (message ID, business key) lưu trong DB/cache.

### Domain Event (Sự Kiện Miền)
Event có ý nghĩa nghiệp vụ trong bounded context: `OrderPlaced`, `InventoryReserved`. Là building block của EDA và Event Sourcing.

### Dual-write Problem (Vấn Đề Ghi Kép)
Ghi database **và** gửi message **riêng biệt** — một trong hai có thể thất bại, gây inconsistency. Outbox Pattern giải quyết vấn đề này.

### EDA — Event-Driven Architecture (Kiến Trúc Hướng Sự Kiện)
Kiến trúc các service giao tiếp qua event async thay vì sync API call. Ưu: loose coupling, scalability. Nhược: complexity, eventual consistency, debugging khó hơn.

### Event Sourcing (Lưu Trữ Sự Kiện)
Lưu trạng thái hệ thống dưới dạng **chuỗi event bất biến** thay vì chỉ snapshot hiện tại. State được rebuild bằng cách replay events. Kết hợp tốt với CQRS.

### Eventual Consistency (Nhất Quán Cuối Cùng)
Trong EDA: dữ liệu giữa các service **không đồng bộ ngay lập tức** nhưng sẽ hội tụ sau một khoảng thời gian. Trade-off chấp nhận được cho async messaging.

### Idempotency (Tính Bất Biến Khi Lặp Lại)
Xử lý cùng message nhiều lần cho **cùng kết quả** như xử lý một lần. Bắt buộc với at-least-once delivery. Kỹ thuật: unique constraint, upsert, idempotency key table.

### Inbox Pattern (Mẫu Hộp Thư Vào)
Lưu message nhận vào bảng inbox trong **cùng transaction DB** với business logic — đảm bảo xử lý exactly-once effect phía consumer. Đôi khi kết hợp Outbox ở phía gửi.

### Orchestration (Điều Phối Tập Trung)
Trong Saga: **orchestrator** (service trung tâm) gửi command và theo dõi trạng thái từng bước. Dễ debug, dễ visualize flow; orchestrator là single point of logic.

### Outbox Pattern (Mẫu Hộp Thoại Ra)
Ghi message vào bảng `outbox` trong **cùng transaction** với business data; background process (hoặc CDC) đọc outbox và publish lên broker. Giải quyết dual-write problem.

### Projection (Chiếu Dữ Liệu)
View/read model được xây dựng bằng cách **consume event stream** và cập nhật database. Trong CQRS/Event Sourcing: nhiều projection từ cùng event log.

### Saga Pattern (Mẫu Saga)
Quản lý **distributed transaction** qua chuỗi local transaction + compensation event/command. Hai kiểu: choreography và orchestration. Thay thế 2PC (two-phase commit) trong microservices.

### Transactional Outbox (Outbox Giao Dịch)
Biến thể Outbox Pattern đảm bảo message chỉ được publish khi DB transaction commit thành công — atomicity giữa state change và event emission.

---

## 🟡 03. Apache Kafka

### acks (Producer Acknowledgment — Xác Nhận Producer)
Mức độ producer đợi broker xác nhận:
- `acks=0`: không đợi — at-most-once, throughput cao nhất.
- `acks=1`: leader ghi xong — có thể mất nếu leader chết trước replicate.
- `acks=all` (hoặc `-1`): tất cả ISR ack — durability cao nhất.

### Batch Size / linger.ms (Kích Thước Batch / Thời Gian Chờ)
Producer gom nhiều message thành batch trước khi gửi. `linger.ms` chờ thêm để đủ batch; tăng throughput, tăng latency nhẹ.

### Broker (Kafka Broker)
Máy chủ Kafka lưu partition, phục vụ produce/consume request. Cluster gồm nhiều broker; mỗi partition có một leader và nhiều follower.

### Compression (Nén)
Kafka hỗ trợ `gzip`, `snappy`, `lz4`, `zstd` — nén batch trước khi ghi disk. Giảm network/disk I/O; tăng CPU. `zstd` cân bằng tốt compression ratio và tốc độ.

### Consumer Group (Nhóm Consumer)
Nhóm consumer instance chia sẻ workload: **mỗi partition chỉ thuộc một consumer** trong group tại một thời điểm. Scale consumer tối đa bằng số partition.

### Follower (Bản Sao Theo Dõi)
Broker replica copy dữ liệu từ leader partition. Không phục vụ read client (trừ follower fetching cho consumer ở một số cấu hình).

### High Water Mark — HWM (Mốc Nước Cao)
Offset cao nhất đã replicate đến tất cả ISR — consumer chỉ đọc đến HWM để tránh đọc message chưa commit trên replica.

### ISR — In-Sync Replicas (Bản Sao Đồng Bộ)
Tập replica đã bắt kịp leader (trong `replica.lag.time.max.ms`). Chỉ ISR được tính khi `acks=all`. Replica rơi khỏi ISR khi lag quá ngưỡng.

### Kafka Connect (Kết Nối Kafka)
Framework tích hợp Kafka với hệ thống bên ngoài qua **connector**: Source (vào Kafka), Sink (ra khỏi Kafka). Chạy distributed mode để scale và HA.

### Kafka Streams (Luồng Kafka)
Thư viện Java xử lý stream: filter, map, aggregate, join, window trực tiếp trên topic. Stateful processing với **state store** local + changelog topic.

### KRaft — Kafka Raft (Metadata Quorum Kafka)
Kiến trúc metadata mới thay **ZooKeeper** — dùng Raft consensus cho controller quorum. Đơn giản hóa vận hành, giảm dependency.

### Leader (Leader Partition)
Broker đảm nhận read/write cho partition. Producer và consumer (mặc định) giao tiếp với leader.

### Log Compaction (Nén Log)
Retention policy giữ **bản ghi mới nhất** cho mỗi key — xóa bản cũ. Phù hợp changelog topic (CDC, compacted state). Không phải nén dữ liệu theo nghĩa compression.

### Offset (Vị Trí Đọc)
Số thứ tự đánh dấu vị trí consumer đã đọc trong partition. Consumer commit offset sau khi xử lý — quyết định replay point.

### Partition (Phân Vùng)
Đơn vị song song hóa trong Kafka topic. Message cùng **partition key** vào cùng partition — đảm bảo ordering trong partition. Số partition = upper bound parallelism.

### Partition Key (Khóa Phân Vùng)
Key producer gửi kèm message — hash vào partition. Thiết kế key quan trọng: `orderId` giữ ordering per order; key kém phân bố gây hot partition.

### Rebalance (Phân Bổ Lại)
Khi consumer join/leave group, partition được **phân bổ lại** giữa members. Gây stop-the-world consume tạm thời — cần tối ưu `session.timeout.ms`, cooperative rebalancing.

### Replication Factor (Hệ Số Nhân Bản)
Số bản copy mỗi partition trên cluster. `replication.factor=3` là phổ biến production — chịu được mất 1 broker (với min ISR=2).

### Retention (Giữ Lại — Kafka)
`retention.ms` / `retention.bytes` — thời gian hoặc dung lượng giữ message trên disk. Hết retention → message bị xóa (trừ compacted topic).

### Schema Registry (Đăng Ký Schema)
Dịch vụ lưu version schema (Avro, Protobuf, JSON Schema). Producer/consumers dùng schema ID trong message — đảm bảo compatibility khi evolve schema.

### Topic (Chủ Đề)
Kênh logical chứa partition — tương tự "table" trong database streaming. Đặt tên theo convention: `domain.entity.event` (ví dụ `orders.order.created`).

### Transactional Producer (Producer Giao Dịch)
Producer Kafka hỗ trợ ghi nhiều partition **atomic** trong transaction — kết hợp `read-process-write` exactly-once trong Kafka Streams. Cần `transactional.id`.

### ZooKeeper (Điều Phối Metadata Cũ)
Hệ thống coordination lưu metadata cluster Kafka (broker, topic, ACL) trong phiên bản cũ. Đang được thay bằng **KRaft** từ Kafka 3.x.

---

## 🟣 04. RabbitMQ & AMQP

### AMQP — Advanced Message Queuing Protocol (Giao Thức Hàng Đợi Tin Nâng Cao)
Giao thức chuẩn mà RabbitMQ implement. Mô hình: Exchange → Binding → Queue → Consumer. Hỗ trợ routing linh hoạt, ack, publisher confirm.

### Binding (Liên Kết)
Quy tắc kết nối **Exchange** với **Queue** — gồm routing key và đôi khi arguments (headers exchange). Message match binding mới vào queue.

### Consumer Ack — Acknowledgment (Xác Nhận Consumer)
Consumer báo broker đã xử lý xong (`basic.ack`) hoặc thất bại (`basic.nack`/`basic.reject`). Manual ack = at-least-once; auto ack = at-most-once risk.

### Dead Letter Exchange — DLX (Exchange Thư Chết)
Exchange nhận message bị reject, hết TTL, hoặc queue đầy — chuyển sang **DLQ** để phân tích và replay thủ công.

### Dead Letter Queue — DLQ (Hàng Đợi Thư Chết)
Queue chứa message xử lý thất bại sau max retry — tránh block queue chính. Cần monitoring và quy trình replay/fix.

### Direct Exchange (Exchange Trực Tiếp)
Route message theo **routing key** khớp chính xác với binding key. Phù hợp work queue, task routing đơn giản.

### Exchange (Bộ Trao Đổi)
Thành phần nhận message từ producer và route đến queue(s) theo loại exchange và binding. Không lưu message lâu — message nằm trong queue.

### Fanout Exchange (Exchange Phát Tán)
Bỏ qua routing key — gửi message đến **tất cả queue** đã bind. Pattern pub/sub broadcast.

### Headers Exchange (Exchange Header)
Route theo header attributes thay vì routing key — linh hoạt nhưng ít dùng hơn topic/direct.

### Mirrored Queue (Hàng Đợi Phản Chiếu)
Cơ chế HA cũ — copy queue sang node khác. Đang được thay bởi **Quorum Queue** (Raft-based).

### Prefetch Count (Số Lượng Prefetch)
Giới hạn số message chưa ack broker gửi đến consumer. Prefetch thấp = fair distribution; cao = throughput cao hơn nhưng có thể imbalance.

### Publisher Confirm (Xác Nhận Publisher)
Broker ack khi message đã vào queue (hoặc routed) — producer biết message không bị mất ở broker. Bắt buộc cho reliable publishing.

### Queue (Hàng Đợi)
Buffer lưu message chờ consumer. Có thể durable (survive restart), exclusive, auto-delete. Message ở đây cho đến khi ack.

### Quorum Queue (Hàng Đợi Quorum)
Queue type HA mới dựa trên **Raft consensus** — thay mirrored queue. Khuyến nghị cho production RabbitMQ 3.8+.

### Routing Key (Khóa Định Tuyến)
String producer gắn kèm message — exchange dùng để quyết định queue đích (direct, topic).

### Topic Exchange (Exchange Chủ Đề)
Route theo pattern routing key: `*` (một word), `#` (nhiều word). Ví dụ: `orders.*.created` match `orders.123.created`.

### TTL — Time To Live (Thời Gian Sống)
Message hoặc queue có TTL — message hết hạn chuyển sang DLX hoặc bị discard. Dùng cho delayed retry, expiration policy.

---

## 🟤 05. Broker Khác & Cloud

### Amazon MSK — Managed Streaming for Apache Kafka (Kafka Quản Lý Trên AWS)
Dịch vụ AWS chạy Kafka cluster managed — provisioning, patching, monitoring. Có MSK Provisioned và MSK Serverless.

### Amazon SNS — Simple Notification Service (Dịch Vụ Thông Báo Đơn Giản)
Pub/Sub managed trên AWS — fan-out message đến SQS, Lambda, HTTP, email, SMS. Thường kết hợp SQS cho durable queue.

### Amazon SQS — Simple Queue Service (Dịch Vụ Hàng Đợi Đơn Giản)
Queue managed serverless trên AWS. Standard queue (at-least-once, best-effort ordering) vs **FIFO queue** (ordering + dedup). **Visibility timeout** quan trọng.

### Apache Pulsar (Broker Đa Tenant)
Message platform: unified queue + streaming, **multi-tenancy**, geo-replication, tiered storage. Tách compute (broker) và storage (BookKeeper).

### Azure Event Hubs (Trung Tâm Sự Kiện Azure)
Dịch vụ ingest event scale cao trên Azure — có **Kafka endpoint** tương thích. Capture lưu raw data sang Blob/Data Lake.

### Confluent Cloud (Nền Tảng Kafka Trên Cloud)
Dịch vụ managed Kafka + Schema Registry + ksqlDB + Connect từ Confluent. Phù hợp team muốn Kafka ecosystem đầy đủ mà không tự vận hành cluster.

### GCP Pub/Sub (Pub/Sub Trên Google Cloud)
Messaging managed trên GCP — push/pull subscription, ordering keys, dead-letter topic. Tích hợp Cloud Functions, Dataflow.

### JetStream (Tầng Persistence NATS)
Extension của NATS thêm persistence, at-least-once, stream/consumer model. Nhẹ hơn Kafka; phù hợp cloud-native, edge.

### NATS (Hệ Thống Messaging Nhẹ)
Messaging system cực nhanh, subject-based routing. **NATS Core** = fire-and-forget, không persistence. **JetStream** = có persistence.

### Ordering Key (Khóa Thứ Tự)
Trong GCP Pub/Sub và Kafka partition key: message cùng ordering key được xử lý tuần tự. Thiết kế key tránh hotspot.

### Redis Pub/Sub (Xuất Bản/Đăng Ký Redis)
Fire-and-forget, không persistence — subscriber offline mất message. Phù hợp real-time notification, không phải reliable queue.

### Redis Streams (Luồng Redis)
Cấu trúc log trong Redis — persistence, consumer groups, ack. Lightweight alternative cho use case đơn giản, đã có Redis trong stack.

### Visibility Timeout (Thời Gian Ẩn)
Trong SQS: sau khi consumer nhận message, message **ẩn** khỏi queue trong khoảng timeout. Hết timeout chưa xóa → hiện lại (at-least-once). Phải set đủ lớn cho processing time.

---

## 🔴 06. Độ Tin Cậy & Xử Lý Lỗi

### Circuit Breaker (Cầu Dao Bảo Vệ)
Pattern ngắt gọi downstream khi lỗi vượt ngưỡng — tránh cascade failure. Với consumer: tạm dừng consume hoặc chuyển thẳng DLQ khi dependency down.

### Dead Letter Handling (Xử Lý Thư Chết)
Quy trình vận hành DLQ: alert, phân loại (poison vs transient), fix/replay, hoặc discard. Cần dashboard và runbook.

### Exponential Backoff (Lùi Lũy Tiến)
Tăng khoảng delay giữa các lần retry theo cấp số nhân: 1s → 2s → 4s → 8s. Giảm tải hệ thống đang gặp sự cố.

### Jitter (Nhiễu Ngẫu Nhiên)
Thêm randomness vào backoff — tránh **thundering herd** (đám đông ồ ạt retry cùng lúc).

### Max Deliver / Max Retry (Số Lần Giao Tối Đa)
Giới hạn số lần retry trước khi chuyển DLQ. Không retry vô hạn — poison message sẽ block queue.

### Poison Message (Tin Nhắn Độc)
Message **luôn gây lỗi** khi xử lý (bad data, bug code) — retry vô hạn không giúp. Cần detect, chuyển DLQ, alert, fix root cause.

### Retry Strategy (Chiến Lược Thử Lại)
Khi xử lý thất bại tạm thời (network, timeout): retry với backoff + jitter. Phân biệt **transient error** (retry được) vs **permanent error** (đưa DLQ ngay).

### Thundering Herd (Bầy Đàn Ồ Ạt)
Nhiều consumer/producer cùng retry hoặc reconnect đồng thời sau outage — gây spike tải. Jitter và staggered retry giảm hiện tượng này.

---

## 🟠 07. Hiệu Năng & Mở Rộng

### Backlog (Hàng Đợi Tồn Đọng)
Số message chưa xử lý trong queue/partition. Tăng backlog = consumer không theo kịp producer — cần scale hoặc tối ưu xử lý.

### Broker Sizing (Định Cỡ Broker)
Capacity planning: disk (retention × throughput), network bandwidth, memory (page cache), CPU (compression, TLS). Kafka disk I/O thường là bottleneck.

### Consumer Lag (Độ Trễ Consumer)
Trong Kafka: chênh lệch giữa **log end offset** và **committed offset** của consumer group. Metric quan trọng nhất — lag cao = xử lý chậm hoặc consumer down.

### Hot Partition (Phân Vùng Nóng)
Partition nhận disproportionate traffic do partition key kém phân bố (ví dụ tất cả message dùng key `null` hoặc `default`). Gây imbalance, một consumer quá tải.

### HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)
Kubernetes scale số pod consumer dựa trên CPU, memory, hoặc custom metric (ví dụ consumer lag qua KEDA). Scale theo lag hiệu quả hơn chỉ CPU.

### KEDA — Kubernetes Event-Driven Autoscaling (Tự Động Mở Rộng Theo Sự Kiện)
Scaler dựa trên message queue depth, Kafka lag, Prometheus metric — trigger HPA scale consumer/producer workers.

### Latency (Độ Trễ)
Thời gian từ produce đến consume xong. Batch và linger tăng throughput nhưng tăng latency. SLA latency vs throughput trade-off.

### Pipelining (Đường Ống)
Gửi nhiều request không đợi response trước — tăng throughput trên một connection. Kafka producer và AMQP đều hỗ trợ.

### Rate Limiting (Giới Hạn Tốc Độ)
Giới hạn số message/giây producer hoặc consumer xử lý — bảo vệ downstream và kiểm soát backpressure.

### Throughput (Thông Lượng)
Số message (hoặc bytes) xử lý trên đơn vị thời gian. Tune: batch size, partition count, compression, số consumer, network.

### Throttling (Hạn Chế)
Broker hoặc client giảm tốc độ khi đạt ngưỡng — tránh OOM hoặc overload. Kafka `quota.bytes.rate`, RabbitMQ memory alarm.

---

## ⚫ 08. Bảo Mật

### ACL — Access Control List (Danh Sách Kiểm Soát Truy Cập)
Quy tắc ai được read/write/create topic hoặc queue. Kafka ACL gắn với principal (user/service). Principle of least privilege.

### API Key (Khóa API)
Credential đơn giản cho managed service (Confluent Cloud, cloud Pub/Sub) — rotate định kỳ, không commit vào code.

### Audit Logging (Ghi Nhật Ký Kiểm Toán)
Ghi lại ai produce/consume, thay đổi ACL, tạo/xóa topic — yêu cầu compliance (SOC2, PCI-DSS).

### Encryption at-rest (Mã Hóa Lưu Trữ)
Message/schema encrypted trên disk broker. Kafka, cloud services hỗ trợ KMS-managed keys.

### Encryption in-transit (Mã Hóa Truyền Tải)
TLS giữa client và broker, broker-to-broker replication. Bắt buộc production — `SSL` hoặc `SASL_SSL` listener.

### mTLS — Mutual TLS (TLS Hai Chiều)
Cả client và server xác thực bằng certificate — mạnh hơn one-way TLS. Phổ biến service-to-service trong Kubernetes.

### RBAC — Role-Based Access Control (Phân Quyền Theo Vai Trò)
Gán quyền theo role (developer, operator, reader) thay vì từng user — dễ quản lý ở scale lớn.

### SASL — Simple Authentication and Security Layer (Lớp Xác Thực Đơn Giản)
Framework auth cho Kafka: **SCRAM-SHA-256/512**, PLAIN, GSSAPI (Kerberos). Kết hợp với SSL cho encryption.

### SCRAM — Salted Challenge Response Authentication Mechanism (Cơ Chế Xác Thực Phản Hồi Thách Thức Có Muối)
Cơ chế auth an toàn hơn PLAIN — password không gửi plaintext qua wire. Khuyến nghị cho Kafka production.

### TLS — Transport Layer Security (Bảo Mật Tầng Truyền Tải)
Mã hóa và xác thực kết nối mạng. Certificate rotation cần quy trình rõ ràng để tránh outage.

---

## 🔵 09. Giám Sát & Observability

### Alert Fatigue (Mệt Mỏi Cảnh Báo)
Quá nhiều alert không actionable — team bỏ qua alert thật. Chỉ alert symptom ảnh hưởng user; dùng SLO-based alerting.

### Correlation ID (ID Tương Quan)
ID duy nhất gắn xuyên suốt request/event qua nhiều service — trace flow trong log và distributed tracing.

### Distributed Tracing (Truy Vết Phân Tán)
Theo dõi message/event qua nhiều service — OpenTelemetry, Jaeger, Zipkin. Span cho produce, consume, process.

### Golden Signals (Tín Hiệu Vàng)
Bốn metric cốt lõi (Google SRE): **Latency**, **Traffic**, **Errors**, **Saturation**. Với messaging: thêm consumer lag, rebalance rate.

### Grafana (Bảng Điều Khiển Trực Quan)
Visualization dashboard cho Prometheus metrics — lag, throughput, error rate, broker health.

### OpenTelemetry — OTel (Chuẩn Telemetry Mở)
Chuẩn thu thập traces, metrics, logs — propagate trace context qua message headers (W3C Trace Context).

### Prometheus (Hệ Thống Metrics)
Time-series database pull metrics từ Kafka/RabbitMQ exporters. Query PromQL cho alerting (Alertmanager).

### Runbook (Sổ Tay Vận Hành)
Tài liệu bước xử lý khi alert fire: lag spike, broker down, DLQ flood. Liên kết từ alert đến runbook giảm MTTR.

### SLI — Service Level Indicator (Chỉ Số Mức Dịch Vụ)
Metric đo chất lượng: p99 end-to-end latency, error rate consume, max consumer lag.

### SLO — Service Level Objective (Mục Tiêu Mức Dịch Vụ)
Target cho SLI: "99% message xử lý trong 30 giây", "lag < 10,000 trong 99.9% thời gian". Drives alerting threshold.

### Under-replicated Partition (Phân Vùng Thiếu Bản Sao)
Kafka partition có replica chưa sync — rủi ro mất data nếu leader fail. Alert ngay trong production.

---

## 🟢 10. Chủ Đề Nâng Cao

### Avro (Định Dạng Serialization Có Schema)
Binary format compact, schema tách riêng trong Schema Registry. **Backward/forward compatibility** khi evolve field.

### Backward Compatibility (Tương Thích Ngược)
Consumer mới đọc được message schema cũ — thêm field optional, không xóa required field.

### CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu)
Theo dõi thay đổi database (INSERT/UPDATE/DELETE) và publish thành event. **Debezium** + Kafka Connect là stack phổ biến.

### Debezium (Connector CDC)
Open-source CDC connector đọc database transaction log (WAL, binlog) → Kafka topic. Không poll, low latency, at-least-once.

### Forward Compatibility (Tương Thích Xuôi)
Consumer cũ đọc được message schema mới — chỉ thêm field, consumer cũ ignore unknown fields.

### Geo-replication (Nhân Bản Địa Lý)
Copy topic/stream sang region/datacenter khác — disaster recovery, active-active. Pulsar và Kafka (MirrorMaker 2) hỗ trợ.

### JSON Schema / Protobuf (Schema JSON / Nhị Phân Google)
Alternative serialization: Protobuf hiệu năng cao, JSON Schema dễ debug. Schema Registry hỗ trợ cả ba (Avro, Protobuf, JSON).

### MirrorMaker 2 (Nhân Bản Liên Cluster Kafka)
Tool replicate topic giữa Kafka cluster — migration, geo-DR, aggregate multi-DC data.

### Schema Evolution (Tiến Hóa Schema)
Thay đổi cấu trúc message theo thời gian — cần compatibility rules và Schema Registry validation trước khi deploy producer mới.

### Serverless Consumer (Consumer Không Máy Chủ)
Lambda, Cloud Functions trigger bởi SQS, Kafka (Event Source Mapping), Pub/Sub — scale tự động, pay-per-use. Cold start và timeout cần lưu ý.

### Single Message Transform — SMT (Biến Đổi Từng Message)
Kafka Connect plugin transform record inline (filter, rename field, extract key) — lightweight hơn Kafka Streams cho ETL đơn giản.

### Windowing (Cửa Sổ Thời Gian)
Trong stream processing: aggregate event trong time window (tumbling, hopping, session). Kafka Streams, Flink, ksqlDB.

---

## 🔗 Các Thuật Ngữ Thường Bị Nhầm Lẫn

| Cặp Thuật Ngữ | Phân Biệt |
|---------------|-----------|
| **Message Queue vs Event Broker** | MQ: xóa sau consume, task-oriented — Event Broker: append log, replay, nhiều consumer group |
| **Topic vs Queue** | Topic (Kafka): log partitioned, retain — Queue (RabbitMQ/SQS): buffer, thường xóa sau ack |
| **At-least-once vs Exactly-once** | At-least-once: dễ, cần idempotent — Exactly-once: khó end-to-end, thường chỉ trong broker boundary |
| **Pub/Sub vs Point-to-Point** | Pub/Sub: mỗi subscriber nhận bản sao — P2P: competing consumers, mỗi message một handler |
| **Command vs Event** | Command: yêu cầu hành động, một target — Event: mô tả đã xảy ra, nhiều listener |
| **Choreography vs Orchestration** | Choreography: phân tán, event-driven — Orchestration: coordinator trung tâm điều khiển saga |
| **Outbox vs Inbox** | Outbox: phía gửi, đảm bảo publish — Inbox: phía nhận, đảm bảo xử lý không trùng |
| **DLQ vs Retry Queue** | DLQ: message thất bại vĩnh viễn, cần can thiệp — Retry queue: delay rồi thử lại tự động |
| **Consumer Lag vs Queue Depth** | Lag (Kafka): offset chênh lệch — Depth (RabbitMQ/SQS): số message trong queue |
| **Partition vs Consumer** | Nhiều partition cho parallelism — Số consumer > partition thì consumer thừa idle |
| **Redis Pub/Sub vs Redis Streams** | Pub/Sub: không persist, mất khi offline — Streams: persist, consumer groups, ack |
| **NATS Core vs JetStream** | Core: cực nhanh, fire-and-forget — JetStream: persistence, at-least-once |
| **acks=1 vs acks=all** | acks=1: chỉ leader ack — acks=all: toàn bộ ISR ack, an toàn hơn |
| **Auto commit vs Manual commit** | Auto: đơn giản, rủi ro mất/duplicate khi crash — Manual: kiểm soát, chuẩn production |
| **ZooKeeper vs KRaft** | ZK: dependency riêng, legacy — KRaft: metadata built-in Kafka, hướng tương lai |
| **SQS Standard vs FIFO** | Standard: throughput cao, ordering best-effort — FIFO: ordering + dedup, throughput thấp hơn |
| **Kafka vs RabbitMQ** | Kafka: log, throughput, replay — RabbitMQ: routing linh hoạt, task queue, protocol AMQP |
| **Idempotency vs Deduplication** | Idempotency: logic xử lý an toàn khi lặp — Dedup: loại bỏ message trùng trước khi xử lý |
| **Event Sourcing vs CQRS** | Event Sourcing: lưu state qua events — CQRS: tách read/write model (có thể không event sourcing) |
| **Throughput vs Latency** | Tăng batch/linger → throughput ↑ latency ↑ — tối ưu theo SLA cụ thể |

---

## 📦 Broker & Công Cụ Quan Trọng

| Tên | Loại | Mục Đích |
|-----|------|----------|
| **Apache Kafka** | Event Broker | Log streaming, high throughput, replay |
| **RabbitMQ** | Message Queue | AMQP routing, task queue, DLQ |
| **Amazon SQS/SNS** | Cloud MQ/Pub-Sub | Serverless, AWS-native |
| **Amazon MSK** | Managed Kafka | Kafka trên AWS không tự ops cluster |
| **Confluent Cloud** | Managed Kafka Platform | Kafka + Schema Registry + ksqlDB |
| **Azure Event Hubs** | Cloud Event Ingest | Kafka-compatible endpoint |
| **GCP Pub/Sub** | Cloud Pub/Sub | Global, push/pull, GCP integration |
| **Apache Pulsar** | Event Broker | Multi-tenant, geo-replication |
| **NATS / JetStream** | Lightweight Messaging | Cloud-native, edge, IoT |
| **Redis Streams** | In-memory Stream | Low-latency, đơn giản |
| **Debezium** | CDC Connector | Database → Kafka events |
| **Kafka Connect** | Integration Framework | Source/sink connectors |
| **Kafka UI / AKHQ** | Web UI | Quản lý topic, consumer group, message browse |
| **kcat (kafkacat)** | CLI Tool | Produce/consume/debug Kafka từ terminal |
| **Prometheus + Grafana** | Monitoring | Metrics, dashboard, alerting |
| **KEDA** | K8s Autoscaler | Scale consumer theo queue lag |
| **Schema Registry** | Schema Management | Avro/Protobuf/JSON schema versioning |
| **MirrorMaker 2** | Replication | Cross-cluster Kafka replication |
| **MassTransit / NServiceBus** | .NET Libraries | Abstraction AMQP/Azure Service Bus |
| **Spring Kafka / kafka-node** | Client Libraries | Producer/consumer trong ứng dụng |

---

## 🔢 Số Liệu & Ngưỡng Tham Khảo

| Chỉ Số | Ngưỡng / Giá Trị Khuyến Nghị |
|--------|------------------------------|
| Kafka `replication.factor` | 3 cho production |
| Kafka `min.insync.replicas` | 2 (với RF=3) |
| Kafka partition count | Bắt đầu 6–12; scale theo consumer throughput |
| Consumer : Partition ratio | Tối đa 1 active consumer / partition trong group |
| RabbitMQ prefetch | 10–50 cho workload cân bằng; 1 cho fair dispatch nghiêm ngặt |
| SQS visibility timeout | ≥ 6 × thời gian xử lý message trung bình |
| DLQ max retry | 3–5 lần trước khi chuyển DLQ |
| Consumer lag alert | Warning: lag > 10K hoặc tăng liên tục 15 phút — Critical: lag > 100K |
| Kafka retention default | 7 ngày (`retention.ms=604800000`) — điều chỉnh theo compliance |
| Message size limit | Kafka default 1MB — tăng cẩn thận, ảnh hưởng broker |
| Session timeout (Kafka consumer) | 10–30 giây — cân bằng failure detection vs rebalance |
| Exponential backoff base | 1–5 giây, max 5–10 lần retry |
| HPA scale theo lag | Scale out khi lag > target per consumer trong 2–3 chu kỳ |

---

## 📚 Liên Kết Trong Knowledge Base

| Chủ Đề | Tham Chiếu |
|--------|------------|
| Nền tảng messaging | [01-fundamentals/](01-fundamentals/) |
| Mẫu kiến trúc EDA | [02-architecture-patterns/](02-architecture-patterns/) |
| Apache Kafka | [03-apache-kafka/](03-apache-kafka/) |
| RabbitMQ | [04-rabbitmq/](04-rabbitmq/) |
| Broker khác | [05-other-brokers/](05-other-brokers/) |
| Độ tin cậy | [06-reliability/](06-reliability/) |
| Hiệu năng | [07-performance-scaling/](07-performance-scaling/) |
| Bảo mật | [08-security/](08-security/) |
| Giám sát | [09-monitoring/](09-monitoring/) |
| Cloud managed | [10-cloud-managed/](10-cloud-managed/) |
| Nâng cao | [11-advanced/](11-advanced/) |
| Phỏng vấn | [12-interview-prep/](12-interview-prep/) |

---

**Cập Nhật Lần Cuối:** 2026-07-03  
**Phiên Bản:** 1.0  
**Liên Quan:** [README.md](README.md) | [INDEX.md](INDEX.md)
