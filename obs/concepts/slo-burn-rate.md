# SLO Burn Rate and Error Budget

**Topic:** [[obs/topics/alerting]], [[obs/topics/metrics]]
**Related:** [[obs/concepts/prometheus]], [[obs/patterns/alert-routing]], [[sre/concepts/slo-sli-sla]]

## The Vocabulary

| Term | Definition | Example |
|---|---|---|
| **SLI** | A measured metric reflecting service quality | 99.2% of requests < 200ms |
| **SLO** | Target for the SLI | 99.5% of requests < 200ms |
| **SLA** | Contractual commitment with penalties | 99% availability or credits issued |
| **Error Budget** | 1 − SLO — how much failure is acceptable | 0.5% of requests may fail per month |
| **Burn Rate** | How fast the error budget is consumed | 10× = monthly budget exhausted in 72 hours |

**SLO > SLA always.** Set SLO = 99.9%, SLA = 99%. The gap is your incident buffer.

---

## Error Budget Math

```
SLO = 99.9% → error budget = 0.1%
Monthly: 0.001 × 30 × 24 × 60 = 43.2 minutes of acceptable downtime
```

| SLO | Monthly downtime |
|---|---|
| 99% | 7.3 hours |
| 99.9% | 43.2 minutes |
| 99.99% | 4.3 minutes |
| 99.999% | 25.9 seconds |

---

## Burn Rate

```
burn_rate = current_error_rate / (1 - SLO_target)

SLO = 99.9%, current error rate = 1%:
  burn_rate = 0.01 / 0.001 = 10×
  Budget exhausts in: 30 days / 10 = 3 days
```

| Burn Rate | Budget exhausted in |
|---|---|
| 1× | 30 days |
| 6× | 5 days |
| 14.4× | ~50 hours |
| 60× | 12 hours |
| 720× | 1 hour |

---

## Multi-Window Burn Rate Alerting

| Alert | Short Window | Long Window | Burn Rate | Severity |
|---|---|---|---|---|
| Page Now | 5m | 1h | > 14.4× | Critical |
| Page Now | 30m | 6h | > 6× | Warning |
| Ticket | 6h | 3d | > 1× | Low |

Two windows prevent false positives from brief spikes — both must confirm.

### Prometheus Recording + Alert Rules

```yaml
groups:
  - name: slo-burn-rate
    rules:
      - record: job:http_error_ratio:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          / sum(rate(http_requests_total[5m])) by (job)

      - record: job:http_error_ratio:rate1h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h])) by (job)
          / sum(rate(http_requests_total[1h])) by (job)

      - record: job:http_error_ratio:rate6h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[6h])) by (job)
          / sum(rate(http_requests_total[6h])) by (job)

      - alert: SLOBurnRateCritical
        expr: |
          job:http_error_ratio:rate5m > (14.4 * 0.001)
          and
          job:http_error_ratio:rate1h > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "SLO burning at >14.4x rate ({{ $labels.job }})"

      - alert: SLOBurnRateWarning
        expr: |
          job:http_error_ratio:rate5m > (6 * 0.001)
          and
          job:http_error_ratio:rate6h > (6 * 0.001)
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "SLO burning at >6x rate ({{ $labels.job }})"
```

---

## SLO by Service Type

| Type | Good event definition |
|---|---|
| Sync API | status not 5xx AND latency < 300ms |
| Batch pipeline | item processed within expected window |
| Storage | object not permanently lost |
| CDN | byte served within latency threshold |

---

## Error Budget Policy

| Budget remaining | Action |
|---|---|
| > 50% | Deploy freely; experiment |
| 25–50% | Slow down; review incidents |
| < 25% | No feature deploys; reliability work only |
| 0% | Full freeze until budget recovers |

Write the policy in a document signed by SRE and product leads **before** the first incident. Without it, every budget exhaustion triggers a political negotiation.

---

## Interview Q&A

**Q: Why is 100% SLO wrong?**
Zero error budget = no deployments, no experiments. Users can't distinguish 99.999% from 100%. Set a realistic target and invest the saved engineering time in features.

**Q: How do you define an SLO for a batch pipeline?**
"99% of jobs complete within 5 minutes of their scheduled start." Measure (jobs on time) / (jobs expected).

**Q: What's different about burn rate vs. threshold alerting?**
A threshold fires identically for a 1-minute spike and a 2-week slow burn — different severity, same alert. Burn rate alerts distinguish by measuring rate-of-consumption relative to the budget window.

## Sources
- [[sre/concepts/slo-sli-sla]]
- [[obs/sources/google-sre-book-obs]]
