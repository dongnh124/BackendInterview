# Kiến Trúc HA Cho MySQL

Mẫu self-managed khi mục tiêu uptime là 99.99%. Bản PostgreSQL cùng mức uptime, dùng Patroni, nằm trong [Hướng Dẫn Phỏng Vấn](../11-phong-van/HUONG_DAN_PHONG_VAN.md). Đây là bản MySQL tương ứng: primary–replica, semisync, proxy, quorum failover, backup độc lập.

## Ngân Sách Downtime

```
99.99% = 52,6 phút / năm
       ≈ 4,4 phút / tháng
```

Vài lần failover tự động, mỗi lần dưới 30 giây, còn nằm trong ngân sách. Một lần restore từ backup full thì không. Vì vậy failover phải có sẵn, và backup dùng cho hỏng dữ liệu chứ không dùng cho chết một máy.

99.99% là ngân sách của cả đường đi: app, proxy, mạng, cơ sở dữ liệu. Sơ đồ dưới đây chỉ khóa phần MySQL. App không có retry khi connection đứt thì failover 20 giây vẫn thành lỗi với người dùng.

## Sơ Đồ

```
                 [App]
                   │
              [VIP hoặc DNS]
               │           │
        [ProxySQL]     [ProxySQL]
               │           │
        ┌──────┴───────────┴──────┐
        │                          │
   [Primary]                  semisync, GTID
   AZ hoặc rack A                 │
        ├──────────────────► [Replica]   rack B
        └──────────────────► [Replica]   rack C

   [Orchestrator] [Orchestrator] [Orchestrator]
        Raft, 3 node, khác máy với mysqld

   [XtraBackup] ──► object storage
        lịch riêng, không phụ thuộc replica còn sống
```

Ba failure domain khác nhau cho ba mysqld. Hai replica cùng một tủ mạng thì mất tủ là mất luôn ứng viên failover.

## Thành Phần

**Dữ liệu.** Một primary, hai replica, GTID, `binlog_format = ROW`. Semisync chờ đúng một replica nhận event vào relay log (`wait_for_replica_count = 1`). Replica thứ hai vẫn async để còn chỗ đọc và còn một bản nữa nếu replica semisync chết.

Primary và replica đặt `sync_binlog = 1` và `innodb_flush_log_at_trx_commit = 1`. Baseline đầy đủ: [Bố Cục Triển Khai](2-bo-cuc-trien-khai.md).

**Đường vào.** Hai ProxySQL sau một VIP. Ghi vào hostgroup của primary. Đọc scale ra replica khi `Seconds_Behind_Source` dưới ngưỡng đã chọn (thường 1–5 giây). Chi tiết luật: [Proxy và Failover](3-proxy-va-failover.md).

**Quyết định promote.** Ba Orchestrator, Raft. Mất primary thì majority Orchestrator mới được promote replica caught up nhất. Một Orchestrator cô lập không được làm việc đó.

**Backup.** XtraBackup hoặc `mysqldump` theo lịch, đẩy ra object storage ở tài khoản khác cụm. Cách làm và kiểm tra restore: [Chiến Lược Sao Lưu](../02-sao-luu-phuc-hoi/chien-luoc-sao-luu.md). Replica không tính là backup: `DROP TABLE` trên primary sẽ có mặt trên replica.

## RPO và RTO Của Mẫu Này

| Sự cố | RPO | RTO mục tiêu |
|-------|-----|----------------|
| Mất primary, semisync còn ACK | Transaction đã trả OK còn trên ít nhất một relay log | Dưới 30 giây, gồm apply nốt relay log |
| Semisync đã timeout, cụm đang async | Phần binlog chưa sang replica | Vẫn dưới 30 giây, nhưng có thể mất transaction |
| Xóa dữ liệu nhầm | Về thời điểm trước lệnh xóa, nếu binlog và backup còn | Hàng chục phút đến vài giờ, tùy kích thước |
| Mất cả 3 máy chủ | Điểm backup gần nhất | Thời gian dựng cụm mới |

Semisync timeout (mặc định 10 giây) làm hàng “RPO gần 0” không còn đúng. Cảnh báo khi `Rpl_semi_sync_source_status` chuyển sang OFF, và khi failover diễn ra trong lúc đó thì nói rõ có thể mất transaction.

**Banking.** Trên ledger, timeout để rất cao để commit chờ replica thay vì hết 10 giây rồi trả OK trong chế độ async. Cảnh báo semisync tắt là open incident, không phải thông tin phụ. Uptime 99.99% đứng sau yêu cầu không mất posting đã báo thành công: ngừng ghi khi không còn replica nhận binlog. Failover ledger theo kiểu semi-automatic ở [Proxy và Failover](3-proxy-va-failover.md) cho đến khi fencing game day đạt. Replica cùng thành phố không thay một bản sao ở site khác; site thứ hai là disaster recovery, backup độc lập vẫn phải có vì replication sang site khác vẫn mang theo lệnh xóa nhầm.

## Việc Không Nhét Vào Sơ Đồ Này

- Group Replication thay cả nhánh Orchestrator. Đó là topology khác, xem [Topology](1-topology.md). Không gắn thêm Orchestrator cho chắc.
- Replica cùng availability zone với primary không tính là chịu mất zone.
- Đọc từ replica cho màn hình read-your-writes. Những câu đó ở lại primary.

## Khi Chọn Managed

Aurora MySQL và Azure Database for MySQL Flexible Server nhận phần Multi-AZ, failover và backup. Sơ đồ self-managed ở trên là để hiểu các vai trò, hoặc khi không được dùng managed. Tham số và Multi-AZ: [Setup SQL trên Cloud](../07-cloud-database/setup-sql-cloud.md).

SLA của nhà cung cấp (thường 99.95% cho RDS Multi-AZ) không cộng thành 99.99% của sản phẩm. Đo uptime ở phía app.

## Checklist

- [ ] Ba mysqld ở ba failure domain
- [ ] GTID, semisync chờ 1 replica, flush log mỗi commit
- [ ] Hai proxy sau VIP, app không giữ IP mysqld
- [ ] Ba Orchestrator Raft, hook cập nhật ProxySQL và chặn primary cũ
- [ ] Backup ra ngoài cụm, đã restore thử
- [ ] Cảnh báo lag, semisync rơi về async, và đĩa binlog
- [ ] Một lần failover game day đã đo được RTO thực
- [ ] Ledger không dùng timeout semisync 10 giây; mất ACK thì ngừng ghi thay vì chuyển async im lặng

---

> **Điểm Mấu Chốt:** 99.99% cần failover tự động dưới một phút và backup cho loại sự cố failover không sửa được. Semisync giữ transaction đã ACK; Orchestrator chọn primary mới; ProxySQL dẫn app sang đó.
