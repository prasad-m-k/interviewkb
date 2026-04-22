# Observability

**Topic:** [[devops/overview]]
**Related:** [[devops/concepts/observability-pillars]], [[devops/patterns/incident-response]]

## The Three Pillars

```
         ┌──────────────────────────────────────────┐
         │           Observability                  │
         │                                          │
         │  Metrics      Logs          Traces       │
         │  (what)       (why)         (where)      │
         │  Prometheus   Loki/ELK      Jaeger/Tempo  │
         └──────────────────────────────────────────┘
```

A system is **observable** if you can answer "what is happening and why" from external outputs alone — without needing to deploy new instrumentation.

Full details: [[devops/concepts/observability-pillars]]

---

## The Four Golden Signals (Google SRE)

For any user-facing service, monitor these four:

| Signal | What it measures | Alert when |
|---|---|---|
| **Latency** | Time to serve a request (p50, p95, p99) | p99 > SLO threshold |
| **Traffic** | Request rate (RPS, QPS) | Sudden drop or spike vs. baseline |
| **Errors** | Rate of failed requests (5xx, timeouts) | > 0.1% of requests |
| **Saturation** | How full is the service? (CPU %, queue depth, connection pool) | > 80% capacity |

---

## RED Method (Per Service)

For microservices, RED is simpler to apply than Four Golden Signals:

- **R**ate — requests per second
- **E**rrors — error rate
- **D**uration — latency distribution

---

## USE Method (Per Resource)

For infrastructure resources (CPU, disk, network):

- **U**tilization — % time the resource is busy
- **S**aturation — how much extra work is queued
- **E**rrors — error events

---

## Alerting Principles

**Alert on symptoms, not causes.**

```
Bad:   Alert when CPU > 80%              (cause — might not affect users)
Good:  Alert when p99 latency > 500ms   (symptom — users are affected)
```

**Alert levels:**
- **Page (wake someone up):** only when an SLO is burning — user impact is happening now
- **Ticket (fix during business hours):** degraded performance, trend approaching threshold
- **Dashboard (no alert):** informational; use in postmortems

**Avoid alert fatigue:** every alert that fires must be actionable. A noisy alert that gets ignored is worse than no alert.

---

## SLI / SLO / SLA / Error Budget

| Term | Definition | Example |
|---|---|---|
| **SLI** (Service Level Indicator) | A measured metric | 99.2% of requests < 200ms in last 30 days |
| **SLO** (Service Level Objective) | Target for an SLI | 99.5% of requests < 200ms |
| **SLA** (Service Level Agreement) | Contract with consequences | 99% uptime or credits issued |
| **Error Budget** | 1 − SLO; how much you can afford to break | 0.5% of requests can fail |

**Error budget policy:**
- Budget healthy → deploy freely, experiment
- Budget at 50% → slow down, focus on reliability
- Budget exhausted → freeze deploys, fix reliability only

---

## Observability Stack (Common)

| Layer | OSS option | Cloud-managed |
|---|---|---|
| Metrics collection | Prometheus | Datadog, CloudWatch |
| Metrics storage | Thanos / Cortex | Datadog, Grafana Cloud |
| Dashboards | Grafana | Datadog, CloudWatch |
| Log aggregation | Fluentd / Fluentbit | Datadog, CloudWatch Logs |
| Log storage/search | Elasticsearch / Loki | Datadog, CloudWatch Insights |
| Tracing | Jaeger / Tempo | Datadog APM, X-Ray |
| Instrumentation | OpenTelemetry | OTel (vendor-neutral) |
| Alerting | Alertmanager | PagerDuty + Datadog |

> **OpenTelemetry** is now the standard for instrumentation — one SDK, any backend. Always mention it as the vendor-neutral approach.

---

## Common Interview Q&A

**Q: Latency is high but error rate is zero. Where do you look?**  
→ [[devops/scenarios/high-latency-no-errors]]

**Q: How do you avoid cardinality explosions in Prometheus?**  
A: Never use high-cardinality labels (user ID, request ID, IP address) in Prometheus metrics — they create millions of time series and kill the database. Use logs or traces for per-request data; metrics for aggregated rates.

**Q: What's the difference between monitoring and observability?**  
A: Monitoring = watching known failure modes (dashboards, alerts for expected problems). Observability = the property of a system that lets you investigate *unknown* failures from its outputs. Observability requires structured logs, distributed traces, and high-cardinality metrics — monitoring alone isn't enough for modern distributed systems.

## Sources
- [[devops/concepts/observability-pillars]]
- [[devops/overview]]
