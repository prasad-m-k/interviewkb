# Scenario: "Your Deploy Just Broke Production"

**Type:** Incident Response
**Difficulty:** Medium–High
**Frequency:** Almost every senior DevOps interview

## The Question

*"You deployed a new version 30 minutes ago. You're getting paged — error rate just spiked to 40%, latency is 10×. Walk me through exactly what you do."*

---

## What the Interviewer Is Testing

- Do you **mitigate first** or do you root-cause first? (mitigate first is correct)
- Do you follow a structured process or thrash randomly?
- Do you know the tools? (kubectl, helm, metrics, logs)
- Can you communicate clearly under pressure?
- Do you understand blameless postmortems?

---

## The Answer (Talk Through Every Step)

### Step 1: Acknowledge and declare (0–2 min)

```
- Acknowledge the alert immediately
- Open incident channel / war room
- Declare severity (P1 if > 10% users affected)
- Page incident commander if not you
- Start a shared doc / incident timeline
```

### Step 2: Confirm the blast radius (2–5 min)

```
What is broken?
- Which service? Which endpoint? All users or subset?
- Is the error 500s or timeouts? Cascading failure?

Dashboard checks:
- Grafana/Datadog: error rate graph — when did it start?
- Latency graph: p50/p95/p99 — all elevated or just p99?
- Traffic graph: unusual spike/drop (DDoS? Bot traffic? Legit spike?)
- Saturation: CPU/memory/disk/connection pool — anything at 100%?
```

### Step 3: Correlate with the deploy (5–8 min)

```bash
# When was the deploy?
kubectl rollout history deployment/my-app
# or: check CI/CD deploy timestamp

# Is the timing correlated?
# Spike started at 14:32, deploy was at 14:28 → very likely related

# What changed?
git log v2.3.0..v2.3.1 --oneline
git diff v2.3.0 v2.3.1
```

### Step 4: Rollback immediately (8–12 min)

**Don't wait to find root cause. Rollback now.**

```bash
# Kubernetes rollback
kubectl rollout undo deployment/my-app

# Monitor rollback
kubectl rollout status deployment/my-app

# Helm rollback
helm rollback my-app 1    # roll back to previous release

# Watch the metrics
# Error rate should drop within 2–3 minutes of rollback completing
```

### Step 5: Confirm recovery (12–15 min)

```
- Error rate back to baseline? (< 0.1%)
- Latency back to normal? (p99 within SLO)
- All pods Running and Ready?
- No more pages?
→ Incident resolved; keep monitoring for 30 min
```

### Step 6: Root cause analysis (post-recovery)

```bash
# Check logs from the bad version
kubectl logs -l app=my-app --since=30m | grep ERROR

# Check if it's correlated with a specific error type
# Example: "NullPointerException in PaymentService.process"
# Cause: new code path doesn't handle null card_type

# Check if it's infrastructure (not code)
# - Did a DB migration run alongside the deploy?
# - Did a config map change?
# - Did a dependency version change?
```

### Step 7: Blameless postmortem

```
Write within 24–48h:
- Timeline (from logs, not memory)
- Root cause
- Contributing factors (why didn't tests catch it?)
- Action items with owners and due dates
- Never: "developer should have been more careful"
Always: "what process or tooling would catch this automatically?"
```

---

## Common Follow-Up Questions

**Q: "What if rollback isn't available / makes things worse?"**  
A: Forward-fix: cherry-pick a targeted patch to a new branch, run CI, deploy the fix. Meanwhile: feature-flag the broken code path off, rate-limit the affected endpoint, or serve a degraded response (graceful degradation) instead of an error.

**Q: "What if the issue isn't related to the deploy?"**  
A: Broaden scope: infrastructure (DB running out of connections, disk full, cert expiry), external dependencies (third-party API down), traffic spike, data issue (corrupt record triggering a bug that was already there).

**Q: "How do you prevent this in the future?"**  
A: Canary deploy with automated rollback on error rate thresholds. Better integration test coverage. Extend canary duration. Database migration in a separate deploy from code change. Improve monitoring to catch the specific error type.

## Sources
- [[devops/patterns/incident-response]]
- [[devops/concepts/deployment-strategies]]
