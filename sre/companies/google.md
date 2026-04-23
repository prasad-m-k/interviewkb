# Google SRE Interview Prep

**Related:** [[dsa/companies/google]], [[sre/patterns/troubleshooting-framework]], [[sre/concepts/slo-sli-sla]]

## Process

Google's SRE hiring splits into two tracks:

| Track | Focus split | Who it's for |
|---|---|---|
| **SRE-SWE** (software-heavy) | ~60% coding / 40% systems | Strong coding + ops background |
| **SRE-PE** (production engineering) | ~40% coding / 60% systems | Infra/ops-dominant background |

Typical loop (5–6 rounds, all 45 min):
1. **Recruiter screen:** Background, experience, career goals.
2. **Phone screen 1 (coding):** 1–2 LeetCode-style problems. Expect Medium–Hard.
3. **Phone screen 2 (systems):** Troubleshooting scenario or system design at Google scale.
4. **Onsite Round A – Coding:** Pure algorithms. Google uses strict time/space analysis.
5. **Onsite Round B – Systems Design:** Design a globally distributed system (rate limiter, pub/sub, time-series DB).
6. **Onsite Round C – SRE Practices:** SLOs, error budgets, postmortems, toil reduction.
7. **Onsite Round D – Troubleshooting:** Live Linux debugging scenario. May involve reading `strace` output.
8. **Onsite Round E – Behavioral (Googliness):** Values alignment, cross-functional influence.

Google uses **unified hiring committees** — all interviewers submit independent scorecards and a committee (not the hiring manager) makes the final call. A single weak signal rarely kills an offer, but two does.

## Emphasis

Google wrote the book on SRE (literally). Every answer should reflect SRE-book principles:

- **SLOs over heroics:** Quantify reliability. If you can't state an error budget, you don't have a reliability story.
- **Toil reduction:** Automate operational work. Mention the toil percentage (Google targets <50% toil per SRE).
- **Error budgets as policy:** When the error budget is exhausted, new features stop until reliability improves.
- **Blameless postmortems:** Incidents are learning opportunities, not blame sessions.
- **Scaling through software, not headcount:** Every manual intervention is a bug in the automation.

## Frequently Tested Topics

### Reliability Engineering
- **SLO/SLI/SLA:** [[sre/concepts/slo-sli-sla]] — Google defines SLIs narrowly (request latency at p99, availability, durability). Know how to compute error budgets, burn rates, and multi-window alerting.
- **Error Budget Policy:** When budget is 10% consumed in 1 hour (fast burn), page immediately. When 5% consumed in 6 hours (slow burn), page during business hours.
- **Toil:** Distinguish toil (manual, repetitive, automatable) from overhead (necessary but non-toil) and engineering work (creates lasting value).
- **Postmortem format:** Timeline → Impact → Root cause → Action items (with owners and due dates). Distinguish root cause from contributing factors.

### Systems Design at Google Scale
- **Globally distributed systems:** Borg/Kubernetes, Chubby (distributed lock service), Spanner (globally consistent SQL), Bigtable, Pub/Sub.
- **Consistency models:** Know the trade-offs — Spanner achieves external consistency via TrueTime. Bigtable is eventually consistent per-row. Know when to choose each.
- **Rate limiting at scale:** Token bucket in Redis with Lua scripting vs. sliding window log vs. fixed window counter. Know why Redis Lua is atomic. [[sre/problems/distributed-rate-limiter]]
- **Distributed tracing:** Dapper (Google's internal) → OpenTelemetry externally. Trace ID propagation, sampling strategies.

### Linux & Systems
- **Deep troubleshooting:** [[sre/scenarios/high-cpu-troubleshooting]] — Google interviewers often give you `strace` or `perf` output and ask you to diagnose.
- **Networking:** [[sre/concepts/networking-fundamentals]] — Know TCP flow control, BBR congestion control (Google's algorithm), and QUIC/HTTP3.
- **Memory:** [[sre/concepts/memory-management]] — OOM Killer, transparent huge pages (THP), `cgroups` v2 memory limits.
- **I/O:** [[sre/concepts/disk-and-io]] — `blkio` cgroup, `ionice`, I/O schedulers (CFQ vs. deadline vs. noop).

### Coding (Python or Go preferred)
- **Graph traversal:** BFS/DFS for dependency resolution (Borg task graph, build systems).
- **Streaming problems:** [[sre/problems/log-parsing-script]] — processing logs that don't fit in memory.
- **Heap / Top-K:** Finding top-K offenders (IPs, endpoints, services) using `heapq`.
- **System scripting:** [[sre/problems/parse-passwd-groups]], [[sre/problems/error-rate-alerter]] — real-world admin scripts.

## Google SRE-Specific Concepts

### Multi-Window Alerting (the "burn rate" model)
Google's recommended alerting strategy from the SRE Workbook:

```
Fast burn:  2% budget consumed in 1 hour  → page immediately
Slow burn:  5% budget consumed in 6 hours → page during biz hours
Ticket:     10% budget in 3 days          → create ticket
None:       within budget                 → no alert
```

The formula: `burn_rate = error_rate / (1 - SLO_target)`

### Capacity Planning
- Know the difference between **provisioning** (static allocation) and **autoscaling** (dynamic).
- Google targets **N+2 redundancy** for critical services.
- Know **"headroom"**: unused capacity reserved for burst. Typically 30–40% at Google.
- Load shedding strategies: LIFO, priority queues, circuit breakers.

### Production Readiness Reviews (PRR)
A PRR is Google's gate before a service takes production traffic. Ask:
- Does the service have SLOs?
- Is there a documented incident response runbook?
- Is monitoring and alerting in place?
- Is the rollout plan reviewed?

## Frequently Seen Problems

| Problem | Type | What Google tests |
|---|---|---|
| Design a globally consistent rate limiter | System Design | Distributed consensus, Redis/Lua, Spanner |
| Design Google Pub/Sub | System Design | Fan-out, at-least-once delivery, dead letter queues |
| Find top K errors in a 1TB log | Coding | Streaming, heap, external sort |
| Given `strace` output, diagnose slow app | Troubleshooting | Syscall literacy, I/O vs CPU vs network |
| Design a monitoring system for 1M services | System Design | Time-series DB, pull vs push metrics, Monarch |
| Parse and aggregate distributed logs | Coding | [[sre/problems/log-parsing-script]] |
| Calculate error budget from SLI data | SRE Practices | Math + policy judgment |

## Behavioral / Culture Fit (Googliness)

Google interviewers use structured behavioral questions mapped to four attributes:
1. **General cognitive ability:** How do you learn new things? Break down a novel technical problem.
2. **Leadership:** Not management — influence without authority. When did you lead without a title?
3. **Googleyness:** Comfort with ambiguity, intellectual humility, collaborative, caring about the user.
4. **Role-related knowledge:** Deep technical knowledge specific to SRE.

Prep questions:
- "Tell me about a system you designed that failed at scale. What did you change?"
- "How do you push back on a product team that wants to disable an SLO to ship faster?"
- "Describe a time you reduced toil. How did you measure the impact?"
- "Tell me about a postmortem you led. What surprised you in the root cause?"

## Tips from Sources

- Google SRE rounds are strict on time/space complexity — state it *before* you write code, not after.
- For system design, mention **SLOs for the system you're designing**, not just architecture. Shows SRE mindset.
- If you get a troubleshooting question with output, read it aloud and narrate your reasoning before jumping to conclusions.
- Google favors **Go** for new SRE tooling; **Python** is acceptable for scripting. C++ for performance-critical paths.
- The behavioral round scores "Googliness" — interviewers explicitly look for intellectual humility. Saying "I don't know, but here's how I'd find out" scores higher than a wrong confident answer.
- Know the difference between **SLO** (internal reliability target) and **SLA** (external contractual commitment). Never promise an SLA in a design without explaining the SLO backing it.

## Sources

- [[sre/sources/google-sre-book]] (Site Reliability Engineering, O'Reilly)
- [[sre/sources/sre-workbook]] (The Site Reliability Workbook, O'Reilly)
