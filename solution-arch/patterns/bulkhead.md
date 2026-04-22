# Bulkhead Pattern

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related concepts:** [[solution-arch/patterns/circuit-breaker]], [[solution-arch/concepts/rate-limiting]]

## What it solves
Isolates failures within a section of the system so that one overloaded or failing component cannot exhaust shared resources (thread pools, connections) and bring down the entire system. Named after ship compartments that prevent flooding one section from sinking the whole vessel.

## Template / Skeleton

```
WITHOUT Bulkhead:
  ┌────────────────────────────────────────┐
  │          Shared Thread Pool (50)       │
  │                                        │
  │  Slow Service A eats all 50 threads ──▶│ Service B starved
  │  → Service B gets no threads          │ → entire app hangs
  └────────────────────────────────────────┘

WITH Bulkhead:
  ┌──────────────────┐  ┌──────────────────┐
  │ Pool A (20 threads)│  │ Pool B (30 threads)│
  │ [Svc A calls]    │  │ [Svc B calls]    │
  │                  │  │                  │
  │ Svc A slow:      │  │ Pool B unaffected│
  │ only Pool A full │  │ Svc B still works│
  └──────────────────┘  └──────────────────┘
```

## Types of Bulkheads

### Thread Pool Isolation
```
Service A ──▶ Thread Pool A (max 20)
Service B ──▶ Thread Pool B (max 30)
Service C ──▶ Thread Pool C (max 10)

If Service A's pool fills: queue or reject A's requests
Service B and C continue normally
```

### Connection Pool Isolation
```
Payment DB ──▶ Connection Pool (max 10 connections)
Analytics DB──▶ Connection Pool (max 20 connections)
Cache       ──▶ Connection Pool (max 50 connections)

Slow analytics queries don't consume payment DB connections
```

### Process / Container Isolation
```
Service A: dedicated pod, CPU/memory limits set
Service B: dedicated pod, CPU/memory limits set

Service A OOMKilled → only Service A affected
Service B continues running
```

## Bulkhead + Circuit Breaker Together

```
Request ──▶ Bulkhead (limit concurrent calls)
                │
                ▼
         Circuit Breaker (detect failure rate)
                │
                ▼
           Dependency

Bulkhead: prevents resource exhaustion
Circuit Breaker: prevents request to broken dependency
Together: comprehensive resilience
```

## Signal phrases
- "Isolate failure to one service"
- "Prevent cascading resource exhaustion"
- "Noisy neighbour problem"
- "One slow API draining the thread pool"

## Complexity

```
Thread pool isolation:  O(n pools) configuration; overhead per pool
Connection pool:        Standard in most DB drivers (configure sizes)
Process isolation:      Container resource limits (Kubernetes requests/limits)
```

## Tuning Bulkhead Sizes

```
Observe peak concurrent requests per downstream service
Set pool size = peak × 1.5 (headroom) or use Little's Law:
  
  Pool size = throughput × average latency
  
  Example: 100 req/sec × 200ms = 20 concurrent requests needed
  → Set pool to 20–25 threads
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
