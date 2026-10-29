# Checklist ôn tập theo JD — DBA vận hành hệ thống

> Bám [JD.md](JD.md): vận hành database production on-premises/cloud, xử lý sự cố và change, giữ hiệu năng — ổn định — toàn vẹn — an toàn dữ liệu.
>
> Đánh dấu `[x]` khi **làm được**, không chỉ đọc qua. Mỗi mục nên trả lời được trong 2–3 phút hoặc kể được một tình huống thật.

## Cách dùng

| Mức | Ý nghĩa với JD |
| --- | --- |
| **P0** | Trách nhiệm chính và mục “kiến thức chuyên môn cần có”. Thiếu mục này là lệch JD. |
| **P1** | Nền tảng JD nêu tên (nhiều DBMS, OS). Nắm một engine sâu, các engine còn lại ở mức vận hành và so sánh. |
| **P2** | Điểm cộng. Distributed database được JD ghi **ưu tiên cao**; cloud, observability, automation đứng ngay sau. |

Tài liệu trong repo ghi đường dẫn tương đối. Mục ghi **tự học** chưa có chương riêng.

Thứ tự ôn đề xuất, khớp trọng số JD:

1. Backup / restore và toàn vẹn dữ liệu
2. Quy trình sự cố, change, request, problem
3. SQL tuning và lỗi production thường gặp
4. Monitoring, alert, session, job, dung lượng
5. Replication, HA/DR, failover
6. Review change/release và kiểm soát rủi ro sau triển khai
7. Bảo mật và access control
8. Linux (và Windows) phục vụ vận hành database
9. Một engine sâu (MySQL hoặc PostgreSQL) + mặt vận hành của engine còn lại
10. Distributed database, rồi cloud, observability, automation

---

## P0 — Phải nắm

### 1. Backup, restore, toàn vẹn dữ liệu

Tài liệu: [02-sao-luu-phuc-hoi](02-sao-luu-phuc-hoi/README.md), [RPO/RTO](02-sao-luu-phuc-hoi/rpo-rto.md), [chiến lược sao lưu](02-sao-luu-phuc-hoi/chien-luoc-sao-luu.md).

- [ ] Định nghĩa RPO và RTO, rồi chọn full / incremental / differential cho một hệ mission-critical
- [ ] Giải thích point-in-time recovery: WAL (PostgreSQL) hoặc binlog (MySQL) bù khoảng trống giữa các bản full
- [ ] Nêu vì sao backup chưa restore-test thì chưa tính là có backup; liệt kê bước kiểm tra restore
- [ ] Theo dõi job backup hằng ngày: thành công, kích thước, thời gian, bản copy offsite, mã hóa
- [ ] Phát hiện backup “xanh” nhưng không dùng được: file hỏng, thiếu log, retention sai, restore ra sai thời điểm
- [ ] Phân biệt logical backup và physical backup; khi nào dùng cái nào
- [ ] Mã hóa backup và kiểm soát ai được quyền restore

### 2. Hiệu năng SQL và lỗi database phổ biến

Tài liệu: [04-toi-uu-hieu-suat](04-toi-uu-hieu-suat/README.md), [execution plan](04-toi-uu-hieu-suat/phan-tich-truy-van.md), [index](04-toi-uu-hieu-suat/thiet-ke-index-nang-cao.md), [statistics](04-toi-uu-hieu-suat/thong-ke-va-ke-hoach-truy-van.md), [lock](04-toi-uu-hieu-suat/khoa-va-deadlock.md), [slow query](04-toi-uu-hieu-suat/phat-hien-truy-van-cham.md). Nền: [index cơ bản](01-co-ban/chien-luoc-index.md), [isolation](01-co-ban/muc-do-isolation.md), [connection pool](01-co-ban/connection-pooling.md).

- [ ] Đọc execution plan: full scan, index không được dùng, nested loop bất lợi, sort/hash tràn disk
- [ ] Chọn index đúng: single, composite, covering; biết index làm chậm ghi
- [ ] Giải thích statistics lệch làm planner chọn plan sai, và cách thu thập lại thống kê
- [ ] Phân biệt lock wait, blocking chain và deadlock; tìm session thủ phạm, xử lý có kiểm soát
- [ ] Mô tả partition (range / list / hash): khi nào giúp prune, khi nào làm change và backup khó hơn — **tự học** phần vận hành partition (repo chưa có chương riêng)
- [ ] Nhận diện lỗi production hay gặp: connection cạn, disk đầy, long transaction, replication lag, autovacuum/purge không kịp, temp đầy
- [ ] Với PostgreSQL: vacuum, bloat, XID wraparound — [autovacuum](04-toi-uu-hieu-suat/autovacuum-va-bloat.md)
- [ ] Đo trước khi tối ưu: slow log, `pg_stat_statements` hoặc tương đương, baseline trước/sau

### 3. Monitoring, log, session, job, dung lượng, alert

Tài liệu một phần: [phát hiện truy vấn chậm](04-toi-uu-hieu-suat/phat-hien-truy-van-cham.md), [vận hành cloud](07-cloud-database/quan-ly-va-van-hanh.md). Phần checklist/runbook/SOP vận hành hằng ngày: **tự học**.

- [ ] Liệt kê tín hiệu cần theo dõi: availability, connection, session active/idle-in-transaction, lock, replication lag, backup status, disk/tablespace, CPU, I/O, job failed
- [ ] Đọc error log / slow log và khoanh thời điểm sự cố bắt đầu
- [ ] Xử lý alert theo runbook: xác nhận alert thật, đo impact, làm bước an toàn, escalate, ghi evidence
- [ ] Phân biệt alert nhiễu và alert cần gọi người; nêu ngưỡng ví dụ (disk > 80%, lag > N giây, backup fail)
- [ ] Xem session đang chạy gì, ai giữ lock, query nào chiếm tài nguyên
- [ ] Kiểm tra job định kỳ (backup, stats, purge, archive log) và biết job nào fail là sự cố
- [ ] Capacity: tốc độ tăng datafile, ngày đầy disk ước tính, ngưỡng đặt mua/mở rộng trước khi ảnh hưởng dịch vụ

### 4. Replication, HA và failover

Tài liệu: [03-ha-replication](03-ha-replication/README.md), [các loại replication](03-ha-replication/cac-loai-replication.md), [sync vs async](03-ha-replication/dong-bo-va-bat-dong-bo.md), [failover](03-ha-replication/chien-luoc-failover.md). MySQL: [08-mysql-kien-truc-trien-khai](08-mysql-kien-truc-trien-khai/README.md).

- [ ] So sánh single-leader, multi-leader, leaderless; sync và async; đánh đổi RPO với độ trễ ghi
- [ ] Đo replication lag và nêu nguyên nhân thường gặp (apply chậm, network, transaction lớn, replica yếu)
- [ ] Mô tả failover có người xác nhận: khi nào failover, khi nào không, cách tránh split-brain
- [ ] Sau failover: ứng dụng trỏ đúng primary, replica bắt kịp, backup chạy lại, không có hai node cùng nhận ghi
- [ ] Nêu HA/DR: cùng site (failover nhanh) khác site/vùng (chịu mất cả datacenter), và RPO/RTO tương ứng

### 5. Change, release, migration — kiểm soát rủi ro

Tài liệu: [06-schema-migrations](06-schema-migrations/README.md), [forward-only](06-schema-migrations/chi-tien-forward-only.md), [expand-contract](06-schema-migrations/expand-contract.md), [rollback](06-schema-migrations/chien-luoc-rollback.md), [công cụ](06-schema-migrations/cong-cu-migration.md), [phối hợp](06-schema-migrations/giao-tiep-nhom.md). MySQL: [apply migration](08-mysql-kien-truc-trien-khai/6-apply-migration.md), [nâng cấp](08-mysql-kien-truc-trien-khai/5-nang-cap.md).

- [ ] Review một change database trước khi lên production: impact, lock, downtime, thời gian chạy, rollback, rủi ro dữ liệu
- [ ] Giải thích expand-contract và vì sao `ALTER` lớn trên bảng nóng không làm trong giờ cao điểm
- [ ] Nêu rollback thật sự làm được (script xuống, restore, feature flag) và rollback không làm được (đã xóa dữ liệu)
- [ ] Checklist sau go-live: error log, latency, lock, replication lag, disk, job, đối soát số dòng
- [ ] Dùng Git để xem diff migration; biết ai review, ai thực thi, ai xác nhận ứng dụng

### 6. Bảo mật và access control

Tài liệu: [05-bao-mat-tuan-thu](05-bao-mat-tuan-thu/README.md), [quyền](05-bao-mat-tuan-thu/kiem-soat-truy-cap.md), [mã hóa](05-bao-mat-tuan-thu/ma-hoa.md), [audit](05-bao-mat-tuan-thu/ghi-nhat-ky-kiem-tra.md), [mạng](05-bao-mat-tuan-thu/bao-mat-mang.md), [bí mật](05-bao-mat-tuan-thu/quan-ly-bi-mat.md).

- [ ] Cấp quyền tối thiểu: tách tài khoản ứng dụng, vận hành, cá nhân; không dùng superuser cho app
- [ ] Mô tả luồng cấp quyền production: ai yêu cầu, ai duyệt, hết hạn khi nào, thu hồi ra sao
- [ ] Mã hóa in transit (TLS) và at rest; bí mật không nằm trong script/repo
- [ ] Audit: ai đăng nhập, ai đổi quyền, ai đọc dữ liệu nhạy cảm; log đủ để điều tra
- [ ] Biết database nằm ở đâu trong yêu cầu PCI-DSS / GDPR ở mức vận hành (không cần thuộc lòng điều luật)

### 7. Hệ điều hành, mạng, storage phục vụ database

**Tự học** (repo chưa có chương OS/mạng/storage). Ưu tiên Linux; Windows Server ở mức vận hành SQL Server; AIX chỉ cần nhận ra khác biệt nếu JD hỏi.

- [ ] Linux: process, disk (`df`/`du`), inode, quyền file, log, `systemctl`, ulimit, cron, user chạy database
- [ ] Đọc được vì sao database không start: disk đầy, quyền data directory, port đã chiếm, OOM, config sai
- [ ] Mạng: port database, firewall, latency, DNS, vì sao app “timeout” trong khi database vẫn sống
- [ ] Storage: filesystem và datafile, IOPS, latency đĩa, tách log và data khi có thể, snapshot khác backup ứng dụng
- [ ] Windows Server (mức cơ bản): service database, Event Viewer, disk, quyền NTFS — đủ để hỗ trợ SQL Server

### 8. Quy trình vận hành: Incident, Change, Request, Problem

**Tự học** ITIL mức thực hành (không cần chứng chỉ để bắt đầu).

- [ ] Phân loại đúng ticket: Incident (dịch vụ đang hỏng), Change (thay đổi có kiểm soát), Request (yêu cầu chuẩn), Problem (nguyên nhân gốc của sự cố lặp)
- [ ] Xác định impact và ưu tiên theo SLA: hệ nào down, bao nhiêu user, có mất dữ liệu không, có workaround không
- [ ] Trong incident: cập nhật tình trạng, rủi ro, ETA, bước tiếp theo; không im lặng đến khi xong
- [ ] Change: cửa sổ bảo trì, kế hoạch rollback, người phê duyệt, kiểm tra sau triển khai
- [ ] Sau sự cố: timeline, nguyên nhân, hành động tránh lặp (RCA ngắn, đủ để ca sau dùng)
- [ ] Ghi evidence: lệnh đã chạy, output, thời điểm, người thực hiện

### 9. Phối hợp và giao tiếp khi xử lý sự cố

Không cần tài liệu riêng. Luyện bằng cách kể lại một case theo các mục dưới.

- [ ] Mở đầu incident bằng impact, việc đang làm, việc cần Dev / Infra / Security / Vendor, ETA tạm
- [ ] Nhờ đúng đội: Infra (máy, đĩa, mạng), Dev (SQL/ứng dụng), Security (quyền, lộ dữ liệu), Vendor (bug engine, license)
- [ ] Bàn giao ca: hiện trạng, việc dở, rủi ro còn, lệnh không được chạy lại
- [ ] Giữ cấu trúc khi áp lực: checklist ngắn, không nhảy bước restore hoặc failover khi chưa xác nhận

---

## P1 — Nền tảng JD nêu tên

Nắm **một** engine ở mức xử lý được sự cố. Các engine kia: kiến trúc lưu trữ, backup, replication, log, chỗ xem session. JD liệt kê Oracle, MySQL, PostgreSQL, SQL Server, MongoDB.

| Engine | Mức với JD | Trong repo |
| --- | --- | --- |
| MySQL | Sâu nếu chọn làm engine chính | [08](08-mysql-kien-truc-trien-khai/README.md): topology, cài đặt, proxy, failover, upgrade, migration |
| PostgreSQL | Sâu nếu chọn làm engine chính | Rải ở [02](02-sao-luu-phuc-hoi/sao-luu-postgresql.md), [04](04-toi-uu-hieu-suat/README.md), [01](01-co-ban/README.md) |
| Oracle | Mặt vận hành + so sánh | **Tự học** |
| SQL Server | Mặt vận hành + so sánh | **Tự học** (cloud chỉ điểm tên RDS/Azure SQL) |
| MongoDB | Replica set, backup, khi nào không thay RDBMS | Một phần trong [loại CSDL](01-co-ban/loai-co-so-du-lieu.md) và [NoSQL cloud](07-cloud-database/setup-nosql-cloud.md) |

- [ ] Với engine chính: start/stop, vị trí data và log, tạo user, backup/restore, xem session, xem replication, ba lỗi hay gặp
- [ ] MySQL: InnoDB, binlog/GTID, primary–replica, semisync hoặc Group Replication, nguy cơ split-brain — [topology](08-mysql-kien-truc-trien-khai/1-topology.md), [proxy và failover](08-mysql-kien-truc-trien-khai/3-proxy-va-failover.md)
- [ ] PostgreSQL: WAL, `EXPLAIN ANALYZE`, autovacuum, streaming replication, role và grant
- [ ] Oracle (mức nói được): instance và tablespace, redo/archive, RMAN, Data Guard, session/blocking, alert log, AWR/ASH hoặc OEM ở mức “dùng để xem gì”
- [ ] SQL Server (mức nói được): data/log file, recovery model, backup full/diff/log, Always On, blocking, error log, Agent job
- [ ] MongoDB (mức nói được): document và replica set, primary election, backup, khi nào thiếu transaction/join khiến không hợp hệ cần toàn vẹn mạnh

---

## P2 — Điểm cộng

### Distributed database — ưu tiên cao trong JD

Repo có replication/failover và sharding ở mức phỏng vấn ([câu sharding](11-phong-van/HUONG_DAN_PHONG_VAN.md)), chưa có chương cluster nhiều node. Phần còn lại: **tự học**.

- [ ] Sharding: chọn shard key, hotspot, query fan-out, reshard / rebalance tốn gì
- [ ] Partitioning trên một node khác sharding trên nhiều node
- [ ] Consistency model: strong, eventual; đọc replica trễ nghĩa là gì với nghiệp vụ
- [ ] Distributed transaction: vì sao 2PC dễ nghẽn và khó vận hành; pattern thường dùng thay thế (saga, outbox) ở mức khái niệm
- [ ] HA/DR nhiều node: quorum, failover, split-brain, tenant/cluster cô lập sự cố
- [ ] Monitor cụm: node nào primary, lag, unbalanced data, disk từng node, failover vừa xảy ra chưa

### Cloud database

Tài liệu: [07-cloud-database](07-cloud-database/README.md) (AWS và Azure). GCP và Oracle Cloud: **tự học** ở mức đối chiếu dịch vụ.

- [ ] Chọn managed SQL hay NoSQL: RDS/Aurora, Azure SQL/PostgreSQL/MySQL, DynamoDB, Cosmos DB, Redis
- [ ] Việc còn thuộc về DBA khi không có OS: parameter group, backup retention, PITR, Multi-AZ, read replica, maintenance window, failover
- [ ] Mạng và bảo mật cloud: VPC/VNet, security group, TLS, IAM auth, xoay secret — [bảo mật cloud](07-cloud-database/bao-mat-va-mang.md)
- [ ] DR trên cloud: snapshot, cross-region, RPO/RTO thực tế khác slide nhà cung cấp
- [ ] Chi phí: instance, storage, IOPS, backup storage, data transfer; right-size trước khi scale — [tối ưu chi phí](07-cloud-database/toi-uu-chi-phi.md)
- [ ] Đối chiếu tên dịch vụ GCP (Cloud SQL, AlloyDB, Spanner) và Oracle Cloud (Base DB, Autonomous) với RDS/Azure — đủ để không bị bí khi JD hỏi

### Observability

- [ ] Prometheus: metric database (up, connection, lag, disk), alert rule, vì sao alert phải có runbook
- [ ] Grafana: dashboard trực ca xem trong 1 phút — availability, lag, backup, disk, slow query
- [ ] Oracle Enterprise Manager hoặc tương đương (RDS Performance Insights, Azure metrics): nhìn wait, session, SQL chiếm tài nguyên

### Automation vận hành

**Tự học.** Mục tiêu là script nhỏ chạy được, không phải framework.

- [ ] Script health check: backup đêm qua, disk, replication lag, session bất thường, job fail
- [ ] Đọc log và rút dòng lỗi lặp (bash hoặc Python)
- [ ] Báo cáo ngắn: top SQL, tăng trưởng dung lượng, backup 7 ngày
- [ ] Biết giới hạn: script không được tự failover hoặc tự xóa dữ liệu khi chưa có người xác nhận

### Big data và streaming — lợi thế, không phải lõi JD

- [ ] Redis: cache, eviction, persistence, vì sao mất Redis không được làm mất dữ liệu nguồn — [caching](04-toi-uu-hieu-suat/chien-luoc-caching.md)
- [ ] Kafka: topic, consumer lag, database là nguồn hay đích; outbox khi cần ghi DB và bắn event cùng lúc (khái niệm)
- [ ] Elasticsearch: search index tách khỏi OLTP; reindex và độ trễ đồng bộ
- [ ] Redshift hoặc kho tương đương: khác OLTP (load theo lô, không phải hàng triệu giao dịch ngắn)

### Migration, upgrade, review SQL, chuẩn hóa vận hành

- [ ] Lập kế hoạch upgrade: bản vá, tương thích client, rolling hay downtime, rollback — [nâng cấp MySQL](08-mysql-kien-truc-trien-khai/5-nang-cap.md)
- [ ] Review SQL trước khi lên production: lock, index, plan, timeout, idempotent
- [ ] Đưa một việc vận hành lặp thành checklist hoặc runbook người trực ca làm được

---

## Sản phẩm cần có trước phỏng vấn

JD đánh giá cách làm việc, không chỉ định nghĩa. Chuẩn bị bốn mẫu ngắn, mỗi mẫu 1 trang:

- [ ] **Runbook** một alert (replication lag hoặc disk > 80%): dấu hiệu, lệnh kiểm tra, ngưỡng escalate, việc không được làm
- [ ] **Checklist change** database: impact, downtime, rollback, rủi ro dữ liệu, theo dõi sau go-live
- [ ] **RCA** một sự cố: timeline, impact, nguyên nhân, hành động
- [ ] **Handover** cuối ca: đang ổn gì, đang treo gì, rủi ro đêm
- [ ] Một câu chuyện STAR: sự cố production, vai trò của mình, phối hợp với đội khác, kết quả — khung trong [hướng dẫn phỏng vấn](11-phong-van/HUONG_DAN_PHONG_VAN.md)

---

## Khoảng trống so với repo hiện tại

| Hạng mục JD | Tài liệu sẵn | Cần bổ sung ngoài repo |
| --- | --- | --- |
| Backup / restore | [02](02-sao-luu-phuc-hoi/README.md) | Đối chiếu công cụ engine mình chọn (RMAN, XtraBackup, pgBackRest) |
| SQL tuning, lock, statistics | [04](04-toi-uu-hieu-suat/README.md) | Partition vận hành; plan của Oracle/SQL Server |
| HA, replication, failover | [03](03-ha-replication/README.md), [08](08-mysql-kien-truc-trien-khai/README.md) | Drill failover trên lab |
| Change / migration | [06](06-schema-migrations/README.md) | Một change thật viết đủ mục rủi ro |
| Bảo mật | [05](05-bao-mat-tuan-thu/README.md) | Luồng cấp quyền production kiểu ITSM |
| Cloud AWS/Azure | [07](07-cloud-database/README.md) | GCP, Oracle Cloud (đối chiếu tên và mô hình HA/backup) |
| MySQL triển khai | [08](08-mysql-kien-truc-trien-khai/README.md) | Lab primary–replica |
| Oracle, SQL Server, MongoDB vận hành | Mỏng | Một trang “cheat sheet vận hành” mỗi engine |
| OS, mạng, storage | Chưa có | Linux cho DBA |
| Incident / Change / Request / Problem | Chưa có | ITIL thực hành + mẫu ticket |
| Session, job, capacity, alert/runbook | Chưa có | Checklist trực ca |
| Distributed (shard, consistency, rebalance) | Vài đoạn phỏng vấn | Chương tự học có chủ đích |
| Grafana, Prometheus, OEM | Nhắc tên | Một dashboard và một alert có runbook |
| Script vận hành | Chưa có | Health check backup + disk + lag |
| Kafka, Elasticsearch, Redshift | Redis/caching có sẵn | Đọc mức khái niệm vận hành |

---

## Tự kiểm tra nhanh

Trả lời miệng, không mở tài liệu:

1. Backup đêm qua thành công. Sáng nay cần dữ liệu lúc 02:17. Bạn làm gì, và vì sao có thể vẫn thất bại?
2. Ứng dụng timeout. Database vẫn `up`. Bạn kiểm tra theo thứ tự nào?
3. Replica lag 10 phút. Bạn failover không? Cần thêm dữ kiện gì?
4. Dev muốn `ALTER TABLE` thêm cột `NOT NULL` không default trên bảng 200 GB lúc 10:00. Bạn hỏi gì trước khi đồng ý?
5. Disk 92%. Việc làm trong 15 phút tới, việc làm trong tuần này?
6. Một change vừa xong, error log tăng, latency P95 gấp đôi. Bạn rollback hay xử lý tiếp? Dựa vào đâu?
7. Sharding theo `user_id` và theo `created_date` khác nhau thế nào về hotspot?
8. Kể một incident: impact, việc đã làm, cập nhật cho các bên, evidence để lại.
