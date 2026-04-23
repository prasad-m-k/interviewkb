# Trace Sampling

**Topic:** [[obs/topics/tracing]]
**Related:** [[obs/concepts/opentelemetry]], [[obs/concepts/tempo-jaeger]]

## Why Sampling Exists

At scale, tracing every request is impossible:
- Google: ~10M QPS. 100% tracing = 10M spans/second to store.
- At 1KB/span: 10 GB/second = 864 TB/day. Impractical.

Sampling is the practice of recording only a fraction of traces while preserving statistical validity and ensuring important traces (errors, slow requests) are always captured.

---

## Head Sampling

Decision made at the **start** of the request — before its outcome is known.

```
Request arrives → flip a coin → 1% chance → trace it; 99% → don't
```

**Implementation (OTel SDK):**
```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased

# Sample 1% of root spans; respect parent's decision for child spans
sampler = ParentBased(root=TraceIdRatioBased(rate=0.01))

tracer_provider = TracerProvider(sampler=sampler, resource=resource)
```

| Pros | Cons |
|---|---|
| Zero overhead for non-sampled requests | Cannot guarantee errors are always sampled |
| Simple to implement | Rare events (1-in-1000 errors) have 99% chance of being missed |
| No buffering required | Cannot make decisions based on outcome |

**`ParentBased` vs `TraceIdRatioBased`:** Always use `ParentBased` in production. It respects the sampling decision from the upstream service — if service A decided to sample a request, service B will also sample it, keeping the full trace coherent. Without it, you get partial traces.

---

## Tail Sampling

Decision made at the **end** of the request — after all spans have been collected.

```
All spans buffered for 30s → look at complete trace → 
  if error or slow → keep 100%
  if healthy → keep 1%
```

**Requires: OTel Collector with tail sampling processor.** The application SDK cannot buffer — it doesn't have the full trace.

```yaml
# otel-collector: tail sampling processor
processors:
  tail_sampling:
    decision_wait: 30s       # wait this long for all spans to arrive
    num_traces: 100000       # max traces in buffer
    expected_new_traces_per_sec: 10000
    policies:
      # Always keep traces with errors
      - name: error-policy
        type: status_code
        status_code: {status_codes: [ERROR]}

      # Always keep slow traces (> 500ms root span)
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 500}

      # Keep 100% of traces from critical services
      - name: critical-service
        type: string_attribute
        string_attribute:
          key: resource.service.name
          values: [payment-svc, auth-svc]

      # Sample 1% of remaining healthy traces
      - name: probabilistic-fallback
        type: probabilistic
        probabilistic: {sampling_percentage: 1}
```

| Pros | Cons |
|---|---|
| 100% capture of errors and slow traces | Requires stateful buffer (OTel Collector) |
| No "lucky errors escape sampling" problem | 30s delay before decision → storage cost for buffered traces |
| Can sample based on any span attribute | Complex to configure and operate |
| Dramatically reduces storage for healthy traces | Multiple Collector instances must share state |

---

## Sampling Strategies Comparison

| Strategy | Capture errors? | Overhead | Complexity | Use when |
|---|---|---|---|---|
| **No sampling (100%)** | Yes | Very high | Low | Dev/staging only |
| **Head: probabilistic** | 1% chance | None (dropped early) | Low | High-QPS services, cost-sensitive |
| **Head: rate-limiting** | Only sampled | None | Low | Stable storage cost regardless of load |
| **Tail sampling** | Yes (100%) | Buffer overhead | High | Production — need to capture all anomalies |
| **Adaptive** | Yes | Medium | Very high | Ultra-scale; Google/Meta-level |

---

## Consistent Sampling (Multi-Service)

All services in a request's call chain must use the **same sampling decision**. Otherwise a trace is partially recorded — some spans exist, some don't — making the trace useless.

**How it works:**
1. The first service (at the edge) makes the sampling decision.
2. The decision is encoded in the `traceparent` header (`sampled` flag, bit 1).
3. Downstream services read this flag and respect it.
4. `ParentBased` sampler in OTel handles this automatically.

**Problem:** If a downstream service uses `TraceIdRatioBased` (not `ParentBased`), it may independently decide to *not* sample even though the upstream decided to sample. The upstream spans exist in the backend; the downstream spans don't. Result: broken trace.

---

## Sampling and SLO Accuracy

**Important:** If you use head sampling at 1%, your error rate metrics are still accurate (they come from Prometheus counters, which record every request). Sampling only affects trace data, not metrics.

However, if you derive error rates *from traces* (instead of metrics), a 1% sample would make your error rate estimates 100× less precise. Always compute SLIs from metrics, not from traces.

---

## Interview Questions

**Q: Why can't the application SDK do tail sampling?**
A: Tail sampling requires buffering the complete trace before making the decision. A span is emitted immediately when a service finishes its work — the SDK doesn't know when all downstream spans have completed. Only a centralized Collector that receives all spans from all services can buffer and make the decision.

**Q: What happens if two OTel Collector replicas each handle a different subset of spans from the same trace?**
A: Tail sampling breaks — neither replica has the complete trace to make a decision on. Fix: use consistent hashing to route all spans from the same trace to the same Collector replica (load balancer with `trace_id` affinity). Alternatively, use a distributed tail sampling system (OTel's experimental stateful Collector).

**Q: How do you handle sampling in a Kafka-based async pipeline?**
A: The producer injects the trace context (including `sampled` flag) into the Kafka message headers. The consumer reads the headers, extracts the context, and respects the sampling decision via `ParentBased` sampler. The gap between produce and consume is represented as two separate spans linked by the trace context, not a continuous span.

## Sources
- [[obs/concepts/opentelemetry]]
- [[obs/topics/tracing]]
