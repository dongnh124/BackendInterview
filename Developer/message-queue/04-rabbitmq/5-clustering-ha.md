# Clustering & HA — Cụm & High Availability

> High Availability (HA — Tính Sẵn Sàng Cao) cho RabbitMQ: clustering (cụm hóa), classic mirrored queues (hàng đợi mirror cổ điển), quorum queues (hàng đợi đồng thuận), stream queues, federation, và capacity planning.

## Mục Lục

1. [RabbitMQ Clustering](#rabbitmq-clustering)
2. [Queue Types](#queue-types)
3. [Classic Mirrored Queues](#classic-mirrored-queues)
4. [Quorum Queues](#quorum-queues)
5. [Stream Queues](#stream-queues)
6. [Federation & Shovel](#federation--shovel)
7. [HA Design Patterns](#ha-design-patterns)
8. [Operations & Troubleshooting](#operations--troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## RabbitMQ Clustering

**RabbitMQ Cluster** — nhiều broker nodes chia sẻ metadata và optionally queue data.

```
┌─────────────────────────────────────────────────────────┐
│                  RABBITMQ CLUSTER (3 nodes)                │
│                                                         │
│  Node 1 (rabbit@node1)    Node 2 (rabbit@node2)         │
│  ┌─────────────────┐      ┌─────────────────┐          │
│  │ Metadata (shared)│◄────►│ Metadata (shared)│          │
│  │ Queue A (leader) │      │ Queue A (replica)│          │
│  │ Queue B (replica)│◄────►│ Queue B (leader) │          │
│  └─────────────────┘      └─────────────────┘          │
│           ▲                        ▲                   │
│           └──────── Node 3 ────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Cluster Characteristics

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Metadata sharing** | Exchanges, queues, bindings, users — replicated across nodes |
| **Client connection** | Client có thể connect bất kỳ node — metadata redirect |
| **Queue data** | Phụ thuộc queue type — classic local, quorum replicated |
| **Node failure** | Metadata survive nếu đủ nodes; queue data tùy HA config |

### Forming Cluster

```bash
# Node 2 join cluster (Node 1 đã chạy)
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Kiểm tra
rabbitmqctl cluster_status
```

**Khuyến nghị:** Cluster **lẻ số nodes** (3, 5) — quorum decisions. Deploy nodes cùng **availability zone** hoặc cross-AZ với latency thấp.

---

## Queue Types

| Queue Type | Replication | Performance | Khuyến Nghị |
| ---------- | ----------- | ----------- | ----------- |
| **Classic** | Không (local) hoặc mirrored | Cao | Legacy — tránh mirrored mới |
| **Quorum** | Raft consensus | Trung bình | **Production default** |
| **Stream** | Replicated log | Cao throughput | Event log, replay |

```javascript
// Quorum queue (khuyến nghị)
await channel.assertQueue('orders.process', {
  durable: true,
  arguments: { 'x-queue-type': 'quorum' },
});

// Stream queue
await channel.assertQueue('events.stream', {
  durable: true,
  arguments: {
    'x-queue-type': 'stream',
    'x-max-length-bytes': 10_000_000_000,  // retention
  },
});
```

---

## Classic Mirrored Queues

**Classic Mirrored Queues (Hàng Đợi Mirror Cổ Điển)** — replicate toàn bộ queue sang node khác. **Deprecated** từ RabbitMQ 3.13, removed trong 4.x.

```
Node 1: Queue "tasks" (master)
Node 2: Queue "tasks" (mirror)  ← sync từ master
Node 3: Queue "tasks" (mirror)
```

### Policy Configuration (Legacy)

```bash
rabbitmqctl set_policy ha-tasks "^tasks\." \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

| ha-mode | Mô Tả |
| ------- | ----- |
| `all` | Mirror sang tất cả nodes |
| `exactly` | Mirror sang N nodes |
| `nodes` | Mirror sang nodes chỉ định |

### Vấn Đề Mirrored Queues

| Vấn Đề | Mô Tả |
| ------ | ----- |
| **Split-brain** | Network partition → 2 masters |
| **Performance** | Sync overhead, blocking operations |
| **Operational complexity** | Master migration, sync issues |
| **Deprecated** | Không dùng cho deployment mới |

> **Migration:** Chuyển sang **quorum queues** cho mọi deployment mới.

---

## Quorum Queues

**Quorum Queues (Hàng Đợi Đồng Thuận)** — dùng **Raft consensus algorithm (thuật toán đồng thuận Raft)** cho replication, khuyến nghị từ RabbitMQ 3.8+, default từ 4.x.

```
Quorum Queue "orders" (replication factor = 3)
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Replica 1│  │ Replica 2│  │ Replica 3│
│ (leader) │◄─┤ (follower)│◄─┤ (follower)│
└──────────┘  └──────────┘  └──────────┘
     Raft quorum: cần majority (2/3) để commit
```

### Đặc Điểm

| Đặc Điểm | Chi Tiết |
| -------- | -------- |
| **Durability** | Message commit khi majority replicas ack |
| **Consistency** | Linearizable — không split-brain |
| **Leader election** | Tự động khi leader node fail |
| **Poison handling** | `x-delivery-limit` — auto dead-letter |
| **Performance** | Thấp hơn classic ~vài % — acceptable trade-off |

### Configuration

```javascript
await channel.assertQueue('payments.process', {
  durable: true,
  arguments: {
    'x-queue-type': 'quorum',
    'x-quorum-initial-group-size': 3,
    'x-delivery-limit': 5,
    'x-dead-letter-exchange': 'payments.dlx',
    'x-max-in-memory-length': 0,  // all messages on disk
  },
});
```

### Quorum Queue Limitations

| Không Hỗ Trợ | Alternative |
| ------------ | ----------- |
| `x-max-priority` | Separate queues per priority |
| Lazy queue mode | Default on-disk behavior |
| Global QoS | Per-consumer prefetch |
| Exclusive queues | Durable shared queues |

---

## Stream Queues

**Stream Queues (Hàng Đợi Luồng)** — append-only log, gần với Kafka model.

```
Stream "events.orders"
  offset 0: { orderId: 1, ... }
  offset 1: { orderId: 2, ... }
  offset 2: { orderId: 3, ... }
  ...
  Consumer đọc từ offset bất kỳ — replay supported
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Replay** | Consumer track offset, đọc lại |
| **Retention** | `x-max-length-bytes`, time-based |
| **Use case** | Event log nhẹ, không cần full Kafka |
| **Replication** | Raft-based như quorum |

```javascript
// Consumer đọc stream từ offset
channel.consume('events.orders', handler, {
  arguments: { 'x-stream-offset': 'first' },  // 'first', 'last', 'next', timestamp
});
```

---

## Federation & Shovel

### Federation Plugin

**Federation** — link exchanges/queues giữa **clusters khác nhau** (cross-datacenter).

```
DC1 Cluster                    DC2 Cluster
┌─────────────┐               ┌─────────────┐
│ exchange A  │──federation──►│ exchange A' │
│ (upstream)  │               │ (downstream)│
└─────────────┘               └─────────────┘
```

**Use case:** Geo-distribution, hub-and-spoke topology.

### Shovel Plugin

**Shovel** — move message từ queue này sang queue/exchange khác (có thể khác cluster).

```
Source Queue (DC1) ──shovel──► Target Exchange (DC2)
```

**Use case:** DLQ replay cross-cluster, migration, bridge.

---

## HA Design Patterns

### Pattern 1: 3-Node Quorum Cluster

```
Production minimum:
  3 nodes, quorum queues
  Load balancer (HAProxy) → AMQP port 5672
  Management: port 15672

  Node failure: quorum majority (2/3) → service continues
  2 nodes fail: quorum lost → unavailable (by design)
```

### Pattern 2: Multi-AZ Deployment

```
AZ-1: Node 1
AZ-2: Node 2
AZ-3: Node 3

Quorum RF=3 across AZs
Trade-off: cross-AZ latency vs AZ failure tolerance
```

### Pattern 3: Blue-Green Queue Migration

```
1. Deploy new quorum queue "orders.v2"
2. Dual-write hoặc shovel từ "orders.v1"
3. Migrate consumers sang v2
4. Drain v1 → decommission
```

### Capacity Planning

| Resource | Guideline |
| -------- | --------- |
| **Memory** | Quorum queues buffer in memory — monitor `mem_used` |
| **Disk** | Persistent messages — SSD, monitor free space |
| **Connections** | ~10K per node typical — use connection pooling |
| **Channels** | Lightweight — prefer channels over connections |
| **File descriptors** | Tune OS ulimit |

```bash
# Monitor
rabbitmq-diagnostics status
rabbitmq-diagnostics check_running
```

---

## Operations & Troubleshooting

### Node Failure

```
1 node fail (3-node cluster):
  → Quorum queues: leader election, brief unavailability
  → Clients: reconnect via load balancer
  → Monitor: queue leader distribution

2 nodes fail:
  → Quorum lost — queues unavailable
  → Need manual intervention or wait for nodes recovery
```

### Split-Brain Prevention

Quorum queues dùng Raft — **không có split-brain** như mirrored queues. Minority partition → unavailable (correct behavior).

### Monitoring Metrics

| Metric | Alert |
| ------ | ----- |
| `rabbitmq_running` | Node down |
| `rabbitmq_queue_messages_ready` | Backlog |
| `rabbitmq_quorum_queue_leader` | Leader changes frequent |
| `rabbitmq_disk_free` | < 10% free |
| `rabbitmq_mem_used` | Memory alarm |

### Memory Alarm

Khi memory vượt threshold, broker **block publishers** — consumers phải drain.

```
Fix: increase memory limit, add consumers, reduce prefetch,
     enable queue TTL, move to quorum with disk-backed
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Mirrored queue vs Quorum queue?

**Trả lời:** **Mirrored** (deprecated) — master-slave sync, split-brain risk, operational issues. **Quorum** — Raft consensus, linearizable, auto leader election, `x-delivery-limit`, khuyến nghị production. Quorum trade-off: không hỗ trợ priority, performance hơi thấp hơn classic.

### Câu 2: RabbitMQ cluster 2 nodes có OK không?

**Trả lời:** **Không khuyến nghị.** Quorum cần **majority** — 2 nodes: 1 node fail = mất quorum (1/2 không đủ majority). Minimum **3 nodes** cho HA thực sự. 2 nodes chỉ tolerate 0 failures.

### Câu 3: Client connect 1 node — node đó chết thì sao?

**Trả lời:** Connection drop → client **reconnect** qua load balancer hoặc danh sách nodes. Queue data trên quorum queues vẫn available qua node khác (nếu quorum intact). Cần **connection recovery** logic trong client library.

### Câu 4: Khi nào dùng Stream queue thay Quorum?

**Trả lời:** **Stream** khi cần **replay**, retention log-like, nhiều consumer đọc cùng offset range, throughput cao. **Quorum** cho traditional queue semantics (competing consumers, ack/nack, DLQ). Stream gần Kafka; Quorum gần classic RabbitMQ.

### Câu 5: Federation khác Shovel thế nào?

**Trả lời:** **Federation** — continuous link, exchange/queue **tự động** forward message giữa brokers/clusters (topology-based). **Shovel** — **point-to-point** move message từ source queue sang destination (task-based, có thể one-time). Federation cho ongoing geo-replication; Shovel cho migration, DLQ replay.

---

**Hoàn thành chủ đề RabbitMQ.** Tiếp theo: [06-reliability](../06-reliability/README.md) — reliability patterns cross-broker, hoặc [12-interview-prep](../12-interview-prep/) — câu hỏi phỏng vấn.
