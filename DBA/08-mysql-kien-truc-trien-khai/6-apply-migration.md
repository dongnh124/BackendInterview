# Apply Migration Theo Topology

File migration có thể giống nhau. Chỗ nó chạy, cách DDL tới các node còn lại, và khoảng thời gian schema không giống nhau thì phụ thuộc topology.

Expand-contract, Flyway và forward-only không đổi theo topology. Xem [06-schema-migrations](../06-schema-migrations/README.md). File này chỉ nói đường apply.

## Việc Giống Nhau

Mọi cụm có replication đều giữ ba quy tắc:

1. **Một writer.** Flyway hoặc Liquibase nối vào primary, hoặc vào cổng read-write của proxy. Replica đang `super_read_only` sẽ từ chối DDL. Nối vào replica khi cờ đó tắt thì tạo errant GTID.
2. **Không `sql_log_bin = 0`.** DDL chỉ nằm trên một node. Node khác không có schema đó, binlog sau đó apply lên schema lệch.
3. **Bảng lịch sử của công cụ phải đi cùng DDL.** `flyway_schema_history` replicate theo binlog. Sau failover, primary mới vẫn biết migration nào đã xong.

DDL trong MySQL là statement, kể cả khi `binlog_format = ROW`. Node nhận event sẽ **chạy lại** câu `ALTER`, không nhận bản copy của bảng.

| Algorithm | Trên node chạy | Trên node nhận replication |
|-----------|----------------|----------------------------|
| `INSTANT` | Đổi metadata | Đổi metadata, nhanh |
| `INPLACE` | Rebuild, DML thường vẫn chạy | Rebuild lại, tốn thời gian bằng node đầu |
| `COPY` | Khóa và copy | Khóa và copy lại |

`INSTANT` là lựa chọn đầu. `COPY` trên bảng lớn làm replica hoặc secondary đứng sau hết thời gian copy.

Backfill (`UPDATE` hàng loạt, `INSERT ... SELECT`) không phải DDL. Cắt chunk. Một transaction quá lớn bị từ chối trên Group Replication (`group_replication_transaction_size_limit`, mặc định 150MB) dù classic replication vẫn nhận.

## Standalone

Không có node thứ hai. Migration xong là schema xong. Metadata lock trên chính node đó vẫn chặn DML của app trong lúc `COPY` hoặc lúc chờ MDL.

Expand-contract vẫn cần nếu app deploy nhiều instance. Topology đơn không bỏ được bước đó.

## Primary–Replica

```
Flyway ──► primary (writer)
              │  DDL statement trong binlog
              ├─► replica A   chạy lại ALTER, SQL thread bị DDL chiếm
              └─► replica B
```

Chạy một lần trên primary. Không chạy lại file trên từng replica.

Trong lúc replica apply DDL, event phía sau nằm chờ. `Seconds_Behind_Source` tăng xấp xỉ thời gian `ALTER`. App đọc qua replica thấy schema cũ, hoặc chờ MDL, cho đến khi catch up. Đọc balance và posting giữ trên primary, như [Proxy và Failover](3-proxy-va-failover.md).

Session của Flyway không đi qua luật tách đọc/ghi. Cả session ghim hostgroup writer. Một `SELECT` lịch sử bị đưa sang replica sẽ đọc nhầm schema hoặc không ghi được.

Công cụ copy ngoài (`gh-ost`, `pt-online-schema-change`) viết DML vào binlog của primary. Replica apply từng chunk, lag tăng theo tốc độ copy. `gh-ost` throttle khi replica lag. `pt-osc` thêm trigger trên primary; trigger đó replicate. Cả hai đều giả định một primary và replica classic.

**Async.** Primary trả OK khi đã ghi binlog local. Replica có thể chưa nhận. Primary chết ngay sau đó thì failover mất DDL vừa apply, cùng hàng trong `flyway_schema_history` nếu event chưa sang replica.

**Semisync.** Primary trả OK khi ít nhất một replica đã nhận event vào relay log, chưa phải khi replica apply xong. Flyway thấy thành công trong khi SQL thread replica vẫn đang chạy `ALTER`. RPO của chính câu DDL gần 0 nếu semisync chưa rơi về async. Lag đọc trên replica vẫn còn.

Failover giữa chừng: `ALTER` InnoDB bị ngắt thì bảng trên node đó giữ schema cũ và event không vào binlog. Flyway thất bại, chạy lại được. `gh-ost` bị ngắt thì để lại bảng ghost và trigger. Dọn trước khi chạy lại. Không promote replica khi GTID của DDL chưa apply xong — cùng điều kiện ở [Nâng cấp](5-nang-cap.md).

## Group Replication và InnoDB Cluster

DDL vẫn chạy trên primary rồi được group certify. Secondary chạy lại câu `ALTER`, giống replica. Khác ở chỗ không có `Seconds_Behind_Source`. Lag là hàng đợi applier (`COUNT_TRANSACTIONS_IN_QUEUE` trong `performance_schema.replication_group_member_stats`).

```
Flyway ──► Router 6446 (read-write) ──► primary hiện tại
                                              │
                                              ├─► secondary
                                              └─► secondary
```

InnoDB Cluster: nối Flyway vào cổng read-write của MySQL Router (mặc định 6446), không nối IP một member và không nối cổng read-only 6447. Sau khi group elect primary mới, 6446 đổi theo. IP member cũ có thể đã thành secondary.

Single-primary: chỉ primary nhận DDL. Multi-primary: không chạy migration khi node khác vẫn DML trên cùng bảng. Certification và MDL đẩy một bên lỗi.

`gh-ost` đo lag kiểu classic replica. Trên group, con số đó không còn nghĩa. DDL native (`INSTANT` / `INPLACE`) là đường mặc định. Dùng `gh-ost` chỉ khi đã có cách throttle theo queue của group, và tool nối vào primary.

Giới hạn kích thước transaction của group áp vào backfill, không áp vào `ALTER` kiểu metadata. Chunk backfill nhỏ hơn `group_replication_transaction_size_limit`.

## Galera

Galera không replicate DDL theo binlog classic. `wsrep_OSU_method` chọn cách schema đổi trên cả cụm.

| Method | DDL replicate? | DML trong lúc DDL | Schema các node |
|--------|----------------|-------------------|-----------------|
| `TOI` (mặc định) | Có, total order | Cả cụm chờ DDL | Giống nhau khi câu lệnh trả về |
| `RSU` | Không | Node khác vẫn ghi | Lệch cho đến khi chạy tay đủ mọi node |
| `NBO` (Galera 4) | Có, với một số DDL như `CREATE INDEX` / `DROP INDEX` | DML trên bảng vẫn chạy | Giống nhau khi xong |

`TOI` với `INSTANT` thì ngắn. `TOI` với `COPY` thì cụm ngừng ghi suốt thời gian copy.

`RSU` là rolling schema upgrade: `SET wsrep_OSU_method = RSU` rồi chạy cùng một DDL trên **từng** node. Quên một node là schema lệch. DML replicate vào node chưa có cột mới sẽ lỗi, hoặc ghi vào cấu trúc cũ.

`pt-osc` dựa trên trigger và khóa đổi tên bảng. Không lấy đó làm đường mặc định trên Galera. `NBO` hoặc `TOI` trong change window đi trước.

## So Sánh

| | Chạy migration ở đâu | DDL tới node khác | Schema có thể lệch? | Lag cần canh |
|--|----------------------|-------------------|---------------------|--------------|
| Standalone | Node duy nhất | Không có | Không | MDL trên node đó |
| Async primary–replica | Primary | Binlog, chạy lại | Có, cho đến khi replica apply | `Seconds_Behind_Source` |
| Semisync primary–replica | Primary | Như async, OK sau ACK relay log | Có, trong lúc apply | Relay log đã có, SQL thread chưa xong |
| GR / InnoDB Cluster | Primary, hoặc Router 6446 | Group, chạy lại | Có, trong lúc applier queue | `COUNT_TRANSACTIONS_IN_QUEUE` |
| Galera `TOI` | Một node, statement được order | Total order | Không sau khi câu lệnh trả về | Cả cụm chờ |
| Galera `RSU` | Tay, từng node | Không | Có, cố ý, đến khi đủ node | Flow control nếu DML không hợp schema |

## Ưu Tiên Trong Môi Trường Banking

Ledger giữ schema giống nhau trên mọi node đang nhận ghi. Một node thiếu cột là posting ghi sai chỗ.

```
Ledger
  → Flyway vào writer, một lần
  → INSTANT, không được thì INPLACE
  → Backfill chia chunk, dưới giới hạn transaction của group nếu dùng GR
  → Galera: TOI hoặc NBO. Không RSU
  → Không sql_log_bin = 0
  → Không promote và không chạy giữa giờ posting hoặc end-of-day batch
  → gh-ost chỉ trên primary–replica, cutover nằm trong change window, failover để semi-automatic

Reporting
  → Replica async trễ sau DDL thì được, app reporting không đọc bảng đang đổi
```

Semisync timeout dài, như [Kiến trúc HA](4-kien-truc-ha.md), cũng áp vào session migration: replica không nhận event thì Flyway chờ, không trả thành công rồi để schema chỉ có trên primary.

## Checklist

- [ ] Tool nối writer hoặc Router 6446, không nối replica
- [ ] Không `sql_log_bin = 0`
- [ ] `ALGORITHM` đã chọn; bảng lớn không dùng `COPY` lúc ledger đang ghi
- [ ] Backfill chia chunk
- [ ] Primary–replica: canh lag replica trước khi cho app đọc schema mới từ replica
- [ ] Group Replication: canh applier queue, không dùng `Seconds_Behind_Source`
- [ ] Galera ledger: `TOI` hoặc `NBO`, không `RSU`
- [ ] `gh-ost` bị dừng giữa chừng thì dọn bảng ghost trước lần chạy lại
- [ ] Change window của ledger không trùng posting và end-of-day batch

---

> **Điểm Mấu Chốt:** Cùng một file migration, primary–replica chạy lại DDL từ binlog, Group Replication chạy lại sau certification, Galera `TOI` chặn cả cụm cho đến khi schema giống nhau. `RSU` và `sql_log_bin = 0` cố ý để schema lệch — không dùng cho ledger.
