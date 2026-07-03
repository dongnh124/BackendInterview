### Câu 1: LST và MCMA đều dùng Express. So sánh Express với Fastify và khi nào bạn cân nhắc đổi hoặc dùng song song?

**Gợi ý trả lời:**
- **Express:** Ecosystem lớn nhất, middleware phong phú, team LST/MCMA đã quen; performance trung bình, callback-style cũ (có thể wrap async).
- **Fastify:** Performance cao (schema-based validation, JSON serialize nhanh), plugin architecture; learning curve và ecosystem nhỏ hơn Express.
- Giữ Express khi: team velocity quan trọng, deadline gần (flash sale LST), nhiều middleware/ORM Sequelize đã tích hợp sẵn.
- Cân nhắc Fastify khi: module mới cần throughput cao (MCMA notification fan-out), greenfield service tách khỏi monolith.
- Tech Lead: chuẩn hóa project template (folder structure, error handler, logging) bất kể framework; tránh mix framework trong cùng codebase không có lý do.

**Câu hỏi mở rộng:**
- Bạn có benchmark Express vs Fastify trên workload thực tế LST/MCMA không? Kết quả ra sao?
- Middleware chain Express có trở thành bottleneck khi traffic cao không?
- Dev trong team đề xuất đổi framework giữa chừng dự án — quy trình ra quyết định của Tech Lead?

**Cách đánh giá:**
- **Không đạt:** Chỉ nói "Express phổ biến nhất" hoặc đề xuất đổi framework mà không có căn cứ.
- **Đạt:** So sánh được 2–3 điểm khác biệt Express vs Fastify.
- **Tốt:** Liên hệ LST/MCMA đang dùng Express; nêu trade-off migration cost và team skill.
- **Xuất sắc:** Đề xuất chiến lược tách service mới (Fastify) giữ nguyên core (Express); shared packages cho types/validation.

---

### Câu 2: LST (React Native) và MCMA (Next.js, Electron) không thể deploy đồng thời với backend — bạn quản lý API versioning và backward compatibility như thế nào?

**Gợi ý trả lời:**
- **Versioning strategy:** URL prefix (`/v1`, `/v2`) hoặc header `Accept-Version` — chọn một convention và enforce xuyên suốt team; GraphQL dùng schema evolution + deprecate field có `@deprecated`.
- **LST — mobile constraint:** User có thể dùng app cũ sau khi backend đã deploy; breaking change (đổi field name, bỏ endpoint) phải có grace period tối thiểu 1–2 release cycle.
- **MCMA — multi-client:** Electron auto-update khác web deploy; backend phải support ít nhất 2 version client song song khi có breaking change.
- **Backward compatible changes:** Thêm field optional, không xóa/đổi type field cũ; expand enum thay vì rename; default value cho field mới.
- **Breaking change process:** Announce timeline → ship adapter layer (v1 controller delegate sang service mới) → monitor traffic v1 → sunset khi usage < threshold.
- **Contract test:** Pact hoặc snapshot test API response; CI fail nếu breaking change không được mark major version.
- **Feature flag:** Bật logic mới trên backend trước khi mobile release feature tương ứng — decouple deploy backend và client.
- Tech Lead: maintain changelog API trên Confluence; sync với mobile/desktop team trước mỗi sprint có API change.

**Câu hỏi mở rộng:**
- Versioning GraphQL schema khác REST thế nào khi deprecate field?
- Minimum supported app version — enforce ở backend (block) hay chỉ warning phía client?
- Sunset `/v1` khi vẫn còn ~5% traffic từ app cũ — bạn xử lý thế nào?

**Cách đánh giá:**
- **Không đạt:** "Deploy backend và app cùng lúc là được" — không hiểu App Store review delay hoặc user không update app.
- **Đạt:** Biết cần versioning và không breaking change tùy tiện.
- **Tốt:** Mô tả strategy cụ thể (URL v1/v2 hoặc header); ví dụ compatible vs breaking change; feature flag decouple deploy.
- **Xuất sắc:** Kể case thực tế LST/MCMA đã xử lý version mismatch; contract test trong CI; sunset policy và monitor old version traffic.

---

### Câu 3: GraphQL (dùng tại LST và MCMA) — khi nào chọn GraphQL thay vì REST? Nhược điểm cần lưu ý?

**Gợi ý trả lời:**
- **Ưu điểm:** Client (React Native) fetch đúng field cần; giảm over-fetching; single endpoint.
- **Nhược điểm:** Caching phức tạp hơn REST (CDN); N+1 query nếu không DataLoader; complexity attack (deep nested query).
- Dùng khi: mobile app nhiều màn hình khác nhau, team frontend autonomous.
- Mitigation: query depth limit, complexity analysis, persisted queries, Redis cache cho hot queries.
- Không thay thế toàn bộ REST — webhook, file upload, internal service có thể vẫn REST.

**Câu hỏi mở rộng:**
- GraphQL subscriptions có dùng cho realtime MCMA thay/bổ sung WebSocket không?
- Upload file (ảnh chat, avatar) xử lý trong GraphQL hay tách REST endpoint?
- Endpoint REST nào tại LST bạn giữ cố ý không chuyển sang GraphQL — vì sao?

**Cách đánh giá:**
- **Không đạt:** "GraphQL hiện đại hơn REST".
- **Đạt:** Biết over-fetching problem và DataLoader.
- **Tốt:** Ví dụ từ LST mobile + backend; nêu security concern.
- **Xuất sắc:** Thảo luận federation vs monolith schema; monitoring GraphQL performance.

---

### Câu 4: TypeScript Senior — bạn tổ chức codebase Node.js TypeScript cho team 6–8 người như thế nào để tránh "any everywhere"?

**Gợi ý trả lời:**
- `strict: true` trong tsconfig; bật `noImplicitAny`, `strictNullChecks`.
- Shared types package hoặc `types/` folder; DTO validation với Zod/class-validator.
- ESLint rule `@typescript-eslint/no-explicit-any`.
- Generic cho repository/service; discriminated union cho error handling.
- CI gate: `tsc --noEmit` + lint trước merge.
- Code review focus type design cho public API/interface.

**Câu hỏi mở rộng:**
- Chiến lược migrate legacy JS sang `strict` TypeScript trong codebase LST/MCMA?
- Zod vs class-validator — bạn chọn theo tiêu chí gì?
- Shared types giữa backend Node.js và React Native (LST) được tổ chức thế nào?

**Cách đánh giá:**
- **Không đạt:** "Dùng TypeScript là được" — không có governance.
- **Đạt:** Biết strict mode và validation library.
- **Tốt:** Có convention folder, ví dụ type-safe error handling hoặc API contract.
- **Xuất sắc:** Mô tả cách onboard junior vào TypeScript; balance strictness vs delivery speed.

---

### Câu 5: Tại LST và MCMA (Express + TypeScript), bạn thiết kế error handling và API response contract thống nhất cho team như thế nào?

**Gợi ý trả lời:**
- Centralized error middleware cuối Express chain; phân loại error: `AppError` (4xx có business meaning) vs unexpected (5xx).
- Response envelope thống nhất: `{ success, data, error: { code, message, details } }` — mobile (LST React Native) và web (MCMA Next.js/Electron) parse cùng format.
- Map Sequelize/DB error sang HTTP status có ý nghĩa (unique violation → 409, foreign key → 400).
- Structured logging (requestId, userId, route) trước khi trả response; không leak stack trace ra client production.
- Document error code catalog trên Confluence; dùng trong code review để tránh mỗi dev tự định nghĩa format.

**Câu hỏi mở rộng:**
- Uncaught promise rejection trong Express async handler — xử lý tập trung thế nào?
- Client LST hiển thị error 4xx (business) khác 5xx (system) ra sao trên React Native?
- Log error có chứa PII (email, phone) — policy redaction của team?

**Cách đánh giá:**
- **Không đạt:** `try/catch` rải rác, response format khác nhau từng endpoint.
- **Đạt:** Biết error middleware và phân biệt operational vs programmer error.
- **Tốt:** Mô tả envelope + logging correlation; ví dụ xử lý Sequelize error tại LST hoặc MCMA.
- **Xuất sắc:** Thảo luận i18n error message (LST global market); integration với Sentry/APM; contract test giữa backend và mobile team.

---

### Câu 6: Sequelize trong dự án LST — ưu/nhược điểm so với Prisma/TypeORM? Khi nào nên migrate ORM?

**Gợi ý trả lời:**
- **Sequelize:** Mature, hỗ trợ nhiều DB, migration tool; typing yếu hơn Prisma, API verbose.
- **Prisma:** Type-safe query, schema-first, DX tốt; ít flexible cho raw SQL phức tạp.
- Giữ Sequelize khi: team đã expert, migration cost cao, nhiều raw query/trigger integration.
- Migrate khi: greenfield module, typing bug nhiều, cần developer velocity.
- Tech Lead: không migrate vì hype — cần POC, cost estimate, incremental migration strategy.

**Câu hỏi mở rộng:**
- MCMA cũng dùng Sequelize — có khác convention hoặc config so với LST không?
- Module greenfield mới trong LST có nên dùng Prisma song song Sequelize không?
- N+1 query phát hiện trên production — quy trình fix và prevent tái diễn?

**Cách đánh giá:**
- **Không đạt:** Không biết ORM đang dùng hoặc không phân biệt được các ORM.
- **Đạt:** Nêu được pro/con cơ bản.
- **Tốt:** Quyết định pragmatic dựa trên context dự án; biết Sequelize limitations (N+1, transaction).
- **Xuất sắc:** Kể experience optimize Sequelize (eager loading, connection pool tuning).

---

### Câu 7: MCMA hỗ trợ Next.js, Electron và nhiều client — authentication và session management trên Node.js backend thiết kế thế nào?

**Gợi ý trả lời:**
- **Access token ngắn hạn (JWT)** + **refresh token** lưu httpOnly cookie (web) hoặc secure storage (Electron/mobile).
- Electron: PKCE hoặc device-specific token; tránh embed long-lived secret trong desktop app.
- Session revocation: Redis blacklist jti/exp; force logout all devices khi đổi password.
- WebSocket auth: validate token lúc handshake, re-auth khi token refresh — không gửi credential trên mỗi message.
- RBAC cho group admin, thread moderator; permission check ở service layer, không chỉ route level.
- Rate limit login và brute-force protection; audit log cho sensitive action.

**Câu hỏi mở rộng:**
- Refresh token reuse detection (token bị dùng lại sau rotate) — có implement không?
- E2E encryption message MCMA có ảnh hưởng flow login/session refresh không?
- Guest user hoặc anonymous preview trong MCMA — auth model khác registered user thế nào?

**Cách đánh giá:**
- **Không đạt:** "Dùng JWT lưu localStorage" — không phân biệt client type.
- **Đạt:** Biết access/refresh token flow cơ bản.
- **Tốt:** Xử lý khác biệt web vs Electron; WebSocket authentication tại MCMA.
- **Xuất sắc:** Token rotation, compromise recovery; E2E encryption tách biệt transport auth vs message encryption; scale session store trên Redis cluster.

---

### Câu 8: LST kết hợp Express monolith và AWS Lambda serverless — bạn phân chia ranh giới giữa sync API và async workload ra sao?

**Gợi ý trả lời:**
- **Express (ECS/EC2 hoặc long-running):** Request-response path cần latency ổn định — auth, catalog browse, cart, order submission nhận request.
- **Lambda:** Event-driven, burst workload — image resize, webhook xử lý payment/notification, scheduled job (flash sale pre-warm, report).
- Trigger qua SQS/SNS/EventBridge thay vì gọi Lambda sync từ request path trừ khi latency chấp nhận được.
- Shared contract: message schema versioned; idempotency key cho event consumer.
- Local dev: SAM/LocalStack hoặc docker-compose mock queue; tránh "chỉ test được trên AWS".
- Tech Lead document decision matrix: khi nào thêm Lambda vs giữ trong monolith (complexity vs cost vs cold start).

**Câu hỏi mở rộng:**
- Payment webhook tại LST nên chạy Lambda hay giữ trong Express — tiêu chí quyết định?
- Cold start Lambda có ảnh hưởng job pre-warm cache trước flash sale không?
- Local dev với Lambda + Express monolith — setup team LST dùng gì?

**Cách đánh giá:**
- **Không đạt:** "Mọi thứ đều Lambda" hoặc không giải thích được vì sao có hai mô hình song song.
- **Đạt:** Phân biệt sync API vs async job cơ bản.
- **Tốt:** Ví dụ workload cụ thể tại LST (flash sale notification, asset processing); nêu cold start concern.
- **Xuất sắc:** Cost/ops trade-off; observability xuyên suốt (trace từ API → queue → Lambda); failure handling khi Lambda retry.

---

### Câu 9: Tại MCMA, bạn phát triển AWS Lambda cho media processing workflow — mô tả pipeline từ upload đến client nhận được media?

**Gợi ý trả lời:**
- Client upload → pre-signed S3 URL (tránh proxy file qua API server) → S3 event trigger Lambda.
- Lambda pipeline: validate MIME/size → virus scan (optional) → transcode/thumbnail (sharp/ffmpeg layer) → ghi metadata vào PostgreSQL/MongoDB.
- BullMQ job cho bước chậm hoặc retry (transcode fail, quota exceeded); Kafka notify message service khi media ready.
- CDN (CloudFront) phía trước S3 cho delivery; signed URL hoặc short-lived token cho private chat media.
- Dead letter queue + alert khi job fail; idempotent processing (cùng object key không tạo duplicate).
- Giới hạn Lambda: memory/timeout cho video lớn — có thể offload sang ECS worker nếu vượt ngưỡng.

**Câu hỏi mở rộng:**
- Upload bị ngắt giữa chừng (mạng chập) — resume/chunk upload xử lý thế nào?
- File fail virus scan hoặc MIME invalid — client MCMA nhận feedback ra sao?
- Pipeline Mediasoup (voice/video call) khác pipeline attachment chat thế nào?

**Cách đánh giá:**
- **Không đạt:** Upload file trực tiếp qua Express body parser — không scale.
- **Đạt:** Biết pre-signed URL và S3 trigger Lambda.
- **Tốt:** Mô tả đủ bước validate → process → notify; BullMQ retry role.
- **Xuất sắc:** Cost optimization (Lambda vs ECS); Mediasoup voice/video tách pipeline khác chat attachment; monitoring processing latency p99.

---

### Câu 10: Tại LST và MCMA, bạn thiết kế observability (logging, metrics, alerting) cho Node.js backend như thế nào để phát hiện sớm sự cố trước khi user báo?

**Gợi ý trả lời:**
- **Structured logging:** JSON log với `requestId`, `userId`, `route`, `durationMs`; correlation ID xuyên suốt API → queue → Lambda (MCMA media, LST async job).
- **Metrics:** RED method — Request rate, Error rate, Duration (p50/p95/p99) theo endpoint; business metric riêng (LST: order success rate, flash sale checkout latency; MCMA: message delivery latency, WS connection count).
- **APM/tracing:** OpenTelemetry hoặc X-Ray/CloudWatch để trace slow query Sequelize, Redis timeout, external API call.
- **Alerting:** Threshold + anomaly (error spike, queue lag, DB connection pool > 80%); on-call runbook link trong alert message.
- **Health check:** `/health` (liveness) vs `/ready` (DB + Redis reachable); graceful shutdown — stop nhận request mới, drain in-flight trước deploy.
- Tech Lead: define SLI/SLO với team (vd. API p99 < 500ms); dashboard shared cho PO/ops; post-release watch window sau deploy.

**Câu hỏi mở rộng:**
- Log volume tăng 10x trong flash sale LST — chiến lược control cost retention?
- Alert fatigue — làm sao tránh on-call ignore alert thật?
- Sampling/trace khi traffic cao để không làm chậm request path?

**Cách đánh giá:**
- **Không đạt:** Chỉ `console.log` và xem log khi có bug; không có alert proactive.
- **Đạt:** Biết structured logging và basic CloudWatch/metrics.
- **Tốt:** Phân biệt liveness/readiness; ví dụ metric cụ thể tại LST (flash sale) hoặc MCMA (chat/WS); correlation ID across services.
- **Xuất sắc:** SLO-driven alerting; kinh nghiệm catch issue trước peak traffic; integrate với CI/CD (smoke test metric sau deploy); cost-aware log retention strategy.

---

### Câu 11: Tại LST (flash sale), nhiều user đặt hàng đồng thời trên cùng sản phẩm — thiết kế database và application layer để tránh oversell và race condition?

**Gợi ý trả lời:**
- **DB level:** `SELECT ... FOR UPDATE` trên inventory row hoặc optimistic locking (version column); transaction bọc check stock + tạo order.
- **Atomic decrement:** `UPDATE inventory SET qty = qty - 1 WHERE product_id = ? AND qty > 0 RETURNING qty` — chỉ 1 request thành công khi stock = 1.
- **Application:** Queue order request qua BullMQ/Kafka; xử lý tuần tự per product_id để giảm lock contention.
- **Idempotency:** Client order key hoặc idempotency header — tránh duplicate order khi mobile retry.
- **Redis pre-check:** Cache stock count cho read path; authoritative source vẫn là PostgreSQL khi commit order.
- **UX:** Trả 409/422 rõ ràng khi hết hàng; không để user hoàn tất payment khi inventory đã về 0.

**Câu hỏi mở rộng:**
- Payment gateway confirm thành công nhưng reserve stock thất bại — flow compensating transaction?
- Idempotency key lưu ở đâu (Redis/PostgreSQL) và TTL bao lâu?
- Load test flash sale LST — tool, scenario và metric pass/fail criteria?

**Cách đánh giá:**
- **Không đạt:** Chỉ nói "dùng transaction" hoặc chỉ cache Redis không có DB guarantee.
- **Đạt:** Biết pessimistic vs optimistic locking; atomic UPDATE pattern.
- **Tốt:** Mô tả flow checkout flash sale tại LST; kết hợp queue + DB.
- **Xuất sắc:** Hot product problem (1 SKU, 10k concurrent); load test approach; partial failure khi payment timeout sau khi reserve stock.

---

### Câu 12: LST và MCMA đều dùng cả PostgreSQL và MongoDB — tiêu chí chọn SQL vs NoSQL cho từng loại data?

**Gợi ý trả lời:**
- **PostgreSQL (LST):** Order, payment, inventory, user account — ACID, transaction, reporting.
- **MongoDB (LST):** Product catalog attribute linh hoạt (fashion variants), CMS content nếu schema thay đổi thường xuyên.
- **PostgreSQL (MCMA):** User profile, group membership, relational data cần join/constraint.
- **MongoDB (MCMA):** Chat messages, thread history, metadata embed — write-heavy, flexible schema.
- **Polyglot persistence:** Không duplicate cùng entity ở hai DB không sync; event-driven sync qua Kafka nếu cần.
- Tech Lead định nghĩa data ownership — module/service nào own DB nào.

**Câu hỏi mở rộng:**
- Có operation nào cần transaction span cả PostgreSQL và MongoDB không? Xử lý thế nào?
- Search tin nhắn MCMA (full-text) — index MongoDB hay sync sang Elasticsearch?
- Khi nào nên migrate entity từ Mongo sang PG hoặc ngược lại?

**Cách đánh giá:**
- **Không đạt:** "MongoDB nhanh hơn PostgreSQL" — stereotype không căn cứ.
- **Đạt:** Phân biệt ACID vs flexible schema.
- **Tốt:** Ví dụ cụ thể từ LST (order PG + catalog Mongo) và MCMA (chat Mongo + user/group PG).
- **Xuất sắc:** Thảo luận consistency pattern, anti-pattern dual-write; migration path khi schema evolve.

---

### Câu 13: Kafka vs BullMQ — bạn đã dùng cả hai. Khi nào chọn cái nào?

**Gợi ý trả lời:**
- **Kafka:** High throughput event streaming, multiple consumer groups, replay. VD tại LST: order placed event, inventory update; tại MCMA: message event, notification fan-out.
- **BullMQ:** Job queue trên Redis, retry with backoff, priority queue, delayed job. VD tại LST: email xác nhận đơn, report; tại MCMA: resize media, push notification retry.
- Kafka: operational overhead cao hơn (ZooKeeper/KRaft, partition tuning).
- BullMQ: phụ thuộc Redis memory; không phù hợp long-term event store.
- Pattern: Kafka cho domain events; BullMQ cho worker tasks triggered từ those events.

**Câu hỏi mở rộng:**
- Kafka consumer xử lý duplicate message (at-least-once) — dedup strategy tại LST/MCMA?
- BullMQ job stuck in active state — detect và recover thế nào?
- Redis down: BullMQ chết theo; Kafka vẫn chạy — ảnh hưởng thiết kế failover?

**Cách đánh giá:**
- **Không đạt:** Coi hai thứ là interchangeable.
- **Đạt:** Phân biệt queue vs stream cơ bản.
- **Tốt:** Ví dụ cụ thể từ LST (order event → Kafka) và MCMA (media/notification → BullMQ).
- **Xuất sắc:** Nêu consumer group strategy, partition key design, monitoring (lag, DLQ).

---

### Câu 14: Kafka và BullMQ tại LST/MCMA khi throughput tăng (flash sale, peak chat) — scale consumer và queue ra sao?

**Gợi ý trả lời:**

**Kafka:**
- **Partition key design:** Order event LST partition by `product_id` hoặc `order_id` — parallel consumer nhưng giữ ordering per key.
- **Consumer group:** Tăng consumer instance ≤ số partition; monitor consumer lag (alert khi lag > threshold).
- **Producer:** `acks=all`, `min.insync.replicas` cho durability; batch + compression giảm overhead.
- **Retention & replay:** Đủ retention để replay khi consumer bug; không dùng Kafka như job queue đơn giản.
- **MCMA:** Message event topic tách notification topic — scale consumer group độc lập.

**BullMQ:**
- **Worker concurrency:** Tăng worker process + `concurrency` per queue; tách queue theo priority (flash sale order > report email).
- **Redis memory:** Monitor memory; queue depth alert; `removeOnComplete` / `removeOnFail` policy tránh Redis đầy.
- **Rate limit worker:** Giới hạn gọi external API (email, push) tránh bị throttle phía thứ ba.
- **DLQ & retry:** Exponential backoff; max retry; dead letter queue + manual replay tool.

**Chung:**
- **Backpressure:** API stop enqueue hoặc trả 503 khi queue depth vượt ngưỡng — bảo vệ downstream.
- **Idempotent consumer:** Dedup bằng `messageId` / idempotency key trong PostgreSQL hoặc Redis SET.
- **LST flash sale:** Pre-warm consumer; load test end-to-end queue → DB, không chỉ API.

**Câu hỏi mở rộng:**
- Consumer lag tăng nhưng CPU thấp — nguyên nhân có thể là gì? (slow DB, external API, partition skew)
- Hot partition (1 product flash sale LST chiếm hết 1 Kafka partition) — giải pháp?
- BullMQ và Kafka dùng chung Redis cluster — rủi ro và tách infra ra sao?

**Cách đánh giá:**
- **Không đạt:** "Thêm consumer là xong" không nêu partition, lag, hay Redis memory.
- **Đạt:** Biết consumer group Kafka và tăng BullMQ worker cơ bản.
- **Tốt:** Partition key design; lag monitoring; backpressure; ví dụ LST order queue vs MCMA notification.
- **Xuất sắc:** Hot partition mitigation; end-to-end capacity test; DLQ replay runbook; cost/ops trade-off Kafka vs BullMQ khi scale.

### Câu 15: Redis caching layer — cache-aside, write-through, write-behind khác nhau thế nào? Áp dụng ra sao cho flash sale LST?

**Gợi ý trả lời:**
- **Cache-aside:** App đọc cache trước, miss thì đọc DB và populate cache. Phổ biến nhất, dễ implement.
- **Write-through:** Ghi đồng thời cache + DB — consistency tốt, write chậm hơn.
- **Write-behind:** Ghi cache trước, async flush DB — throughput cao, risk mất data nếu cache crash.
- Flash sale: pre-warm cache product/inventory trước event; short TTL; distributed lock (Redlock) cho inventory decrement.
- Cache stampede: mutex, request coalescing, probabilistic early expiration.

**Câu hỏi mở rộng:**
- Admin đổi giá/số lượng giữa phiên flash sale đang chạy — invalidate cache thế nào ngay lập tức?
- Redis Cluster vs Sentinel/replica — LST chọn gì và vì sao?
- Metric nào chứng minh cache stampede đã xảy ra và fix hiệu quả?

**Cách đánh giá:**
- **Không đạt:** Chỉ biết "cache để nhanh".
- **Đạt:** Giải thích cache-aside flow.
- **Tốt:** Flash sale scenario với inventory oversell prevention.
- **Xuất sắc:** Số liệu hit rate, eviction policy; Redis Cluster vs single instance decision.

---

