### Tổ chức code của dự án đó như thế nào, có những layer nào?

**Follow-up:**
- Logic validate user nằm ở controller hay service?

**Gợi ý đáp án:**
- Controller/route: nhận request, validate input, gọi service, trả response — không SQL trực tiếp
- Service: business rule (tính giá, check tồn kho, state transition)
- Repository/DAO: query DB, map entity ↔ DTO
- Module theo domain khi file > 300 dòng hoặc 2 team cùng sửa 1 folder
- Spaghetti: controller 200 dòng, import chéo module, test không mock được DB

### Giải thích 5 nguyên lý SOLID và cho ví dụ trong dự án Node.js/React?

**Gợi ý đáp án:**
- S – Single Responsibility: Mỗi class/module chỉ có một lý do để thay đổi. VD: tách UserService (business logic) khỏi UserRepository (DB access).
- O – Open/Closed: Mở rộng hành vi mà không sửa code cũ. VD: thêm payment provider mới qua interface, không sửa PaymentService hiện tại.
- L – Liskov Substitution: Subclass thay thế được base class mà không phá vỡ logic. VD: PayPalPayment và StripePayment đều implement IPayment.
- I – Interface Segregation: Không ép client implement method không dùng. VD: tách IReadable / IWritable thay vì một interface lớn.
- D – Dependency Inversion: Phụ thuộc vào abstraction, không phụ thuộc concrete. VD: inject IEmailService thay vì gọi trực tiếp SendGridClient.

### Singleton Pattern là gì và cho ví dụ trong dự án Node.js/React?

**Gợi ý đáp án:**
- Singleton: Đảm bảo chỉ một instance tồn tại. Dùng cho DB connection pool, logger, config manager..

### Sự khác biệt giữa HTTP methods GET, POST, PUT, PATCH, DELETE? Khi nào dùng PUT vs PATCH?

**Gợi ý đáp án:**
- GET: Lấy resource, idempotent, safe (không thay đổi state).
- POST: Tạo resource mới, không idempotent.
- PUT: Thay thế toàn bộ resource, idempotent.
- PATCH: Cập nhật một phần resource, idempotent (nên thiết kế như vậy).
- DELETE: Xóa resource, idempotent.
- PUT vs PATCH: PUT gửi full object; PATCH gửi partial fields ({ "name": "new" }).

### Các HTTP status code quan trọng và khi nào trả về?

**Follow-up:**
- 401 vs 403 khác nhau thế nào trong thực tế?
- Khi nào dùng 202 Accepted thay vì 201?

**Gợi ý đáp án:**
- 2xx: 200 OK, 201 Created (+ Location header), 204 No Content.
- 3xx: 301/302 redirect, 304 Not Modified (caching).
- 4xx: 400 Bad Request (validation), 401 Unauthorized (chưa auth), 403 Forbidden (không có quyền), 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests.
- 5xx: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout.
- Nguyên tắc: 4xx = lỗi client, 5xx = lỗi server. Không dùng 200 kèm { success: false }.

### Websocket là gì, khi nào thì sử dụng websocket?

### Authentication vs Authorization?

**Follow-up:**
- Access token vs Session token?

**Gợi ý đáp án:**
- Access token ngắn hạn (15m), refresh token dài hạn (7d), lưu refresh trong httpOnly secure cookie
- Refresh token rotation: mỗi lần refresh cấp token mới, invalidate cũ
- Blacklist/revocation list hoặc session store (Redis) cho logout
- Không lưu access token trong localStorage (XSS risk)

### Index hoạt động như thế nào? Khi nào index giúp và khi nào làm chậm?

**Gợi ý đáp án:**
- Index (B-Tree phổ biến) tạo cấu trúc tìm kiếm nhanh, tránh full table scan.
- Giúp: WHERE, JOIN, ORDER BY trên indexed columns.
- Làm chậm: INSERT/UPDATE/DELETE (phải update index), quá nhiều index, index trên column cardinality thấp (gender).
- Composite index: thứ tự column quan trọng (leftmost prefix rule).
- EXPLAIN/EXPLAIN ANALYZE để phân tích query plan.

### Transaction là gì, sử dụng khi nào?

**Follow-up:**
- Giải thích về ACID?

**Gợi ý đáp án:**
- ACID: Atomicity, Consistency, Isolation, Durability.
- Isolation levels (từ thấp đến cao):
- Read Uncommitted: Dirty read.
- Read Committed: Tránh dirty read, vẫn non-repeatable read (PG default).
- Repeatable Read: Tránh non-repeatable read, vẫn phantom read (MySQL InnoDB default).
- Serializable: An toàn nhất, chậm nhất.
- Vấn đề: Dirty read, Non-repeatable read, Phantom read, Lost update.
- Giải pháp lost update: optimistic locking (version column), pessimistic locking (SELECT FOR UPDATE).

### N+1 query problem — phát hiện và fix

### Connection Pool (Bể Kết Nối) — tại sao cần và sizing

**Gợi ý đáp án:**
- Mỗi DB connection tốn ~1–5MB RAM trên DB server. Tạo connection mới tốn ~20–50ms.
- Connection pool tái sử dụng connections

### Event Loop hoạt động như thế nào?

**Follow-up:**
- Microtask vs macrotask — thứ tự thực thi?

**Gợi ý đáp án:**
Node.js chạy JavaScript trên single main thread (luồng chính đơn) với Event Loop điều phối async operations:
1.	Code sync chạy trên Call Stack (Ngăn Xếp Gọi Hàm)
2.	Async I/O được đăng ký với libuv (thư viện C xử lý I/O)
3.	Khi I/O hoàn thành, callback vào Callback Queue (Hàng Đợi Callback)
4.	Event Loop lấy callback từ queue đưa lên Call Stack khi stack rỗng
5.	Microtasks (Promise callbacks, process.nextTick) được ưu tiên trước macrotasks (setTimeout, setInterval, setImmediate, I/O callbacks)

### Dùng await trong for loop và dùng await trong array foreach/map có gì khác biệt?

**Gợi ý đáp án:**
dùng await trong for/for...of sẽ chờ từng promise theo thứ tự (tuần tự), còn dùng await trong Array.forEach/map không chờ (một callback async sẽ khởi tạo promise mà forEach/map không đợi) — nếu muốn chạy song song với map thì dùng Promise.all, nếu muốn tuần tự thì dùng for/for...of.

### Promise.all vs Promise.allSettled vs Promise.race?

**Gợi ý đáp án:**
- Promise.all: Tất cả resolve → array kết quả. Một reject → reject ngay (fail-fast).
- Promise.allSettled: Chờ tất cả, trả [{status, value/reason}] — không fail-fast.
- Promise.race: Kết quả của promise đầu tiên settle — dùng cho timeout pattern.

### Nguyên nhân phổ biến gây ra Memory leak và cách debug?

**Gợi ý đáp án:**
- Nguyên nhân: global variables tích lũy, closures giữ reference, event listeners không remove, timers không clear, cache không giới hạn.
- Debug: node --inspect, Chrome DevTools Memory tab, process.memoryUsage(), heap snapshot, clinic.js, memwatch.
- Phòng tránh: LRU cache với max size, removeListener, clearInterval, WeakMap/WeakRef.

### Middleware trong Express hoạt động như thế nào? Thứ tự middleware quan trọng ra sao?

**Gợi ý đáp án:**
- Middleware là function (req, res, next) => {} xử lý request theo pipeline.
- Thứ tự đăng ký = thứ tự thực thi.
- Thứ tự phổ biến: helmet (security) → cors → body-parser → auth → routes → error handler (cuối cùng).
- Error middleware có 4 params: (err, req, res, next).
- Quên gọi next() → request bị treo.

### Log levels và khi nào dùng từng level?

**Follow-up:**
- Xem log ở production như nào?

**Gợi ý đáp án:**
- ERROR: Lỗi cần xử lý ngay, ảnh hưởng user. Exception, failed payment.
- WARN: Bất thường nhưng hệ thống vẫn chạy. Deprecated API, retry succeeded.
- INFO: Business events quan trọng. User login, order created, service started.
- DEBUG: Chi tiết cho development/troubleshooting. Query params, intermediate values.
- TRACE: Rất chi tiết, hiếm dùng.
- Production: thường INFO trở lên. DEBUG chỉ khi troubleshoot.

### Message Queue giải quyết vấn đề gì? So sánh sync vs async communication?

**Follow-up:**
- Trong luồng thanh toán mà order service cần thông tin user thì có dùng message queue được không?
- Nếu monitor thấy api thanh toán bị chậm do gọi sang user service lấy thông tin user mất 3s thì cách giải quyết như nào?

**Gợi ý đáp án:**
- Vấn đề giải quyết: Decoupling services, async processing, load leveling, reliability (message persist khi consumer down).
- Sync (HTTP/RPC): Đơn giản, immediate response. Nhược: tight coupling, cascade failure, peak load.
- Async (MQ): Producer không cần chờ consumer. Buffer khi consumer chậm. Retry dễ hơn.
- Use cases: gửi email, xử lý order, event notification, data pipeline, log aggregation.

### Kafka vs RabbitMQ – khác nhau cơ bản và khi nào chọn gì?

**Gợi ý đáp án:**
| | Kafka | RabbitMQ |
|---|-------|----------|
| Model | Distributed commit log | Traditional message broker |
| Throughput | Rất cao (hàng triệu msg/s) | Trung bình (hàng chục nghìn/s) |
| Message retention | Configurable, lâu dài | Xóa sau khi consume |
| Consumer | Pull-based, consumer group | Push-based, competing consumers |
| Use case | Event streaming, log aggregation, analytics | Task queue, RPC, complex routing |
| Routing | Topic + partition | Exchange (direct, topic, fanout, headers) |

- **ActiveMQ:** JMS standard, Java ecosystem, ít phổ biến hơn trong Node.js world.
- Node.js: `kafkajs`, `amqplib` (RabbitMQ).

### Triển khai tính năng notification cho hệ thống mới?

**Follow-up:**
- Cần những tính năng gì?
- Triển khai tính năng chính?
- Monitor như nào?

### CI/CD pipeline cho Node.js app – các stage cơ bản?

**Gợi ý đáp án:**
- Source: Trigger từ git push/PR (webhook).
- Build: npm ci, compile TypeScript, build Docker image.
- Test: Unit test, integration test, lint (ESLint), security scan.
- Quality Gate: Coverage threshold, no critical vulnerability.
- Publish: Push Docker image lên registry (ECR, Docker Hub).
- Deploy: Deploy lên staging → smoke test → deploy production.
- Notify: Slack/email kết quả.


### Docker swarm và K8s có gì khác nhau, khi nào thì dùng?

### Git Merge, Rebase, và Squash?

**Gợi ý đáp án:**
- Merge: "Giữ lại toàn bộ dấu vết và lịch sử". Nó tạo một commit gộp (merge commit) để nối hai nhánh. Lịch sử sẽ đan xen, chi tiết, nhưng có thể hơi rối mắt.
- Rebase: "Viết lại lịch sử". Nó bứng toàn bộ các commit của bạn dời lên đỉnh của nhánh chính, tạo thành một đường thẳng tắp (linear). Lịch sử rất gọn gàng nhưng làm thay đổi cấu trúc commit ban đầu.
- Squash: "Tập hợp thành một". Nó gộp tất cả các commit nhỏ trong nhánh của bạn thành một commit duy nhất rồi đưa vào nhánh đích. Giúp main branch gọn gàng nhưng vẫn giữ được chi tiết code