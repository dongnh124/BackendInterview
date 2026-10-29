# Kiến Trúc Triển Khai MySQL

Cách đặt các tiến trình MySQL, proxy và failover thành một hệ thống chạy được. Lý thuyết replication, backup và dịch vụ managed nằm ở folder khác — file ở đây chỉ nói MySQL tự quản lý được lắp với nhau như thế nào.

## Các Tài Liệu

Đọc theo số thứ tự.

| Tài liệu | Mô tả |
|----------|-------|
| [1. Topology](1-topology.md) | Standalone, primary–replica, Group Replication, InnoDB Cluster, Galera |
| [2. Bố cục triển khai](2-bo-cuc-trien-khai.md) | Gói cài, `my.cnf`, cổng, user, TLS, sizing |
| [3. Proxy và failover](3-proxy-va-failover.md) | ProxySQL, MySQL Router, Orchestrator, split-brain |
| [4. Kiến trúc HA](4-kien-truc-ha.md) | Mẫu self-managed cho uptime 99.99% |
| [5. Nâng cấp](5-nang-cap.md) | Rolling upgrade, đổi cấu hình, kiểm tra trước khi promote |
| [6. Apply migration](6-apply-migration.md) | DDL trên standalone, primary–replica, Group Replication, Galera |

## Chọn Topology

```
Chỉ một máy, chấp nhận downtime khi sửa?
  └─ Có → Standalone

Cần đọc scale, ghi vẫn qua một primary?
  ├─ Mất vài giây dữ liệu khi primary chết là chấp nhận được
  │    └─ Primary–replica async + GTID
  └─ Commit đã ACK với client phải còn trên ít nhất một replica
       └─ Primary–replica semisync + GTID

Cần cụm tự elect primary, không muốn orchestrator bên ngoài?
  └─ Group Replication (single-primary) hoặc InnoDB Cluster
     (Group Replication + MySQL Shell + MySQL Router)

Cần ghi nhiều node, certification trên cả cụm?
  └─ Galera (Percona XtraDB Cluster hoặc MariaDB)
     Không phải sản phẩm Oracle MySQL
```

InnoDB Cluster là cách đóng gói Group Replication để vận hành, không phải một replication protocol khác.

## Liên Kết

- Replication, semisync, lag, failover nói chung: [03-ha-replication](../03-ha-replication/README.md)
- `mysqldump`, Percona XtraBackup: [02-sao-luu-phuc-hoi](../02-sao-luu-phuc-hoi/README.md)
- Aurora MySQL, Azure Database for MySQL: [07-cloud-database](../07-cloud-database/README.md)

## Nguyên Tắc

1. **GTID bật từ đầu** trên mọi topology có replica. Ghép binlog bằng file/position chỉ dùng khi đang di chuyển cụm cũ.
2. **Replica không được ghi.** `super_read_only = ON` cho đến lúc được promote.
3. **Proxy không phải HA.** ProxySQL và MySQL Router chỉ chuyển kết nối. Việc elect primary là việc của Orchestrator hoặc Group Replication.
4. **Backup đứng riêng.** Replica chết cùng primary không thay được backup.
5. **Ba node cho quorum.** Hai node không có majority khi mạng bị cắt.

## Ưu Tiên Trong Môi Trường Banking

Ledger và payment chọn mất khả năng ghi hơn là trả OK cho client rồi mất transaction. Reporting và kênh chỉ đọc thì được nới.

| Việc | Ưu tiên | Tránh |
|------|---------|--------|
| Ghi ledger, balance, posting | Semisync với timeout dài, hoặc Group Replication single-primary | Async; multi-primary (Galera ghi nhiều node, GR multi-primary) |
| Failover ledger | Semi-automatic: công cụ phát hiện, on-call xác nhận rồi mới promote | Auto-promote khi chưa có fencing game day |
| Failover reporting, notification | Automatic, vì sai sót không đổi balance | — |
| Đọc balance vừa ghi | Primary | Replica đang lag |
| Máy production | Bản có support contract, cấu hình giống nhau | Community không có nơi gọi khi sự cố, hoặc lệch `my.cnf` giữa các node |

Chi tiết nằm ở từng file, mục cùng tên.

---

> **Điểm Mấu Chốt:** Triển khai MySQL là chọn một topology, rồi đặt proxy, failover và backup quanh nó. Đừng trộn Group Replication với Orchestrator trên cùng một cụm — hai cơ chế cùng promote primary sẽ đánh nhau.
