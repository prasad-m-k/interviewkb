# Overview — SRE / DevOps Interview Prep

## Coverage map

| Domain | Status | Key gaps |
|---|---|---|
| Linux CLI & file manipulation | done | shell scripting patterns |
| Process management & signals | done | cgroups, namespaces (container internals) |
| Networking troubleshooting | done | load balancer internals, eBPF |
| Log analysis | done | centralized logging (ELK, Loki) |
| Disk & I/O | done | RAID, filesystem internals |
| SLO / SLI / SLA | done | error budget burn rate alerts |
| Troubleshooting framework | done | — |
| Shell scripting | not started | bash patterns, cron |
| Containers / Kubernetes | not started | must-have for FAANG SRE |
| CI/CD | not started | pipelines, deployment strategies |

## How to use
- Linux/systems round: drill [[sre/concepts/file-manipulation]] and [[sre/patterns/troubleshooting-framework]]
- Design round: [[sre/concepts/slo-sli-sla]] + reliability patterns
- Hands-on: practice every `awk`/`sed`/`find` one-liner in a terminal
