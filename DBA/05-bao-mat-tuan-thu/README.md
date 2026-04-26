# Bảo Mật & Tuân Thủ CSDL

Bảo vệ dữ liệu với bảo mật nhiều tầng và đáp ứng yêu cầu quy định.

## Các Chủ Đề Cốt Lõi

1. **Kiểm Soát Truy Cập** — Quyền tối thiểu, RBAC
2. **Mã Hóa** — Lưu trữ và truyền tải
3. **Ghi Nhật Ký Kiểm Tra** — Tuân thủ và điều tra
4. **Bảo Mật Mạng** — Firewall, VPN, TLS
5. **Tuân Thủ GDPR** — Xử lý và xóa PII
6. **PCI-DSS** — Bảo mật dữ liệu thanh toán
7. **Quản Lý Bí Mật** — Xoay vòng key, vault

---

## Các Tầng Bảo Mật

```
Tầng 5: KIỂM TOÁN & GIÁM SÁT
  - Thay đổi DDL được ghi log
  - Kiểm toán truy vấn trên bảng nhạy cảm
  - Tích hợp SIEM

Tầng 4: MÃ HÓA
  - Lưu trữ (AES-256)
  - Truyền tải (TLS 1.2+)
  - Quản lý key (KMS/HSM)

Tầng 3: PHÂN QUYỀN
  - Vai trò quyền tối thiểu
  - Phân tách schema
  - Row-level security (RLS)

Tầng 2: XÁC THỰC
  - Không mật khẩu dùng chung
  - Tích hợp IAM/AD
  - MFA cho truy cập DBA
  - Service account theo ứng dụng

Tầng 1: MẠNG
  - Mạng con riêng tư
  - Security groups
  - Quy tắc firewall
```

---

## Triển Khai

### Bảo Mật Mạng

```sql
-- Cấu hình mạng PostgreSQL
-- postgresql.conf:
listen_addresses = '10.0.1.0/24'  -- Chỉ mạng riêng

-- pg_hba.conf:
# Chỉ cho phép app servers
host    mydb    app_user    10.0.1.0/24    md5

# Truy cập DBA qua jump host
host    mydb    dba_user    10.0.2.50/32   md5
```

### Thiết Lập Xác Thực

```sql
-- Tạo role service (không phải superuser)
CREATE ROLE myapp_reader WITH LOGIN PASSWORD 'strong_password';
CREATE ROLE myapp_writer WITH LOGIN PASSWORD 'strong_password';

-- Cấp quyền tối thiểu
GRANT CONNECT ON DATABASE mydb TO myapp_reader;
GRANT USAGE ON SCHEMA public TO myapp_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO myapp_reader;

-- Writer role nhận INSERT/UPDATE/DELETE
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp_writer;

-- Không bao giờ dùng superuser trong ứng dụng!
```

### Thiết Lập Mã Hóa

```sql
-- Kiểm tra mã hóa lưu trữ có được bật không
SHOW ssl;  -- Phải là 'on'
SHOW ssl_cert_file;

-- PostgreSQL: extension pgcrypto
CREATE EXTENSION pgcrypto;

-- Mã hóa cột nhạy cảm
ALTER TABLE users ADD COLUMN ssn_encrypted bytea;

-- Mã hóa dữ liệu
UPDATE users SET ssn_encrypted = pgp_sym_encrypt(ssn, 'encryption_key');

-- Truy vấn dữ liệu mã hóa (giải mã khi truy xuất)
SELECT pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM users WHERE id = 123;
```

### Ghi Nhật Ký Kiểm Tra

```sql
-- PostgreSQL: extension pgaudit
CREATE EXTENSION pgaudit;

-- Log tất cả thay đổi DDL
ALTER SYSTEM SET pgaudit.log = 'DDL';

-- Log SELECT trên bảng nhạy cảm
ALTER SYSTEM SET pgaudit.log_statement = 'all';
ALTER SYSTEM SET pgaudit.role = 'audit_role';

-- Reload cấu hình
SELECT pg_reload_conf();

-- Xem audit logs
tail -f /var/log/postgresql/postgresql.log | grep AUDIT
```

---

## Khung Tuân Thủ

### GDPR (Quy Định Bảo Vệ Dữ Liệu Chung)

**Yêu cầu chính:**

```
1. Kiểm kê Dữ liệu
   - Biết PII nào bạn có
   - Lưu ở đâu
   - Ai có quyền truy cập

2. Quyền Xóa (Right to Delete)
   - Xóa PII khách hàng theo yêu cầu
   - Chứng minh đã xóa
   - Tính đến backup/replica

3. Mã Hóa
   - Dữ liệu nhạy cảm được mã hóa
   - Trong quá trình truyền (TLS) và lưu trữ

4. Kiểm Soát Truy Cập
   - Chỉ người cần thiết truy cập PII
   - Thỏa thuận vendor (data processors)

5. Thông Báo Vi Phạm
   - Thông báo trong 72 giờ nếu vi phạm
   - Tài liệu hóa điều tra vi phạm
```

**Triển Khai:**

```sql
-- Gắn tag bảng nhạy cảm
COMMENT ON TABLE users IS 'GDPR: Chứa PII';
COMMENT ON COLUMN users.ssn IS 'GDPR: Nhạy cảm, mã hóa';

-- Chính sách lưu giữ dữ liệu
DELETE FROM deleted_users WHERE deleted_at < NOW() - INTERVAL '90 days';

-- Lưu giữ backup để tuân thủ (giữ 7 năm)
-- = 365 * 7 = 2555 ngày lưu giữ tối thiểu
```

### PCI-DSS (Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ)

**Yêu cầu chính:**

```
1. Cấu hình firewall
   - Hạn chế truy cập dữ liệu cardholder
   - VPN cho truy cập từ xa

2. Không hardcode mật khẩu
   - Xoay vòng thường xuyên
   - Lưu trong vault

3. Mã hóa dữ liệu cardholder
   - Lưu trữ và truyền tải
   - Tokenize khi có thể

4. Phát hiện thay đổi
   - Cảnh báo thay đổi không phép
   - Kiểm toán tất cả thay đổi CSDL

5. Kiểm tra
   - Kiểm tra thâm nhập hàng năm
   - Quét lỗ hổng hàng quý

6. Kiểm soát truy cập
   - ID user duy nhất
   - Hạn chế theo nhu cầu công việc

7. Logs và giám sát
   - Tất cả truy cập được log
   - Logs được bảo vệ khỏi bị xóa
```

**Triển Khai:**

```sql
-- Không bao giờ lưu dữ liệu thẻ đầy đủ!
-- Dùng tokenization thay thế

-- Sai:
CREATE TABLE payments (
    id SERIAL,
    card_number VARCHAR(16),  -- Vi phạm PCI-DSS!
    amount DECIMAL(10,2)
);

-- Đúng:
CREATE TABLE payments (
    id SERIAL,
    card_token VARCHAR(32),   -- Tokenized, không phải thẻ thực
    amount DECIMAL(10,2)
);

-- Lưu thẻ thực chỉ tại payment processor
```

---

## Quản Lý Bí Mật

### Thiết Lập Vault (HashiCorp Vault)

```bash
# Lưu thông tin đăng nhập CSDL an toàn
vault kv put secret/databases/mydb \
  username=myapp_user \
  password=GeneratedStrongPassword123!

# Xoay vòng thông tin đăng nhập
vault read -field=password secret/databases/mydb

# Ứng dụng lấy bí mật lúc runtime
# (Không lưu trong config files!)
```

### Lịch Xoay Vòng Key

```
Thông tin đăng nhập CSDL: Mỗi 90 ngày
Chứng chỉ TLS: Mỗi 365 ngày
Khóa mã hóa: Mỗi năm
Khóa mã hóa backup: Mỗi 2 năm
```

---

## Các Lỗi Bảo Mật Thường Gặp

```
❌ Mật khẩu dùng chung (nhiều người, nhiều hệ thống)
❌ Superuser cho kết nối ứng dụng
❌ Kết nối không mã hóa qua mạng
❌ PII trong môi trường non-production
❌ Không có audit logging
❌ Bảo mật backup yếu
❌ Kết nối CSDL trực tiếp từ app servers (dùng proxy)
❌ Cùng mật khẩu cho dev/staging/prod
❌ Không có chính sách xoay vòng bí mật
❌ Lưu mật khẩu trong code/config

✓ Thông tin đăng nhập duy nhất cho mỗi ứng dụng
✓ Vai trò quyền tối thiểu
✓ TLS ở mọi nơi
✓ Che giấu dữ liệu trong non-prod
✓ Audit logs toàn diện
✓ Backup mã hóa trong tài khoản riêng
✓ Connection pooler/proxy
✓ Bí mật đặc thù theo môi trường
✓ Xoay vòng bí mật tự động
✓ Thông tin đăng nhập được quản lý bởi Vault
```

---

## Checklist Kiểm Tra Bảo Mật

**Mạng:**

- [ ] CSDL trong mạng con riêng tư
- [ ] Quy tắc firewall chỉ whitelist IP đã biết
- [ ] VPN bắt buộc cho truy cập DBA
- [ ] TLS 1.2+ cho tất cả kết nối
- [ ] Không có IP công khai trên CSDL

**Kiểm Soát Truy Cập:**

- [ ] Không mật khẩu dùng chung
- [ ] Service account theo ứng dụng
- [ ] Quyền tối thiểu theo role
- [ ] Superuser bị vô hiệu hóa cho ứng dụng
- [ ] MFA cho truy cập DBA console
- [ ] Xem xét truy cập hàng quý

**Mã Hóa:**

- [ ] Mã hóa lưu trữ được bật
- [ ] Khóa mã hóa trong KMS/HSM
- [ ] Chính sách xoay vòng key được triển khai
- [ ] Mã hóa truyền tải (TLS)
- [ ] Cột nhạy cảm được mã hóa

**Kiểm Toán & Logging:**

- [ ] Thay đổi DDL được log
- [ ] Lần đăng nhập thất bại được log
- [ ] Kiểm toán truy vấn trên bảng nhạy cảm
- [ ] Logs được gửi đến SIEM
- [ ] Chính sách lưu giữ log được đặt
- [ ] Logs được bảo vệ khỏi giả mạo

**Tuân Thủ:**

- [ ] Kiểm kê dữ liệu được tài liệu hóa
- [ ] Chính sách riêng tư phù hợp với thực tiễn dữ liệu
- [ ] Thỏa thuận vendor được thiết lập
- [ ] Kế hoạch ứng phó sự cố vi phạm
- [ ] Đánh giá bảo mật thường xuyên
- [ ] Kiểm tra thâm nhập hoàn thành

---

## Câu Hỏi Phỏng Vấn

1. **Thiết kế CSDL an toàn cho ứng dụng fintech xử lý dữ liệu thanh toán**
   - Tokenization (không lưu số thẻ đầy đủ)
   - Tuân thủ PCI-DSS (mã hóa, kiểm toán, kiểm soát truy cập)
   - TLS cho tất cả kết nối
   - Vault riêng cho bí mật
   - Kiểm tra thâm nhập hàng quý
   - Audit logs bất biến

2. **Xử lý yêu cầu xóa người dùng (GDPR)?**
   - Tìm tất cả dữ liệu của người dùng đó
   - Tính đến backup
   - Xóa từ primary và replica
   - Xác minh xóa bằng checksum
   - Tài liệu hóa xóa để tuân thủ
   - Lưu ý: Một số backup có thể giữ dữ liệu (legal hold)

3. **Phương pháp quản lý bí mật của bạn?**
   - Không hardcode thông tin đăng nhập
   - Bí mật được quản lý bởi Vault
   - Xoay vòng tự động (90 ngày)
   - Bí mật khác nhau theo môi trường
   - TTL giới hạn cho thông tin đăng nhập
   - Log truy cập vào vault

---

> **Điểm Mấu Chốt:** Bảo mật không phải là tính năng thêm vào cuối — nó được xây dựng từ đầu. Làm cho dễ dàng để làm đúng (API tốt, tài liệu rõ ràng), và khó để làm sai (không có thông tin đăng nhập trong code, không có truy cập trực tiếp).
