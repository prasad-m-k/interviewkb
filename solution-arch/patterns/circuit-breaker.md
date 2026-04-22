# Circuit Breaker

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related concepts:** [[solution-arch/patterns/bulkhead]], [[solution-arch/concepts/rate-limiting]]

## What it solves
In a microservices system, a slow or failing downstream service can cause callers to wait indefinitely, exhaust thread pools, and cascade failures across the system. The circuit breaker pattern **fails fast** when a dependency is unhealthy.

## Template / Skeleton

```
                     ┌─────────────────────────────────┐
                     │         Circuit Breaker          │
                     │                                  │
Request ────────────▶│  State Machine:                  │──▶ Dependency
                     │                                  │
                     │  CLOSED ──(failures > threshold)──▶ OPEN
                     │     ▲                            │    │
                     │     │     (half-open probe ok)   │    │
                     │  HALF-OPEN ◀────(timeout)─────── │◀───┘
                     └─────────────────────────────────┘
```

### States

```
CLOSED (normal operation):
  All requests pass through to dependency.
  Count failures. If failures exceed threshold → OPEN.

OPEN (tripped — failing fast):
  All requests immediately return fallback/error.
  No requests sent to dependency (protecting it from overload).
  After timeout period → HALF-OPEN.

HALF-OPEN (probing):
  Allow a small number of test requests through.
  If they succeed → CLOSED (recovery).
  If they fail   → OPEN again (still broken).
```

### Example Configuration
```yaml
circuit_breaker:
  failure_threshold: 5          # open after 5 consecutive failures
  success_threshold: 2          # close after 2 consecutive successes (half-open)
  timeout: 30s                  # stay open for 30 seconds
  request_volume_threshold: 20  # min requests before evaluating failure rate
```

## Signal phrases
- "Prevent cascading failures across services"
- "Fail fast when a downstream service is degraded"
- "Automatic recovery / self-healing after downstream recovers"
- Slow third-party API, flaky database connection

## Complexity

| State | Latency | Risk |
|-------|---------|------|
| CLOSED | Normal | Cascading failure possible |
| OPEN | Near-zero (immediate fail) | Requests dropped |
| HALF-OPEN | Normal (probe) | May trip open again |

## Fallback strategies when OPEN

```
1. Return cached stale data         (best UX)
2. Return a default/empty response  (graceful degradation)
3. Return HTTP 503 Service Unavailable
4. Queue the request for later      (if operation can be deferred)
```

## Circuit Breaker in Code (pseudo)

```python
breaker = CircuitBreaker(
    failure_threshold=5,
    recovery_timeout=30,
    fallback=lambda: CachedData.get_last()
)

@breaker
def call_payment_service(order):
    return payment_api.charge(order)
```

Libraries: Resilience4j (Java), Polly (.NET), Hystrix (legacy Java), resilience4j-python.

## Problems using this pattern
- [[solution-arch/scenarios/design-rate-limiter]]
- [[solution-arch/scenarios/high-availability-platform]]

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
