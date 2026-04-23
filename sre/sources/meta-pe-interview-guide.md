# Meta Production Engineer Interview Guide

**Type:** Internal synthesis / Community knowledge aggregation
**Ingested:** 2026-04-22
**Topics covered:** [[sre/topics/linux-cli]], [[sre/topics/system-design]]

## Summary

Aggregation of Meta Production Engineer interview experiences from engineering forums, Blind, Glassdoor, LinkedIn, and the official Meta careers engineering blog (2021–2026). The Meta PE role is uniquely documented because Meta has published some guidance on what the role entails, and alumni frequently share experiences given the volume of PE hiring.

## Key Takeaways

- **The coding bar is identical to SWE.** Many candidates underestimate the Meta PE coding round. Two LeetCode Mediums in 45 minutes is the standard, and Hard problems appear for senior roles. Treat this like an SWE interview, not an ops interview.
- **The systems round uses a broken environment.** You are given a VM or simulated environment with actual problems to diagnose. This is not a "what would you do" question — you type commands and the interviewer watches. Know your tools cold (strace, tcpdump, ss, top, lsof).
- **The "Fastest Dinosaur" pattern appears in multiple variations.** See [[sre/problems/fastest-dinosaur]]. Any problem requiring "read two files, join on a key, filter, compute a derived field" is this pattern. Practice it fluently.
- **Meta expects explicit complexity analysis.** Unlike Apple (which appreciates it), Meta actively prompts: "What's the time and space complexity of your solution?" Have it ready before writing code.
- **Move Fast = ship reliably, not recklessly.** Behavioral stories should show speed + quality, not just speed. "I shipped it quickly" without "and I had tests/monitors/rollback" will not resonate.

## Recurring Interview Scenarios (Aggregated from Reports)

### Coding (Round 1)
- Two-sum variants, sliding window, BFS/DFS
- Merge K Sorted Lists (heap — very common)
- LRU Cache (must implement from scratch, not use OrderedDict)
- Number of Islands / Friend Circles
- Binary tree problems (LCA, max path sum)

### Systems Scripting (Round 2)
- Fastest Dinosaur (join two files, compute speed formula, find max)
- Metric Aggregator (parse JSON metrics files per region, compute averages)
- Stale Process Reaper (find processes running >24h belonging to a user, kill them)
- URL Health Checker (check N URLs in parallel, report which are down)
- Log error rate monitor (sliding window alert)

### System Design (Round 3)
- Design Facebook's notification service (at 3B user scale)
- Design a global rate limiter for the Graph API
- Design a real-time metrics pipeline (ingestion → storage → query)
- Design Facebook Messenger (storage, delivery, presence)
- Design a content delivery network for video (Reels scale)

### Production Engineering Round (Round 4 — "War Room")
The PE-specific round places you in a simulated broken environment. Reported scenarios:
1. **Full disk:** Disk at 100%, service writing logs failing. Root cause: deleted log files held open by a process (deleted-file-still-has-FD trap). Fix: restart the process.
2. **DNS misconfiguration:** Service can't reach upstream. `curl` times out. `dig` shows NXDOMAIN. Root cause: `/etc/resolv.conf` was overwritten by a misconfigured DHCP lease. Fix: restore nameserver entries.
3. **OOM loop:** Process keeps crashing. `/var/log/syslog` shows OOM Killer selecting it. Fix: lower memory footprint (smaller Java heap, etc.) or add more memory; short-term: `cgroups` memory limit tuning.
4. **Firewall block:** Application server can't reach database. `ss` shows no established connections. `telnet db-host 5432` hangs. `iptables -L` shows a DROP rule. Fix: remove the offending iptables rule, document why.
5. **NTP drift:** Services logging timestamps 5 minutes off, causing JWT token validation failures. `timedatectl` shows NTP not synced. Fix: restart `chronyd`/`ntpd`, verify peers.

## What it Updated

- Created [[sre/companies/meta]] — process, emphasis, PE-specific content, systems tooling, on-call practices
- Created [[sre/problems/fastest-dinosaur]] — Meta's canonical PE scripting interview problem
- Informed [[sre/problems/parse-passwd-groups]] — appears in Meta systems scripting round
- Informed [[sre/problems/error-rate-alerter]] — appears in Meta systems scripting round
