# Bố Cục Triển Khai

Baseline cho một node MySQL 8.0 hoặc 8.4 tự cài trên Linux. Cấu hình replication chi tiết và cách đo lag nằm ở [03-ha-replication](../03-ha-replication/README.md). Phần này chỉ chốt những gì mọi node trong topology phải giống nhau.

## Gói Cài

Dùng một dòng sản phẩm cho cả cụm.

| Gói | Dùng khi |
|-----|----------|
| MySQL Community 8.0 hoặc 8.4 | Topology Oracle: primary–replica, Group Replication, InnoDB Cluster |
| Percona Server | Cùng protocol với Community, thêm công cụ Percona |
| Percona XtraDB Cluster hoặc MariaDB Galera | Chỉ khi đã chọn Galera ở [Topology](1-topology.md) |

Không trộn binary Oracle MySQL với MariaDB trong một chuỗi replication. GTID và binlog không hứa tương thích.

**Banking.** Production của ledger chạy bản có support contract (MySQL Enterprise hoặc Percona có support), cùng một patch trên mọi node. Community không có kênh gọi khi đang giữa sự cố thì không dùng cho hệ thống ghi balance. Galera chỉ khi mục [Topology](1-topology.md) đã chọn, và vẫn một node được ghi.

## Thư Mục và Cổng

```
/etc/mysql/mysql.conf.d/mysqld.cnf   cấu hình đọc lúc khởi động
/var/lib/mysql                       datadir
/var/log/mysql/error.log             lỗi server
/var/log/mysql/mysql-bin.*           binary log
/run/mysqld/mysqld.sock              client trên cùng máy

3306    cổng SQL
33060   X Protocol (mysqlx). Tắt nếu không dùng MySQL Shell qua X
33061   group communication của Group Replication, chỉ mở giữa các node trong group
```

Đường dẫn gói distro có thể là `/etc/my.cnf`. Một file `!includedir` kéo thêm thư mục con. Sửa đúng file đang được nạp — `mysqld --verbose --help` in ra thứ tự file nó đọc.

Quyền datadir thuộc user `mysql`. Không đặt datadir trên đĩa dùng chung rồi cho hai mysqld mở cùng lúc.

## `my.cnf` Baseline

Mỗi node một `server_id` khác nhau. Phần còn lại giống nhau để failover không đổi hành vi.

```ini
[mysqld]
server_id = 1
datadir = /var/lib/mysql
bind-address = 10.0.0.11

# Replication. Bật trước khi có dữ liệu production.
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
sync_binlog = 1

# Replica không nhận ghi cho đến khi được promote.
# Primary đặt OFF. Xem 3-proxy-va-failover.md.
super_read_only = ON
read_only = ON

# Bên trong máy dedicated. Kết nối từ app đi qua proxy.
max_connections = 500
innodb_buffer_pool_size = 12G          # ví dụ máy dedicated ~16GB RAM
innodb_flush_log_at_trx_commit = 1
```

`sync_binlog = 1` và `innodb_flush_log_at_trx_commit = 1` để commit đã ACK không chỉ nằm trên disk cache. Tắt một trong hai thì RPO của semisync không còn đúng như sơ đồ.

`binlog_expire_logs_seconds = 604800` giữ binlog 7 ngày. Replica ngừng lâu hơn khoảng này thì không catch up bằng binlog, phải clone lại. Ledger thường cần investigation window và point-in-time dài hơn 7 ngày: tăng retention cho khớp chính sách nội bộ, và đưa binlog ra archive riêng cùng backup.

## User

Ba user, ba việc. Host giới hạn theo IP hoặc mạng con, không dùng `'%'` cho user replication.

```sql
-- Replica kéo binlog. Chỉ từ IP các replica.
CREATE USER 'repl'@'10.0.1.%' IDENTIFIED BY '...';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.1.%';

-- Ứng dụng. Không ALL PRIVILEGES, không SUPER.
CREATE USER 'app'@'10.0.2.%' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'10.0.2.%';

-- ProxySQL hoặc Orchestrator hỏi topology.
CREATE USER 'monitor'@'10.0.3.%' IDENTIFIED BY '...';
GRANT REPLICATION CLIENT, PROCESS ON *.* TO 'monitor'@'10.0.3.%';
```

Mật khẩu không nằm trong `my.cnf` của ứng dụng trong git. Cách giữ bí mật: [Quản Lý Bí Mật](../05-bao-mat-tuan-thu/quan-ly-bi-mat.md).

`REPLICATION SLAVE` là privilege của user mà replica dùng để đọc binlog. `REPLICATION_SLAVE_ADMIN` là privilege khác, dành cho lệnh quản trị như `CHANGE REPLICATION SOURCE TO` trên chính máy đó.

## TLS

Bật trên đường app → proxy và proxy → mysqld. Replication cũng đi qua TLS nếu binlog rời khỏi một VLAN khóa.

**Banking.** TLS bật trên cả đường replication, kể cả trong VLAN. `sync_binlog = 1` và `innodb_flush_log_at_trx_commit = 1` không được hạ để đổi lấy latency trên ledger. App không có `SUPER`. Mọi phiên quản trị ghi vào audit log: [Ghi Nhật Ký Kiểm Tra](../05-bao-mat-tuan-thu/ghi-nhat-ky-kiem-tra.md). Dữ liệu thẻ không lưu dạng đọc được trong schema này: [PCI-DSS](../05-bao-mat-tuan-thu/tuan-thu-pci-dss.md).

```ini
require_secure_transport = ON
ssl_ca   = /etc/mysql/certs/ca.pem
ssl_cert = /etc/mysql/certs/server-cert.pem
ssl_key  = /etc/mysql/certs/server-key.pem
```

`REQUIRE SSL` trên user `app` và `repl` để một client bỏ TLS không vào được dù `require_secure_transport` bị tắt nhầm lúc bảo trì.

## Mạng

```
App subnet ──3306/TCP──► proxy
proxy     ──3306/TCP──► mysqld
replica   ──3306/TCP──► primary          chỉ chiều replica đi lấy binlog
GR node   ──33061/TCP─► GR node          cả hai chiều, trong group
Không mở 3306 ra internet
```

Security group hoặc firewall viết theo IP node, không theo `0.0.0.0/0`.

## Sizing

Trên máy chỉ chạy mysqld:

```
innodb_buffer_pool_size ≈ 70% RAM
Phần còn lại: OS, connection, và file binlog đang ghi

Đĩa dữ liệu và binlog tách volume nếu ghi nặng
  → hết chỗ binlog không làm đầy đĩa datadir, và ngược lại

IOPS đủ cho flush mỗi commit (sync_binlog = 1)
  → đĩa chậm làm commit chậm trước khi replication kịp làm chậm
```

`max_connections` trên mysqld để thấp hơn tổng pool của mọi app cộng lại. App giữ pool ở ProxySQL hoặc ở process app, không mở hàng nghìn connection thẳng vào primary. Xem [Connection Pooling](../01-co-ban/connection-pooling.md).

## Checklist Một Node

- [ ] `server_id` duy nhất trong cụm
- [ ] GTID và `binlog_format = ROW` bật trước khi nhận dữ liệu
- [ ] Replica đang `super_read_only = ON`
- [ ] User `repl`, `app`, `monitor` tách privilege và tách dải IP
- [ ] `require_secure_transport = ON` nếu client không nằm cùng máy
- [ ] Cổng 3306 không ra internet
- [ ] Buffer pool khoảng 70% RAM trên máy dedicated
- [ ] Binlog giữ đủ lâu để replica chậm nhất còn catch up, và đủ investigation window của ledger
- [ ] TLS trên đường replication; flush log mỗi commit không bị tắt trên cụm ghi balance

---

> **Điểm Mấu Chốt:** Mọi node cùng một baseline, khác `server_id` và khác read_only flag. Cấu hình lệch giữa primary và replica biến lần failover thành lần đổi hành vi.
