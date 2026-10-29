# Node.js Backend — Mock Interview

---

## Câu 1

**Interviewer:** Bạn cấu trúc codebase của dự án như nào? Cách tổ chức file theo feature base và theo file type base có ưu nhược điểm gì?

**Candidate:**

Ở các dự án Node.js backend mình thường làm, mình không chọn một kiểu tuyệt đối mà kết hợp theo hướng **hybrid** — phần lớn theo **feature-based** (tổ chức theo tính năng/nghiệp vụ), còn phần **cross-cutting** (dùng chung toàn hệ thống) thì tách ra theo **layer** hoặc **type** (loại file/chức năng).

### Cấu trúc mình hay dùng

```
src/
├── modules/                    # feature-based
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── users.routes.ts
│   │   ├── users.dto.ts
│   │   ├── users.types.ts
│   │   └── __tests__/
│   ├── orders/
│   └── payments/
├── shared/                     # cross-cutting concerns
│   ├── database/
│   ├── cache/
│   ├── middleware/
│   ├── utils/
│   └── errors/
├── config/
├── app.ts
└── server.ts
```

Nguyên tắc cốt lõi: **mọi thứ thuộc về một domain** (miền nghiệp vụ) thì nằm trong folder của domain đó. Chỉ những thứ thực sự dùng chung — như database connection, logger, error handler — mới đưa ra `shared/`.

### File type base (tổ chức theo loại file)

Đây là kiểu kiểu cũ, hay gặp ở MVC (Model-View-Controller) monolith:

```
src/
├── controllers/
│   ├── users.controller.ts
│   └── orders.controller.ts
├── services/
├── repositories/
├── routes/
└── models/
```

**Ưu điểm:**

- Dễ hiểu với người mới — mở project biết ngay file loại gì nằm đâu.
- Phù hợp dự án nhỏ, ít feature, team nhỏ (2–3 người).
- Onboarding (đưa người mới vào team) nhanh vì convention (quy ước) rõ ràng.

**Nhược điểm:**

- Khi scale (mở rộng), mỗi feature bị **scatter** (phân tán) across nhiều folder — sửa một tính năng phải nhảy qua 4–5 thư mục.
- Khó enforce **module boundary** (ranh giới module) — dev dễ import chéo lung tung giữa các domain.
- Khó tách thành **microservice** (dịch vụ nhỏ độc lập) sau này vì logic một feature không gom một chỗ.
- Ownership (quyền sở hữu code) mờ — ai cũng sửa chung folder `services/`, dễ conflict (xung đột) khi nhiều người làm song song.
- Test cũng bị tách rời — khó viết integration test (test tích hợp) theo từng feature.



### Feature base (tổ chức theo tính năng)

Mỗi folder là một **bounded context** (ngữ cảnh giới hạn) hoặc ít nhất là một business capability (khả năng nghiệp vụ):

```
src/modules/orders/
├── orders.controller.ts
├── orders.service.ts
├── orders.repository.ts
├── orders.events.ts        # domain events (sự kiện nghiệp vụ)
├── orders.routes.ts
└── __tests__/
```

**Ưu điểm:**

- **High cohesion** (gắn kết cao) — mọi thứ liên quan đến `orders` nằm cùng chỗ, dễ đọc, dễ sửa, dễ review.
- **Low coupling** (phụ thuộc thấp) giữa các module nếu enforce đúng — module A không import trực tiếp internal của module B.
- Scale team tốt — mỗi squad (nhóm) own một vài module, ít đụng nhau.
- Dễ extract (tách) thành microservice: copy folder `orders/` ra là gần xong service mới.
- Phù hợp **Domain-Driven Design** (thiết kế hướng domain) — folder structure reflect (phản ánh) business domain.

**Nhược điểm:**

- Dễ bị **duplicate code** (trùng lặp code) nếu không có `shared/` rõ ràng — mỗi feature tự viết lại validation, pagination...
- Cần discipline (kỷ luật) cao hơn — phải define rule import (ví dụ: chỉ export qua `index.ts` hoặc public API của module).
- Với feature rất nhỏ (CRUD đơn giản), có thể overkill (làm quá mức cần thiết) — tạo nhiều folder cho ít code.
- Khó hơn cho junior mới vào nếu chưa quen pattern này.



### Cách mình quyết định chọn gì

Mình dùng bảng này:


| Tiêu chí                  | File type base          | Feature base                 |
| ------------------------- | ----------------------- | ---------------------------- |
| Team size                 | 1–3 dev                 | 4+ dev hoặc nhiều squad      |
| Số feature                | < 10 endpoints đơn giản | Nhiều domain phức tạp        |
| Lifespan (tuổi thọ dự án) | MVP, prototype          | Long-term production         |
| Mục tiêu                  | Ship nhanh              | Maintainability (dễ bảo trì) |


Thực tế ở production, mình hay dùng **feature-based + layered bên trong mỗi feature**:

```
modules/orders/
├── application/     # use cases, orchestration (điều phối logic)
├── domain/          # entities, business rules (thực thể, quy tắc nghiệp vụ)
├── infrastructure/  # DB, external API adapters (kết nối hạ tầng)
└── presentation/    # controller, DTO, routes (lớp giao tiếp HTTP)
```

Đây gần với **Clean Architecture** (kiến trúc sạch) hoặc **Hexagonal** (kiến trúc lục giác) — domain ở giữa, infrastructure (hạ tầng) ở ngoài. Lợi là test domain logic không cần database, mock (giả lập) repository dễ dàng.

### Những rule mình enforce trong team

1. **Module chỉ expose public API** qua `index.ts` — internal file không được import từ module khác.
2. **Không import chéo giữa modules** — nếu `orders` cần data từ `users`, gọi qua service interface hoặc domain event, không import thẳng `users.repository.ts`.
3. **Shared code phải thực sự generic** (dùng chung thật) — nếu chỉ 2 module dùng, cân nhắc để duplicate nhỏ thay vì tạo abstraction (trừu tượng hóa) sớm.
4. Dùng **ESLint boundary rules** hoặc **dependency-cruiser** để enforce (bắt buộc) import graph (đồ thị phụ thuộc) — không chỉ dựa vào convention miệng.



### Ví dụ thực tế

Ở dự án e-commerce trước, team 8 người, ban đầu dùng file type base. Sau 6 tháng, mỗi lần hotfix (sửa gấp production) phải touch 5–6 file ở 4 folder khác nhau, PR (pull request) conflict liên tục. Migrate (chuyển đổi) sang feature-based theo từng module — bắt đầu với `payments` vì isolated (tách biệt) nhất — trong 2 sprint. Kết quả: thời gian review PR giảm, onboarding dev mới vào module cụ thể nhanh hơn vì chỉ cần hiểu 1 folder thay vả cả codebase.

Tóm lại: **file type base cho dự án nhỏ/ngắn hạn, feature base cho dự án production cần scale team và maintain lâu dài**. Cách expert không phải chọn một phe mà biết khi nào dùng gì, và hybrid khi cần.

---

## Câu 2

**Interviewer:** SOLID là gì? Bạn thường dùng design pattern nào trong quá trình code?

**Candidate:**

**SOLID** là bộ 5 nguyên tắc thiết kế phần mềm, gốc từ OOP (lập trình hướng đối tượng), nhưng trong Node.js backend mình vẫn áp dụng rất nhiều — không phải vì "phải OOP", mà vì nó giải quyết đúng các pain point (điểm đau) hay gặp khi dự án lớn dần: code khó sửa, khó test, một thay đổi nhỏ kéo theo sửa cả đống chỗ khác.

**S — Single Responsibility (trách nhiệm đơn):** Mỗi module chỉ nên có một lý do để thay đổi. Ví dụ service xử lý order thì chỉ lo business logic (logic nghiệp vụ) — không gửi email, không format HTTP response, không viết query DB trực tiếp. Trong thực tế Node.js, mình hay thấy SRP thể hiện ở mức file/module: controller lo HTTP, service lo logic, repository lo persistence (lưu trữ dữ liệu).

**O — Open/Closed (mở để mở rộng, đóng để sửa đổi):** Thêm behavior (hành vi) mới mà không phải sửa code cũ. Case hay gặp nhất là payment — ban đầu chỉ có Stripe, sau thêm VNPay, MoMo. Nếu thiết kế đúng thì chỉ cần thêm implementation (triển khai) mới, không đụng vào service đang chạy ổn. Cái này rất quan trọng ở production vì sửa code đang stable (ổn định) luôn có risk (rủi ro) regression (lỗi quay lại).

**L — Liskov Substitution (thay thế được mà không phá behavior):** Nếu A thay được B thì dùng A không được làm hỏng logic bên gọi. Ví dụ điển hình khi vi phạm: có repository read-only nhưng vẫn kế thừa repository đầy đủ, gọi delete thì throw error — caller (bên gọi) không biết thì sẽ bug. Mình thường enforce (bắt buộc) LSP qua interface contract (hợp đồng interface): mọi implementation của cache provider phải có cùng semantics (ngữ nghĩa), ví dụ get trả null khi miss chứ không throw.

**I — Interface Segregation (tách interface nhỏ):** Đừng bắt client depend (phụ thuộc) vào thứ nó không dùng. Auth service chỉ cần đọc user thì inject reader thôi, không cần biết bulk import hay generate report tồn tại. Interface càng gọn thì coupling (gắn chặt) càng thấp.

**D — Dependency Inversion (phụ thuộc abstraction):** Business logic không nên phụ thuộc trực tiếp vào Postgres, Redis hay Stripe — mà phụ thuộc vào abstraction (lớp trừu tượng), rồi inject implementation (triển khai cụ thể) từ ngoài vào. NestJS có DI container (bộ chứa dependency injection) sẵn; Express thuần thì mình tự wire (nối) ở chỗ khởi tạo app. Lợi lớn nhất là test và đổi hạ tầng dễ hơn nhiều.

---

Về **design pattern**, mình không áp pattern cho có — mỗi pattern giải một bài toán cụ thể. Những cái mình dùng nhiều nhất:

**Repository** — gần như mặc định mọi dự án. Tách business logic khỏi ORM/query cụ thể. Service chỉ biết "lấy order theo id", "lưu order" — không quan tâm đằng sau là Prisma, TypeORM hay raw SQL. Lợi lớn nhất là test service bằng mock (giả lập) in-memory, không cần spin up database.

**Strategy** — khi có nhiều cách làm cùng một việc và cần swap (hoán đổi) runtime. Payment gateway, rule tính giá, kênh gửi notification, storage S3 hay local — đều là case điển hình.

**Factory** — khi logic tạo object phức tạp, cần gom một chỗ. Ví dụ user chọn gửi qua email, SMS hay push thì factory quyết định tạo notifier nào. Thường kết hợp với DI để service không biết concrete class (lớp cụ thể).

**Middleware** — đặc thù Node.js. Request đi qua chuỗi auth → rate limit → validate → handler. Mỗi middleware một concern (mối quan tâm), dễ tách, dễ test riêng.

**Observer / Event** — decouple (tách rời) side effect (tác dụng phụ). Tạo order xong emit event, listener khác lo gửi email, trừ kho, đẩy analytics. Service tạo order không cần biết những việc đó. Ở production scale lớn thì chuyển sang message queue (hàng đợi tin nhắn) như RabbitMQ hay SQS thay vì EventEmitter trong process — vì cần reliability (độ tin cậy) và retry (thử lại).

**Adapter** — wrap third-party API (API bên thứ ba). Stripe, VNPay, SendGrid mỗi bên một format khác nhau — adapter translate (chuyển đổi) về domain model (mô hình nghiệp vụ) nội bộ. Domain layer không nên biết Stripe tính tiền bằng cents hay partner API trả field lạ.

**Decorator** — trong NestJS/TypeScript hay gặp qua annotation như Injectable, Cacheable, Retry. Gắn thêm behavior mà không sửa method gốc.

---

Còn pattern mình **cố ý không dùng sớm**: Singleton — Node.js module system đã tự singleton (một instance) rồi, tạo thêm class Singleton thường gây global state (trạng thái toàn cục) khó test. CQRS/Event Sourcing thì powerful (mạnh) nhưng overkill cho CRUD app — chỉ khi thực sự cần audit trail (dấu vết kiểm toán) hoặc replay event (phát lại sự kiện).

Cách mình chọn pattern: không hỏi "dùng pattern gì" mà hỏi coupling (gắn chặt) đang ở đâu, thứ gì hay thay đổi nhất, test cần mock gì. Cần swap algorithm → Strategy; cần tách data access → Repository; cần tách side effect → Event. Pattern tự nhiên nổi lên từ problem, không phải ngược lại.

Tóm lại: **SOLID là guideline (hướng dẫn) để code sống sót khi team và requirement grow (phát triển); design pattern là tool giải bài toán cụ thể** — không phải checklist phải nhồi hết vào mọi project.

---

## Câu 3

**Interviewer:** Restful API là gì? Bạn thường dùng những status code nào?

**Candidate:**

**RESTful API** là cách thiết kế API theo kiến trúc **REST** (Representational State Transfer — chuyển giao trạng thái biểu diễn). Nói đơn giản: client và server giao tiếp qua **HTTP** bằng cách thao tác lên **resource** (tài nguyên) — user, order, product — thông qua **URL** và **HTTP method** (phương thức HTTP), thay vì gọi function kiểu RPC (Remote Procedure Call — gọi thủ tục từ xa) như "createUser", "deleteOrder".

Một API được coi là RESTful khi tuân theo vài nguyên tắc cốt lõi:

**Resource-oriented (hướng tài nguyên):** Mọi thứ là danh từ, không phải động từ. Đúng là `GET /users/123`, sai là `POST /getUserById`. URL đại diện cho resource, HTTP method đại diện cho hành động.

**HTTP method đúng semantics (ngữ nghĩa):** GET để đọc, POST để tạo, PUT/PATCH để cập nhật, DELETE để xóa. GET và DELETE phải **idempotent** (lặp lại nhiều lần cho cùng kết quả), GET còn phải **safe** (không thay đổi trạng thái server).

**Stateless (không trạng thái):** Mỗi request phải chứa đủ thông tin để server xử lý — thường qua token (JWT), API key. Server không "nhớ" session (phiên) giữa các request như kiểu traditional web app.

**Representation (biểu diễn):** Resource có thể trả về nhiều format — JSON, XML — thường dùng `Content-Type` và `Accept` header để negotiate (thương lượng). Thực tế backend Node.js gần như luôn JSON.

**HATEOAS** (Hypermedia as the Engine of Application State) — về lý thuyết response nên có link dẫn đến action tiếp theo. Thực tế ít team làm đủ mức này, nhưng mình vẫn hay trả pagination link (`next`, `prev`) trong list endpoint.

---

Về **status code**, mình coi đây là contract (hợp đồng) giữa backend và client — client không cần parse (phân tích) body để biết thành công hay thất bại. Mình hay dùng nhóm sau:

**2xx — Thành công:**

- **200 OK** — request thành công, có body trả về. Dùng cho GET, PUT, PATCH khi trả data.
- **201 Created** — tạo resource mới thành công. Hay kèm `Location` header trỏ đến resource vừa tạo, ví dụ `Location: /users/456`.
- **204 No Content** — thành công nhưng không có body. Thường dùng cho DELETE hoặc PUT/PATCH khi không cần trả data lại.

**3xx — Chuyển hướng** (ít dùng trong API thuần, hay gặp hơn ở gateway/proxy):

- **301/302** — redirect (chuyển hướng) vĩnh viễn/tạm thời.
- **304 Not Modified** — dùng với caching (bộ nhớ đệm), client gửi `If-None-Match` hoặc `If-Modified-Since`, server trả 304 nếu data chưa đổi — tiết kiệm bandwidth (băng thông).

**4xx — Lỗi phía client:**

- **400 Bad Request** — request sai format, validation fail (xác thực dữ liệu thất bại). Ví dụ thiếu field bắt buộc, email không đúng format. Body nên có error detail rõ ràng để client hiển thị.
- **401 Unauthorized** — chưa authenticate (xác thực). Chưa gửi token, token hết hạn, token sai. Client nên redirect (chuyển hướng) login.
- **403 Forbidden** — đã authenticate nhưng không có permission (quyền). Ví dụ user thường cố gọi admin endpoint. Khác 401 ở chỗ identity (danh tính) đã biết, chỉ thiếu quyền.
- **404 Not Found** — resource không tồn tại. `GET /users/999` mà user 999 không có. Lưu ý: đừng dùng 404 để che giấu 403 — nếu user không có quyền xem resource, nên cân nhắc trả 403 thay vì 404 tùy security policy (chính sách bảo mật).
- **409 Conflict** — xung đột trạng thái. Hay dùng khi duplicate (trùng lặp) — email đã tồn tại, optimistic locking (khóa lạc quan) version mismatch.
- **422 Unprocessable Entity** — request đúng format nhưng business rule (quy tắc nghiệp vụ) không cho phép. Ví dụ đặt hàng nhưng hết hàng, số dư không đủ. Khác 400 ở chỗ syntax (cú pháp) đúng nhưng semantics (ngữ nghĩa nghiệp vụ) sai.
- **429 Too Many Requests** — rate limit (giới hạn tần suất) bị vượt. Nên kèm `Retry-After` header.

**5xx — Lỗi phía server:**

- **500 Internal Server Error** — lỗi không lường trước, bug, DB connection fail. Không bao giờ expose (lộ) stack trace ra client ở production.
- **502 Bad Gateway** — server đứng giữa (reverse proxy, API gateway) nhận response lỗi từ upstream (dịch vụ phía sau).
- **503 Service Unavailable** — server tạm thời không phục vụ được — maintenance (bảo trì), overload (quá tải). Nên kèm `Retry-After`.
- **504 Gateway Timeout** — upstream không phản hồi kịp thời gian.

---

Một vài **nguyên tắc thực tế** mình hay enforce trong team:

Thứ nhất, **nhất quán** — toàn bộ API dùng cùng convention. Lỗi validation luôn 400 hay 422 thì chọn một cái và stick (giữ nguyên), đừng endpoint nào một kiểu.

Thứ hai, **error response có structure (cấu trúc) thống nhất** — ví dụ luôn có `code`, `message`, `details`. Client parse một lần, xử lý mọi endpoint.

Thứ ba, **đừng lạm dụng 200 cho mọi thứ** — kiểu trả 200 kèm `{ success: false }` trong body. Đây là anti-pattern (phản mẫu) phổ biến, làm client phải check cả status lẫn body, mất đi lợi ích của HTTP status code.

Thứ tư, **phân biệt rõ 401 vs 403** — nhiều dev nhầm hai cái này. Rule đơn giản: chưa biết bạn là ai → 401; biết rồi nhưng không cho → 403.

Thứ năm, với **partial update** mình prefer (ưu tiên) PATCH hơn PUT — PUT thay thế toàn bộ resource, PATCH chỉ sửa field cần thiết. Trả 200 kèm resource đã update, hoặc 204 nếu client không cần body.

Tóm lại: **RESTful API là cách thiết kế API tận dụng HTTP đúng bản chất — resource, method, status code, stateless**; còn status code là ngôn ngữ giao tiếp chuẩn giữa server và client, dùng đúng thì client integrate (tích hợp) dễ hơn rất nhiều mà không cần đọc document chi tiết từng endpoint.

---

## Câu 4

**Interviewer:** Trong multi region marketplace, làm chức năng payment, thì giá sản phẩm được lưu vào database như thế nào khi có nhiều loại đơn vị tiền tệ khác nhau?

**Candidate:**

Đây là bài toán mình gặp khá nhiều ở marketplace đa quốc gia. Câu trả lời ngắn: **không lưu một cột `price` duy nhất**, mà tách rõ ba lớp — **giá hiển thị (list price)**, **tỷ giá (exchange rate)**, và **giá chốt tại thời điểm thanh toán (price snapshot)**. Mỗi lớp phục vụ một mục đích khác nhau.

---

### Nguyên tắc vàng khi lưu tiền vào database

Trước hết, **không bao giờ dùng float/double** để lưu tiền — lỗi làm tròn sẽ tích tụ và gây lệch sổ sách. Mình hay dùng một trong hai cách:

- Lưu **minor unit (đơn vị nhỏ nhất)** dưới dạng integer — VND lưu nguyên đồng, USD lưu cents (1 USD = 100 cents), JPY (yên Nhật) cũng lưu nguyên vì không có decimal (số thập phân).
- Hoặc dùng kiểu **DECIMAL** với precision (độ chính xác) cố định — ví dụ `DECIMAL(19, 4)`.

Kèm theo luôn **currency code** theo chuẩn ISO 4217 — `VND`, `USD`, `EUR`, `JPY`. Không lưu ký hiệu `₫` hay `$` trong database.

---

### Cách lưu giá sản phẩm — hai approach chính

**Approach 1: Base currency + quy đổi runtime (chuyển đổi khi chạy)**

Lưu một giá gốc, thường là USD, rồi khi user ở region khác thì nhân với tỷ giá hiện tại để hiển thị.

Ưu điểm: admin chỉ cần cập nhật một giá, đơn giản. Nhược điểm: giá hiển thị thay đổi theo tỷ giá — user VN hôm nay thấy 500.000đ, mai có thể 510.000đ dù seller không đổi giá. Với marketplace, điều này thường **không chấp nhận được** vì seller muốn kiểm soát giá từng thị trường, và marketing hay chạy campaign (chiến dịch) giá cố định theo region.

**Approach 2: Giá cố định theo từng currency — đây là cách mình recommend (khuyên dùng) cho marketplace**

Tách bảng `product_prices` (hoặc embed trong product nếu ít currency):

Mỗi record gồm: `product_id`, `currency`, `amount_minor` (hoặc amount decimal), `region` (tùy chọn), `effective_from` / `effective_to` (hiệu lực từ/đến).

Một sản phẩm có thể có: 29.99 USD cho US, 699.000 VND cho VN, 24.99 EUR cho EU — **do seller hoặc ops team set (thiết lập)**, không phải máy tự quy đổi.

Ưu điểm: giá ổn định, dễ làm pricing strategy (chiến lược định giá) theo thị trường, dễ audit (kiểm toán). Nhược điểm: phải maintain (duy trì) nhiều giá hơn, nhưng với marketplace thì trade-off (đánh đổi) này xứng đáng.

Nếu seller tự định giá, mình thường cho họ chọn currency chính rồi nhập giá các market khác — có thể suggest (gợi ý) bằng tỷ giá nhưng **không auto-overwrite (tự động ghi đè)** trừ khi seller đồng ý.

---

### Tỷ giá (exchange rate) lưu riêng

Dù dùng approach nào, mình vẫn có bảng `exchange_rates` riêng:

`from_currency`, `to_currency`, `rate`, `source` (nguồn — ECB, Stripe, manual), `effective_at` (thời điểm hiệu lực).

Dùng cho: gợi ý giá cho seller, reconcile (đối soát) doanh thu về base currency của platform, báo cáo tài chính. **Không dùng tỷ giá real-time (thời gian thực) để tính tiền user phải trả** trừ khi business chấp nhận risk (rủi ro) đó.

---

### Phần quan trọng nhất cho payment: price snapshot (chụp giá)

Khi user checkout (thanh toán), **không bao giờ đọc lại giá từ bảng product** — phải snapshot vào `order_items`:

Mỗi line item lưu: `unit_price`, `currency`, `quantity`, `subtotal`, kèm `fx_rate` nếu có quy đổi, `tax_amount`, `discount_amount`.

Lý do: giá product có thể đổi ngày mai, seller có thể sửa giá, campaign kết thúc — nhưng order đã tạo thì **immutable (bất biến)**. Đây là requirement (yêu cầu) cơ bản cho accounting (kế toán), refund (hoàn tiền), dispute (khiếu nại).

Tương tự, bảng `payments` lưu: `amount`, `currency`, `presentment_currency` (tiền hiển thị cho user), `settlement_currency` (tiền platform/seller nhận), `exchange_rate_at_payment` — vì Stripe/PayPal có thể charge user bằng VND nhưng settle (quyết toán) về USD cho seller.

---

### Multi region còn ảnh hưởng gì nữa?

**Presentment vs settlement currency (tiền hiển thị vs tiền quyết toán):** User VN thấy giá VND, thanh toán VND qua MoMo/VNPay — nhưng nếu seller US thì platform phải quy đổi và chuyển USD. Payment gateway (cổng thanh toán) như Stripe hỗ trợ multi-currency nhưng fee (phí) và FX spread (chênh lệch tỷ giá) khác nhau — cần quyết định ai chịu phí chuyển đổi.

**Rounding rules (quy tắc làm tròn):** Mỗi currency có quy tắc khác — VND không có xu, USD làm tròn 2 chữ số. Khi quy đổi phải define rule: làm tròn ở bước nào, theo currency nào. Sai rounding có thể lệch vài đồng mỗi order, nhân lên hàng triệu order là thành vấn đề.

**Tax/VAT theo region:** Giá product có thể là giá chưa thuế hay đã gồm thuế tùy thị trường. EU thường hiển thị giá inclusive (đã gồm thuế), US thường exclusive (chưa thuế). Nên tách `price_amount` và `tax_amount` rõ ràng, không nhồi chung.

**Rate locking (khóa tỷ giá):** Nếu có cross-currency checkout — user VN mua sản phẩm giá USD — nên lock tỷ giá tại thời điểm add to cart hoặc bắt đầu checkout, với TTL (thời gian sống) ví dụ 15–30 phút. Hết TTL thì refresh rate và confirm lại với user.

---

### Flow thực tế mình hay implement

User vào site → detect region/currency từ IP, preference (tùy chọn), hoặc user setting → query `product_prices` theo currency đó → hiển thị giá.

User checkout → tạo `order` + snapshot giá vào `order_items` → tính tax theo region → tổng `order.total_amount` + `order.currency`.

Gọi payment gateway với đúng `amount` + `currency` đã snapshot → webhook (callback) confirm → update `payment` status.

Reconciliation (đối soát) cuối ngày: quy đổi về base currency của platform để báo cáo, dùng tỷ giá tại thời điểm payment đã lưu, không dùng tỷ giá hiện tại.

---

### Những mistake (lỗi) hay gặp

Lưu giá bằng float → lệch tiền. Chỉ lưu một currency rồi convert real-time lúc checkout → user thấy giá nhảy, dispute nhiều. Không snapshot giá vào order → refund và accounting nightmare (ác mộng kế toán). Nhầm presentment và settlement currency → seller nhận sai số tiền. Hardcode (mã hóa cứng) exchange rate trong code thay vì bảng riêng → không audit được.

Tóm lại: **lưu giá theo từng currency trong bảng riêng, dùng minor unit integer, snapshot toàn bộ giá và tỷ giá vào order tại thời điểm thanh toán** — product price là "giá niêm yết", order price là "giá giao dịch", hai thứ khác nhau và không được trộn lẫn.

---

## Câu 5

**Interviewer:** Làm gì khi có lỗi trên production? Phân biệt lỗi FE và BE như thế nào?

**Candidate:**

Khi có lỗi production, mình không panic (hoảng loạn) mà đi theo một quy trình cố định — **detect (phát hiện) → triage (phân loại) → mitigate (giảm thiểu) → fix → post-mortem (rút kinh nghiệm)**. Với backend dev thì phần quan trọng nhất là biết nhanh lỗi thuộc phía nào, impact (ảnh hưởng) bao nhiêu user, và có cần rollback (hoàn tác) ngay không.

---

### Bước 1: Nhận biết và đánh giá mức độ

Lỗi production thường đến từ nhiều kênh: alert từ monitoring (giám sát) như Datadog, Sentry, PagerDuty; user report qua support; hoặc QA phát hiện sau deploy (triển khai).

Việc đầu tiên mình làm là trả lời ba câu hỏi:

- **Ai bị ảnh hưởng?** Toàn bộ user hay một segment (nhóm) — chỉ mobile, chỉ region VN, chỉ seller?
- **Mức độ nghiêm trọng?** Payment down (sập thanh toán) là P0, trang About lỗi là P3. Mình hay dùng severity (mức độ): P0 critical (nghiêm trọng), P1 high, P2 medium, P3 low.
- **Có workaround (cách xử lý tạm) không?** Nếu user vẫn checkout được qua kênh khác thì có thời gian investigate (điều tra) kỹ hơn.

Song song đó, mình check **dashboard (bảng điều khiển)**: error rate (tỷ lệ lỗi), latency (độ trễ), throughput (thông lượng), CPU/memory của service. Một spike (đột biến) 5xx sau deploy gần đây thì nghi ngờ release mới trước.

---

### Bước 2: Mitigate trước, root cause (nguyên nhân gốc) sau

Nguyên tắc incident response (xử lý sự cố): **restore service (khôi phục dịch vụ) trước, tìm nguyên nhân sau**.

Các hành động mitigate thường gặp:

- **Rollback** deployment nếu lỗi xuất hiện ngay sau release — đây là cách nhanh nhất nếu có CI/CD (tích hợp/triển khai liên tục) tốt.
- **Feature flag (cờ tính năng)** tắt feature đang lỗi mà không cần deploy lại — lý do mình luôn advocate (ủng hộ) feature flag cho feature mới.
- **Scale up** nếu là vấn đề capacity (sức chứa) — traffic tăng đột ngột, connection pool (bể kết nối) cạn.
- **Circuit breaker (cầu dao)** ngắt call sang dependency (phụ thuộc) đang down — ví dụ recommendation service chết thì trả empty list thay vì crash cả trang chủ.
- **Kill switch (công tắc dừng khẩn)** cho payment nếu phát hiện double charge (trừ tiền hai lần) — ưu tiên bảo vệ tiền user.

Trong lúc mitigate, mình update status page hoặc channel nội bộ — communication (giao tiếp) quan trọng không kém fix.

---

### Bước 3: Investigate — tìm root cause

Sau khi service ổn định lại, mình đi sâu vào nguyên nhân. Với backend Node.js, toolchain (bộ công cụ) mình hay dùng:

- **Logs (nhật ký)** có structured logging (ghi log có cấu trúc) — JSON với `requestId`, `userId`, `traceId` để correlate (liên kết) request xuyên suốt hệ thống.
- **APM (Application Performance Monitoring — giám sát hiệu năng ứng dụng)** — Datadog, New Relic — xem slow query, memory leak, error stack.
- **Distributed tracing (theo dõi phân tán)** — Jaeger, OpenTelemetry — trace một request từ API gateway qua nhiều microservice.
- **Sentry** cho error tracking (theo dõi lỗi) — group (nhóm) lỗi theo fingerprint (dấu vân tay), biết lỗi mới hay regression (tái phát).

Mình luôn cố reproduce (tái hiện) trước — dùng cùng input, cùng user role, cùng region. Không reproduce được thì nhìn vào log của user bị ảnh hưởng qua `requestId` họ cung cấp hoặc timestamp (mốc thời gian).

---

### Phân biệt lỗi FE và BE — đây là phần hay bị tranh cãi

Thực tế không phải lúc nào cũng black-and-white (đen trắng). Nhưng mình có checklist (danh sách kiểm tra) để phân loại nhanh:

**Dấu hiệu lỗi Backend:**

- Network tab (tab mạng) trong DevTools thấy API trả **4xx/5xx** — đặc biệt 500, 502, 503, 504.
- Response body có error structure từ server — `code`, `message`, stack trace (ở staging, không phải production).
- Lỗi xảy ra **cả trên mobile lẫn web**, hoặc cả khi gọi API trực tiếp qua Postman/curl — chứng tỏ không phải UI render.
- Monitoring backend spike error rate đúng thời điểm user report.
- Lỗi liên quan data — sai số tiền, duplicate order, permission denied (từ chối quyền) dù UI gửi đúng payload (dữ liệu gửi đi).

**Dấu hiệu lỗi Frontend:**

- API trả **200 OK** với data đúng, nhưng UI hiển thị sai — blank screen (màn hình trắng), crash JS, layout vỡ.
- Console (bảng điều khiển) browser có **JavaScript error** — `TypeError: Cannot read property 'x' of undefined`.
- Chỉ xảy ra trên **một browser/device** hoặc một version app cụ thể.
- Lỗi sau FE deploy, BE không deploy gì.
- User thao tác đúng flow nhưng FE không gửi request — button disabled (vô hiệu hóa) do validation phía client, hoặc race condition (điều kiện đua) trong state management (quản lý trạng thái).

**Lỗi nằm ở ranh giới — hay gặp nhất:**

- **Contract mismatch (lệch hợp đồng API):** BE đổi field từ `userName` sang `user_name`, FE chưa update → FE crash dù BE "đúng spec mới". Đây là lỗi **integration (tích hợp)**, không thuần FE hay BE — cần API versioning (phiên bản hóa API) và backward compatibility (tương thích ngược).
- **CORS error** — nhìn như FE nhưng thực ra BE chưa config header đúng.
- **Timeout** — FE báo "request failed" nhưng nguyên nhân có thể BE chậm, DB lock, hoặc network giữa các service.
- **Authentication (xác thực)** — token hết hạn: FE nên refresh (làm mới) token nhưng không làm → 401. Vừa là FE bug vừa có thể argue (tranh luận) BE nên trả error message rõ hơn.

Cách mình xử lý tranh chấp FE/BE: **lấy một `requestId` cụ thể, trace end-to-end (đầu cuối)**. Xem request FE gửi gì, BE nhận gì, BE trả gì, FE render gì. Trong 5 phút thường biết lỗi nằm ở tầng nào. Không đổ lỗi — focus vào fix.

---

### Quy trình collaboration (phối hợp) khi incident

Mình thường assign (phân công) **incident commander (chỉ huy sự cố)** — một người điều phối, không phải người fix. Người fix focus debug, commander lo communication và quyết định rollback hay không.

Channel war room (phòng họp khẩn) trên Slack/Teams, update mỗi 15–30 phút. Ghi timeline (mốc thời gian): lúc nào phát hiện, lúc nào mitigate, lúc nào fix xong.

Sau incident — **blameless post-mortem (rút kinh nghiệm không đổ lỗi cá nhân)**: what happened, why, action items (hạng mục hành động) — thêm alert, thêm test, cải thiện runbook (sổ tay xử lý). Mục tiêu là không lặp lại cùng một lỗi, không phải tìm người để blame (đổ lỗi).

---

### Những thứ mình chuẩn bị trước để đỡ vật khi incident

**Observability (khả năng quan sát) đầy đủ** — logs, metrics (số liệu), traces. Không có thì debug production như mò kim đáy bể.

**Request ID xuyên suốt** — FE gửi `X-Request-Id`, BE propagate (truyền tiếp) qua mọi service và log.

**Health check endpoint** — `/health` và `/ready` cho load balancer và k8s probe (kiểm tra).

**Rollback pipeline (quy trình hoàn tác)** một lệnh — deploy xong mà lỗi thì rollback trong 2 phút, không phải build lại.

**Runbook** cho incident thường gặp — DB connection pool exhausted (cạn kiệt), Redis down, payment webhook fail.

**On-call rotation (luân phiên trực)** — ai được gọi lúc 2h sáng phải rõ ràng.

---

### Ví dụ thực tế

Từng có incident checkout fail (thanh toán thất bại) — user báo "bấm thanh toán không được". Mình check Sentry: spike `TypeError` phía FE. Nhưng trace vào thì thấy BE trả field `totalAmount` dạng string thay vì number sau một PR refactor (tái cấu trúc). FE cũ expect number nên crash khi gọi `.toFixed()`.

Kết luận: **integration bug** — BE breaking change (thay đổi phá vỡ) không có versioning. Mitigate: BE hotfix (sửa gấp) trả lại number. Long-term (dài hạn): thêm JSON schema validation (xác thực schema) ở cả hai phía, integration test trong CI, và API contract test.

Tóm lại: **production incident thì mitigate trước, investigate sau; phân biệt FE/BE bằng cách trace một request cụ thể end-to-end, không đoán**; và quan trọng nhất là hệ thống observability và quy trình đã chuẩn bị sẵn — senior không chỉ biết fix bug mà biết thiết kế hệ thống để khi lỗi xảy ra thì detect và recover (phục hồi) nhanh.

---
