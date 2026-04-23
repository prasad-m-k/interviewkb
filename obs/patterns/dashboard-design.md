# Dashboard Design

**Topic:** [[obs/topics/metrics]]
**Related:** [[obs/concepts/grafana]], [[obs/patterns/four-golden-signals]]

## Core Principles

### 1. Dashboards Answer Questions
Every dashboard should have a stated purpose: "Is checkout healthy?" or "What is causing high latency?" A dashboard with no stated purpose becomes a dumping ground.

### 2. Top-Down Drill-Down
Structure from coarse (system health) to fine (individual component):
```
Level 1: Is anything broken?          (Red/green health indicators)
Level 2: Which service is broken?     (Service-level RED metrics)
Level 3: What specifically is wrong?  (Endpoint, pod, DB query)
Level 4: Infrastructure root cause    (CPU, memory, network, disk)
```

### 3. Time Alignment
All panels should share the same time range. Use Grafana's global time picker, not per-panel overrides (causes confusion during incidents).

### 4. Context Alongside Numbers
Numbers alone are meaningless without context:
- Show SLO threshold as a horizontal reference line on latency graphs.
- Show deployment markers (annotations) on graphs so spikes correlate to code changes.
- Show p50, p95, p99 together — not just p99.

---

## Dashboard Anatomy

### Header Row (Always First)
```
[ Service: $service ] [ Env: $env ] [ Time: last 1h ]

Stat: Current error rate    Stat: Error budget remaining    Stat: p99 latency
[  0.02%  ✓  ]             [  87% remaining  ]              [  142ms  ]
```

### Mandatory Rows for Any Service
1. **Traffic** — RPS/QPS over time by endpoint or service
2. **Errors** — Error rate + error count + top error messages (log panel)
3. **Latency** — p50/p95/p99 time series + heatmap for distribution
4. **Saturation** — CPU, memory, connection pools
5. **SLO Status** — Error budget burn gauge

---

## Annotation: Deployment Markers

One of the highest-value additions to any dashboard. Correlate metric changes with code deploys instantly:

```python
# Post a Grafana annotation via API on every deploy
import requests

def post_deploy_annotation(service: str, version: str, grafana_url: str, api_key: str):
    response = requests.post(
        f"{grafana_url}/api/annotations",
        headers={"Authorization": f"Bearer {api_key}"},
        json={
            "tags": ["deploy", service],
            "text": f"Deployed {service} {version}",
            "time": int(time.time() * 1000),
        }
    )
    return response.json()
```

In CI/CD (GitHub Actions / Jenkins):
```yaml
- name: Annotate Grafana on deploy
  run: |
    curl -X POST \
      -H "Authorization: Bearer $GRAFANA_API_KEY" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"Deployed $SERVICE_NAME $IMAGE_TAG\", \"tags\": [\"deploy\", \"$SERVICE_NAME\"]}" \
      https://grafana.example.com/api/annotations
```

---

## Variable Design

Good variables make one dashboard work for all services, namespaces, and environments:

```yaml
Variables (in order):
  1. env:       prod | staging | dev
  2. cluster:   derived from env (prod → cluster-a, cluster-b)
  3. namespace: query Prometheus for namespaces in $cluster
  4. service:   query Prometheus for services in $namespace
  5. pod:       query Prometheus for pods of $service
```

**Pattern:** Cascading variables — each filters the options for the next. This prevents "service=checkout in env=staging with cluster=prod-cluster" impossible combinations.

---

## Heatmap vs. Time Series for Latency

**Time series (p99):** Shows whether p99 is trending up, but hides the shape of the distribution.

**Heatmap (histogram):** Shows how the distribution is shifting — is the p99 tail getting longer, or is the whole distribution shifting up?

```
Time series (p99):
│ 500ms ┤████
│ 400ms ┤████████                    ← just one number per point
│ 300ms ┤████████████

Heatmap (distribution):
│ 500ms │ ░░░░░░▒▒▒▒▒               ← shows concentration of requests
│ 400ms │ ░░░▒▒▒▒███████▒▒▒
│ 300ms │ ░▒▒████████████████▒░
│ 200ms │ ░███████████████████████░  ← most requests here
```

Use heatmap when you suspect bimodal distributions — fast cache hits + slow DB misses show up as two bands.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| More than 20 panels on one dashboard | Cognitive overload; panels too small to read | Split into linked dashboards |
| Raw counter values | Always increasing; meaningless spikes on restart | Use `rate()` or `increase()` |
| Averages for latency | Hides tail; SLO violations invisible | Use `histogram_quantile(0.99, ...)` |
| No reference lines | Can't see if metric exceeds SLO | Add threshold line at SLO target |
| Panels from different time sources | Graphs out of sync during incidents | Use global time picker consistently |
| "Junk drawer" dashboards | Everything is on one dashboard | Purpose-built dashboards for each use case |
| Graph colors with no convention | Errors in green; healthy in red | Convention: red=bad, green=good, yellow=warning |

---

## Dashboard-as-Code (Grafonnet / Terraform)

```hcl
# Terraform Grafana provider
resource "grafana_dashboard" "service_overview" {
  config_json = templatefile("${path.module}/dashboards/service-overview.json.tmpl", {
    service     = var.service_name
    datasource  = grafana_data_source.prometheus.uid
    loki_uid    = grafana_data_source.loki.uid
    tempo_uid   = grafana_data_source.tempo.uid
  })
  folder = grafana_folder.production.id
}
```

Benefits: dashboards are code-reviewed, version-controlled, reproducible across environments.

---

## Interview Questions

**Q: How do you ensure dashboards are useful during an incident?**
A: (1) Test the dashboard before the incident: "if p99 spiked to 2s right now, would I be able to see why from this dashboard?" (2) Include deployment annotations. (3) Include runbook link in the dashboard title row. (4) Ensure all panels use the same time range variable. (5) Keep the incident-relevant panels in the top row so they're visible without scrolling.

**Q: What's wrong with using averages in dashboards?**
A: Average latency masks tail experiences. If 1% of users get a 10-second response and 99% get a 50ms response, average ≈ 150ms — looks fine, but 1% of users are suffering. SLOs should be based on percentiles. Replace averages with p50, p95, p99.

**Q: How do you handle dashboard proliferation?**
A: (1) Dashboard-as-code in Git — creates visibility into creation/modification. (2) Enforce a folder structure with owners. (3) Track dashboard view counts via Grafana API; delete dashboards with no views in 90 days. (4) Publish a "canonical" set of dashboards per service tier — teams customize but start from a standard template.

## Sources
- [[obs/concepts/grafana]]
- [[obs/patterns/four-golden-signals]]
