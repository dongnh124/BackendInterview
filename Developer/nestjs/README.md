# 🦁 NestJS Developer Knowledge Roadmap

> Hướng dẫn toàn diện về phát triển backend với NestJS — từ nền tảng đến production-grade architecture.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Phase 1 — Nền Tảng (Tuần 1–2)**

- [ ] TypeScript nâng cao: Generics, Decorators, Metadata
- [ ] Module system & Dependency Injection (DI — Tiêm Phụ Thuộc)
- [ ] Controller, Service, Provider cơ bản
- [ ] Lifecycle hooks & Module scope

### **Phase 2 — Core Skills (Tuần 3–6)**

- [ ] Database integration: TypeORM / Prisma / Mongoose
- [ ] Authentication: JWT (JSON Web Token) + Guards
- [ ] Validation với class-validator & class-transformer
- [ ] Exception filters & Interceptors
- [ ] Testing: Unit & Integration

### **Phase 3 — Production Skills (Tuần 7–10)**

- [ ] Microservices & Message queues (Kafka, RabbitMQ)
- [ ] CQRS (Command Query Responsibility Segregation — Phân Tách Lệnh Truy Vấn)
- [ ] Caching với Redis
- [ ] Docker & Kubernetes deployment
- [ ] Monitoring & Observability

### **Phase 4 — Chuyên Sâu (Tuần 11+)**

- [ ] GraphQL với NestJS
- [ ] WebSockets & Server-Sent Events
- [ ] Advanced microservices patterns
- [ ] Event sourcing & Domain-Driven Design

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                 | Ưu Tiên | Thời Gian | Trạng Thái |
| ---------------------------------------- | ------- | --------- | ---------- |
| **Module & Dependency Injection**        | ⭐⭐⭐  | 1 tuần    | -          |
| **Database & ORM Integration**           | ⭐⭐⭐  | 2 tuần    | -          |
| **Authentication & Authorization**       | ⭐⭐⭐  | 1 tuần    | -          |
| **Validation & Serialization**           | ⭐⭐⭐  | 1 tuần    | -          |
| **Testing (Unit & Integration)**         | ⭐⭐⭐  | 1 tuần    | -          |
| **Async Messaging (Kafka/RabbitMQ)**     | ⭐⭐⭐  | 2 tuần    | -          |
| **Performance & Caching**                | ⭐⭐⭐  | 1 tuần    | -          |
| **Microservices Architecture**           | ⭐⭐⭐  | 2 tuần    | -          |
| **Docker & Kubernetes Deployment**       | ⭐⭐    | 1 tuần    | -          |
| **GraphQL Integration**                  | ⭐⭐    | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Chủ Đề

### 📁 **1. Nền Tảng** (`01-fundamentals/`)

- TypeScript nâng cao cho NestJS: Generics, Decorators, Reflect Metadata
- Kiến trúc module: Module phân cấp, Shared Module, Dynamic Module
- DI container (Dependency Injection Container — Bộ Chứa Phụ Thuộc): providers, scope, circular dependency
- Lifecycle hooks: `onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`
- Request lifecycle: Middleware → Guards → Interceptors → Pipes → Handlers → Exception Filters

### 📁 **2. Tầng Web** (`02-web-layer/`)

- Controller & Routing: HTTP methods, Route parameters, Query params
- Middleware & Guards: `CanActivate`, role-based guard
- Interceptors (Bộ Chặn): transform response, logging, timeout
- Pipes (Ống Xử Lý): validation, transformation, built-in vs custom
- Exception Filters (Bộ Lọc Ngoại Lệ): global vs controller-scoped, HTTP exceptions
- OpenAPI / Swagger: `@ApiProperty`, schema generation, decorators

### 📁 **3. Database & ORM** (`03-database/`)

- **TypeORM** — Entity, Repository, DataSource, Migrations
- **Prisma** — Schema-first, type-safe client, migrations
- **Mongoose** — Schema, Model, virtual, middleware
- Transactions & Unit of Work pattern
- N+1 problem & Eager/Lazy loading
- Redis integration: caching, session, pub/sub

### 📁 **4. Bảo Mật** (`04-security/`)

- JWT — Signing, refresh token, blacklisting với Redis
- OAuth2 & OIDC (OpenID Connect — Kết Nối Danh Tính Mở): Passport.js strategies
- RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò) & ABAC
- Rate limiting (Giới Hạn Tốc Độ): `@nestjs/throttler`, Redis-backed limiter
- Helmet, CORS (Cross-Origin Resource Sharing), CSRF protection
- Secret management: dotenv, Vault, AWS Secrets Manager

### 📁 **5. Async Messaging** (`05-async-messaging/`)

- Microservice transports: TCP, Redis, MQTT, RabbitMQ, Kafka
- `@MessagePattern` và `@EventPattern` decorators
- Kafka integration: producer, consumer, consumer groups, partitions
- RabbitMQ integration: exchanges, queues, bindings, DLQ (Dead Letter Queue — Hàng Đợi Thư Chết)
- BullMQ — Job queues, retry strategies, rate limiting
- Request-reply pattern vs fire-and-forget

### 📁 **6. Testing — Kiểm Thử** (`06-testing/`)

- Unit testing với Jest: mocking providers, testing modules
- `Test.createTestingModule()` — NestJS testing utilities
- Integration testing: Supertest + real database
- Testcontainers: PostgreSQL, Redis, Kafka trong Docker
- E2E (End-to-End — Kiểm Thử Đầu Cuối) testing: full application flow
- Test coverage & mutation testing

### 📁 **7. Hiệu Năng** (`07-performance/`)

- Caching strategies: In-memory, Redis, HTTP cache (CacheInterceptor)
- Connection pooling (Quản Lý Hồ Kết Nối): TypeORM pool config, PgBouncer
- Compression, response streaming
- Profiling NestJS apps: clinic.js, `--prof`, heap snapshots
- Load testing: k6, Artillery với NestJS endpoints

### 📁 **8. Kiến Trúc** (`08-architecture/`)

- Clean Architecture trong NestJS: Layers, Use Cases, Ports & Adapters
- CQRS (Command Query Responsibility Segregation): `@nestjs/cqrs`, CommandBus, QueryBus
- Event Sourcing (Lưu Trữ Sự Kiện) & EventBus
- DDD (Domain-Driven Design — Thiết Kế Hướng Miền): Aggregate, Entity, Value Object, Bounded Context
- Microservices decomposition & inter-service communication
- API Gateway pattern & BFF (Backend For Frontend)

### 📁 **9. Triển Khai** (`09-deployment/`)

- Docker: Multi-stage build cho NestJS, `.dockerignore`, health check
- Kubernetes: Deployment, Service, HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang), ConfigMap, Secret
- PM2: Process management, cluster mode, zero-downtime restart
- CI/CD: GitHub Actions, GitLab CI — build, test, Docker push
- Graceful shutdown (Tắt Máy An Toàn): `enableShutdownHooks()`, thoát sạch
- Observability: Structured logging (pino), OpenTelemetry tracing, Prometheus metrics

### 📁 **10. Nâng Cao** (`10-advanced/`)

- GraphQL: `@nestjs/graphql`, Code-first vs Schema-first, DataLoader, subscriptions
- WebSockets: `@nestjs/websockets`, Socket.IO gateway, namespace, rooms
- gRPC (Google Remote Procedure Call): `@GrpcMethod`, Protobuf, interceptors
- Server-Sent Events (Sự Kiện Máy Chủ Gửi): `@Sse()`, Observable streams
- Custom decorators & metadata reflection
- NestJS CLI plugins & schematics

### 📁 **11. Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 30 câu hỏi phỏng vấn NestJS kèm đáp án chi tiết
- System design scenarios với NestJS
- Coding challenges thực tế
- Mẫu câu chuyện STAR (Situation-Task-Action-Result)
- Kế hoạch học 90 ngày

---

## 🎓 Theo Nền Tảng Sử Dụng

### **NestJS + TypeORM + PostgreSQL**

```
Ưu điểm: Type-safe, migrations, relations, mature ecosystem
Phù hợp: Dự án enterprise, domain phức tạp, RDBMS workload
Học tại: 01-fundamentals, 03-database/1-typeorm.md, 08-architecture
```

### **NestJS + Prisma + PostgreSQL**

```
Ưu điểm: Schema-first, auto-generated client, DX tuyệt vời
Phù hợp: Dự án mới, team nhỏ, prototyping nhanh
Học tại: 03-database/2-prisma.md, 06-testing
```

### **NestJS + Mongoose + MongoDB**

```
Ưu điểm: Flexible schema, document model, horizontal scaling
Phù hợp: Schema thay đổi liên tục, IoT, real-time data
Học tại: 03-database/3-mongoose.md, 10-advanced/2-websockets.md
```

### **NestJS Microservices**

```
Ưu điểm: Native microservices support, multiple transports
Phù hợp: Distributed systems, event-driven architecture
Học tại: 05-async-messaging, 08-architecture/3-microservices.md
```

---

## 🔗 Điều Hướng Nhanh

| Chủ Đề                          | Thư Mục / File                                                          | Ưu Tiên            |
| ------------------------------- | ----------------------------------------------------------------------- | ------------------- |
| Bắt đầu với NestJS              | [01-fundamentals/](./01-fundamentals/)                                  | Bắt đầu tại đây    |
| Câu hỏi phỏng vấn               | [11-interview-prep/](./11-interview-prep/)                              | Trước phỏng vấn     |
| JWT & Auth                      | [04-security/1-jwt-authentication.md](./04-security/1-jwt-authentication.md) | Thiết yếu      |
| TypeORM & Migrations            | [03-database/1-typeorm.md](./03-database/1-typeorm.md)                  | Thiết yếu           |
| Microservices với Kafka         | [05-async-messaging/3-kafka-integration.md](./05-async-messaging/3-kafka-integration.md) | Production |
| Clean Architecture & CQRS      | [08-architecture/](./08-architecture/)                                  | Senior level        |
| Kubernetes Deployment           | [09-deployment/2-kubernetes.md](./09-deployment/2-kubernetes.md)        | DevOps              |

---

## 📊 Ma Trận Kỹ Năng

### Beginner — Mới Bắt Đầu (0–1 năm)

- [ ] Hiểu Dependency Injection và IoC (Inversion of Control — Đảo Ngược Điều Khiển)
- [ ] Xây dựng REST API với Controller, Service, Repository
- [ ] Kết nối database với TypeORM hoặc Prisma
- [ ] Xác thực người dùng với JWT
- [ ] Viết unit test cơ bản với Jest

**Thời gian đạt được:** 2–3 tháng

### Intermediate — Trung Cấp (1–3 năm)

- [ ] Thiết kế Guards và Interceptors tùy chỉnh
- [ ] Tích hợp message queue (Kafka / RabbitMQ)
- [ ] Implement CQRS pattern với `@nestjs/cqrs`
- [ ] Viết integration tests đầy đủ với Testcontainers
- [ ] Deploy ứng dụng với Docker & Kubernetes

**Thời gian đạt được:** 2–3 tháng để nâng cao

### Advanced — Nâng Cao (3–5+ năm)

- [ ] Thiết kế microservices system với event-driven architecture
- [ ] Implement Event Sourcing & DDD trong NestJS
- [ ] GraphQL federation với NestJS
- [ ] Tối ưu performance ở production (profiling, caching, query tuning)
- [ ] Dẫn dắt architectural decisions trong team

**Thời gian đạt được:** Học liên tục

---

## 🚀 Bắt Đầu

### Bước 1: Thiết Lập Mục Tiêu

```
Chọn con đường phù hợp:
- Backend Developer (REST API, monolith)
- Fullstack Developer (NestJS + GraphQL + frontend)
- Microservices Engineer (distributed systems)
```

### Bước 2: Thiết Lập Môi Trường

```bash
# Cài NestJS CLI
npm i -g @nestjs/cli

# Tạo project mới
nest new my-nestjs-app

# Chạy development server
cd my-nestjs-app && npm run start:dev
```

### Bước 3: Học + Thực Hành

```
1. Đọc một module (30 phút)
2. Tự code lại ví dụ (30–60 phút)
3. Thêm unit test cho code vừa viết (30 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện STAR

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation  — Bối cảnh dự án
- Task       — Nhiệm vụ cụ thể
- Action     — Hành động bạn thực hiện
- Result     — Kết quả đạt được (số liệu cụ thể)
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Top Câu Hỏi Theo Danh Mục

#### Module & Dependency Injection

- [ ] Giải thích DI container hoạt động như thế nào trong NestJS?
- [ ] Phân biệt `Singleton`, `Request`, `Transient` scope?
- [ ] Circular dependency xảy ra khi nào và cách giải quyết?

#### Authentication & Authorization

- [ ] Thiết kế JWT authentication flow hoàn chỉnh
- [ ] Sự khác biệt giữa Guards và Middleware?
- [ ] Implement RBAC (Role-Based Access Control) trong NestJS?

#### Database & ORM

- [ ] So sánh TypeORM và Prisma — khi nào dùng cái nào?
- [ ] Giải thích N+1 problem và cách giải quyết trong NestJS?
- [ ] Transaction management trong TypeORM?

#### Microservices & Messaging

- [ ] Các transport options trong NestJS microservices?
- [ ] Phân biệt `@MessagePattern` và `@EventPattern`?
- [ ] Xử lý partial failure trong microservices?

#### Architecture

- [ ] Giải thích CQRS và khi nào nên dùng?
- [ ] Implement Clean Architecture trong NestJS như thế nào?
- [ ] Phân biệt Event Sourcing và Event-Driven Architecture?

Xem `11-interview-prep/` để có hướng dẫn phỏng vấn đầy đủ.

---

## ✅ Tự Đánh Giá

Trước phỏng vấn hoặc nhận role mới, kiểm tra:

- [ ] Có thể giải thích DI container không cần tài liệu
- [ ] Có thể thiết kế authentication flow đầy đủ
- [ ] Có thể đọc và viết TypeORM entity với relations
- [ ] Có thể debug memory leak trong NestJS app
- [ ] Có thể implement custom Guard và Interceptor
- [ ] Có thể thiết kế microservice với message queue
- [ ] Có thể deploy NestJS app lên Kubernetes
- [ ] Có thể viết test suite với coverage > 80%
- [ ] Có thể thảo luận CQRS và khi nào dùng
- [ ] Có thể giải thích trade-off giữa monolith và microservices

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"NestJS: A Progressive Node.js Framework"** — Official docs (miễn phí, luôn cập nhật)
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — System design
- **"Clean Architecture"** by Robert C. Martin — Architecture patterns
- **"Domain-Driven Design"** by Eric Evans — DDD concepts
- **"Building Microservices"** by Sam Newman — Microservices patterns

### Tài Liệu Chính Thức

- [NestJS Documentation](https://docs.nestjs.com/)
- [TypeORM Documentation](https://typeorm.io/)
- [Prisma Documentation](https://www.prisma.io/docs)
- [class-validator](https://github.com/typestack/class-validator)
- [Jest Documentation](https://jestjs.io/docs/getting-started)

### Bài Viết & Blog

- NestJS official blog
- Trilon.io (NestJS experts)
- dev.to/nestjs tag
- Medium — NestJS tag

---

## 📋 Cách Sử Dụng Guide Này

### Để Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Học tuần tự qua từng phase
3. Thực hành code sau mỗi module
4. Xây dựng một project portfolio

### Để Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep/](./11-interview-prep/)
2. Ôn lại chủ đề theo role mục tiêu
3. Chuẩn bị câu chuyện STAR từ kinh nghiệm thực tế
4. Luyện giải thích concepts không cần nhìn tài liệu

### Để Làm Việc Thực Tế

1. Tra cứu [Architecture Guides](./08-architecture/) khi cần quyết định kiến trúc
2. Dùng [Security](./04-security/) checklist cho security review
3. Xem [Performance](./07-performance/) khi gặp vấn đề hiệu năng
4. Theo dõi [Deployment](./09-deployment/) runbooks khi deploy

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn learning path (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Thiết lập môi trường phát triển với NestJS CLI
├─ 5️⃣  Hoàn thành bài tập cho từng module
├─ 6️⃣  Xây dựng project portfolio
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
**Maintainer:** Backend Interview Prep
