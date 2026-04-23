---
tags:
  - index
  - sre
  - reliability
  - interview-prep
  - infrastructure
---

# SRE Index
Last updated: 2026-04-22 (enhanced)

## Overview
- [[sre/overview]] — Coverage map and usage guide

## Topics
- [[sre/topics/linux-cli]] — Survey of all Linux CLI categories; command map; common question types
- [[sre/topics/system-design]] — SRE system design framework; 10 canonical designs; NALSD; distributed systems fundamentals; Apple/Google/Meta-specific patterns

## Concepts
- [[sre/concepts/file-manipulation]] — find, grep, awk, sed, sort/uniq/cut, xargs, permissions, FDs, pipes
- [[sre/concepts/process-signals]] — Process states, ps/top, signals (SIGTERM vs SIGKILL), lsof, strace, /proc
- [[sre/concepts/networking-troubleshooting]] — ping, dig, curl, ss/netstat, tcpdump, iptables; scenario playbooks
- [[sre/concepts/log-analysis]] — tail -f, journalctl, grep/awk pipelines, nginx log parsing, p99 from logs
- [[sre/concepts/disk-and-io]] — df, du, iostat, deleted-file trap, inode exhaustion, NFS hangs
- [[sre/concepts/linux-boot-process]] — BIOS/UEFI, GRUB2, Kernel, Init/Systemd, recovery modes
- [[sre/concepts/memory-management]] — Virtual memory, paging, RSS/VSZ, OOM Killer, COW
- [[sre/concepts/slo-sli-sla]] — SLI/SLO/SLA definitions, error budgets, availability nines table, composite SLO
- [[sre/concepts/networking-fundamentals]] — TCP handshake, TLS 1.2/1.3, encryption by layer, forward/reverse proxies, Zscaler/SASE zero trust
- [[sre/concepts/load-balancers]] — L4 vs L7 LB, NLB vs ALB, LB algorithms (round-robin, consistent hashing, least-conn), DB load balancing, K8s Service types + Ingress + kube-proxy internals

## Patterns
- [[sre/patterns/troubleshooting-framework]] — USE+RED model; 6 scenario playbooks (slow, full disk, 100% CPU, unreachable, OOM, EMFILE)

## Flashcards
- [[sre/flashcards/slo-sli-sla-scenarios]] — 10 scenario-based questions: error budget math, incident response, setting SLIs for batch vs API, alert fatigue, introducing SLOs to a resistant team

## Problems
- [[sre/problems/log-parsing-script]] — Medium; Streaming logs; O(N) time, O(U) space; Apple, Google
- [[sre/problems/distributed-rate-limiter]] — Hard; System Design; Redis/Token Bucket; Apple, Stripe
- [[sre/problems/parse-passwd-groups]] — Medium; /etc/passwd + /etc/group join; primary vs supplementary group membership
- [[sre/problems/error-rate-alerter]] — Medium; Sliding window deque; real-time error rate monitoring with hysteresis
- [[sre/problems/disk-space-hogs]] — Easy–Med; Min-heap top-N; symlink safety, permission handling
- [[sre/problems/ssl-cert-checker]] — Medium; Parallel I/O; TLS SNI; fleet-wide certificate expiry
- [[sre/problems/zombie-process-finder]] — Easy–Med; /proc parsing; process tree; wait() semantics
- [[sre/problems/fastest-dinosaur]] — Easy–Med; Hash-join two CSV files; filter + formula + argmax; Meta PE classic

## Scenarios
- [[sre/scenarios/high-cpu-troubleshooting]] — Linux internals; top, vmstat, strace; Apple, Google

## Companies
- [[sre/companies/apple]] — Heavy on Linux internals, networking, clean automation; privacy emphasis; Darwin/launchctl tooling
- [[sre/companies/google]] — SRE-SWE/PE split; SLO/error budget focus; toil reduction; Borg/Spanner/Monarch context
- [[sre/companies/meta]] — Production Engineer role; coding at SWE bar; C++ + Python; TAO/Scuba/ODS context
- [[sre/companies/amazon]] — SDE-Infra / Systems Dev Engineer; 16 Leadership Principles; AWS services depth; Dynamo paper

## Sources
- [[sre/sources/google-sre-book]] — Book; O'Reilly 2016; SLOs, toil, error budgets, four golden signals; blameless postmortems
- [[sre/sources/sre-workbook]] — Book; O'Reilly 2018; practical SLO rollout; multi-window burn rate alerting; sustainable on-call
- [[sre/sources/scaling-memcache-facebook]] — Paper; USENIX NSDI 2013; lease mechanism; McRouter; gutter pools; invalidation vs. update
- [[sre/sources/apple-interview-wiki]] — Community synthesis; Darwin/XNU focus; privacy engineering questions; scripting round patterns
- [[sre/sources/meta-pe-interview-guide]] — Community synthesis; "war room" scenarios; Fastest Dinosaur pattern; PE coding bar = SWE bar
