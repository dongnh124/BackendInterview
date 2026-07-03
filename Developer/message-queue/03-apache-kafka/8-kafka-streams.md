# Kafka Streams — Xử Lý Luồng Dữ Liệu

> Kafka Streams library: stream processing (xử lý luồng), topology (topology xử lý), state stores (kho trạng thái), joins (kết hợp stream), windowing (cửa sổ thời gian), và exactly-once processing semantics.

## Mục Lục

1. [Kafka Streams Là Gì?](#kafka-streams-là-gì)
2. [Core Concepts](#core-concepts)
3. [Stream Topology](#stream-topology)
4. [State Stores](#state-stores)
5. [Aggregations & Counting](#aggregations--counting)
6. [Joins](#joins)
7. [Windowing](#windowing)
8. [Exactly-Once Processing](#exactly-once-processing)
9. [Kafka Streams vs Alternatives](#kafka-streams-vs-alternatives)
10. [Operations & Monitoring](#operations--monitoring)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kafka Streams Là Gì?

**Kafka Streams** là **client library (thư viện client)** cho stream processing — nhúng trực tiếp trong Java/Scala application, **không cần cluster riêng** như Spark Streaming hay Flink.

```
┌─────────────────────────────────────────────────────────┐
│              Kafka Streams Application                   │
│                                                         │
│  Input Topic ──► map/filter ──► aggregate ──► Output Topic│
│                      │              │                    │
│                      ▼              ▼                    │
│                 State Store    State Store              │
│                 (RocksDB)      (changelog topic)          │
└─────────────────────────────────────────────────────────┘
         ▲                                           │
         │         reads/writes                      │
         └──────────── Kafka Cluster ◄───────────────┘
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Embedded library** | Chạy trong app JVM — không separate cluster |
| **Elastic scaling** | Scale bằng cách chạy thêm app instances |
| **Fault tolerant** | State replicated qua changelog topics |
| **Exactly-once** | EOS semantics với Kafka transactions |

---

## Core Concepts

### Stream vs Table (KStream vs KTable)

| Concept | Mô Tả | Analogy |
| ------- | ----- | ------- |
| **KStream** | Stream of events — mỗi record độc lập | Event log |
| **KTable** | Changelog stream → materialized table | Latest state per key |
| **GlobalKTable** | Replicated table toàn bộ trên mỗi instance | Broadcast lookup table |

```
KStream (orders):
  { orderId: 1, status: CREATED }
  { orderId: 1, status: PAID }
  → 2 events riêng biệt

KTable (orders):
  { orderId: 1, status: CREATED }
  { orderId: 1, status: PAID }
  → Table state: { orderId: 1, status: PAID }  (latest wins)
```

### Processor Topology

Application = **DAG (Directed Acyclic Graph — Đồ Thị Không Chu Trình)** of processors:

```
Source ──► Filter ──► Map ──► Aggregate ──► Sink
```

---

## Stream Topology

### Basic Example (Java)

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

orders
  .filter((key, order) -> order.getAmount() > 100000)
  .mapValues(order -> {
    order.setProcessedAt(Instant.now());
    return order;
  })
  .to("high-value-orders");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

### Common Operations

| Operation | Mô Tả |
| --------- | ----- |
| `map` / `mapValues` | Transform record |
| `filter` | Drop records |
| `flatMap` | 1 input → N outputs |
| `branch` | Split stream theo predicate |
| `peek` | Side effect (logging) — không đổi stream |
| `through` | Write to topic rồi read lại |

### DSL vs Processor API

| API | Mô Tả |
| --- | ----- |
| **Streams DSL** | High-level — map, filter, join, aggregate |
| **Processor API** | Low-level — custom Processor, StateStore |

> Hầu hết use case dùng **DSL** — Processor API cho logic đặc biệt.

---

## State Stores

**State Store** lưu state local (RocksDB) — replicated qua **changelog topic** (compacted) để fault recovery.

```
Aggregate count per userId:
  State Store: { user-1: 5, user-2: 3 }
  Changelog topic: user-counts-changelog (compact)
  
Instance crash → new instance restore state từ changelog
```

| Store Type | Use Case |
| ---------- | -------- |
| **KeyValueStore** | Lookup, aggregate |
| **WindowStore** | Time-windowed data |
| **SessionStore** | Session windows |

```java
KTable<String, Long> counts = orders
  .groupByKey()
  .count(Materialized.as("order-counts-store"));
```

### Interactive Queries

Query state store từ app khác (hoặc REST layer):

```java
ReadOnlyKeyValueStore<String, Long> store = streams.store(
  StoreQueryParameters.fromNameAndType("order-counts-store",
    QueryableStoreTypes.keyValueStore())
);
Long count = store.get("user-123");
```

---

## Aggregations & Counting

```java
KStream<String, Order> orders = builder.stream("orders");

KTable<String, Double> revenuePerCustomer = orders
  .groupBy((key, order) -> order.getCustomerId())
  .aggregate(
    () -> 0.0,
    (customerId, order, total) -> total + order.getAmount(),
    Materialized.as("revenue-store")
  );
```

```
Input:
  { customer: A, amount: 100 }
  { customer: A, amount: 50 }
  { customer: B, amount: 200 }

Output KTable:
  A → 150
  B → 200
```

---

## Joins

### Join Types

| Join | Mô Tả |
| ---- | ----- |
| **KStream-KStream** | Join 2 event streams — time window required |
| **KStream-KTable** | Enrich stream với lookup table |
| **KStream-GlobalKTable** | Enrich với broadcast table |
| **KTable-KTable** | Join 2 changelog tables |

### Stream-Table Join (Phổ Biến)

```java
KStream<String, Order> orders = builder.stream("orders");
KTable<String, Customer> customers = builder.table("customers");

KStream<String, EnrichedOrder> enriched = orders.join(
  customers,
  (order, customer) -> new EnrichedOrder(order, customer)
);
```

```
Order event (customerId=42) JOIN Customer table (id=42)
→ EnrichedOrder với customer info

Nếu customer chưa có trong KTable → order bị drop (hoặc null join)
```

### Join Window

Stream-stream join cần **window** — records trong cùng window và cùng key mới join:

```java
orders.join(payments,
  (order, payment) -> new Matched(order, payment),
  JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
);
```

---

## Windowing

**Windowing (Cửa Sổ Thời Gian)** — group events theo khoảng thời gian.

| Window Type | Mô Tả |
| ----------- | ----- |
| **Tumbling** | Fixed, non-overlapping — mỗi 5 phút |
| **Hopping** | Fixed, overlapping — 5 phút window, advance 1 phút |
| **Session** | Activity-based — gap timeout giữa events |
| **Sliding** | Mỗi record trigger window |

```java
orders
  .groupByKey()
  .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
  .count()
  .toStream()
  .to("orders-per-5min");
```

```
Tumbling 5-min:
  [00:00-00:05): count=120
  [00:05-00:10): count=95

Grace period: cho phép late-arriving events vào window đã đóng
```

### Grace Period

```java
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1))
// Window 5 phút + grace 1 phút cho late events
```

---

## Exactly-Once Processing

```properties
processing.guarantee=exactly_once_v2
```

```
EOS v2:
- Transactional producer cho output topics
- Transactional offset commit
- Atomic: process + write output + commit offset

Khi fail → abort transaction → no partial output
```

| processing.guarantee | Semantics |
| -------------------- | --------- |
| `at_least_once` | Default — có thể duplicate |
| `exactly_once_v2` | EOS — Kafka 2.5+ |

> **Trade-off:** EOS tăng latency và broker load — dùng khi cần thiết.

---

## Kafka Streams vs Alternatives

| | Kafka Streams | ksqlDB | Flink | Spark Streaming |
| --- | ------------- | ------ | ----- | --------------- |
| **Deploy** | Embedded JVM app | Server (SQL) | Cluster | Cluster |
| **API** | Java DSL | SQL | Java/Scala/Python | Spark API |
| **State** | Local RocksDB | Internal | Managed | Managed |
| **Best for** | Kafka-native microservices | Quick SQL analytics | Complex CEP | Batch + stream unified |

```
Chọn Kafka Streams khi:
  ✅ Đã dùng Kafka ecosystem
  ✅ Logic trong microservice JVM
  ✅ Không muốn vận hành cluster riêng

Chọn Flink/Spark khi:
  ✅ Complex event processing quy mô lớn
  ✅ Team có data engineering chuyên trách
```

---

## Operations & Monitoring

### Scaling

```
Application instances = consumer group members
Partitions phải đủ để scale:

  6 partitions, 3 instances → mỗi instance ~2 tasks
  6 partitions, 6 instances → mỗi instance 1 task
  6 partitions, 10 instances → 4 idle
```

### Important Metrics

| Metric | Ý Nghĩa |
| ------ | ------- |
| `process-latency-avg` | Thời gian xử lý record |
| `commit-latency-avg` | Offset commit time |
| `rocksdb-bytes-written` | State store I/O |
| Consumer lag per partition | Processing backlog |

### Failure Recovery

```
Instance crash:
1. Tasks rebalance to surviving instances
2. Restore state from changelog topics (có thể mất vài phút)
3. Resume processing from committed offset

→ Changelog topic replication factor ≥ 3 production
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Kafka Streams khác Kafka Consumer thường thế nào?

**Gợi ý trả lời:** Consumer thường: đọc → xử lý → commit. **Kafka Streams**: framework với topology, state stores, joins, windowing, EOS — abstraction cao hơn cho stream processing.

### Câu 2: KStream vs KTable?

**Gợi ý trả lời:** **KStream** — mỗi record là event độc lập. **KTable** — changelog, materialize **latest value per key** — dùng cho lookup/enrichment và aggregations.

### Câu 3: State store lưu ở đâu?

**Gợi ý trả lời:** **Local RocksDB** trên mỗi instance + **changelog topic** (compacted) trên Kafka để replicate/recover. Crash → instance mới rebuild state từ changelog.

### Câu 4: Stream-stream join tại sao cần window?

**Gợi ý trả lời:** Hai event streams vô hạn — cần **bound** records nào được join. Window giới hạn: events cùng key trong khoảng thời gian X mới match.

### Câu 5: exactly_once_v2 trong Streams hoạt động ra sao?

**Gợi ý trả lời:** Dùng **Kafka transactions** — atomic write output topics + commit consumer offsets. Fail → abort, không partial results. Trade-off latency/throughput.

---

**Hoàn thành module Kafka:** Quay lại [README.md](./README.md) hoặc tiếp tục [04-rabbitmq](../04-rabbitmq/README.md).
