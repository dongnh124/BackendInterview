# Tối Ưu Hiệu Suất CSDL

Nắm vững tối ưu truy vấn, index và điều chỉnh CSDL cho hệ thống production.

## Các Chủ Đề Cốt Lõi

1. **Phân Tích Truy Vấn** — EXPLAIN ANALYZE
2. **Thiết Kế Index** — Chọn index đúng một cách chiến lược
3. **Thống Kê** — Giữ query planner được thông báo
4. **Autovacuum** — Tự động hóa bảo trì
5. **Quản Lý Khóa** — Ngăn tranh chấp
6. **Connection Pooling** — Sử dụng tài nguyên hiệu quả
7. **Phát Hiện Truy Vấn Chậm** — Tìm vấn đề
8. **Chiến Lược Caching** — Tối ưu tầng ứng dụng

---

## Quy Trình Phân Tích Truy Vấn

### Bước 1: Xác Định Truy Vấn Chậm

```sql
-- PostgreSQL: pg_stat_statements
CREATE EXTENSION pg_stat_statements;

SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Bước 2: Lấy Kế Hoạch Truy Vấn

```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
AND o.created_at > '2026-01-01';
```

### Bước 3: Giải Thích Kế Hoạch

```
Seq Scan on orders   ← XẤU (quét toàn bảng)
├─ Filter: (status = 'pending') AND (created_at > ...)
└─ Rows: 50000

so với

Index Scan using idx_orders_status_date on orders  ← TỐT
├─ Index Cond: (status = 'pending') AND (created_at > ...)
└─ Rows: 150
```

**Các chỉ số cần tìm:**

- **Seq Scan** → Tìm index bị thiếu
- **Filter** → Có thể làm điều kiện index không?
- **Rows** → Ước tính vs thực tế (chênh lệch lớn = stats cũ)
- **Buffers** → Hits vs reads (hits cao = hiệu quả)

### Bước 4: Tối Ưu

```sql
-- Tùy chọn 1: Thêm index
CREATE INDEX idx_orders_status_date
ON orders(status, created_at);

-- Tùy chọn 2: Cải thiện join
-- Có thể bảng users nên được pre-join
-- Hoặc dùng denormalization

-- Tùy chọn 3: Viết lại truy vấn
-- Đôi khi logic truy vấn có thể đơn giản hóa
```

---

## Nguyên Tắc Thiết Kế Index

### Thứ Tự Composite Index

```sql
-- Sai: Sai thứ tự
CREATE INDEX idx_bad ON orders(created_at, status);

-- Truy vấn: WHERE status = 'pending' AND created_at > '2026-01-01'
-- Không thể dùng index hiệu quả

-- Đúng: Bằng trước, phạm vi sau
CREATE INDEX idx_good ON orders(status, created_at);

-- Bây giờ truy vấn dùng index hiệu quả:
-- 1. Tìm status = 'pending' (phạm vi index)
-- 2. Trong phạm vi đó, tìm created_at > '2026-01-01'
```

### Covering Index

```sql
-- Truy vấn chỉ cần 3 cột
SELECT id, status, total FROM orders WHERE user_id = 1;

-- Không có covering index:
-- 1. Quét index trên user_id
-- 2. Tra cứu bảng cho mỗi hàng (status, total)

-- Với covering index (bao gồm cột cần):
CREATE INDEX idx_covering ON orders(user_id)
INCLUDE (status, total);  -- PostgreSQL 11+

-- Lợi ích: Index-only scan, không tra cứu bảng
```

### Partial Index

```sql
-- Filter phổ biến: người dùng active
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- Chỉ index người dùng active (nhỏ hơn, nhanh hơn)
-- Truy vấn: SELECT * FROM users WHERE email = ? AND status = 'active'
-- Dùng index ✓

-- Truy vấn: SELECT * FROM users WHERE email = ? AND status = 'deleted'
-- Không dùng index (predicate không khớp) ✗
```

---

## Bảo Trì

### Điều Chỉnh Autovacuum

```sql
-- Tại sao vacuum quan trọng:
-- PostgreSQL dùng MVCC (nhiều phiên bản)
-- Phiên bản cũ tích lũy (bloat)
-- VACUUM xóa hàng chết

-- Kiểm tra cài đặt hiện tại:
SHOW autovacuum_vacuum_scale_factor;  -- Mặc định: 0.2 (thay đổi 20%)
SHOW autovacuum_analyze_scale_factor; -- Mặc định: 0.1 (thay đổi 10%)

-- Cho bảng lớn, cập nhật nhiều:
ALTER TABLE hot_table SET (
  autovacuum_vacuum_scale_factor = 0.05,  -- Vacuum khi 5% thay đổi
  autovacuum_vacuum_cost_limit = 1000     -- Tích cực hơn
);

-- Kiểm tra bloat:
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Cập Nhật Thống Kê

```sql
-- Query planner cần thống kê chính xác
-- Để ước tính hàng chính xác

-- Cập nhật thủ công:
ANALYZE;  -- Cập nhật stats tất cả bảng

-- Cho bảng cụ thể:
ANALYZE hot_table;

-- Kiểm tra tuổi stats:
SELECT schemaname, tablename, last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables
ORDER BY last_analyze ASC;
```

---

## Quản Lý Khóa

### Xác Định Truy Vấn Chạy Lâu

```sql
-- PostgreSQL: Tìm truy vấn đang chặn
SELECT pid, usename, state, wait_event_type, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY backend_start ASC;

-- Giết truy vấn có vấn đề:
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE pid = 12345;
```

### Phát Hiện Deadlock

```sql
-- Log deadlocks:
SET log_min_duration_statement = 1000;  -- Log truy vấn chậm
-- Tìm thông báo deadlock trong log

-- Giảm cơ hội deadlock:
-- 1. Truy cập bảng theo cùng thứ tự
-- 2. Giữ transaction ngắn
-- 3. Dùng index phù hợp (giảm lock footprint)
-- 4. Giảm isolation level nếu chấp nhận được
```

---

## Kích Thước Connection Pool

### Công Thức

```
connections = ((số_nhân * 2) + số_đĩa_hiệu_dụng)

Ví dụ: CPU 4 nhân, 1 đĩa = (4 * 2) + 1 = 9 kết nối

Quy tắc: Bắt đầu nhỏ (5-10), tăng dựa trên giám sát
```

### Cấu Hình PgBouncer

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction         -- Một kết nối mỗi transaction
max_client_conn = 1000          -- Max clients
default_pool_size = 25          -- Kết nối đến backend
min_pool_size = 5               -- Tối thiểu kết nối
```

---

## Các Chỉ Số Dashboard Hiệu Suất

| Chỉ số                | Truy vấn                            | Mục tiêu | Cảnh báo |
| --------------------- | ----------------------------------- | -------- | -------- |
| Truy vấn chậm (> 1s)  | pg_stat_statements                  | < 5      | > 10     |
| Sequential scans      | pg_stat_user_tables                 | < 100    | > 1000   |
| Index bloat           | pg_stat_user_indexes                | < 50%    | > 70%    |
| Autovacuum lag        | pg_stat_user_tables                 | < 1 ngày | > 3 ngày |
| Sử dụng connection pool | current_setting('max_connections') | < 80%   | > 90%    |
| Replication lag       | pg_last_wal_receive_lsn()           | < 1 giây | > 5 giây |

---

## Các Tình Huống Tối Ưu Phổ Biến

### Vấn Đề N+1 Query

```sql
-- Xấu: 1 + N truy vấn
SELECT * FROM orders;
-- Cho mỗi order: SELECT * FROM users WHERE id = ?

-- Tốt: Join đơn
SELECT o.*, u.name FROM orders o
JOIN users u ON o.user_id = u.id;
```

### Tránh Hàm Trên Cột Có Index

```sql
-- Xấu: Không thể dùng index
SELECT * FROM users WHERE LOWER(email) = 'test@email.com';

-- Tốt: Dùng functional index
CREATE INDEX idx_email_lower ON users(LOWER(email));

-- Hoặc tốt hơn: Chuẩn hóa dữ liệu lúc chèn
SELECT * FROM users WHERE email = 'test@email.com';
```

### Phân Trang Ở Quy Mô Lớn

```sql
-- Chậm: Offset lớn
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 10000;

-- Nhanh: Keyset pagination
SELECT * FROM orders WHERE id > last_id ORDER BY id LIMIT 10;
```

---

## Checklist Tối Ưu

- [ ] Truy vấn chạy nhiều nhất được phân tích với EXPLAIN
- [ ] Index phù hợp được tạo (không có index không dùng)
- [ ] Thống kê cập nhật (ANALYZE hoàn thành)
- [ ] Autovacuum được điều chỉnh cho workload
- [ ] Connection pool đúng kích thước
- [ ] Truy vấn chạy lâu được xác định và tối ưu
- [ ] Không có pattern N+1 query
- [ ] Replication lag được giám sát
- [ ] Slow query log được bật và xem xét
- [ ] Baseline hiệu suất được tài liệu hóa

---

## Chủ Đề Nâng Cao

- [ ] Query plan cache và prepared statements
- [ ] Bloom filters và JIT compilation
- [ ] Chiến lược partitioning
- [ ] Materialized views cho báo cáo phức tạp
- [ ] Thống kê cấp cột
- [ ] Điều chỉnh cost-based optimizer
- [ ] Thực thi truy vấn song song

---

> **Điểm Mấu Chốt:** Index tốt nhất là index ngăn ngừa full table scan cho các truy vấn chạy nhiều nhất. Luôn đo trước và sau khi tối ưu để đảm bảo cải thiện là thực sự.
