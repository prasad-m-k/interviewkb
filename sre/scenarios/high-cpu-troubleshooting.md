# Troubleshooting: High CPU

**Topic:** [[sre/topics/linux-cli]]
**Pattern:** [[sre/patterns/troubleshooting-framework]] (USE Model)
**Companies:** [[sre/companies/apple]], [[sre/companies/google]], [[sre/companies/meta]]

## Scenario
A user reports that the web application is extremely sluggish. You log into the server and notice the load average is significantly higher than the number of CPU cores.

## Step 1: Identification (The "U" in USE)
Check **Utilization**:
- `top` or `htop`: Identify which processes are consuming CPU.
- `uptime`: Check load averages (1, 5, 15 minutes). If load avg > core count, CPU is saturated.
- `vmstat 1`: Check `us` (user), `sy` (system), `id` (idle), and `wa` (iowait).

## Step 2: Isolation
- **High `us` (User):** An application bug (infinite loop, heavy computation).
- **High `sy` (System):** Kernel overhead (context switching, interrupt handling).
- **High `wa` (IO Wait):** CPU is waiting for disk/network. The problem is likely I/O, not the CPU itself.

## Step 3: Deep Dive
If a specific process (e.g., PID 1234) is the culprit:
- `strace -p 1234`: See what system calls it's making. Are there thousands of failed `open()` calls?
- `perf top -p 1234`: See which function inside the process is consuming the most cycles.
- `lsof -p 1234`: Check open files/sockets. Is it leaking file descriptors?

## Step 4: Resolution
- **Short term:** `kill -15` (SIGTERM) or `renice` the process.
- **Long term:** Fix the code bug, add more CPU cores, or implement rate limiting.

## Apple-Specific Follow-ups
- "What if `top` shows high CPU but no process accounts for it?" (Answer: Short-lived processes, use `execsnoop` from BCC/ebpf).
- "How do you check CPU steal time in a virtualized environment?" (Answer: Check `st` in `top`).
- "Explain the difference between Load Average and CPU Utilization." (Answer: Load includes processes waiting for disk/uninterruptible sleep).

## Key Tools
- `top`, `htop`, `uptime`, `vmstat`, `iostat`, `strace`, `perf`, `lsof`.

## Sources
- [[sre/companies/apple]]
- [[sre/patterns/troubleshooting-framework]]
