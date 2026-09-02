# System Design: High-Fan-Out Real-Time Notification / Pub-Sub System

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/service-discovery]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/inbox]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

**Not the same problem as:** [[solution-arch/scenarios/design-notification-system]] covers per-user, multi-channel dispatch (email/SMS/push for one order confirmation to one user) — a product-application-layer concern with modest fan-out. This page covers **broadcast infrastructure**: one publish event fanning out to millions of concurrently connected subscribers of a topic. The two compose — a product notification system like the sibling page could sit on top of this infrastructure as one specific publisher for one specific use case (e.g. "notify everyone subscribed to this live event").

---

## Step 1: Requirements

**Functional:**
- Publish a message to a topic; deliver it to every currently-connected subscriber of that topic
- Support per-topic ordering where a subscriber declares it needs strict order for that topic (not a global ordering guarantee across all topics)
- Handle subscriber connect / disconnect / reconnect, with bounded catch-up of messages missed while disconnected
- Support subscribers connected from any region, receiving the same publish with region-appropriate latency

**Non-Functional:**
- **Fan-out latency:** publish-to-delivery latency at millions-of-subscribers scale should stay in the low hundreds of milliseconds for the median subscriber, independent of how many other subscribers exist
- **Backpressure isolation:** one slow or stalled subscriber must never slow down or block delivery to the other subscribers of the same topic
- **Bounded catch-up window:** an offline subscriber's missed messages are retained and delivered on reconnect only up to a bounded window (e.g. minutes to low hours) — after that, the subscriber is told to resync state rather than replay an unbounded backlog
- **Throughput headroom:** a viral or live-event spike can push subscriber counts and publish rates well above steady-state baseline; the fan-out tier must absorb this without publish-path degradation
- **Availability:** publish path degrading in one region should not stop delivery to subscribers connected to healthy regions

---

## Step 2: High-Level Architecture

```
Publisher
   │  publish(topic, message)     — this call must stay cheap
   │  regardless of subscriber count; it does NOT loop over subscribers
   ▼
Topic Log (partitioned, ordered per partition)
   │  message appended once, subscribers read independently
   ▼
Fan-Out Tier (decoupled from publish path)
   │  maintains per-region, per-connection-shard subscriber registries
   │  pulls new messages from the topic log and pushes to open
   │  connections it owns
   ├──▶ Region A: connection-holding servers ──▶ millions of open
   │     connections (WebSocket/long-poll/gRPC-stream)
   ├──▶ Region B: connection-holding servers ──▶ millions of open
   │     connections
   └──▶ Region C: connection-holding servers ──▶ millions of open
         connections

Per-subscriber bounded queue sits between the fan-out tier and each
connection: a slow subscriber's queue fills and starts dropping/
coalescing for THAT subscriber only — it never blocks the shared
topic log or other subscribers' queues.
```

The load-bearing design decision is decoupling **publish** (append to the log, O(1) regardless of subscriber count) from **fan-out** (a separate tier that reads the log and pushes to connections). A naive design that has the publish call iterate over subscriber lists directly ties publish latency to subscriber count and turns a viral spike into a publish-path outage.

---

## Step 3: Delivery Guarantee and Ordering

```
Delivery guarantee: at-least-once.
  Exactly-once end-to-end (publish → arbitrary subscriber code) isn't
  achievable without cooperation from the subscriber, so the system
  guarantees at-least-once and pushes dedup to the subscriber side —
  see [[solution-arch/patterns/inbox]] for the standard shape of that
  dedup (subscriber tracks last-seen message ID per topic, drops
  redelivered duplicates).

Ordering: per-topic-partition ordering only, not global.
  A subscriber that needs order for a specific topic subscribes to
  that topic's single partition (or reassembles order client-side
  from a monotonic per-partition sequence number). Enforcing a single
  global order across all topics and all publishers would force every
  publish through one serialization point — that single point becomes
  the throughput ceiling for the entire system, for an ordering
  guarantee almost no subscriber actually needs.
```

---

## Step 4: Backpressure — Protecting the Fast Subscribers from the Slow Ones

```
Problem: subscriber S has a flaky connection or a slow consumer loop.
         Without isolation, buffering for S can grow unbounded and
         either exhaust fan-out-tier memory or force the tier to
         slow down delivery to everyone to avoid that.

Fix: bounded, per-subscriber queue with an explicit policy for what
happens when it's full — chosen per topic's semantics:

  DROP-OLDEST:  keep the queue's newest N messages, discard older
                ones once full. Correct for "current state" topics
                where only the latest value matters (e.g. a live
                score ticker) — a stale intermediate value is
                worthless once a newer one exists.
  COALESCE:     merge/collapse redundant updates instead of dropping
                (e.g. multiple "typing..." events collapse to one).
  DISCONNECT:   if delivery falls too far behind, close the connection
                and force the client to reconnect and do a bounded
                catch-up/resync rather than accumulate an ever-growing
                backlog.

The queue depth limit is what turns "one bad subscriber" from a
platform-wide backpressure problem into a contained, per-subscriber
one — this is the same isolation goal as
[[solution-arch/patterns/bulkhead]] applied to fan-out queues instead
of thread/connection pools.
```

---

## Step 5: Graceful Degradation on Regional Push-Infra Failure

```
Region B's connection-holding servers become unhealthy:
  1. Health checks detect it; a circuit breaker in front of Region B's
     fan-out tier trips.
  2. Region B's subscribers' connections drop; clients reconnect —
     ideally to a healthy region if the client library supports
     cross-region failover, otherwise they retry Region B with backoff.
  3. Messages published during the outage are NOT dropped for Region
     B's subscribers — they sit in the topic log (which is
     multi-region-replicated) and are delivered as catch-up once
     Region B recovers or the subscriber reconnects elsewhere,
     bounded by the catch-up window from Step 1.
  4. If the outage exceeds the catch-up window, affected subscribers
     are told to do a full state resync rather than receive a partial,
     confusing replay.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Publish/fan-out coupling | Decoupled — publish appends to a log, a separate tier fans out | Keeps publish latency independent of subscriber count |
| Delivery guarantee | At-least-once + subscriber-side dedup | End-to-end exactly-once isn't achievable without subscriber cooperation |
| Ordering scope | Per-topic-partition, not global | Global ordering would force a single serialization point and cap throughput |
| Slow-subscriber handling | Bounded per-subscriber queue with drop/coalesce/disconnect policy | Isolates one bad subscriber from affecting the rest — same goal as bulkhead |
| Offline catch-up | Bounded replay window, then forced resync | Unbounded replay risks an ever-growing backlog that never actually helps a badly-lagged client |
| Regional failure handling | Circuit breaker + multi-region-replicated log, not silent message drop | Subscribers reconnecting after an outage still get correct (bounded) catch-up |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #pub-sub #fan-out #real-time #system-design #nalsd*
