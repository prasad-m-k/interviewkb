# Apple SRE Interview Prep

**Topic:** [[sre/topics/mlops]] (Infrastructure focus)
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

## SRE Scenarios
- [[sre/scenarios/high-cpu-troubleshooting]]
- [[sre/problems/distributed-rate-limiter]]
- [[sre/problems/log-parsing-script]]

## Behavioral / Culture Fit
- **Ownership:** "I noticed X was broken, so I fixed Y to prevent it."
- **Calmness:** How do you react when a global service is down?
- **Privacy:** How would you design a logging system that doesn't leak PII?

## Sources
- [[sre/sources/apple-interview-wiki]] (Internal synthesis)
