# Meta Production Engineer Interview Prep

**Related:** [[sre/patterns/troubleshooting-framework]], [[sre/concepts/slo-sli-sla]], [[sre/concepts/networking-fundamentals]]

## The Role

Meta's **Production Engineer (PE)** is their equivalent of Google SRE but with a heavier systems programming emphasis. PEs are embedded within product teams (Instagram, WhatsApp, Feed, Ads, etc.) and own the reliability and performance of those systems.

Key differences from a generic SRE role:
- PEs write production C++ and Python code, not just scripts.
- PEs are expected to optimize kernel parameters, debug memory corruption, and read assembly in extreme cases.
- PEs participate in on-call rotations with engineers and own the SLA for their service.
- At Meta, the PE hiring bar for coding is the same as Software Engineer E4/E5.

## Process

1. **Recruiter screen:** Background, motivation, and experience summary.
2. **Technical phone screen 1 (coding):** 1–2 LeetCode problems, Medium difficulty. Python or C++.
3. **Technical phone screen 2 (systems):** Troubleshooting scenario or networking/OS deep dive.
4. **Meta Virtual Onsite (4 rounds, each 45 min):**
   - **Round 1 – Coding:** 1–2 LeetCode Medium/Hard. Focus on correctness and efficiency.
   - **Round 2 – System Design:** Design a large-scale infra system (chat, notifications, CDN, cache).
   - **Round 3 – Production Engineering (systems):** Debug a production incident. Linux, networking, DB.
   - **Round 4 – Behavioral:** Values alignment using Meta's core values.

Meta uses **a loop-based debrief** where all interviewers discuss together after the onsite. Borderline cases get a committee review.

## Emphasis

Meta PEs are expected to be strong in all three pillars simultaneously:

| Pillar | Weight | What they look for |
|---|---|---|
| **Coding** | High | LeetCode Medium–Hard, clean idiomatic code, correct complexity |
| **Systems** | High | Linux internals, networking, debugging at FB scale |
| **Reliability** | Medium | SLOs, on-call practice, incident response, capacity |

Meta's values that appear in PE interviews:
- **Move fast:** Ship reliably, not recklessly. Fast != unsafe.
- **Focus on long-term impact:** Don't fix symptoms; eliminate classes of failure.
- **Build social value:** Systems must serve billions; corner cases matter.

## Frequently Tested Topics

### Production Debugging (Round 3 focus)
Meta PEs are expected to have strong incident diagnosis skills:

- **High memory usage:** `smem`, `/proc/<pid>/smaps`, `valgrind --leak-check`, `gperftools` heap profiler, huge page fragmentation.
- **High CPU:** `perf top`, `flamegraphs` (Brendan Gregg style), `strace` for syscall overhead, `taskset` for CPU pinning.
- **Network latency spikes:** `ss -i` (retransmit counters), `tcp_retrans_collapse`, MTU mismatch, NIC queue depths, `ethtool -S`.
- **Database performance:** Slow query log analysis, index utilization (`EXPLAIN`), connection pool saturation, lock contention (`SHOW ENGINE INNODB STATUS`).
- **Service unreachable:** DNS resolution chain, `iptables` / `nftables` rules, kernel connection backlog (`net.core.somaxconn`), SYN flood detection.

### Linux Internals
- **Process scheduling:** CFS (Completely Fair Scheduler), cgroups v2, `nice`/`renice`, real-time priorities.
- **Memory:** [[sre/concepts/memory-management]] — Huge pages, NUMA topology, `numactl`, transparent huge pages (THP) and when to disable them (latency-sensitive services).
- **I/O:** [[sre/concepts/disk-and-io]] — `blktrace`, `fio` for benchmarking, `io_uring` (new async I/O), NVMe vs HDD scheduler differences.
- **Networking:** [[sre/concepts/networking-fundamentals]] — TCP BBR, UDP multicast (used in internal meta video delivery), DPDK for high-performance networking.

### System Design at Meta Scale
Meta runs some of the world's largest distributed systems:

- **TAO:** Facebook's distributed graph database. Key insight: optimized for social graph reads (fan-out of Like/Comment). Eventual consistency with local caches.
- **Memcache at Facebook:** Scaling Memcache (McRouter, regional pools, lease-based thundering herd prevention).
- **Scuba:** Facebook's fast, in-memory time-series and log analytics system. Used for debugging (not monitoring). Columns are optional — schema-on-write.
- **ODS (Operational Data Store):** Meta's time-series metrics store. Pull-based collection, Gorilla-style compression.
- **Tupperware:** Meta's container orchestration (like Kubernetes but predates it). Knows hardware topology.

Common design questions:
- Design Facebook Messenger at scale (storage, fanout, presence)
- Design a CDN for video delivery with 3 billion users
- Design a notification service (push, pull, fan-out strategies)
- Design a rate limiter for the Graph API

### Coding (LeetCode focus)
Meta coding is stricter than Google's on constant factor efficiency. They use a shared code editor (CoderPad).

Frequently tested patterns:
- **BFS/DFS:** Social graph traversal, friend-of-friend, island problems.
- **Trees:** LCA, serialize/deserialize, diameter.
- **Sliding window / two pointers:** Moving averages, max sliding window.
- **Union-Find:** Connected components in social graph.
- **Heap:** Top-K friends by activity, merge K sorted streams.

High-frequency problems:
- Merge K Sorted Lists
- Number of Islands / Friend Circles (Union-Find)
- LRU Cache (linked hashmap — implement from scratch)
- Median of Data Stream
- Binary Tree Maximum Path Sum

### Scripting / Automation
PEs write real production scripts:
- [[sre/problems/parse-passwd-groups]] — User/group enumeration
- [[sre/problems/error-rate-alerter]] — Sliding window anomaly detection
- [[sre/problems/ssl-cert-checker]] — Fleet-wide certificate expiry
- [[sre/problems/disk-space-hogs]] — Disk usage analysis

## Meta-Specific On-Call Practices

### Severity Levels
| SEV | Description | Response |
|---|---|---|
| SEV1 | Complete outage, revenue impact | All hands, 15-min bridge open |
| SEV2 | Significant degradation, partial outage | On-call + secondary, 30-min |
| SEV3 | Minor degradation, degraded feature | On-call, business hours response |
| SEV4 | Potential issue, no user impact | Ticket, 1-week SLA |

### Postmortem Culture
Meta does **blameless postmortems** with:
1. Timeline of events (what was observed and when)
2. Root cause analysis (5 Whys)
3. Action items with DRI (Directly Responsible Individual) and deadline
4. "What went well" — acknowledge working processes

### On-Call Handoff
Meta uses a weekly on-call rotation. Key expectations:
- Maintain a runbook for every alert.
- Alert P90 response: SEV2 within 15 minutes, SEV3 within 2 hours.
- After an incident, document within 48 hours.

## Behavioral / Culture Fit

Meta uses the **S.T.A.R.** format and maps answers to core values.

Prep questions:
- "Tell me about a time you improved the reliability of a system. What was the before/after?"
- "Describe a production incident you were on-call for. Walk me through your response."
- "How do you balance shipping new features vs. paying down reliability debt?"
- "Tell me about a time you disagreed with your team on a technical approach. What happened?"
- "Give an example of a high-impact change you made with little guidance."

**Key phrases that score well:**
- "I owned the on-call rotation and drove the action items to completion."
- "I measured the impact by comparing p99 latency before and after."
- "I wrote a runbook so the next engineer could resolve this in under 10 minutes."

## Tips from Sources

- Meta PE coding round is exactly the same bar as SWE. Don't underprepare on LeetCode.
- In the system design round, explicitly ask about scale (DAU, QPS, storage) before diving in — interviewers expect this.
- For the PE-specific round (Round 3), demonstrate that you know tools (strace, perf, tcpdump) AND their output. "I'd run strace" is not enough; explain what you're looking for.
- Meta's design questions often have a follow-up: "How does your design handle a 10× traffic spike?" Always answer with autoscaling + load shedding + circuit breakers.
- Bring up **data locality and NUMA** when discussing performance — Meta operates at a scale where these matter and interviewers notice when you do.
- Use Facebook-specific terminology when appropriate: TAO for graph reads, Thrift for RPC, Scuba for fast log queries.

## Sources

- [[sre/sources/meta-pe-interview-guide]]
- [[sre/sources/scaling-memcache-facebook]] (USENIX NSDI 2013)
