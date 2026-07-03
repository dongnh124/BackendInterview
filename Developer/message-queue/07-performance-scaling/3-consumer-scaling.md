# Consumer Scaling — Mở Rộng Consumer

> Consumer Scaling (Mở Rộng Consumer) là phản xạ đầu tiên khi lag tăng — nhưng có **giới hạn cứng** bởi partition count (Kafka) và prefetch/worker design (RabbitMQ). Hiểu khi nào scale giúp và khi nào lãng phí.

## Mục Lục

1. [Tóm Tắt Nhanh](#tóm-tắt-nhanh)
2. [Scaling Laws — Quy Luật Mở Rộng](#scaling-laws--quy-luật-mở-rộng)
3. [Kafka Consumer Group Scaling](#kafka-consumer-group-scaling)
4. [RabbitMQ Consumer Scaling](#rabbitmq-consumer-scaling)
5. [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
6. [Vertical vs Horizontal Scaling](#vertical-vs-horizontal-scaling)
7. [Scaling Decision Tree](#scaling-decision-tree)
8. [Rebalance & Rolling Deploy](#rebalance--rolling-deploy)
9. [Anti-Patterns](#anti-patterns)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────────┐
│              CONSUMER SCALING — KEY FACTS                          │
├─────────────────────────────────────────────────────────────────┤
│  Kafka: max effective consumers = partition count per topic       │
│  RabbitMQ: scale bằng thêm consumer instances + prefetch tune     │
│  Scale consumer KHÔNG giúp nếu bottleneck là downstream (DB/API)  │
│  Monitor: lag per partition, CPU%, handler latency P99            │
└─────────────────────────────────────────────────────────────────┘
```

| Broker | Scaling Unit | Hard Limit |
| ------ | ------------ | ---------- |
| **Kafka** | Consumer trong cùng group | = partition count |
| **RabbitMQ** | Consumer trên cùng queue | Queue throughput, broker RAM |
| **SQS** | Lambda / ECS tasks | Account limits, API rate |
| **Redis Streams** | Consumer group members | Stream partition (1 stream) |

---

## Scaling Laws — Quy Luật Mở Rộng

### Kafka — 1 Partition = 1 Consumer Max

```
Topic: 6 partitions, Consumer Group "order-processors"

Consumers = 1:  C1 đọc P0-P5 (6 partitions)     → OK
Consumers = 3:  C1:P0,P1  C2:P2,P3  C3:P4,P5     → OK, balanced
Consumers = 6:  Mỗi consumer 1 partition          → Max parallelism
Consumers = 10: 6 active, 4 IDLE                    → Lãng phí!
```

### Throughput vs Consumer Count

```
Throughput
    │
    │                    ╱──── Plateau (bottleneck = handler/DB)
    │                 ╱
    │              ╱
    │           ╱
    │        ╱
    │─────╱
    └──────────────────────────────────► Consumer Count
         1    3    6    10   20
              ↑
         partition count = 6
```

**Plateau reasons (Lý do bão hòa):**

1. Đã đạt partition count limit
2. Downstream DB connection pool exhausted
3. Handler CPU-bound trên mỗi instance
4. Network bandwidth broker

---

## Kafka Consumer Group Scaling

### Cấu Hình Scale-Friendly

```javascript
const consumer = kafka.consumer({
  groupId: 'order-processors',
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
  maxBytesPerPartition: 1048576,
  maxWaitTimeInMs: 500,
});

// Static membership — tránh rebalance khi pod restart
// group.instance.id = process.env.HOSTNAME
```

### Scale Out Procedure

```
1. Check current lag (total + per partition)
2. Check consumer count vs partition count
3. If consumers < partitions AND consumer CPU < 70%:
   → Scale out (+1 to +2 instances)
4. Wait rebalance complete (~30s-2min)
5. Monitor lag trend 10-15 phút
6. Repeat nếu lag vẫn tăng
```

### K8s Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-consumer
spec:
  replicas: 6          # Match partition count
  template:
    spec:
      containers:
        - name: consumer
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          env:
            - name: KAFKA_GROUP_INSTANCE_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # Static membership
```

### Multi-Topic Consumer

```
Consumer group đọc nhiều topics:
  topic-a: 12 partitions
  topic-b: 6 partitions

→ Max parallelism = 12 (partition lớn nhất quyết định assignment phức tạp)
→ Cân nhắc tách consumer group per topic cho scale độc lập
```

---

## RabbitMQ Consumer Scaling

RabbitMQ scale bằng **thêm consumer processes** compete trên cùng queue.

```
Queue: orders (competing consumers)

  Consumer-1 ──┐
  Consumer-2 ──┼──► Queue ──► Round-robin dispatch
  Consumer-3 ──┘
  Consumer-4 ──┘
```

### Prefetch & Concurrency

```javascript
// Node.js — multiple consumers trên 1 process
const CONSUMER_COUNT = 4;

for (let i = 0; i < CONSUMER_COUNT; i++) {
  const ch = await connection.createChannel();
  await ch.prefetch(25);  // 4 × 25 = 100 unacked max
  await ch.consume('orders', handleMessage, { noAck: false });
}
```

| Pattern | Khi Nào Dùng |
| ------- | ------------ |
| **1 process, N channels** | I/O-bound handler |
| **N processes (PM2/cluster)** | CPU-bound handler |
| **K8s replicas** | Production, auto-scale |
| **prefetch=1** | Fair dispatch, long-running tasks |
| **prefetch=50+** | Short tasks, high throughput |

### Quorum Queue Scaling

**Quorum Queue (Hàng Đợi Đồng Thuận)** — replicate qua Raft, throughput thấp hơn classic queue nhưng durable hơn. Scale consumer tương tự classic queue.

> Xem [04-rabbitmq/5-clustering-ha.md](../04-rabbitmq/5-clustering-ha.md)

---

## Horizontal Pod Autoscaler (HPA)

**HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang** scale replicas dựa trên metrics.

### HPA với Custom Metrics (Kafka Lag)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-consumer
  minReplicas: 2
  maxReplicas: 12          # ≤ partition count!
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_group_lag
          selector:
            matchLabels:
              group: order-processors
              topic: orders
        target:
          type: AverageValue
          averageValue: "1000"   # Scale khi lag/pod > 1000
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Chậm scale down
```

### HPA Best Practices

| Practice | Lý Do |
| -------- | ----- |
| `maxReplicas ≤ partition count` | Tránh idle pods |
| Slow scale-down (5-10 min) | Tránh flapping khi lag spike ngắn |
| Scale-up nhanh hơn scale-down | Respond spike kịp thời |
| Kết hợp CPU + custom lag metric | CPU alone không đủ cho I/O-bound |
| Alert khi hit maxReplicas | Cần tăng partition hoặc optimize handler |

### KEDA (Kubernetes Event-Driven Autoscaling)

```yaml
# KEDA ScaledObject — native Kafka lag trigger
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-scaler
spec:
  scaleTargetRef:
    name: order-consumer
  minReplicaCount: 1
  maxReplicaCount: 12
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "500"
```

---

## Vertical vs Horizontal Scaling

| Approach | Mô Tả | Ưu | Nhược |
| -------- | ----- | -- | ----- |
| **Vertical (Scale Up)** | Tăng CPU/RAM per instance | Đơn giản, ít rebalance | Single point, ceiling |
| **Horizontal (Scale Out)** | Thêm instances | Linear scale (đến limit) | Rebalance, complexity |

### Khi Nào Vertical?

```
- Handler CPU-bound, single-threaded (Node.js event loop)
- Partition count = 1 (không scale horizontal được)
- Dev/staging environment
```

### Khi Nào Horizontal?

```
- I/O-bound handler (DB, HTTP calls)
- Partition count > 1
- Production với traffic variable
- Cần fault tolerance (1 pod die, others continue)
```

---

## Scaling Decision Tree

```
Consumer Lag tăng?
        │
        ├── Consumer count < partition count?
        │       ├── YES → Consumer CPU < 70%?
        │       │           ├── YES → SCALE OUT consumer
        │       │           └── NO → OPTIMIZE handler (profile code)
        │       └── NO → Đã max consumer
        │               │
        │               ├── Handler latency P99 cao?
        │               │       ├── YES → Downstream bottleneck
        │               │       │         → Scale DB, connection pool, cache
        │               │       └── NO → Producer burst?
        │               │               → Rate limit, backpressure
        │               │
        │               └── Partition skew (hot partition)?
        │                       → Fix key design (xem partitioning doc)
        │
        └── Lag spike ngắn (< 5 min)?
                → Có thể transient — monitor trước khi scale
```

---

## Rebalance & Rolling Deploy

### Rolling Deploy Impact

```
Deployment 6 replicas → rolling update:

T0: 6 consumers active
T1: Kill pod-1 → rebalance (5 active) → brief lag spike
T2: New pod-1 starting → rebalance (6 active)
...
Total: N rebalances trong 1 deploy → lag spike tạm thời
```

### Giảm Impact

| Technique | Mô Tả |
| --------- | ----- |
| **Static group membership** | `group.instance.id` — pod restart không revoke ngay |
| **Cooperative sticky assignor** | Chỉ move partitions cần thiết |
| **maxSurge=0, maxUnavailable=1** | Từng pod một |
| **Deploy off-peak** | Traffic thấp |
| **Pause autoscale during deploy** | Tránh compound rebalance |

---

## Anti-Patterns

| Anti-Pattern | Vấn Đề | Fix |
| ------------ | ------ | --- |
| Scale 3 → 30 consumers ngay | Rebalance storm, downstream overwhelm | Scale gradual +2 |
| HPA maxReplicas > partitions | Idle pods, wasted cost | Cap at partition count |
| Scale consumer khi DB maxed | Connection pool exhausted, worse | Scale DB first |
| Ignore per-partition lag | Hot partition hidden in average | Monitor per-partition |
| Scale during deploy + HPA | Double rebalance chaos | Coordinate deploy + autoscale |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Topic 12 partitions, lag cao — deploy bao nhiêu consumers?

**Trả lời:** Bắt đầu **6 consumers** (50% partition), monitor 10 phút. Nếu lag giảm và CPU < 70% → tăng lên **12** (max effective). Không deploy >12 — idle và lãng phí. Nếu 12 consumers mà lag vẫn cao → bottleneck không phải consumer count.

### Câu 2: HPA scale dựa trên CPU 70% có đủ không?

**Trả lời:** **Không đủ** cho I/O-bound consumer (chờ DB/API). CPU thấp nhưng lag cao. Dùng **custom metric: consumer lag per pod** hoặc **KEDA Kafka trigger**. Kết hợp CPU làm secondary signal.

### Câu 3: Rolling deploy gây lag spike — acceptable không?

**Trả lời:** **Brief spike** (vài phút) thường acceptable nếu SLA cho phép. Nếu không: dùng static membership, cooperative assignor, deploy off-peak, hoặc blue-green với consumer group mới.

---

**Cập Nhật:** 2026-07-03
