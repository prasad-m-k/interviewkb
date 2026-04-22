# Deployment Strategies

**Topic:** [[devops/topics/ci-cd]]
**Related:** [[devops/patterns/zero-downtime-deployment]], [[devops/concepts/gitops]]

## Overview

| Strategy | Downtime | Rollback speed | Risk | Resource cost |
|---|---|---|---|---|
| **Recreate** | Yes (gap between old/new) | Fast (re-deploy old) | High | Low |
| **Rolling update** | No | Medium (wait for rollback) | Medium | Low (gradual) |
| **Blue-Green** | No | Instant (flip LB) | Low | 2× (two envs) |
| **Canary** | No | Fast (route away) | Lowest | Low (small % traffic) |
| **Shadow / Dark launch** | No | N/A (no user impact) | None | 2× (duplicate traffic) |
| **A/B testing** | No | Fast | Low | Low |

---

## Recreate

Stop all old instances, start all new ones. Simple but causes downtime.

```
v1: [●●●] → stop → [] → start → [●●●] v2
              ↑ downtime here
```

**Use when:** single-instance apps, dev environments, DB schema migrations that aren't backward-compatible.

---

## Rolling Update

Gradually replace old pods with new ones. Kubernetes does this by default.

```
v1: [●●●●●●]
    [●●●●●○] ← one new pod (●=v1, ○=v2)
    [●●●●○○]
    [●●●○○○]
    [○○○○○○] ← all new
```

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1         # allow 1 extra pod during rollout
    maxUnavailable: 0   # never reduce below desired count
```

**Weakness:** during rollout, both versions serve traffic simultaneously. If APIs are not backward-compatible, this causes errors. Requires expand-contract pattern for DB changes.

---

## Blue-Green

Two identical environments (blue = current, green = new). Switch load balancer to green when ready.

```
        ┌─── Load Balancer ───┐
        │                     │
   Blue (v1)            Green (v2) ← deploy here, test
   [serving]            [standby → live]
```

```
Step 1: Deploy v2 to Green env (no traffic yet)
Step 2: Run smoke tests + integration tests against Green
Step 3: Flip LB: route 100% traffic to Green
Step 4: Blue stays up as instant rollback target
Step 5: After confidence period, decommission Blue
```

**Rollback:** flip LB back to Blue — takes seconds.  
**Cost:** requires 2× resources for the transition period.  
**DB consideration:** Green and Blue must be able to share the same DB during cutover (forward/backward schema compatibility).

---

## Canary

Route a small % of real traffic to the new version. Gradually increase if metrics stay healthy.

```
        ┌── Load Balancer ──────┐
        │                       │
   Stable v1 (95%)        Canary v2 (5%)
        │                       │
        └───────────────────────┘
                  ↓
         Watch: error rate, latency, business metrics
                  ↓
         Healthy? → increase canary % → 10% → 25% → 100%
         Unhealthy? → route 0% to canary → roll back
```

**Kubernetes + Argo Rollouts:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5      # 5% canary
        - pause: {duration: 10m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:             # auto-rollback if error rate > 5%
        templates:
          - templateName: error-rate
        args:
          - name: service
            value: my-app
```

**Best for:** high-traffic services where you want to validate on real users before full rollout. Catches issues that staging doesn't (real data, real traffic patterns).

---

## Feature Flags (Release Toggles)

Decouple code deployment from feature release. Code ships disabled; a flag enables it per user/cohort.

```python
if feature_flags.is_enabled("new_checkout_flow", user_id=user.id):
    return new_checkout_flow()
else:
    return old_checkout_flow()
```

**Types:**
- **Release toggle:** temporary; disable in-progress features; remove after launch
- **Experiment toggle:** A/B testing; varies by user cohort
- **Ops toggle:** kill switch for production; disable expensive features under load
- **Permission toggle:** enable for specific users/tenants

**Tools:** LaunchDarkly, Unleash, Flipt, GrowthBook (OSS)

**Danger:** toggle debt — old flags never cleaned up create combinatorial complexity. Enforce TTL and regular cleanup.

---

## Expand-Contract Pattern (Database + Code Deployments)

Zero-downtime DB schema changes alongside rolling code updates:

```
Phase 1 — Expand:
  Add new column (nullable, backward-compatible)
  Deploy code that WRITES to both old + new column
  (old code still works, reads old column)

Phase 2 — Migrate:
  Backfill old data into new column
  Deploy code that reads from NEW column
  (old column still populated for old code)

Phase 3 — Contract:
  Remove old column
  Deploy code that no longer uses old column
```

Never do Phase 1 + Phase 3 in the same deploy. Always 3 separate deploys across 3 time periods.

---

## Common Interview Q&A

**Q: How do you do zero-downtime deployment?**  
→ [[devops/patterns/zero-downtime-deployment]]

**Q: When would you choose canary over blue-green?**  
A: Canary when you want gradual validation on real traffic with automatic rollback triggers. Blue-green when you need instant, clean cutover and can afford 2× resources. Many teams use both: blue-green for major changes (schema migrations), canary for code changes.

**Q: What's the danger of a rolling update with incompatible API changes?**  
A: During rollout, v1 and v2 pods both serve traffic. If v2 changes an API contract (renamed field, removed endpoint), clients calling v1 get one response and v2 another. Solution: always version APIs, use backward-compatible changes, or use blue-green for breaking changes.

## Sources
- [[devops/patterns/zero-downtime-deployment]]
- [[devops/topics/ci-cd]]
