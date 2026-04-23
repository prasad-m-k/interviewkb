# Prometheus Official Documentation

**Type:** Official Documentation
**Source:** prometheus.io/docs
**Ingested:** 2026-04-22
**Topics covered:** [[obs/topics/metrics]], [[obs/concepts/prometheus]], [[obs/concepts/cardinality]]

## Most Interview-Relevant Sections

**Data Model:** Metric names + label sets → unique time series. The format is `metric_name{label="value"} number timestamp`.

**Metric Types:** Counter (rate/increase), Gauge (direct value), Histogram (percentile computation), Summary (deprecated for aggregation).

**PromQL:** The query language. Key functions: `rate()`, `irate()`, `histogram_quantile()`, `increase()`, `topk()`, `count by()`, `sum by()`.

**Recording Rules:** Precompute expensive queries as new time series. Critical for SLO calculations and dashboard performance.

**Alerting Rules:** YAML format with `expr:`, `for:`, `labels:`, `annotations:`. The `for:` clause requires the condition to be sustained before firing.

**Federation:** Prometheus scraping another Prometheus. Used for hierarchical aggregation (leaf → regional → global). Replaced by Thanos for most production deployments.

**Storage:** Local TSDB with default 15-day retention. Blocks compacted from 2h → 1d → 5d → ... Write-Ahead Log (WAL) for crash recovery.

## Key Facts for Interviews

- Prometheus uses **pull** (scrape) not push — this is a deliberate design decision for target health visibility.
- `up{job="...", instance="..."}=1` if last scrape succeeded, `0` if failed. Query `count(up == 0)` for down targets.
- Prometheus HA: run two identical instances, deduplicate with Thanos Querier. No leader election.
- Pushgateway: for short-lived batch jobs only. Not for long-running services.

## What it Updated
- Created [[obs/concepts/prometheus]]
- Informed [[obs/topics/metrics]]
- Informed [[obs/concepts/cardinality]]
