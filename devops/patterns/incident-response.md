# Pattern: Incident Response

**Topic:** [[devops/overview]]
**Related:** [[devops/topics/observability]], [[devops/scenarios/production-outage]]

## The Framework: Detect → Triage → Mitigate → Resolve → Learn

```
Alert fires / user report
         │
    ┌────▼─────┐
    │  DETECT  │  Acknowledge alert; declare incident if warranted
    └────┬─────┘  Set severity (P1/P2/P3); page incident commander
         │
    ┌────▼─────┐
    │  TRIAGE  │  What is the blast radius? Who is affected?
    └────┬─────┘  Is it still ongoing? What changed recently?
         │
    ┌────▼──────────┐
    │   MITIGATE    │  Stop the bleeding FIRST — rollback, reroute, scale
    └────┬──────────┘  Don't wait for root cause to mitigate
         │
    ┌────▼──────────┐
    │   RESOLVE     │  Restore normal service; confirm metrics healthy
    └────┬──────────┘  Identify root cause
         │
    ┌────▼──────────┐
    │    LEARN      │  Blameless postmortem; action items with owners + due dates
    └───────────────┘
```

---

## Severity Levels

| Severity | Definition | Response |
|---|---|---|
| **P0 / SEV-1** | All users affected; core functionality broken | Immediate page; all hands; 15-min updates |
| **P1 / SEV-2** | Partial user impact; major feature broken | Page on-call; 30-min updates |
| **P2 / SEV-3** | Minor degradation; workaround exists | Ticket; fix in 24h |
| **P3 / SEV-4** | Cosmetic issue or edge case | Backlog |

---

## Roles

| Role | Responsibility |
|---|---|
| **Incident Commander (IC)** | Coordinates the response; makes decisions; owns communication |
| **Tech Lead** | Drives the technical investigation |
| **Communications Lead** | Status page, stakeholder updates, customer communication |
| **Scribe** | Documents timeline in shared doc as it happens |

---

## Mitigation Playbook

**Always prefer reversible mitigation first:**

```
1. Rollback deploy              ← most common; takes < 5 min
2. Feature flag off             ← instant, no deploy needed
3. Reroute traffic              ← blue-green flip, regional failover
4. Scale up                     ← if capacity issue
5. Kill circuit breaker         ← isolate failing dependency
6. Shed load (rate limit)       ← protect the system if overwhelmed
```

**Do NOT:** try to hotfix in production while the incident is ongoing. Rollback first, fix in dev, redeploy.

---

## Postmortem Template

```markdown
## Incident Report: [Title]

**Date:** YYYY-MM-DD
**Duration:** Xh Ym
**Severity:** P1
**Impact:** 15% of users unable to checkout; $Xk revenue impact

## Timeline
| Time (UTC) | Event |
|---|---|
| 10:34 | Alert fires: checkout error rate > 5% |
| 10:37 | On-call acknowledges; incident declared |
| 10:42 | Identified: new payment-svc v2.3.1 deployed at 10:28 |
| 10:45 | Rollback to v2.3.0 initiated |
| 10:51 | Error rate returns to baseline; incident resolved |

## Root Cause
payment-svc v2.3.1 had a missing null check on card_type field.
Cards without card_type (older accounts) caused an unhandled exception.

## Contributing Factors
- Integration tests did not cover null card_type scenario
- Canary period was 5 minutes — too short to catch the tail case (1% of users)

## Action Items
| Item | Owner | Due |
|---|---|---|
| Add null card_type test case | @dev-team | 2026-04-28 |
| Extend canary period to 30 min | @platform | 2026-04-25 |
| Add per-error-type alerting | @sre | 2026-04-30 |
```

---

## Common Interview Q&A

**Q: "Walk me through how you'd respond to a production incident."**  
→ [[devops/scenarios/production-outage]]

**Q: What makes a good postmortem?**  
A: Blameless — no finger-pointing, focus on system and process failures. Concrete action items with owners and due dates (not vague "we'll be more careful"). Timeline reconstructed from logs. Published widely so the whole org learns, not just the team that was paged.

**Q: How do you prevent alert fatigue?**  
A: Alert on symptoms (user-facing impact), not causes. Every alert must be actionable. Require runbooks for every alert. Regularly review alert noise (weekly alert review meeting). If an alert fires and the response is always "ignore it" — delete it.

## Sources
- [[devops/topics/observability]]
- [[devops/overview]]
