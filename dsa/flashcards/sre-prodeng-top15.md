# Flashcards — Top 15 DSA & Systems for SRE/PE (Google & Meta)

**Topic:** [[dsa/topics/algorithms]], [[sre/index]], [[mlops/topics/mlops]]
**Created:** 2026-04-22

Format: read the question, formulate your answer, then expand the callout to check.

---

## Card 1: Log Parsing (Streaming)
**Q: You need to parse a 100GB log file to find the Top 10 IP addresses by request count. The machine has only 4GB of RAM. How do you implement this efficiently?**

> [!note]- Answer
> **Approach:** External Merge Sort or Streaming with a Dictionary (if IP cardinality fits).
>
> **Implementation:**
> 1. **Streaming:** Use a Python generator or Go `Scanner` to read the file line by line. Do NOT use `.readlines()`.
> 2. **Counting:** Use a `collections.Counter` (Python) to store `{ip: count}`.
>    - *RAM Check:* An IPv4 address is 4 bytes + Python object overhead. 1 million IPs ≈ 50-100MB. Most production logs have < 10M unique IPs, fitting easily in 4GB.
> 3. **Top K:** Use `counter.most_common(10)` which uses a heap internally (O(N log 10)).
>
> **If IPs don't fit in RAM:**
> 1. Use **External Sort**: Split file into chunks that fit in RAM, sort them, and write to temp files.
> 2. Use a **K-way Merge** (using a min-heap) to stream the sorted chunks and aggregate counts.
>
> **Complexity:** O(N) time, O(unique_ips) space.

---

## Card 2: Token Bucket Rate Limiter
**Q: Implement a Token Bucket rate limiter. How does it handle bursts vs. average rate?**

> [!note]- Answer
> **Logic:**
> - A bucket holds up to `max_tokens`.
> - Tokens are added at a fixed `refill_rate` (e.g., 10 tokens/sec).
> - Each request takes 1 token. If the bucket is empty, the request is dropped/throttled.
>
> **Implementation (Lazy Refill):**
> Instead of a background thread, calculate tokens on every request:
> `current_tokens = min(max_tokens, current_tokens + (now - last_refill_time) * refill_rate)`
>
> **Bursts:** The bucket allows bursts up to `max_tokens`.
> **Average Rate:** Once the bucket is empty, the rate is strictly limited to the `refill_rate`.
>
> **SRE Context:** Critical for protecting services from thundering herds.
>
> *See also:* [[solution-arch/concepts/rate-limiting]]

---

## Card 3: Topological Sort (Dependencies)
**Q: You have 1,000 microservices with a list of dependencies. How do you determine a safe start-up order? What if there's a circular dependency?**

> [!note]- Answer
> **Algorithm:** Kahn’s Algorithm (Topological Sort).
>
> **Steps:**
> 1. Build an adjacency list and calculate the **in-degree** (number of dependencies) for each service.
> 2. Put all services with `in-degree == 0` into a queue.
> 3. While queue is not empty:
>    - Pop service `S`, add to start-up order.
>    - For each service `D` that depends on `S`: decrement `in-degree[D]`.
>    - If `in-degree[D] == 0`, add `D` to the queue.
>
> **Circular Dependency:** If the start-up order length < total services, a cycle exists.
>
> **Complexity:** O(V + E) time and space.
>
> *See also:* [[dsa/problems/course-schedule-ii]]

---

## Card 4: Binary Search (Log Analysis)
**Q: You have a sorted log file with millions of entries. How do you find the first log entry that occurred after `10:00:00 AM`?**

> [!note]- Answer
> **Approach:** Binary Search on the file offsets.
>
> **Steps:**
> 1. Get total file size using `os.path.getsize()`.
> 2. `low = 0`, `high = file_size`.
> 3. `mid = (low + high) // 2`.
> 4. `file.seek(mid)`. 
> 5. **Crucial:** Move to the start of the *next* full line (read until `\n`).
> 6. Parse the timestamp and compare to `10:00:00 AM`.
> 7. Adjust `low` or `high` accordingly.
>
> **Complexity:** O(log(File_Size) * Avg_Line_Length) time, O(1) space.

---

## Card 5: Monotonic Stack (Metrics)
**Q: Given a stream of CPU usage metrics, find the time it takes for each measurement to be exceeded by a higher measurement.**

> [!note]- Answer
> **Approach:** Monotonic Decreasing Stack.
>
> **Algorithm:**
> 1. Iterate through metrics with index `i` and value `v`.
> 2. While `stack` and `metrics[stack[-1]] < v`:
>    - `prev_idx = stack.pop()`
>    - `result[prev_idx] = i - prev_idx`
> 3. Push `i` onto the stack.
>
> **Complexity:** O(N) time (each element pushed/popped once), O(N) space.
>
> *See also:* [[dsa/problems/daily-temperatures]]

---

## Card 6: Concurrency (Worker Pool)
**Q: You need to check the health of 10,000 URLs. How do you do this in Python without spawning 10,000 threads?**

> [!note]- Answer
> **Approach:** Worker Pool (Thread Pool) or Asyncio.
>
> **Implementation (ThreadPoolExecutor):**
> ```python
> from concurrent.futures import ThreadPoolExecutor
> def check_url(url): ...
> with ThreadPoolExecutor(max_workers=100) as executor:
>     executor.map(check_url, url_list)
> ```
>
> **Implementation (Asyncio):**
> Use `aiohttp` and a `Semaphore` to limit concurrency.
>
> **Why?** Spawning 10k threads causes high memory overhead and context-switching churn. 100-500 workers is usually optimal for network-bound tasks.

---

## Card 7: Lowest Common Ancestor (Service Mesh)
**Q: In a service dependency tree, how do you find the closest common dependency of two leaf services?**

> [!note]- Answer
> **Algorithm:** Lowest Common Ancestor (LCA).
>
> **Recursive Approach:**
> - If current node is `null` or matches either service, return current node.
> - Recurse left and right.
> - If left AND right return non-null, current node is LCA.
> - Else return whichever is non-null.
>
> **Complexity:** O(N) time, O(H) space.
>
> *See also:* [[dsa/problems/lowest-common-ancestor]]

---

## Card 8: Interval Merging (Availability)
**Q: A server has multiple maintenance windows `[[start, end]]`. How do you find the total time the server was down?**

> [!note]- Answer
> **Approach:** Merge Intervals.
>
> **Algorithm:**
> 1. Sort intervals by `start` time.
> 2. Initialize `merged = [intervals[0]]`.
> 3. For each `interval`:
>    - If `interval.start <= merged[-1].end`: `merged[-1].end = max(merged[-1].end, interval.end)`.
>    - Else: `merged.append(interval)`.
> 4. Sum `(end - start)` for each interval in `merged`.
>
> **Complexity:** O(N log N) due to sorting.
>
> *See also:* [[dsa/problems/merge-intervals]]

---

## Card 9: LRU Eviction (Caching)
**Q: Why is a HashMap alone insufficient for an LRU cache?**

> [!note]- Answer
> **Reason:** HashMaps do not maintain order.
>
> **The Solution:** Combine a **HashMap** with a **Doubly Linked List**.
> - **HashMap:** Provides O(1) access to any node.
> - **Doubly Linked List:** Provides O(1) removal and insertion at the ends (given the node pointer from the HashMap).
>
> **When a key is accessed:** Move its corresponding node to the "Most Recently Used" end of the DLL.
> **When capacity is exceeded:** Remove the node from the "Least Recently Used" end.
>
> *See also:* [[dsa/problems/lru-cache]]

---

## Card 10: Linux - File Descriptors
**Q: A process is throwing "Too many open files". How do you debug which files are open and how do you increase the limit?**

> [!note]- Answer
> **Debug:**
> - `lsof -p <pid>`: List open files for the process.
> - `ls /proc/<pid>/fd | wc -l`: Count open file descriptors.
>
> **Limits:**
> - `ulimit -n`: Check current shell session limit.
> - `/etc/security/limits.conf`: Permanent system-wide/user-level limits.
> - `sysctl fs.file-max`: System-wide kernel limit.
>
> **SRE Tip:** Often caused by connection leaks (sockets not being closed).

---

## Card 11: Linux - Zombie Processes
**Q: What is a zombie process and how do you "kill" it?**

> [!note]- Answer
> **Definition:** A process that has finished execution but still has an entry in the process table. It happens because the parent hasn't yet read its exit status via `wait()`.
>
> **How to kill:** You cannot `kill -9` a zombie (it's already dead).
> 1. Send `SIGCHLD` to the parent to nudge it to `wait()`.
> 2. If that fails, kill the **parent** process. The zombie will be inherited by `init` (PID 1), which will properly reap it.

---

## Card 12: Network - TCP 3-Way Handshake
**Q: Describe the TCP 3-way handshake. What happens if the final ACK is lost?**

> [!note]- Answer
> **Handshake:**
> 1. Client → Server: `SYN`
> 2. Server → Client: `SYN-ACK`
> 3. Client → Server: `ACK`
>
> **Lost final ACK:**
> - The server stays in `SYN_RECV` state and will retransmit the `SYN-ACK`.
> - If the client starts sending data immediately, the data packet often carries the `ACK` in its header, which completes the handshake.

---

## Card 13: Google NALSD - Throughput
**Q: How long does it take to transfer 100TB over a 10Gbps link? (Give the "SRE Napkin Math" answer)**

> [!note]- Answer
> **Math:**
> 1. 10 Gbps = 1.25 GB/s (Divide by 8 bits).
> 2. 100 TB = 100,000 GB.
> 3. 100,000 / 1.25 = 80,000 seconds.
> 4. 80,000 / 3,600 ≈ **22.2 hours**.
>
> **Real-world SRE answer:** Factor in TCP overhead and disk I/O limits. Usually assume ~80% efficiency → **~28 hours**.

---

## Card 14: Meta PE - Dinosaur Problem (Data Merging)
**Q: You have two files: `stats.csv` (Name, Leg_Length, Diet) and `speed.csv` (Name, Speed). How do you print the names of only bipedal (2-legged) dinosaurs from fastest to slowest?**

> [!note]- Answer
> **Approach:**
> 1. Read `stats.csv`. Filter for bipedal and store in a dictionary: `{name: True}`.
> 2. Read `speed.csv`. If `name` is in your dictionary, store `(speed, name)` in a list.
> 3. Sort the list by speed descending.
> 4. Print the names.
>
> **Complexity:** O(N + M + K log K) where N, M are file lines and K is bipedal count.

---

## Card 15: SRE - Error Budgets
**Q: If your SLO is 99.9% uptime per month, how much downtime are you allowed in minutes?**

> [!note]- Answer
> **Math:**
> - 100% - 99.9% = 0.1% error budget.
> - 30 days * 24 hours * 60 minutes = 43,200 minutes.
> - 0.1% of 43,200 = **43.2 minutes**.
>
> **SRE Tip:** 99.99% (four nines) is only **~4.3 minutes** per month.
