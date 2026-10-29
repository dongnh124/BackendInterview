# Proxy và Failover

Proxy trả lời câu “ghi vào đâu, đọc ở đâu”. Failover trả lời câu “ai là primary”. Hai việc đó không gộp vào một process.

```
[App]
   │
[ProxySQL hoặc MySQL Router]     ← đổi đích kết nối
   │
[Primary] ←── [Replica] [Replica]
   ▲
[Orchestrator hoặc Group Replication]   ← quyết định ai là primary
```

Chiến lược failover thủ công, semi-automatic và tự động nói chung: [Chiến Lược Failover](../03-ha-replication/chien-luoc-failover.md). File này đặt Orchestrator và Router vào đúng chỗ trên sơ đồ MySQL. MHA được nhắc ở tài liệu đó như một tên cũ; cụm mới dùng Orchestrator hoặc Group Replication.

## Chọn Một Cặp

| Topology | Proxy | Ai promote |
|----------|-------|----------------|
| Primary–replica, async hoặc semisync | ProxySQL | Orchestrator |
| InnoDB Cluster | MySQL Router | Group Replication |
| Galera | ProxySQL | Không elect primary; node mất quorum thì ngừng ghi |

Không chạy Orchestrator trên InnoDB Cluster. Group Replication đã elect primary. Orchestrator nhìn `SHOW REPLICA STATUS` sẽ không thấy topology nó hiểu, hoặc sẽ cố promote một node mà nhóm không công nhận.

## ProxySQL

Đứng trước cụm primary–replica. App chỉ biết một địa chỉ.

```
Cổng 6033   app kết nối vào đây
Cổng 6032   admin cấu hình ProxySQL

Hostgroup 10   writer   = primary hiện tại
Hostgroup 20   reader   = các replica
```

Luật tối thiểu:

- `SELECT` thường → hostgroup 20
- `INSERT`, `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` → hostgroup 10
- Transaction đã ghi thì các câu sau trong transaction đó cũng vào hostgroup 10

Đọc từ replica là đọc cũ nếu replica đang lag. Transaction cần read-your-writes thì ghim vào writer. Đo lag trước khi đưa một replica vào hostgroup 20: [Đồng Bộ vs Bất Đồng Bộ](../03-ha-replication/dong-bo-va-bat-dong-bo.md).

ProxySQL không tự biết primary mới sau failover. Orchestrator, qua hook, sửa hostgroup 10. Hai instance ProxySQL đứng sau một VIP để chính proxy không thành điểm chết.

User `monitor` cần `REPLICATION CLIENT` để ProxySQL thấy replica còn sống. Privilege này không cho sửa dữ liệu.

## MySQL Router

Chỉ dùng với InnoDB Cluster. Bootstrap một lần từ metadata của cụm:

```bash
mysqlrouter --bootstrap clusteradmin@10.0.0.11:3306 --user=mysqlrouter
```

Sau bootstrap, Router tự đọc bảng metadata và mở cổng cố định:

```
6446   đọc-ghi, luôn trỏ primary của Group Replication
6447   chỉ đọc, trỏ secondary
```

App ghi vào 6446, đọc scale vào 6447. Khi Group Replication elect primary mới, Router đổi 6446 theo, không cần hook bên ngoài. Chạy ít nhất hai Router sau một VIP.

## Orchestrator

Dịch vụ riêng, không nằm trong mysqld. Nó đọc `SHOW REPLICA STATUS` (trước 8.0.22 là `SHOW SLAVE STATUS`) và vẽ cây primary–replica.

Ba node Orchestrator chạy Raft. Quyết định failover cần majority. Một node Orchestrator chết không được phép auto-promote replica — nếu không, hai Orchestrator mất liên lạc sẽ elect hai primary.

Khi primary không trả lời:

```
1. Đa số Orchestrator đồng ý là primary đã mất
2. Chọn replica có GTID đầy đủ nhất, không có errant GTID
3. Chờ replica apply hết relay log
4. SET super_read_only = OFF trên replica đó
5. Replica cũ: super_read_only = ON nếu còn SSH/SQL tới được
6. Hook cập nhật hostgroup writer của ProxySQL
7. Primary cũ nếu sống lại thì gắn lại làm replica, không mở ghi
```

Bước 5 và 7 là fencing. Bỏ qua thì primary cũ vẫn nhận ghi từ ai đó còn giữ connection trực tiếp.

## Group Replication Thì Khác

Không có bước 2–5 ở trên. Nhóm Paxos mất primary thì phía còn majority elect một secondary thành primary. Phía minority, kể cả primary cũ bị cô lập, không commit được.

Router thấy primary mới qua metadata. Không có hook ProxySQL.

Cụm 3 node chịu chết 1 node. Chết 2 node thì phần còn lại không có majority và ngừng ghi. Đó là đánh đổi để không split-brain.

## Split-Brain

Hai node cùng nhận ghi, rồi cùng mang GTID mà bên kia không có. Ghép lại sẽ gãy hoặc mất hàng.

Chặn ở primary–replica:

- Mọi replica `super_read_only = ON`. `read_only` một mình không chặn user có privilege cao.
- App không có connection string thẳng tới IP primary. Chỉ tới proxy.
- Orchestrator từ chối promote replica đang giữ transaction lạ (errant GTID): `gtid_executed` của replica trừ đi `gtid_executed` của primary phải rỗng.
- Primary cũ bị cô lập mạng hoặc bị đặt `super_read_only` trước khi primary mới mở ghi.
- Một cụm Orchestrator Raft, không phải hai bộ script failover.

Chặn ở Group Replication: không thêm node vào nhóm nếu nó đã có GTID ngoài nhóm. Không chạy hai cụm cùng `group_replication_group_name`.

## Ưu Tiên Trong Môi Trường Banking

Hai primary cùng ghi một ledger là sự cố nghiêm trọng hơn vài phút không ghi được. Vì vậy failover của ledger đi chậm hơn failover của reporting.

```
Ledger
  → Orchestrator (hoặc cụm GR) báo primary mất
  → Người trực xác nhận fencing: primary cũ không còn nhận ghi
  → Mới promote và trỏ proxy
  → Automatic chỉ sau khi game day đã chứng minh fencing và hết errant GTID

Reporting, inquiry không giữ balance
  → Failover tự động được
  → SELECT balance vừa ghi vẫn vào hostgroup writer
```

Câu `SELECT` balance dùng để quyết định một posting tiếp theo đi cùng đường ghi. Đưa nó sang replica lag là trừ tiền trên số liệu cũ.

## Checklist

- [ ] Một cặp duy nhất: ProxySQL + Orchestrator, hoặc Router + Group Replication
- [ ] App không nối thẳng IP của mysqld
- [ ] Replica `super_read_only = ON`
- [ ] Orchestrator chạy 3 node nếu failover tự động
- [ ] Hook sau failover cập nhật proxy và chặn ghi trên primary cũ
- [ ] Errant GTID làm failover dừng lại để người xem, không tự ý bỏ qua
- [ ] Đọc-ghi trong cùng transaction không bị đưa sang replica
- [ ] Ledger không auto-promote khi on-call chưa xác nhận, trừ khi đã có fencing game day

---

> **Điểm Mấu Chốt:** ProxySQL không elect primary. Orchestrator không nhận kết nối của app. Mỗi cái một việc, và chỉ một trong hai cơ chế được quyền promote.
