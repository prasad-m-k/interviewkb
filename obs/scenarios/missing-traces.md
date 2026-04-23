# Scenario: Missing or Broken Traces

**Topic:** [[obs/topics/tracing]]
**Related:** [[obs/concepts/opentelemetry]], [[obs/concepts/sampling]], [[obs/concepts/tempo-jaeger]]

## The Scenario

Engineers open Tempo/Jaeger to debug a latency issue and find:
- Traces stop midway — some services have spans, others don't.
- All traces show single-span root spans (no call hierarchy).
- Errors exist in logs but no corresponding traces are found.
- The trace search returns no results for the affected time window.

---

## Cause 1: Context Propagation Broken (Most Common)

**Symptom:** Trace spans from upstream service exist, but downstream service spans appear as disconnected root spans.

```
# What you see (broken):
Trace 1: checkout-svc POST /checkout  (0ms → 250ms)  ← missing payment spans

Trace 2: payment-svc POST /charge     (0ms → 230ms)  ← appears as a separate root trace!
```

**Root cause:** A service in the call chain is not forwarding the `traceparent` header to its downstream calls.

```python
# BUG: Makes downstream HTTP call without forwarding trace headers
def call_payment_service(order_id: str) -> dict:
    # This creates a new root span in payment-svc — trace is broken!
    response = requests.post("http://payment-svc/charge", json={"order_id": order_id})
    return response.json()

# FIX: Inject context into outgoing call
from opentelemetry.propagate import inject

def call_payment_service(order_id: str) -> dict:
    headers = {}
    inject(headers)  # adds traceparent, tracestate
    response = requests.post("http://payment-svc/charge", headers=headers, json={"order_id": order_id})
    return response.json()
```

**Auto-instrumentation fix:** Using `opentelemetry-instrumentation-requests` automatically handles header injection:
```bash
pip install opentelemetry-instrumentation-requests
opentelemetry-instrument python app.py
```

**Debugging approach:**
```bash
# Trace an actual request manually and check headers
curl -v http://checkout-svc/checkout \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"

# Check if payment-svc received the header by looking at its incoming request logs
{app="payment-svc"} | json | http_headers =~ ".*traceparent.*"
```

---

## Cause 2: Sampling Dropped the Trace

**Symptom:** Some traces exist for a service, but the specific trace you need (e.g., the one that errored) is missing.

**Root cause:** Head sampling at 1% dropped the trace before it was recorded.

**Fix for error traces:** Switch to tail sampling in the OTel Collector:
```yaml
processors:
  tail_sampling:
    policies:
      - name: errors-always
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 1000}
      - name: probabilistic-rest
        type: probabilistic
        probabilistic: {sampling_percentage: 1}
```

Full reference: [[obs/concepts/sampling]]

**Verification:** Check the OTel Collector's own metrics:
```promql
# What fraction of traces are being sampled?
otelcol_processor_tail_sampling_sampling_decision_latency_count{decision="sampled"}
  /
otelcol_processor_tail_sampling_sampling_decision_latency_count

# Dropped vs sampled
otelcol_processor_tail_sampling_sampling_decision_timer_latency_count
```

---

## Cause 3: OTel Collector is Down or Dropping Spans

**Symptom:** Traces were working, then suddenly no new traces appear. All services' spans are absent.

**Diagnosis:**
```bash
# Check Collector health
curl http://otel-collector:13133/    # health check endpoint (port 13133)

# Check Collector's own metrics (port 8888 by default)
curl http://otel-collector:8888/metrics | grep otelcol_exporter

# Key metrics
otelcol_exporter_sent_spans_total       # spans successfully exported
otelcol_exporter_send_failed_spans_total # spans that failed to export
otelcol_processor_batch_batch_send_size # batch sizes being sent
```

```promql
# Alert: Collector dropping spans
rate(otelcol_exporter_send_failed_spans_total[5m]) > 0
```

**Common causes:**
- Collector OOM (memory_limiter not configured)
- Backend (Tempo/Jaeger) is down
- Network policy blocking Collector → Tempo
- OTLP endpoint wrong (wrong port, wrong protocol gRPC vs HTTP)

**Fix memory issues:**
```yaml
processors:
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s
```

---

## Cause 4: Missing Spans in Async Workflows

**Symptom:** Trace shows spans from the producer service but nothing from the consumer (Kafka, SQS, RabbitMQ).

**Root cause:** The message payload doesn't carry the trace context.

```python
# BUG: Kafka producer doesn't inject trace context
def publish_event(event: dict):
    producer.send("events", json.dumps(event).encode())

# FIX: Inject context into Kafka message headers
from opentelemetry.propagate import inject

def publish_event(event: dict):
    headers = {}
    inject(headers)  # adds traceparent to headers dict
    kafka_headers = [(k, v.encode()) for k, v in headers.items()]
    producer.send("events", json.dumps(event).encode(), headers=kafka_headers)
```

```python
# Consumer: extract context from message headers
from opentelemetry.propagate import extract
from opentelemetry import trace

def consume_message(message):
    headers = {k: v.decode() for k, v in message.headers}
    ctx = extract(headers)  # restores the trace context
    with tracer.start_as_current_span("process_event", context=ctx):
        process(message.value)
```

---

## Cause 5: Clock Skew Between Services

**Symptom:** Traces exist but spans appear out of order or have negative durations.

```
# Wrong: child span starts before parent (clock skew)
checkout-svc   0ms → 250ms
payment-svc   -50ms → 230ms  ← started 50ms "before" parent???
```

**Root cause:** System clocks not synchronized between hosts. NTP drift of 50ms+ causes this.

**Fix:**
```bash
# Check NTP sync status
timedatectl status
chronyc tracking

# Force sync if drifted
chronyc makestep
```

In Kubernetes: NTP is managed at the node level. If traces show clock issues, check node NTP configuration:
```bash
kubectl get nodes -o json | jq '.items[].status.conditions[] | select(.type=="SystemClock")'
```

---

## Trace Search Returns Nothing

**Symptom:** Searching Tempo for `{resource.service.name="checkout-svc"}` returns no results.

**Checklist:**
1. Is Tempo receiving spans? Check `otelcol_exporter_sent_spans_total` from Collector.
2. Is the service name correct? `{resource.service.name="checkout-svc"}` vs `{resource.service.name="checkout"}`.
3. Is the time range correct? Tempo has default search range limits.
4. Is there an index? Tempo tag search requires the tag index feature enabled.

```bash
# Query Tempo API directly to test
curl "http://tempo:3200/api/search?service.name=checkout-svc&start=$(date -d '1 hour ago' +%s)&end=$(date +%s)"
```

---

## Interview Q&A

**Q: How do you debug a broken trace (spans disconnected)?**
A: First, verify the trace_id appears in the downstream service's logs (the span was recorded, just not linked). If it's there but as a new root span, the context propagation header wasn't forwarded. Check the outbound HTTP client in the upstream service — is it using auto-instrumentation or manual injection? If neither: add `opentelemetry-instrumentation-requests` or manual `inject(headers)` before the call.

**Q: Tail sampling requires all spans from a trace to arrive at the same Collector instance. How do you ensure this?**
A: Configure the load balancer in front of the Collector fleet to use consistent hashing on the `trace_id` — all spans from the same trace are routed to the same Collector replica. The OTel Collector's `loadbalancingexporter` handles this automatically when running a Collector fleet.

**Q: What is the difference between sampling rate and trace completeness?**
A: Sampling rate is the fraction of traces recorded. Completeness is whether each recorded trace has all its spans. You can have high sampling rate (100% of traces) but low completeness (only half the spans from each trace). Broken completeness is usually a context propagation problem; low sampling rate is a config decision.

## Sources
- [[obs/concepts/sampling]]
- [[obs/concepts/opentelemetry]]
- [[obs/concepts/tempo-jaeger]]
