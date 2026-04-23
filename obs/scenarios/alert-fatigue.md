# Scenario: Alert Fatigue

**Topic:** [[obs/topics/alerting]]
**Pattern:** [[obs/concepts/slo-burn-rate]], [[obs/patterns/alert-routing]]
**Companies:** [[obs/companies/google]], [[obs/companies/meta]]

## The Scenario

The on-call engineer pages their manager:
> "I'm getting paged 30+ times per week, half the alerts clear themselves, and I can't tell which ones are real. I've started ignoring the PagerDuty notifications."

This is **alert fatigue** — one of the most dangerous reliability anti-patterns. When engineers stop trusting the pager, real incidents go undetected.

---

## Why Alert Fatigue is Dangerous

- Engineers begin "alarm desensitization" — unconsciously tuning out notifications.
- Real SEV1 incidents are missed because they look like noise.
- On-call becomes a morale hazard; team turnover increases.
- Cognitive load from constant interruptions degrades engineer performance during business hours.

**Google's guideline:** No more than 2 actionable pages per on-call shift. If you're exceeding this consistently, your alerting needs work, not your engineers.

---

## Step 1: Audit Alert Volume

Before fixing anything, measure the problem:

```promql
# Alertmanager: alerts fired per day
increase(alertmanager_alerts_received_total[24h])

# Alerts by receiver (which team is being overwhelmed?)
increase(alertmanager_notifications_total[7d]) by (receiver)

# Alert firing duration (how long do alerts stay open?)
avg by (alertname) (alertmanager_alert_active_time_seconds)
```

Also pull the Alertmanager logs or PagerDuty history:
```bash
# From Alertmanager API: get all recent alerts
curl http://alertmanager:9093/api/v2/alerts | jq '.[] | {name: .labels.alertname, state: .status.state, starts: .startsAt}'
```

**Classification exercise:** For every alert that fired in the last month, classify it:
| Class | Definition | Action |
|---|---|---|
| Actionable | Required a specific on-call action | Keep; improve runbook |
| Self-resolving | Cleared without action | Lower severity or eliminate |
| Duplicate | Same root cause as another alert | Add inhibition rule |
| Flapping | Fired and resolved repeatedly | Add `for:` duration or remove |
| Informational | No action needed; just FYI | Move to Slack or eliminate |

---

## Step 2: Eliminate Self-Resolving Alerts

Self-resolving alerts (fire briefly, then clear without action) are the #1 source of noise.

**Fix: Add `for:` duration**
```yaml
# Before: fires on any 5xx spike, even 1-second transients
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.01

# After: must sustain for 5 minutes before firing
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.01
  for: 5m   # ← sustained condition required
```

**Tune the evaluation window:**
```yaml
# Narrow window = more sensitive to brief spikes
rate(http_requests_total{status=~"5.."}[1m]) > 0.01   # fires on 1-minute blip

# Wider window = smoothed; avoids transients
rate(http_requests_total{status=~"5.."}[15m]) > 0.01  # requires sustained 15-minute trend
```

---

## Step 3: Switch to SLO Burn Rate Alerting

Threshold-based alerts ("fire if error rate > 1%") produce too many false positives because they fire on any brief spike regardless of actual SLO impact.

Burn rate alerting fires only when the SLO is being consumed faster than the budget allows:

```yaml
# Before: fires whenever error rate exceeds 1%
- alert: HighErrorRate
  expr: job:http_error_ratio:rate5m > 0.01

# After: fires only when SLO budget is burning too fast
- alert: SLOBurnRateCritical
  expr: |
    job:http_error_ratio:rate5m > (14.4 * 0.001)  # 14.4x burn rate for 99.9% SLO
    and
    job:http_error_ratio:rate1h > (14.4 * 0.001)
  for: 2m
```

Result: a 5-second outage doesn't page. A sustained 1% error rate (burning 10× the budget) always pages.

Full reference: [[obs/concepts/slo-burn-rate]]

---

## Step 4: Add Inhibition Rules

If the database is down, suppress the 20 upstream service alerts that are just symptoms:

```yaml
inhibit_rules:
  - source_matchers:
      - alertname = "PostgreSQLDown"
    target_matchers:
      - alertname =~ "ServiceHighErrorRate|ServiceHighLatency"
    equal: ['cluster']

  - source_matchers:
      - alertname = "ClusterNotReachable"
    target_matchers:
      - severity = "warning"
    equal: ['cluster']
```

---

## Step 5: Tier Your Alerts

Not every alert needs to be a page. Create a clear severity model and enforce it:

```yaml
# CRITICAL → PagerDuty (wake someone up)
# Only when: SLO is actively burning, user impact now
labels:
  severity: critical

# WARNING → Slack #alerts (fix during business hours)
# When: trending toward SLO violation, no immediate user impact
labels:
  severity: warning

# INFO → Grafana annotation (no notification, visible in dashboard)
# When: informational trend, useful for postmortems
labels:
  severity: info
```

Enforce the model: if an alert labeled `critical` doesn't require waking someone up, downgrade it.

---

## Step 6: On-Call Review Process

After every on-call rotation, do a 30-minute review:

```markdown
## On-Call Review Template

Rotation: [Week of YYYY-MM-DD] | Engineer: [Name]

### Alert Audit
| Alert | Count | Required action? | Classification | Fix needed |
|-------|-------|------------------|----------------|------------|
| HighCPU-pod-a | 12 | No (auto-resolved) | Self-resolving | Add `for: 10m` |
| DBConnectionsHigh | 3 | Yes → pool increase | Actionable | Improve runbook |
| KubeJobFailed | 8 | No (cron, expected) | Informational | Move to Slack |

### Action Items
- [ ] Add inhibition rule for KubeJobFailed during maintenance windows — owner: @sre-team
- [ ] Raise DBConnections `for` from 1m to 5m — owner: @platform
```

---

## Measuring Improvement

After fixes, track:

```promql
# Alerts per week (trending down = good)
increase(alertmanager_alerts_received_total[7d])

# % of pages requiring action (should trend toward 100%)
# (manual tracking in postmortems or on-call review)

# Mean time between pages (MTBP) — should be > 4 hours per on-call
# (measure from PagerDuty API)
```

**Target state:** ≤ 2 actionable pages per shift, 95%+ of pages require on-call action.

---

## Interview Q&A

**Q: How do you convince a team that an alert should be removed?**
A: Data. Show the alert history: "This alert fired 40 times in the last month. Of those, 37 self-resolved in under 2 minutes with no action taken. The 3 times it was actionable were all captured by the SLO burn rate alert 10 minutes earlier. It adds noise without adding signal." The data usually wins the argument.

**Q: What's the risk of being too aggressive in reducing alerts?**
A: You could eliminate an alert that catches a rare but real issue. Mitigation: (1) don't delete alerts immediately — first downgrade to `severity: info` (no page) for a month; if nothing is missed, then delete. (2) ensure the SLO burn rate alert covers the symptom before removing the cause-based alert.

**Q: How do you handle flapping alerts?**
A: Flapping (fires, resolves, fires, resolves repeatedly) is usually caused by a metric oscillating around a threshold. Fix: (1) add hysteresis — alert at 90%, resolve at 70% (not at 90%); (2) use a longer `for:` duration; (3) use a wider time window in the query to smooth the metric.

## Sources
- [[obs/concepts/slo-burn-rate]]
- [[obs/patterns/alert-routing]]
- [[obs/sources/google-sre-book-obs]]
