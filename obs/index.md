---
tags:
  - index
  - observability
  - monitoring
  - metrics
  - logging
  - tracing
  - interview-prep
---

# Observability Index
Last updated: 2026-04-22

## Overview
- [[obs/overview]] — Coverage map, study strategy, the three pillars at a glance

## Topics (Survey Pages)
- [[obs/topics/metrics]] — Metric types, cardinality, time-series DBs, PromQL, RED/USE/Golden Signals
- [[obs/topics/logging]] — Structured logs, log pipelines, LogQL, log levels, sampling
- [[obs/topics/tracing]] — Spans, traces, context propagation, sampling strategies, TraceQL
- [[obs/topics/alerting]] — Alert design philosophy, routing, escalation, burn-rate alerting

## Concepts (Deep Dives)
- [[obs/concepts/prometheus]] — Data model, metric types, PromQL, federation, remote write, Thanos/Cortex
- [[obs/concepts/opentelemetry]] — OTel SDK, OTLP, auto-instrumentation, Collector pipeline, signals
- [[obs/concepts/grafana]] — Dashboard design, panels, variables, templating, Loki+Tempo integration
- [[obs/concepts/loki]] — Loki architecture, push vs pull, LogQL, label design, chunk/index storage
- [[obs/concepts/tempo-jaeger]] — Distributed tracing backends, TraceQL, Tempo architecture, Jaeger vs Tempo
- [[obs/concepts/slo-burn-rate]] — Error budget math, burn rate, multi-window alerting, alert thresholds
- [[obs/concepts/cardinality]] — What causes explosions, high-cardinality label patterns, detection, remediation
- [[obs/concepts/sampling]] — Head vs tail sampling, probabilistic, adaptive, rule-based; when each applies
- [[obs/concepts/ebpf-observability]] — Zero-instrumentation observability, network flows, CPU flame graphs, Cilium

## Patterns
- [[obs/patterns/four-golden-signals]] — Latency, Traffic, Errors, Saturation — with PromQL for each
- [[obs/patterns/red-use-method]] — RED for services; USE for resources; when to use which
- [[obs/patterns/alert-routing]] — Alertmanager trees, grouping, inhibition, silences, on-call routing
- [[obs/patterns/dashboard-design]] — Information hierarchy, drill-down, golden signal dashboards, variables

## Scenarios (Interview-Style Debugging)
- [[obs/scenarios/high-latency-no-errors]] — p99 spike with 0% error rate — step-by-step diagnosis
- [[obs/scenarios/alert-fatigue]] — Too many noisy alerts — how to measure, triage, and eliminate
- [[obs/scenarios/cardinality-explosion]] — Prometheus OOM from label abuse — detection and cleanup
- [[obs/scenarios/missing-traces]] — Trace gaps, broken context propagation, sampling blindspots

## Companies
- [[obs/companies/google]] — Monarch, Dapper/OpenCensus, Borgmon; observability at 10M+ time series
- [[obs/companies/meta]] — Scuba, ODS, Laser, TAO monitoring; Perfpipe, Pixel-based RUM
- [[obs/companies/apple]] — Privacy-first observability, on-device diagnostics, MetricKit

## Flashcards
- [[obs/flashcards/obs-top15]] — Top 15 observability interview Q&A; Prometheus, OTel, alerting, tracing

## Sources
- [[obs/sources/google-sre-book-obs]] — SRE Book chapters 6 & 10: monitoring philosophy, four golden signals
- [[obs/sources/opentelemetry-spec]] — OTel specification: signals, context, propagators
- [[obs/sources/prometheus-docs]] — Official Prometheus docs: data model, PromQL, alerting rules
