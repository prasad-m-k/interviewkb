# Log Parsing Script

**Difficulty:** Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** Streaming / Hash Map
**Companies:** [[sre/companies/apple]], [[sre/companies/google]], [[sre/companies/amazon]]

## Problem
You have a 100GB log file. Each line contains an IP address, a timestamp, a request method, and a status code.
Write a script to find the top 10 IP addresses that have the most `5xx` error responses.

## Approach
Since the file is 100GB, we cannot load it into memory. We must process it line-by-line (streaming).

1. **Iterate** through the file line by line.
2. **Filter** lines where the status code starts with `5`.
3. **Count** occurrences of each IP address in a hash map (dictionary).
4. **Sort** the dictionary by value in descending order and take the top 10.

**Edge Case:** If the number of *unique* IPs with 5xx errors is also too large for memory, we would need to use an external sort or a heavy-hitting algorithm like Count-Min Sketch. However, for most interviews, a hash map is sufficient as the number of unique IPs usually fits in RAM.

## Solution (Python)
```python
import sys
from collections import Counter

def get_top_ips(filename, limit=10):
    counts = Counter()
    
    try:
        with open(filename, 'r') as f:
            for line in f:
                parts = line.split()
                if len(parts) < 4:
                    continue
                
                ip = parts[0]
                status = parts[-1] # Assuming status is the last element
                
                if status.startswith('5'):
                    counts[ip] += 1
    except FileNotFoundError:
        print("File not found.")
        return

    for ip, count in counts.most_common(limit):
        print(f"{ip}: {count}")

# Usage: python script.py access.log
if __name__ == "__main__":
    if len(sys.argv) > 1:
        get_top_ips(sys.argv[1])
```

## Solution (One-liner / Bash)
For a quick interview answer or small files:
```bash
grep " 5[0-9][0-9] " access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -n 10
```
*Note: The `sort` in the middle is O(N log N) and can be slow for 100GB.*

## Complexity
- **Time:** O(N) where N is the number of lines (plus O(U log U) to sort unique IPs U).
- **Space:** O(U) where U is the number of *unique* IP addresses.

## Key Insight
Streaming is mandatory for "Big Data" scale. Always ask: "Does the set of unique keys fit in memory?"

## Sources
- [[sre/companies/apple]]
