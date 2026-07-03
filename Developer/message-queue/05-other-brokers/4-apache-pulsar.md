# Apache Pulsar — Multi-Tenancy & Geo-Replication

> Apache Pulsar — distributed messaging platform: tách storage (Apache BookKeeper) và compute (broker), multi-tenancy native, geo-replication, unified queue + streaming model, Pulsar Functions, và so sánh với Kafka.

## Mục Lục

1. [Tổng Quan Apache Pulsar](#tổng-quan-apache-pulsar)
2. [Kiến Trúc Pulsar](#kiến-trúc-pulsar)
3. [Tenant, Namespace & Topic](#tenant-namespace--topic)
4. [Producers & Consumers](#producers--consumers)
5. [Subscription Types](#subscription-types)
6. [Geo-Replication](#geo-replication)
7. [Tiered Storage](#tiered-storage)
8. [Pulsar Functions](#pulsar-functions)
9. [Kafka Compatibility](#kafka-compatibility)
10. [So Sánh Với Kafka](#so-sánh-với-kafka)
11. [Thiết Kế Thực Tế](#thiết-kế-thực-tế)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Apache Pulsar

**Apache Pulsar** là distributed pub-sub messaging system được thiết kế cho **multi-tenant**, **geo-distributed** deployments với throughput cao.

```
┌─────────────────────────────────────────────────────────────────┐
│                    APACHE PULSAR                                 │
│                                                                 │
│  Điểm khác biệt cốt lõi so với Kafka:                           │
│  - Storage (BookKeeper) tách khỏi serving layer (Broker)          │
│  - Multi-tenancy first-class (tenant/namespace)                 │
│  - Unified model: streaming + queuing cùng API                  │
│  - Geo-replication built-in                                     │
│  - Tiered storage (S3, GCS)                                     │
└─────────────────────────────────────────────────────────────────┘
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Throughput** | Hàng triệu message/giây |
| **Latency** | Vài millisecond |
| **Durability** | BookKeeper replicated storage |
| **Model** | Topic-based pub/sub + subscription modes |
| **License** | Apache 2.0 |

**Origin:** Developed at Yahoo, donated to Apache — được dùng bởi StreamNative, Tencent, Verizon và nhiều enterprise.

---

## Kiến Trúc Pulsar

### Tách Storage và Compute

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Producers   │────►│ Pulsar       │────►│  Consumers   │
│              │     │ Brokers      │     │              │
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ Apache          │
                   │ BookKeeper      │
                   │ (Bookies)       │
                   │ ─ Ledgers ─     │
                   └─────────────────┘
```

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Broker** | Stateless — nhận/proxy message, không lưu data lâu dài |
| **BookKeeper (Bookie)** | Lưu message trong **ledgers** — replicated, durable |
| **ZooKeeper / etcd** | Metadata coordination (cluster config) |
| **Proxy** | Optional — client connection routing |

**Lợi ích tách layer:**

```
✅ Scale storage và compute độc lập
✅ Broker failure — data an toàn trên BookKeeper
✅ Rebalance broker không ảnh hưởng data
✅ Tiered storage offload cold data
```

### Message Flow

```
1. Producer gửi message đến Broker
2. Broker ghi vào BookKeeper ledger (quorum ack)
3. Broker ack producer
4. Consumer request message từ Broker
5. Broker đọc từ BookKeeper (hoặc cache) → deliver
```

---

## Tenant, Namespace & Topic

### Hierarchy

```
Tenant (tổ chức/khách hàng)
  └── Namespace (môi trường/ứng dụng)
        └── Topic (stream/queue)
              └── Partition (optional)
```

**Ví dụ:**

```
persistent://acme-corp/production/orders
           │          │          │
         tenant    namespace   topic
```

| Level | Mô Tả | Ví Dụ |
| ----- | ----- | ----- |
| **Tenant** | Isolation cấp cao nhất — billing, ACL | `acme-corp`, `tenant-a` |
| **Namespace** | Group topics, policies | `production`, `staging` |
| **Topic** | Message stream | `orders`, `payments` |

### Topic Types

| Type | Prefix | Mô Tả |
| ---- | ------ | ----- |
| **Persistent** | `persistent://` | Durable — lưu BookKeeper |
| **Non-persistent** | `non-persistent://` | Ephemeral — không lưu |

### Namespace Policies

```bash
# Retention policy
pulsar-admin namespaces set-retention production/acme-corp \
  --time 7d --size 10GB

# Message TTL
pulsar-admin namespaces set-message-ttl production/acme-corp \
  --messageTTL 86400

# Backlog quota
pulsar-admin namespaces set-backlog-quota production/acme-corp \
  --limit 1GB --policy producer_request_hold
```

**Multi-tenancy native** — mỗi tenant có quota, ACL, retention riêng — phù hợp **SaaS platform**.

---

## Producers & Consumers

### Producer

```java
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://localhost:6650")
    .build();

Producer<String> producer = client.newProducer(Schema.STRING)
    .topic("persistent://public/default/orders")
    .create();

MessageId msgId = producer.send("order-123-created");
```

| Feature | Mô Tả |
| ------- | ----- |
| **Batching** | Gộp message — tăng throughput |
| **Compression** | LZ4, ZLIB, ZSTD |
| **Message routing** | Single partition, round-robin, custom |
| **Delayed delivery** | Schedule message tương lai |

### Consumer & Subscription

```java
Consumer<String> consumer = client.newConsumer(Schema.STRING)
    .topic("persistent://public/default/orders")
    .subscriptionName("fulfillment-service")
    .subscriptionType(SubscriptionType.Shared)
    .subscribe();

Message<String> msg = consumer.receive();
// process...
consumer.acknowledge(msg);
```

---

## Subscription Types

**Subscription (Đăng Ký)** — cách consumer nhận message từ topic.

| Type | Mô Tả | Tương Đương |
| ---- | ----- | ----------- |
| **Exclusive** | Một consumer duy nhất per subscription | Dedicated queue |
| **Shared** | Nhiều consumer, round-robin | Kafka consumer group competing |
| **Failover** | Một active, còn lại standby | Active-passive HA |
| **Key_Shared** | Cùng key → cùng consumer (ordering per key) | Kafka partition key |

```
Topic "orders" (4 partitions)
        │
Subscription "fulfillment" (Shared)
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
  C1   C2   C3   C4   ← round-robin messages

Subscription "analytics" (Shared)  ← consumer group KHÁC
   │
   ▼
  All messages (pub/sub — mỗi subscription nhận full copy)
```

**Quan trọng:** Mỗi **subscription** là một **cursor (con trỏ)** độc lập trên topic — giống Kafka consumer groups.

### Key_Shared — Ordering Per Key

```java
consumer.newConsumer()
    .subscriptionType(SubscriptionType.Key_Shared)
    .keySharedPolicy(KeySharedPolicy.autoSplitHashRange())
```

Message cùng **partition key** luôn đến cùng consumer — ordering per entity.

---

## Geo-Replication

**Geo-Replication (Nhân Bản Địa Lý)** — replicate topic across Pulsar clusters ở regions khác nhau.

```
┌─────────────────┐         ┌─────────────────┐
│ Cluster US-EAST │◄───────►│ Cluster EU-WEST │
│                 │  async  │                 │
│ Topic: orders   │  repl   │ Topic: orders   │
└────────┬────────┘         └────────┬────────┘
         │                           │
    Producers US                 Consumers EU
```

### Cấu Hình

```bash
# Tạo namespace với replication clusters
pulsar-admin namespaces set-clusters production/acme-corp \
  --clusters us-east, eu-west, ap-southeast
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Async replication** | Cross-region latency |
| **Local produce/consume** | Low latency trong region |
| **Global topic** | Cùng tên topic trên mọi cluster |
| **Conflict** | Per-partition ordering — design key carefully |

**So với Kafka:** Pulsar geo-replication **built-in**; Kafka cần MirrorMaker 2 hoặc Confluent Cluster Linking.

### Use Cases

```
✅ Global SaaS — users gần region gần nhất
✅ Disaster recovery — failover region
✅ Data locality compliance — data ở EU, US riêng
```

---

## Tiered Storage

**Tiered Storage (Lưu Trữ Phân Tầng)** — offload old data từ BookKeeper sang object storage.

```
Hot data (recent)     ──► BookKeeper (fast, expensive)
Cold data (old)       ──► S3 / GCS / Azure Blob (cheap)
```

```bash
pulsar-admin namespaces set-offload-policies production/acme-corp \
  --driver aws-s3 \
  --bucket pulsar-offload \
  --region ap-southeast-1 \
  --rubric MAX_SIZE\=1GB
```

**Lợi ích:**

- Retention dài (months/years) với cost thấp
- BookKeeper chỉ giữ hot data — giảm storage pressure
- Consumer vẫn đọc được historical data (transparent)

---

## Pulsar Functions

**Pulsar Functions** — lightweight stream processing **trong Pulsar**, không cần Flink/Spark riêng.

```
Topic A ──► Pulsar Function ──► Topic B
              (transform, filter, aggregate)
```

```java
// Java function
public class OrderEnricher implements Function<Order, EnrichedOrder> {
    @Override
    public EnrichedOrder process(Order input, Context context) {
        return enrich(input);
    }
}
```

```bash
pulsar-admin functions create \
  --jar functions.jar \
  --classname com.example.OrderEnricher \
  --inputs persistent://public/default/orders \
  --output persistent://public/default/orders-enriched
```

| So Với | Pulsar Functions | Kafka Streams |
| ------ | ---------------- | ------------- |
| **Deployment** | Pulsar cluster | Separate app |
| **Complexity** | Đơn giản transforms | Full stream processing |
| **State** | Limited | Rich state stores |

**Phù hợp:** Simple ETL, enrichment, routing — không thay Flink cho complex analytics.

---

## Kafka Compatibility

Pulsar hỗ trợ **Kafka protocol** — Kafka clients có thể connect Pulsar.

```
Kafka Producer/Consumer ──► Pulsar Broker (Kafka protocol handler)
                                    │
                              BookKeeper storage
```

| Feature | Support |
| ------- | ------- |
| **Kafka API** | Produce, consume cơ bản |
| **Consumer groups** | Mapped to Pulsar subscriptions |
| **Transactions** | Limited |
| **Kafka Connect** | Qua Pulsar IO |

**Migration path:** Chạy Pulsar làm backend, giữ Kafka client code — migrate dần sang Pulsar native API.

---

## So Sánh Với Kafka

| Tiêu Chí | Apache Pulsar | Apache Kafka |
| -------- | ------------- | ------------ |
| **Architecture** | Broker + BookKeeper | Broker-centric log |
| **Multi-tenancy** | ✅ Native | ⚠️ Manual (prefix, ACL) |
| **Geo-replication** | ✅ Built-in | ⚠️ MirrorMaker |
| **Queue model** | ✅ Shared subscription | Consumer group only |
| **Tiered storage** | ✅ Native | ✅ (recent versions) |
| **Ecosystem** | Nhỏ hơn | Rất lớn |
| **Ops complexity** | Cao (3 components) | Cao (KRaft simpler now) |
| **Community** | Growing | Mature, larger |
| **Managed** | StreamNative, Tencent | MSK, Confluent Cloud |

### Khi Nào Chọn Pulsar

```
✅ Multi-tenant SaaS platform
✅ Geo-replication across regions native
✅ Unified queue + streaming
✅ Long retention với tiered storage cost-effective
✅ Kafka migration với protocol compatibility

❌ Team chưa có Pulsar experience
❌ Cần ecosystem Kafka (Connect, ksqlDB) đầy đủ
❌ Simple task queue — overkill
```

---

## Thiết Kế Thực Tế

### Pattern: Multi-Tenant SaaS Event Bus

```
Tenant A ──► persistent://tenant-a/prod/events
Tenant B ──► persistent://tenant-b/prod/events
                    │
            Namespace policies:
            - Retention per tenant plan
            - Quota: throughput, storage
            - ACL isolation
```

### Pattern: Global Order System

```
US Producers ──► US Cluster ──geo-repl──► EU Cluster ◄── EU Consumers
                      │                           │
                      └─────── ap-southeast ──────┘
```

### Checklist Production

```
□ BookKeeper ensemble 3+ bookies, quorum write
□ Namespace retention & TTL configured
□ Subscription type phù hợp (Shared vs Key_Shared)
□ Geo-replication monitor lag cross-cluster
□ Tiered storage cho long retention
□ Authentication (JWT, TLS)
□ Monitor: backlog, publish/consume rate, storage
□ Dead letter topic cho failed messages (custom)
```

### Anti-Patterns

```
❌ Dùng Pulsar cho simple cron job queue
❌ Key_Shared không set key — mất ordering benefit
❌ Geo-replication không monitor cross-region lag
❌ Ignore backlog quota — producer block hoặc drop
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Pulsar architecture khác Kafka thế nào?

**Đáp án mẫu:** **Pulsar** tách **broker** (stateless serving) và **BookKeeper** (durable storage). Broker có thể scale/restart mà data an toàn trên bookies. **Kafka** broker vừa serve vừa lưu partition log. Pulsar linh hoạt scale storage/compute độc lập; Kafka đơn giản hơn về component count với KRaft.

### Câu 2: Multi-tenancy trong Pulsar hoạt động ra sao?

**Đáp án mẫu:** **Tenant** → **Namespace** → **Topic** hierarchy. Mỗi tenant có ACL, quota (throughput, storage), retention policy riêng. Phù hợp SaaS — mỗi khách hàng một tenant, isolate billing và resources. Kafka cần manual prefix topic + ACL phức tạp hơn.

### Câu 3: Subscription types — Shared vs Key_Shared?

**Đáp án mẫu:** **Shared** — round-robin, không ordering guarantee. **Key_Shared** — message cùng key đến cùng consumer, ordering per key + parallel across keys. Chọn Key_Shared khi cần per-entity ordering (order per customer) mà vẫn scale nhiều consumer.

### Câu 4: Geo-replication Pulsar vs Kafka MirrorMaker?

**Đáp án mẫu:** **Pulsar** — namespace-level config, async replication built-in, global topic name. **Kafka** — MirrorMaker 2 separate tool, config phức tạp, offset mapping manual. Pulsar advantage cho multi-region by design; Kafka mature hơn với tooling ecosystem.

### Câu 5: Khi nào chọn Pulsar thay Kafka?

**Đáp án mẫu:** Multi-tenant SaaS, geo-replication native, unified queue+stream, tiered storage cost optimization, Kafka protocol migration path. **Không chọn** khi team đã invest Kafka ecosystem (Connect, Streams, ksqlDB), hoặc use case đơn giản không cần Pulsar features — ops overhead không justify.

---

**Hoàn thành chủ đề 05-other-brokers.** Tiếp theo: [06-reliability/README.md](../06-reliability/README.md) — Độ tin cậy & xử lý lỗi.
