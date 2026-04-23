# Grafana

**Topic:** [[obs/topics/metrics]], [[obs/topics/logging]], [[obs/topics/tracing]]
**Related:** [[obs/concepts/prometheus]], [[obs/concepts/loki]], [[obs/concepts/tempo-jaeger]]

## What it is

Grafana is the standard open-source visualization and dashboarding platform for observability data. It connects to any data source (Prometheus, Loki, Tempo, Elasticsearch, CloudWatch, etc.) and renders dashboards, alerting, and correlations.

**The LGTM Stack** (Loki + Grafana + Tempo + Mimir/Prometheus) is the canonical open-source observability stack. Grafana is the single UI for all three pillars.

---

## Dashboard Design Principles

### Information Hierarchy
A well-designed Grafana dashboard follows a top-down drill-down pattern:

```
Level 1: Service Health Overview (1 row)
  → RED dashboard: Rate, Error, Duration for all services
  → SLO burn rate gauges
  
Level 2: Individual Service Detail (1 row per service)
  → p50/p95/p99 latency
  → Error rate by endpoint
  → Request volume (traffic)
  → Saturation (CPU, memory, connection pools)

Level 3: Infrastructure (1 row)
  → Node CPU/memory/disk/network
  → Pod restart counts
  → HPA scale events

Level 4: Dependencies (1 row)
  → DB query latency
  → Cache hit rate
  → Downstream service error rates
```

### Dashboard Variables (Templating)
Variables make dashboards reusable across environments, namespaces, and services:

```yaml
# In dashboard JSON (variables section)
- name: namespace
  type: query
  datasource: Prometheus
  query: "label_values(kube_pod_info, namespace)"
  multi: true
  includeAll: true

- name: service
  type: query
  datasource: Prometheus
  query: "label_values(http_requests_total{namespace=~\"$namespace\"}, service)"
```

Then use in panels: `{namespace=~"$namespace", service=~"$service"}`

---

## Panel Types and When to Use Them

| Panel | Use case | Example metric |
|---|---|---|
| **Time series** | Any metric over time | p99 latency, error rate, RPS |
| **Stat** | Single current value | Current error rate, uptime |
| **Gauge** | Value with min/max (% style) | CPU utilization, memory usage |
| **Bar gauge** | Comparison across N items | Error rate per service |
| **Table** | Multi-column structured data | Top 10 slow endpoints |
| **Heatmap** | Distribution over time | Request latency histogram |
| **Histogram** | Distribution at a point in time | Latency percentile distribution |
| **Logs panel** | Log stream display | Error logs from Loki |
| **Traces panel** | Trace search + gantt view | Traces from Tempo |
| **Geomap** | Geographic distribution | Traffic by region |
| **Node graph** | Service dependency topology | Service map |

---

## The Golden Signal Dashboard (Template)

```
Row 1: SLO Status
  - Stat: Current error rate
  - Stat: Error budget remaining (%)
  - Stat: SLO burn rate (current)
  - Time series: Error budget burn over 30 days

Row 2: Traffic
  - Time series: Requests per second (by endpoint)
  - Bar gauge: Top 10 endpoints by volume

Row 3: Errors
  - Time series: Error rate (5xx + 4xx)
  - Table: Error count by endpoint + status code
  - Logs panel: Recent error log lines (Loki, filtered by level=error)

Row 4: Latency
  - Time series: p50, p95, p99 latency
  - Heatmap: Request duration distribution (histogram)
  - Stat: p99 latency vs. SLO threshold

Row 5: Saturation
  - Time series: CPU utilization by pod
  - Time series: Memory working set vs. limit
  - Time series: DB connection pool usage
```

---

## Grafana Alerting

Grafana has its own alerting engine (v9+) that evaluates queries from any data source — not just Prometheus.

```yaml
# Grafana alert rule (Dashboard > Alert)
apiVersion: 1
groups:
  - orgId: 1
    name: SLO Alerts
    folder: Production
    interval: 1m
    rules:
      - uid: slo-error-rate
        title: High Error Rate
        condition: A
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: |
                sum(rate(http_requests_total{status=~"5.."}[5m]))
                / sum(rate(http_requests_total[5m]))
              intervalMs: 1000
              maxDataPoints: 43200
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: "Error rate > 1% for 5 minutes"
          runbook_url: "https://wiki.internal/runbooks/error-rate"
        labels:
          severity: critical
        isPaused: false
```

Grafana can route alerts to: PagerDuty, OpsGenie, Slack, email, webhooks.

---

## Correlating Metrics → Logs → Traces

This is Grafana's killer feature: navigating between all three pillars without leaving the UI.

### Metrics → Logs (Explore panel link)
```yaml
# In a Prometheus panel, add a "data link" to Loki
dataLinks:
  - title: "View logs for this service"
    url: '/explore?left={"datasource":"loki","queries":[{"expr":"{app=\"${__field.labels.service}\"}","refId":"A"}]}'
    targetBlank: true
```

### Metrics → Traces (Exemplars)
Enable exemplar display in Prometheus histogram panels:
```yaml
# In panel settings:
# Options > Exemplars: enabled
# Click an exemplar dot → opens trace in Tempo
```

### Logs → Traces (Derived fields)
```yaml
# In Loki data source config:
derivedFields:
  - name: TraceID
    matcherRegex: '"trace_id":"(\w+)"'
    url: '${__value.raw}'
    datasourceUid: tempo   # opens matching trace in Tempo
```

Result: in any log panel, `trace_id` values become clickable links that open the full trace.

---

## Grafana Dashboard as Code

Store dashboards in Git using `grafana-dashboard-exporter` or Grafonnet (Jsonnet library):

```jsonnet
// Grafonnet: Generate a dashboard programmatically
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;

dashboard.new(
  'Service Overview',
  tags=['production', 'sre'],
  time_from='now-1h',
)
.addRow(
  row.new('Traffic').addPanel(
    graphPanel.new('Requests per second')
    .addTarget(prometheus.target(
      'sum(rate(http_requests_total[5m])) by (service)',
      legendFormat='{{service}}'
    ))
  )
)
```

---

## Interview Questions

**Q: What is the difference between Grafana alerting and Prometheus alerting?**
A: Prometheus alerting (via Alertmanager) evaluates PromQL-only rules. Grafana alerting (v9+) can evaluate queries from any data source — including Loki, Elasticsearch, CloudWatch. Teams with a single Prometheus instance often use Prometheus alerting; teams with multiple data sources prefer Grafana alerting for unified alert management.

**Q: What is a Grafana data source?**
A: A configured connection to a backend (Prometheus, Loki, Tempo, MySQL, etc.). Each data source has a UID used in panel queries and data links. Data sources can be configured via the UI or as YAML in `datasources/` for GitOps.

**Q: How do you prevent dashboard sprawl?**
A: (1) Enforce a naming convention and folder structure. (2) Use dashboard-as-code (Grafonnet or Terraform Grafana provider) — dashboards live in Git, not just in the database. (3) Audit quarterly: delete dashboards not viewed in 90 days. (4) Create a small set of canonical "golden" dashboards for each service tier.

## Sources
- [[obs/concepts/prometheus]]
- [[obs/concepts/loki]]
- [[obs/concepts/tempo-jaeger]]
