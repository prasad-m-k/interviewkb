# Blue-Green & Canary Deployments

**Topic:** [[solution-arch/topics/cloud-agnostic-principles]]
**Related concepts:** [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/api-gateway]]

## What it solves
Deploying new software versions without downtime and with the ability to instantly roll back if problems are detected.

---

## Blue-Green Deployment

Two identical production environments: **Blue** (current) and **Green** (new version). Switch traffic atomically.

```
Step 1: Blue live, Green idle
  Traffic ──▶ Load Balancer ──100%──▶ Blue (v1)
                                       Green (v2, idle)

Step 2: Deploy v2 to Green, run tests
  Traffic ──▶ Load Balancer ──100%──▶ Blue (v1)
                                       Green (v2) ← deploy + smoke test

Step 3: Switch traffic
  Traffic ──▶ Load Balancer ──100%──▶ Blue (v1, idle)
                              └──100%──▶ Green (v2, live)

Step 4: Rollback (if issue found)
  Traffic ──▶ Load Balancer ──100%──▶ Blue (v1)  ← instant switch back
```

**Pros:** Instant cutover and rollback; zero downtime; full production testing before switch.
**Cons:** Double infrastructure cost; data migrations are complex (both versions must be compatible with the same DB schema during transition).

---

## Canary Deployment

Gradually shift traffic from old version to new, monitoring for errors at each step.

```
Step 1: 5% canary
  Traffic ──▶ LB ──95%──▶ v1 (stable)
               └───5%──▶ v2 (canary) ← monitor error rate

Step 2: 25% (if metrics look good)
  Traffic ──▶ LB ──75%──▶ v1
               └──25%──▶ v2

Step 3: 50%
  Traffic ──▶ LB ──50%──▶ v1
               └──50%──▶ v2

Step 4: 100% (full rollout)
  Traffic ──▶ LB ──100%──▶ v2
              v1 decommissioned
```

**Rollback:** At any step, if error rate or latency spikes → set traffic back to 0% for v2.

**Automated canary analysis:**
```
Deploy v2 at 5%
  Monitor for 10 minutes:
    Error rate v2 vs v1 (SLO: v2 ≤ 1.1× v1 error rate)
    Latency P99 v2 vs v1 (SLO: v2 ≤ 1.2× v1)
  
  Pass → promote to 25%
  Fail → auto rollback to 0%
```
Tools: Argo Rollouts, Spinnaker, Flagger.

---

## Rolling Deployment

Replace instances one at a time, maintaining availability throughout.

```
Fleet: [v1] [v1] [v1] [v1] [v1]

Step 1: [v2] [v1] [v1] [v1] [v1]
Step 2: [v2] [v2] [v1] [v1] [v1]
Step 3: [v2] [v2] [v2] [v1] [v1]
Step 4: [v2] [v2] [v2] [v2] [v1]
Step 5: [v2] [v2] [v2] [v2] [v2]
```

**Pros:** No extra infra cost; always serving traffic.
**Cons:** Both versions run simultaneously — API must be backwards compatible; hard to roll back.

---

## Comparison

| Strategy | Cost | Rollback | Downtime | Complexity |
|----------|------|----------|---------|------------|
| Big Bang | Low | Manual | Yes | Low |
| Rolling | Low | Hard | No | Medium |
| Blue-Green | 2× infra | Instant | No | Medium |
| Canary | Low extra | Instant | No | High |

---

## Database Schema Migrations with Deployments

```
The hard part: DB schema must work with BOTH v1 and v2 during transition.

Strategy: Expand → Migrate → Contract

Step 1 (Expand): Add new column (nullable, no constraint)
  Deploy v2 which writes to both old and new column
  v1 continues writing to old column only

Step 2 (Migrate): Backfill new column for existing rows
  ALTER TABLE orders ADD COLUMN new_status VARCHAR;
  UPDATE orders SET new_status = status WHERE new_status IS NULL;

Step 3 (Contract): Drop old column after v1 is fully replaced
  ALTER TABLE orders DROP COLUMN status;
```

Never rename or drop columns in the same deploy that switches traffic — always do it in subsequent deploys.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
