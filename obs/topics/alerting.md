# Alerting

**Topic:** [[obs/overview]]
**Related:** [[obs/concepts/slo-burn-rate]], [[obs/patterns/alert-routing]], [[obs/scenarios/alert-fatigue]]

Survey of alerting: design philosophy, alert types, routing, escalation, and the burn-rate model.

---

## The Core Principle: Alert on Symptoms, Not Causes

The most important rule in alerting:

```
BAD:  Alert when CPU > 80%           — cause; might not affect users at all
BAD:  Alert when GC pause > 200ms    — cause; internal metric; might be fine
GOOD: Alert when p99 latency > 500ms — symptom; users are experiencing slowness
GOOD: Alert when error rate > 0.1%   — symptom; users are seeing errors
```

**Why?** Cause-based alerts flood on-call with false positives. A CPU at 85% might be fine — the service is handling load. A p99 at 2 seconds is always bad — users are suffering.

**Consequence:** Every alert should be (1) actionable and (2) user-impacting. If an alert fires and the on-call can do nothing about it, or if users aren't affected, it should not be a page.

---

## Alert Severity Tiers

| Tier | Meaning | Delivery | Response |
|---|---|---|---|
| **Critical / Page** | SLO is burning fast; user impact now | PagerDuty / OpsGenie call+SMS | Wake someone up immediately |
| **Warning / Ticket** | SLO burning slowly; will miss by end of window | Slack + ticket | Fix during business hours |
| **Info / Dashboard** | Informational trend | Grafana annotation | No response; use in postmortems |

**Alert fatigue** occurs when the pager is too noisy — on-call engineers start ignoring alerts. The fix is ruthlessly promoting noisy alerts from "page" to "ticket" or eliminating them entirely.

---

## SLO Burn Rate Alerting (The Google Model)

Traditional threshold alerting is brittle. The Google SRE Workbook's multi-window burn rate model is better.

### Burn Rate Formula
```
burn_rate = current_error_rate / (1 - SLO_target)
```

Example: SLO = 99.9% → error budget = 0.1% = 0.001
- Current error rate: 2% → burn rate = 0.02 / 0.001 = 20×
- At 20× burn rate, the monthly budget (720 hours) exhausts in: 720 / 20 = 36 hours

### Multi-Window Alert Rules

| Alert | Burn Rate | Short Window | Long Window | Severity |
|---|---|---|---|---|
| Immediate page | > 14.4× | 5m | 1h | Critical |
| Slow page | > 6× | 30m | 6h | Warning |
| Ticket | > 1× | 6h | 3d | Info |

```yaml
# Prometheus alerting rule (YAML)
- alert: HighErrorBudgetBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
    ) / 0.001 > 14.4
    and
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) /
      sum(rate(http_requests_total[1h]))
    ) / 0.001 > 14.4
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning at >14.4x rate"
    runbook: "https://wiki.internal/runbooks/high-error-rate"
```

The **dual window** (5m and 1h) prevents false positives from brief spikes — both the short-term rate and the recent trend must confirm the burn rate.

Full deep dive: [[obs/concepts/slo-burn-rate]]

---

## Alerting Anti-Patterns

| Anti-pattern | What happens | Fix |
|---|---|---|
| **Alert on every 5xx** | Pages for every single error, even expected ones | Alert on *rate* exceeding SLO threshold |
| **Alert on high CPU** | False positives; CPU can be high without user impact | Alert on p99 latency or saturation |
| **Alert without runbook** | On-call doesn't know what to do; slow MTTR | Every alert must have a linked runbook |
| **Missing `for:` duration** | Flapping alerts; fires on brief spikes | Use `for: 5m` to require sustained condition |
| **All alerts are critical** | "Boy who cried wolf" — pager fatigue → ignored | Use severity tiers strictly |
| **Alert on test environments** | Too noisy; burns trust | Route test alerts to a dev channel, not pager |
| **No alert deduplication** | Same alert fires 50 times before acknowledged | Alertmanager group_wait + group_interval |

---

## Alertmanager (Prometheus)

Alertmanager receives firing alerts from Prometheus and handles: routing, deduplication, inhibition, and silences.

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s          # wait before sending first notification (group formation)
  group_interval: 5m       # wait before sending update for same group
  repeat_interval: 4h      # re-notify if still firing after this interval
  receiver: 'default-pagerduty'

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: false

    - match:
        severity: warning
      receiver: 'slack-warnings'
      continue: false

    - match_re:
        service: "^(frontend|checkout)$"
      receiver: 'checkout-team-pagerduty'

inhibit_rules:
  # If cluster is down, suppress pod-level alerts (they're noise from root cause)
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '(PodCrashLooping|HighLatency|HighErrorRate)'
    equal: ['cluster']
```

Full reference: [[obs/patterns/alert-routing]]

---

## Dead Man's Switch

A "dead man's switch" (also: heartbeat alert) is an alert that fires when **no data is received** — the opposite of a threshold alert. Used to detect when monitoring itself breaks.

```yaml
# Alert fires if no samples received in 5 minutes from any Prometheus instance
- alert: PrometheusDown
  expr: absent(up{job="prometheus"}) == 1
  for: 5m
  annotations:
    summary: "Prometheus is not scraping targets — monitoring is blind"
```

An on-call SRE's nightmare: the service is down, but no alert fired because the monitoring system itself failed. Dead man's switches detect this.

---

## Runbooks: The Other Half of Alerting

Every alert is only as useful as its runbook. A runbook template:

```markdown
# Alert: HighErrorBudgetBurn

## What is happening?
HTTP 5xx rate is exceeding 14.4× the SLO burn rate. Users are experiencing errors.

## Immediate diagnosis steps
1. Check Grafana error rate dashboard: <link>
2. Which endpoints are erroring? `{job="api", status=~"5.."}`
3. Check recent deployments: `kubectl rollout history deployment/api -n prod`
4. Check downstream dependencies: DB connections, cache hit rate

## Common causes and fixes
- **New deployment:** Roll back with `kubectl rollout undo deployment/api -n prod`
- **DB connection pool exhausted:** Increase pool size in config; check for slow queries
- **Downstream service down:** Check <service> status; enable circuit breaker if available

## Escalation
- After 15 min without resolution: page the team lead
- After 30 min: declare SEV2 incident; open Zoom bridge

## Post-incident
File a postmortem within 48 hours at <link>
```

---

## Interview Questions

**Q: What is alert fatigue and how do you fix it?**
A: Alert fatigue occurs when too many alerts fire, the on-call starts ignoring them. Fix: (1) audit alert history — what fired in the last month? (2) for every alert that fired without requiring action, ask "should this be a ticket or eliminated?" (3) set `for: 5m` on all alerts to suppress transient spikes, (4) implement SLO burn rate alerting instead of raw thresholds.

**Q: What is an inhibition rule?**
A: Inhibition suppresses ("inhibits") alerts when a higher-priority alert is already firing. Example: if the database cluster is down, suppress individual pod-level alerts about high error rates (they're all caused by the DB outage — adding noise doesn't help).

**Q: What is the difference between group_wait, group_interval, and repeat_interval?**
A: `group_wait` — how long to buffer before sending the first notification (allows multiple related alerts to group). `group_interval` — how long to wait before sending a notification about new alerts that joined an existing group. `repeat_interval` — how long before re-notifying about a still-firing alert.

## Sources
- [[obs/concepts/slo-burn-rate]]
- [[obs/patterns/alert-routing]]
- [[obs/sources/google-sre-book-obs]]
