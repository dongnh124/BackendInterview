# Bộ câu hỏi phỏng vấn Backend Node.js

> **Techstack:** NestJS, TypeScript, PostgreSQL/MySQL, MongoDB, Redis, Kafka, Docker, AWS/GCP, Git, CI/CD
>
> **Quy ước đánh giá level:**
> - **Junior (1-2 năm):** Trả lời được ý cơ bản, biết khái niệm nhưng chưa sâu, ít kinh nghiệm thực tế
> - **Mid (2-4 năm):** Trả lời rõ ràng, có ví dụ thực tế, hiểu được trade-off của các giải pháp
> - **Senior (4+ năm):** Trả lời sâu, phân tích được ưu nhược điểm, đưa ra giải pháp phù hợp với ngữ cảnh cụ thể, có kinh nghiệm xử lý sự cố thực tế

---

## 1. Code / Project Structure

### Câu 1.1
**Câu hỏi:** Khi bạn bắt đầu một dự án NestJS mới, bạn tổ chức cấu trúc thư mục như thế nào? Tại sao bạn lại chọn cách tổ chức đó thay vì các cách khác?

**Câu hỏi mở rộng:**
- Bạn phân biệt thế nào giữa việc tổ chức theo tính năng (feature-based) và tổ chức theo loại file (type-based)? Khi nào nên dùng cách nào?
- Khi dự án lớn lên, có hàng trăm file, bạn xử lý thế nào để cấu trúc vẫn dễ quản lý?

**Cách trả lời tốt:**
- Giải thích được 2 cách tổ chức chính: theo tính năng (mỗi module chứa controller, service, repository riêng) và theo loại (gom tất cả controller vào 1 thư mục, tất cả service vào 1 thư mục)
- Nêu được lý do chọn tổ chức theo tính năng cho dự án lớn vì dễ tách module, dễ bảo trì
- Đề cập đến shared/common module cho các thành phần dùng chung
- Senior: Nói về monorepo, chia thành các thư viện dùng chung khi có nhiều dịch vụ

**Đánh giá level:**
- Junior: Biết cấu trúc mặc định của NestJS, liệt kê được các thư mục cơ bản
- Mid: Giải thích được lý do chọn cách tổ chức, có kinh nghiệm refactor cấu trúc
- Senior: Đưa ra chiến lược tổ chức phù hợp với quy mô dự án, đề cập monorepo/micro-frontend

---

### Câu 1.2
**Câu hỏi:** Trong NestJS, khái niệm Module, Controller, Service, Repository đóng vai trò gì? Tại sao NestJS lại tách ra như vậy thay vì gom hết vào một file?

**Câu hỏi mở rộng:**
- Nếu một Service cần gọi đến Service của module khác, bạn xử lý thế nào? Có vấn đề gì xảy ra khi hai module phụ thuộc lẫn nhau không?
- Bạn có biết khái niệm "tiêm phụ thuộc" (dependency injection) không? NestJS áp dụng nó như thế nào?

**Cách trả lời tốt:**
- Module: đóng gói và quản lý các thành phần liên quan, kiểm soát phạm vi sử dụng
- Controller: nhận yêu cầu từ bên ngoài, xác nhận dữ liệu đầu vào, chuyển tiếp cho Service
- Service: chứa logic nghiệp vụ chính
- Repository: tương tác với cơ sở dữ liệu, tách biệt logic truy vấn
- Giải thích được vấn đề phụ thuộc vòng (circular dependency) và cách giải quyết bằng forwardRef
- Senior: Nói về lợi ích của việc tách lớp cho testing, bảo trì, và thay thế thành phần

**Đánh giá level:**
- Junior: Mô tả được vai trò cơ bản của từng thành phần
- Mid: Giải thích được dependency injection, xử lý được circular dependency
- Senior: Hiểu sâu về IoC container, custom provider, scope của provider (singleton, request, transient)

---

### Câu 1.3
**Câu hỏi:** Khi bạn viết code TypeScript, bạn sử dụng kiểu dữ liệu (type) và giao diện (interface) như thế nào? Khi nào bạn chọn dùng cái này thay vì cái kia?

**Câu hỏi mở rộng:**
- Bạn có dùng Generic không? Cho ví dụ một trường hợp bạn đã dùng Generic trong dự án thực tế?
- Bạn xử lý thế nào khi dữ liệu từ bên ngoài (API, database) không khớp với kiểu dữ liệu bạn đã định nghĩa?

**Cách trả lời tốt:**
- Interface dùng khi cần khai báo hình dạng (shape) của đối tượng, có thể mở rộng (extend) và hợp nhất (merge)
- Type linh hoạt hơn: union type, intersection type, mapped type, conditional type
- Trong NestJS, thường dùng interface cho DTO, entity; dùng type cho union, utility type
- Senior: Nói về strict mode, utility types (Partial, Pick, Omit), discriminated union

**Đánh giá level:**
- Junior: Phân biệt được interface và type ở mức cơ bản
- Mid: Sử dụng được Generic, biết các utility types phổ biến
- Senior: Áp dụng advanced types, xây dựng type-safe API, runtime validation với class-validator/zod

---

### Câu 1.4
**Câu hỏi:** Bạn có kinh nghiệm viết kiểm thử (test) cho ứng dụng NestJS không? Bạn thường viết những loại test nào và tổ chức chúng ra sao?

**Câu hỏi mở rộng:**
- Khi Service của bạn phụ thuộc vào database và các dịch vụ bên ngoài, bạn viết test như thế nào mà không cần kết nối thật?
- Bạn đặt mục tiêu bao phủ code (coverage) bao nhiêu phần trăm? Tại sao?

**Cách trả lời tốt:**
- Unit test: kiểm thử từng hàm/method riêng lẻ, mock các dependency
- Integration test: kiểm thử sự tương tác giữa các thành phần (ví dụ Controller + Service + DB)
- E2E test: kiểm thử toàn bộ luồng từ request đến response
- Sử dụng Testing module của NestJS để tạo môi trường test
- Mock dependency bằng jest.mock hoặc custom provider
- Senior: Chiến lược test pyramid, contract testing, test database riêng

**Đánh giá level:**
- Junior: Biết viết unit test cơ bản với Jest
- Mid: Viết được integration test, mock dependency phức tạp, biết setup test database
- Senior: Xây dựng chiến lược test cho cả dự án, CI/CD integration, performance testing

---

## 2. SOLID / Design Pattern

### Câu 2.1
**Câu hỏi:** Bạn có thể giải thích nguyên tắc "trách nhiệm đơn lẻ" (Single Responsibility) bằng ví dụ thực tế trong dự án NestJS không? Tại sao việc tuân thủ nguyên tắc này lại quan trọng?

**Câu hỏi mở rộng:**
- Bạn đã bao giờ gặp một class hoặc function quá lớn, làm quá nhiều việc chưa? Bạn xử lý thế nào?
- Nguyên tắc này áp dụng thế nào ở mức module, không chỉ ở mức class?

**Cách trả lời tốt:**
- Mỗi class/module chỉ nên có một lý do để thay đổi
- Ví dụ: tách UserService (xử lý logic user) khỏi EmailService (gửi email), không gom vào một service
- Ví dụ: Controller chỉ nhận request và trả response, không chứa logic nghiệp vụ
- Lợi ích: dễ test, dễ bảo trì, dễ tái sử dụng, giảm side effect khi thay đổi
- Senior: Đề cập đến việc cân bằng giữa tách quá nhỏ (over-engineering) và gom quá lớn (god class)

**Đánh giá level:**
- Junior: Giải thích được khái niệm, cho ví dụ đơn giản
- Mid: Áp dụng được trong thực tế, nhận biết khi nào cần refactor
- Senior: Cân bằng giữa nguyên tắc và thực tiễn, áp dụng ở nhiều cấp độ (function, class, module, service)

---

### Câu 2.2
**Câu hỏi:** Nguyên tắc "đảo ngược phụ thuộc" (Dependency Inversion) nói rằng code cấp cao không nên phụ thuộc trực tiếp vào code cấp thấp. Bạn hiểu điều này thế nào và NestJS hỗ trợ nguyên tắc này ra sao?

**Câu hỏi mở rộng:**
- Nếu bạn muốn thay đổi từ MySQL sang MongoDB mà không ảnh hưởng đến logic nghiệp vụ, bạn thiết kế code như thế nào?
- Custom Provider trong NestJS hoạt động thế nào? Khi nào bạn cần dùng nó?

**Cách trả lời tốt:**
- Code cấp cao (Service) phụ thuộc vào abstraction (Interface), không phụ thuộc trực tiếp vào implementation (Repository cụ thể)
- NestJS hỗ trợ qua dependency injection: inject interface, cung cấp implementation qua provider
- Ví dụ: định nghĩa IUserRepository interface, implement MySQLUserRepository và MongoUserRepository, inject qua token
- Lợi ích: dễ thay thế, dễ test (mock interface), giảm coupling
- Senior: Factory provider, async provider, dynamic module

**Đánh giá level:**
- Junior: Hiểu khái niệm dependency injection cơ bản
- Mid: Áp dụng được interface + injection trong NestJS, viết custom provider
- Senior: Thiết kế hệ thống linh hoạt với abstract layer, plugin architecture

---

### Câu 2.3
**Câu hỏi:** Bạn đã sử dụng những mẫu thiết kế (design pattern) nào trong dự án thực tế? Cho ví dụ cụ thể và giải thích tại sao bạn chọn mẫu đó?

**Câu hỏi mở rộng:**
- Bạn có biết mẫu "kho lưu trữ" (Repository Pattern) không? Nó khác gì so với việc gọi trực tiếp ORM trong Service?
- Mẫu "chiến lược" (Strategy Pattern) có thể giải quyết bài toán gì trong dự án thực tế?

**Cách trả lời tốt:**
- Repository Pattern: tách logic truy vấn database ra khỏi business logic, dễ thay đổi ORM hoặc database
- Strategy Pattern: xử lý nhiều loại thanh toán (thẻ, ví điện tử, chuyển khoản) mà không cần if-else dài
- Observer Pattern: NestJS EventEmitter để xử lý sự kiện bất đồng bộ
- Factory Pattern: tạo đối tượng phức tạp với nhiều biến thể
- Decorator Pattern: NestJS sử dụng rất nhiều (Guard, Interceptor, Pipe)
- Senior: CQRS pattern, Saga pattern, Event Sourcing

**Đánh giá level:**
- Junior: Biết 1-2 pattern cơ bản, giải thích được khái niệm
- Mid: Áp dụng được 3-4 pattern trong dự án thực tế, giải thích trade-off
- Senior: Kết hợp nhiều pattern, biết khi nào KHÔNG nên dùng pattern, áp dụng domain-driven design

---

### Câu 2.4
**Câu hỏi:** Trong NestJS có các khái niệm Middleware, Guard, Interceptor, Pipe. Chúng khác nhau thế nào và bạn sử dụng từng cái trong trường hợp nào?

**Câu hỏi mở rộng:**
- Thứ tự thực thi của chúng là gì? Nếu bạn cần log thời gian xử lý request, bạn dùng cái nào?
- Bạn đã bao giờ viết custom Decorator chưa? Cho ví dụ?

**Cách trả lời tốt:**
- Middleware: chạy đầu tiên, xử lý chung (logging, CORS), giống Express middleware
- Guard: kiểm tra quyền truy cập (authentication, authorization), trả về true/false
- Interceptor: can thiệp trước và sau khi handler xử lý (transform response, cache, logging thời gian)
- Pipe: validate và transform dữ liệu đầu vào (ValidationPipe, ParseIntPipe)
- Thứ tự: Middleware → Guard → Interceptor (before) → Pipe → Handler → Interceptor (after)
- Senior: Custom decorator kết hợp với metadata (SetMetadata, Reflector)

**Đánh giá level:**
- Junior: Phân biệt được vai trò cơ bản
- Mid: Sử dụng thành thạo, viết custom Guard/Interceptor/Pipe
- Senior: Kết hợp chúng để xây dựng cross-cutting concerns phức tạp, hiểu execution context

---

## 3. HTTP / RESTful API

### Câu 3.1
**Câu hỏi:** Khi thiết kế API theo kiểu RESTful, bạn đặt tên đường dẫn (URL) và chọn phương thức HTTP (GET, POST, PUT, PATCH, DELETE) theo nguyên tắc nào? Cho ví dụ với một tài nguyên "đơn hàng" (order)?

**Câu hỏi mở rộng:**
- Phân biệt PUT và PATCH? Khi nào dùng cái nào?
- Khi một hành động không phải là thao tác CRUD (ví dụ: hủy đơn hàng, duyệt đơn hàng), bạn thiết kế URL thế nào?

**Cách trả lời tốt:**
- URL dùng danh từ số nhiều: `/orders`, `/orders/:id`
- GET: lấy dữ liệu, POST: tạo mới, PUT: cập nhật toàn bộ, PATCH: cập nhật một phần, DELETE: xóa
- PUT gửi toàn bộ đối tượng, PATCH chỉ gửi phần thay đổi
- Hành động đặc biệt: `POST /orders/:id/cancel` hoặc `PATCH /orders/:id/status`
- Phân trang: `/orders?page=1&limit=20`
- Senior: Versioning API (v1/v2), HATEOAS, idempotency

**Đánh giá level:**
- Junior: Biết các phương thức HTTP cơ bản, đặt URL hợp lý
- Mid: Thiết kế API nhất quán, xử lý edge case, pagination, filtering
- Senior: API versioning strategy, backward compatibility, API documentation (Swagger/OpenAPI)

---

### Câu 3.2
**Câu hỏi:** Bạn xử lý mã trạng thái HTTP (status code) như thế nào trong API? Khi nào trả về 200, 201, 400, 401, 403, 404, 500? Bạn có format chung cho response không?

**Câu hỏi mở rộng:**
- Khi validate dữ liệu đầu vào thất bại, bạn trả về response như thế nào để phía giao diện (frontend) dễ xử lý?
- Bạn xử lý lỗi chung (global error handling) trong NestJS thế nào?

**Cách trả lời tốt:**
- 200: thành công chung, 201: tạo mới thành công, 204: xóa thành công (không có nội dung)
- 400: dữ liệu đầu vào sai, 401: chưa đăng nhập, 403: không có quyền, 404: không tìm thấy
- 422: dữ liệu hợp lệ về mặt cú pháp nhưng không xử lý được
- 500: lỗi hệ thống không mong muốn
- Response format nhất quán: `{ success, data, error: { code, message, details } }`
- NestJS: ExceptionFilter để bắt và format lỗi toàn cục
- Senior: Error code mapping, i18n error messages, correlation ID

**Đánh giá level:**
- Junior: Biết các status code phổ biến
- Mid: Thiết kế error response format, implement global exception filter
- Senior: Chiến lược error handling toàn diện, monitoring, alerting dựa trên error rate

---

### Câu 3.3
**Câu hỏi:** Bạn có biết những cách nào để tối ưu hiệu suất của API không? Ví dụ khi một API trả về dữ liệu rất chậm, bạn sẽ kiểm tra và cải thiện như thế nào?

**Câu hỏi mở rộng:**
- Bạn đã dùng bộ nhớ đệm (caching) ở tầng API chưa? HTTP caching hoạt động thế nào?
- Rate limiting là gì và bạn triển khai nó trong NestJS như thế nào?

**Cách trả lời tốt:**
- Kiểm tra: profiling query database, kiểm tra N+1 query, kiểm tra network latency
- Tối ưu database: thêm index, optimize query, pagination
- Caching: Redis cache, HTTP cache headers (ETag, Cache-Control)
- Response optimization: chỉ trả về field cần thiết, compression (gzip)
- Rate limiting: @nestjs/throttler, bảo vệ API khỏi lạm dụng
- Senior: CDN, connection pooling, lazy loading, GraphQL cho flexible queries

**Đánh giá level:**
- Junior: Biết một số kỹ thuật cơ bản (pagination, index)
- Mid: Áp dụng được nhiều kỹ thuật, đo lường hiệu suất
- Senior: Thiết kế chiến lược tối ưu toàn diện, load testing, capacity planning

---

### Câu 3.4
**Câu hỏi:** Ngoài REST, bạn có biết các giao thức hoặc kiểu giao tiếp nào khác giữa client và server không? Khi nào bạn chọn dùng chúng thay vì REST?

**Câu hỏi mở rộng:**
- WebSocket phù hợp cho những bài toán nào? Bạn đã triển khai WebSocket trong NestJS chưa?
- GraphQL có ưu nhược điểm gì so với REST?

**Cách trả lời tốt:**
- WebSocket: giao tiếp hai chiều thời gian thực (chat, notification, live update)
- GraphQL: client tự chọn dữ liệu cần lấy, giảm over-fetching/under-fetching
- gRPC: giao tiếp giữa các microservice, hiệu suất cao, dùng Protocol Buffer
- Server-Sent Events (SSE): server đẩy dữ liệu một chiều xuống client
- Senior: So sánh trade-off, khi nào dùng cái gì, hybrid approach

**Đánh giá level:**
- Junior: Biết tồn tại các giao thức khác, mô tả sơ lược
- Mid: Đã sử dụng ít nhất 1-2 giao thức khác trong dự án thực tế
- Senior: Chọn được giao thức phù hợp cho từng bài toán, triển khai production-ready

---

## 4. Async/Await

### Câu 4.1
**Câu hỏi:** Trong JavaScript/TypeScript, khi bạn gọi một hàm bất đồng bộ (async function), chuyện gì thực sự xảy ra bên dưới? Promise hoạt động thế nào?

**Câu hỏi mở rộng:**
- Sự khác nhau giữa callback, Promise, và async/await là gì? Tại sao async/await được ưa chuộng hơn?
- "Callback hell" là gì và async/await giải quyết vấn đề đó thế nào?

**Cách trả lời tốt:**
- Promise là một đối tượng đại diện cho kết quả của thao tác bất đồng bộ (pending → fulfilled/rejected)
- async/await là cú pháp giúp viết code bất đồng bộ giống code đồng bộ, dễ đọc hơn
- Bên dưới, async/await vẫn dùng Promise, chỉ là cú pháp đẹp hơn (syntactic sugar)
- Callback hell: callback lồng nhau nhiều tầng, khó đọc, khó debug
- Senior: Microtask queue, Promise.allSettled vs Promise.all, error propagation

**Đánh giá level:**
- Junior: Sử dụng được async/await, hiểu Promise cơ bản
- Mid: Giải thích được cơ chế hoạt động, xử lý lỗi trong async code
- Senior: Hiểu sâu về microtask queue, tối ưu async operations, xử lý memory leak

---

### Câu 4.2
**Câu hỏi:** Khi bạn cần gọi nhiều tác vụ bất đồng bộ cùng lúc (ví dụ gọi 3 API khác nhau), bạn có những cách nào? Sự khác nhau giữa `Promise.all`, `Promise.allSettled`, `Promise.race` là gì?

**Câu hỏi mở rộng:**
- Nếu bạn có 1000 tác vụ cần thực hiện nhưng không muốn chạy hết cùng lúc (sợ quá tải), bạn xử lý thế nào?
- Khi dùng `Promise.all`, nếu 1 trong 3 API bị lỗi thì chuyện gì xảy ra? Bạn xử lý thế nào?

**Cách trả lời tốt:**
- `Promise.all`: chạy song song, thất bại nếu bất kỳ Promise nào lỗi
- `Promise.allSettled`: chạy song song, trả về kết quả của tất cả (cả thành công và thất bại)
- `Promise.race`: trả về kết quả của Promise hoàn thành đầu tiên
- Giới hạn đồng thời: sử dụng thư viện như p-limit, p-queue, hoặc tự implement batching
- Senior: Promise.any, xử lý timeout cho Promise, cancellation pattern

**Đánh giá level:**
- Junior: Biết dùng Promise.all
- Mid: Phân biệt được các loại, xử lý lỗi đúng cách
- Senior: Implement batching/throttling, xử lý backpressure, timeout strategy

---

### Câu 4.3
**Câu hỏi:** Bạn đã gặp những lỗi phổ biến nào khi làm việc với async/await? Ví dụ: quên await, hay lỗi "unhandled promise rejection" xảy ra khi nào?

**Câu hỏi mở rộng:**
- Nếu trong vòng lặp `for` bạn gọi await, nó khác gì so với dùng `Promise.all` với `map`?
- Làm thế nào để debug một đoạn code bất đồng bộ phức tạp?

**Cách trả lời tốt:**
- Quên await: hàm trả về Promise thay vì giá trị thực, logic chạy sai
- Unhandled rejection: Promise bị reject nhưng không có catch, có thể crash app
- Await trong for loop: chạy tuần tự (chậm); Promise.all + map: chạy song song (nhanh)
- Lỗi: try/catch không bắt được lỗi của Promise không được await
- Debug: async stack trace, console.log tại các điểm quan trọng, debugger
- Senior: Memory leak do Promise không resolve, event listener leak, proper cleanup

**Đánh giá level:**
- Junior: Biết dùng try/catch với async/await
- Mid: Nhận biết và sửa được các lỗi phổ biến, tối ưu performance
- Senior: Thiết kế error handling strategy, graceful shutdown, resource cleanup

---

### Câu 4.4
**Câu hỏi:** Trong NestJS, bạn xử lý các tác vụ bất đồng bộ nặng (ví dụ: gửi email, tạo báo cáo PDF, xử lý ảnh) như thế nào mà không làm chậm API response?

**Câu hỏi mở rộng:**
- Bạn biết gì về hàng đợi công việc (job queue)? Bull/BullMQ hoạt động thế nào trong NestJS?
- Nếu tác vụ bất đồng bộ bị lỗi, bạn xử lý thế nào? Cơ chế thử lại (retry) hoạt động ra sao?

**Cách trả lời tốt:**
- Đưa tác vụ nặng vào hàng đợi (queue) thay vì xử lý trực tiếp trong request
- NestJS + Bull/BullMQ: tạo queue, producer đẩy job, consumer xử lý
- API trả về ngay response (202 Accepted), job chạy nền
- Retry strategy: số lần thử lại, thời gian chờ giữa các lần (exponential backoff)
- Senior: Dead letter queue, job priority, concurrency control, monitoring job

**Đánh giá level:**
- Junior: Hiểu khái niệm xử lý nền
- Mid: Triển khai được Bull queue trong NestJS, cấu hình retry
- Senior: Thiết kế hệ thống queue phức tạp, monitoring, scaling worker

---

## 5. Event Loop

### Câu 5.1
**Câu hỏi:** Bạn có thể giải thích vòng lặp sự kiện (event loop) trong Node.js hoạt động thế nào không? Tại sao Node.js chỉ dùng một luồng (single thread) mà vẫn xử lý được nhiều yêu cầu cùng lúc?

**Câu hỏi mở rộng:**
- Nếu bạn chạy một đoạn code tính toán nặng (ví dụ: mã hóa dữ liệu lớn) trên Node.js, chuyện gì xảy ra với các request khác?
- Các giai đoạn (phases) của event loop là gì?

**Cách trả lời tốt:**
- Node.js dùng mô hình single-threaded với event loop và non-blocking I/O
- Event loop liên tục kiểm tra có callback nào sẵn sàng để thực thi không
- I/O operations (đọc file, query database, gọi API) được chuyển cho hệ điều hành hoặc thread pool (libuv)
- Khi I/O hoàn thành, callback được đưa vào hàng đợi để event loop xử lý
- Các phase: timers → pending callbacks → idle/prepare → poll → check → close callbacks
- Senior: Microtask vs macrotask, process.nextTick vs setImmediate, starvation

**Đánh giá level:**
- Junior: Biết Node.js là single-threaded, giải thích cơ bản
- Mid: Hiểu các phase của event loop, biết tác động của blocking code
- Senior: Giải thích chi tiết microtask/macrotask, tối ưu event loop, monitoring event loop lag

---

### Câu 5.2
**Câu hỏi:** `process.nextTick()`, `setImmediate()`, và `setTimeout(fn, 0)` khác nhau thế nào? Thứ tự thực thi của chúng ra sao?

**Câu hỏi mở rộng:**
- Trong thực tế, bạn có dùng `process.nextTick()` không? Nếu lạm dụng nó thì xảy ra vấn đề gì?
- Microtask và macrotask khác nhau thế nào?

**Cách trả lời tốt:**
- `process.nextTick()`: chạy ngay sau operation hiện tại, trước khi event loop tiếp tục (microtask)
- `setImmediate()`: chạy ở phase "check" của event loop iteration tiếp theo
- `setTimeout(fn, 0)`: chạy ở phase "timers" của event loop iteration tiếp theo
- Thứ tự: nextTick → Promise.then → setImmediate/setTimeout (tùy context)
- Lạm dụng nextTick: block event loop, starve I/O callbacks
- Senior: Recursive nextTick vs recursive setImmediate, performance implications

**Đánh giá level:**
- Junior: Biết sự tồn tại, không phân biệt rõ
- Mid: Giải thích được thứ tự thực thi, biết khi nào dùng cái nào
- Senior: Hiểu sâu về microtask queue, debugging event loop issues

---

### Câu 5.3
**Câu hỏi:** Khi bạn phát hiện ứng dụng Node.js bị chậm dần hoặc "đơ" (không phản hồi), bạn nghĩ nguyên nhân có thể là gì liên quan đến vòng lặp sự kiện? Bạn sẽ kiểm tra và sửa thế nào?

**Câu hỏi mở rộng:**
- Bạn biết cách nào để đo "độ trễ event loop" (event loop lag)?
- Worker thread trong Node.js dùng khi nào?

**Cách trả lời tốt:**
- Nguyên nhân: CPU-intensive operation chạy trên main thread, synchronous I/O, blocking code
- Kiểm tra: event loop lag monitoring, profiling CPU usage, flame graph
- Giải pháp: đưa tác vụ nặng sang worker thread, child process, hoặc queue
- Worker thread: tính toán nặng (crypto, image processing, data parsing)
- Tools: clinic.js, 0x, node --prof, built-in profiler
- Senior: Heap snapshot, memory profiling, event loop utilization metric

**Đánh giá level:**
- Junior: Biết khái niệm blocking, đưa ra giải pháp cơ bản
- Mid: Sử dụng được profiling tools, triển khai worker thread
- Senior: Monitoring event loop trong production, performance tuning, capacity planning

---

## 6. Parallelism vs Concurrency

### Câu 6.1
**Câu hỏi:** Bạn phân biệt thế nào giữa "xử lý đồng thời" (concurrency) và "xử lý song song" (parallelism)? Node.js hỗ trợ cái nào?

**Câu hỏi mở rộng:**
- Nếu máy chủ có 8 lõi CPU, Node.js mặc định chỉ dùng 1 lõi. Làm sao để tận dụng hết các lõi?
- Cluster mode trong Node.js/PM2 hoạt động thế nào?

**Cách trả lời tốt:**
- Concurrency: xử lý nhiều tác vụ trong cùng khoảng thời gian bằng cách chuyển đổi qua lại (Node.js event loop)
- Parallelism: xử lý nhiều tác vụ đồng thời trên nhiều CPU/core (worker threads, cluster)
- Node.js mặc định là concurrent (single thread + event loop), không phải parallel
- Cluster mode: fork nhiều process, mỗi process chạy trên 1 core, load balancer phân phối request
- PM2: quản lý cluster dễ dàng, auto-restart, monitoring
- Senior: Shared memory (SharedArrayBuffer), Atomics, khi nào cần parallel thực sự

**Đánh giá level:**
- Junior: Phân biệt được khái niệm
- Mid: Cấu hình cluster mode, sử dụng PM2
- Senior: Thiết kế hệ thống tận dụng multi-core, worker thread communication, scaling strategy

---

### Câu 6.2
**Câu hỏi:** Khi bạn cần xử lý một lượng lớn dữ liệu (ví dụ import file CSV 1 triệu dòng), bạn sẽ thiết kế luồng xử lý như thế nào để không ảnh hưởng đến các API khác?

**Câu hỏi mở rộng:**
- Bạn biết gì về Stream trong Node.js? Nó giúp gì trong trường hợp này?
- Nếu xử lý mỗi dòng cần gọi database, bạn tối ưu thế nào?

**Cách trả lời tốt:**
- Dùng Stream để đọc file từng phần (chunk), không load toàn bộ vào bộ nhớ
- Đưa tác vụ vào queue, worker xử lý nền
- Batch insert: gom nhiều dòng thành 1 lần insert thay vì insert từng dòng
- Giới hạn concurrency: xử lý tối đa N dòng cùng lúc
- Progress tracking: lưu tiến độ để resume nếu bị gián đoạn
- Senior: Backpressure handling, Transform stream, pipeline API, database transaction cho batch

**Đánh giá level:**
- Junior: Đề cập được Stream hoặc queue
- Mid: Triển khai được Stream + batch processing, xử lý backpressure
- Senior: Thiết kế pipeline hoàn chỉnh với error handling, resume, monitoring

---

### Câu 6.3
**Câu hỏi:** Trong môi trường microservice, khi nhiều instance của cùng một dịch vụ đang chạy, làm thế nào để đảm bảo chúng không xử lý trùng lặp cùng một tác vụ?

**Câu hỏi mở rộng:**
- Bạn biết gì về distributed lock? Redis lock (Redlock) hoạt động thế nào?
- Idempotency key là gì và nó giải quyết vấn đề gì?

**Cách trả lời tốt:**
- Distributed lock: dùng Redis SET NX EX để tạo khóa phân tán, chỉ 1 instance được xử lý
- Redlock algorithm: tạo lock trên nhiều Redis node để đảm bảo an toàn
- Message queue: consumer group trong Kafka đảm bảo mỗi message chỉ được 1 consumer xử lý
- Idempotency key: client gửi key duy nhất, server kiểm tra trước khi xử lý để tránh trùng
- Database: dùng unique constraint, optimistic locking (version column)
- Senior: Fencing token, lock timeout strategy, compare-and-swap

**Đánh giá level:**
- Junior: Biết vấn đề tồn tại, đề cập database lock
- Mid: Triển khai được Redis lock, hiểu consumer group
- Senior: Thiết kế hệ thống phân tán an toàn, xử lý edge case (lock expiration, split brain)

---

## 7. Authentication / Authorization

### Câu 7.1
**Câu hỏi:** Bạn phân biệt thế nào giữa "xác thực" (authentication - xác minh bạn là ai) và "phân quyền" (authorization - xác minh bạn được làm gì)? Bạn triển khai chúng trong NestJS thế nào?

**Câu hỏi mở rộng:**
- JWT hoạt động thế nào? Nó gồm những phần nào?
- Khi người dùng đổi mật khẩu hoặc bị khóa tài khoản, làm sao để JWT đang hoạt động bị vô hiệu hóa ngay?

**Cách trả lời tốt:**
- Authentication: xác minh danh tính (đăng nhập, JWT verify)
- Authorization: kiểm tra quyền (role-based, permission-based)
- JWT: Header (algorithm) + Payload (claims) + Signature, stateless
- NestJS: Passport strategy cho authentication, custom Guard cho authorization
- Vô hiệu hóa JWT: blacklist trong Redis (kèm TTL bằng thời gian hết hạn token), token versioning
- Senior: Refresh token rotation, OAuth2 flow, RBAC vs ABAC

**Đánh giá level:**
- Junior: Phân biệt được authentication/authorization, dùng JWT cơ bản
- Mid: Triển khai đầy đủ auth flow, refresh token, role-based access
- Senior: Thiết kế hệ thống phân quyền phức tạp, OAuth2/SSO, security best practices

---

### Câu 7.2
**Câu hỏi:** Access token và refresh token khác nhau thế nào? Tại sao cần dùng cả hai thay vì chỉ dùng một token có thời hạn dài?

**Câu hỏi mở rộng:**
- Refresh token nên lưu ở đâu (cookie hay localStorage)? Tại sao?
- Bạn biết gì về "xoay vòng refresh token" (refresh token rotation)? Nó bảo vệ khỏi tấn công gì?

**Cách trả lời tốt:**
- Access token: thời hạn ngắn (15-30 phút), gửi kèm mỗi request, stateless
- Refresh token: thời hạn dài (7-30 ngày), chỉ dùng để lấy access token mới
- Lý do tách: access token bị lộ thì thiệt hại giới hạn (hết hạn nhanh), refresh token được bảo vệ tốt hơn
- Lưu trữ: refresh token trong httpOnly cookie (chống XSS), access token trong memory
- Refresh token rotation: mỗi lần dùng refresh token, tạo token mới và vô hiệu hóa token cũ
- Senior: Phát hiện token bị đánh cắp qua rotation, family invalidation

**Đánh giá level:**
- Junior: Biết khái niệm access/refresh token
- Mid: Triển khai đầy đủ token flow, biết cách lưu trữ an toàn
- Senior: Refresh token rotation, threat modeling, security audit

---

### Câu 7.3
**Câu hỏi:** Bạn triển khai phân quyền theo vai trò (role-based) trong NestJS thế nào? Ví dụ: admin được làm mọi thứ, user chỉ được xem và sửa dữ liệu của mình?

**Câu hỏi mở rộng:**
- Nếu hệ thống cần phân quyền chi tiết hơn (ví dụ: user A được sửa bài viết của mình nhưng không được sửa bài viết của user B), bạn thiết kế thế nào?
- CASL hoặc các thư viện phân quyền khác có giúp gì không?

**Cách trả lời tốt:**
- Role-based: gán role cho user, Guard kiểm tra role trước khi cho phép truy cập
- Custom decorator `@Roles('admin')` + RolesGuard kiểm tra metadata
- Resource-based: kiểm tra ownership (user chỉ được truy cập resource của mình)
- CASL: thư viện phân quyền linh hoạt, định nghĩa ability (can/cannot) cho từng role
- Database: bảng roles, permissions, role_permissions, user_roles
- Senior: ABAC (attribute-based), policy pattern, permission caching

**Đánh giá level:**
- Junior: Implement basic role check
- Mid: Thiết kế RBAC system, resource-based authorization
- Senior: Complex permission system, CASL integration, permission inheritance

---

### Câu 7.4
**Câu hỏi:** Bạn biết những lỗ hổng bảo mật phổ biến nào khi xây dựng API? Bạn phòng chống chúng thế nào?

**Câu hỏi mở rộng:**
- SQL Injection và NoSQL Injection khác nhau thế nào?
- CORS là gì và bạn cấu hình nó trong NestJS thế nào?

**Cách trả lời tốt:**
- SQL/NoSQL Injection: dùng parameterized query, ORM, validate input
- XSS: sanitize output, Content Security Policy header
- CSRF: SameSite cookie, CSRF token
- CORS: chỉ cho phép domain tin cậy, cấu hình origin, methods, headers
- Rate limiting: chống brute force, DDoS
- Helmet: thêm security headers
- Senior: OWASP Top 10, security audit, penetration testing, secret management

**Đánh giá level:**
- Junior: Biết 1-2 lỗ hổng phổ biến
- Mid: Áp dụng được các biện pháp phòng chống cơ bản
- Senior: Security mindset, threat modeling, defense in depth

---

## 8. Error Handling / Logging

### Câu 8.1
**Câu hỏi:** Bạn thiết kế hệ thống xử lý lỗi (error handling) trong ứng dụng NestJS thế nào? Bạn phân biệt giữa lỗi nghiệp vụ (business error) và lỗi kỹ thuật (technical error) không?

**Câu hỏi mở rộng:**
- Bạn có tạo các lớp lỗi riêng (custom exception class) không? Cấu trúc chúng thế nào?
- Global exception filter trong NestJS hoạt động thế nào? Bạn customize nó ra sao?

**Cách trả lời tốt:**
- Business error: lỗi logic nghiệp vụ (đơn hàng hết hàng, số dư không đủ) → trả về message có ý nghĩa cho user
- Technical error: lỗi hệ thống (database timeout, service unavailable) → log chi tiết, trả về message chung
- Custom exception: extends HttpException hoặc tạo base exception class
- Exception filter: bắt tất cả exception, format response nhất quán, log error
- Error code system: mỗi loại lỗi có mã riêng để frontend/mobile xử lý
- Senior: Error boundary, circuit breaker pattern, graceful degradation

**Đánh giá level:**
- Junior: Dùng try/catch cơ bản, throw HttpException
- Mid: Custom exception hierarchy, global exception filter, error code
- Senior: Comprehensive error strategy, error tracking (Sentry), alerting, graceful degradation

---

### Câu 8.2
**Câu hỏi:** Bạn tổ chức hệ thống ghi log (logging) thế nào? Bạn log những thông tin gì và dùng công cụ nào?

**Câu hỏi mở rộng:**
- Các mức độ log (log level) như debug, info, warn, error dùng khi nào?
- Trong hệ thống microservice, làm thế nào để theo dõi một request đi qua nhiều dịch vụ?

**Cách trả lời tốt:**
- Log levels: debug (development), info (thông tin chung), warn (cảnh báo), error (lỗi cần xử lý)
- Structured logging: log dạng JSON để dễ tìm kiếm và phân tích
- Thông tin cần log: timestamp, request ID, user ID, action, duration, error stack
- Tools: Winston, Pino (performance tốt hơn), NestJS Logger
- Centralized logging: ELK stack (Elasticsearch + Logstash + Kibana), CloudWatch
- Distributed tracing: correlation ID/request ID xuyên suốt các service
- Senior: OpenTelemetry, Jaeger/Zipkin, log aggregation, log retention policy

**Đánh giá level:**
- Junior: Dùng console.log, biết log level cơ bản
- Mid: Structured logging, centralized log, correlation ID
- Senior: Distributed tracing, observability (logs + metrics + traces), log analysis

---

### Câu 8.3
**Câu hỏi:** Khi ứng dụng đang chạy trên môi trường thật (production) gặp lỗi, bạn phát hiện và xử lý thế nào? Bạn có quy trình nào cho việc này không?

**Câu hỏi mở rộng:**
- Bạn đã dùng công cụ theo dõi lỗi (error tracking) nào chưa? Sentry hoạt động thế nào?
- Health check endpoint là gì và tại sao cần nó?

**Cách trả lời tốt:**
- Error tracking: Sentry, Bugsnag - tự động bắt và phân loại lỗi
- Alerting: thông báo qua Slack/email khi error rate tăng bất thường
- Health check: endpoint `/health` kiểm tra trạng thái app, database, Redis, external services
- NestJS Terminus: module hỗ trợ health check
- Monitoring: metrics (response time, error rate, memory usage) qua Prometheus + Grafana
- Incident response: runbook, on-call rotation, post-mortem
- Senior: SLO/SLA, error budget, canary deployment, rollback strategy

**Đánh giá level:**
- Junior: Kiểm tra log thủ công
- Mid: Sử dụng error tracking tool, cấu hình health check, alerting
- Senior: Xây dựng observability platform, incident management process, SRE practices

---

## 9. Database

### Câu 9.1
**Câu hỏi:** Bạn chọn giữa cơ sở dữ liệu quan hệ (PostgreSQL/MySQL) và phi quan hệ (MongoDB) dựa trên tiêu chí nào? Cho ví dụ bài toán phù hợp với từng loại?

**Câu hỏi mở rộng:**
- Trong cùng một dự án, bạn có dùng cả hai loại không? Khi nào?
- ACID trong database quan hệ là gì? MongoDB có hỗ trợ không?

**Cách trả lời tốt:**
- SQL: dữ liệu có quan hệ rõ ràng, cần consistency (tài chính, đơn hàng, quản lý user)
- MongoDB: dữ liệu linh hoạt, schema thay đổi thường xuyên, nested document (CMS, log, catalog sản phẩm đa dạng)
- Polyglot persistence: dùng SQL cho core business, MongoDB cho flexible data, Redis cho cache
- ACID: Atomicity, Consistency, Isolation, Durability - đảm bảo giao dịch an toàn
- MongoDB: hỗ trợ transaction từ version 4.0, nhưng performance kém hơn SQL cho transaction phức tạp
- Senior: CAP theorem, eventual consistency, database per service pattern

**Đánh giá level:**
- Junior: Biết sự khác biệt cơ bản
- Mid: Đưa ra quyết định có lý do, hiểu ACID, sử dụng cả hai
- Senior: Polyglot persistence strategy, CAP theorem, data modeling cho microservice

---

### Câu 9.2
**Câu hỏi:** Bạn sử dụng ORM (TypeORM, Prisma, Mongoose) thế nào trong NestJS? ORM có nhược điểm gì không? Khi nào bạn viết truy vấn thô (raw query)?

**Câu hỏi mở rộng:**
- Vấn đề N+1 query là gì? Bạn phát hiện và giải quyết nó thế nào?
- Migration trong database là gì? Bạn quản lý schema change thế nào?

**Cách trả lời tốt:**
- ORM: mapping giữa object và table, giảm boilerplate, type-safe query
- Nhược điểm: query phức tạp khó viết, performance không tối ưu bằng raw query, learning curve
- Raw query: report phức tạp, aggregate query, performance-critical operation
- N+1: lấy danh sách user → mỗi user query thêm 1 lần lấy orders → N+1 queries. Fix: eager loading, join, dataloader
- Migration: version control cho schema, up/down script, không sửa trực tiếp database
- Senior: Query builder vs raw query benchmark, ORM cache layer, read replica

**Đánh giá level:**
- Junior: Sử dụng ORM cơ bản, CRUD operations
- Mid: Xử lý N+1, viết migration, complex query
- Senior: Optimize ORM performance, query analysis, migration strategy cho zero-downtime deployment

---

### Câu 9.3
**Câu hỏi:** Khi truy vấn database bị chậm, bạn phân tích và tối ưu thế nào? Bạn biết gì về chỉ mục (index) trong database?

**Câu hỏi mở rộng:**
- Tại sao không đánh index cho tất cả các cột? Nhược điểm của việc quá nhiều index là gì?
- EXPLAIN plan trong PostgreSQL/MySQL cho bạn biết thông tin gì?

**Cách trả lời tốt:**
- Phân tích: dùng EXPLAIN/EXPLAIN ANALYZE để xem query plan
- Kiểm tra: full table scan, index scan, join type
- Index: B-tree (mặc định, phù hợp range query), hash (equality), GIN (full-text search), composite index
- Nhược điểm index: tốn bộ nhớ, làm chậm write operations (INSERT, UPDATE, DELETE)
- Tối ưu: chỉ index cột hay query, composite index theo thứ tự selectivity
- Senior: Partial index, covering index, index-only scan, query plan caching, connection pool tuning

**Đánh giá level:**
- Junior: Biết khái niệm index, tạo index cơ bản
- Mid: Đọc EXPLAIN plan, chọn đúng loại index, composite index
- Senior: Advanced indexing strategy, query optimization, database tuning

---

### Câu 9.4
**Câu hỏi:** Transaction trong database là gì? Khi nào bạn cần dùng transaction? Cho ví dụ thực tế trong dự án?

**Câu hỏi mở rộng:**
- Các mức độ cô lập (isolation level) là gì? Dirty read, phantom read khác nhau thế nào?
- Trong hệ thống phân tán (nhiều database), bạn xử lý transaction thế nào?

**Cách trả lời tốt:**
- Transaction: nhóm nhiều thao tác thành một đơn vị, hoặc tất cả thành công hoặc tất cả hoàn tác
- Ví dụ: chuyển tiền (trừ tài khoản A + cộng tài khoản B), đặt hàng (tạo order + giảm inventory)
- Isolation levels: Read Uncommitted → Read Committed → Repeatable Read → Serializable
- Dirty read: đọc dữ liệu chưa commit; Phantom read: dữ liệu thay đổi giữa 2 lần đọc trong cùng transaction
- NestJS + TypeORM: EntityManager.transaction() hoặc QueryRunner
- Senior: Distributed transaction (Saga pattern, 2PC), eventual consistency, compensating transaction

**Đánh giá level:**
- Junior: Biết khái niệm transaction, dùng cơ bản
- Mid: Hiểu isolation levels, implement transaction trong NestJS
- Senior: Distributed transaction, Saga pattern, handling partial failure

---

### Câu 9.5
**Câu hỏi:** Khi dữ liệu trong bảng tăng lên hàng triệu, hàng chục triệu dòng, bạn xử lý thế nào để ứng dụng vẫn hoạt động tốt?

**Câu hỏi mở rộng:**
- Bạn biết gì về phân vùng bảng (table partitioning)? Khi nào nên dùng?
- Read replica là gì? Bạn cấu hình trong NestJS thế nào?

**Cách trả lời tốt:**
- Indexing: đúng cột, đúng loại
- Pagination: cursor-based thay vì offset-based cho dữ liệu lớn
- Table partitioning: chia bảng theo thời gian, khu vực - query nhanh hơn vì chỉ scan partition liên quan
- Archiving: di chuyển dữ liệu cũ sang bảng/database riêng
- Read replica: tách read/write, write vào master, read từ replica
- Caching: cache kết quả query phổ biến vào Redis
- Senior: Sharding, materialized view, CQRS, database per service

**Đánh giá level:**
- Junior: Đề cập indexing, pagination
- Mid: Partitioning, read replica, archiving strategy
- Senior: Sharding strategy, CQRS, database scaling architecture

---

## 10. Cloud Service

### Câu 10.1
**Câu hỏi:** Bạn đã triển khai ứng dụng Node.js trên dịch vụ đám mây (AWS/GCP) chưa? Bạn dùng những dịch vụ nào và tại sao chọn chúng?

**Câu hỏi mở rộng:**
- Sự khác nhau giữa EC2/Compute Engine, ECS/Cloud Run, và Lambda/Cloud Functions là gì? Khi nào dùng cái nào?
- Bạn xử lý biến môi trường (environment variables) và bí mật (secrets) trên cloud thế nào?

**Cách trả lời tốt:**
- EC2/Compute Engine: máy ảo, toàn quyền kiểm soát, phù hợp ứng dụng truyền thống
- ECS/Cloud Run: chạy container, tự động scale, không cần quản lý server
- Lambda/Cloud Functions: serverless, chạy theo sự kiện, trả tiền theo lần gọi
- Secret management: AWS Secrets Manager, Parameter Store, GCP Secret Manager
- Environment: tách config theo môi trường (dev, staging, production)
- Senior: Infrastructure as Code (Terraform/CDK), cost optimization, multi-region deployment

**Đánh giá level:**
- Junior: Biết các dịch vụ cơ bản, đã deploy lên cloud
- Mid: Chọn được dịch vụ phù hợp, cấu hình auto-scaling, quản lý secrets
- Senior: Thiết kế cloud architecture, IaC, cost optimization, disaster recovery

---

### Câu 10.2
**Câu hỏi:** Docker là gì và tại sao bạn dùng nó? Bạn viết Dockerfile cho ứng dụng NestJS thế nào?

**Câu hỏi mở rộng:**
- Multi-stage build là gì? Tại sao nên dùng?
- Docker Compose dùng khi nào? Bạn cấu hình nó thế nào cho môi trường phát triển?

**Cách trả lời tốt:**
- Docker: đóng gói ứng dụng + dependencies vào container, đảm bảo chạy giống nhau mọi nơi
- Dockerfile: FROM node → COPY package.json → npm install → COPY source → BUILD → CMD
- Multi-stage build: stage 1 build app, stage 2 chỉ copy artifact → image nhỏ hơn, bảo mật hơn
- Docker Compose: chạy nhiều container cùng lúc (app + database + Redis), tiện cho development
- Best practices: .dockerignore, non-root user, health check, layer caching
- Senior: Docker networking, volume management, security scanning, container orchestration

**Đánh giá level:**
- Junior: Viết được Dockerfile cơ bản, dùng Docker Compose
- Mid: Multi-stage build, optimize image size, Docker best practices
- Senior: Container orchestration (K8s), security hardening, CI/CD với Docker

---

### Câu 10.3
**Câu hỏi:** Bạn triển khai CI/CD (tích hợp liên tục / triển khai liên tục) thế nào? Pipeline của bạn gồm những bước nào?

**Câu hỏi mở rộng:**
- Bạn dùng công cụ CI/CD nào? (GitHub Actions, GitLab CI, Jenkins, CircleCI)
- Làm thế nào để triển khai mà không gây gián đoạn dịch vụ (zero-downtime deployment)?

**Cách trả lời tốt:**
- CI pipeline: lint → test → build → security scan → push Docker image
- CD pipeline: deploy to staging → integration test → deploy to production
- Tools: GitHub Actions (phổ biến, miễn phí cho open source), GitLab CI (self-hosted)
- Zero-downtime: rolling update, blue-green deployment, canary deployment
- Environment: dev → staging → production, mỗi environment có config riêng
- Senior: GitOps (ArgoCD), feature flags, progressive delivery, rollback automation

**Đánh giá level:**
- Junior: Biết khái niệm CI/CD, đã dùng 1 tool
- Mid: Thiết kế pipeline hoàn chỉnh, auto deploy
- Senior: Advanced deployment strategies, GitOps, monitoring deployment health

---

### Câu 10.4
**Câu hỏi:** Bạn lưu trữ file (ảnh, tài liệu) do người dùng tải lên ở đâu? Tại sao không lưu trực tiếp trên server?

**Câu hỏi mở rộng:**
- Pre-signed URL là gì và hoạt động thế nào?
- Bạn xử lý thế nào khi cần resize ảnh hoặc convert định dạng file?

**Cách trả lời tốt:**
- Lưu trên Object Storage: AWS S3, GCP Cloud Storage - scalable, durable, cost-effective
- Không lưu trên server: server có thể scale (thêm/bớt instance), file sẽ mất hoặc không đồng bộ
- Pre-signed URL: tạo URL có thời hạn để client upload/download trực tiếp từ S3, giảm tải server
- CDN: CloudFront/Cloud CDN phía trước S3 để serve file nhanh hơn
- Image processing: Lambda/Cloud Function trigger khi upload, resize/convert rồi lưu lại
- Senior: Lifecycle policy, storage class (Standard, IA, Glacier), multipart upload, virus scanning

**Đánh giá level:**
- Junior: Biết dùng S3, upload cơ bản
- Mid: Pre-signed URL, CDN integration, file processing pipeline
- Senior: Storage architecture, cost optimization, security (encryption, access policy)

---

## 11. Cache

### Câu 11.1
**Câu hỏi:** Bộ nhớ đệm (cache) giải quyết vấn đề gì? Bạn dùng Redis làm cache trong NestJS thế nào? Cho ví dụ cụ thể?

**Câu hỏi mở rộng:**
- Cache ở tầng nào? (Application cache, HTTP cache, Database cache, CDN cache)
- Khi nào KHÔNG nên dùng cache?

**Cách trả lời tốt:**
- Cache giảm thời gian truy xuất, giảm tải cho database
- Redis: in-memory data store, hỗ trợ nhiều kiểu dữ liệu, TTL
- Ví dụ: cache danh sách sản phẩm phổ biến, cache user session, cache API response
- Tầng cache: Browser → CDN → API Gateway → Application (Redis) → Database query cache
- Không nên cache: dữ liệu thay đổi liên tục, dữ liệu nhạy cảm, dữ liệu cần chính xác tuyệt đối
- NestJS: CacheModule với cache-manager, hoặc trực tiếp dùng ioredis
- Senior: Cache warming, cache stampede prevention, multi-level cache

**Đánh giá level:**
- Junior: Biết khái niệm cache, dùng Redis cơ bản
- Mid: Implement cache layer trong NestJS, chọn đúng dữ liệu để cache
- Senior: Multi-level caching strategy, cache performance tuning

---

### Câu 11.2
**Câu hỏi:** Khi dữ liệu trong database thay đổi, làm sao để cache không trả về dữ liệu cũ (stale data)? Bạn biết những chiến lược cập nhật cache nào?

**Câu hỏi mở rộng:**
- Cache-aside, Write-through, Write-behind khác nhau thế nào?
- "Bão cache" (cache stampede/thundering herd) là gì và bạn phòng chống thế nào?

**Cách trả lời tốt:**
- Cache-aside (Lazy loading): đọc cache trước, miss thì đọc DB rồi set cache. Ưu: đơn giản. Nhược: cache miss đầu tiên chậm
- Write-through: ghi DB và cache cùng lúc. Ưu: cache luôn mới. Nhược: write chậm hơn
- Write-behind: ghi cache trước, ghi DB bất đồng bộ sau. Ưu: write nhanh. Nhược: risk mất dữ liệu
- Cache invalidation: xóa cache khi data thay đổi, set TTL hợp lý
- Cache stampede: nhiều request cùng miss cache → đổ dồn vào DB. Fix: lock mechanism, stale-while-revalidate
- Senior: Cache invalidation patterns, distributed cache consistency, probabilistic early expiration

**Đánh giá level:**
- Junior: Biết set TTL, xóa cache khi update
- Mid: Implement cache-aside, xử lý cache invalidation
- Senior: Chọn đúng cache strategy cho từng use case, xử lý cache stampede, distributed cache

---

### Câu 11.3
**Câu hỏi:** Ngoài dùng làm cache, Redis còn dùng được cho những bài toán nào khác? Bạn đã dùng Redis cho mục đích nào ngoài cache chưa?

**Câu hỏi mở rộng:**
- Pub/Sub trong Redis hoạt động thế nào? Nó khác gì so với message queue như Kafka?
- Redis có thể thay thế database chính không? Tại sao?

**Cách trả lời tốt:**
- Session storage: lưu phiên đăng nhập, nhanh hơn database
- Rate limiting: dùng INCR + EXPIRE để đếm request trong khoảng thời gian
- Distributed lock: SET NX EX cho mutual exclusion
- Leaderboard/Ranking: Sorted Set
- Pub/Sub: real-time notification, event broadcasting
- Queue: Redis List như simple queue (LPUSH + BRPOP), Bull/BullMQ dùng Redis làm backend
- Redis vs Kafka: Redis Pub/Sub fire-and-forget (không lưu message), Kafka lưu message có thể replay
- Senior: Redis Streams, Redis Cluster, persistence (RDB, AOF), memory management

**Đánh giá level:**
- Junior: Dùng Redis chủ yếu cho cache
- Mid: Sử dụng 2-3 use case khác (session, rate limiting, pub/sub)
- Senior: Advanced Redis patterns, cluster management, memory optimization

---

## 12. Message Queue

### Câu 12.1
**Câu hỏi:** Hàng đợi tin nhắn (message queue) giải quyết vấn đề gì? Tại sao các hệ thống lớn cần dùng nó thay vì gọi trực tiếp giữa các dịch vụ?

**Câu hỏi mở rộng:**
- Giao tiếp đồng bộ (synchronous) và bất đồng bộ (asynchronous) giữa các dịch vụ khác nhau thế nào?
- Nếu service B đang chết mà service A cần gửi dữ liệu cho nó, message queue giúp gì?

**Cách trả lời tốt:**
- Decoupling: các dịch vụ không cần biết nhau, chỉ cần biết topic/queue
- Resilience: nếu consumer chết, message vẫn nằm trong queue, xử lý sau khi phục hồi
- Load leveling: điều phối lượng request lớn, consumer xử lý theo khả năng
- Async processing: không cần chờ kết quả ngay, trả về response nhanh
- Ví dụ: đặt hàng → gửi message → email service xử lý gửi email, inventory service cập nhật kho
- Senior: Event-driven architecture, choreography vs orchestration, exactly-once delivery

**Đánh giá level:**
- Junior: Biết khái niệm queue, lý do sử dụng
- Mid: Triển khai producer/consumer trong dự án, xử lý error
- Senior: Event-driven architecture design, message patterns, reliability guarantees

---

### Câu 12.2
**Câu hỏi:** Kafka hoạt động thế nào? Bạn giải thích các khái niệm: topic, partition, consumer group, offset?

**Câu hỏi mở rộng:**
- Tại sao Kafka cần partition? Nó giúp gì cho việc mở rộng (scaling)?
- Kafka đảm bảo thứ tự tin nhắn (message ordering) thế nào?

**Cách trả lời tốt:**
- Topic: kênh phân loại message, mỗi loại sự kiện 1 topic (order-created, payment-completed)
- Partition: chia topic thành nhiều phần, cho phép xử lý song song
- Consumer group: nhóm consumer, mỗi partition chỉ được 1 consumer trong group xử lý → không trùng lặp
- Offset: vị trí đọc của consumer, cho phép replay message
- Ordering: đảm bảo thứ tự trong cùng 1 partition (dùng message key để route cùng entity vào cùng partition)
- Senior: Replication factor, ISR, acks configuration, exactly-once semantics

**Đánh giá level:**
- Junior: Biết Kafka là message broker, mô tả sơ lược
- Mid: Hiểu rõ các khái niệm, triển khai producer/consumer trong NestJS
- Senior: Kafka cluster management, performance tuning, exactly-once delivery, schema registry

---

### Câu 12.3
**Câu hỏi:** Khi consumer xử lý message bị lỗi, bạn xử lý thế nào? Message bị mất hoặc bị xử lý trùng lặp thì sao?

**Câu hỏi mở rộng:**
- Dead letter queue (hàng đợi tin nhắn chết) là gì? Khi nào cần dùng?
- Idempotent consumer là gì? Bạn triển khai thế nào?

**Cách trả lời tốt:**
- Retry: thử lại với exponential backoff (1s, 2s, 4s, 8s...)
- Dead letter queue: sau N lần retry thất bại, chuyển message sang DLQ để xem xét thủ công
- Message loss: cấu hình acks=all, replication factor > 1, consumer commit offset sau khi xử lý xong
- Duplicate processing: idempotent consumer - kiểm tra message đã xử lý chưa (deduplication table, idempotency key)
- At-least-once vs at-most-once vs exactly-once delivery semantics
- Senior: Outbox pattern (đảm bảo message được gửi khi DB transaction commit), saga pattern

**Đánh giá level:**
- Junior: Biết retry cơ bản
- Mid: Implement retry + DLQ, hiểu delivery semantics
- Senior: Exactly-once processing, outbox pattern, distributed transaction handling

---

### Câu 12.4
**Câu hỏi:** Ngoài Kafka, bạn biết những hệ thống message queue nào khác? Khi nào bạn chọn cái này thay vì cái kia?

**Câu hỏi mở rộng:**
- RabbitMQ và Kafka khác nhau cơ bản ở điểm nào?
- AWS SQS/SNS hoặc GCP Pub/Sub khác gì so với self-hosted message queue?

**Cách trả lời tốt:**
- RabbitMQ: message broker truyền thống, hỗ trợ nhiều routing pattern, message bị xóa sau khi consumed
- Kafka: distributed log, lưu message lâu dài, replay được, throughput cao
- RabbitMQ: phù hợp task queue, routing phức tạp. Kafka: phù hợp event streaming, data pipeline
- SQS/SNS: managed service, không cần quản lý infrastructure, tích hợp tốt với AWS ecosystem
- Bull/BullMQ: dùng Redis, phù hợp job queue đơn giản trong 1 ứng dụng
- Senior: Event streaming vs message queuing, choosing the right tool, hybrid approach

**Đánh giá level:**
- Junior: Biết 1-2 loại queue
- Mid: So sánh được ưu nhược điểm, chọn tool phù hợp
- Senior: Thiết kế messaging architecture cho hệ thống phức tạp

---

## 13. System Design

### Câu 13.1
**Câu hỏi:** Nếu bạn cần thiết kế một hệ thống rút gọn đường dẫn (URL shortener) như bit.ly, bạn sẽ thiết kế thế nào?

**Câu hỏi mở rộng:**
- Bạn tạo mã ngắn (short code) thế nào? Hash hay counter?
- Hệ thống cần xử lý hàng triệu lượt truy cập mỗi ngày, bạn tối ưu thế nào?

**Cách trả lời tốt:**
- API: POST /shorten (tạo short URL), GET /:code (redirect)
- Generate short code: Base62 encoding từ auto-increment ID, hoặc hash (MD5/SHA) rồi lấy 6-7 ký tự
- Database: key-value store hoặc SQL table (short_code, original_url, created_at, click_count)
- Tối ưu read: cache hot URLs trong Redis, TTL theo popularity
- Redirect: 301 (permanent, browser cache) vs 302 (temporary, server đếm click)
- Senior: Distributed ID generation, consistent hashing, analytics pipeline, rate limiting, custom domain

**Đánh giá level:**
- Junior: Thiết kế API cơ bản, database schema đơn giản
- Mid: Cache layer, xử lý collision, analytics
- Senior: Scalable architecture, distributed system considerations, capacity estimation

---

### Câu 13.2
**Câu hỏi:** Khi bạn cần chuyển từ kiến trúc một khối (monolith) sang kiến trúc vi dịch vụ (microservice), bạn tiếp cận thế nào? Có nên chuyển hết sang microservice ngay không?

**Câu hỏi mở rộng:**
- Microservice có nhược điểm gì mà nhiều người không nhắc đến?
- Bạn chia ranh giới dịch vụ (service boundary) dựa trên tiêu chí nào?

**Cách trả lời tốt:**
- Không chuyển hết ngay: strangler fig pattern - tách dần từng module ra microservice
- Bắt đầu từ module ít phụ thuộc, rõ ràng nhất (ví dụ: notification service, file upload service)
- Nhược điểm microservice: phức tạp về vận hành, distributed transaction, network latency, debugging khó
- Service boundary: dựa trên domain (bounded context - DDD), team structure (Conway's law)
- Inter-service communication: sync (HTTP/gRPC) vs async (message queue)
- Senior: Data ownership, API gateway, service mesh, observability cho microservice

**Đánh giá level:**
- Junior: Biết khái niệm, liệt kê ưu nhược điểm
- Mid: Có kinh nghiệm tách service, xử lý communication giữa services
- Senior: Migration strategy, DDD, organizational impact, operational maturity assessment

---

### Câu 13.3
**Câu hỏi:** Bạn thiết kế hệ thống thông báo (notification system) thế nào? Hệ thống cần gửi thông báo qua nhiều kênh (email, SMS, push notification, in-app)?

**Câu hỏi mở rộng:**
- Làm sao đảm bảo thông báo được gửi đúng 1 lần, không trùng lặp, không bị mất?
- Nếu cần gửi hàng triệu thông báo cùng lúc (ví dụ: khuyến mãi), bạn xử lý thế nào?

**Cách trả lời tốt:**
- Architecture: API → Message Queue → Notification Workers → Delivery Channels
- Template system: tách nội dung ra template, hỗ trợ đa ngôn ngữ
- Channel abstraction: Strategy pattern cho mỗi kênh (email, SMS, push)
- User preference: cho user chọn kênh và loại thông báo muốn nhận
- Reliability: retry mechanism, delivery tracking, DLQ
- Batch sending: chia thành batch, rate limiting per channel (email provider limit)
- Senior: Priority queue, real-time vs batch, analytics (open rate, click rate), A/B testing

**Đánh giá level:**
- Junior: Thiết kế cơ bản, gọi trực tiếp email API
- Mid: Queue-based, multi-channel, retry
- Senior: Scalable architecture, delivery guarantee, analytics, user preference management

---

### Câu 13.4
**Câu hỏi:** Bạn hiểu gì về "khả năng mở rộng" (scalability) của hệ thống? Khi lượng người dùng tăng gấp 10 lần, bạn cần thay đổi gì?

**Câu hỏi mở rộng:**
- Mở rộng ngang (horizontal scaling) và mở rộng dọc (vertical scaling) khác nhau thế nào?
- Load balancer hoạt động thế nào? Có những thuật toán phân phối nào?

**Cách trả lời tốt:**
- Vertical scaling: tăng cấu hình máy (CPU, RAM) - có giới hạn, đắt đỏ
- Horizontal scaling: thêm nhiều máy, phân tải - khả năng mở rộng cao hơn
- Stateless application: không lưu state trên server → dễ scale ngang
- Load balancer: phân phối request đều các server. Thuật toán: Round Robin, Least Connections, IP Hash
- Database scaling: read replica, sharding, caching layer
- Bottleneck identification: profiling, monitoring, load testing
- Senior: Auto-scaling, capacity planning, cost estimation, database sharding strategy

**Đánh giá level:**
- Junior: Biết khái niệm scaling, load balancer
- Mid: Thiết kế stateless app, cấu hình load balancer, caching
- Senior: End-to-end scaling strategy, bottleneck analysis, cost-performance trade-off

---

## 14. Git Flow

### Câu 14.1
**Câu hỏi:** Bạn sử dụng quy trình làm việc với Git (Git workflow) thế nào trong team? Bạn biết những mô hình nào (Git Flow, GitHub Flow, Trunk-based)?

**Câu hỏi mở rộng:**
- Khi nào bạn chọn Git Flow thay vì GitHub Flow? Ưu nhược điểm của mỗi loại?
- Branch naming convention của bạn thế nào?

**Cách trả lời tốt:**
- Git Flow: main, develop, feature/*, release/*, hotfix/* - phù hợp release cycle dài
- GitHub Flow: main + feature branches - đơn giản, phù hợp continuous deployment
- Trunk-based: commit trực tiếp vào main (hoặc short-lived branch) - phù hợp team mature, có CI/CD tốt
- Branch naming: feature/TICKET-123-add-login, bugfix/TICKET-456-fix-crash
- Commit message convention: Conventional Commits (feat:, fix:, chore:, refactor:)
- Senior: Chọn workflow phù hợp team size và release cycle, migration giữa các workflow

**Đánh giá level:**
- Junior: Biết tạo branch, merge cơ bản
- Mid: Áp dụng Git Flow hoặc GitHub Flow, commit message convention
- Senior: Chọn và tùy chỉnh workflow cho team, giải quyết vấn đề phức tạp

---

### Câu 14.2
**Câu hỏi:** Khi bạn và đồng nghiệp cùng sửa một file và xảy ra xung đột (merge conflict), bạn giải quyết thế nào? Bạn có mẹo gì để giảm thiểu conflict?

**Câu hỏi mở rộng:**
- `git rebase` và `git merge` khác nhau thế nào? Khi nào dùng cái nào?
- `git rebase -i` (interactive rebase) dùng làm gì?

**Cách trả lời tốt:**
- Giải quyết conflict: hiểu code cả 2 bên, chọn giữ phần nào hoặc kết hợp, chạy test sau khi resolve
- Giảm conflict: pull thường xuyên, chia task hợp lý (ít overlap file), feature branch ngắn hạn
- Merge: giữ lịch sử đầy đủ, tạo merge commit. Phù hợp shared branch
- Rebase: viết lại lịch sử thành đường thẳng, gọn gàng. Phù hợp feature branch cá nhân
- Interactive rebase: squash commits, reword message, reorder commits trước khi tạo pull request
- Senior: Rerere (reuse recorded resolution), cherry-pick strategy, bisect for debugging

**Đánh giá level:**
- Junior: Giải quyết được conflict đơn giản
- Mid: Thành thạo rebase vs merge, interactive rebase, cherry-pick
- Senior: Advanced Git operations, reflog recovery, complex merge strategies

---

### Câu 14.3
**Câu hỏi:** Quy trình code review của team bạn thế nào? Bạn nhìn vào những gì khi review code của đồng nghiệp?

**Câu hỏi mở rộng:**
- Bạn xử lý thế nào khi reviewer và tác giả không đồng ý về cách viết code?
- Automated code review (linting, static analysis) giúp gì?

**Cách trả lời tốt:**
- Checklist review: logic đúng không, edge case, error handling, security, performance, code style
- Pull request nên nhỏ, tập trung vào 1 mục đích, có description rõ ràng
- Review comment nên constructive, giải thích lý do, đưa ra suggestion
- Automated: ESLint, Prettier, SonarQube, type checking - bắt lỗi cơ bản trước khi human review
- Khi bất đồng: thảo luận dựa trên best practices, data, hoặc nhờ tech lead quyết định
- Senior: Code review culture, mentoring qua review, review checklist, review metrics

**Đánh giá level:**
- Junior: Review được logic cơ bản
- Mid: Review toàn diện (logic, performance, security), viết PR description tốt
- Senior: Xây dựng code review culture, mentoring, automated quality gates

---

## 15. Giải quyết bài toán thực tế

### Câu 15.1
**Câu hỏi:** Người dùng phản ánh rằng trang danh sách sản phẩm tải rất chậm (mất 5-10 giây). API trả về danh sách sản phẩm với thông tin chi tiết, ảnh, đánh giá, và sản phẩm liên quan. Bạn sẽ điều tra và cải thiện thế nào?

**Câu hỏi mở rộng:**
- Bạn bắt đầu điều tra từ đâu? Frontend hay backend?
- Nếu query database là bottleneck, bạn tối ưu thế nào mà không thay đổi cấu trúc database?

**Cách trả lời tốt:**
- Bước 1: Đo lường - dùng browser DevTools xem request nào chậm, backend profiling
- Bước 2: Phân tích API - kiểm tra query database (EXPLAIN), N+1 query, unnecessary data
- Bước 3: Tối ưu query - thêm index, fix N+1 (eager loading), chỉ SELECT cần thiết
- Bước 4: Caching - cache danh sách sản phẩm phổ biến vào Redis
- Bước 5: API optimization - pagination, lazy load (ảnh, đánh giá load riêng), response compression
- Bước 6: Infrastructure - CDN cho ảnh, read replica cho database
- Senior: Performance budget, monitoring, alerting khi response time tăng

**Đánh giá level:**
- Junior: Đề xuất 1-2 giải pháp (pagination, cache)
- Mid: Phân tích có hệ thống, áp dụng nhiều kỹ thuật
- Senior: End-to-end optimization strategy, đo lường trước-sau, monitoring ongoing

---

### Câu 15.2
**Câu hỏi:** Hệ thống thanh toán của bạn cần đảm bảo: khi người dùng thanh toán thành công, đơn hàng phải được cập nhật, email xác nhận phải được gửi, và kho hàng phải được trừ. Nếu một trong các bước bị lỗi, bạn xử lý thế nào?

**Câu hỏi mở rộng:**
- Nếu đã trừ tiền nhưng cập nhật đơn hàng thất bại thì sao?
- Saga pattern giải quyết vấn đề này thế nào?

**Cách trả lời tốt:**
- Nếu cùng database: dùng database transaction (all or nothing)
- Nếu khác service: không dùng được 1 transaction, cần distributed transaction
- Saga pattern: chuỗi các bước, mỗi bước có compensating action (hoàn tác) nếu bước sau thất bại
- Choreography: mỗi service lắng nghe event và phản ứng. Orchestration: 1 service điều phối
- Ví dụ: Payment success → Create Order → Reduce Inventory → Send Email. Nếu Reduce Inventory fail → Cancel Order → Refund Payment
- Idempotency: mỗi bước phải xử lý được việc bị gọi lại (retry safe)
- Senior: Outbox pattern, event sourcing, compensation logic design, monitoring saga execution

**Đánh giá level:**
- Junior: Đề cập transaction, xử lý cơ bản
- Mid: Implement saga pattern, retry mechanism, compensating action
- Senior: Distributed transaction design, failure analysis, monitoring, testing failure scenarios

---

### Câu 15.3
**Câu hỏi:** Ứng dụng của bạn đang chạy bình thường trên production, đột nhiên lúc 3 giờ sáng bạn nhận được cảnh báo: memory usage tăng liên tục và API response time chậm dần. Bạn xử lý thế nào?

**Câu hỏi mở rộng:**
- Memory leak trong Node.js thường do những nguyên nhân nào?
- Bạn có thể chụp "ảnh bộ nhớ" (heap snapshot) trên production không? Có rủi ro gì?

**Cách trả lời tốt:**
- Immediate: kiểm tra monitoring dashboard (memory, CPU, request rate), xem có deploy gần đây không
- Short-term: restart instance bị ảnh hưởng (nếu có load balancer), scale thêm instance
- Investigation: heap snapshot, memory profiling, xem log tìm pattern bất thường
- Common causes: event listener không remove, global variable tích lũy, cache không giới hạn, closure giữ reference
- Fix: tìm và fix memory leak, thêm monitoring cho memory, set alert threshold
- Prevention: load testing, memory limit per process, graceful restart khi memory cao
- Senior: Heap snapshot analysis (Chrome DevTools, clinic.js), canary deployment để detect sớm, postmortem process

**Đánh giá level:**
- Junior: Biết restart là giải pháp tạm, nêu 1-2 nguyên nhân
- Mid: Phân tích có hệ thống, sử dụng profiling tool, fix memory leak
- Senior: Production debugging strategy, incident response process, prevention measures

---

### Câu 15.4
**Câu hỏi:** Bạn cần thiết kế tính năng "giới hạn số lần gọi API" (rate limiting) cho hệ thống. Ví dụ: mỗi người dùng chỉ được gọi 100 lần/phút. Bạn thiết kế thế nào?

**Câu hỏi mở rộng:**
- Có những thuật toán rate limiting nào? Ưu nhược điểm?
- Khi có nhiều server, rate limit phải hoạt động thế nào?

**Cách trả lời tốt:**
- Fixed window: đếm request trong window cố định (mỗi phút reset). Đơn giản nhưng có burst ở ranh giới
- Sliding window: đếm trong cửa sổ trượt, chính xác hơn
- Token bucket: mỗi user có bucket chứa token, mỗi request tiêu 1 token, token được bổ sung đều đặn
- Leaky bucket: request xếp hàng và xử lý ở tốc độ cố định
- Distributed: dùng Redis (INCR + EXPIRE) để đếm across multiple servers
- NestJS: @nestjs/throttler cho basic, custom Guard + Redis cho distributed
- Response: trả về 429 Too Many Requests, kèm header Retry-After
- Senior: Per-endpoint limits, tiered rate limiting (free vs premium), graceful degradation

**Đánh giá level:**
- Junior: Biết khái niệm, dùng thư viện có sẵn
- Mid: Implement với Redis, hiểu thuật toán cơ bản
- Senior: Chọn thuật toán phù hợp, distributed rate limiting, monitoring, dynamic rate adjustment

---

### Câu 15.5
**Câu hỏi:** Team bạn tiếp nhận một dự án cũ (legacy code) không có tài liệu, không có test, code lộn xộn. Bạn sẽ tiếp cận và cải thiện dự án này thế nào?

**Câu hỏi mở rộng:**
- Bạn ưu tiên refactor những gì trước?
- Làm sao để refactor mà không làm hỏng tính năng đang chạy?

**Cách trả lời tốt:**
- Bước 1: Hiểu hệ thống - đọc code, chạy app, nói chuyện với người dùng/stakeholder
- Bước 2: Thêm monitoring + logging trước - hiểu hệ thống hoạt động thế nào trên production
- Bước 3: Viết test cho các luồng quan trọng nhất (integration test / E2E test)
- Bước 4: Refactor dần - "boy scout rule" (để code sạch hơn mỗi lần chạm vào)
- Bước 5: Tách module rõ ràng, thêm TypeScript types
- Không: rewrite toàn bộ từ đầu (rủi ro cao, tốn thời gian)
- Senior: Strangler fig pattern, feature flags, parallel run (old vs new), technical debt tracking

**Đánh giá level:**
- Junior: Muốn viết lại từ đầu, không có chiến lược rõ ràng
- Mid: Tiếp cận từng bước, viết test trước khi refactor
- Senior: Chiến lược cải thiện dài hạn, cân bằng giữa refactor và deliver feature, stakeholder management

---

## Bảng đánh giá tổng hợp

| Tiêu chí | Junior | Mid | Senior |
|----------|--------|-----|--------|
| Kiến thức lý thuyết | Biết khái niệm cơ bản | Hiểu sâu, giải thích được | Phân tích trade-off, so sánh giải pháp |
| Kinh nghiệm thực tế | Ít dự án, chủ yếu làm theo hướng dẫn | 2-3 dự án, tự giải quyết vấn đề | Nhiều dự án, dẫn dắt kỹ thuật |
| Giải quyết vấn đề | Cần gợi ý, giải quyết từng phần | Tự phân tích, đưa ra giải pháp | Hệ thống hóa, xem xét nhiều khía cạnh |
| Thiết kế hệ thống | Không hoặc rất cơ bản | Thiết kế được module/service | Thiết kế kiến trúc toàn hệ thống |
| Code quality | Code chạy được | Code sạch, có test | Code maintainable, scalable, testable |
| Communication | Trả lời ngắn, thiếu chi tiết | Trình bày rõ ràng, có ví dụ | Giải thích phức tạp thành đơn giản |

---

## Gợi ý sử dụng

1. **Junior (20-30 phút):** Chọn 8-10 câu từ các topic cơ bản (1-5), hỏi câu hỏi chính, mở rộng nếu ứng viên trả lời tốt
2. **Mid (30-45 phút):** Chọn 10-15 câu trải đều các topic, luôn hỏi câu mở rộng
3. **Senior (45-60 phút):** Tập trung vào topic 9-15 (Database, Cloud, Cache, Queue, System Design, Bài toán thực tế), hỏi sâu về trade-off và kinh nghiệm

> **Lưu ý:** Không cần hỏi hết tất cả câu hỏi. Dựa vào phản hồi của ứng viên để quyết định hỏi sâu hay chuyển topic. Mục tiêu là đánh giá tư duy và kinh nghiệm, không phải kiểm tra kiến thức thuộc lòng.
