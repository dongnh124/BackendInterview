# Nâng Cấp và Đổi Cấu Hình

Thứ tự an toàn trên cụm có replica: sửa replica trước, primary sau. Primary đang nhận ghi là máy đổi cuối cùng.

Khởi tạo replica và đo lag: [03-ha-replication](../03-ha-replication/README.md). File này chỉ nói thứ tự khi phiên bản hoặc `my.cnf` thay đổi.

## Lệch Phiên Bản Được Phép

Trong upgrade window, replica được mới hơn primary đúng một bậc hỗ trợ. MySQL 8.4 replica kéo binlog từ primary 8.0 là đường nâng cấp thường dùng. Chiều ngược lại thì không: primary mới không được đứng trước replica cũ, vì replica cũ có thể không hiểu event trong binlog.

```
Được, trong lúc nâng cấp:
  Primary 8.0  →  Replica 8.4  →  Replica 8.4

Không giữ lâu:
  Primary 8.4  →  Replica 8.0

Không trộn:
  Oracle MySQL với MariaDB
  5.7 với 8.4 trong cùng một chuỗi
```

Hết upgrade window thì mọi node cùng phiên bản. Cụm lệch phiên bản không phải trạng thái vận hành.

MySQL 8 tự nâng data dictionary lúc mở datadir bằng binary mới. Không có bước `mysql_upgrade` riêng như 5.7. Vẫn đọc release notes của đúng bản sẽ nhảy tới: một số bản đổi mặc định (`utf8mb4`, authentication plugin, tên variable source/replica).

## Rolling Upgrade

```
1. Đọc release notes. Ghi variable nào bị đổi tên hoặc bị bỏ.
2. Nâng từng replica:
     dừng SQL thread nếu cần, cài binary mới, mở mysqld
     chờ Replica_IO_Running và Replica_SQL_Running = Yes
     chờ catch up GTID của primary
3. Khi mọi replica đã ở bản mới, chuyển ghi:
     Orchestrator promote một replica đã nâng
     hoặc chờ maintenance window rồi nâng primary tại chỗ
4. Primary cũ gắn lại làm replica, rồi nâng nốt nếu chưa nâng.
5. ProxySQL trỏ writer vào primary mới trước khi app ghi lại.
```

Group Replication và InnoDB Cluster cũng lên từng thành viên, nhưng thành viên cuối cùng rời nhóm phải còn quorum. Không tắt hai node cùng lúc trên cụm ba node.

Galera làm tương tự: node ở chế độ bảo trì, SST hoặc IST khi vào lại, không bao giờ còn dưới quorum.

## Đổi Cấu Hình

Dynamic variable (`SET PERSIST`) apply không cần restart và được ghi vào `mysqld-auto.cnf`. Variable chỉ có hiệu lực lúc khởi động thì rolling restart:

```
1. Sửa my.cnf trên một replica
2. Restart replica đó
3. Chờ GTID catch up và lag về ngưỡng
4. Lặp lại replica tiếp theo
5. Promote một replica đã nhận cấu hình mới
   hoặc restart primary trong change window đã báo
6. Máy vừa nhường vai primary được restart sau khi đã là replica
```

Không restart hai replica cùng lúc nếu chỉ còn một replica bắt semisync. Primary sẽ chờ đến timeout rồi rơi về async.

Đổi `server_id`, datadir, hoặc bật GTID trên cụm đang chạy không nằm trong rolling restart thường. Bật GTID trên cụm cũ đã có dữ liệu là một dự án riêng: làm trên bản sao, reconcile, rồi cutover. Đừng bật `gtid_mode = ON` một phát trên primary production.

## Kiểm Tra Trước Khi Promote

Replica chỉ được thành primary khi cả bốn điều sau đúng.

```sql
-- 1. Cả hai replication thread đang chạy
SHOW REPLICA STATUS\G
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes

-- 2. Đã apply hết gì primary đã thực thi
--    Retrieved_Gtid_Set phủ Executed_Gtid_Set của primary
--    Seconds_Behind_Source = 0 sau khi IO đã catch up

-- 3. Không có errant GTID
--    gtid_executed của replica trừ gtid_executed của primary = rỗng
```

Điều thứ tư không nằm trong `SHOW REPLICA STATUS`: relay log đã apply xong. Semisync ACK trước khi SQL thread chạy, nên một replica “đã nhận” vẫn có thể chưa apply. Orchestrator chờ bước này. Làm tay thì chờ `Replica_SQL_Running_State` hết backlog trước khi `SET GLOBAL super_read_only = OFF`.

Sau khi promote:

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
```

Primary cũ, nếu còn mở được, đặt ngay:

```sql
SET GLOBAL super_read_only = ON;
```

Rồi trỏ nó về primary mới bằng `CHANGE REPLICATION SOURCE TO` và GTID hiện tại. Không để nó nhận ghi “thêm một lúc”.

## Ưu Tiên Trong Môi Trường Banking

Nâng cấp ledger nằm trong change window đã được duyệt, có người thứ hai xác nhận, và đã chạy trên bản sao dữ liệu production. Không promote một replica vừa nâng lên làm primary giữa giờ posting hoặc giữa end-of-day batch.

```
Ưu tiên: mọi replica lên bản mới trong change window → reconcile → cutover có chủ đích
Tránh:    Orchestrator tự elect replica bản mới khi primary cũ vẫn đang nhận posting
```

Lệch phiên bản chỉ tồn tại trong change window đó. Hết change window, mọi node cùng một patch. Đổi `sync_binlog`, `innodb_flush_log_at_trx_commit`, hoặc timeout semisync trên ledger đi cùng change record, không `SET PERSIST` tay rồi quên.

## Checklist

- [ ] Replica nâng trước, primary sau
- [ ] Không để primary mới hơn replica
- [ ] Mỗi replica catch up GTID trước khi đụng replica kế tiếp
- [ ] Cụm quorum không mất majority trong lúc restart
- [ ] Errant GTID được xử lý trước khi promote
- [ ] Relay log apply xong trước khi tắt `super_read_only`
- [ ] Proxy đổi writer trước khi ứng dụng ghi vào primary mới
- [ ] Primary cũ bị khóa ghi rồi mới được gia nhập lại
- [ ] Ledger cutover trong change window đã duyệt, sau khi bản sao dữ liệu production đã nâng thử

---

> **Điểm Mấu Chốt:** Replica mới hơn primary thì được, trong lúc nâng cấp. Primary mới hơn replica thì binlog có thể không apply được. Promote chỉ diễn ra sau khi GTID khớp và relay log đã chạy hết.
