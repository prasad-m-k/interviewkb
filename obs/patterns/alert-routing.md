# Alert Routing with Alertmanager

**Topic:** [[obs/topics/alerting]]
**Related:** [[obs/concepts/prometheus]], [[obs/concepts/slo-burn-rate]]

## What Alertmanager Does

Alertmanager receives firing alerts from Prometheus (or Grafana) and handles:
1. **Deduplication:** Multiple Prometheus replicas may send the same alert — Alertmanager deduplicates.
2. **Grouping:** Multiple related alerts (all pods in a namespace crashing) are batched into one notification.
3. **Routing:** Send alerts to the right team, via the right channel.
4. **Inhibition:** Suppress lower-priority alerts when a higher-priority alert is already firing.
5. **Silences:** Temporarily suppress alerts during maintenance windows.

---

## Routing Tree

Alertmanager routing is a **tree**: each alert matches a node, and the first matching route wins (unless `continue: true`).

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/..."

route:
  # Default receiver (catch-all)
  receiver: 'slack-general'
  group_by: ['alertname', 'cluster']
  group_wait: 30s          # Wait for more alerts before sending first notification
  group_interval: 5m       # Wait before sending notifications about new alerts in group
  repeat_interval: 4h      # Re-notify if alert is still firing

  routes:
    # Critical: page immediately
    - matchers:
        - severity = critical
      receiver: 'pagerduty-critical'
      group_wait: 0s         # Don't wait; page immediately
      repeat_interval: 1h
      continue: false        # Don't fall through to other routes

    # Team-specific routing
    - matchers:
        - team = payments
      receiver: 'payments-team-slack'
      routes:
        # Nested: payments + critical → payments PagerDuty
        - matchers:
            - severity = critical
          receiver: 'payments-team-pagerduty'

    # Infrastructure: separate on-call
    - matchers:
        - alertname =~ "Node.*|Kube.*"
      receiver: 'infra-team-pagerduty'

    # Catch-all warning: ticket only
    - matchers:
        - severity = warning
      receiver: 'jira-tickets'

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PD_INTEGRATION_KEY>'
        description: '{{ template "pagerduty.default.description" . }}'
        severity: critical

  - name: 'slack-general'
    slack_configs:
      - channel: '#alerts'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

  - name: 'jira-tickets'
    webhook_configs:
      - url: 'http://jira-webhook-bridge:8080/webhook'
```

---

## Grouping (Deduplication)

Grouping prevents alert storms. When 50 pods crash simultaneously, you don't want 50 separate pages:

```yaml
route:
  group_by: ['alertname', 'cluster', 'namespace']
  # All PodCrashLooping alerts in the same namespace/cluster → one notification
  group_wait: 30s     # Wait 30s for more alerts to join the group
  group_interval: 5m  # After first notification, wait 5m before sending updates
```

**group_by labels matter:** Group too broadly (just `alertname`) and one notification covers different severity problems. Group too narrowly (every label) and you get one notification per pod.

---

## Inhibition Rules

Inhibition suppresses "child" alerts when a "parent" alert is firing — preventing noise when a higher-level alert already captures the situation.

```yaml
inhibit_rules:
  # If the whole cluster is down, suppress pod-level alerts
  - source_matchers:
      - alertname = "ClusterDown"
    target_matchers:
      - alertname =~ "PodCrashLooping|HighLatency|DBConnectionsExhausted"
    equal: ['cluster']

  # If the DB is down, suppress upstream service connection errors
  - source_matchers:
      - alertname = "DatabaseDown"
      - service = "postgres"
    target_matchers:
      - alertname = "ServiceHighErrorRate"
      - reason = "db_connection_error"
    equal: ['cluster', 'namespace']

  # If a critical alert fires, suppress warnings for the same service
  - source_matchers:
      - severity = "critical"
    target_matchers:
      - severity = "warning"
    equal: ['service', 'cluster']
```

---

## Silences

Silences temporarily suppress alerts during planned maintenance. Created via the Alertmanager UI or API:

```bash
# Create a silence via amtool CLI
amtool silence add \
  alertname=~"Node.*" \
  cluster=us-east-1 \
  --duration=4h \
  --comment="Node maintenance window: replacing node-3"

# List active silences
amtool silence list

# Expire a silence
amtool silence expire <silence_id>
```

**Best practice:** Always set an expiry. Forgotten silences are one of the most common causes of missed alerts.

---

## On-Call Integration

### PagerDuty
```yaml
receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: '<PAGERDUTY_ROUTING_KEY>'
        description: '{{ .CommonAnnotations.summary }}'
        severity: '{{ if eq (index .Alerts 0).Labels.severity "critical" }}critical{{ else }}warning{{ end }}'
        details:
          alert_count: '{{ len .Alerts }}'
          runbook: '{{ (index .Alerts 0).Annotations.runbook_url }}'
          dashboard: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
```

### OpsGenie
```yaml
receivers:
  - name: opsgenie-critical
    opsgenie_configs:
      - api_key: '<OPSGENIE_API_KEY>'
        message: '{{ .CommonAnnotations.summary }}'
        priority: P1
        tags: '{{ range .Alerts }}{{ .Labels.service }},{{ end }}'
```

### Slack with Rich Formatting
```yaml
receivers:
  - name: slack-alerts
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#alerts-prod'
        title: '{{ if eq .Status "resolved" }}✅ RESOLVED: {{ else }}🔥 FIRING: {{ end }}{{ .CommonAnnotations.summary }}'
        text: |
          *Severity:* {{ .CommonLabels.severity }}
          *Service:* {{ .CommonLabels.service }}
          *Runbook:* {{ (index .Alerts 0).Annotations.runbook_url }}
          {{ range .Alerts }}
          • *Alert:* {{ .Annotations.description }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}{{ if eq (index .Alerts 0).Labels.severity "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'
        actions:
          - type: button
            text: 'Runbook'
            url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
          - type: button
            text: 'Dashboard'
            url: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
```

---

## Alert Notification Template Best Practices

Every alert notification should contain:
1. **What is broken** — service name, alert name
2. **How bad** — severity, how many instances
3. **Impact** — users affected, SLO budget being burned
4. **What to do first** — runbook link
5. **Where to look** — dashboard link, Grafana explore link

```yaml
annotations:
  summary: "API error rate {{ printf \"%.2f\" $value }}% exceeds SLO threshold"
  description: |
    Service {{ $labels.service }} in {{ $labels.namespace }} has an error rate of
    {{ printf "%.2f" $value }}%, exceeding the 0.1% SLO threshold.
    Burn rate: approximately {{ div $value 0.001 | printf "%.0f" }}x.
  runbook_url: "https://wiki.internal/runbooks/{{ $labels.service }}-high-errors"
  dashboard_url: "https://grafana.internal/d/service-overview?var-service={{ $labels.service }}"
```

---

## Interview Questions

**Q: What is the difference between group_wait and group_interval?**
A: `group_wait` is how long Alertmanager waits before sending the *first* notification for a new group — allowing multiple related alerts to accumulate into one message. `group_interval` is how long to wait before sending follow-up notifications about *new* alerts that join an already-notified group.

**Q: When would you use `continue: true` in a routing rule?**
A: When an alert should be sent to *multiple* receivers. By default, the first matching route wins. `continue: true` passes the alert to the next sibling route after matching, allowing it to be routed to both the team Slack *and* the global PagerDuty.

**Q: How do you test alerting rules without triggering real pages?**
A: (1) Use `amtool check-config alertmanager.yml` to validate configuration. (2) Use the Alertmanager test webhook to inspect routing without delivery. (3) Use `promtool check rules rules.yml` for Prometheus alert rule validation. (4) Set up a staging Alertmanager with routes pointing to a test Slack channel.

## Sources
- [[obs/topics/alerting]]
- [[obs/concepts/slo-burn-rate]]
