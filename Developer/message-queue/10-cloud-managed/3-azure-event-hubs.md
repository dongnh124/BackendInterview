# Azure Event Hubs — Event Ingestion Trên Azure

> Azure Event Hubs: big data streaming platform trên Azure, Kafka-compatible endpoint, Capture to Blob Storage, throughput units (TU — Đơn Vị Thông Lượng), và tích hợp Azure ecosystem (Stream Analytics, Azure Functions, Spark).

## Mục Lục

1. [Tổng Quan Event Hubs](#tổng-quan-event-hubs)
2. [Event Hubs vs Kafka vs Service Bus](#event-hubs-vs-kafka-vs-service-bus)
3. [Tiers & Throughput Units](#tiers--throughput-units)
4. [Partitions & Consumer Groups](#partitions--consumer-groups)
5. [Kafka Endpoint Compatibility](#kafka-endpoint-compatibility)
6. [Capture Feature](#capture-feature)
7. [Security & Networking](#security--networking)
8. [Tích Hợp Azure Ecosystem](#tích-hợp-azure-ecosystem)
9. [Migration & Best Practices](#migration--best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Event Hubs

**Azure Event Hubs** là fully managed **event ingestion (thu thập sự kiện)** service — nhận hàng triệu events/giây và phân phối cho consumers hoặc storage.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AZURE EVENT HUBS ARCHITECTURE                            │
│                                                                             │
│  ┌──────────────┐     ┌─────────────────────────────────────────────┐      │
│  │  Producers   │────►│  Event Hubs Namespace                       │      │
│  │  (SDK, Kafka │     │  ┌─────────────────────────────────────┐    │      │
│  │   endpoint,  │     │  │ Event Hub (tương đương Kafka topic) │    │      │
│  │   REST)      │     │  │ • Partitions (1–32 Standard, 100 Premium)│   │      │
│  └──────────────┘     │  │ • Retention (1–7 ngày Standard)     │    │      │
│                       │  │ • Capture → Blob (optional)         │    │      │
│  ┌──────────────┐     │  └─────────────────────────────────────┘    │      │
│  │  Consumers   │◄────│                                             │      │
│  │  (SDK, Kafka,│     └─────────────────────────────────────────────┘      │
│  │   Stream     │                                                           │
│  │   Analytics) │                                                           │
│  └──────────────┘                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Khái Niệm Azure | Tương Đương Kafka |
| --------------- | ----------------- |
| **Namespace** | Cluster |
| **Event Hub** | Topic |
| **Partition** | Partition |
| **Consumer Group** | Consumer Group |
| **Offset** | Offset (Event Hub SDK) hoặc Kafka offset |

---

## Event Hubs vs Kafka vs Service Bus

Azure có **ba messaging services** — chọn đúng là quan trọng:

| | Event Hubs | Azure Service Bus | Apache Kafka |
| - | ---------- | ----------------- | ------------ |
| **Model** | Event streaming / log | Message queue + topics | Event log |
| **Throughput** | Rất cao (millions/sec) | Trung bình | Rất cao |
| **Ordering** | Per partition | FIFO sessions (Premium) | Per partition |
| **Retention** | 1–90 ngày (tier-dependent) | Until consumed/deleted | Configurable |
| **Replay** | ✅ (retention window) | ⚠️ Limited | ✅ |
| **Use case** | Telemetry, analytics, IoT | Enterprise messaging, workflows | Event streaming platform |

```
Event Hubs     → Big data ingestion, analytics pipeline, IoT telemetry
Service Bus    → Order processing, workflow, request-reply, DLQ native
Kafka (HDInsight/self-hosted) → Full Kafka ecosystem khi cần
```

**Event Hubs Premium** thêm AMQP protocol — gần Service Bus hơn cho enterprise messaging patterns.

---

## Tiers & Throughput Units

### Pricing Tiers

| Tier | Throughput | Partitions | Retention | Capture |
| ---- | ---------- | ---------- | --------- | ------- |
| **Basic** | 1 MB/s ingress | 32 max | Không configurable | ❌ |
| **Standard** | TU-based, 1 MB/s per TU | 32 per hub | 1–7 ngày | ✅ |
| **Premium** | Processing Units (PU) | 100 per hub | 1–90 ngày | ✅ |
| **Dedicated** | Isolated capacity | Custom | Custom | ✅ |

### Throughput Units (TU)

**TU (Throughput Unit — Đơn Vị Thông Lượng)** — đơn vị capacity trên Standard tier:

```
1 TU = 1 MB/s ingress HOẶC 2 MB/s egress (whichever hit first)
       + 1000 brokered connections
       + 84 GB storage (retention included)

Scale: 1–40 TU per namespace (auto-inflate available)
```

### Premium Processing Units (PU)

```
Dedicated compute và storage — không noisy neighbor
1 PU ≈ higher throughput và partition count
Phù hợp: enterprise SLA, low latency, zone redundancy
```

### Auto-Inflate

```
Namespace tự động tăng TU khi traffic spike
Max TU cap configurable — tránh cost runaway
```

---

## Partitions & Consumer Groups

### Partition Design

```
Event Hub partitions = parallelism unit

Rules:
• Partition count FIXED at creation — không giảm, tăng có giới hạn
• 1 partition = max 1 active consumer per consumer group
• Message ordering guaranteed WITHIN partition only
• Partition key → hash → same partition (ordering per key)
```

```csharp
// .NET SDK — send với partition key
await producer.SendAsync(new EventData(eventBody), partitionKey: "order-12345");
// Tất cả events order-12345 → cùng partition → ordered
```

### Consumer Groups

```
Mỗi consumer group đọc independently — giống Kafka
Default: $Default (chỉ 1 consumer app nên dùng)
Tạo riêng cho mỗi downstream: analytics-group, notification-group

⚠️ Max 5 consumer groups per Event Hub (Standard)
   Premium: 1000 consumer groups
```

### Offset Management

```csharp
// Checkpoint — lưu vị trí đọc (tương đương Kafka offset commit)
// Lưu trong Azure Blob Storage (EventProcessorClient)

var processor = new EventProcessorClient(
    blobContainerClient,
    "$Default",
    connectionString,
    eventHubName
);

processor.ProcessEventAsync += async (args) => {
    // Process event
    await ProcessEvent(args.Data);
    // Checkpoint sau khi xử lý xong
    await args.UpdateCheckpointAsync();
};
```

---

## Kafka Endpoint Compatibility

Event Hubs cung cấp **Kafka protocol endpoint** — dùng Kafka clients mà không đổi code nhiều.

### Setup Kafka Client

```properties
# Kafka producer/consumer config pointing to Event Hubs
bootstrap.servers=<namespace>.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$ConnectionString" \
  password="Endpoint=sb://<namespace>.servicebus.windows.net/;...";
```

### Compatibility Matrix

| Kafka Feature | Event Hubs Support |
| ------------- | ------------------ |
| Produce/Consume | ✅ |
| Consumer groups | ✅ |
| Partition assignment | ✅ |
| Idempotent producer | ⚠️ Limited |
| Transactions | ❌ |
| Kafka Connect | ❌ (dùng Azure alternatives) |
| Kafka Streams | ⚠️ Limited — test carefully |
| Admin API (create topic) | ⚠️ Pre-create Event Hub in Azure |

**Lưu ý quan trọng:** Kafka endpoint là **compatibility layer** — không phải Kafka 100%. Test kỹ trước khi migrate production workloads phức tạp.

### Khi Dùng Kafka Endpoint

```
✅ Existing Kafka clients — minimize code change
✅ Simple produce/consume patterns
✅ Mirror from existing Kafka cluster

❌ Kafka Connect, Kafka Streams phức tạp
❌ Exactly-once transactions
❌ Admin operations via Kafka API
```

---

## Capture Feature

**Capture (Thu Thập)** — tự động export events sang **Azure Blob Storage** hoặc **Azure Data Lake Storage Gen2** — data lake ingestion không cần consumer code.

```
Event Hub ──► Capture (auto) ──► Blob Storage
                    │
                    ├── Format: Avro (default) hoặc AvroDeflate
                    ├── Path: {Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}
                    └── Interval: 60–900 seconds (hoặc size-based)
```

### Capture Configuration

| Setting | Mô Tả |
| ------- | ----- |
| **Time window** | Capture mỗi N giây (min 60, max 900) |
| **Size window** | Capture khi đủ N MB |
| **Destination** | Blob container hoặc ADLS Gen2 |
| **Archive naming** | `{Namespace}/{EventHub}/{PartitionId}/...` |

### Use Cases

```
• Long-term storage beyond Event Hub retention
• Batch analytics với Spark/Synapse trên captured files
• Compliance archive
• Cold path analytics (không cần real-time consumer)
```

```
Hot path:  Event Hub → Stream Analytics → Power BI (real-time)
Cold path: Event Hub → Capture → Blob → Spark/Synapse (batch)
```

---

## Security & Networking

### Authentication

| Method | Mô Tả |
| ------ | ----- |
| **Connection String** | Shared access — dev/simple apps |
| **SAS (Shared Access Signature — Chữ Ký Truy Cập Chia Sẻ)** | Scoped permissions, time-limited |
| **Microsoft Entra ID (Azure AD)** | OAuth — recommended production |
| **Managed Identity** | Azure resources authenticate without secrets |

```csharp
// Managed Identity — không hardcode connection string
var credential = new DefaultAzureCredential();
var producer = new EventHubProducerClient(
    fullyQualifiedNamespace,
    eventHubName,
    credential
);
```

### Authorization (RBAC)

```
Azure RBAC roles:
• Azure Event Hubs Data Owner — full access
• Azure Event Hubs Data Sender — produce only
• Azure Event Hubs Data Receiver — consume only
```

### Network Isolation

```
• Private Endpoint — traffic qua Azure backbone, không public internet
• Service Endpoints — restrict access từ specific VNet
• IP Firewall rules — allowlist IP ranges
• TLS 1.2+ enforced
```

---

## Tích Hợp Azure Ecosystem

### Azure Stream Analytics

```sql
-- Stream Analytics query trên Event Hub input
SELECT
    customer_id,
    COUNT(*) AS event_count,
    System.Timestamp() AS window_end
INTO
    [output-powerbi]
FROM
    [input-eventhub]
TIMESTAMP BY event_time
GROUP BY customer_id, TumblingWindow(minute, 5)
```

### Azure Functions — Event Hub Trigger

```csharp
[FunctionName("ProcessOrder")]
public static async Task Run(
    [EventHubTrigger("orders", Connection = "EventHubConnection")] EventData[] events,
    ILogger log)
{
    foreach (var eventData in events)
    {
        var order = JsonSerializer.Deserialize<Order>(eventData.Body);
        await ProcessOrder(order);
    }
}
```

| Integration | Pattern |
| ----------- | ------- |
| **Event Hubs → Functions** | Event-driven serverless processing |
| **Event Hubs → Stream Analytics** | Real-time SQL analytics |
| **Event Hubs → Synapse/Spark** | Batch + streaming analytics |
| **Event Hubs → Logic Apps** | Workflow automation |
| **Event Hubs → Power BI** | Real-time dashboards |

### Event Grid Integration

```
Event Hubs có thể publish availability events → Event Grid → downstream triggers
Use case: alert khi Capture file ready, consumer lag events
```

---

## Migration & Best Practices

### Migration từ Kafka

```
Phase 1: Tạo Event Hub với partition count tương đương
Phase 2: Dual-write — producers gửi cả Kafka và Event Hubs
Phase 3: Migrate consumers — test với Kafka endpoint hoặc native SDK
Phase 4: Validate ordering, latency, error handling
Phase 5: Cutover producers, decommission Kafka
```

### Design Best Practices

```
1. Partition count = expected max parallelism (khó thay đổi sau)
2. Partition key = business entity ID (orderId, customerId) cho ordering
3. Separate consumer groups per downstream service
4. Enable Capture cho long-term storage — không rely chỉ retention
5. Managed Identity thay connection strings
6. Monitor TU/PU utilization — scale trước khi throttle
7. Checkpoint sau successful processing — at-least-once semantics
8. Idempotent consumers — duplicate possible on retry
```

### Monitoring

| Metric | Azure Monitor | Action |
| ------ | ------------- | ------ |
| **Incoming Messages** | Namespace level | Baseline traffic |
| **Incoming Bytes** | Per Event Hub | TU utilization |
| **Throttled Requests** | Namespace | Scale TU/PU |
| **Server Errors** | Namespace | Investigate |
| **Capture backlog** | Per hub | Check storage destination |

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Event Hubs khác Service Bus? | Event Hubs = high throughput streaming; Service Bus = enterprise messaging, DLQ, sessions |
| TU là gì? | 1 MB/s ingress hoặc 2 MB/s egress + connections + storage |
| Kafka endpoint có phải Kafka thật? | Compatibility layer — không 100% feature parity |
| Capture dùng để làm gì? | Auto-export to Blob/ADLS — data lake, long-term archive |
| Partition count có đổi được không? | Fixed at creation — plan carefully |
| Checkpoint vs offset? | Checkpoint lưu trong Blob; tương đương Kafka offset commit |
| Event Hubs vs MSK? | Event Hubs Azure-native; MSK full Kafka; chọn theo cloud và ecosystem |

---

## Liên Kết Liên Quan

| Chủ Đề | File |
| ------ | ---- |
| Kafka partitions | [03-apache-kafka/2-topics-partitions.md](../03-apache-kafka/2-topics-partitions.md) |
| Consumer groups | [03-apache-kafka/3-consumer-groups.md](../03-apache-kafka/3-consumer-groups.md) |
| Amazon MSK | [1-aws-msk.md](./1-aws-msk.md) |
| GCP Pub/Sub | [4-gcp-pubsub.md](./4-gcp-pubsub.md) |
| Cloud overview | [README.md](./README.md) |

---

**Cập Nhật:** 2026-07-03
