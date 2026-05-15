# AWS Database Services — Bảng Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Database Services — dịch vụ cơ sở dữ liệu trên Amazon Web Services

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/databases/
├── README.md                                   [BẮT ĐẦU TỪ ĐÂY] Lộ trình học & tổng quan
├── INDEX.md                                    Bảng chỉ mục đầy đủ (file này)
│
├── 01-rds-fundamentals/
│   ├── README.md                               Tổng quan RDS, engine types, storage
│   ├── 1-engine-types.md                       (Tạo sau) MySQL, PostgreSQL, MariaDB, Oracle, SQL Server
│   ├── 2-instance-storage.md                   (Tạo sau) Instance classes, storage types, IOPS
│   ├── 3-multi-az.md                           (Tạo sau) Multi-AZ deployment, failover tự động
│   ├── 4-read-replicas.md                      (Tạo sau) Read replicas, cross-region replication
│   ├── 5-parameter-groups.md                   (Tạo sau) Parameter groups, option groups, tuning
│   └── 6-rds-proxy.md                          (Tạo sau) RDS Proxy, connection pooling
│
├── 02-aurora/
│   ├── README.md                               (Tạo sau) Tổng quan Aurora, kiến trúc, so sánh
│   ├── 1-aurora-architecture.md                (Tạo sau) Shared storage, cluster endpoints
│   ├── 2-aurora-serverless.md                  (Tạo sau) Aurora Serverless v2, auto-scaling
│   ├── 3-aurora-global.md                      (Tạo sau) Global Database, cross-region replication
│   ├── 4-aurora-vs-rds.md                      (Tạo sau) Trade-offs, cost comparison
│   └── 5-aurora-ha-failover.md                 (Tạo sau) High availability, failover mechanisms
│
├── 03-dynamodb/
│   ├── README.md                               (Tạo sau) Tổng quan DynamoDB, data model
│   ├── 1-data-model.md                         (Tạo sau) Tables, partition key, sort key, attributes
│   ├── 2-capacity-modes.md                     (Tạo sau) On-demand vs provisioned, auto-scaling
│   ├── 3-indexes.md                            (Tạo sau) GSI, LSI — thiết kế và use cases
│   ├── 4-streams-lambda.md                     (Tạo sau) DynamoDB Streams, Lambda integration
│   ├── 5-transactions-acid.md                  (Tạo sau) Transactions, ACID trên DynamoDB
│   ├── 6-dax.md                                (Tạo sau) DynamoDB Accelerator, in-memory cache
│   └── 7-access-patterns.md                    (Tạo sau) Single-table design, access patterns
│
├── 04-elasticache/
│   ├── README.md                               (Tạo sau) Tổng quan ElastiCache, Redis vs Memcached
│   ├── 1-redis-vs-memcached.md                 (Tạo sau) So sánh chi tiết, khi nào dùng gì
│   ├── 2-redis-cluster.md                      (Tạo sau) Cluster mode, replication groups, sharding
│   ├── 3-caching-strategies.md                 (Tạo sau) Lazy loading, write-through, write-around
│   ├── 4-persistence.md                        (Tạo sau) RDB, AOF, backup/restore
│   └── 5-security.md                           (Tạo sau) VPC, encryption, auth tokens, IAM
│
├── 05-ha-backup/
│   ├── README.md                               (Tạo sau) HA strategies, backup overview
│   ├── 1-rpo-rto.md                            (Tạo sau) RPO, RTO — business requirements, design
│   ├── 2-automated-backups.md                  (Tạo sau) RDS automated backups, retention policy
│   ├── 3-snapshots.md                          (Tạo sau) Manual snapshots, cross-region copy
│   ├── 4-pitr.md                               (Tạo sau) Point-in-Time Recovery, restore procedures
│   └── 5-disaster-recovery.md                  (Tạo sau) DR strategies, multi-region failover
│
├── 06-security/
│   ├── README.md                               (Tạo sau) Bảo mật cơ sở dữ liệu toàn diện
│   ├── 1-vpc-security-groups.md                (Tạo sau) VPC design, private subnets, security groups
│   ├── 2-iam-authentication.md                 (Tạo sau) IAM roles, database authentication
│   ├── 3-encryption.md                         (Tạo sau) KMS, encryption at rest & in transit
│   ├── 4-secrets-management.md                 (Tạo sau) Secrets Manager, Parameter Store, rotation
│   └── 5-audit-compliance.md                   (Tạo sau) CloudTrail, Activity Streams, PCI/HIPAA/GDPR
│
├── 07-performance-tuning/
│   ├── README.md                               (Tạo sau) Tối ưu hiệu năng — methodology
│   ├── 1-performance-insights.md               (Tạo sau) RDS Performance Insights, top SQL
│   ├── 2-cloudwatch-metrics.md                 (Tạo sau) Key metrics, enhanced monitoring
│   ├── 3-slow-query-analysis.md                (Tạo sau) Slow query log, EXPLAIN plans
│   ├── 4-connection-pooling.md                 (Tạo sau) RDS Proxy, pgBouncer, connection limits
│   ├── 5-dynamodb-performance.md               (Tạo sau) Hot partitions, throttling, DAX
│   └── 6-index-optimization.md                 (Tạo sau) Index design, GSI optimization, covering indexes
│
├── 08-migration/
│   ├── README.md                               (Tạo sau) Database migration — chiến lược & công cụ
│   ├── 1-dms-overview.md                       (Tạo sau) AWS DMS architecture, task types
│   ├── 2-sct.md                                (Tạo sau) Schema Conversion Tool, heterogeneous migration
│   ├── 3-cdc-online-migration.md               (Tạo sau) CDC, online migration, zero-downtime
│   ├── 4-migration-strategies.md               (Tạo sau) Lift-and-shift, re-platform, re-architect
│   └── 5-cutover-runbook.md                    (Tạo sau) Cutover planning, rollback, validation
│
├── 09-monitoring/
│   ├── README.md                               (Tạo sau) Monitoring & observability toàn diện
│   ├── 1-cloudwatch-dashboards.md              (Tạo sau) Dashboard setup, key database metrics
│   ├── 2-alerting-strategy.md                  (Tạo sau) Thresholds, SLO, alert routing
│   ├── 3-rds-enhanced-monitoring.md            (Tạo sau) OS-level metrics, process monitoring
│   ├── 4-dynamodb-monitoring.md                (Tạo sau) DynamoDB CloudWatch alarms, capacity alerts
│   └── 5-activity-streams.md                   (Tạo sau) Database Activity Streams, audit integration
│
├── 10-advanced/
│   ├── README.md                               (Tạo sau) Chủ đề nâng cao — multi-region, analytics
│   ├── 1-redshift.md                           (Tạo sau) Redshift architecture, distribution keys, Spectrum
│   ├── 2-aurora-global-advanced.md             (Tạo sau) Multi-region active-active, conflict resolution
│   ├── 3-dynamodb-global-tables.md             (Tạo sau) Global Tables, eventual consistency, conflicts
│   ├── 4-neptune.md                            (Tạo sau) Graph database, Gremlin, SPARQL
│   ├── 5-documentdb.md                         (Tạo sau) DocumentDB, MongoDB migration, compatibility
│   └── 6-timestream.md                         (Tạo sau) Time-series database, IoT use cases
│
├── 11-cost-optimization/
│   ├── README.md                               (Tạo sau) Tối ưu chi phí database trên AWS
│   ├── 1-rds-pricing.md                        (Tạo sau) RDS pricing models, reserved instances
│   ├── 2-aurora-cost.md                        (Tạo sau) Aurora vs RDS cost, serverless cost model
│   ├── 3-dynamodb-cost.md                      (Tạo sau) On-demand vs provisioned, auto-scaling
│   ├── 4-elasticache-cost.md                   (Tạo sau) ElastiCache pricing, reserved nodes
│   └── 5-rightsizing.md                        (Tạo sau) Instance sizing, storage optimization
│
└── 12-interview-prep/
    ├── README.md                               (Tạo sau) Tổng quan chuẩn bị phỏng vấn
    ├── INTERVIEW_GUIDE.md                      (Tạo sau) Top 20 câu hỏi, tips & tricks
    ├── system-design-scenarios.md              (Tạo sau) Kịch bản thiết kế hệ thống
    ├── trade-off-discussions.md                (Tạo sau) SQL vs NoSQL, RDS vs DynamoDB
    └── star-stories.md                         (Tạo sau) Mẫu câu chuyện STAR theo sự cố
```

---

## ✅ Những Gì Đã Được Tạo

| Chủ Đề                                        | File                                        | Trạng Thái | Chất Lượng    |
| --------------------------------------------- | ------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**                      | README.md                                   | ✅          | Toàn Diện    |
| **Bảng Chỉ Mục Đầy Đủ**                      | INDEX.md                                    | ✅          | Toàn Diện    |
| **RDS Fundamentals — Tổng Quan**              | 01-rds-fundamentals/README.md               | ✅          | Toàn Diện    |
| **RDS — Engine Types**                        | 01-rds-fundamentals/1-engine-types.md       | ✅          | Toàn Diện    |
| **RDS — Instance & Storage**                  | 01-rds-fundamentals/2-instance-storage.md   | ✅          | Toàn Diện    |
| **RDS — Multi-AZ Deployment**                 | 01-rds-fundamentals/3-multi-az.md           | ✅          | Toàn Diện    |
| **RDS — Read Replicas**                       | 01-rds-fundamentals/4-read-replicas.md      | ✅          | Toàn Diện    |
| **RDS — Parameter Groups & Option Groups**    | 01-rds-fundamentals/5-parameter-groups.md   | ✅          | Toàn Diện    |
| **RDS — RDS Proxy**                           | 01-rds-fundamentals/6-rds-proxy.md          | ✅          | Toàn Diện    |

---

## 🎯 Cần Tạo Tiếp (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao — Kỹ Năng Core

- [x] `01-rds-fundamentals/README.md` — RDS overview, engine types, Multi-AZ, Read Replicas ✅ **Hoàn thành**
- [ ] `02-aurora/README.md` — Aurora architecture, cluster, serverless, global database
- [ ] `03-dynamodb/README.md` — DynamoDB data model, capacity modes, indexes
- [ ] `05-ha-backup/README.md` — HA strategies, backup, PITR, disaster recovery
- [ ] `12-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao

- [ ] `04-elasticache/README.md` — ElastiCache Redis, Memcached, caching strategies
- [ ] `06-security/README.md` — VPC, IAM, KMS, Secrets Manager, compliance
- [ ] `07-performance-tuning/README.md` — Performance Insights, slow query, RDS Proxy
- [ ] `08-migration/README.md` — DMS, SCT, CDC, cutover planning

### Ưu Tiên Thấp Hơn — Tham Khảo

- [ ] `09-monitoring/README.md` — CloudWatch, alerting, Activity Streams
- [ ] `10-advanced/README.md` — Redshift, Neptune, DocumentDB, Timestream
- [ ] `11-cost-optimization/README.md` — Reserved instances, right-sizing
- [ ] `12-interview-prep/system-design-scenarios.md` — Design problems

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình Học (Beginner/Intermediate/Advanced)
3. Học từng section theo thứ tự
4. Thực hành trên AWS Console (dùng Free Tier)
5. Xây dựng portfolio project kết hợp nhiều dịch vụ
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào RDS + Aurora + DynamoDB (luôn được hỏi)
3. Học 05-ha-backup/ (backup, HA luôn xuất hiện trong phỏng vấn)
4. Chuẩn bị trade-off discussions (SQL vs NoSQL, RDS vs DynamoDB)
5. Luyện tập system design với database considerations
6. Chuẩn bị câu chuyện STAR về database incidents
```

### Cho Vai Trò Thực Tế

```
Dùng làm tài liệu tham khảo:
- Vấn đề hiệu năng:   Xem 07-performance-tuning/
- Sự cố & giám sát:   Xem 09-monitoring/ và 05-ha-backup/
- Di chuyển database: Xem 08-migration/
- Bảo mật:            Xem 06-security/
- Tối ưu chi phí:     Xem 11-cost-optimization/
```

### Cho Thiết Kế Hệ Thống

```
1. Đọc README.md — phần Tổng Quan Các Dịch Vụ để chọn DB phù hợp
2. Tham khảo 02-aurora/ cho high-traffic OLTP
3. Tham khảo 03-dynamodb/ cho key-value/document workloads
4. Dùng 05-ha-backup/ để thiết kế DR strategy
5. Dùng 04-elasticache/ để thêm caching layer
```

---

## 📊 Ước Tính Thời Gian Học

| Section                                       | Thời Gian   | Độ Khó  | Ưu Tiên |
| --------------------------------------------- | ----------- | ------- | ------- |
| RDS Fundamentals — Nền Tảng RDS               | 4-6 giờ     | ⭐       | Bắt buộc |
| Aurora — Cơ Sở Dữ Liệu Đám Mây               | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| DynamoDB — NoSQL Serverless                   | 8-10 giờ    | ⭐⭐⭐   | Bắt buộc |
| ElastiCache — Bộ Nhớ Đệm                      | 3-4 giờ     | ⭐⭐     | Bắt buộc |
| HA & Backup — Sẵn Sàng Cao & Sao Lưu         | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| Security — Bảo Mật                            | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| Performance Tuning — Tối Ưu Hiệu Năng        | 6-8 giờ     | ⭐⭐⭐   | Nên Có  |
| Migration — Di Chuyển Database               | 4-6 giờ     | ⭐⭐     | Nên Có  |
| Monitoring — Giám Sát                         | 3-4 giờ     | ⭐⭐     | Nên Có  |
| Advanced — Nâng Cao (Redshift, Neptune...)    | 10-15 giờ   | ⭐⭐⭐   | Tốt Nếu Có |
| Cost Optimization — Tối Ưu Chi Phí           | 2-3 giờ     | ⭐       | Tốt Nếu Có |

**Tổng thời gian: 52-74 giờ để nắm vững AWS Database Services**

---

## 🎓 Mức Độ Kỹ Năng

### Beginner — Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Phân biệt RDS, Aurora, DynamoDB
- [ ] Hiểu Multi-AZ (Đa Vùng Sẵn Sàng) và Read Replicas (Bản Sao Đọc)
- [ ] Cơ bản DynamoDB — table, partition key, sort key
- [ ] Cấu hình automated backup (sao lưu tự động)
- [ ] Bảo mật cơ bản với VPC và Security Groups

**Thời gian đạt mức này:** 1-2 tháng

### Intermediate — Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết kế Aurora Cluster với HA
- [ ] DynamoDB access patterns & GSI design
- [ ] ElastiCache Redis cho caching & sessions
- [ ] RDS Performance Insights & slow query analysis
- [ ] Database migration với DMS
- [ ] Security hardening — KMS, Secrets Manager, IAM

**Thời gian để nâng cấp:** 2-3 tháng

### Advanced — Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Multi-region active-active architecture (Kiến Trúc Đa Vùng Chủ-Chủ)
- [ ] DynamoDB Global Tables & conflict resolution
- [ ] Redshift architecture & query optimization
- [ ] Capacity planning (Lập Kế Hoạch Năng Lực) & cost modeling
- [ ] Compliance frameworks (PCI-DSS, HIPAA, GDPR)
- [ ] Zero-downtime migration strategies

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Cần Gì                                  | Vị Trí                                                                             |
| --------------------------------------- | ---------------------------------------------------------------------------------- |
| Tổng quan nhanh                         | [README.md](README.md)                                                             |
| RDS basics                              | [01-rds-fundamentals/README.md](01-rds-fundamentals/README.md)                     |
| Aurora deep dive                        | [02-aurora/README.md](02-aurora/README.md)                                         |
| DynamoDB patterns                       | [03-dynamodb/README.md](03-dynamodb/README.md)                                     |
| ElastiCache Redis                       | [04-elasticache/README.md](04-elasticache/README.md)                               |
| Backup & disaster recovery              | [05-ha-backup/README.md](05-ha-backup/README.md)                                   |
| Bảo mật database                        | [06-security/README.md](06-security/README.md)                                     |
| Tối ưu hiệu năng                        | [07-performance-tuning/README.md](07-performance-tuning/README.md)                 |
| Di chuyển database                      | [08-migration/README.md](08-migration/README.md)                                   |
| Câu hỏi phỏng vấn                       | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md)       |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Database Services — Tiến Độ Hoàn Thành

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] RDS engine types & storage
- [ ] Multi-AZ vs Read Replicas
- [ ] DynamoDB table design cơ bản
- [ ] Automated backups & snapshots
- [ ] VPC & security groups

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] Aurora cluster & serverless
- [ ] DynamoDB GSI/LSI & access patterns
- [ ] ElastiCache Redis caching strategies
- [ ] RDS Performance Insights
- [ ] KMS encryption & Secrets Manager
- [ ] DMS database migration

### Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7-10)

- [ ] Multi-region HA design
- [ ] DynamoDB Global Tables
- [ ] Redshift basics
- [ ] Cost optimization strategies
- [ ] Monitoring & alerting setup

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Neptune & DocumentDB
- [ ] Aurora Global Database advanced
- [ ] Compliance frameworks
- [ ] System design với database
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích sự khác biệt RDS, Aurora, DynamoDB, ElastiCache không cần ghi chú
- [ ] Thiết kế backup strategy cho RPO/RTO đã cho
- [ ] Lựa chọn database phù hợp cho từng use case
- [ ] Đọc và phân tích CloudWatch metrics cơ bản

### ✅ Năng Lực Vận Hành

- [ ] Troubleshoot slow queries với Performance Insights
- [ ] Thiết lập Multi-AZ và Read Replicas đúng cách
- [ ] Thực hiện database migration với DMS không có downtime
- [ ] Implement security best practices (KMS, VPC, IAM)

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời 20 câu hỏi AWS Database tự tin
- [ ] Kể 2-3 câu chuyện database incident (STAR format)
- [ ] Thiết kế hệ thống có database considerations
- [ ] Thảo luận trade-offs giữa các dịch vụ
- [ ] Hiểu sâu ít nhất 1 dịch vụ (RDS/Aurora hoặc DynamoDB)

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp với kinh nghiệm của bạn
3. Xem qua `01-rds-fundamentals/README.md`
4. Tạo tài khoản AWS Free Tier nếu chưa có

### Ngắn Hạn (2 Tuần Tiếp)

1. Hoàn thành `01-rds-fundamentals/` và `02-aurora/`
2. Bắt đầu học DynamoDB cơ bản — tạo table thử nghiệm
3. Thực hành trên AWS Console — tạo RDS instance (Free Tier)

### Trung Hạn (4 Tuần Tiếp)

1. Hoàn thành tất cả core sections (01-06)
2. Deep dive một dịch vụ (Aurora hoặc DynamoDB)
3. Chuẩn bị 2-3 câu chuyện database incident
4. Thực hiện mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tiếp)

1. Nắm vững một dịch vụ hoàn toàn
2. Hiểu trade-offs giữa tất cả dịch vụ
3. Xây dựng portfolio project dùng nhiều AWS database services
4. Bắt đầu ứng tuyển vào vai trò liên quan đến AWS

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng thực hành:** Đừng chỉ đọc — tạo thật sự trên AWS Console, làm hỏng, rồi fix
2. **So sánh liên tục:** Sau mỗi dịch vụ mới, so sánh với dịch vụ đã biết
3. **Hiểu WHY, không chỉ WHAT:** Tại sao Aurora nhanh hơn RDS? Tại sao DynamoDB có millisecond latency?
4. **Chia sẻ kiến thức:** Giảng lại cho người khác giúp consolidate learning
5. **Theo dõi AWS updates:** AWS release tính năng mới thường xuyên — đọc AWS Blog hàng tuần
6. **Biết cost trước khi dùng:** Luôn ước tính chi phí trước khi tạo tài nguyên
7. **Test backups:** Restore test là điều bắt buộc — backup không được test là backup chưa tồn tại
8. **Ghi lại incidents:** Mọi sự cố đều là cơ hội học tập

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Chào đón đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Hướng dẫn dịch vụ AWS mới

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.1
**Trạng Thái:** ✅ README & INDEX Hoàn Thành | ✅ 01-rds-fundamentals Hoàn Thành | 🚧 Các Section Khác Đang Xây Dựng
