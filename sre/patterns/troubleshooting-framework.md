# Troubleshooting Framework — "The Service Is Slow / Down"

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/process-signals]], [[sre/concepts/networking-troubleshooting]], [[sre/concepts/disk-and-io]], [[sre/concepts/log-analysis]]

## The framework: USE + RED

Two complementary models. Use both in interviews — it shows you think about resources *and* user experience.

**USE** (for resources — every physical/virtual resource):
- **U**tilization: % of time resource is busy
- **S**aturation: amount of work queued/waiting
- **E**rrors: error count

**RED** (for services — every microservice/API):
- **R**ate: requests per second
- **E**rrors: error rate
- **D**uration: latency distribution (p50, p95, p99)

---

## Scenario 1: "The service is slow"

```
1. Confirm the symptom
   └─ Is it all users or some? All endpoints or one?
      Is it slow now or intermittent?

2. RED check (service layer)
   └─ Rate: is traffic normal, spiked, or dropped?
      Errors: are there elevated 5xx/timeouts?
      Duration: which percentile is elevated — p50 (everyone slow) vs p99 (tail latency)?

3. USE check (resource layer)
   ├─ CPU: top, mpstat
   │    High user% → CPU-bound; profile the app
   │    High sys% → too many syscalls; strace
   │    High iowait% → disk bottleneck; iostat
   │
   ├─ Memory: free -h, /proc/meminfo
   │    Low free + high swap → swapping → severe performance degradation
   │    OOM killer active? → grep "Out of memory" /var/log/messages
   │
   ├─ Disk I/O: iostat -xz 1
   │    High %util / high await → disk saturated
   │
   └─ Network: ss -s, netstat -s | grep retransmit
        High retransmits → packet loss on the network path

4. Check dependencies
   └─ Is the database slow? (check DB query logs, slow query log)
      Is a downstream API timing out? (curl latency test)
      DNS resolution slow? (time dig hostname)

5. Check logs
   └─ grep ERROR/WARN around the time slowness started
      Correlate with deployments, cron jobs, traffic spikes
```

---

## Scenario 2: "The disk is full"

```bash
df -h                                          # which filesystem?
du -sh /* 2>/dev/null | sort -h               # which top-level dir?
du -sh /var/log/* | sort -h                   # drill down
find /var -type f -size +500M 2>/dev/null     # big files
lsof | grep deleted                            # deleted but held open
df -i                                          # inode exhaustion?
```

Fix options (in order of safety):
1. `> /var/log/bigfile.log` — truncate, not delete, to keep FD valid
2. Compress old logs: `gzip /var/log/old.log`
3. Restart the process holding deleted file descriptors
4. Delete truly unnecessary files

---

## Scenario 3: "A process is using 100% CPU"

```bash
top -b -n1 | head -20                  # identify PID
ps aux -p <PID>                        # confirm, get command
strace -p <PID> -c -e trace=all       # what syscalls, how frequent?
cat /proc/<PID>/status                 # threads, state
```

Causes:
- **Infinite loop / busy-wait**: `strace` shows rapid repeated syscalls; application bug
- **GC pressure (JVM)**: `jstack`, `jmap`; look for full GC every few seconds
- **CPU-bound computation**: profiling needed; check if expected

---

## Scenario 4: "Service is unreachable / connection refused"

```bash
# Is the process running?
ps aux | grep servicename

# Is it listening on the right port?
ss -tulnp | grep :8080

# Can I reach it locally?
curl localhost:8080/health

# Firewall blocking?
iptables -L -n | grep 8080

# DNS resolving correctly?
dig service.internal

# Can I reach it from another host?
curl http://10.0.1.5:8080/health

# Check logs for crash/startup errors
journalctl -u service -n 100 --no-pager
```

---

## Scenario 5: "Out of memory / OOM kills"

```bash
# Recent OOM kills
dmesg | grep -i "killed process"
grep -i "out of memory" /var/log/messages

# Current memory state
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree"

# Which process is the OOM killer likely to target?
# Killer scores processes by (memory_use * oom_score_adj)
cat /proc/<PID>/oom_score
cat /proc/<PID>/oom_score_adj

# Memory per process
ps aux --sort=-%mem | head -15

# Prevent a critical process from being OOM-killed
echo -1000 > /proc/<PID>/oom_score_adj
```

---

## Scenario 6: "Too many open files (EMFILE)"

```bash
ulimit -n                            # soft limit for shell
cat /proc/<PID>/limits               # limits for specific process
lsof -p <PID> | wc -l               # actual FD count
ls /proc/<PID>/fd | wc -l

# Raise limit for current session
ulimit -n 65536

# Permanent: /etc/security/limits.conf
# deploy soft nofile 65536
# deploy hard nofile 65536

# systemd services: in the unit file
# [Service]
# LimitNOFILE=65536
```

---

## General interview communication template

When asked a troubleshooting question in an interview, structure your answer:

1. **Clarify**: "Is this happening now? All users? All endpoints? Any recent changes?"
2. **Observe**: "I'd first look at metrics — latency, error rate, throughput."
3. **Hypothesize**: "The most likely causes are X, Y, Z. I'd check X first because..."
4. **Diagnose**: walk through specific commands
5. **Mitigate**: "While debugging I'd consider rolling back the last deploy / shedding load / increasing capacity."
6. **Fix and prevent**: "Root cause fix + monitoring to detect this earlier next time."

Interviewers want to see systematic thinking, not just the right command.

## Sources
*(none yet)*
