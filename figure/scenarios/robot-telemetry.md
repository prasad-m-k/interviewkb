# Robot Fleet Telemetry and Observability

**Difficulty:** Hard
**Topic:** Infrastructure design, observability
**Companies:** [[figure/index]]
**Related:** [[devops/concepts/observability-pillars]], [[sre/concepts/slo-sli-sla]]

---

## Problem

Design the observability stack for Figure's humanoid robot fleet. The system must collect, store, and make queryable: robot health metrics, sensor telemetry, software logs, and incident signals — from 1,000 robots in real-time, with retention for post-incident analysis and ML training data.

---

## Clarifying Questions

- What are the most critical signals for production safety? (Joint overtemperature? Battery low? Software crash?)
- What is the expected data volume per robot per second?
- Who are the consumers of this data? (Ops team for alerts, ML team for training, engineering for debugging)
- What retention is required? (Incident replay? 90 days? Indefinitely for training data?)
- What are the latency requirements for alerts? (Safety alerts: sub-second? Performance: minutes OK?)

---

## The Data Problem: Scale and Variety

A humanoid robot with sensors generates approximately:

| Sensor Type | Frequency | Data Rate (per robot) |
|---|---|---|
| Joint encoders (30+ joints) | 1000 Hz | ~10 KB/s |
| Force/torque sensors (hands) | 500 Hz | ~5 KB/s |
| RGB cameras (2-6) | 30 FPS @ 1080p | ~60 MB/s raw; ~500 KB/s compressed |
| Depth cameras (1-2) | 30 FPS | ~5 MB/s compressed |
| IMU (6-axis) | 200 Hz | ~2 KB/s |
| Thermal sensors | 10 Hz | ~1 KB/s |
| Battery / power | 10 Hz | ~1 KB/s |
| Software logs | Event-driven | ~10 KB/s at normal ops |

**Full sensor stream per robot:** terabytes per hour if stored raw. This is not feasible at fleet scale. Observability design is fundamentally about what data to keep, at what fidelity, for how long.

**Key decision: streaming telemetry vs. on-robot buffering**
- Streaming everything in real-time: requires high bandwidth per robot (hundreds of Mbps), challenging in factory WiFi
- On-robot buffering with selective upload: keep low-fidelity telemetry streaming; buffer high-fidelity data locally; upload on trigger (incident, end-of-shift, maintenance window)

---

## Three-Tier Data Model

```
Tier 1: Real-time Health Metrics (always streaming, low fidelity)
  • Joint temperatures, battery %, error codes, software heartbeat
  • Compressed to ~50 KB/s per robot
  • Latency: < 1s to observability backend
  • Retention: 30 days (rolling)
  • Consumer: ops alerting, fleet health dashboard

Tier 2: Operational Telemetry (sampled, medium fidelity)
  • Joint positions, forces, velocities at 10 Hz (downsampled from 1000 Hz)
  • Selected camera frames (1 FPS keyframes)
  • ~1 MB/s per robot
  • Latency: < 30s acceptable
  • Retention: 90 days
  • Consumer: engineering debugging, performance analysis, ML training samples

Tier 3: Incident Recordings (triggered, full fidelity)
  • All sensor streams at full frequency, triggered by safety alert or explicit request
  • Stored locally on robot (ring buffer, last 30 minutes)
  • Uploaded on incident trigger or at end of shift
  • 60-200 MB/s per robot during recording; stored to object store
  • Retention: 1 year minimum (may be needed for insurance/regulatory)
  • Consumer: incident investigation, safety review, ML model improvement
```

---

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│  ROBOT (Edge)                                                  │
│                                                               │
│  • Sensor aggregator process collects all sensor data         │
│  • Lightweight agent (telegraf or custom) extracts Tier 1     │
│  • On-robot ring buffer: stores last 30 min full fidelity     │
│  • Local log storage: /var/log/figure/ with rotation          │
│  • Watchdog process: detects software crashes, triggers alert │
└───────────────────────────────────────────────────────────────┘
        │ Tier 1 (50 KB/s), Tier 2 (1 MB/s, batched)
        │ Factory WiFi / Ethernet
        ▼
┌───────────────────────────────────────────────────────────────┐
│  FACTORY EDGE COLLECTOR (on-prem, per facility)               │
│                                                               │
│  • Aggregates from all robots on local network                │
│  • Handles WiFi disruptions: buffers locally up to 10 min     │
│  • Tier 1: forwards to cloud Prometheus remote-write          │
│  • Tier 2: buffers and forwards to cloud object store         │
│  • Runs InfluxDB or Victoria Metrics for local Tier 1 cache   │
│  • Provides local dashboard (Grafana) for ops floor           │
└───────────────────────────────────────────────────────────────┘
        │ Stable WAN link (Azure ExpressRoute or Site-to-Site VPN)
        ▼
┌───────────────────────────────────────────────────────────────┐
│  CLOUD OBSERVABILITY BACKEND                                  │
│                                                               │
│  Tier 1 (metrics):                                            │
│  • Prometheus / Thanos or VictoriaMetrics cluster             │
│  • Grafana for dashboards, alerting rules                     │
│  • PagerDuty / OpsGenie integration for on-call pages         │
│                                                               │
│  Tier 2 (structured telemetry):                               │
│  • Azure Data Lake / S3-compatible store (MinIO self-hosted)  │
│  • Apache Parquet format for efficient querying               │
│  • Queried by ML teams for training data                      │
│  • Apache Spark / Databricks for batch analysis               │
│                                                               │
│  Tier 3 (incident recordings):                                │
│  • Object store with lifecycle policies (warm → cold → glacier)│
│  • Video indexed by incident_id and robot_id                  │
│  • Accessed via incident review tooling                       │
│                                                               │
│  Logs (all tiers):                                            │
│  • Loki or Elasticsearch for log aggregation and search       │
│  • Structured JSON logs from robot software                   │
│  • Queryable by robot_id, timestamp, severity, component      │
└───────────────────────────────────────────────────────────────┘
```

---

## Key Metrics to Define

### Safety Metrics (P0 — alert < 1s)

| Metric | SLO Breach Threshold | Alert Action |
|---|---|---|
| Joint temperature | > 80°C on any joint | Page on-call + send stop command to robot |
| Battery level | < 10% and not docked | Warn; < 5% = force park sequence |
| Safety system heartbeat | Missing > 2s | Immediate stop; page critical |
| Unexpected contact force | > 150N (tunable) | Stop arm motion; alert |
| Software watchdog | Process crash | Restart + page on-call |

### Operational Metrics (P1 — alert < 60s)

| Metric | Meaning |
|---|---|
| Cycle time deviation | Robot taking >20% longer than baseline for a given task |
| Error rate by task type | Failed pick attempts / total pick attempts |
| Fleet availability | % of robots in production-ready state |
| Software version distribution | % of fleet on target version |
| Network packet loss per robot | Connectivity quality indicator |

### Infrastructure Metrics (standard SRE)
Standard Prometheus exporters for the cloud infrastructure: node_exporter, kube-state-metrics, and application-level metrics from the fleet management service.

---

## Alerting Philosophy

**Safety alerts:** page immediately, 24/7, no suppression. A robot harming a human is a company-ending event.

**Operational alerts:** page during business hours or on-call for off-hours. These are availability issues, not safety issues.

**Toil alerts:** avoid. An alert that fires more than twice per week and requires human action but no safety risk is an automation opportunity, not a recurring alert to live with.

For alert fatigue specifically: Figure's manufacturing environment will generate many transient anomalies (a robot briefly miscounts parts; a sensor flickers). Implement alert dampening (fire only if condition persists for N seconds) and correlate related alerts (10 robots simultaneously showing high joint temp = network issue preventing cooling fans from responding, not 10 separate incidents).

---

## Local vs. Cloud: The Connectivity Problem

Factory WiFi is shared, congested, and not always reliable. The observability stack must:
1. **Never drop safety alerts** — safety signals must have a dedicated, low-latency path with priority QoS
2. **Degrade gracefully** — if WAN link to cloud drops, local Grafana/InfluxDB remains operational; robots continue operating
3. **Buffer and replay** — when WAN reconnects, Tier 2 telemetry is replayed to cloud without gaps

**Design choice:** separate network paths for Tier 1 (safety + metrics) vs. Tier 2 (bulk telemetry). Tier 1 uses QoS-prioritized VLAN; Tier 2 uses best-effort with bandwidth throttling so it never starves Tier 1.

---

## Training Data Pipeline (Intersection with ML Team)

Tier 2 and Tier 3 data are also the primary source for ML model improvement:
- Telemetry from successful tasks → positive training examples
- Telemetry from failed tasks (failed grasp, collision) → failure mode analysis + targeted augmentation
- Incident recordings → rare edge case coverage

The observability system must support ML-friendly access:
- Label API: ML team can tag a time window (start, end, robot_id, label) to create training samples
- Dataset export: generate Parquet/TFRecord exports filtered by label, date range, robot type
- Data lineage: track which training data came from which robot software version

---

## SLOs for the Observability System

The observability system itself needs SLOs:

| SLI | Target |
|---|---|
| Safety alert delivery latency (robot sensor → on-call page) | P99 < 5 seconds |
| Tier 1 metric freshness (age of latest datapoint) | P99 < 10 seconds |
| Log ingestion latency | P99 < 60 seconds |
| Tier 2 telemetry export availability (ML team can query) | 99.5% |
| Incident recording retrieval (access Tier 3 data) | < 15 minutes from request |

---

## Interview Follow-Ups to Anticipate

- "Your factory WiFi drops for 20 minutes. What happens to your monitoring? What data is lost? What is preserved?"
- "A robot falls in the BMW factory. Walk me through how you use your observability stack to reconstruct what happened."
- "The ML team says they need all raw sensor data for training, but that's 200 MB/s per robot. How do you handle this?"
- "Your Prometheus cluster falls behind on ingestion because 50 new robots were added at once. What do you do?"
- "How do you distinguish 'robot hardware is degrading' from 'software bug causing erratic behavior' from telemetry alone?"

---

## Sources
- [[devops/concepts/observability-pillars]]
- [[sre/concepts/slo-sli-sla]]
- [[figure/company-context]]
