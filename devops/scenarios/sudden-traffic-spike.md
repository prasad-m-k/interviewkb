# Scenario: Sudden Traffic Spike

**Type:** Capacity / Scaling
**Difficulty:** Medium
**Frequency:** High — tests architecture thinking and operational depth

## The Question

*"Your service just got 10× its normal traffic. Latency is degrading. What do you do?"*

---

## What the Interviewer Is Testing

- Do you know the difference between scaling up vs out?
- Do you understand auto-scaling configurations?
- Can you protect upstream and downstream dependencies?
- Do you think about the queue, the DB, the cache — not just the app servers?
- Do you distinguish between a legit spike vs traffic attack?

---

## Immediate Response (First 5 Minutes)

### Step 1: Confirm it's a real traffic spike (not an attack)

```bash
# Traffic volume
kubectl top pods -n prod        # are pods actually under load?

# Grafana: check RPS graph
# Is it organic (gradual ramp)? Viral (sudden step)? Attack (constant high rate)?

# Check user agent / IP distribution
# Huge traffic from one IP or bot user agent → rate limit / block first
```

### Step 2: Scale out immediately

```bash
# Manual horizontal scale (buy time)
kubectl scale deployment my-app --replicas=20

# Or if HPA is already configured, check if it's firing:
kubectl get hpa my-app
# TARGETS: 85%/70%  MINPODS: 3  MAXPODS: 20  REPLICAS: 3
# If replicas haven't scaled yet → HPA lag (15–60s default)
```

### Step 3: Check each layer of the stack

```
Layer           Symptom                    Fix
─────────────────────────────────────────────────────────
Load Balancer   Connection limit hit       Increase LB capacity / add nodes
App pods        CPU throttling, OOM        Scale pods; increase requests/limits
DB connections  Connection pool exhausted  PgBouncer; reduce pool size per pod
DB itself       Lock contention, slow Q    Read replica; cache hot queries
Cache           Cache miss storm           Warm cache; increase TTL
Message queue   Queue depth growing        Scale consumers
Downstream APIs Rate limited by partner    Circuit breaker; graceful degradation
```

---

## HPA (Horizontal Pod Autoscaler) Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # scale out when CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # react quickly to spikes
      policies:
        - type: Percent
          value: 100                    # double replicas at a time
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # scale down slowly (avoid flapping)
```

---

## The Database Connection Problem

Every new pod opens `N` DB connections. With 20 pods × 10 connections = 200 connections. Most PostgreSQL configs cap at 100–200. You hit connection exhaustion before you hit pod CPU limits.

**Fix: PgBouncer (connection pooler)**
```
20 pods × 10 "connections" → PgBouncer → 20 real DB connections
```

```yaml
# Check current connections
kubectl exec -it postgres-pod -- psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# Add PgBouncer as a sidecar or dedicated service
# Set POOL_MODE=transaction (one DB conn serves many app requests)
```

---

## Load Shedding and Graceful Degradation

When you can't scale fast enough, **degrade gracefully** instead of crashing:

```python
# Option 1: Return cached/stale data
@app.route("/recommendations")
def get_recommendations():
    if current_load > THRESHOLD:
        return get_cached_recommendations()   # stale is better than error
    return compute_fresh_recommendations()

# Option 2: Disable expensive features
@app.route("/search")
def search():
    if current_load > THRESHOLD:
        return simplified_search()            # no ML ranking
    return full_search_with_ranking()

# Option 3: Rate limit per client
from flask_limiter import Limiter
limiter = Limiter(app, key_func=get_remote_address, default_limits=["100/minute"])
```

---

## Cluster Autoscaler vs KEDA

| | HPA | Cluster Autoscaler | KEDA |
|---|---|---|---|
| **Scales** | Pods | Nodes | Pods |
| **Trigger** | CPU/memory | Node resource pressure | Custom metrics (queue depth, HTTP requests, custom) |
| **Latency** | 15–30s | 1–5 min | Seconds (event-driven) |
| **Use case** | CPU-bound services | Node provisioning | Queue consumers, event-driven workloads |

**KEDA example (scale on SQS queue depth):**
```yaml
triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/my-queue
      queueLength: "10"    # scale out when > 10 msgs per pod
```

---

## Common Follow-Up Questions

**Q: "What if 10× traffic is sustained — it's a new normal?"**  
A: Right-size the HPA max; scale the DB (vertical + read replicas); evaluate if architecture handles it (consider async/queue-based processing, CDN for static content, database sharding).

**Q: "What's the difference between scaling up and scaling out?"**  
A: Scale up (vertical) = bigger machine, more CPU/RAM per instance. Scale out (horizontal) = more instances of the same size. Scale out is preferred for stateless services (resilient, cheaper, no single point of failure). Scale up is sometimes needed for stateful systems (DB primary).

**Q: "How would you design this service to handle 100× traffic?"**  
A: CDN for static/cacheable content; async processing via queues; read replicas for DB; distributed cache (Redis); stateless app pods (easy horizontal scale); event-driven architecture for spiky workloads.

## Sources
- [[devops/concepts/kubernetes-architecture]]
- [[devops/topics/containers-orchestration]]
