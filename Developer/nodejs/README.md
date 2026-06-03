# 🟢 Node.js Developer Knowledge Roadmap

> Hướng dẫn toàn diện về phát triển backend với Node.js — từ kiến thức nền tảng đến vận hành hệ thống thực tế.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Phase 1 — Nền Tảng (Tuần 1–2)**

- [ ] JavaScript nâng cao: closure, prototype, hoisting, event loop
- [ ] Node.js runtime: module system (CommonJS & ESM), libuv, V8 engine
- [ ] Lập trình bất đồng bộ: callback, Promise, async/await
- [ ] HTTP cơ bản, RESTful API design, JSON

### **Phase 2 — Kỹ Năng Core (Tuần 3–6)**

- [ ] Web framework: Express.js / Fastify — routing, middleware, error handling
- [ ] Cơ sở dữ liệu: Sequelize (SQL), Prisma, Mongoose (MongoDB)
- [ ] Authentication — JWT (JSON Web Token), OAuth2, session management
- [ ] Validation, error handling, logging

### **Phase 3 — Vận Hành & Kiểm Thử (Tuần 7–10)**

- [ ] Testing: Jest, Supertest, integration & unit tests
- [ ] Performance: caching (Redis), clustering, profiling
- [ ] Message queues: Kafka, RabbitMQ, Bull/BullMQ
- [ ] Docker, CI/CD pipeline, deployment

### **Phase 4 — Chuyên Sâu (Tuần 11+)**

- [ ] Kiến trúc: microservices, event-driven, clean architecture
- [ ] Advanced: Streams, WebSockets, gRPC, GraphQL
- [ ] Observability: tracing, metrics, structured logging
- [ ] Cloud deployment: Kubernetes, Helm, ArgoCD

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                             | Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------ | ------- | --------- | ---------- |
| **JavaScript & Node.js Nền Tảng**   | ⭐⭐⭐  | 2 tuần    | -          |
| **REST API & Web Framework**         | ⭐⭐⭐  | 2 tuần    | -          |
| **Database & ORM**                   | ⭐⭐⭐  | 2 tuần    | -          |
| **Authentication & Security**        | ⭐⭐⭐  | 2 tuần    | -          |
| **Testing (Kiểm Thử)**               | ⭐⭐⭐  | 1 tuần    | -          |
| **Async & Message Queue**            | ⭐⭐⭐  | 2 tuần    | -          |
| **Performance & Caching**            | ⭐⭐⭐  | 2 tuần    | -          |
| **Architecture (Kiến Trúc)**         | ⭐⭐    | 2 tuần    | -          |
| **Deployment & CI/CD**               | ⭐⭐    | 2 tuần    | -          |
| **Advanced Topics**                  | ⭐⭐    | 3 tuần    | -          |

---

## 🗂️ Tổng Quan Chủ Đề

### 📁 **1. Fundamentals — Nền Tảng** (`01-fundamentals/`)

- JavaScript nâng cao: scope, closure, prototype chain, this keyword
- Event Loop — vòng lặp sự kiện: call stack, task queue, microtask queue
- Module system — hệ thống module: CommonJS (`require`) vs ESM (`import/export`)
- Node.js internals: libuv, V8 engine, worker threads
- Streams — luồng dữ liệu và Buffer

### 📁 **2. Web Layer — Tầng Web** (`02-web-layer/`)

- Express.js & Fastify: routing, middleware pipeline
- RESTful API design — thiết kế API: versioning, resource naming, HTTP methods
- Request/Response lifecycle — vòng đời yêu cầu/phản hồi
- Error handling — xử lý lỗi: centralized error middleware, custom error classes
- Validation: Joi, Zod, class-validator
- OpenAPI / Swagger documentation

### 📁 **3. Database & ORM** (`03-database/`)

- Sequelize — ORM cho SQL (PostgreSQL, MySQL)
- Prisma — type-safe ORM thế hệ mới
- Mongoose — ODM cho MongoDB
- Transactions — giao dịch và isolation levels
- N+1 Problem — vấn đề truy vấn N+1 và cách khắc phục
- Connection pooling — quản lý pool kết nối cơ sở dữ liệu
- Redis integration — tích hợp Redis cho caching & session

### 📁 **4. Security — Bảo Mật** (`04-security/`)

- JWT — JSON Web Token: signing, verification, refresh token strategy
- OAuth2 & OIDC — OpenID Connect: authorization code flow
- Helmet, CORS — Cross-Origin Resource Sharing, CSRF protection
- Rate limiting — giới hạn tốc độ yêu cầu: express-rate-limit, redis-based
- Input sanitization & SQL injection prevention
- Secret management — quản lý bí mật: environment variables, Vault

### 📁 **5. Async & Messaging — Bất Đồng Bộ & Nhắn Tin** (`05-async-messaging/`)

- Promise patterns: chaining, Promise.all, Promise.race, error propagation
- async/await best practices — xử lý lỗi, pitfalls thường gặp
- EventEmitter — phát và lắng nghe sự kiện trong Node.js
- Kafka integration — tích hợp Kafka: producer, consumer, consumer groups
- RabbitMQ integration — tích hợp RabbitMQ: exchanges, queues, DLQ
- Bull/BullMQ — job queue với Redis: priority, retry, rate limiting
- Scheduled tasks — cron jobs với node-cron / agenda

### 📁 **6. Testing — Kiểm Thử** (`06-testing/`)

- Unit testing — kiểm thử đơn vị với Jest
- Mocking — giả lập phụ thuộc: jest.mock, sinon
- Integration testing — kiểm thử tích hợp với Supertest
- Test containers — testcontainers-node cho database testing
- E2E testing — kiểm thử đầu cuối với Playwright / Cypress
- Code coverage — độ bao phủ mã nguồn: Istanbul / c8

### 📁 **7. Performance — Hiệu Năng** (`07-performance/`)

- Caching strategies — chiến lược cache: in-memory, Redis, HTTP cache
- Clustering — phân cụm: Node.js cluster module, PM2 cluster mode
- Connection pooling optimization — tối ưu hóa pool kết nối
- Query optimization — tối ưu hóa truy vấn và index
- Profiling — phân tích hiệu năng: Node.js profiler, clinic.js, 0x
- Load testing — kiểm thử tải: k6, Artillery, autocannon

### 📁 **8. Architecture — Kiến Trúc** (`08-architecture/`)

- Clean Architecture — kiến trúc sạch: layers, dependency inversion
- Layered Architecture — kiến trúc phân tầng: controller, service, repository
- Microservices — kiến trúc vi dịch vụ: decomposition, inter-service communication
- API Gateway — cổng API: routing, aggregation, authentication
- Event-Driven Architecture — kiến trúc hướng sự kiện: event sourcing, CQRS
- Domain-Driven Design — DDD cơ bản: bounded context, aggregate

### 📁 **9. Deployment — Triển Khai** (`09-deployment/`)

- Docker — container hóa ứng dụng Node.js: multi-stage builds
- Kubernetes — orchestration: Deployment, Service, ConfigMap, Secret
- CI/CD pipeline — tích hợp & triển khai liên tục: GitHub Actions, GitLab CI
- PM2 — process manager cho production
- Health checks & graceful shutdown — kiểm tra sức khỏe & tắt an toàn
- Observability — quan sát hệ thống: structured logging, tracing, metrics

### 📁 **10. Advanced — Nâng Cao** (`10-advanced/`)

- Streams & Backpressure — xử lý dữ liệu lớn theo luồng
- WebSockets — giao tiếp thời gian thực: Socket.IO, ws library
- gRPC — Remote Procedure Call hiệu năng cao với Protocol Buffers
- GraphQL — truy vấn API linh hoạt: Apollo Server, resolvers, dataloaders
- Worker Threads — luồng worker cho CPU-intensive tasks
- Native Addons — mở rộng Node.js với C/C++ qua N-API

### 📁 **11. Interview Prep — Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 30 câu hỏi Node.js phỏng vấn kèm đáp án
- System design scenarios — bài toán thiết kế hệ thống
- Coding challenges — bài tập code thực tế
- STAR stories — kể chuyện kinh nghiệm theo phương pháp STAR
- Kế hoạch ôn tập 90 ngày

---

## 🎓 Ma Trận Kỹ Năng

### Beginner — Mới Bắt Đầu (0–1 năm)

- [ ] Hiểu event loop, call stack, task queue
- [ ] Viết REST API cơ bản với Express.js
- [ ] Dùng async/await đúng cách
- [ ] Kết nối và truy vấn cơ sở dữ liệu
- [ ] Viết unit test đơn giản với Jest

### Intermediate — Trung Cấp (1–3 năm)

- [ ] Thiết kế middleware pipeline có cấu trúc
- [ ] Implement JWT authentication hoàn chỉnh
- [ ] Tối ưu hóa N+1 queries với eager loading
- [ ] Tích hợp Redis cho caching và session
- [ ] Cấu hình Docker & CI/CD pipeline cơ bản
- [ ] Viết integration tests với Supertest

### Advanced — Nâng Cao (3–5+ năm)

- [ ] Thiết kế microservices với event-driven architecture
- [ ] Implement distributed tracing — theo dõi phân tán
- [ ] Tối ưu hóa hiệu năng với profiling thực tế
- [ ] Thiết kế hệ thống chịu tải cao (high-throughput)
- [ ] Kubernetes deployment & autoscaling
- [ ] Implement circuit breaker, retry, bulkhead patterns

---

## 🚀 Bắt Đầu

### Bước 1: Xác Định Mục Tiêu

```
Chọn hướng đi:
- Junior Developer — tập trung 01–04, 06
- Backend Engineer — tập trung 01–08
- Senior / Lead — toàn bộ, đặc biệt 08–10
- Interview only — tập trung 11-interview-prep
```

### Bước 2: Dựng Môi Trường Thực Hành

```bash
# Khởi động stack đầy đủ với Docker Compose
docker-compose up -d
# Bao gồm: Node.js app, PostgreSQL, Redis, RabbitMQ, MongoDB
```

### Bước 3: Học Kết Hợp Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Tự code lại ví dụ (30–60 phút)
3. Giải bài tập thực tế (30 phút)
4. Đánh giá lại checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation  — Tình huống gặp phải
- Task       — Nhiệm vụ được giao
- Action     — Hành động đã thực hiện
- Result     — Kết quả đạt được
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Phổ Biến Theo Chủ Đề

#### JavaScript & Node.js Nền Tảng

- [ ] Giải thích event loop hoạt động như thế nào?
- [ ] Sự khác biệt giữa `process.nextTick`, `setImmediate`, và `setTimeout`?
- [ ] CommonJS và ESM module khác nhau điểm gì?
- [ ] Khi nào dùng Worker Threads thay vì Cluster?

#### REST API & Web Framework

- [ ] Middleware pipeline trong Express.js hoạt động ra sao?
- [ ] Cách thiết kế error handling toàn cục?
- [ ] So sánh Express.js với Fastify?
- [ ] Thiết kế API versioning strategy?

#### Database & Performance

- [ ] N+1 problem là gì, giải quyết như thế nào?
- [ ] Khi nào dùng Redis thay vì in-memory cache?
- [ ] Giải thích connection pooling trong Node.js?
- [ ] Transaction isolation level nào phù hợp cho từng trường hợp?

#### Security — Bảo Mật

- [ ] JWT access token và refresh token khác nhau thế nào?
- [ ] Cách phòng chống SQL Injection trong Node.js?
- [ ] CORS — Cross-Origin Resource Sharing hoạt động ra sao?
- [ ] Giải thích OAuth2 authorization code flow?

#### Architecture & System Design

- [ ] Thiết kế hệ thống notification real-time cho 1 triệu users?
- [ ] Khi nào nên tách monolith sang microservices?
- [ ] Cách implement distributed transaction trong microservices?
- [ ] Event sourcing và CQRS — Command Query Responsibility Segregation?

Xem `11-interview-prep/` để có hướng dẫn Q&A đầy đủ.

---

## ✅ Self-Assessment Checklist — Tự Đánh Giá

Trước phỏng vấn hoặc vai trò mới, kiểm tra:

- [ ] Có thể giải thích event loop không cần tài liệu
- [ ] Có thể thiết kế REST API hoàn chỉnh với authentication
- [ ] Có thể debug memory leak trong Node.js
- [ ] Có thể viết test coverage > 80% cho service mới
- [ ] Có thể thiết kế caching strategy cho hệ thống đọc nhiều
- [ ] Có thể migrate database an toàn không downtime
- [ ] Có thể thiết kế job queue với retry và dead letter queue
- [ ] Có thể respond với incident (slow API, memory spike)
- [ ] Có thể giải thích trade-off giữa monolith và microservices
- [ ] Có thể thiết kế hệ thống horizontally scalable

---

## 📖 Tài Liệu Tham Khảo

### Sách Nền Tảng

- **"Node.js Design Patterns"** — Mario Casciaro & Luciano Mammino — Patterns thực tế
- **"You Don't Know JS"** — Kyle Simpson — JavaScript chuyên sâu
- **"Designing Data-Intensive Applications"** — Martin Kleppmann — Thiết kế hệ thống lớn
- **"Clean Architecture"** — Robert C. Martin — Kiến trúc phần mềm sạch

### Tài Liệu Chính Thức

- [Node.js Documentation](https://nodejs.org/en/docs/)
- [Express.js Guide](https://expressjs.com/en/guide/)
- [Fastify Documentation](https://www.fastify.io/docs/)
- [Prisma Documentation](https://www.prisma.io/docs/)

### Blogs & Cộng Đồng

- Node.js Weekly Newsletter
- NodeSource Blog
- nearForm Engineering Blog
- r/node (Reddit)

---

## 📊 Ước Tính Thời Gian Học

| Module                         | Thời Gian   | Độ Khó | Ưu Tiên |
| ------------------------------ | ----------- | ------ | ------- |
| Fundamentals — Nền Tảng        | 4–6 giờ     | ⭐     | Bắt buộc |
| Web Layer — Tầng Web           | 6–8 giờ     | ⭐⭐   | Bắt buộc |
| Database & ORM                 | 8–10 giờ    | ⭐⭐   | Bắt buộc |
| Security — Bảo Mật             | 6–8 giờ     | ⭐⭐   | Bắt buộc |
| Testing — Kiểm Thử             | 4–6 giờ     | ⭐⭐   | Bắt buộc |
| Async & Messaging              | 6–8 giờ     | ⭐⭐   | Bắt buộc |
| Performance — Hiệu Năng        | 8–10 giờ    | ⭐⭐⭐ | Nên có   |
| Architecture — Kiến Trúc       | 10–12 giờ   | ⭐⭐⭐ | Nên có   |
| Deployment — Triển Khai        | 6–8 giờ     | ⭐⭐   | Nên có   |
| Advanced — Nâng Cao            | 15–20 giờ   | ⭐⭐⭐ | Tùy chọn |

**Tổng cộng: 73–96 giờ để nắm vững Node.js backend**

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này toàn bộ
├─ 2️⃣  Chọn learning path (Junior/Backend Engineer/Senior)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Dựng môi trường Docker để thực hành
├─ 5️⃣  Hoàn thành bài tập từng module
├─ 6️⃣  Xây dựng project portfolio (ví dụ: REST API + auth + queue)
└─ 7️⃣  Luyện phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
**Duy Trì Bởi:** Backend Interview Prep
