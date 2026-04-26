# Di Chuyển Schema & Quản Lý Thay Đổi

Thay đổi schema an toàn, không thời gian chết cho CSDL production.

## Các Chủ Đề Cốt Lõi

1. **Công Cụ Migration** — Flyway, Liquibase, sqitch
2. **Mẫu Expand-Contract** — Thay đổi không thời gian chết
3. **Chỉ Tiến** — Triết lý không rollback
4. **Xác Thực** — Kiểm tra toàn vẹn dữ liệu
5. **Triển Khai** — Phối hợp với ứng dụng
6. **Chiến Lược Rollback** — Quy trình phục hồi
7. **Testing** — Xác thực staging
8. **Giao Tiếp** — Phối hợp nhóm

---

## Các Loại Migration Theo Rủi Ro

```
RỦI RO THẤP (Có thể làm bất kỳ lúc nào, không lo khóa):
✓ Thêm cột nullable với default
✓ Thêm index (có thể khóa ngắn, nhưng online trong CSDL hiện đại)
✓ Thêm stored procedure/function
✓ Thêm check constraint (cho hàng mới)

RỦI RO TRUNG BÌNH (Có thể làm nhưng cần phối hợp):
⚠ Đổi tên cột (với alias)
⚠ Đổi tên bảng (cần thay đổi ứng dụng)
⚠ Thêm cột NOT NULL (phải backfill trước)
⚠ Tăng kích thước cột (thường an toàn)

RỦI RO CAO (Cần chiến lược không thời gian chết):
❌ Xóa cột (thay đổi gây phá vỡ)
❌ Xóa bảng (thay đổi gây phá vỡ)
❌ Thay đổi kiểu cột (có thể mất dữ liệu)
❌ Thêm FOREIGN KEY vào bảng lớn
❌ Xóa dữ liệu hàng loạt
```

---

## Mẫu Expand-Contract

### Ví dụ: Thêm cột bắt buộc vào bảng users

**Vấn đề:**

```
Thêm cột "status" với ràng buộc NOT NULL
Nếu chỉ ADD COLUMN với NOT NULL, nó khóa bảng cho mọi người
```

**Giải Pháp: Cách Tiếp Cận Ba Giai Đoạn**

#### GIAI ĐOẠN 1: EXPAND (Triển khai)

```sql
-- 1. Thêm cột nullable
ALTER TABLE users ADD COLUMN status VARCHAR(50);

-- 2. Thêm default cho hàng tương lai
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';

-- 3. Tạo index nếu cần (có thể concurrent)
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Ứng dụng tiếp tục hoạt động (code cũ bỏ qua cột mới)
-- Migration script: < 1 giây, không chặn
```

**Triển khai:**
- Thay đổi schema: Dễ, không chặn
- Ứng dụng: Không cần thay đổi (tương thích ngược)

#### GIAI ĐOẠN 2: MIGRATE (Backfill dữ liệu)

```sql
-- 1. Backfill dữ liệu theo batch (tránh khóa toàn bảng)
BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 1 AND 10000;
COMMIT;

BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 10001 AND 20000;
COMMIT;
-- ... lặp lại cho tất cả hàng

-- 2. Giám sát hoàn thành
SELECT COUNT(*) FROM users WHERE status IS NULL;  -- Phải là 0

-- 3. Tùy chọn: Thêm check constraint (không khóa hàng hiện có)
ALTER TABLE users ADD CONSTRAINT users_status_not_null
  CHECK (status IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;
```

**Thời gian:**
- Backfill xảy ra dần dần (phút đến giờ)
- Có thể chạy trong giờ hành chính (batch tránh khóa)
- Ứng dụng vẫn hoạt động

#### GIAI ĐOẠN 3: CONTRACT (Dọn dẹp)

```sql
-- 1. Thêm ràng buộc NOT NULL (chỉ ảnh hưởng hàng mới)
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- 2. Xóa dữ liệu/index tạm thời nếu có
-- Ứng dụng được cập nhật để dùng cột mới

-- 3. Nếu cần rollback, tạo lại cột từ backup
```

**Kết quả:**
- Không có thời gian chết
- Rollback có thể ở mỗi giai đoạn
- Ứng dụng có thể được kiểm tra với cột mới

---

## Mẫu Runbook Migration

```sql
-- ============================================
-- Migration: YYYY-MM-DD-HH-MM Thêm cột status
-- Tác giả: [Tên]
-- Mức rủi ro: THẤP
-- Thời gian ước tính: 5 phút
-- ============================================

-- Đã kiểm tra trên: Môi trường staging ngày 2026-04-26
-- Rollback: Đơn giản (DROP COLUMN nếu cần)

BEGIN;

-- Giai đoạn 1: Expand
ALTER TABLE users ADD COLUMN status VARCHAR(50);
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';

-- Xác minh
SELECT * FROM users LIMIT 1;

COMMIT;

-- Giai đoạn 2: Migrate (transaction riêng biệt)
-- Có thể chạy bất đồng bộ
BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL;
COMMIT;

-- Giai đoạn 3: Contract (trong migration tương lai)
-- ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- Các bước xác minh:
-- 1. Kiểm tra row count trước và sau: SELECT COUNT(*) FROM users;
-- 2. Kiểm tra dữ liệu: SELECT DISTINCT status FROM users;
-- 3. Kiểm tra index: SELECT * FROM users WHERE status = 'active' LIMIT 1;
```

---

## Công Cụ Migration

### Flyway

```bash
# Khởi tạo
flyway init

# Tạo migration
# File: V1__Create_users_table.sql

# Migrate
flyway migrate

# Thông tin
flyway info
```

**Cấu trúc file migration:**

```
db/migration/
├── V1__Tao_schema_ban_dau.sql
├── V2__Them_bang_users.sql
├── V3__Them_index_tren_email.sql
├── V4__Mo_rong_cot_status.sql  ← Migration của chúng ta
└── V5__Them_cot_metadata.sql
```

### Liquibase

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog>
    <changeSet id="1" author="john">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="name" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>

    <changeSet id="2" author="john">
        <addColumn tableName="users">
            <column name="status" type="VARCHAR(50)" defaultValue="active"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
```

---

## Triết Lý Chỉ Tiến (Forward-Only)

```
Cách tiếp cận truyền thống (thân thiện rollback):
- Có thể hoàn tác bất kỳ thay đổi nào
- Công cụ phức tạp (versioning lên và xuống)
- Rủi ro (rollback có thể giới thiệu bug mới)

Cách tiếp cận chỉ tiến:
- Thay đổi chỉ đi về phía trước
- Versioning đơn giản hơn (không bao giờ hoàn tác)
- An toàn hơn (deploy fix tiến, không lùi)
```

**Tại sao chỉ tiến?**

```
Tình huống: Migration xấu được deploy, phải rollback
- Truyền thống: Rollback schema, ứng dụng vẫn kỳ vọng schema mới → Hỗn loạn
- Chỉ tiến: Sửa schema với migration mới (thay đổi có kiểm soát an toàn hơn)

Tình huống: Migration mất dữ liệu, muốn hoàn tác
- Backup vẫn có sẵn (độc lập với versioning schema)
- Khôi phục từ backup nếu cần phục hồi dữ liệu
- Versioning schema là mối quan tâm riêng
```

---

## Kiểm Tra Quy Trình Migration

### Checklist Trước Migration

```
48 Giờ Trước:
☐ Migration được kiểm tra trong staging (với dữ liệu production-like)
☐ Kế hoạch rollback được viết và kiểm tra
☐ Tác động hiệu suất được đánh giá
☐ Kế hoạch giao tiếp được tạo
☐ Kỹ sư on-call được xác định

4 Giờ Trước:
☐ Backup CSDL hoàn thành
☐ Migration script được peer review
☐ Scripts rollback sẵn sàng
☐ Thành viên nhóm ở vị trí (app eng, DBA, oncall)

Trong Migration:
☐ Giám sát tranh chấp khóa
☐ Giám sát tỷ lệ lỗi ứng dụng
☐ Rollback sẵn sàng thực hiện
```

### Xác Thực Sau Migration

```sql
-- Kiểm tra toàn vẹn dữ liệu:
SELECT COUNT(*) AS tong_users FROM users;
-- So với trước: phải khớp

SELECT COUNT(DISTINCT status) FROM users;
-- Phải có các status kỳ vọng

SELECT * FROM users WHERE status IS NULL;
-- Phải rỗng (nếu constraint đã thêm)

-- Xác minh index:
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';
-- Phải dùng index, không phải seq scan
```

---

## Các Mẫu Phổ Biến

### Đổi Tên Cột (Không Thời Gian Chết)

```sql
-- Giai đoạn 1: Thêm cột mới với cùng dữ liệu
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);
UPDATE users SET email_new = email;
CREATE INDEX idx_email_new ON users(email_new);

-- Giai đoạn 2: Dual-write (ứng dụng ghi cả hai cột)
-- Code ứng dụng:
UPDATE users SET email = ?, email_new = ? WHERE id = ?;

-- Giai đoạn 3: Chuyển đọc sang cột mới
-- Code ứng dụng:
SELECT email_new AS email FROM users WHERE id = ?;

-- Giai đoạn 4: Xóa cột cũ
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

### Thêm Foreign Key vào Bảng Lớn

```sql
-- Sai: Add trực tiếp khóa bảng
-- ALTER TABLE orders ADD CONSTRAINT fk_user
-- FOREIGN KEY (user_id) REFERENCES users(id);

-- Đúng: NOT VALID, sau đó validate
ALTER TABLE orders ADD CONSTRAINT fk_user
FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Validate riêng (có thể dùng concurrent index)
ALTER TABLE orders VALIDATE CONSTRAINT fk_user;
```

---

## Câu Hỏi Phỏng Vấn

1. **Làm thế nào thêm cột NOT NULL vào bảng 100M hàng mà không có thời gian chết?**
   - Dùng mẫu expand-contract
   - Thêm cột nullable trước
   - Backfill theo batch trong giờ hành chính
   - Cuối cùng thêm constraint

2. **Migration thất bại giữa chừng. Bạn làm gì?**
   - Đánh giá blast radius (hàng nào bị ảnh hưởng?)
   - Quyết định: Tiếp tục tiến hay khôi phục backup
   - Nếu tiếp tục: Xác định điều gì thất bại, sửa trong migration mới
   - Tài liệu hóa nguyên nhân gốc và biện pháp phòng ngừa

3. **Thiết kế chiến lược migration cho đổi tên bảng trong production**
   - Tạo bảng mới với tên mới (rỗng)
   - Thêm trigger để sao chép ghi vào cả hai
   - Backfill dữ liệu hiện có
   - Chuyển ứng dụng đọc từ mới
   - Xóa bảng cũ

---

## Checklist

- [ ] Công cụ migration thiết lập (Flyway/Liquibase)
- [ ] Versioning migration trong source control
- [ ] Kiểm tra trên staging trước
- [ ] Peer review các thay đổi SQL
- [ ] Quy trình rollback được tài liệu hóa
- [ ] Các bước xác thực dữ liệu được viết
- [ ] Cách tiếp cận tương thích ngược được dùng
- [ ] Kế hoạch giao tiếp nhóm
- [ ] Giám sát được bật trong migration
- [ ] Quy trình post-mortem cho bất kỳ vấn đề nào

---

> **Điểm Mấu Chốt:** Migration tốt nhất là migration bạn không bao giờ cần rollback. Thiết kế tương thích ngược, kiểm tra kỹ trong staging, và luôn có cách tiến về phía trước ngay cả khi có vấn đề.
