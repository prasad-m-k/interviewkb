# Apple SRE Interview — Internal Synthesis

**Type:** Internal synthesis / Community knowledge aggregation
**Ingested:** 2026-04-22
**Topics covered:** [[sre/topics/linux-cli]], [[sre/topics/system-design]], [[sre/concepts/apple-xnu-internals]]

## Summary

This source represents an aggregation of Apple SRE interview experiences from engineering forums, Blind, Glassdoor, LinkedIn posts, and firsthand accounts shared in the SRE community between 2020–2026. Unlike published books or papers, this is empirically derived knowledge — what candidates actually reported being asked.

Apple does not publicly describe its interview process in detail. The information here is inferred from patterns across many individual reports.

## Key Takeaways

- **"How it works, not just how to use it"** is the defining Apple SRE expectation. Knowing that `epoll` exists isn't enough — you need to explain that `epoll` is a Linux kernel I/O event notification facility that uses a red-black tree to store monitored file descriptors and a linked list to return ready events, giving O(1) notification complexity per event regardless of the number of monitored FDs.
- **Darwin/macOS internals are tested alongside Linux.** Apple interviewers assume you know standard Linux but distinguish themselves by probing XNU/Darwin. Know `launchctl`, `launchd`, `DTrace`, and how the Mach microkernel differs from a monolithic kernel.
- **Privacy is not just an ethics question — it's an engineering constraint.** "How would you design a logging system that doesn't leak PII?" expects an architectural answer (metadata-only logs, on-device processing, differential privacy for aggregates, per-user encryption keys), not a policy answer.
- **iCloud scale means Apple's reliability bar is very high.** iCloud has hundreds of millions of active users. System design answers must handle that scale without hand-waving.
- **The behavioral bar emphasizes calmness.** Apple's culture is less "Move Fast" and more "Get it right." Behavioral stories should emphasize careful engineering, deliberate testing, and measured rollout — not speed.

## Recurring Interview Scenarios (Aggregated from Reports)

### Linux/Systems Questions Reported Most Frequently
1. "Walk me through the Linux boot process from power button to login prompt."
2. "Explain the difference between load average and CPU utilization."
3. "What happens when you run `ssh remote-host` from the command line?" (Covers DNS, TCP, SSH key exchange, PTY allocation)
4. "How does `epoll` work under the hood?"
5. "Explain virtual memory, page tables, and the TLB."
6. "What is copy-on-write (COW) and how does fork() use it?"
7. "What is an inode? What happens when you delete a file?"

### Networking Questions Reported Most Frequently
1. "Walk me through a full TCP connection lifecycle (3-way handshake + 4-way close)."
2. "How does DNS work? What's the difference between recursive and iterative resolution?"
3. "Explain TLS 1.3 handshake. What changed from TLS 1.2?"
4. "You can't reach a server. Walk me through your troubleshooting."
5. "What is QUIC? Why did HTTP/3 switch from TCP to QUIC?"

### Scripting/Coding
1. "Parse a large log file and find the top 10 IPs with 5xx errors." (Streaming, O(N) time)
2. "Write a script to monitor a service and restart it if it crashes." (Process management, signals)
3. "Parse `/etc/passwd` and `/etc/group` to list all users and their groups."
4. "Given a directory, find the 5 largest files by size."
5. "Write a rate limiter that allows N requests per second per user."

### System Design
1. "Design a logging system for iCloud that handles 100 million users and doesn't leak PII."
2. "Design a distributed rate limiter for Apple's App Store API."
3. "Design an alerting system for an Apple infrastructure service."
4. "How would you design a zero-downtime deployment pipeline for an iCloud service?"

## What it Updated

- Created [[sre/companies/apple]] — process, emphasis, scripting problems, Darwin tooling, privacy design questions, incident response
- Informed [[sre/problems/parse-passwd-groups]] — sourced from actual Apple interview reports
- Informed [[sre/problems/log-parsing-script]] — confirmed as one of the top Apple scripting questions
