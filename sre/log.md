# SRE Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" sre/log.md | tail -10`

---

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
