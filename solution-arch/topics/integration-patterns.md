# Integration Patterns

**Related:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/message-queues]], [[solution-arch/patterns/event-driven-architecture]]

---

## Synchronous vs Asynchronous Communication

```
Synchronous (Request-Response):
  Client ──request──▶ Service B
  Client ◀──response── Service B
  Client blocks until response arrives.
  + Simple, immediate feedback
  - Tight coupling, cascading failures, latency adds up

Asynchronous (Message-based):
  Client ──message──▶ Queue ──▶ Service B (processes later)
  Client continues immediately.
  + Decoupled, resilient, natural backpressure
  - Eventual consistency, harder to debug, no immediate feedback
```

**Rule:** Use sync for user-facing reads. Use async for writes, workflows, fan-out.

---

## API Styles

### REST (Representational State Transfer)
- Resources identified by URLs; HTTP verbs (GET, POST, PUT, DELETE, PATCH)
- Stateless; cacheable; widely understood
- Weakness: over-fetching / under-fetching; versioning churn

```
GET    /orders/{id}        → fetch order
POST   /orders             → create order
PUT    /orders/{id}        → replace order
PATCH  /orders/{id}        → partial update
DELETE /orders/{id}        → cancel order
```

### GraphQL
- Client specifies exactly what fields it needs in one request
- Single endpoint; schema-typed
- Strength: eliminates over/under-fetching; great for mobile (bandwidth-conscious)
- Weakness: complex caching; N+1 query problem; schema churn

```graphql
query {
  order(id: "123") {
    status
    user { name email }
    items { productId quantity }
  }
}
```

### gRPC
- Binary protocol (Protobuf); HTTP/2; strongly typed contracts
- Strength: very fast (3–10× REST), bidirectional streaming, code generation
- Weakness: not browser-native; debugging harder; requires Protobuf toolchain

```protobuf
service OrderService {
  rpc GetOrder(OrderRequest) returns (OrderResponse);
  rpc StreamOrderUpdates(OrderRequest) returns (stream OrderEvent);
}
```

### When to use which

| Situation | Recommendation |
|-----------|---------------|
| Public API, external devs | REST |
| Mobile clients with varied data needs | GraphQL |
| Internal service-to-service | gRPC |
| Streaming / real-time | gRPC or WebSocket |
| Simple webhook/callback | REST |

---

## Messaging Patterns

### Queue (Point-to-Point)
One producer → one consumer. Message deleted after consumption.

```
Producer ──▶ [msg] [msg] [msg] ──▶ Consumer
                   Queue
```
Use for: task queues, job distribution, work offloading.

### Pub/Sub (Publish-Subscribe)
One producer → multiple independent consumers. Each gets a copy.

```
Publisher ──▶ Topic ──▶ Subscriber A
                   └──▶ Subscriber B
                   └──▶ Subscriber C
```
Use for: event notification, fan-out, decoupled integrations.

### Topic Partitioning (Kafka model)
```
Topic: orders
  Partition 0: [order_1, order_4, order_7]  ──▶ Consumer Group A / Instance 0
  Partition 1: [order_2, order_5, order_8]  ──▶ Consumer Group A / Instance 1
  Partition 2: [order_3, order_6, order_9]  ──▶ Consumer Group A / Instance 2
```
Ordering guaranteed within a partition. Scale consumers = scale partitions.

---

## Data Integration: ETL vs ELT

```
ETL (Extract Transform Load):
  Source DB ──extract──▶ Transform Engine ──load──▶ Data Warehouse
  Transform happens before load; warehouse gets clean data.
  Classic: Informatica, SSIS, Talend.

ELT (Extract Load Transform):
  Source DB ──extract──▶ Data Lake (raw) ──transform (in-place)──▶ Analytics Layer
  Load raw first; transform inside the warehouse using SQL.
  Modern: dbt + BigQuery/Snowflake/Redshift.
  Better: warehouse compute is cheap; keep raw data for reprocessing.
```

---

## Webhook vs Polling

```
Polling:
  Client ──GET /status?──▶ Server  (every 5s)
  Server ◀──"not ready"── Server
  Client ──GET /status?──▶ Server
  Server ◀──"ready!"   ── Server
  Wastes requests; latency = poll interval.

Webhook:
  Client registers: POST /register { callback_url }
  Server ──POST callback_url { event }──▶ Client (when event occurs)
  No wasted requests; near-realtime.
  Requires: client must be reachable (public endpoint or tunnel).

Long Polling:
  Client ──GET /events──▶ Server (holds connection open)
  Server responds when event occurs, then client reconnects immediately.
  Middle ground: lower latency than polling, no server-push infrastructure.

Server-Sent Events (SSE) / WebSocket:
  Persistent connection; server pushes events in real-time.
  SSE: one-way (server → client). WebSocket: bidirectional.
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
