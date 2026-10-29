# Topology Triển Khai MySQL

Mỗi kiểu dưới đây là một cách xếp node. Cơ chế async, semisync và lag được giải thích ở [Đồng Bộ vs Bất Đồng Bộ](../03-ha-replication/dong-bo-va-bat-dong-bo.md). Binary log ở [Các Loại Nhân Bản](../03-ha-replication/cac-loai-replication.md).

Phiên bản giả định: MySQL 8.0 hoặc 8.4. Thuật ngữ trong repo ở `03-ha-replication` còn dùng master/slave; từ 8.0.22 tài liệu chính thức dùng source/replica. Cùng một vai trò.

## 1. Standalone

```
[App] → [mysqld]
```

Một tiến trình, một đĩa dữ liệu. Không có replica để đọc hay để lên thay.

| | |
|--|--|
| RPO | Bằng khoảng cách tới backup gần nhất |
| RTO | Thời gian cài lại máy và restore |
| Khi dùng | Dev, công cụ nội bộ, dữ liệu dựng lại được |
| Khi không dùng | Production có người dùng đang ghi |

## 2. Primary–Replica, GTID, Async

```
[App ghi] → [Primary]
               │  binlog, GTID, async
               ├─→ [Replica A]  ← app đọc
               └─→ [Replica B]
```

Primary ghi binlog rồi trả OK cho client, không chờ replica. Replica có thể chậm vài giây hoặc vài phút.

| | |
|--|--|
| RPO | Vài giây nếu lag thấp; bằng phần binlog chưa kịp sang replica nếu primary mất đĩa |
| RTO | 30 giây–vài phút nếu có công cụ promote; lâu hơn nếu làm tay |
| Khi dùng | Đọc nhiều hơn ghi, mất một ít transaction cuối khi cháy máy là chấp nhận được |
| Khi không dùng | Sau khi client nhận OK thì không được mất transaction đó |

Failover không nằm sẵn trong mysqld. Cần Orchestrator hoặc một người làm đúng thứ tự ở [Proxy và Failover](3-proxy-va-failover.md).

## 3. Primary–Replica, Semisync

Cùng hình với async. Khác ở thời điểm primary được phép trả OK.

```
Client COMMIT
  → Primary ghi binlog
  → Chờ ít nhất 1 replica nhận event vào relay log
  → Trả OK
Nếu quá rpl_semi_sync_source_timeout (mặc định 10 giây)
  → Rơi về async cho đến khi có replica nhận lại
```

ACK của semisync là “replica đã nhận vào relay log”, chưa phải “đã apply xong”. Replica được promote phải apply nốt relay log trước khi nhận ghi.

| | |
|--|--|
| RPO | Gần 0 với commit đã trả OK, miễn là chưa rơi về async |
| RTO | Giống async, cộng thời gian apply relay log còn đọng |
| Khi dùng | OLTP cần commit đã ACK còn sống trên một máy khác |
| Khi không dùng | Replica ở xa và không chịu thêm network latency trên mỗi commit |

Timeout là cái bẫy: semisync im lặng thành async khi replica treo. Giám sát `Rpl_semi_sync_source_status` và số lần timeout, đừng chỉ tin tên cấu hình.

**Banking.** Với ledger, timeout mặc định 10 giây là quá ngắn: hết giờ là commit vẫn trả OK dù chưa có replica nào nhận. Tăng `rpl_semi_sync_source_timeout` đủ dài để client bị chặn và on-call thấy sự cố, thay vì transaction biến mất trong im lặng. Group Replication single-primary rõ hơn ở điểm này: không đủ majority thì commit trả lỗi, không có đường rơi về async.

Chi tiết plugin và semisync variable: [Đồng Bộ vs Bất Đồng Bộ](../03-ha-replication/dong-bo-va-bat-dong-bo.md).

## 4. Group Replication

```
[App ghi] → [Primary]
              │  Paxos, majority
              ├─→ [Secondary]
              └─→ [Secondary]
         group communication port, thường 33061
```

Replication nằm trong mysqld (`group_replication`). Transaction chỉ commit khi majority thành viên certify. Mất liên lạc thì phía minority không ghi được — đó là cách ngăn split-brain, và cũng là lý do cụm 2 node vô dụng: đứt một bên là không còn majority.

Chế độ vận hành:

- **Single-primary** (mặc định): một node nhận ghi, các node kia chỉ đọc. Đủ cho hầu hết OLTP.
- **Multi-primary**: mọi node nhận ghi. Xung đột hai transaction cùng sửa một hàng làm một bên rollback.

Điều kiện để bật: GTID, `binlog_format = ROW`, transactional table là InnoDB. Tối thiểu 3 thành viên cho production.

| | |
|--|--|
| RPO | 0 đối với transaction đã commit, nếu còn majority |
| RTO | Vài giây để nhóm elect primary mới |
| Khi dùng | Muốn HA gói trong MySQL, không muốn dịch vụ failover riêng |
| Khi không dùng | Chỉ có 2 máy, hoặc ghi rất lớn trên mạng chậm (mỗi commit chờ majority) |

## 5. InnoDB Cluster

```
[App] → [MySQL Router]
            │
            ├─→ [Primary]     ┐
            ├─→ [Secondary]   ├─ Group Replication
            └─→ [Secondary]   ┘
                 metadata do MySQL Shell quản lý
```

Ba thành phần, một protocol:

- Group Replication giữ dữ liệu và elect primary
- MySQL Shell (`dba.createCluster`, `cluster.addInstance`) tạo cụm và metadata
- MySQL Router đọc metadata để biết đâu là cổng ghi, đâu là cổng đọc

Không gắn Orchestrator lên cụm này. Router và Group Replication đã đổi hướng ghi khi primary đổi. Thêm một công cụ promote nữa sẽ có hai quyết định trái nhau.

| | |
|--|--|
| RPO / RTO | Như Group Replication |
| Khi dùng | Muốn một bộ Oracle hỗ trợ: Shell + Router + GR |
| Khi không dùng | Đã có ProxySQL và Orchestrator cho cụm primary–replica |

## 6. Galera

```
[App] → [Proxy] → [Node] ←→ [Node] ←→ [Node]
                  wsrep, certification trước khi commit
```

Galera không có trong Oracle MySQL. Nó đi cùng Percona XtraDB Cluster hoặc MariaDB Galera Cluster. Mọi node có thể ghi. Trước khi commit, transaction được certify trên cụm; hai transaction đụng cùng hàng thì một bên bị từ chối.

Khác Group Replication: wsrep protocol, state transfer (SST/IST) riêng, và flow control có thể làm cả cụm chậm theo node chậm nhất. Số node nên là số lẻ. Hai node thì thêm garbd (arbitrator, không chứa dữ liệu) để vẫn có vote khi một node chết.

| | |
|--|--|
| RPO | 0 với transaction đã commit trên cụm còn quorum |
| RTO | Node còn lại nhận ghi ngay nếu còn quorum |
| Khi dùng | Đội đã chạy Percona hoặc MariaDB và cần ghi nhiều node |
| Khi không dùng | Đang chuẩn hoá trên Oracle MySQL 8 — dùng Group Replication |

## So Sánh

| | Đọc scale | Ghi | Mất commit đã ACK | Ai elect primary | Số node tối thiểu |
|--|-----------|-----|-------------------|----------------|-------------------|
| Standalone | Không | 1 node | Có, đến lần backup | Không có | 1 |
| Async GTID | Replica | 1 primary | Có, phần binlog chưa gửi | Bên ngoài | 2 |
| Semisync | Replica | 1 primary | Không, trừ khi đã rơi về async | Bên ngoài | 2 |
| Group Replication | Secondary | 1 hoặc nhiều | Không, nếu còn majority | Trong mysqld | 3 |
| InnoDB Cluster | Secondary | 1 (single-primary) | Không, nếu còn majority | Trong mysqld, Router đi theo | 3 |
| Galera | Mọi node | Mọi node | Không, nếu còn quorum | Không có primary cố định | 3, hoặc 2 + garbd |

## Ưu Tiên Trong Môi Trường Banking

Hệ thống ghi balance và posting dùng một primary. Hai transaction cùng sửa một hàng mà một bên bị cụm từ chối sau khi ứng dụng đã tính balance là loại lỗi ngân hàng không chấp nhận — đó là lý do multi-primary đứng sau single-primary.

```
Ledger, core payment, balance
  → Semisync (timeout dài) hoặc Group Replication / InnoDB Cluster, single-primary
  → Async chỉ để kéo thêm replica đọc cho reporting, không để giữ commit duy nhất

Inquiry, reporting, notification
  → Async replica đọc được
  → Standalone chỉ với synthetic data ở môi trường dev

Galera
  → Chỉ khi cả tổ chức đã vận hành Percona hoặc MariaDB
  → Vẫn một node ghi, proxy chặn ghi vào các node còn lại
```

## Checklist

- [ ] Đã chọn một replication protocol, không chồng Galera lên Group Replication
- [ ] Production có replica thì GTID bật trên mọi node
- [ ] Cụm cần quorum thì có ít nhất 3 vote (node hoặc garbd)
- [ ] Người vận hành nói được RPO khi primary mất điện ngay sau commit
- [ ] Đã chỉ failover về một chỗ: Orchestrator hoặc Group Replication
- [ ] Ledger không dùng async làm đường giữ commit, và không dùng multi-primary

---

> **Điểm Mấu Chốt:** Async cho phép mất transaction đã trả OK. Semisync và Group Replication giữ những transaction đó, đổi lại commit phải chờ một máy khác. Chọn trước khi viết sơ đồ mạng.
