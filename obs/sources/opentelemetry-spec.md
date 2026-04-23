# OpenTelemetry Specification

**Type:** Specification / Documentation
**Source:** opentelemetry.io/docs/specs/otel
**Ingested:** 2026-04-22
**Topics covered:** [[obs/topics/tracing]], [[obs/topics/metrics]], [[obs/topics/logging]]

## What the Spec Covers

The OTel specification defines: the data model for each signal (traces, metrics, logs), the API for producing telemetry, the SDK for processing and exporting, and the OTLP wire protocol for transmission.

## Key Concepts from the Spec

**Semantic Conventions:** Standardized attribute names (e.g., `http.method`, `db.system`, `k8s.pod.name`) that ensure different frameworks and backends understand each other. Interviewing engineers should know the major conventions.

**Resource vs. Span Attributes:** Resource attributes describe the entity (service.name, host.name) and appear on all telemetry from that entity. Span attributes describe the specific operation and appear only on that span.

**Exemplars:** Sample `trace_id` values attached to metric observations, enabling the metrics→traces navigation workflow.

**Context Propagation:** W3C Trace Context is the standard. The `traceparent` header carries trace_id, span_id, and sampling decision. All OTel-instrumented services must propagate this header.

## What it Updated
- Created [[obs/concepts/opentelemetry]]
- Informed [[obs/topics/tracing]]
- Informed [[obs/concepts/sampling]]
