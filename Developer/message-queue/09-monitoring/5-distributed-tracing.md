# Distributed Tracing — Truy Vết Phân Tán Cho Messaging

> Correlation ID (ID Tương Quan), trace context propagation (lan truyền ngữ cảnh dấu vết), và OpenTelemetry (OTel — Chuẩn Mở Quan Sát) cho end-to-end visibility (khả năng quan sát đầu-cuối) qua message brokers.

## Mục Lục

1. [Tại Sao Cần Tracing Cho Messaging](#tại-sao-cần-tracing-cho-messaging)
2. [Correlation ID vs Distributed Tracing](#correlation-id-vs-distributed-tracing)
3. [Trace Context Propagation Qua Broker](#trace-context-propagation-qua-broker)
4. [OpenTelemetry Cho Messaging](#opentelemetry-cho-messaging)
5. [Implementation Patterns](#implementation-patterns)
6. [Kafka Headers & RabbitMQ Properties](#kafka-headers--rabbitmq-properties)
7. [Trace-Based Debugging Workflow](#trace-based-debugging-workflow)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Tracing Cho Messaging

Metrics cho biết **lag bao nhiêu**; logs cho biết **lỗi gì**; traces cho biết **message đi đâu, mất bao lâu ở đâu**.

```
HTTP Request (có trace)          Messaging (không trace)
─────────────────────────        ─────────────────────────────
API Gateway ──► Service A        Producer ──► ??? ──► Consumer ──► ???
     │                │              │                         │
     └── trace_id ────┘              └── black box ────────────┘

Với tracing:
Producer ──[trace_id: abc-123]──► Broker ──[trace_id: abc-123]──► Consumer ──► DB
         span: produce                    span: consume              span: db_write
```

| Vấn Đề Không Có Trace | Trace Giải Quyết |
| --------------------- | ---------------- |
| "Order X chậm — lỗi ở đâu?" | Xem span breakdown: produce 5ms, queue 30s, process 200ms |
| "Message duplicate từ đâu?" | Trace ID xuất hiện 2 lần → identify retry source |
| "Cross-service debug" | Một trace_id link tất cả services |
| "SLA breach root cause" | P99 latency ở queue wait vs processing |

---

## Correlation ID vs Distributed Tracing

| Khái Niệm | Mô Tả | Độ Phức Tạp |
| --------- | ----- | ----------- |
| **Correlation ID** | Single ID gắn với business transaction | Thấp — log grep |
| **Distributed Tracing** | Tree of spans với parent-child relationships | Cao — Jaeger/Zipkin UI |

### Correlation ID (Minimum Viable)

```json
// Message payload hoặc header
{
  "correlationId": "order-2024-abc-123",
  "eventType": "OrderCreated",
  "payload": { "orderId": "abc-123" }
}
```

```javascript
// Producer
const correlationId = req.headers['x-correlation-id'] || uuid();
logger.info({ correlationId, action: 'publishing' }, 'Publishing order event');
await producer.send({
  headers: { 'x-correlation-id': correlationId },
  value: JSON.stringify({ correlationId, ...order })
});

// Consumer
const correlationId = message.headers['x-correlation-id'];
logger.info({ correlationId, action: 'consuming' }, 'Processing order event');
```

**Ưu:** Đơn giản, hoạt động với mọi broker  
**Nhược:** Chỉ log correlation, không có timing breakdown

### Distributed Tracing (Full Observability)

```
Trace: order-abc-123
├── Span: HTTP POST /orders (API Gateway)     50ms
│   └── Span: produce OrderCreated (Producer)  5ms
│       └── Span: broker store (Kafka)         2ms
│           └── Span: consume OrderCreated     3ms
│               └── Span: process order        150ms
│                   └── Span: DB insert         80ms
│                   └── Span: publish Payment   5ms
```

**Ưu:** Timing breakdown, visual UI, dependency map  
**Nhược:** Cần OpenTelemetry SDK, trace backend (Jaeger, Tempo)

---

## Trace Context Propagation Qua Broker

W3C **Trace Context** standard định nghĩa cách propagate trace qua async boundaries:

```
┌─────────────────────────────────────────────────────────────────┐
│              TRACE CONTEXT PROPAGATION FLOW                        │
│                                                                  │
│  Producer                    Broker                 Consumer     │
│  ┌──────────┐              ┌────────┐            ┌──────────┐  │
│  │ Active   │──inject────►│ Message│──extract──►│ Continue │  │
│  │ Span     │  headers    │ Headers│  headers   │ Span     │  │
│  └──────────┘              └────────┘            └──────────┘  │
│                                                                  │
│  Headers propagated:                                             │
│  traceparent: 00-{trace-id}-{span-id}-01                        │
│  tracestate: vendor-specific-data                                │
│  baggage: key-value context (user-id, tenant-id)                │
└─────────────────────────────────────────────────────────────────┘
```

### Inject (Producer) / Extract (Consumer)

```javascript
const { propagation, context, trace } = require('@opentelemetry/api');

// Producer — inject trace context vào message headers
async function publishWithTrace(producer, topic, message) {
  const activeSpan = trace.getActiveSpan();
  const headers = {};
  
  propagation.inject(context.active(), headers, {
    set: (carrier, key, value) => { carrier[key] = value; }
  });

  await producer.send({
    topic,
    messages: [{
      key: message.orderId,
      value: JSON.stringify(message),
      headers: Object.entries(headers).map(([k, v]) => ({ key: k, value: Buffer.from(v) }))
    }]
  });
}

// Consumer — extract trace context từ message headers
async function consumeWithTrace(message) {
  const headers = {};
  message.headers.forEach(h => { headers[h.key] = h.value.toString(); });

  const parentContext = propagation.extract(context.active(), headers, {
    get: (carrier, key) => carrier[key]
  });

  return context.with(parentContext, async () => {
    const span = tracer.startSpan('consume OrderCreated');
    try {
      await processOrder(JSON.parse(message.value));
    } finally {
      span.end();
    }
  });
}
```

---

## OpenTelemetry Cho Messaging

**OpenTelemetry (OTel — Chuẩn Mở Quan Sát)** là standard cho instrumentation, cung cấp SDK cho mọi ngôn ngữ và exporter cho Jaeger, Zipkin, Datadog, etc.

### Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Application │  │ OTel SDK    │  │ OTel        │
│ Code        │─►│ (auto/manual│─►│ Collector   │
│             │  │ instrument) │  │             │
└─────────────┘  └─────────────┘  └──────┬──────┘
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                    ┌──────────┐  ┌──────────┐  ┌──────────┐
                    │ Jaeger   │  │ Grafana  │  │ Datadog  │
                    │          │  │ Tempo    │  │          │
                    └──────────┘  └──────────┘  └──────────┘
```

### Semantic Conventions Cho Messaging

OpenTelemetry định nghĩa **semantic conventions** cho messaging spans:

| Attribute | Ví Dụ | Mô Tả |
| --------- | ----- | ----- |
| `messaging.system` | `kafka`, `rabbitmq` | Broker type |
| `messaging.destination` | `orders` | Topic/queue name |
| `messaging.operation` | `publish`, `receive`, `process` | Operation type |
| `messaging.message.id` | `0:12345:67890` | Unique message ID |
| `messaging.kafka.partition` | `3` | Kafka partition |
| `messaging.kafka.consumer.group` | `order-processor` | Consumer group |

### Node.js OpenTelemetry Setup

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { KafkaJsInstrumentation } = require('@opentelemetry/instrumentation-kafkajs');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

const sdk = new NodeSDK({
  serviceName: 'order-processor',
  traceExporter: new JaegerExporter({ endpoint: 'http://jaeger:14268/api/traces' }),
  instrumentations: [
    new KafkaJsInstrumentation({
      producerHook: (span, payload) => {
        span.setAttribute('messaging.destination', payload.topic);
      },
      consumerHook: (span, payload) => {
        span.setAttribute('messaging.kafka.partition', payload.partition);
      }
    })
  ]
});

sdk.start();
```

---

## Implementation Patterns

### Pattern 1: Correlation ID Only (Đơn Giản)

```
Phù hợp: Team nhỏ, ít services, chỉ cần log correlation

Flow:
1. API nhận request → generate correlationId
2. Gắn vào message header
3. Consumer log với correlationId
4. Log aggregation (ELK/Loki) search by correlationId
```

### Pattern 2: W3C Trace Context (Khuyến Nghị)

```
Phù hợp: Microservices, cần timing breakdown

Flow:
1. HTTP request có trace context (từ API gateway)
2. Producer inject traceparent vào Kafka headers
3. Consumer extract → tạo child span
4. Downstream calls tiếp tục propagate
5. Jaeger UI hiển thị full trace tree
```

### Pattern 3: Baggage Cho Business Context

```javascript
// Propagate business context qua async boundary
const { baggage } = require('@opentelemetry/api');

const ctx = baggage.setEntry(
  context.active(),
  'tenant-id',
  { value: 'tenant-abc' }
);

// Baggage tự động propagate qua message headers
// Consumer đọc tenant-id từ baggage
const tenantId = baggage.getEntry(ctx, 'tenant-id')?.value;
```

### Pattern 4: Trace Links Cho Fan-Out

Khi một message produce nhiều downstream messages (fan-out), dùng **span links** thay vì parent-child:

```
Span: consume OrderCreated
  ├── Span: produce PaymentRequested (link, not child)
  ├── Span: produce InventoryReserved (link)
  └── Span: produce NotificationSent (link)
```

---

## Kafka Headers & RabbitMQ Properties

### Kafka — Message Headers

```java
// Producer
ProducerRecord<String, String> record = new ProducerRecord<>("orders", orderId, payload);
record.headers().add("traceparent", traceParent.getBytes());
record.headers().add("x-correlation-id", correlationId.getBytes());
producer.send(record);

// Consumer
Headers headers = record.headers();
String traceParent = new String(headers.lastHeader("traceparent").value());
// Extract và continue trace
```

### RabbitMQ — AMQP Properties

```javascript
// Publisher
channel.publish('orders', routingKey, Buffer.from(payload), {
  headers: {
    'traceparent': traceParent,
    'x-correlation-id': correlationId
  },
  correlationId: correlationId,  // Built-in AMQP property
  messageId: uuid(),
  timestamp: Date.now()
});

// Consumer
const traceParent = msg.properties.headers['traceparent'];
const correlationId = msg.properties.correlationId;
```

| Broker | Mechanism | Built-in Fields |
| ------ | --------- | --------------- |
| **Kafka** | Record headers (key-value) | Không có built-in correlation |
| **RabbitMQ** | AMQP properties + headers | `correlationId`, `messageId` |
| **SQS** | Message attributes | `AWSTraceHeader` (X-Ray) |
| **NATS** | Message headers | Custom headers |

---

## Trace-Based Debugging Workflow

### Scenario: Order Processing Chậm

```
1. User report: "Order abc-123 chưa được xử lý sau 10 phút"

2. Search Jaeger by tag:
   messaging.message.id = "abc-123"
   hoặc search logs: correlationId = "abc-123"

3. Trace timeline:
   ├── produce OrderCreated     2ms   ✅
   ├── [GAP — 9 phút 58 giây]         ⚠️ ← bottleneck ở đây
   ├── consume OrderCreated     3ms   ✅
   └── process order           150ms  ✅

4. Diagnosis: 10 phút gap = consumer lag (queue wait)
   → Check consumer lag metrics
   → Không phải processing slow

5. Resolution: Scale consumers
```

### Scenario: Duplicate Processing

```
1. Search traces với business ID
2. Thấy 2 consume spans cùng trace_id hoặc correlation_id
3. Check span attributes:
   - Cùng offset? → Rebalance replay
   - Khác offset? → At-least-once duplicate
4. Verify idempotency key trong processing span
```

---

## Best Practices

| Practice | Chi Tiết |
| -------- | -------- |
| **Always propagate correlation ID** | Minimum — mọi message phải có correlation ID |
| **Use W3C traceparent** | Standard, interoperable across services |
| **Sample wisely** | 100% dev; 1–10% production (head-based hoặc tail-based) |
| **Don't trace broker internals** | Trace application spans, không trace Kafka broker |
| **Include business IDs in spans** | `orderId`, `userId` as span attributes (not labels) |
| **Baggage cho tenant context** | Multi-tenant: propagate tenant-id qua baggage |
| **Link traces to logs** | Log `trace_id` → click từ log sang trace |

### Sampling Strategy

```yaml
# Production sampling — balance cost vs visibility
sampler:
  type: parentbased_traceidratio
  arg: 0.1  # 10% traces

# Always sample errors (tail-based sampling via OTel Collector)
processors:
  tail_sampling:
    policies:
      - name: errors
        type: status_code
        status_code: ERROR
      - name: slow
        type: latency
        threshold_ms: 5000
```

---

## Câu Hỏi Phỏng Vấn

**Q: Correlation ID vs trace ID — khác nhau thế nào?**

> Correlation ID là business-level identifier (order ID, request ID) để grep logs. Trace ID là technical identifier cho distributed tracing với span tree và timing. Production nên có cả hai — correlation ID cho business debug, trace ID cho performance analysis.

**Q: Làm sao propagate trace context qua Kafka?**

> Inject W3C traceparent header khi produce. Consumer extract header và tạo child span. OpenTelemetry SDK có sẵn propagation API và auto-instrumentation cho kafkajs, spring-kafka.

**Q: Tracing async messaging khác HTTP tracing thế nào?**

> HTTP: parent span chờ child (sync). Messaging: producer span kết thúc trước consumer span bắt đầu (async). Dùng span links cho fan-out. Gap giữa produce và consume span = queue wait time = consumer lag visualization.

**Q: Sampling 100% traces trong production có OK không?**

> Thường không — storage cost cao. Dùng 1–10% head-based sampling + tail-based sampling cho errors/slow traces. Critical paths (payment) có thể 100%.

---

**Tiếp theo:** [6-troubleshooting-playbook.md](./6-troubleshooting-playbook.md) — Sổ tay xử lý sự cố messaging.
