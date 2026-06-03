# NestJS Developer Knowledge Base — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về phát triển backend với NestJS

## 📁 Cấu Trúc Thư Mục

```
Developer/nestjs/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/
│   ├── README.md                               Tổng quan nền tảng TypeScript & NestJS
│   ├── 1-typescript-advanced.md               Generics, Decorators, Reflect Metadata
│   ├── 2-module-system.md                     Module phân cấp, Dynamic Module, Shared Module
│   ├── 3-dependency-injection.md              DI container, Provider scope, Circular dependency
│   ├── 4-lifecycle-hooks.md                   onModuleInit, onApplicationBootstrap, shutdown hooks
│   └── 5-request-lifecycle.md                Middleware → Guards → Interceptors → Pipes → Filters
│
├── 02-web-layer/
│   ├── README.md                               Tổng quan tầng web, Controllers & Routing
│   ├── 1-controllers-routing.md               HTTP methods, Route params, Query params, Headers
│   ├── 2-pipes-validation.md                  Built-in pipes, custom pipes, class-validator, DTOs
│   ├── 3-guards.md                            CanActivate, role guards, JWT guard
│   ├── 4-interceptors.md                      Transform response, logging, caching, timeout
│   ├── 5-exception-filters.md                 HttpException, global filters, custom exceptions
│   └── 6-openapi-swagger.md                  @ApiProperty, NestJS Swagger plugin, auth setup
│
├── 03-database/
│   ├── README.md                               Tổng quan database, ORM/ODM so sánh
│   ├── 1-typeorm.md                           Entity, Repository, DataSource, Relations, Migrations
│   ├── 2-prisma.md                            Schema-first, Prisma Client, migrations, seeding
│   ├── 3-mongoose.md                          Schema, Model, virtual, middleware, populate
│   ├── 4-transactions.md                      TypeORM transactions, Unit of Work, rollback
│   ├── 5-n-plus-one.md                        N+1 problem, eager loading, DataLoader pattern
│   └── 6-redis-integration.md                Caching, session store, pub/sub với ioredis
│
├── 04-security/
│   ├── README.md                               Tổng quan bảo mật NestJS
│   ├── 1-jwt-authentication.md                JWT signing, refresh tokens, blacklist với Redis
│   ├── 2-oauth2-passport.md                   OAuth2 flows, Passport.js strategies, OIDC
│   ├── 3-rbac-abac.md                         Role-Based Access Control, resource-based policies
│   ├── 4-rate-limiting.md                     @nestjs/throttler, Redis-backed limiter, bypass strategies
│   ├── 5-helmet-cors-csrf.md                  Security headers, CORS config, CSRF protection
│   └── 6-secret-management.md                @nestjs/config, Vault integration, AWS Secrets Manager
│
├── 05-async-messaging/
│   ├── README.md                               Tổng quan async patterns & microservice transports
│   ├── 1-microservice-transports.md           TCP, Redis, MQTT, gRPC transport options
│   ├── 2-message-event-patterns.md            @MessagePattern, @EventPattern, ClientProxy
│   ├── 3-kafka-integration.md                 kafkajs: producer, consumer, consumer groups, partitions
│   ├── 4-rabbitmq-integration.md              amqplib: exchanges, bindings, DLQ, retry policies
│   ├── 5-bullmq-job-queues.md                BullMQ: jobs, concurrency, retry, rate limiting
│   └── 6-request-reply-pattern.md            Sync vs async messaging, saga orchestration
│
├── 06-testing/
│   ├── README.md                               Chiến lược kiểm thử, test pyramid trong NestJS
│   ├── 1-unit-testing.md                      Jest: Test.createTestingModule(), mocking providers
│   ├── 2-mocking-providers.md                 jest.mock, custom providers, overrideProvider
│   ├── 3-integration-testing.md               Supertest, real database, module bootstrap
│   ├── 4-testcontainers.md                   PostgreSQL, Redis, Kafka trong Docker containers
│   └── 5-e2e-testing.md                      Full application flow, auth scenarios, teardown
│
├── 07-performance/
│   ├── README.md                               Tổng quan hiệu năng & tối ưu hóa NestJS
│   ├── 1-caching-strategies.md                CacheInterceptor, Redis cache, HTTP cache, TTL
│   ├── 2-connection-pooling.md                TypeORM pool config, PgBouncer, connection limits
│   ├── 3-compression-streaming.md             compression middleware, response streaming
│   ├── 4-profiling.md                         clinic.js, --prof flag, heap snapshots, memory leaks
│   └── 5-load-testing.md                     k6, Artillery, benchmark NestJS endpoints
│
├── 08-architecture/
│   ├── README.md                               Tổng quan kiến trúc & design patterns
│   ├── 1-clean-architecture.md                Layers, dependency rule, Ports & Adapters
│   ├── 2-cqrs-pattern.md                      @nestjs/cqrs, CommandBus, QueryBus, EventBus
│   ├── 3-microservices.md                     Decomposition strategies, inter-service communication
│   ├── 4-event-driven.md                     Event sourcing, CQRS + ES, saga pattern
│   ├── 5-ddd.md                              Aggregate, Entity, Value Object, Bounded Context
│   └── 6-api-gateway-bff.md                  Gateway patterns, aggregation, BFF pattern
│
├── 09-deployment/
│   ├── README.md                               Tổng quan triển khai & vận hành NestJS
│   ├── 1-docker.md                            Multi-stage build, .dockerignore, health check
│   ├── 2-kubernetes.md                        Deployment, Service, HPA, ConfigMap, Secret, Ingress
│   ├── 3-pm2.md                               Process management, cluster mode, ecosystem file
│   ├── 4-cicd-pipeline.md                     GitHub Actions, GitLab CI — build, test, push
│   ├── 5-graceful-shutdown.md                 enableShutdownHooks(), drain connections, timeout
│   └── 6-observability.md                     pino logging, OpenTelemetry, Prometheus metrics
│
├── 10-advanced/
│   ├── README.md                               Tổng quan chủ đề nâng cao
│   ├── 1-graphql.md                           @nestjs/graphql, Code-first, DataLoader, subscriptions
│   ├── 2-websockets.md                        @nestjs/websockets, Socket.IO gateway, namespaces
│   ├── 3-grpc.md                              @GrpcMethod, Protobuf definitions, client streaming
│   ├── 4-server-sent-events.md               @Sse(), Observable streams, reconnection
│   ├── 5-custom-decorators.md                Param decorators, class decorators, metadata reflection
│   └── 6-cli-schematics.md                  NestJS CLI, custom schematics, code generation
│
└── 11-interview-prep/
    ├── README.md                               Hướng dẫn chuẩn bị phỏng vấn NestJS
    ├── INTERVIEW_GUIDE.md                      Top 30 câu hỏi & đáp án chi tiết
    ├── system-design-scenarios.md              Bài toán thiết kế hệ thống với NestJS
    ├── coding-challenges.md                   Bài tập code NestJS thực tế
    ├── star-stories.md                        Mẫu câu chuyện STAR cho backend developer
    └── 90-day-study-plan.md                   Kế hoạch học 90 ngày có cấu trúc
```

---

## ✅ Trạng Thái Nội Dung

| Chủ Đề                              | Thư Mục                    | Trạng Thái | Chất Lượng    |
| ----------------------------------- | -------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**           | README.md                  | ✅         | Toàn diện     |
| **Chỉ Mục Đầy Đủ**                 | INDEX.md                   | ✅         | Đầy đủ        |
| **Nền Tảng TypeScript & NestJS**   | 01-fundamentals/           | 🚧         | Đang xây dựng |
| **Tầng Web**                       | 02-web-layer/              | 🚧         | Đang xây dựng |
| **Database & ORM**                 | 03-database/               | 🚧         | Đang xây dựng |
| **Bảo Mật**                        | 04-security/               | 🚧         | Đang xây dựng |
| **Async & Messaging**              | 05-async-messaging/        | 🚧         | Đang xây dựng |
| **Kiểm Thử**                       | 06-testing/                | 🚧         | Đang xây dựng |
| **Hiệu Năng**                      | 07-performance/            | 🚧         | Đang xây dựng |
| **Kiến Trúc**                      | 08-architecture/           | 🚧         | Đang xây dựng |
| **Triển Khai**                     | 09-deployment/             | 🚧         | Đang xây dựng |
| **Nâng Cao**                       | 10-advanced/               | 🚧         | Đang xây dựng |
| **Chuẩn Bị Phỏng Vấn**             | 11-interview-prep/         | 🚧         | Đang xây dựng |

---

## 🎯 Ưu Tiên Tạo Nội Dung (Theo Thứ Tự)

### Ưu Tiên Cao — Core NestJS Skills

- [ ] `01-fundamentals/3-dependency-injection.md` — DI container, scope, circular dep
- [ ] `01-fundamentals/5-request-lifecycle.md` — Middleware → Guards → Pipes → Filters
- [ ] `02-web-layer/2-pipes-validation.md` — class-validator, DTOs, transform
- [ ] `03-database/1-typeorm.md` — Entity, Repository, migrations
- [ ] `04-security/1-jwt-authentication.md` — JWT đầy đủ với refresh token
- [ ] `06-testing/1-unit-testing.md` — Jest + Test.createTestingModule()

### Ưu Tiên Trung Bình — Production Skills

- [ ] `05-async-messaging/3-kafka-integration.md` — Kafka với NestJS microservices
- [ ] `07-performance/1-caching-strategies.md` — Redis CacheInterceptor
- [ ] `08-architecture/2-cqrs-pattern.md` — CQRS với @nestjs/cqrs
- [ ] `09-deployment/1-docker.md` — Multi-stage Docker build
- [ ] `11-interview-prep/INTERVIEW_GUIDE.md` — Top 30 câu hỏi

### Ưu Tiên Thấp — Advanced & Reference

- [ ] `10-advanced/1-graphql.md` — GraphQL Code-first với NestJS
- [ ] `10-advanced/2-websockets.md` — Socket.IO gateway
- [ ] `08-architecture/5-ddd.md` — DDD trong NestJS
- [ ] `11-interview-prep/system-design-scenarios.md` — System design
- [ ] `11-interview-prep/90-day-study-plan.md` — Kế hoạch học

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Để Tự Học

```
1. Bắt đầu với README.md
2. Chọn Learning Path (Beginner/Intermediate/Advanced)
3. Học tuần tự qua từng module
4. Thực hành code — đừng chỉ đọc
5. Xây dựng project để consolidate kiến thức
```

### Để Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md ngay
2. Tập trung vào role cụ thể:
   - Junior: 01-fundamentals, 02-web-layer, 03-database
   - Mid: + 04-security, 05-async-messaging, 06-testing
   - Senior: + 07-performance, 08-architecture, 09-deployment
3. Chuẩn bị 3–5 câu chuyện STAR từ kinh nghiệm thực tế
4. Luyện explain concepts không cần tài liệu
```

### Để Làm Việc Thực Tế

```
Dùng như reference:
- API design: Xem 02-web-layer/ cho controller & validation
- Auth issue: Xem 04-security/1-jwt-authentication.md
- Performance issue: Xem 07-performance/ để debug
- Architecture decision: Xem 08-architecture/ trade-offs
- Deployment setup: Theo 09-deployment/ runbooks
```

### Để System Design

```
1. Xem 08-architecture/README.md để chọn pattern
2. Dùng 05-async-messaging/ cho event-driven design
3. Tham khảo 07-performance/ cho scalability
4. Kết hợp với 09-deployment/ cho infrastructure
```

---

## 📊 Ước Tính Thời Gian Học

| Module                         | Thời Gian   | Độ Khó  | Ưu Tiên    |
| ------------------------------ | ----------- | ------- | ---------- |
| Fundamentals — Nền Tảng        | 4–6 giờ     | ⭐      | Bắt buộc   |
| Web Layer — Tầng Web           | 6–8 giờ     | ⭐⭐    | Bắt buộc   |
| Database & ORM                 | 8–10 giờ    | ⭐⭐    | Bắt buộc   |
| Security — Bảo Mật             | 6–8 giờ     | ⭐⭐    | Bắt buộc   |
| Testing — Kiểm Thử             | 4–6 giờ     | ⭐⭐    | Bắt buộc   |
| Async & Messaging              | 8–10 giờ    | ⭐⭐⭐  | Bắt buộc   |
| Performance — Hiệu Năng        | 6–8 giờ     | ⭐⭐⭐  | Nên có     |
| Architecture — Kiến Trúc       | 10–15 giờ   | ⭐⭐⭐  | Nên có     |
| Deployment — Triển Khai        | 6–8 giờ     | ⭐⭐    | Nên có     |
| Advanced — Nâng Cao            | 15–20 giờ   | ⭐⭐⭐  | Tùy chọn   |

**Tổng cộng: 73–99 giờ để nắm vững NestJS backend**

---

## 🎓 Mức Độ Kỹ Năng Được Hỗ Trợ

### Beginner — Mới Bắt Đầu (0–1 năm)

- [ ] Dependency Injection và IoC (Inversion of Control — Đảo Ngược Điều Khiển)
- [ ] Xây dựng REST API với Controllers và Services
- [ ] Kết nối database với TypeORM hoặc Prisma
- [ ] Xác thực người dùng với JWT
- [ ] Validation với class-validator và DTOs (Data Transfer Objects)

**Thời gian đạt được:** 2–3 tháng

### Intermediate — Trung Cấp (1–3 năm)

- [ ] Thiết kế Guards và Interceptors tùy chỉnh
- [ ] Tích hợp message queue (Kafka / RabbitMQ)
- [ ] Implement CQRS (Command Query Responsibility Segregation) pattern
- [ ] Viết integration tests đầy đủ với Testcontainers
- [ ] Deploy ứng dụng với Docker & Kubernetes

**Thời gian đạt được:** 2–3 tháng để nâng cao

### Advanced — Nâng Cao (3–5+ năm)

- [ ] Thiết kế microservices system với event-driven architecture
- [ ] Implement Event Sourcing & DDD (Domain-Driven Design — Thiết Kế Hướng Miền)
- [ ] GraphQL federation và subscriptions
- [ ] Tối ưu performance ở production (profiling, caching, query tuning)
- [ ] Kubernetes với HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) và autoscaling

**Thời gian đạt được:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                         | Vị Trí                                                                                    |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| Tổng quan nhanh                 | [README.md](README.md)                                                                    |
| DI & Module system              | [01-fundamentals/3-dependency-injection.md](01-fundamentals/3-dependency-injection.md)   |
| Request lifecycle               | [01-fundamentals/5-request-lifecycle.md](01-fundamentals/5-request-lifecycle.md)         |
| Validation & DTOs               | [02-web-layer/2-pipes-validation.md](02-web-layer/2-pipes-validation.md)                 |
| JWT authentication              | [04-security/1-jwt-authentication.md](04-security/1-jwt-authentication.md)               |
| TypeORM & Migrations            | [03-database/1-typeorm.md](03-database/1-typeorm.md)                                     |
| Kafka integration               | [05-async-messaging/3-kafka-integration.md](05-async-messaging/3-kafka-integration.md)   |
| CQRS pattern                    | [08-architecture/2-cqrs-pattern.md](08-architecture/2-cqrs-pattern.md)                   |
| Docker deployment               | [09-deployment/1-docker.md](09-deployment/1-docker.md)                                   |
| Câu hỏi phỏng vấn               | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md)             |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và tự theo dõi:

```markdown
## NestJS Knowledge Completion

### Phase 1: Nền Tảng (Tuần 1–2)

- [ ] TypeScript decorators & generics
- [ ] Module system & Dynamic Modules
- [ ] Dependency Injection container
- [ ] Lifecycle hooks
- [ ] Request lifecycle (Middleware → Guards → Pipes → Filters)

### Phase 2: Core Skills (Tuần 3–6)

- [ ] Controllers, Routes, DTOs
- [ ] Pipes & Validation (class-validator)
- [ ] Guards & JWT Auth
- [ ] Interceptors & Exception Filters
- [ ] TypeORM / Prisma với NestJS
- [ ] Transactions & N+1 prevention
- [ ] Unit testing với Jest

### Phase 3: Advanced (Tuần 7–10)

- [ ] Kafka / RabbitMQ microservices transport
- [ ] CQRS với @nestjs/cqrs
- [ ] Redis caching (CacheInterceptor)
- [ ] Docker & Kubernetes deployment
- [ ] Integration tests với Testcontainers

### Phase 4: Specialization (Tuần 11+)

- [ ] Event Sourcing & DDD
- [ ] GraphQL với @nestjs/graphql
- [ ] gRPC transport
- [ ] System design scenarios
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích DI container và request lifecycle không cần tài liệu
- [ ] Thiết kế REST API đầy đủ với auth, validation, error handling
- [ ] Viết TypeORM entities với relations và migrations
- [ ] Debug memory leak và async bugs trong NestJS
- [ ] Viết test suite đầy đủ với coverage > 80%

### ✅ Năng Lực Vận Hành

- [ ] Thiết kế microservice system với message queue
- [ ] Deploy NestJS app lên Kubernetes với health checks
- [ ] Implement Redis caching có TTL (Time-to-Live) hợp lý
- [ ] Respond với production incident (memory leak, slow endpoint)
- [ ] Implement security best practices end-to-end

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời top 30 câu hỏi NestJS tự tin
- [ ] Kể 2–3 câu chuyện incident (format STAR)
- [ ] Thiết kế hệ thống có xét đến scalability và fault tolerance
- [ ] Thảo luận trade-off giữa monolith và microservices có lý
- [ ] Live code bài toán NestJS trong 30 phút

---

## 💡 Pro Tips — Mẹo Học Hiệu Quả

1. **Hiểu DI container thật sâu:** Đây là trái tim của NestJS, phân biệt NestJS với Express thuần
2. **Học bằng cách làm:** Đừng chỉ đọc — tự viết lại từng ví dụ, thêm edge cases
3. **Debug request lifecycle:** Đặt console.log trong từng tầng để hiểu thứ tự thực thi
4. **Đọc source code NestJS:** Hiểu cách decorators hoạt động giúp debug tốt hơn nhiều
5. **Test-first mindset:** Viết test trước code — TDD (Test-Driven Development) làm API design rõ ràng hơn
6. **Benchmark trước khi optimize:** Dùng k6 hoặc Artillery, đừng đoán mò
7. **Document quyết định kiến trúc:** Ghi lại tại sao chọn CQRS, không chỉ ghi cách implement
8. **Xây project thật:** Todo API → Blog backend → E-commerce microservices (tăng dần complexity)

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

Đây là tài liệu sống, luôn chào đón đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm ví dụ thực tế từ kinh nghiệm cá nhân
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Thêm bài tập thực hành cho từng module
- [ ] Cập nhật khi NestJS ra version mới

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0 — README & INDEX Complete
**Trạng Thái:** ✅ README.md | ✅ INDEX.md | 🚧 Các module đang xây dựng
