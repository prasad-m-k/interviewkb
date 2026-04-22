# Scenario: Monolith to Microservices Migration

**Patterns:** [[solution-arch/patterns/strangler-fig]], [[solution-arch/patterns/saga]], [[solution-arch/patterns/outbox]]
**Concepts:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/message-queues]]

---

## The Business Problem

You have a monolith that is:
- Deployed as one unit (any change requires full redeploy)
- Scaled as a unit (can't scale just the checkout service independently)
- Owned by one large team (changes require coordination across everyone)
- Causing slow delivery (large test suite, deployment fear)

---

## Step 1: Understand Before Acting

**Don't decompose prematurely.** Wrong service boundaries are worse than a monolith.

```
Before migrating:
  1. Map domain boundaries (Event Storming workshop)
     Identify bounded contexts: Users, Orders, Inventory, Payment, Shipping

  2. Analyse coupling
     Which modules change together? → May belong together
     Which modules scale independently? → Candidates for extraction

  3. Identify pain points
     What's slow to deploy? What's hardest to scale? → Extract these first

  4. Measure the monolith
     Response times, resource bottlenecks, deployment frequency
```

---

## Step 2: The Strangler Fig Strategy

```
Rule: Never do a big-bang rewrite. Extract incrementally.

Start: 100% in monolith
  │
  Step 1: Add facade (API Gateway) in front — don't change monolith yet
  │
  Step 2: Extract lowest-coupling, highest-value domain first
  │         (e.g., User Profile — simple reads, no transactions)
  │
  Step 3: Route /api/users to new User Service; monolith unchanged
  │
  Step 4: Run both in parallel; validate new service produces same results
  │
  Step 5: Remove user code from monolith
  │
  Step N: Repeat for next domain
  │
  End: All traffic in microservices; monolith decommissioned
```

---

## Step 3: Domain Extraction Sequence (Priority Order)

```
Extract in this order (easiest to hardest):

1. Read-only aggregations (reports, dashboards)
   → No writes; no consistency concerns; easy

2. Leaf services (no downstream dependencies)
   → User Profile, Product Catalogue
   → Own DB, no transactions with other domains

3. Orchestrating services (have dependencies)
   → Order Service (depends on Inventory, Payment)
   → Requires Saga pattern for distributed transactions

4. Core transaction domains (last)
   → Payment, Inventory
   → Hardest: strong consistency, regulatory requirements
```

---

## Step 4: Data Migration — The Hard Part

```
Problem: Monolith has one big shared DB; microservices need their own DBs.

Strategy: Shared DB → Separate DB (gradual)

Phase 1 (Logical separation):
  New service initially reads/writes monolith DB
  Add schema prefix: users.*, orders.* (boundaries explicit)
  Service code clear; DB still shared

Phase 2 (Physical separation — strangler moment):
  Set up new DB for the service
  Sync data: CDC (Debezium) or custom sync job
  New service reads from both; validates consistency
  Cut writes to new DB only
  Drain old DB of service's data

Phase 3 (Complete):
  Old DB tables for this domain are archived/dropped
  Service fully owns its DB
  No cross-service DB queries allowed
```

---

## Step 5: Cross-Service Communication Design

```
Sync (HTTP/gRPC) — for:
  User-facing reads where data must be current
  Simple request-response with immediate result needed
  Example: GET /orders/123 → calls User Service for customer details

Async (Kafka events) — for:
  Cross-service writes
  Fan-out (notify N services of one event)
  Non-critical background work
  Example: OrderPlaced event → Inventory reserves stock, Notification sends email

Anti-pattern: Synchronous chains
  Order Service ──▶ Inventory ──▶ Shipping ──▶ Notification (all sync)
  → One slow service blocks entire chain
  → Fix: async events for non-critical steps
```

---

## Step 6: Handling Distributed Transactions

When Order Service and Inventory Service must be consistent:

```
Option 1: Saga (preferred for most cases)
  Order Service creates order (pending)
  Publishes OrderCreated event
  Inventory Service reserves stock
  Publishes StockReserved event
  Order Service confirms (completed)
  
  Failure: compensation transactions roll back

Option 2: Eventual consistency + compensating UI
  Allow over-selling temporarily
  Detect on fulfilment
  Notify customer, offer alternatives
  (Works for low-stock items in e-commerce)

Option 3: Shared service for coupled operations
  If Order + Inventory are always transactional together → maybe one service
```

---

## Step 7: Team Structure

Conway's Law: "Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure."

```
Wrong: One team owns all microservices → still monolith in practice
Right: Each service owned by one autonomous team

Team topology:
  Stream-aligned team: owns User Service end-to-end (code → deploy → oncall)
  Platform team: provides shared infrastructure (CI/CD, observability, service mesh)
  Enabling team: helps stream teams adopt new patterns

Service ownership rules:
  One team per service
  Team owns: deploy, data, SLA, oncall
  No sharing: if two teams need same data, define clear owner + API
```

---

## Step 8: Checklist Before Extracting a Service

```
✅ Bounded context is well-defined (domain expert agrees)
✅ API contract documented (OpenAPI / Protobuf)
✅ Data ownership clear (no other service directly queries this DB)
✅ Observability in place (logs, metrics, traces before cutting traffic)
✅ Canary deployment configured (gradual traffic shift)
✅ Runbook written (how to roll back if new service breaks)
✅ SLA defined (what latency/availability does this service promise?)
✅ Integration tests updated to test new API boundary
```

---

## Common Pitfalls

```
❌ Distributed monolith: microservices but tightly coupled via sync chains
   Fix: async events; define contracts; don't share DBs

❌ Premature decomposition: too many services before domain is understood
   Fix: start with 3–5 services; split further when boundaries prove stable

❌ Missing observability: can't debug distributed systems without traces
   Fix: distributed tracing (Jaeger) and structured logging from day 1

❌ No team ownership: service owned by "everyone" = maintained by no one
   Fix: hard rule — every service has exactly one owning team

❌ Big-bang data migration: move all data at once, hope nothing breaks
   Fix: strangler fig data strategy — sync, validate, cut
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
- [[solution-arch/sources/clean-architecture]]
