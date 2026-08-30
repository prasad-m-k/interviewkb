---
uid: 21f1535f-a0eb-4d16-b064-ed86a43e2627
---

# Cost Architecture & FinOps

**Related:** [[solution-arch/topics/enterprise-architecture-frameworks]], [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/openai-platform-architecture]]

> **Gap-fill note:** the existing SA KB covers availability, scalability, and security NFRs in depth but had no dedicated page on **cost as a first-class architectural quality attribute**. Cost is consistently under-weighted by candidates relative to how often senior interviews probe it — and LLM token spend makes this gap acute for AI systems specifically.

---

## Why Cost Is an Architecture Attribute, Not a Finance Afterthought

```
Traditional NFRs an SA designs for:        Cost is the same SHAPE of
  Availability   → redundancy               problem, with its own
  Scalability    → horizontal scale         trade-off curve:
  Security       → encryption, IAM
  Performance    → caching, indexing        Cost ←──────────▶ every
                                             other NFR above.
                                             More redundancy = more $.
                                             More caching = less $
                                             but staleness risk.
                                             Bigger/smarter model =
                                             better answers, more $.
```

An architecture that meets every NFR except cost isn't a viable design — it's a design the business can't afford to run. Interviewers test whether you treat cost with the same rigor as latency or availability, or whether you only mention it if explicitly asked.

---

## FinOps Framework — The Three Phases

```
┌─────────────────────────────────────────────────────────────┐
│                      FINOPS LIFECYCLE                          │
├─────────────────────────────────────────────────────────────┤
│  INFORM                                                        │
│  → Full cost visibility: tag every resource by team/product/   │
│    environment; allocate shared costs (e.g. a shared Kubernetes │
│    cluster, a shared LLM gateway) back to consuming teams        │
│  → Without this, "reduce cost" has no owner and no target        │
├─────────────────────────────────────────────────────────────┤
│  OPTIMIZE                                                       │
│  → Rightsizing (are we over-provisioned?), reserved/committed   │
│    capacity vs on-demand, spot/preemptible for fault-tolerant    │
│    workloads, storage tiering, and — for AI workloads — model    │
│    routing to the cheapest model that clears the quality bar     │
├─────────────────────────────────────────────────────────────┤
│  OPERATE                                                        │
│  → Continuous governance: budgets + alerts, automated shutdown   │
│    of idle resources, cost anomaly detection, and a recurring    │
│    review cadence (not a one-time cost-cutting exercise)          │
└─────────────────────────────────────────────────────────────┘
```

---

## Classic Cloud Cost Levers (Still the Foundation)

```
Compute
  - Right-size instances to actual utilization (most over-provision)
  - Reserved/committed-use discounts for steady-state baseline load
  - Spot/preemptible instances for stateless, interruption-tolerant
    batch or worker workloads
  - Auto-scaling to match capacity to real-time demand, not peak

Storage
  - Lifecycle policies: hot → warm → cold → archive tiers by
    access-recency (same principle as [[solution-arch/concepts/caching]]
    eviction, applied to storage tiers instead of cache entries)
  - Deduplication and compression before storing, not after

Network
  - Minimize cross-AZ/cross-region data transfer (often a hidden,
    large line item) — co-locate chatty services
  - CDN for static/cacheable content to cut origin egress cost

Data
  - Query cost governance for pay-per-scan warehouses (partition
    pruning, column pruning — a poorly written query can cost
    100x a well-written one on the same data)
```

---

## LLM/AI Cost Architecture — The New Cost Surface

Token spend behaves differently from traditional infra cost: it scales with *usage pattern and prompt design*, not just provisioned capacity, and a single careless architecture choice (e.g., re-sending full conversation history every turn) can silently 10x spend.

```
┌────────────────────────────────────────────────────────────┐
│                LLM COST LEVERS, RANKED BY IMPACT              │
├────────────────────────────────────────────────────────────┤
│ 1. Model routing                                               │
│    Send only tasks that NEED the flagship model to it;         │
│    route simple classification/extraction to a small/mini      │
│    model. Often the single largest lever — flagship models      │
│    can be 10-20x the cost per token of a small model.            │
├────────────────────────────────────────────────────────────┤
│ 2. Prompt caching                                               │
│    Structure prompts with static content (system prompt,        │
│    tool definitions) first so the provider's prefix cache        │
│    hits on repeated calls — see                                  │
│    [[solution-arch/topics/llm-application-architecture]]         │
├────────────────────────────────────────────────────────────┤
│ 3. Context window discipline                                    │
│    Summarize/window conversation history instead of resending    │
│    full history every turn; truncate/rank retrieved documents     │
│    to only what's needed, not "retrieve top-50 to be safe"         │
├────────────────────────────────────────────────────────────┤
│ 4. Response caching (exact + semantic)                           │
│    Avoid a full model call entirely for repeated/similar           │
│    queries — see [[solution-arch/topics/llm-application-architecture]]│
├────────────────────────────────────────────────────────────┤
│ 5. Output length control                                         │
│    Constrain max_tokens and prompt for concise output where       │
│    verbosity isn't adding value — output tokens are typically      │
│    priced higher than input tokens                                 │
├────────────────────────────────────────────────────────────┤
│ 6. Batch processing for non-real-time workloads                   │
│    Providers offer discounted batch/async tiers (often ~50%)       │
│    for workloads that don't need synchronous responses               │
├────────────────────────────────────────────────────────────┤
│ 7. Fine-tuning a smaller model                                     │
│    Trade a one-time fine-tuning cost for a much cheaper             │
│    per-request cost at scale, once volume justifies it —            │
│    see [[solution-arch/topics/openai-platform-architecture]]        │
├────────────────────────────────────────────────────────────┤
│ 8. Agent loop budgets                                              │
│    Cap max iterations/tool calls per agent run — an unbounded        │
│    agent loop is an unbounded cost, not just a latency risk           │
│    (see [[solution-arch/topics/agentic-ai-architecture]])            │
└────────────────────────────────────────────────────────────┘
```

**Interview answer shape for "our LLM bill is too high":**
```
1. INFORM first: break down spend by feature/endpoint — is it
   concentrated in one flow? (usually yes — the 80/20 rule applies)
2. Check the cheap wins first: model routing, prompt caching,
   context trimming — these require no quality trade-off if done
   right
3. Only then consider quality-affecting levers: smaller model with
   fine-tuning, reduced retrieval depth — and validate against the
   eval suite (see [[solution-arch/concepts/llm-observability-and-evals]])
   before shipping, since these DO risk a quality regression
4. Put a budget + alert per feature going forward so this doesn't
   silently recur
```

---

## Cost Observability — You Can't Optimize What You Can't See

```
Per-request cost attribution needed for AI systems:
  request → (model used, input tokens, output tokens, tool calls,
             retrieval calls) → $ cost, tagged by feature/team

Without this, cost conversations become anecdotal ("it feels
expensive") instead of data-driven ("checkout-assistant is 60% of
spend because it resends full order history every turn").
```

---

## Sources
- [[solution-arch/topics/openai-platform-architecture]]
- [[solution-arch/topics/llm-application-architecture]]
