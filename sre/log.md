# SRE Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" sre/log.md | tail -10`

---

## [2026-08-18] ingest | Escalation management, RCA basics, and expanded SLO/SLA examples

- User asked for dedicated interview notes on escalation management (including how to CLOSE an escalation specifically), RCA basics, and "at least a few" concrete SLO and SLA examples each. Audit found slo-sli-sla.md had definitions but only one one-line SLI example (no worked SLO/SLA examples), and neither escalation management nor RCA methodology had any dedicated page anywhere in the vault — only scattered passing mentions (a single escalation snippet inside a runbook template in obs/topics/alerting.md; a "Root Cause" template FIELD in devops/patterns/incident-response.md with no methodology for how to fill it).
- Created concept: escalation-management — escalation tiers (Tier 1/2/3/Executive), a concrete "when to escalate" checklist (time-boxed no-progress, growing blast radius, missing authority, guessing not diagnosing), the handoff structure of an effective escalation message (what/who/tried/need/status), and — the piece explicitly requested — a "how to close an escalation" exit checklist (verified resolution, explicit handback confirmation, stakeholder notification, filed follow-ups, formal severity downgrade, RCA scheduled).
- Created concept: rca-basics — root cause vs. contributing factor vs. trigger event (with a worked example), the 5 Whys technique (full 5-step worked example ending in a process-gap root cause, not a code patch), when 5 Whys breaks down (multi-causal incidents) and the Fishbone/Ishikawa alternative for that case, corrective vs. preventive action items, and 5 common RCA pitfalls (stopping at proximate cause, "human error" as a terminal answer, single-cause bias, confusing detectability with root cause, unowned action items).
- Expanded: slo-sli-sla.md — added "SLO examples" (4 worked examples across request-based/batch/latency-only/lag-based service shapes, each with a "why this shape" rationale) and "SLA examples" (4 worked examples including two non-uptime SLA shapes — support response time, security patch response — since interviewers sometimes test whether a candidate assumes SLA always means uptime).
- Updated: index.md (2 new concepts, updated slo-sli-sla summary), devops/patterns/incident-response.md (cross-linked from Related, the Root Cause template field, and Sources — this file keeps owning the overall incident lifecycle/postmortem template; the new sre/ pages specialize on the escalation sub-process and the RCA methodology that fills its Root Cause section, not duplicating either).
- Notes: the first hand-drawn Fishbone/Ishikawa diagram (diagonal branches converging on a central incident node) was abandoned in favor of a column-based 4-category-converging-to-one-incident layout, built and verified with a Python alignment script — true diagonal ASCII fishbone art can't be reliably column-verified the way the box-drawing diagrams from earlier sessions could, so a provably-aligned simplification was chosen over an authentic-looking but fragile one.

## [2026-08-18] update | HTTP-layer handshake (ALPN, HTTP/2 preface, QUIC) added to networking-fundamentals

- Extended networking-fundamentals.md's existing TCP/TLS handshake mechanics with the HTTP-layer handshake this file was missing: ALPN protocol negotiation (piggybacked on the TLS ClientHello/ServerHello, zero extra RTT), HTTP/2's connection preface + SETTINGS frame exchange (required after ALPN picks "h2," before any request/response frames are valid), and HTTP/3/QUIC's combined transport+TLS 1.3 handshake (1 RTT vs TCP+TLS's 2 RTT, plus 0-RTT resumption) contrasted directly against the sequential TCP-then-TLS flow already documented.
- Refreshed the "what happens when you type a URL" interview Q&A to include ALPN and the H2 preface step; added a new Q&A directly answering "is there really a separate HTTP handshake, or is it just TCP+TLS."
- Triggered by a user request for interview-ready HTTP handshake/status-code/header coverage — the parallel solution-arch/ additions (http-status-codes.md, http-headers.md) are logged in solution-arch/log.md; this entry covers only the sre/-owned protocol-mechanics piece, per this vault's existing sre-owns-mechanics/solution-arch-owns-design-decisions split.
- Notes: cross-linked forward to solution-arch/concepts/network-architecture-fundamentals.md for the "where do you actually deploy HTTP/3" architecture decision rather than duplicating it here.

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
