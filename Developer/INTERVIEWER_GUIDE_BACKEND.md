# Hướng Dẫn Phỏng Vấn Backend Developer (Middle → Senior)

> Tài liệu dành cho **người phỏng vấn** — không phải ứng viên. Mục tiêu: đặt câu hỏi có cấu trúc, nhận diện level thực tế, và đánh giá công bằng dựa trên **cách ứng viên suy nghĩ**, không chỉ thuộc lòng đáp án.

## Mục Lục

1. [Nguyên Tắc Phỏng Vấn](#1-nguyên-tắc-phỏng-vấn)
2. [Khung Đánh Giá Level](#2-khung-đánh-giá-level)
3. [Cấu Trúc Buổi Phỏng Vấn](#3-cấu-trúc-buổi-phỏng-vấn)
4. [Câu Hỏi Theo Topic — Backend & Node.js](#4-câu-hỏi-theo-topic--backend--nodejs)
5. [System Design & Live Coding](#5-system-design--live-coding)
6. [Behavioral & Leadership (Senior)](#6-behavioral--leadership-senior)
7. [Scorecard & Quyết Định Tuyển Dụng](#7-scorecard--quyết-định-tuyển-dụng)
8. [Liên Kết](#8-liên-kết)

---

## 1. Nguyên Tắc Phỏng Vấn

### Đánh giá gì?

| Tiêu chí | Trọng số gợi ý | Mô tả |
| -------- | -------------- | ----- |
| **Problem solving** | 30% | Cách tiếp cận, đặt câu hỏi làm rõ, xử lý edge case |
| **Technical depth** | 30% | Hiểu *tại sao*, không chỉ *là gì* |
| **Production experience** | 20% | Ví dụ thực tế, incident, trade-off đã trải qua |
| **Communication** | 10% | Giải thích rõ ràng, cấu trúc câu trả lời |
| **Culture fit** | 10% | Collaboration, ownership, học hỏi |

### Nguyên tắc vàng

1. **Một câu hỏi → nhiều follow-up** — đào sâu thay vì hỏi 50 câu nông.
2. **Không hỏi trivia** — ưu tiên kiến thức áp dụng được trong production.
3. **Cho thời gian suy nghĩ** — im lặng 10–15 giây là bình thường.
4. **Ghi chú signal, không ghi đáp án** — "đề cập connection pool exhaustion" quan trọng hơn "đúng/sai".
5. **Calibration** — so sánh ứng viên với rubric, không với bản thân interviewer.

### Cách đặt follow-up hiệu quả

```
"Giả sử traffic tăng 10x — điều gì break trước?"
"Bạn đã gặp bug này trong production chưa? Xử lý thế nào?"
"Nếu không dùng Redis, bạn chọn giải pháp gì?"
"Trade-off của approach này là gì?"
```

---

## 2. Khung Đánh Giá Level

### Middle Backend (2–4 năm kinh nghiệm)

**Kỳ vọng:**
- Tự triển khai feature end-to-end (API → DB → test cơ bản)
- Hiểu REST, transaction, indexing, caching ở mức áp dụng
- Debug được lỗi thường gặp (N+1, memory leak cơ bản, timeout)
- Viết code readable, có error handling

**Red flags:**
- Chỉ biết dùng framework, không giải thích được request lifecycle
- Không từng xử lý production incident
- Trả lời mơ hồ khi hỏi "tại sao chọn cách này"

### Senior Backend (4+ năm, ownership rõ ràng)

**Kỳ vọng:**
- Thiết kế module/service, đưa ra trade-off có căn cứ
- Tối ưu performance, security, reliability ở scale thực tế
- Mentor junior, review code có chất lượng
- System design: ước lượng capacity, chọn pattern phù hợp context

**Red flags:**
- Biết buzzword (CQRS, event sourcing) nhưng không giải thích được khi nào *không* dùng
- Không có ví dụ production cụ thể (metrics, latency, incident)
- Over-engineer mọi bài toán

### Bảng signal nhanh

| Signal | Middle ✅ | Senior ✅ | Cả hai 🚩 |
| ------ | --------- | --------- | --------- |
| Giải thích concept | Đúng ý chính | Thêm trade-off, edge case, production story | Thuộc lòng nhưng không giải thích được |
| Debug scenario | Tìm hướng hợp lý | Structured approach + metrics/logs | Đoán mò, không hỏi thêm context |
| System design | Components cơ bản đúng | Bottleneck, failure mode, cost | Vẽ diagram đẹp nhưng thiếu data flow |
| Code | Working + readable | Idempotent, testable, observable | Copy pattern không hiểu |

---

## 3. Cấu Trúc Buổi Phỏng Vấn

### Middle (90 phút)

| Phút | Phần | Nội dung |
| ---- | ---- | -------- |
| 5 | Warm-up | Giới thiệu, overview CV |
| 15 | Experience deep dive | 1 project ứng viên tự chọn — hỏi architecture, role |
| 35 | Technical Q&A | 4–5 topic, 2–3 câu/topic + follow-up |
| 25 | Live coding | API endpoint hoặc fix bug |
| 10 | Q&A ngược | Ứng viên hỏi team |

### Senior (120 phút)

| Phút | Phần | Nội dung |
| ---- | ---- | -------- |
| 10 | Warm-up + expectations | Role, scope ownership |
| 20 | Experience | System đã build/lead — scale, incident, decision |
| 30 | Technical deep dive | 3 topic sâu + trade-off |
| 35 | System design | 1 scenario (URL shortener, notification, order system) |
| 15 | Behavioral | STAR — conflict, mentoring, technical decision |
| 10 | Q&A ngược | |

---

## 4. Câu Hỏi Theo Topic — Backend & Node.js

Mỗi câu gồm: **Câu hỏi** → **Follow-up** → **Gợi ý đáp án** → **Rubric Middle/Senior**

**Danh sách topic:** HTTP & API (1) · Database (2) · Caching (3) · **Node.js (4)** · Architecture (5) · DevOps (6) · Concurrency (7)

---

### Topic 1: HTTP & API Design

#### Q1.1: RESTful API — bạn thiết kế resource `orders` thế nào?

**Follow-up:**
- Pagination, filtering, sorting implement ra sao?
- Versioning API khi breaking change?
- Idempotency cho `POST /payments`?

**Gợi ý đáp án:**
- Resource naming số nhiều: `GET /orders`, `GET /orders/:id`, `POST /orders`
- HTTP verb đúng semantics: `PATCH` partial update, `PUT` full replace
- Status code có ý nghĩa: 201 + Location header, 409 conflict, 422 validation
- Pagination: cursor-based cho dataset lớn, offset cho admin UI nhỏ
- Idempotency-Key header cho payment/order creation

| Level | Signal |
| ----- | ------ |
| **Middle** | Đặt endpoint đúng, biết status code cơ bản, có pagination |
| **Senior** | Cursor vs offset trade-off, versioning strategy (URL vs header), idempotency + retry safety |
| **🚩** | Mọi thứ dùng POST; không phân biệt 400 vs 422 vs 500 |

---

#### Q1.2: Authentication flow — JWT access + refresh token

**Follow-up:**
- Lưu refresh token ở đâu? Cookie vs localStorage?
- Token bị compromise — xử lý thế nào?
- Logout all devices?

**Gợi ý đáp án:**
- Access token ngắn hạn (15m), refresh token dài hạn (7d), lưu refresh trong httpOnly secure cookie
- Refresh token rotation: mỗi lần refresh cấp token mới, invalidate cũ
- Blacklist/revocation list hoặc session store (Redis) cho logout
- Không lưu access token trong localStorage (XSS risk)

| Level | Signal |
| ----- | ------ |
| **Middle** | Giải thích được flow cơ bản, biết JWT structure |
| **Senior** | Rotation, revocation strategy, threat model (XSS/CSRF), mention OAuth2 khi SSO |
| **🚩** | "JWT stateless nên không cần logout" — thiếu hiểu revocation |

📖 Tham chiếu: [05-security.md](./basic/05-security.md)

---

#### Q1.3: CORS — browser gọi API cross-origin hoạt động thế nào?

**Follow-up:**
- Preflight request (`OPTIONS`) kích hoạt khi nào?
- `Access-Control-Allow-Credentials: true` — lưu ý gì?
- Mobile app / server-to-server có cần CORS không?

**Gợi ý đáp án:**
- Same-origin policy: browser chặn JS đọc response từ domain khác (protocol + host + port)
- Server phải trả header: `Access-Control-Allow-Origin`, `Allow-Methods`, `Allow-Headers`
- **Preflight**: request "không đơn giản" (custom header, PUT/DELETE, `Content-Type: application/json`) → browser gửi `OPTIONS` trước
- Credentials (cookie): không dùng `Allow-Origin: *` — phải chỉ định origin cụ thể + `Allow-Credentials: true`
- Mobile native / backend-to-backend: không qua browser → **không** áp dụng CORS (nhưng vẫn cần auth)

| Level | Signal |
| ----- | ------ |
| **Middle** | Giải thích được preflight, biết cấu hình Allow-Origin cơ bản |
| **Senior** | Wildcard vs whitelist origin, CSRF liên quan cookie cross-site, API gateway CORS layer |
| **🚩** | `Allow-Origin: *` + credentials; hoặc "CORS là security ở server" (nhầm — CORS là browser policy) |

---

#### Q1.4: Rate limiting cho public API — thiết kế thế nào?

**Follow-up:**
- Token bucket vs sliding window — khác gì?
- 4 instances API — rate limit 100 req/min/user hoạt động distributed thế nào?
- Trả response gì khi bị limit? Client retry ra sao?

**Gợi ý đáp án:**
- **Token bucket**: bucket chứa token, refill theo rate — cho phép burst ngắn
- **Fixed window**: đếm theo cửa sổ cố định (phút) — dễ implement, có edge burst ở boundary
- **Sliding window**: mượt hơn fixed window — Redis ZSET timestamp hoặc sliding window log
- Distributed: shared store (Redis) thay in-memory Map — key `ratelimit:{userId}:{window}`
- Response: `429 Too Many Requests` + header `Retry-After`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Phân tier: per IP (anonymous), per API key (authenticated), per endpoint (expensive ops)

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết 429, implement rate limit cơ bản (middleware + counter) |
| **Senior** | Distributed Redis limit, sliding window, burst vs steady rate, bypass cho internal traffic |
| **🚩** | In-memory limit trên multi-instance; không có Retry-After header |

📖 Tham chiếu: [05-security.md](./basic/05-security.md)

---

#### Q1.5: REST vs GraphQL vs gRPC — khi nào chọn approach nào?

**Follow-up:**
- GraphQL N+1 problem — giải quyết thế nào?
- gRPC streaming use case?
- Public API cho third-party — chọn gì?

**Gợi ý đáp án:**

| | REST | GraphQL | gRPC |
| - | ---- | ------- | ---- |
| **Phù hợp** | CRUD public API, cache HTTP | Mobile/BFF, nhiều client khác field needs | Internal microservice, high perf |
| **Ưu** | Đơn giản, cache, tooling | Flexible query, 1 round-trip | Binary protobuf, streaming, contract |
| **Nhược** | Over/under-fetching | Complexity, query cost control | Browser hạn chế, learning curve |

- GraphQL N+1: DataLoader batch loading, query depth/complexity limit
- gRPC streaming: real-time feed, large file transfer, bidirectional stream
- Public third-party: REST + OpenAPI (familiar, cacheable) hoặc GraphQL có rate limit + persisted queries

| Level | Signal |
| ----- | ------ |
| **Middle** | Mô tả được ưu nhược từng loại, đã dùng REST thành thạo |
| **Senior** | Chọn theo context (team, client, perf), GraphQL cost analysis, gRPC + API gateway |
| **🚩** | "GraphQL thay REST mọi nơi" hoặc không biết gRPC dùng internal |

📖 Tham chiếu: [07-deep-technical-knowledge.md](./basic/07-deep-technical-knowledge.md)

---

### Topic 2: Database & Data Modeling

#### Q2.1: ACID và transaction isolation — giải thích và cho ví dụ

**Follow-up:**
- Phantom read là gì? Level nào prevent?
- Khi nào chấp nhận READ COMMITTED thay vì SERIALIZABLE?
- Optimistic vs pessimistic locking?

**Gợi ý đáp án:**
- ACID: Atomicity, Consistency, Isolation, Durability
- Isolation levels: READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
- Phantom read: row mới xuất hiện trong range query — REPEATABLE READ (PostgreSQL) hoặc SERIALIZABLE
- Optimistic locking: version column, retry on conflict — phù hợp low contention
- Pessimistic: `SELECT FOR UPDATE` — high contention, cần chắc chắn

| Level | Signal |
| ----- | ------ |
| **Middle** | Giải thích ACID, biết dùng transaction trong code |
| **Senior** | Chọn isolation level theo use case, dead lock handling, ví dụ race condition thực tế |
| **🚩** | Mọi transaction đều SERIALIZABLE "cho chắc" |

---

#### Q2.2: N+1 query problem — phát hiện và fix

**Follow-up:**
- Làm sao phát hiện trong production?
- DataLoader pattern hoạt động thế nào?
- Khi eager load quá nhiều — vấn đề gì?

**Gợi ý đáp án:**
- N+1: 1 query lấy list + N query lấy relation cho từng item
- Fix: JOIN/eager loading, batch query (`WHERE id IN (...)`), DataLoader
- Phát hiện: slow query log, APM (query count per request), ORM debug
- Eager load thừa → over-fetching, memory tăng

| Level | Signal |
| ----- | ------ |
| **Middle** | Nhận ra N+1, fix bằng include/join |
| **Senior** | DataLoader cho GraphQL, monitoring query count, balance eager vs lazy |
| **🚩** | Không biết N+1; hoặc "luôn eager load hết" |

📖 Tham chiếu: [03-database.md](./basic/03-database.md)

---

#### Q2.3: Indexing — khi nào index giúp, khi nào làm chậm?

**Follow-up:**
- Composite index `(a, b, c)` — query nào dùng được?
- Covering index là gì?
- Khi nào *không* nên thêm index?

**Gợi ý đáp án:**
- B-tree index: equality + range trên leftmost prefix
- Composite `(status, created_at)`: `WHERE status = ? ORDER BY created_at` ✅
- Write amplification: mỗi INSERT/UPDATE phải update index
- Không index: column low cardinality (boolean), bảng nhỏ, write-heavy với ít read

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết index column hay query, EXPLAIN cơ bản |
| **Senior** | Composite prefix rule, partial index, index maintenance cost |
| **🚩** | Index mọi column |

---

#### Q2.4: SQL vs NoSQL — khi nào chọn loại nào?

**Follow-up:**
- E-commerce order system — SQL hay NoSQL? Tại sao?
- MongoDB embed document vs reference — trade-off?
- Khi nào cần cả hai (polyglot persistence)?

**Gợi ý đáp án:**
- **SQL (PostgreSQL, MySQL)**: structured data, ACID transaction, complex join, strong consistency
- **NoSQL document (MongoDB)**: flexible schema, nested document, horizontal scale, eventual consistency OK
- **NoSQL key-value (Redis)**: cache, session — không thay DB chính
- **NoSQL wide-column (Cassandra)**: write-heavy, time-series, multi-region
- Order/payment → SQL (transaction, referential integrity)
- Product catalog linh hoạt → document DB hoặc JSON column trong PostgreSQL
- Polyglot: PostgreSQL (transactional) + Elasticsearch (search) + Redis (cache)

| Level | Signal |
| ----- | ------ |
| **Middle** | Phân biệt use case cơ bản, đã dùng ít nhất 1 SQL DB |
| **Senior** | Polyglot persistence, CAP trade-off cụ thể, migration path, không religious |
| **🚩** | "NoSQL luôn nhanh hơn SQL" hoặc chọn DB theo trend không theo requirement |

📖 Tham chiếu: [03-database.md](./basic/03-database.md)

---

#### Q2.5: Normalization vs denormalization — cân bằng thế nào?

**Follow-up:**
- 3NF là gì? Khi nào chấp nhận duplicate data?
- `order_items` lưu `product_name` snapshot — có nên không?
- Materialized view dùng khi nào?

**Gợi ý đáp án:**
- **Normalization**: giảm redundancy, update anomaly — 1NF → 2NF → 3NF (loại functional dependency)
- **Denormalization**: duplicate data có chủ đích — giảm join, tăng read perf
- Order item snapshot `product_name`, `price_at_purchase` — **nên** (product đổi giá/tên sau này không ảnh hưởng lịch sử)
- Denormalize khi: read >> write, join expensive, reporting/analytics
- Materialized view: pre-compute aggregation, refresh theo schedule hoặc trigger
- Trade-off: consistency phức tạp hơn khi denormalize — cần sync strategy

| Level | Signal |
| ----- | ------ |
| **Middle** | Hiểu duplicate vs normalize, biết snapshot giá trong order |
| **Senior** | Eventual consistency khi denormalize, materialized view refresh, CQRS read model |
| **🚩** | Normalize tuyệt đối mọi nơi; hoặc denormalize không có lý do |

---

#### Q2.6: Read replica — replication lag xử lý thế nào?

**Follow-up:**
- User tạo post rồi refresh — không thấy post (stale read). Fix?
- Read replica vs connection pool routing?
- Failover khi primary down — RPO/RTO nghĩa là gì?

**Gợi ý đáp án:**
- Primary (write) + replica (read) — async replication → **replication lag** (ms đến vài giây)
- **Read-your-writes**: sau write, route read về primary hoặc session stickiness trong lag window
- Sticky session: `last_write_timestamp` — read replica chỉ khi `now - last_write > lag_threshold`
- Monitoring: `pg_stat_replication.replay_lag`, alert khi lag > SLA
- Failover: promote replica → primary; RPO (data loss window), RTO (downtime window)
- Không đọc replica cho critical path ngay sau write (payment status, auth)

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết primary/replica, hiểu lag có thể xảy ra |
| **Senior** | Read-your-writes pattern, lag monitoring, failover story, synchronous replica khi cần |
| **🚩** | "Replica luôn đồng bộ realtime"; route mọi read về replica ngay sau write |

📖 Tham chiếu: [03-database.md](./basic/03-database.md) | [09-computer-science-fundamentals/07-data-modeling.md](./basic/09-computer-science-fundamentals/07-data-modeling.md)

---

### Topic 3: Caching & Performance

#### Q3.1: Cache-aside pattern — mô tả flow và failure modes

**Follow-up:**
- Cache stampede / thundering herd — prevent thế nào?
- TTL chọn như thế nào?
- Cache invalidation strategy?

**Gợi ý đáp án:**
- Flow: read cache → miss → read DB → write cache → return
- Stampede: nhiều request cùng miss → lock/mutex, early expiration jitter, request coalescing
- Invalidation: TTL, event-driven (pub/sub on update), version key
- "There are only two hard things: cache invalidation and naming things"

| Level | Signal |
| ----- | ------ |
| **Middle** | Mô tả cache-aside, biết dùng Redis |
| **Senior** | Stampede prevention, consistency trade-off, cache warming |
| **🚩** | Cache vĩnh viễn không invalidation |

---

#### Q3.2: Connection pool — sizing và exhaustion

**Follow-up:**
- Pool size 100 có nghĩa là gì với 4 app instances?
- Triệu chứng pool exhausted?
- `max_connections` PostgreSQL vs pool size?

**Gợi ý đáp án:**
- Mỗi instance có pool riêng: 4 instances × 20 connections = 80 DB connections
- Rule of thumb: `pool_size ≈ (core_count × 2) + effective_spindle_count` (cần tune theo workload)
- Exhaustion: requests hang, timeout, `TimeoutError acquiring connection`
- DB `max_connections` phải > tổng pool tất cả instances + admin connections

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết dùng pool, không mở connection mỗi request |
| **Senior** | Multi-instance math, monitoring pool metrics, pgBouncer khi cần |
| **🚩** | `pool: { max: 1000 }` không hiểu tại sao |

---

### Topic 4: Node.js

> Thay thế phần Security & Testing — dùng khi phỏng vấn ứng viên **Node.js/TypeScript backend**. Chi tiết thêm: [INTERVIEWER_GUIDE.md](./nodejs/11-interview-prep/INTERVIEWER_GUIDE.md)

#### Q4.1: Event Loop hoạt động như thế nào?

**Follow-up:**
- Microtask vs macrotask — thứ tự thực thi?
- `process.nextTick()` vs `setImmediate()`?
- Cho đoạn code — predict output thứ tự `console.log`?

**Gợi ý đáp án:**
- Single main thread chạy JS; async I/O delegate cho libuv thread pool
- Phases: timers → pending → idle → poll → check → close
- Microtasks (Promise, `nextTick`) chạy sau sync code, trước macrotask tiếp theo
- `nextTick` ưu tiên hơn Promise; `setImmediate` chạy ở check phase sau I/O
- Output mẫu: sync → nextTick → Promise → setTimeout

| Level | Signal |
| ----- | ------ |
| **Middle** | Giải thích call stack + callback queue, biết Promise trước setTimeout |
| **Senior** | libuv phases, nextTick starvation, thread pool cho `fs`/`crypto` |
| **🚩** | "Node single-threaded nên không cần quan tâm async" |

📖 Tham chiếu: [1-event-loop-questions.md](./nodejs/11-interview-prep/1-event-loop-questions.md)

---

#### Q4.2: `async/await` — parallel vs sequential và error handling

**Follow-up:**
- `await` trong `for` loop — chạy tuần tự hay song song?
- `Promise.all` vs `Promise.allSettled` — khi nào dùng?
- Unhandled rejection trong production — xử lý thế nào?

**Gợi ý đáp án:**
- `for...of` + `await` = sequential; `Promise.all(items.map(fn))` = parallel
- `Promise.all`: fail-fast; `allSettled`: chờ hết, trả status từng promise
- `process.on('unhandledRejection')` + log; ESLint `no-floating-promises`
- Express: wrap async handler `.catch(next)` hoặc `express-async-errors`

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết parallel vs sequential, try/catch cơ bản |
| **Senior** | Timeout với `Promise.race`, abort controller, graceful shutdown on rejection |
| **🚩** | Fire async trong route không await — unhandled rejection |

---

#### Q4.3: Express middleware flow và async error handling

**Follow-up:**
- Thứ tự middleware quan trọng thế nào?
- Error middleware signature?
- Express vs NestJS — khi nào chọn?

**Gợi ý đáp án:**
- Middleware chạy theo thứ tự đăng ký: body parser → auth → routes → **error handler cuối**
- Error handler: 4 params `(err, req, res, next)`
- Async throw không tự catch trong Express 4 → cần wrapper
- Express: linh hoạt, ecosystem lớn; NestJS: DI, structure, enterprise; Fastify: performance

| Level | Signal |
| ----- | ------ |
| **Middle** | Viết middleware, biết `next(err)` |
| **Senior** | Error taxonomy (operational vs programmer), NestJS guard/interceptor flow |
| **🚩** | Error handler đặt trước routes; `res.status(500).send(err.stack)` prod |

📖 Tham chiếu: [2-api-design-questions.md](./nodejs/11-interview-prep/2-api-design-questions.md)

---

#### Q4.4: Event Loop bị block và memory leak — triệu chứng production

**Follow-up:**
- `bcrypt.hashSync` trong request handler — vấn đề gì?
- EventEmitter `MaxListenersExceededWarning` nghĩa là gì?
- Làm sao đo event loop lag?

**Gợi ý đáp án:**
- CPU-bound sync code block main thread → latency spike toàn process
- Fix: `await bcrypt.hash` (thread pool), Worker Threads, queue job (BullMQ)
- Memory leak: global cache không TTL, forgot `removeListener`, closure giữ ref lớn
- Đo: `perf_hooks.monitorEventLoopDelay`, clinic.js, heap snapshot compare

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết tránh sync crypto/JSON.parse file lớn trên main thread |
| **Senior** | clinic.js flame graph, heap diff, worker pool sizing |
| **🚩** | Restart pod là fix duy nhất; `while(true)` trong middleware |

📖 Tham chiếu: [07-performance/README.md](./nodejs/07-performance/README.md)

---

#### Q4.5: Scale Node.js API lên nhiều instances — lưu ý gì?

**Follow-up:**
- In-memory cache trên 4 pods — vấn đề gì?
- Graceful shutdown khi K8s gửi SIGTERM?
- Socket.io multi-instance cần gì?

**Gợi ý đáp án:**
- Stateless API → scale freely; session/cache phải dùng Redis, không in-memory
- SIGTERM: `server.close()` → drain in-flight → disconnect DB → exit
- K8s: `preStop` hook + `terminationGracePeriodSeconds`
- Socket.io: Redis adapter pub/sub cross instance
- `new PrismaClient()` singleton per process, không per request

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết stateless design, `server.close()` on SIGTERM |
| **Senior** | Connection storm serverless, BullMQ worker scale riêng, readiness probe coordination |
| **🚩** | In-memory Map session trên multi-instance |

📖 Tham chiếu: [5-system-design-scenarios.md](./nodejs/11-interview-prep/5-system-design-scenarios.md)

---

### Topic 5: Architecture & Distributed Systems

> Ưu tiên câu hỏi **tình huống production** — hỏi "bạn đã làm thế nào" và "nếu X fail thì sao", tránh thuần định nghĩa lý thuyết.

**Mức độ:** ⭐ Dễ · ⭐⭐ Trung bình · ⭐⭐⭐ Khó

#### Q5.1: Monolith vs Microservices — khi nào tách service?

**Follow-up:**
- Bạn đã tách service nào? Metric trước/sau?
- Distributed transaction xử lý thế nào?
- API Gateway vs BFF?

**Gợi ý đáp án:**
- Tách khi: team scale, independent deploy cần thiết, domain boundary rõ
- Không tách khi: team nhỏ, chưa có observability, "vì trendy"
- Distributed tx: Saga (choreography/orchestration), eventual consistency, outbox pattern
- Không dùng 2PC trong hầu hết trường hợp web scale

| Level | Signal |
| ----- | ------ |
| **Middle** | Hiểu khái niệm, ưu nhược cơ bản |
| **Senior** | Ví dụ thực tế, saga/outbox, team topology (Conway's law) |
| **🚩** | Microservices cho mọi project; hoặc "monolith luôn tốt hơn" |

📖 Tham chiếu: [01-system-design.md](./basic/01-system-design.md)

---

#### Q5.2: Message queue — Kafka vs RabbitMQ, khi nào dùng?

**Follow-up:**
- At-least-once delivery — xử lý duplicate message?
- Dead letter queue (DLQ)?
- Ordering guarantee?

**Gợi ý đáp án:**
- Kafka: high throughput, log retention, replay, event streaming
- RabbitMQ: routing phức tạp, task queue, lower latency messaging
- At-least-once: consumer idempotent (idempotency key, upsert)
- DLQ: message fail sau N retry → manual investigation
- Ordering: partition key trong Kafka; single consumer per queue

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết dùng queue cho async job |
| **Senior** | Delivery semantics, poison message, backpressure |
| **🚩** | Fire-and-forget không handle failure |

---

#### Q5.3: CAP theorem — giải thích trong context thực tế

**Follow-up:**
- Database bạn dùng thiên về CP hay AP?
- Eventual consistency — ví dụ user thấy data cũ?
- Circuit breaker khi nào cần?

**Gợi ý đáp án:**
- CAP: trong partition, chọn Consistency hoặc Availability (Partition tolerance bắt buộc trong distributed)
- PostgreSQL: CP (strong consistency trong single cluster)
- Cassandra/DynamoDB: AP tunable
- Eventual consistency: read-your-writes, session consistency giảm confusion
- Circuit breaker: downstream fail liên tục → fail fast, tránh cascade

| Level | Signal |
| ----- | ------ |
| **Middle** | Giải thích 3 chữ cái đúng ý |
| **Senior** | PACELC, practical consistency models, circuit breaker + retry policy |
| **🚩** | "CAP nghĩa là chỉ chọn 2 trong 3 mãi mãi" |

---

#### Q5.4: ⭐ Dễ — Tách layer trong monolith: project bạn tổ chức code thế nào?

**Tình huống:** Team 5 người, 1 repo Node.js/NestJS, chưa tách microservice.

**Follow-up:**
- Logic validate order nằm ở controller hay service?
- Khi nào tách module theo domain (`orders/`, `payments/`)?
- Dấu hiệu code đang "spaghetti"?

**Gợi ý đáp án:**
- Controller/route: nhận request, validate input, gọi service, trả response — **không** SQL trực tiếp
- Service: business rule (tính giá, check tồn kho, state transition)
- Repository/DAO: query DB, map entity ↔ DTO
- Module theo domain khi file > 300 dòng hoặc 2 team cùng sửa 1 folder
- Spaghetti: controller 200 dòng, `import` chéo module, test không mock được DB

| Level | Signal |
| ----- | ------ |
| **Middle** | Mô tả được 3 layer, đã refactor ít nhất 1 lần |
| **Senior** | Boundary rõ, DTO không leak entity, ví dụ cụ thể module đã tách |
| **🚩** | "Một file `index.js` 2000 dòng nhưng chạy được" |

---

#### Q5.5: ⭐ Dễ — User đặt hàng xong cần gửi email xác nhận — bạn implement thế nào?

**Tình huống:** `POST /orders` thành công, cần gửi email trong vài giây.

**Follow-up:**
- Gửi email sync trong request handler — rủi ro gì?
- SMTP chậm 5s — user phải chờ?
- Email fail nhưng order đã tạo — xử lý thế nào?

**Gợi ý đáp án:**
- **Không** gọi SMTP sync trong request — user chờ lâu, timeout, order rollback oan
- Đúng: lưu order → publish job queue (BullMQ/SQS) → worker gửi email async
- Email fail: retry 3 lần exponential backoff → DLQ → alert ops; order vẫn valid
- Optional: trả 201 ngay, email "đang gửi" — UX chấp nhận delay vài giây

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết tách async job, không block response |
| **Senior** | Retry + DLQ, idempotent email send, monitoring queue depth |
| **🚩** | `await sendEmail()` trong controller; hoặc rollback order vì email fail |

---

#### Q5.6: ⭐ Dễ — Service gọi API bên thứ 3 (payment gateway) hay timeout — bạn xử lý?

**Tình huống:** Gọi Stripe/PayPal, đôi khi 30s không response.

**Follow-up:**
- Set timeout bao lâu? Retry mấy lần?
- Timeout xảy ra — biết payment đã charge chưa?
- User bấm "Thanh toán" 2 lần — sao?

**Gợi ý đáp án:**
- HTTP client timeout 5–10s (không chờ 30s default)
- Retry chỉ khi **idempotent** (GET status) — không blind retry POST charge
- Timeout → query payment status API / webhook là source of truth
- Double-click: `Idempotency-Key` gửi lên gateway, UI disable button sau click
- Log correlation ID để support tra cứu

| Level | Signal |
| ----- | ------ |
| **Middle** | Có timeout, biết không retry POST mù quáng |
| **Senior** | Reconciliation job, webhook + polling fallback, idempotency key |
| **🚩** | Không timeout; retry POST 5 lần → double charge |

---

#### Q5.7: ⭐⭐ Trung bình — Deploy API mới lên production không downtime — bạn làm gì?

**Tình huống:** K8s, 3 pods đang chạy v1, cần deploy v2 có đổi schema DB.

**Follow-up:**
- Rolling update vs blue/green — team bạn dùng gì?
- v2 cần column mới — deploy DB migration trước hay sau app?
- Pod cũ còn xử lý request khi SIGTERM?

**Gợi ý đáp án:**
- Rolling update: K8s thay từng pod; readiness probe pass mới nhận traffic
- **Expand-contract migration**: thêm column nullable trước → deploy app đọc/ghi cả 2 → migrate data → deploy app chỉ dùng mới → xóa cũ
- Không deploy breaking schema + code cùng lúc
- `preStop` + graceful shutdown: drain in-flight request trước khi kill
- Blue/green khi cần rollback nhanh (< 1 phút switch LB)

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết rolling deploy, migration chạy trước app (additive) |
| **Senior** | Expand-contract story, rollback plan, feature flag tắt code path mới |
| **🚩** | `DROP COLUMN` deploy cùng app mới; kill pod ngay không drain |

📖 Tham chiếu: [04-infrastructure-devops.md](./basic/04-infrastructure-devops.md)

---

#### Q5.8: ⭐⭐ Trung bình — Order service cần thông tin user (tên, tier VIP) — lấy data thế nào?

**Tình huống:** `order-service` cần `user.name` khi hiển thị invoice. `user-service` sở hữu data user.

**Follow-up:**
- Gọi HTTP sync mỗi lần tạo order — vấn đề gì khi Black Friday?
- Cache user info ở order-service — stale khi user đổi tên?
- Duplicate data (lưu `user_name` trong order) — trade-off?

**Gợi ý đáp án:**
- Sync HTTP: đơn giản nhưng coupling + latency + user-service down → order fail
- **Snapshot denormalize**: lưu `user_name`, `user_tier` vào `orders` lúc tạo — invoice đúng lịch sử, không phụ thuộc user-service sau đó
- Cache Redis TTL 5 phút: giảm call nhưng có stale window — OK cho non-critical
- Event `UserUpdated` → order-service update cache (eventual consistency)
- Chọn theo requirement: invoice cần snapshot; realtime dashboard có thể cache

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết gọi API hoặc lưu snapshot, nhận ra coupling |
| **Senior** | Denormalize có chủ đích, event sync, fallback khi user-service down |
| **🚩** | Shared DB giữa 2 service; hoặc sync call không timeout |

---

#### Q5.9: ⭐⭐⭐ Khó — Đặt hàng: trừ kho + charge payment — payment gateway timeout giữa chừng

**Tình huống:**
```
1. Reserve inventory ✓
2. Call payment API → timeout (không biết charge thành công hay chưa)
3. User thấy lỗi, bấm đặt lại
```

**Follow-up:**
- Rollback inventory ngay khi timeout — rủi ro gì?
- Thiết kế trạng thái order (`PENDING_PAYMENT`, `PAID`, `FAILED`)?
- Reconciliation job chạy khi nào?

**Gợi ý đáp án:**
- **Không** rollback ngay — payment có thể đã thành công, rollback → mất tiền + mất hàng
- Flow an toàn:
  1. Tạo order `PENDING_PAYMENT`, reserve inventory (TTL 15 phút)
  2. Gọi payment với idempotency key = `order_id`
  3. Timeout → giữ `PENDING`, **không** release inventory ngay
  4. Webhook hoặc reconciliation job query payment status
  5. Paid → `CONFIRMED`; Failed/expire → release inventory
- User đặt lại: cùng `order_id` / idempotency → không double charge
- Saga orchestration hoặc state machine rõ ràng — log mỗi bước

| Level | Signal |
| ----- | ------ |
| **Middle** | Nhận ra không rollback mù, có state machine cơ bản |
| **Senior** | Idempotency end-to-end, reconciliation, inventory TTL, ví dụ incident đã gặp |
| **🚩** | 2PC across services; rollback inventory ngay khi timeout |

---

#### Q5.10: ⭐⭐⭐ Khó — Newsfeed/dashboard đọc nặng, ghi ít — tối ưu kiến trúc thế nào?

**Tình huống:** 10k QPS đọc feed, 100 QPS post bài mới. PostgreSQL `SELECT` feed chậm dù đã index.

**Follow-up:**
- Cache Redis toàn bộ feed — invalidate khi có post mới?
- Materialized view / bảng `feed_items` denormalized?
- Khi nào mới cần tách read DB (CQRS)?

**Gợi ý đáp án:**
- **Bước 1 (thường đủ)**: Redis cache feed per user, TTL 30s–2 phút + invalidate on write (hoặc chấp nhận stale ngắn)
- **Bước 2**: Bảng `user_feed` pre-computed — worker fan-out khi có post mới (write path chậm hơn, read path O(1))
- **Bước 3**: Read replica cho query nặng + route read traffic
- CQRS khi: read/write model khác hẳn, team scale, event sourcing có sẵn — **không** nhảy thẳng CQRS cho CRUD đơn giản
- Fan-out on write (Twitter) vs fan-out on read (Facebook) — chọn theo tỷ lệ follower

| Level | Signal |
| ----- | ------ |
| **Middle** | Cache + index, biết read >> write |
| **Senior** | Fan-out strategy, pre-computed feed, incremental CQRS, metric trước/sau |
| **🚩** | CQRS + Kafka cho 100 user; cache vĩnh viễn không invalidate |

📖 Tham chiếu: [01-system-design.md](./basic/01-system-design.md) | [5-system-design-scenarios.md](./nodejs/11-interview-prep/5-system-design-scenarios.md)

---

### Topic 6: DevOps & Observability

#### Q6.1: Production incident — bạn debug latency spike 5xx thế nào?

**Follow-up:**
- Metric nào xem đầu tiên?
- Rollback vs hotfix decision?
- Post-mortem viết gì?

**Gợi ý đáp án:**
- Triage: error rate, latency p99, recent deploy, dependency health
- Logs: correlation ID trace request path
- APM: slow query, external API timeout
- Rollback nếu deploy gần đây; hotfix nếu data corruption
- Post-mortem: timeline, root cause, action items — blameless

| Level | Signal |
| ----- | ------ |
| **Middle** | Check logs, recent deploy, restart service |
| **Senior** | Structured triage, metrics/dashboards, blameless post-mortem story |
| **🚩** | "Chưa từng on-call" cho senior role |

📖 Tham chiếu: [04-infrastructure-devops.md](./basic/04-infrastructure-devops.md)

---

#### Q6.2: Health check — liveness vs readiness probe

**Gợi ý đáp án:**
- **Liveness**: process còn sống — fail → restart pod
- **Readiness**: sẵn sàng nhận traffic — fail → remove khỏi load balancer
- Readiness check DB connection; liveness chỉ check app không deadlock

| Level | Signal |
| ----- | ------ |
| **Middle** | Biết khái niệm |
| **Senior** | Startup probe, graceful shutdown, dependency trong readiness |
| **🚩** | Liveness check DB → restart loop khi DB down |

---

### Topic 7: Concurrency & CS Fundamentals

#### Q7.1: Concurrency vs Parallelism — khác nhau thế nào?

**Follow-up:**
- Race condition — ví dụ và fix?
- Mutex vs semaphore?
- Thread-safe trong context backend bạn dùng?

**Gợi ý đáp án:**
- Concurrency: nhiều task tiến triển (có thể interleave trên 1 core)
- Parallelism: thực sự chạy đồng thời trên nhiều core
- Race condition: shared mutable state không sync → lock, atomic operation, immutable data
- Backend: DB transaction isolation, distributed lock (Redis Redlock — có controversy)

| Level | Signal |
| ----- | ------ |
| **Middle** | Phân biệt được 2 khái niệm |
| **Senior** | Áp dụng vào stack cụ thể, distributed lock trade-off |
| **🚩** | Nhầm lẫn hoàn toàn |

📖 Tham chiếu: [09-computer-science-fundamentals/03-concurrency.md](./basic/09-computer-science-fundamentals/03-concurrency.md)

---

## 5. System Design & Live Coding

### System Design Scenarios (chọn 1 theo level)

| Scenario | Middle focus | Senior focus |
| -------- | ------------ | ------------ |
| URL Shortener | CRUD, hash, redirect 301 | Collision handling, analytics, rate limit |
| Notification System | Queue + worker | Multi-channel, dedup, priority, DLQ |
| E-commerce Order | Order state machine | Inventory reservation, payment saga |
| Rate Limiter API | Token bucket cơ bản | Distributed rate limit, sliding window |

**Cách chấm System Design:**

```
□ Clarify requirements (functional + non-functional)
□ Estimation (QPS, storage) — senior bắt buộc
□ High-level diagram hợp lý
□ Deep dive 1–2 components
□ Failure modes & monitoring
□ Trade-offs được nêu rõ
```

### Live Coding (Middle)

**Đề gợi ý:** Implement `POST /users` với validation email, hash password, return 201.

**Chấm:**
- Input validation
- Password không log/plaintext
- Error handling (duplicate email → 409)
- Structure (handler → service → repository)

**Không chấm:** syntax hoàn hảo, biến naming fancy.

### Live Coding (Senior)

**Đề gợi ý:** Fix endpoint chậm — code có N+1 và thiếu index. Hoặc: implement idempotent webhook handler.

**Chấm:**
- Identify root cause nhanh
- Fix đúng layer
- Đề xuất test + monitoring

---

## 6. Behavioral & Leadership (Senior)

Dùng STAR (Situation → Task → Action → Result). Hỏi 2–3 câu:

| Câu hỏi | Đánh giá |
| ------- | -------- |
| Kể incident production nghiêm trọng nhất bạn xử lý? | Ownership, calm under pressure, learning |
| Khi junior code không đạt — bạn review/mentor thế nào? | Feedback constructive, không blame |
| Quyết định technical bị challenge — ví dụ? | Data-driven, communicate trade-off |
| Deadline gấp vs quality — bạn cân bằng? | Pragmatism, không cut corner vô tội vạ |

**Red flag senior:** Mọi câu trả lời "tôi làm hết một mình", đổ lỗi team/PM.

📖 Tham chiếu: [06-leadership-soft-skills.md](./basic/06-leadership-soft-skills.md)

---

## 7. Scorecard & Quyết Định Tuyển Dụng

### Scorecard mẫu (1–4 scale)

| Tiêu chí | 1 Strong No | 2 No | 3 Yes | 4 Strong Yes |
| -------- | ----------- | ---- | ----- | ------------ |
| Technical depth | Sai căn bản | Nông | Đủ level | Vượt level |
| Problem solving | Không có hướng | Cần gợi ý nhiều | Tự giải quyết | Elegant solution |
| Production exp | Không có | Lý thuyết | Có ví dụ | Deep war stories |
| Communication | Khó hiểu | OK | Rõ ràng | Xuất sắc |
| Culture | Red flags | Neutral | Good fit | Strong fit |

### Quy tắc quyết định

- **Hire:** Đa số ≥ 3, không có tiêu chí critical = 1
- **No Hire:** Bất kỳ critical skill = 1 (vd: senior mà system design = 1)
- **Borderline:** Debrief panel — không hire by default; cần ít nhất 1 Strong Yes advocate

### Calibration tips

- Ghi evidence cụ thể: *"Giải thích được cache stampede và đề xuất mutex"* — không ghi *"OK"*
- So sánh với bar đã thống nhất trước khi phỏng vấn
- Tránh halo effect (ứng viên friendly → đánh giá technical cao)

---

## 8. Liên Kết

| Tài liệu | Mô tả |
| -------- | ----- |
| [Node.js Interviewer Guide](./nodejs/11-interview-prep/INTERVIEWER_GUIDE.md) | Câu hỏi & rubric chuyên Node.js |
| [Node.js Candidate Prep](./nodejs/11-interview-prep/INTERVIEW_GUIDE.md) | Đáp án chi tiết tham chiếu khi chấm |
| [Code Review Guide](./basic/08-review-code.md) | Tiêu chí đánh giá code trong live coding |
| [System Design](./basic/01-system-design.md) | Kiến thức nền cho system design round |

---

**Cập nhật:** 2026-06-30 | **Đối tượng:** Interviewer | **Level:** Middle → Senior
