# Linux CLI for SRE/DevOps Interviews

**Related concepts:** [[sre/concepts/file-manipulation]], [[sre/concepts/process-signals]], [[sre/concepts/networking-troubleshooting]], [[sre/concepts/log-analysis]], [[sre/concepts/disk-and-io]]
**Related patterns:** [[sre/patterns/troubleshooting-framework]]

## Why this matters in interviews
SRE and DevOps interviews routinely include a live terminal or "what command would you run?" round. They test whether you can actually operate a production system under pressure — not just describe it. Google, Meta, Apple, and Amazon SRE loops all include a Linux/systems round.

## Command map by category

### File & text manipulation
| Command | One-liner purpose |
|---|---|
| `find` | Locate files by name, type, size, mtime, permissions |
| `grep` | Search file content by regex |
| `awk` | Field-based text processing; mini-language for structured text |
| `sed` | Stream editor; substitution, deletion, in-place edits |
| `sort` | Sort lines; `sort -k` for specific fields |
| `uniq` | Deduplicate adjacent lines; `-c` for counts |
| `cut` | Extract columns by delimiter |
| `wc` | Count lines (`-l`), words (`-w`), bytes (`-c`) |
| `xargs` | Feed find/grep output as arguments to another command |
| `tee` | Write to both stdout and a file simultaneously |

**Deep dive:** [[sre/concepts/file-manipulation]]

### Processes
| Command | Purpose |
|---|---|
| `ps aux` | Snapshot of all running processes |
| `top` / `htop` | Live process monitor; CPU, MEM, load average |
| `kill` / `pkill` | Send signals to processes |
| `lsof` | List open files, sockets, file descriptors |
| `strace` | Trace system calls (debugging + perf analysis) |
| `pstree` | Show parent-child process relationships |
| `nice` / `renice` | Set CPU scheduling priority |

**Deep dive:** [[sre/concepts/process-signals]]

### Networking
| Command | Purpose |
|---|---|
| `curl` / `wget` | HTTP requests; test endpoints |
| `dig` / `nslookup` | DNS resolution |
| `netstat` / `ss` | Active connections, listening ports |
| `tcpdump` | Capture and inspect network packets |
| `traceroute` / `mtr` | Trace packet path; diagnose latency hops |
| `iptables` | Firewall rules |
| `ping` | ICMP reachability |

**Deep dive:** [[sre/concepts/networking-troubleshooting]]

### Disk & I/O
| Command | Purpose |
|---|---|
| `df -h` | Disk space by filesystem |
| `du -sh *` | Disk usage by directory |
| `iostat` | Disk I/O statistics |
| `fdisk` / `lsblk` | Block device layout |
| `mount` / `umount` | Filesystem mounting |

**Deep dive:** [[sre/concepts/disk-and-io]]

### Logs
| Command | Purpose |
|---|---|
| `tail -f` | Follow a log file in real time |
| `journalctl` | Query systemd logs; `-u` for unit, `-f` to follow |
| `grep + awk` | Extract, filter, and aggregate from log files |
| `zcat` / `zgrep` | Read/search gzip-compressed log archives |

**Deep dive:** [[sre/concepts/log-analysis]]

## SRE-specific concepts
- [[sre/concepts/slo-sli-sla]] — Reliability targets, error budgets
- [[sre/patterns/troubleshooting-framework]] — Systematic approach: "the service is slow"

## Common interview question types
- "How do you find all files modified in the last 24 hours?" → `find`
- "Parse this log file and give me the top 10 IPs by request count" → `awk` + `sort` + `uniq`
- "A process is consuming 100% CPU — walk me through diagnosing it" → `top`, `ps`, `strace`, `lsof`
- "The disk is full. How do you find what's taking the space?" → `df`, `du`, `lsof` for deleted files
- "A service is unreachable. How do you troubleshoot?" → [[sre/patterns/troubleshooting-framework]]

## Sources
*(none yet)*
