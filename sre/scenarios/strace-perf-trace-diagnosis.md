# Troubleshooting: Reading strace / perf Output Cold

**Topic:** [[sre/topics/linux-cli]]
**Pattern:** [[sre/patterns/troubleshooting-framework]]
**Related:** [[sre/concepts/process-signals]], [[sre/scenarios/high-cpu-troubleshooting]]
**Companies:** [[sre/companies/google]]
**Level:** L6 — Google explicitly hands candidates raw trace output with no framing and expects them to narrate the diagnosis live; the skill is syscall/symbol literacy under time pressure, not knowing the commands exist.

## Scenario
The interviewer pastes real `strace` or `perf` output with no context beyond "this process is slow, what's wrong?" You have to read the trace cold and narrate your reasoning out loud before you get more information. Below are the three shapes this most commonly takes.

## Shape 1: strace showing a syscall storm
```
epoll_wait(4, [], 0, 0)                = 0
epoll_wait(4, [], 0, 0)                = 0
epoll_wait(4, [], 0, 0)                = 0
... (repeated thousands of times, 0 events each) ...
```
**Read this as:** a busy-poll loop — the process is calling `epoll_wait` with a zero or near-zero timeout in a tight loop instead of blocking until an event arrives. It burns CPU without doing useful work. Say out loud: "This looks like a missing or misconfigured timeout on the event loop — I'd check the epoll_wait timeout argument and whether the app should be blocking here instead of polling."

## Shape 2: strace showing failed syscalls
```
open("/etc/app/config.yaml", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/etc/app/config.yaml", O_RDONLY) = -1 ENOENT (No such file or directory)
stat("/var/cache/app/session_9f3a", ...) = -1 EACCES (Permission denied)
```
**Read this as:** the repeated identical failed `open()` is a retry loop against a file that will never appear — a config path bug, not a transient issue (transient would show occasional success). The `EACCES` is a permissions problem, not a missing-file problem — different fix (check ownership/mode, not the path). Naming the *specific errno* out loud (`ENOENT` vs `EACCES` vs `EMFILE`) is what separates a strong answer from a vague one.

## Shape 3: perf top showing where cycles actually go
```
  Overhead  Shared Object       Symbol
    42.31%  libc.so.6           __memcpy_avx_unaligned
    18.02%  app_binary          json_parse_value
     9.15%  [kernel]            copy_user_enhanced_fast_string
     6.40%  libpthread.so       pthread_mutex_lock
```
**Read this as:** almost half the cycles are in `memcpy` — the app is copying large buffers far more than expected for its workload. Combined with `json_parse_value` showing up prominently, the likely story is "parsing a payload that's larger than it should be, or parsing it in a way that copies the buffer repeatedly (e.g. re-parsing on every retry, or a streaming parser that isn't actually streaming)." The `pthread_mutex_lock` overhead is a secondary finding worth flagging — contention on a shared lock, worth a follow-up question about how many threads are blocked on it (`cat /proc/<pid>/status`, thread count vs. runnable count).

## The narration structure that scores well
Regardless of which shape you get, walk through it in this order out loud:
1. **What's repeating and how fast** — a single slow call is a different problem than a fast tight loop.
2. **What each line's return value / overhead % actually means** — don't just read syscall names, read the *result*.
3. **The most likely root cause given the pattern**, stated as a hypothesis, not a certainty: "this looks like X — I'd confirm by checking Y next."
4. **What you'd check next** to confirm or rule the hypothesis out — interviewers are grading whether you know the next command, not just the diagnosis.

## Why this is asked at L6 specifically
Anyone can run `strace` when told a process is slow. Reading output cold, correctly distinguishing a busy-loop from a blocked-on-I/O pattern from a lock-contention pattern, and naming the specific next diagnostic step — without being told what to look for — is what separates "knows the tool exists" from "has actually debugged production systems under pressure." This is functionally an oral exam on [[sre/concepts/process-signals]] and syscall literacy.

## Sources
- [[sre/companies/google]]
- [[sre/scenarios/high-cpu-troubleshooting]]
