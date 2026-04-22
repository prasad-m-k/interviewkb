---
tags: [sre, networking, interview-prep]
---
# Networking & LB Scenarios — Flashcards

> [!question]- 1. "Users report intermittent 502s from the ALB. Backend pods are Running and Ready. What do you check?"
> **502 = ALB got a bad response (or no response) from the backend.**
> 
> Checklist in order:
> 1. **ALB access logs** — which target IP is returning 502s? Is it one pod or all?
> 2. **Target group health** — is the pod passing the health check? `aws elbv2 describe-target-health`
> 3. **Backend logs** — is the app crashing mid-request? Check for `SIGPIPE`, connection reset errors
> 4. **Keep-alive mismatch** — ALB keeps connections open; if backend closes idle connections faster than ALB's idle timeout (60s default), ALB sends a request on a dead connection → 502. Fix: set backend keep-alive timeout > ALB idle timeout, or set ALB idle timeout lower.
> 5. **Response timeout** — backend is alive but taking > ALB timeout (default 60s). Fix: raise timeout or fix the slow path.
> 6. **Pod OOM** — container killed mid-request (Exit 137). `kubectl describe pod` → `OOMKilled`.
> 
> **Most common root cause in prod:** keep-alive timeout mismatch (nginx default keep-alive = 75s, ALB default = 60s → nginx closes first → ALB 502).

---

> [!question]- 2. "A service works fine inside the cluster but times out from outside. Port 443 is open on the LB."
> Work inward from the LB:
> ```
> External client → Cloud LB → NodePort → ClusterIP → Pod
> ```
> 1. **LB → node:** `curl http://NodeIP:NodePort` from a bastion — does it respond?
>    - No → Security group / firewall blocking NodePort range (30000–32767)
> 2. **Node → pod:** `kubectl exec` into a pod and `curl ClusterIP:port` — does it respond?
>    - No → kube-proxy not running, or NetworkPolicy blocking ingress
> 3. **Pod health:** `kubectl get endpoints svc-name` — does the Service have pod IPs listed?
>    - Empty endpoints → readiness probe failing; pod not added to endpoints
> 4. **TLS cert:** if 443, check cert CN/SANs match the hostname. `openssl s_client -connect host:443`
> 5. **Ingress rule:** if using Ingress, check host/path rules match exactly (case-sensitive, trailing slash matters)

---

> [!question]- 3. "Database connections are exhausted. Application is throwing 'too many connections'. You have 5 minutes."
> **Immediate mitigation (< 2 min):**
> ```bash
> # See who's holding connections
> psql -c "SELECT count(*), state, application_name FROM pg_stat_activity GROUP BY state, application_name ORDER BY count DESC;"
> # Kill idle connections older than 10 min
> psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND query_start < now() - interval '10 minutes';"
> ```
> 
> **Root cause options:**
> | Symptom | Cause | Fix |
> |---|---|---|
> | Connections from many app pods | No connection pool | Add PgBouncer |
> | Connections from one pod | Connection leak (not closing after use) | Fix app code; use context manager |
> | Connections = idle in transaction | Long-running uncommitted txn | Set `statement_timeout`, `idle_in_transaction_session_timeout` |
> | Spike after deploy | New pods opened connections before old ones closed | Rolling deploy + pool = 2× connections briefly; pre-scale PgBouncer |
> 
> **PgBouncer transaction pooling:** 100 app "connections" → 10 real DB connections. Set `pool_mode=transaction`.

---

> [!question]- 4. "TCP connection succeeds but HTTP request hangs forever. No timeout on either side. Debug."
> The TCP 3-way handshake completes (so routing, firewall, and port binding are fine). The hang is at the HTTP layer.
> 
> Suspects:
> 1. **Backend accepted the TCP connection but isn't reading from the socket** — process is alive but stuck (deadlock, GC pause, thread pool full). Check: `ss -tnp | grep :8080` — connection in `ESTABLISHED` but no data moving.
> 2. **Proxy buffering** — nginx received headers but is waiting for the full request body before forwarding (large file upload with `proxy_request_buffering on`).
> 3. **HTTP/1.1 pipelining issue** — second request on a keep-alive connection never processed because server is still "handling" the first (which it actually finished but forgot to flush).
> 4. **Load balancer has a request but backend is at max concurrency** — request is queued inside the LB or app server (gunicorn worker pool full, Tomcat thread pool full).
> 
> **Diagnose:**
> ```bash
> tcpdump -i eth0 port 8080 -A   # see if any bytes flow after the initial request
> ss -tnp                         # ESTABLISHED connections and their queue sizes
> kubectl exec pod -- cat /proc/$(pgrep gunicorn)/status | grep Threads
> ```

---

> [!question]- 5. "You're seeing asymmetric latency: requests from us-east users are fast, eu-west users are 3× slower. Same service."
> Classic **geographic routing / CDN miss** problem.
> 
> Checklist:
> 1. **Is there a CDN?** If so, are eu-west requests hitting a nearby PoP or going all the way to us-east origin?
>    - Check: `curl -I https://example.com` from eu-west — look at `X-Cache`, `CF-RAY`, `X-Served-By` headers
> 2. **DNS routing** — is Route 53 / Cloud DNS returning the nearest regional endpoint to eu-west users?
>    - `dig +short example.com` from eu-west — what IP? Is it a us-east IP?
> 3. **No multi-region deployment** — there's literally only a us-east deployment. eu-west pays the transatlantic RTT (~80ms × 2 for TLS = +320ms minimum).
>    - Fix: deploy to eu-west region, or add CloudFront/Cloudflare in front
> 4. **TLS handshake cost** — 2 RTTs × 80ms = 320ms just for TLS. TLS 1.3 reduces to 1 RTT = 160ms. HTTP/2 connection reuse eliminates per-request handshake.
> 5. **TCP slow start** — new connections start with small congestion window. Long-distance connections never ramp up fully for small responses. Fix: HTTP/2 multiplexing, or increase `tcp_initcwnd`.

---

> [!question]- 6. "Explain what happens to in-flight requests during a Kubernetes rolling deploy."
> **The problem:** K8s sends SIGTERM to the old pod AND removes it from endpoints simultaneously. But iptables rules take 1–3 seconds to propagate across all nodes. During that window, new requests can still arrive at the terminating pod.
> 
> ```
> t=0s  Pod deletion triggered
> t=0s  K8s removes pod from Endpoints object
> t=0s  K8s sends SIGTERM to container
> t=0s–2s  iptables rules still route traffic to this pod (propagation lag)
> t=0s  preStop hook fires: sleep 10  ← keeps pod alive to drain
> t=10s  preStop completes → app receives SIGTERM
> t=10s–30s  app drains in-flight requests
> t=30s  terminationGracePeriodSeconds → SIGKILL
> ```
> 
> **Required config for zero dropped requests:**
> ```yaml
> lifecycle:
>   preStop:
>     exec:
>       command: ["/bin/sh", "-c", "sleep 10"]   # wait for iptables propagation
> terminationGracePeriodSeconds: 30
> # + app must handle SIGTERM gracefully (stop accepting, finish in-flight)
> # + maxUnavailable: 0 in rolling update strategy
> # + readinessProbe defined (traffic only routes to ready pods)
> ```

---

> [!question]- 7. "Consistent hashing vs modulo hashing — why does it matter for a cache?"
> **Modulo hashing:** `server = hash(key) % N`
> - Add 1 server: N becomes N+1 → almost every key maps to a different server
> - **Cache hit rate drops to ~0%** temporarily — all requests go to the DB
> - At scale (100M keys, adding 1 cache node): nearly all keys remap = thundering herd on DB
> 
> **Consistent hashing:** keys and servers placed on a virtual ring (0 to 2³²)
> ```
> Key → hash → position on ring → walk clockwise to nearest server
> 
> Adding server D between B and C:
>   Only keys between B and D remap to D
>   All other keys unchanged
>   ~1/N keys remapped (vs (N-1)/N for modulo)
> ```
> 
> **Virtual nodes:** each physical server gets V positions on the ring (V=150 typical). Smooths out uneven key distribution. If one server has fewer virtual nodes, it handles less traffic — use for capacity weighting.
> 
> **Used in:** Redis Cluster, Cassandra, Memcached (ketama), CDN routing, Envoy's ring hash LB policy.

## Sources
- [[sre/concepts/load-balancers]]
- [[sre/concepts/networking-fundamentals]]
- [[devops/concepts/kubernetes-architecture]]
- [[devops/patterns/zero-downtime-deployment]]
