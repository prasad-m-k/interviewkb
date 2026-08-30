# Solution Architecture Flashcards (Anki Format)

> Import into Anki using the "Basic" note type. Each `## Q:` / `**A:**` block = one card.
> Or use the Obsidian Anki Sync plugin.

---

## Fundamentals

## Q: What is the difference between Availability and Reliability in system design?
**A:**
- **Availability:** Is the system responding? (% uptime)
- **Reliability:** Is it giving correct results?

A system can be available (returns 200 OK) but unreliable (returns wrong data). Both are needed; they require different mitigations — availability needs redundancy, reliability needs checksums and idempotency.

---

## Q: What do "four nines" and "five nines" of availability mean in actual downtime?
**A:**
- **99.99% (four nines):** ~52 minutes downtime per year / ~4.4 min per month
- **99.999% (five nines):** ~5.3 minutes per year / ~26 seconds per month

Four nines is achievable with multi-AZ architecture. Five nines requires active-active multi-region and extensive operational investment.

---

## Q: What is the NFR interview checklist a Solution Architect runs through before designing?
**A:**
1. **Scale:** DAU, RPS, read:write ratio, data volume
2. **Availability:** SLA uptime requirement
3. **Consistency:** strong or eventual?
4. **Latency:** P50/P99 target (user-facing vs background)
5. **Durability:** can acknowledged writes be lost?
6. **Security:** PII, regulated data, access model

---

## Q: What is an Architecture Decision Record (ADR)?
**A:** A short document capturing a significant architectural decision: Context (why the decision needed to be made), Decision (what was decided), Consequences (trade-offs, pros/cons), and Alternatives considered. Stored in version control alongside code. Enables future developers to understand why the system is designed as it is.

---

## Q: What is Conway's Law and how does it affect microservices?
**A:** "Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure." — Mel Conway

Implication for microservices: if three teams work on one codebase, you get a three-module design. For effective microservices, structure teams around domain services (stream-aligned teams) so each team owns one service end-to-end.

---

## CAP Theorem

## Q: State the CAP theorem and what the practical trade-off is.
**A:** In a distributed system, you can guarantee at most 2 of: **Consistency** (every read gets the latest write), **Availability** (every request gets a non-error response), **Partition Tolerance** (works despite network failures).

Since partitions always happen in real distributed systems, the real choice is **C vs A** during a partition: either return an error (CP) or return possibly-stale data (AP).

---

## Q: Give a real example where you would choose CP over AP, and vice versa.
**A:**
- **CP:** Bank balance check. Showing stale balance could allow overdraft. Better to return an error than wrong data.
- **AP:** Social media feed. Showing a post from 2 seconds ago is acceptable. Availability > perfect freshness.
- **AP:** Product catalogue. Brief price staleness is fine; catalogue must be reachable.

---

## Q: What is PACELC and how does it improve on CAP?
**A:** PACELC extends CAP: **P**artition → **A**vailability vs **C**onsistency; **E**lse (no partition) → **L**atency vs **C**onsistency.

Even without a partition, a distributed system must trade latency (fast local reads) against consistency (slow coordination). CAP ignores this normal-operation trade-off. PACELC captures the full picture.

---

## Caching

## Q: Explain Cache-Aside vs Write-Through caching strategies.
**A:**
- **Cache-Aside (Lazy):** App checks cache; miss → query DB → populate cache → return. Cache only holds requested data. Cache failure is non-fatal.
- **Write-Through:** App writes to cache and DB synchronously on every write. Cache always consistent with DB. Higher write latency; cache may hold unread data.

Use cache-aside for read-heavy workloads. Write-through when reads always follow writes.

---

## Q: What is cache stampede (thundering herd) and how do you prevent it?
**A:** When a popular cached item expires, many simultaneous requests miss the cache and all hit the database simultaneously, overwhelming it.

Prevention:
1. **Mutex/lock:** only one request fetches from DB; others wait
2. **Probabilistic early expiry:** randomly expire items slightly before TTL to stagger refreshes
3. **Async refresh:** background process refreshes before expiry; serve stale until refreshed
4. **Jitter on TTL:** add random offset to TTLs to stagger expirations across the fleet

---

## Q: What is a hot key in Redis and how do you mitigate it?
**A:** A hot key is a single cache key that receives an overwhelming proportion of requests (e.g., a celebrity's profile, a viral product listing).

Mitigations:
1. **Local in-process cache** (L1 before Redis): serve from memory for hottest keys
2. **Key sharding:** store as `key:0`, `key:1`, ..., `key:N`; random read from any shard
3. **Read replicas:** Redis replication; route reads across replicas

---

## Load Balancing

## Q: What is the difference between L4 and L7 load balancing?
**A:**
- **L4 (Transport):** Routes by IP:port. Cannot inspect HTTP content. Faster. No SSL termination. Examples: AWS NLB, HAProxy TCP.
- **L7 (Application):** Routes by URL path, headers, cookies. Can do SSL termination, sticky sessions, path-based routing. Slower but smarter. Examples: AWS ALB, nginx, Envoy.

Use L4 for raw throughput; L7 for smart routing across microservices.

---

## Q: What is the "thundering herd" problem when a load balancer adds a new node?
**A:** When a new node joins (e.g., after auto-scaling), the load balancer might route a disproportionate initial burst of traffic to it before it's warm (caches empty, JIT not compiled). This makes the new node slow, and may cascade.

Mitigation: **Gradual ramp-up** — start new instance at 0% weight, increase gradually over 1–2 minutes as it warms up.

---

## Messaging

## Q: What are the three message delivery guarantees? Which is hardest to achieve?
**A:**
- **At-most-once:** Send once; may be lost on failure. Easiest.
- **At-least-once:** Retry until acked; may be duplicated. Common default.
- **Exactly-once:** Delivered and processed exactly once. Hardest — requires idempotent consumers + transactional producers.

In practice: use **at-least-once + idempotent consumers** = effectively exactly-once.

---

## Q: When would you use a queue vs a database for storing jobs?
**A:**
- **Queue (Kafka/SQS):** Natural at-least-once delivery, built-in retry, backpressure, fan-out, consumer groups. Best for: background jobs, event-driven workflows.
- **Database (job table):** Flexible queries, ACID guarantees, easy to monitor. Best for: scheduled jobs, jobs needing complex state transitions, audit requirements.

Rule of thumb: if you need to query/filter pending jobs by arbitrary criteria → DB. If you need high-throughput, simple enqueue/dequeue → queue.

---

## Q: What is Kafka consumer lag and why does it matter?
**A:** Consumer lag = difference between the latest offset in a partition and the consumer group's committed offset. It measures how far behind a consumer is.

Why it matters: high lag = consumers are not keeping up with producers. If lag grows unboundedly, the system is overloaded. SLO: if lag > X messages or Y minutes, alert — backlog may affect downstream freshness or fill disk.

---

## Patterns

## Q: Explain the Circuit Breaker pattern and its three states.
**A:** Prevents cascading failures by stopping calls to a failing service.

**CLOSED:** Normal operation. All calls pass through. Count failures.
**OPEN:** Threshold exceeded. All calls fail immediately (return error/fallback). No calls to dependency. Timer runs.
**HALF-OPEN:** After timeout, allow a probe request. Success → CLOSED. Failure → OPEN again.

---

## Q: What is the Saga pattern and what are its two styles?
**A:** Manages distributed transactions across microservices via a sequence of local transactions, each with a compensating transaction.

**Choreography:** Services emit events and react to each other's events. Decoupled; no central coordinator. Hard to trace.

**Orchestration:** Central Saga Orchestrator tells each service what to do. Clear flow; easy to debug. Orchestrator could become a bottleneck.

---

## Q: What is the Outbox Pattern and what problem does it solve?
**A:** Solves the **dual-write problem**: ensuring a DB write and a message publish are atomic without 2PC.

Implementation: In the same DB transaction, write to the business table AND an outbox table. A relay process (polling or CDC) reads the outbox and publishes to the message broker, then marks entries published.

Guarantee: at-least-once delivery. Consumers must be idempotent.

---

## Q: What is the Strangler Fig pattern?
**A:** Incrementally replaces a legacy system by building a new system alongside it. A facade routes requests to either old or new based on which functionality has been migrated. The old system is "strangled" as more traffic moves to the new one, until the old system can be decommissioned.

Key rules: Never rewrite while migrating. Facade must be transparent to clients. Exit criteria per domain.

---

## Q: What is the BFF (Backend for Frontend) pattern?
**A:** A dedicated aggregation layer per client type (mobile, web, partner). Each BFF calls multiple backend services and returns exactly the data that specific client needs — no over-fetching, no under-fetching.

When to create a BFF: clients have fundamentally different data needs, different auth requirements, or are owned by different teams. Not worth creating if clients need nearly identical data.

---

## Q: What is the difference between Blue-Green and Canary deployments?
**A:**
- **Blue-Green:** Two identical environments. Deploy to idle (Green), test, then switch 100% traffic instantly. Rollback = instant switch back. Cost: 2× infra.
- **Canary:** Gradually shift traffic (5% → 25% → 100%) monitoring metrics at each stage. Auto-rollback if error rate spikes. Cost: minimal infra overhead. Requires observability.

Use Blue-Green for major schema/version changes needing instant rollback. Use Canary for continuous delivery with risk monitoring.

---

## Data Architecture

## Q: What is the difference between OLTP and OLAP and why must they be separate?
**A:**
- **OLTP:** Many short transactions (ms), writes + reads, normalised, current data. e.g., PostgreSQL.
- **OLAP:** Few long analytical queries (seconds-minutes), read-heavy, denormalised, historical. e.g., BigQuery, Snowflake.

They must be separate because analytical queries (full table scans, complex aggregations) hold locks, consume CPU, and degrade transaction performance on OLTP databases. Use a data warehouse or CQRS read models for analytics.

---

## Q: What is CQRS and when would you use it?
**A:** Command Query Responsibility Segregation separates write (command) and read (query) paths into distinct models, often backed by different stores.

Use when:
- Read patterns are very different from write patterns
- Need multiple read models (dashboard, mobile, search)
- Read and write sides need to scale independently
- An audit/event log is needed

Don't use for simple CRUD apps — unnecessary complexity.

---

## Q: What is Event Sourcing and how does it differ from a traditional audit table?
**A:** Event Sourcing stores the full sequence of events that led to current state, making events the **primary source of truth**. Current state is derived by replaying events.

Traditional audit table: a changelog appended alongside a mutable state table. The mutable state is still the source of truth; the audit table is secondary.

Key difference: in event sourcing you can reconstruct any past state by replaying up to that point. With a traditional audit table, you cannot (you only know what changed, not the full context).

---

## Q: What are the consistency models from strongest to weakest?
**A:**
1. **Linearisability:** Every read sees the most recent write globally. Single-machine feel. High latency.
2. **Sequential consistency:** All nodes see ops in same order; may lag real time.
3. **Causal consistency:** Causally-related ops appear in order; concurrent ops may differ.
4. **Eventual consistency:** All replicas will converge given no new writes. Stale reads possible.

Money → linearisable. Social feed → eventual consistency is fine.

---

## Security

## Q: What is Zero Trust Architecture?
**A:** "Never trust, always verify." Every request is authenticated and authorised regardless of network origin — even internal traffic. No implicit trust for being "inside the network."

Pillars:
1. Verify explicitly (identity + device + context for every request)
2. Least privilege (minimal access, time-limited)
3. Assume breach (segment networks; monitor east-west traffic)

Enabled by: mTLS between services (service mesh), RBAC/ABAC, short-lived tokens, comprehensive audit logging.

---

## Q: What is the STRIDE threat model?
**A:**
- **S**poofing — impersonating another user/system → AuthN, mTLS
- **T**ampering — modifying data in transit/rest → integrity checks, HMAC
- **R**epudiation — denying an action was taken → audit logs, signing
- **I**nformation disclosure — data leakage → encryption, access control
- **D**enial of Service — overwhelming the system → rate limiting, WAF
- **E**levation of privilege — gaining unauthorised higher access → RBAC, least privilege

---

## Q: What is the difference between mTLS and regular TLS?
**A:**
- **TLS (one-way):** Client verifies server's certificate. Server doesn't verify client identity. Standard HTTPS.
- **mTLS (mutual):** Both sides present and verify certificates. Client AND server authenticated. Used for service-to-service authentication in microservices/service mesh.

mTLS eliminates the need for API keys between services — identity proven cryptographically.

---

## Scalability

## Q: Explain consistent hashing. Why is it better than modular hashing for distributed caches?
**A:** Nodes and keys are hashed to positions on a virtual ring (0–2³²). A key maps to the nearest node clockwise.

Why better than `hash(key) % N`:
- Adding/removing 1 node → only `~1/N` keys reassign (not all keys)
- Prevents cache stampede when scaling
- Used by: Redis Cluster, Cassandra, DynamoDB, Memcached

---

## Q: When would you choose sharding over read replicas for database scaling?
**A:**
- **Read replicas:** Write bottleneck not yet hit; read-heavy workload. Easier; data stays on one primary.
- **Sharding:** Write throughput exceeds single-node capacity OR dataset doesn't fit on single disk OR true horizontal write scaling needed.

Order of escalation: index tuning → caching → read replicas → vertical scale → sharding.
Sharding is last resort — adds significant complexity (cross-shard queries, resharding, hot spots).

---

## Q: What is the hot spot problem in sharding and how do you fix it?
**A:** A shard key that concentrates access on one shard (e.g., celebrity user_id, trending product) makes that shard a bottleneck.

Fixes:
1. **Compound key:** `(user_id, random_suffix)` distributes hot user across N virtual shards
2. **Dedicated shard:** give known hot entities their own shard
3. **Application-layer cache:** cache hot key aggressively in Redis, reducing DB hits
4. **Read replicas per shard:** scale reads on the hot shard specifically

---

## Q: What is Little's Law and how do you apply it to size a thread pool?
**A:** **L = λ × W** — average number of items in system = arrival rate × average time spent.

Thread pool sizing:
  `threads_needed = requests_per_second × average_latency_seconds`

Example: 100 req/sec, P99 latency 200ms → `100 × 0.2 = 20 concurrent requests` → set pool to 20–25 threads.

Undersized pool: requests queue → latency grows → cascading failure. Oversized: wasted resources.

---

## Q: What is the expand-migrate-contract pattern for zero-downtime DB schema changes?
**A:** A three-phase approach to changing DB schema while keeping both old and new app versions compatible:

1. **Expand:** Add new column/table (nullable, no constraint). Deploy app writing to both old and new schema.
2. **Migrate:** Backfill existing rows. Validate data quality.
3. **Contract:** Remove old column after all traffic is on new app version.

Never rename or drop a column in the same deployment that switches traffic — always spread across 3 separate deploys.

---

## AI Solution Architecture, Agentic AI & OpenAI

## Q: What's the decision framework for choosing prompting vs RAG vs fine-tuning vs agentic architecture?
**A:**
- **New/changing knowledge needed** → RAG (retrieval)
- **Static context that fits in the prompt** → put it in the system prompt
- **New output format/style/narrow skill, no new knowledge** → fine-tuning
- **Neither of the above, just phrasing/reasoning help** → prompt engineering
- **Task needs multiple steps/tool calls/runtime decisions** → agentic architecture

Common trap: reaching for fine-tuning to "teach the model facts" — fine-tuning changes behavior/style reliably, not factual knowledge. Use RAG for facts.

---

## Q: What is the difference between a "workflow" and an "agent"?
**A:** A workflow is a fixed, developer-defined sequence of LLM calls and tools. An agent is a system where the model's own output dynamically decides the next action — the model is the router, not the code. Start with the simplest workflow that works; add agentic complexity only when task variability demands it.

---

## Q: What are the six core agentic design patterns?
**A:** Prompt chaining (fixed sequential steps), routing (classify then dispatch to a specialist), parallelization (sectioning or voting), orchestrator-worker (dynamically spawned subtasks), evaluator-optimizer (generate-critique loop), and the fully autonomous agent (open-ended ReAct loop). Complexity and cost increase in roughly that order — pick the simplest one that fits.

---

## Q: Why is prompt injection fundamentally different from SQL injection?
**A:** SQL injection exploits a syntactic boundary — escaping/parameterization closes it completely. Prompt injection exploits a semantic boundary: the model has no hard separation between "instructions" and "data," both are just tokens. There is no complete syntactic fix — defense requires layering (input/output classifiers, least-privilege tool scoping, human approval gates on high-risk actions).

---

## Q: How do you prevent a tool-using agent from double-executing a refund on retry?
**A:** Every side-effecting tool call carries an idempotency key (derived from conversation/turn ID); the downstream service dedupes on that key — same idempotency principle used everywhere else in distributed systems, now triggered by a non-deterministic LLM caller instead of a flaky network client.

---

## Q: Chat Completions vs Responses API vs Assistants API — how do you choose?
**A:** Chat Completions: full manual control over state and the tool-call loop, best for custom/portable orchestration. Responses API: OpenAI's converged surface with optional server-side state and built-in tools, best for new builds. Assistants API: fully managed threads/runs, fastest to stand up, least orchestration control. Choose based on how much orchestration control you need vs how much plumbing you want the vendor to manage.

---

## Q: OpenAI direct vs Azure OpenAI — what usually decides it for a regulated enterprise?
**A:** Azure OpenAI usually wins when the enterprise already runs on Azure AD/MSAL for identity and needs data residency, Private Link/VNet isolation, and existing compliance certifications (SOC 2, HIPAA, FedRAMP) — it inherits the existing IAM and compliance posture instead of building a parallel one. OpenAI direct wins when access to the newest models ahead of Azure's rollout is the dominant requirement.

---

## Q: What does Model Context Protocol (MCP) solve that function calling alone doesn't?
**A:** Function calling defines the model-facing contract for one request. MCP standardizes the integration layer underneath it — how an application discovers and connects to a growing ecosystem of tool servers — turning an N×M (apps × tools) bespoke-integration problem into an N+M one.

---

## Q: What's the single largest lever for reducing LLM token cost, and why?
**A:** Model routing — sending only tasks that need it to the flagship model and routing simple classification/extraction to a small/cheap model. Flagship models can cost 10-20x per token vs small models, making routing the highest-leverage, lowest-quality-risk cost lever, ahead of prompt caching, context trimming, and response caching.

---

## Q: Why can't traditional unit tests validate an LLM application?
**A:** Output is non-deterministic — the same input can produce differently-worded but equally valid outputs, or subtly wrong ones that still look plausible. The substitute is an eval-gated pipeline: run every prompt/model/config change against a fixed golden set (exact-match, rubric, or LLM-as-judge scoring) before deploy, canary on live traffic, and monitor production proxy metrics (thumbs-down rate, escalation rate) — the direct analogue of CI/CD for prompt/model changes.

---

## Q: A regulator asks why your AI system denied a customer's application — what must the architecture support?
**A:** A full, immutable trace reconstructable for the specific point in time the decision was made: exact prompt sent, retrieved context, model version, output, and any tool calls — requiring versioned prompts and versioned retrieval index snapshots, not just current state. This must be designed in from day one, not retrofitted after an audit request.

---

## Q: EA vs SA vs TA — what's the scope difference?
**A:** Enterprise Architect: whole-organization technology strategy and standards, multi-year horizon. Solution Architect: one program's technical blueprint within EA guardrails, months-to-a-year horizon. Technical/Application Architect: internal design of one application/service, weeks-to-months horizon. A design interview should open by clarifying which of these scopes is actually being asked about.

---

## REST API Design, Caching & Hot-Path System Design

## Q: What's the difference between "safe" and "idempotent" for an HTTP method, and why does it matter architecturally?
**A:** Safe = doesn't change server state (GET, HEAD) — can be retried/prefetched/cached freely. Idempotent = calling it N times has the same effect as once (GET, PUT, DELETE) — safe for a client to retry blindly after a timeout. POST is neither, which is why a client can't safely retry a timed-out POST /orders without risking a duplicate order — hence idempotency keys.

## Q: How do idempotency keys retrofit safety onto a non-idempotent POST?
**A:** The client sends a self-generated `Idempotency-Key` header, the same value on retry. The server persists (key → response) for a bounded window; if it sees a repeated key, it returns the ORIGINAL response instead of re-executing the operation (e.g. re-creating the order or re-charging the card).

## Q: Cursor-based vs offset-based pagination — when does the difference actually matter?
**A:** Offset (`?offset=100&limit=20`) gets expensive at large offsets and is unstable if rows are inserted/deleted mid-pagination (rows shift, causing skips/duplicates). Cursor-based (`?after=cursor123`) stays O(1) via an indexed `WHERE id > :cursor` and is stable under concurrent writes — the right default for large or frequently-mutated collections; offset is fine for small, mostly-static admin lists needing "jump to page N."

## Q: What's the core lens of "hot-path-first" system design?
**A:** Reads want speed (and often tolerate staleness); writes want correctness (and never should trade it away). Classify every endpoint by traffic volume and freshness requirement BEFORE choosing any infrastructure — then scale only what's actually hot. "You do not scale the whole system, you scale what is hot."

## Q: Display reads vs decision reads — what's the distinction and why does conflating them cause bugs?
**A:** Display reads (product pages, search, recommendations) tolerate eventual consistency — a few seconds of staleness is invisible to the user. Decision reads (checkout, inventory confirmation) require fresh, authoritative data. Conflating them is exactly how a user sees one price browsing (stale cache) and gets charged a different amount at checkout (correct primary read) — a revenue/trust bug caused by routing a decision read through the display-read's cache.

## Q: How does the outbox pattern apply specifically to cache invalidation?
**A:** Write the invalidation event into the SAME database transaction as the data change (an outbox table), so both succeed or both fail together — never a separate, non-atomic "update DB, then tell the cache" step that can partially fail. A background worker publishes the event asynchronously; the cache consumer deletes the stale key. TTL remains as a safety net in case the invalidation pipeline itself fails silently.

## Q: Why is TTL called "the safety net, not the primary mechanism" in a well-designed cache invalidation pipeline?
**A:** Even a reliable outbox-driven invalidation path can fail silently (consumer down, event delayed, bug). TTL bounds the WORST CASE staleness window so the system self-heals regardless — but relying on TTL alone as the only invalidation mechanism (no event-driven path) means accepting the full TTL window as guaranteed staleness on every write, not just as a rare fallback.

## Q: Why is naive hash(key) % N a bad way to shard a distributed cache?
**A:** Changing N (a node joins or leaves) remaps almost every key at once, causing a near-total cache wipe exactly when the cluster is already in a fragile state. Consistent hashing (nodes and keys hashed onto the same ring) bounds the remapped fraction to roughly 1/N instead.

## Q: What is a cache "gutter pool" and what problem does it solve?
**A:** A small pool of spare cache nodes on standby. When a primary cache node fails, its traffic routes to the gutter pool instead of triggering a rehash across survivors — avoiding a stampede of simultaneous cache misses hitting the database right when the cluster is already degraded. From Facebook's Memcache scaling paper.

## Q: How does Facebook's "lease" mechanism prevent thundering herd on a popular cache key's expiry?
**A:** When a hot key expires, the first server to miss gets a short-lived lease token and is the ONLY one allowed to query the DB and repopulate the cache; every other server missing the same key during the lease window waits/retries instead of also querying the DB — turning N simultaneous DB queries into 1.

---

## Tiered Caching Platforms (Tier-0 / Platform-Team View)

## Q: In a two-tier cache design (in-process + distributed), what determines whether a dataset belongs in Tier 1 or Tier 2?
**A:** Working-set size and sharing need. Tier 1 (in-process/local) is free and sub-microsecond but small and per-instance — good for small, extremely hot data (routing tables, flags). Tier 2 (distributed) is shared across the fleet and survives individual restarts — needed once data must stay consistent/shared across instances or exceeds one process's memory.

## Q: What is zone-aware replication in a distributed cache, and what does it protect against?
**A:** Cache data is replicated across multiple availability zones, and the client library prefers reading from the local AZ's replica (for latency and cost) but fails over to another AZ's replica if the local one is unhealthy. It protects against an entire AZ failure turning into a simultaneous, fleet-wide cache wipe. (Public real-world precedent: Netflix's EVCache, a Memcached-based distributed cache.)

## Q: Why does a platform team put an abstraction/SDK layer in front of its caching system instead of letting callers talk to the cache directly?
**A:** Two reasons: (1) developer productivity — every calling team gets a single, secure, consistent interface instead of hand-rolling cache-aside logic and retry policy; (2) platform optionality — because callers depend on the abstraction rather than the backend, the platform team can change sharding strategy, swap the underlying engine, or migrate a dataset to a different storage system without every caller changing code.

## Q: At extreme scale, what are the highest-impact levers for keeping a caching platform's cost down?
**A:** Roughly in order: (1) push small hot working sets to the free in-process tier instead of the distributed tier; (2) tune TTL/eviction per dataset rather than one global policy; (3) compress cached values to cut RAM footprint (usually the dominant cost driver); (4) decide build-vs-buy per dataset, not platform-wide; (5) prefer local-AZ reads by default to avoid cross-AZ data-transfer cost on every request.

---

## Microsoft Responsible AI (CoreAI) #ResponsibleAI

## Q: What are the two distinct surfaces a Responsible AI role at Microsoft's CoreAI org must govern?
**A:** (1) Product-facing RAI — guardrails, content safety, and governance for AI systems shipped to customers (see [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/azure-ai-content-safety]]). (2) SDLC-facing RAI — governing the AI tooling used internally to build software (AI-generated requirements, design docs, code) so hallucinated or subtly wrong output doesn't silently become production infrastructure (see [[solution-arch/concepts/responsible-ai-sdlc-governance]]). Most candidates only prepare the first.

## Q: What are Azure AI Content Safety's 4 built-in harm categories, and how are they scored?
**A:** Hate, Sexual, Violence, Self-Harm. Each is scored independently on a discrete severity scale of 0/2/4/6 (not a continuous score) — content can be high-severity on one category and zero on another. Discrete levels make the block/allow threshold an auditable policy decision per category rather than an ad hoc per-team judgment call.

## Q: What do Prompt Shields' two sub-detectors catch, and which one maps to indirect prompt injection?
**A:** User Prompt attacks catch direct jailbreak attempts in the user's own input. Document attacks catch indirect/injected instructions hidden inside retrieved documents, emails, or web content the model ingests — the productized defense for the indirect-injection threat described in [[solution-arch/concepts/ai-guardrails-and-safety]].

## Q: Is Azure OpenAI's content filter opt-in or opt-out?
**A:** On by default and not fully removable without a Microsoft-approved modified-content-filter application — a platform-level enforcement decision, not an app-level toggle. This is the concrete mechanism behind the "provider enforces a guardrail floor" framing in [[solution-arch/topics/ai-governance-responsible-ai]].

## Q: Give an example of an SDLC-facing RAI risk that has no equivalent in product-facing RAI.
**A:** Package hallucination / slopsquatting — an AI coding assistant suggests a plausible-but-nonexistent dependency name; an attacker has pre-registered that exact name as a malicious package. The control is a dependency-verification CI gate blocking anything outside an approved registry, regardless of how confidently the assistant suggested it. See [[solution-arch/concepts/responsible-ai-sdlc-governance]].

## Q: Why should the code-review bar for AI-authored changes be HIGHER, not lower, than for human-authored changes?
**A:** Automation bias — reviewers are measurably less scrutinous of confident, well-formatted AI output than of a colleague's rough draft. A structured, explicit review checklist for AI-assisted changes counteracts this; treating AI-authored code as "probably fine, it's AI" is backwards.

## Q: A CoreAI content-safety service sits in the hot path of every AI call across Microsoft's product surface. What's the central availability design decision?
**A:** Fail-open vs fail-closed policy, decided per harm-severity category, not as an afterthought — a regional outage in the safety layer can't be allowed to take down every downstream AI product (Copilot, Office, Teams, Azure OpenAI customers), but it also can't silently let unmoderated content through for high-severity categories. This is the same HA reasoning as [[solution-arch/scenarios/high-availability-platform]] applied to a company-wide hard dependency.
