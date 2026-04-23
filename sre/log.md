# SRE Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" sre/log.md | tail -10`

---

## [2026-04-22] update | Dead-link audit + cross-KB coverage expansion
- Created: topics/system-design — comprehensive 10-question SRE design framework; NALSD; CAP; failure patterns; Apple/Google/Meta-specific design patterns
- Created: companies/amazon — SDE-Infra role; 16 Leadership Principles deep-dive; AWS services (S3/DynamoDB/Kinesis/Route53/ELB); Dynamo paper; COE/pre-mortem culture
- Created: problems/fastest-dinosaur — Meta PE classic; hash-join two CSV files on common key; filter+formula+argmax pattern; real-world PE equivalences
- Fixed: companies/apple.md — corrected dead topic link (mlops → linux-cli)
- Updated: index.md — added system-design topic, amazon company, fastest-dinosaur problem

## [2026-04-22] update | Google SRE + Meta PE prep + Apple enhancements + scripting problems
- Created: companies/google (full interview prep: process, SLO/error-budget, systems design, Borg/Spanner/Monarch context, behavioral)
- Created: companies/meta (Production Engineer role: coding at SWE bar, C++/Python, TAO/Scuba/ODS, on-call SEV levels, postmortem format)
- Updated: companies/apple — added incident response (P1–P4), privacy-by-design questions, Darwin/launchctl/DTrace tooling, behavioral prep, scripting problem table
- Created: problems/parse-passwd-groups (primary + supplementary group membership join; key gotcha)
- Created: problems/error-rate-alerter (sliding window deque; hysteresis state machine)
- Created: problems/disk-space-hogs (min-heap top-N; symlink safety; hard-link dedup follow-up)
- Created: problems/ssl-cert-checker (parallel I/O; TLS SNI; ThreadPoolExecutor)
- Created: problems/zombie-process-finder (/proc parsing; wait() semantics; cannot kill zombies insight)
- Updated: index.md — added all new company and problem pages
- Notes: Tricky problems focus on real sysadmin tasks that appear in Apple/Google/Meta SRE rounds, not abstract LeetCode.

## [2026-04-22] update | Apple SRE topic linking and missing concepts
- Created: concepts/linux-boot-process (BIOS/UEFI, GRUB, systemd)
- Created: concepts/memory-management (Virtual memory, paging, OOM, COW)
- Updated: companies/apple.md — converted all frequently tested topics into explicit wiki links to existing and new concept pages.
- Updated: index.md — added new concept pages to the content catalog.
- Notes: Ensured "out of the box" completeness by filling gaps in core systems knowledge required for Apple interviews.

## [2026-04-22] update | Apple-specific SRE interview content
- Created: companies/apple (process, emphasis, Linux internals, networking, coding)
- Created: problems/log-parsing-script (streaming O(N) log analysis)
- Created: problems/distributed-rate-limiter (system design with Redis/Lua)
- Created: scenarios/high-cpu-troubleshooting (USE model, Linux toolset)
- Updated: index.md — added new problems, scenarios, and company entry
- Notes: Emphasized Apple's focus on privacy, internal mechanics (how things work vs just using them), and scale.

## [2026-04-22] update | Networking fundamentals + load balancers
- Created: concepts/networking-fundamentals — TCP handshake (3-way + 4-way close), TLS 1.2/1.3 with handshake diagrams, session key derivation, encryption by OSI layer, forward/reverse proxies, CONNECT tunneling, API gateway vs reverse proxy, Zscaler ZIA/ZPA + SASE/zero trust architecture, port reference
- Created: concepts/load-balancers — L4 vs L7 deep-dive, NLB vs ALB comparison, all LB algorithms (round-robin, weighted, least-conn, least-response-time, IP hash, consistent hashing, power of two choices), health checks, DB layer LB (read/write split, PgBouncer, ProxySQL, sharding), K8s Service types (ClusterIP/NodePort/LB/Ingress), kube-proxy iptables/IPVS/eBPF internals, Gateway API YAML
- Updated: index.md

## [2026-04-21] query | Scenario-based SLO/SLI/SLA questions
- Filed as: flashcards/slo-sli-sla-scenarios
- 10 scenarios: error budget math, SLO pushback, incident response, composite SLO, alert fatigue/burn rate, batch pipeline SLIs, introducing SLOs to resistant teams

## [2026-04-21] update | SRE knowledge base initialized
- Created directory structure: sre/, topics/, concepts/, patterns/, problems/, flashcards/, companies/
- Created: sre/index.md, sre/log.md, sre/overview.md
- Created topics: linux-cli
- Created concepts: file-manipulation, process-signals, networking-troubleshooting, log-analysis, disk-and-io, slo-sli-sla
- Created patterns: troubleshooting-framework (USE+RED; 6 scenario playbooks)
- Domain: SRE/DevOps interview prep
