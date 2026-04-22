---
tags:
  - index
  - sre
  - reliability
  - interview-prep
  - infrastructure
---

# SRE Index
Last updated: 2026-04-22

## Overview
- [[sre/overview]] — Coverage map and usage guide

## Topics
- [[sre/topics/linux-cli]] — Survey of all Linux CLI categories; command map; common question types

## Concepts
- [[sre/concepts/file-manipulation]] — find, grep, awk, sed, sort/uniq/cut, xargs, permissions, FDs, pipes
- [[sre/concepts/process-signals]] — Process states, ps/top, signals (SIGTERM vs SIGKILL), lsof, strace, /proc
- [[sre/concepts/networking-troubleshooting]] — ping, dig, curl, ss/netstat, tcpdump, iptables; scenario playbooks
- [[sre/concepts/log-analysis]] — tail -f, journalctl, grep/awk pipelines, nginx log parsing, p99 from logs
- [[sre/concepts/disk-and-io]] — df, du, iostat, deleted-file trap, inode exhaustion, NFS hangs
- [[sre/concepts/slo-sli-sla]] — SLI/SLO/SLA definitions, error budgets, availability nines table, composite SLO
- [[sre/concepts/networking-fundamentals]] — TCP handshake, TLS 1.2/1.3, encryption by layer, forward/reverse proxies, Zscaler/SASE zero trust
- [[sre/concepts/load-balancers]] — L4 vs L7 LB, NLB vs ALB, LB algorithms (round-robin, consistent hashing, least-conn), DB load balancing, K8s Service types + Ingress + kube-proxy internals

## Patterns
- [[sre/patterns/troubleshooting-framework]] — USE+RED model; 6 scenario playbooks (slow, full disk, 100% CPU, unreachable, OOM, EMFILE)

## Flashcards
- [[sre/flashcards/slo-sli-sla-scenarios]] — 10 scenario-based questions: error budget math, incident response, setting SLIs for batch vs API, alert fatigue, introducing SLOs to a resistant team

## Problems
*(none yet)*

## Companies
*(none yet)*
