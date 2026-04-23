# Meta Production Engineer (PE) Interview Prep (2026)

## Process
The Meta PE role is a hybrid of Software Engineering and Systems Engineering.

- **Initial Screen** (60 min): 
  - 1 Algorithms problem (LeetCode Medium).
  - 1 Systems Scripting problem (Practical file/system manipulation).
- **On-site** (4-5 rounds):
  - **Coding (Algorithm):** 2 LeetCode-style problems in 45 min.
  - **Coding (Systems):** Focused on Python/Go automation of Linux tasks.
  - **System Design:** Scalable Architecture (e.g., "Design a global logging service").
  - **Systems/Troubleshooting:** Interactive debugging of a broken Linux environment.
  - **Behavioral:** Focus on Meta's core values (Move Fast, Impact).

## Emphasis
- **Meta Top 50:** Meta heavily reuses problem patterns. Focus on the most frequent tagged questions.
- **Pythonic Linux:** Practice using `os`, `sys`, `subprocess`, and `json` modules to solve systems tasks.
- **Speed:** The Algorithm coding round expects 2 Mediums in 45 minutes. Optimize for speed and pattern recognition.
- **Linux Fundamentals:** Deep knowledge of the stack—from system calls (`strace`) to network sockets (`netstat`/`ss`).

## Frequently Tested Topics

### 1. Algorithms (LeetCode Style)
- **Arrays & HashMaps:** Subarray sum, two sum, valid palindrome.
- **Binary Search:** Find element in rotated array, search range.
- **Trees & Graphs:** BFS/DFS are paramount. Number of Islands, Tree Vertical Order.
- **Heaps:** Merge K Sorted Lists, Top K Frequent.

### 2. Systems Scripting (PE Specific)
- **Data Munging:** Merging CSVs, parsing JSON outputs from system tools.
- **Process Management:** Monitoring and killing stale processes based on CPU/Time.
- **Network Health:** URL checkers, port scanners, health check aggregators.
- **Concurrency:** Handling multiple tasks in parallel without race conditions.

## Role-Specific DSA Problems

| Problem | Difficulty | PE Context |
|---|---|---|
| [[dsa/problems/number-of-islands]] | Medium | Cluster discovery / identifying isolated nodes. |
| [[dsa/problems/merge-intervals]] | Medium | Consolidating maintenance schedules or IP ranges. |
| [[dsa/problems/lowest-common-ancestor]] | Medium | Finding shared dependencies in a service tree. |
| [[dsa/problems/top-k-frequent-elements]] | Medium | Identifying the top talkers in a network log. |
| [[dsa/problems/valid-parentheses]] | Easy | Validating configuration file syntax (JSON/YAML). |
| [[dsa/problems/lru-cache]] | Medium | Core building block for high-performance systems. |

## Practical Systems Coding Examples
- **Stale Process Reaper:** Find all processes belonging to user `www` that have been running for more than 24 hours and kill them.
- **Metric Aggregator:** Given a directory of JSON files containing server metrics, calculate the average CPU usage per region.
- **Fastest Dinosaur:** (Classic Meta PE) Given two files—one with names/feet and another with names/speeds—merge them and print the fastest bipedal dinosaurs.

## Troubleshooting Round (The "War Room")
- You are given access to a "broken" Linux environment.
- **Goal:** Find the root cause and fix it.
- **Tools:** `top`, `ps`, `lsof`, `netstat`, `df`, `iostat`, `strace`, `tcpdump`.
- **Common Issues:** Full disk (hidden by open file handles), rogue cron jobs, DNS misconfiguration, firewall rules.

## Behavioral (Meta Values)
- **Impact:** "What was the most impactful thing you built?"
- **Move Fast:** "Tell me about a time you shipped something quickly to fix a major issue."
- **Directness:** "How do you handle conflict with a teammate?"

## Tips
- **Speed is key:** For the 45-min coding round, don't over-engineer. Get a working solution, then optimize.
- **Don't ignore Linux:** You can pass the coding but fail the troubleshooting/systems rounds. Know your `/proc` filesystem.
- **Write clean scripts:** In the systems round, use functions, handle exceptions, and avoid `os.system()`—prefer `subprocess.run()`.
