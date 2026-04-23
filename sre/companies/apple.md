# Apple SRE Interview Prep

**Topic:** [[sre/topics/linux-cli]] (Infrastructure focus)
**Related:** [[dsa/companies/apple]], [[sre/patterns/troubleshooting-framework]]

## Process
The Apple SRE process is team-specific (iCloud, Apple Music, Infrastructure) but generally follows this structure:

1. **Recruiter Screen:** Experience, background, and culture fit.
2. **Technical Phone Screen 1 (Linux/Troubleshooting):** Live troubleshooting in a terminal or deep dive into OS internals.
3. **Technical Phone Screen 2 (Networking):** TCP/IP, DNS, Load Balancing, and Security.
4. **Onsite (Virtual):** 4–5 rounds including:
   - **Systems Internals:** OS, memory, I/O.
   - **Coding/Automation:** Scripting (Python/Bash) and DSA (LeetCode Medium).
   - **System Design:** Distributed systems for reliability.
   - **SRE Practices:** SLOs, monitoring, incident response.
   - **Behavioral:** Privacy, ownership, and "calm under pressure."

## Emphasis
- **Internal Knowledge:** Don't just know *how* to use a tool; know *why* it works (e.g., how `epoll` works under the hood).
- **Privacy & Security:** Apple's core differentiator. Design systems with "privacy by default."
- **Scale:** Apple services (iCloud, App Store) handle hundreds of millions of users. Answers must scale.
- **Clean Code:** Clean, production-ready code is expected even in whiteboard rounds.

## Frequently Tested Topics

### Linux & Systems
- **Boot Process:** [[sre/concepts/linux-boot-process]] (BIOS/UEFI → GRUB → Kernel → Init/Systemd).
- **Apple Internals (XNU/Darwin):** [[sre/concepts/apple-xnu-internals]] — Mach microkernel, BSD layer, vnodes, and `launchd`.
- **Process Management:** [[sre/concepts/process-signals]] (`fork()`, `exec()`, zombie processes, signals, and COW).
- **Memory:** [[sre/concepts/memory-management]] (Virtual memory, page faults, OOM Killer).
- **I/O:** [[sre/concepts/disk-and-io]] (Hard links vs. soft links, file descriptors, `lsof`, `strace`).

### Networking
- **TCP/IP & Protocols:** [[sre/concepts/networking-fundamentals]] (Three-way handshake, window scaling, DNS, HTTP/2 vs. HTTP/3, TLS 1.3).
- **Load Balancing:** [[sre/concepts/load-balancers]] (L4 vs. L7, consistent hashing, global traffic management/GTM).
- **Troubleshooting:** [[sre/concepts/networking-troubleshooting]] (`dig`, `tcpdump`, `ss`, `iptables`).

### Coding & Scripting
- **Log Parsing:** Parsing TBs of logs to find top errors (O(N) time, O(K) space).
- **DSA:**
  - [[dsa/problems/lru-cache]] (Very frequent)
  - [[dsa/problems/top-k-frequent-elements]]
  - [[dsa/problems/merge-intervals]]
  - [[dsa/problems/trapping-rain-water]] (Hard variant)

## SRE Scenarios and Problems

### Troubleshooting Scenarios
- [[sre/scenarios/high-cpu-troubleshooting]] — USE model; `top`, `vmstat`, `strace`, `perf`
- [[sre/problems/distributed-rate-limiter]] — Redis + Lua; token bucket at Apple scale

### Scripting / Coding Problems
Apple coding rounds often feature real sysadmin problems rather than abstract LeetCode:

| Problem | What it tests |
|---|---|
| [[sre/problems/log-parsing-script]] | Streaming large files; O(N) time, O(U) space |
| [[sre/problems/parse-passwd-groups]] | File parsing; joining two data sources; /proc literacy |
| [[sre/problems/disk-space-hogs]] | Heap / top-K; permission handling; symlink safety |
| [[sre/problems/error-rate-alerter]] | Sliding window; deque; streaming state machine |
| [[sre/problems/ssl-cert-checker]] | Parallel I/O; TLS; fleet-wide automation |
| [[sre/problems/zombie-process-finder]] | /proc parsing; process tree; wait() semantics |

### Apple-Specific Tooling Deep Dives
Apple interviewers expect comfort with Darwin/macOS-specific tooling in addition to standard Linux:

- **`launchctl`:** List, start, stop, and inspect `launchd` daemons (`launchctl list`, `launchctl bootout`, `launchctl kickstart`). Know the difference between user vs. system domains.
- **`pmset`:** Power management settings; relevant for macOS infrastructure (Mac mini server farms). Know `pmset -g assertions` to diagnose wake locks.
- **`DTrace`:** Apple's kernel instrumentation framework (available on macOS). Know basic D scripts for syscall tracing. Equivalent to Linux `perf` + `eBPF`.
- **`Instruments`:** GUI profiling tool (Time Profiler, Allocations, Network). Know how to read a flame graph from Instruments output.
- **`notarization` and `Gatekeeper`:** Understand code signing, notarization requirements, and how to diagnose Gatekeeper rejections (`spctl --assess`, `codesign -vvv`). Relevant for deployment automation at Apple.
- **`Activity Monitor` CLI equivalents:** `sample <pid>`, `heap <pid>`, `leaks <pid>` — Apple's lightweight profiling tools.

## Incident Response and On-Call

Apple's on-call culture for SRE/infrastructure teams:

### Severity Levels
| SEV | Scope | SLA |
|---|---|---|
| P1 | Global service outage (iCloud, App Store down) | Immediate, all-hands bridge |
| P2 | Regional outage or major feature degraded | On-call + escalation within 15 min |
| P3 | Single feature degraded, no data loss | On-call response within 1 hour |
| P4 | Minor issue, fully workaround-able | Business hours ticket |

### Postmortem Expectations
Apple expects blameless postmortems with:
1. **Timeline** — chronological events with timestamps.
2. **Customer impact** — number of users affected and duration.
3. **Root cause** — technical root cause, not just the symptom.
4. **Five Whys** — drill past the surface cause to the systemic issue.
5. **Action items** — each with an owner and a due date.

### Interview Simulation Tip
Apple often asks you to walk through a real incident you were involved in. Structure it as:
> "At [timestamp], I was paged for [symptom]. I started with [first tool], which showed [finding]. I then [action], which confirmed [root cause]. The fix was [resolution]. Post-incident, we [prevention]."

## Privacy-by-Design Questions

Apple's SRE interviews include privacy-specific design questions. Common patterns:

- "Design a logging system for iCloud that stores enough data to debug issues but doesn't leak user content."
  - **Answer framework:** Log metadata (request IDs, error codes, latency) but never log content (file names, message bodies). Use differential privacy for aggregate analytics. Store logs with user-specific encryption keys so access requires user consent.

- "How would you design a crash reporting system that protects user privacy?"
  - **Answer framework:** Symbolicate on-device before upload, strip personally identifiable frames, aggregate crash signatures server-side, never include full stack traces with user data.

- "A debugging runbook requires reading production database records. How do you structure access?"
  - **Answer framework:** Role-based access with audit logging, time-bounded access tokens (not persistent credentials), dual-approval for high-sensitivity data, and synthetic/anonymized test environments for routine debugging.

## Behavioral / Culture Fit

Apple culture is less codified than Amazon's Leadership Principles but has consistent themes:

**Core themes:**
- **Ownership and craftsmanship:** "I cared about getting this right, not just shipping."
- **Calmness under pressure:** Interviewers assess whether you panic or methodically diagnose.
- **Privacy awareness:** Apple's core brand differentiator. Mention privacy implications proactively.
- **Cross-functional collaboration:** SREs work with hardware, firmware, CoreOS, and product teams.

**Common behavioral questions:**
- "Describe an outage you were on-call for. How did you diagnose it?"
- "Tell me about a time you disagreed with a technical decision and how you handled it."
- "How do you handle code review feedback you disagree with?"
- "Describe a project where you had to balance speed and correctness."
- "Tell me about a system you designed that had privacy implications. How did you address them?"

## Sources
- [[sre/sources/apple-interview-wiki]] (Internal synthesis)
