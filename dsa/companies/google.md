# Google SRE Interview Prep (2026)

## Process
Google has two SRE tracks: **SRE-SWE** (Software focus) and **SRE-SE** (Systems focus).

- **Initial Screen** (45-60 min): 
  - SRE-SWE: Algorithm coding (Medium).
  - SRE-SE: Practical scripting / Linux systems.
- **On-site** (4-5 rounds):
  - **Coding (1-2 rounds):** SRE-specific practical coding or algorithms.
  - **Non-Abstract Large System Design (NALSD):** Focus on scaling, physics, and resource constraints (IOPS, bandwidth, latency).
  - **Systems/Troubleshooting:** Live debugging of a broken service.
  - **Googleyness & Leadership:** Behavioral.

## Emphasis
- **Reliability & Scalability:** Every solution must scale to millions of requests and handle machine failures.
- **Precision:** Google values "napkin math." Know your latency numbers (L1 cache vs. Disk vs. Cross-region network).
- **Automation:** "Toil" is the enemy. If a task is manual, automate it in your interview solution.
- **Error Handling:** Production code doesn't just work on the happy path. Handle timeouts, retries with backoff, and partial failures.

## Frequently Tested Topics (SRE Focus)
- **Data Streaming:** Processing logs/metrics that don't fit in memory (generators, iterators).
- **Concurrency:** Thread-safe counters, rate limiters, worker pools.
- **Graphs/Trees:** Dependency resolution (Topological Sort), resource discovery (BFS).
- **Strings/Regex:** Parsing malformed config files or complex logs.
- **Intervals:** Availability windows, monitoring thresholds.

## Role-Specific DSA Problems

| Problem | Difficulty | SRE Context |
|---|---|---|
| [[dsa/problems/design-hit-counter]] | Medium | Monitoring metrics (past 5 min RPS). |
| [[dsa/problems/course-schedule-ii]] | Medium | Service dependency resolution / Dag execution. |
| [[dsa/problems/lru-cache]] | Medium | In-memory cache for high-traffic endpoints. |
| [[dsa/problems/merge-intervals]] | Medium | Combining maintenance windows or alert ranges. |
| [[dsa/problems/task-scheduler]] | Medium | Job runner with execution constraints. |
| [[dsa/problems/daily-temperatures]] | Medium | Finding the next spike in a metrics stream (Monotonic Stack). |

## Practical Coding Scenarios
- **Log Aggregator:** Parse a massive log file and find the top 10 IP addresses with the most 5xx errors.
- **Configuration Rollout:** Write a script to push a config change to 10k servers. If error rate > 5%, stop and roll back.
- **Token Bucket:** Implement a rate limiter that allows bursts but maintains an average rate.
- **File System Crawler:** Find the largest files in a directory tree without exceeding a memory limit.

## NALSD (Non-Abstract Large System Design)
Unlike standard SWE System Design, NALSD is about **capacity planning**.
- **The Physics:** "How many SSDs do we need to store 50PB of video at 3x replication?"
- **The Bandwidth:** "Can we transfer this data over a 10Gbps link in 24 hours?"
- **The Failure:** "If a whole datacenter goes offline, where does the traffic go?"

## Behavioral (Googleyness)
- **Psychological Safety:** How do you handle a teammate's mistake?
- **Post-mortems:** "Tell me about a time you broke something in production and how you handled the post-mortem."
- **Ambiguity:** "Describe a time you started a project with no clear requirements."

## Tips
- **Python for SRE:** Use `collections.deque`, `heapq`, and `itertools`.
- **Go for SRE:** Understand channels and waitgroups for concurrency.
- **Memory vs. Time:** Always prioritize O(1) space over O(N) if the input can be billions of lines.
