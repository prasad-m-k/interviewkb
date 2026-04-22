# Log Analysis

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/file-manipulation]], [[sre/concepts/slo-sli-sla]]

## Real-time monitoring

```bash
# Follow a file as it grows
tail -f /var/log/app.log

# Follow multiple files simultaneously
tail -f /var/log/app.log /var/log/nginx/error.log

# Follow last 100 lines, then stream
tail -n 100 -f /var/log/app.log

# Follow with grep filter (use --line-buffered for real-time)
tail -f /var/log/app.log | grep --line-buffered "ERROR"

# systemd logs: follow a specific service
journalctl -u nginx -f

# systemd logs: since last boot, priority error and above
journalctl -b -p err
```

---

## Filtering and extracting from logs

```bash
# Lines containing ERROR in last hour (if logs have timestamps)
grep "ERROR" /var/log/app.log | grep "$(date +'%Y-%m-%d %H')"

# Count errors per minute
grep "ERROR" app.log | awk '{print $1, $2}' | cut -c1-16 | uniq -c

# Extract all HTTP status codes from nginx access log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Extract all unique error messages (strip timestamps and PIDs)
grep "ERROR" app.log | sed 's/[0-9]\{4\}-[0-9-T:\.]*//g' | sort | uniq -c | sort -rn

# Top 10 slowest requests (nginx: $request_time is field 11)
awk '{print $11, $7}' access.log | sort -rn | head -10

# Find requests with response time > 2 seconds
awk '$11 > 2.0 {print $0}' access.log

# All unique IPs that got 5xx errors
awk '$9 ~ /^5/ {print $1}' access.log | sort -u
```

---

## Apache / nginx access log format

Standard combined log format:
```
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326 "http://ref.com/" "Mozilla/5.0"
$1        $2 $3    $4                            $6  $7                 $8    $9  $10   $11              $12
```

Fields: `$1`=IP, `$4`=timestamp, `$6`=method, `$7`=path, `$9`=status, `$10`=bytes, `$11`=referer, `$12`=user-agent

```bash
# Count requests by status code
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Top 10 URLs by request count
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head

# Top 10 IPs by bandwidth
awk '{bytes[$1] += $10} END {for (ip in bytes) print bytes[ip], ip}' access.log | sort -rn | head

# Error rate (5xx / total)
total=$(wc -l < access.log)
errors=$(awk '$9 ~ /^5/' access.log | wc -l)
echo "scale=4; $errors / $total" | bc
```

---

## Searching compressed log archives

```bash
# Search gzip archives without decompressing
zgrep "ERROR" /var/log/app.log.gz
zgrep "ERROR" /var/log/app.log.*.gz

# Read gzip
zcat /var/log/app.log.1.gz | grep "FATAL"

# Search across all rotated logs (compressed and plain)
grep -h "ERROR" /var/log/app.log /var/log/app.log.1
zcat /var/log/app.log.*.gz | grep "ERROR"
```

---

## journalctl — systemd logs

```bash
# Logs for a service
journalctl -u nginx

# Follow
journalctl -u nginx -f

# Since a time
journalctl -u nginx --since "2026-04-21 10:00" --until "2026-04-21 11:00"

# Since last boot
journalctl -b

# Priority filter (emerg alert crit err warning notice info debug)
journalctl -p err

# Last 100 lines
journalctl -n 100

# Output formats
journalctl -o json-pretty -u nginx -n 5
journalctl -o short-iso -u nginx

# Disk usage
journalctl --disk-usage

# Vacuum old logs
journalctl --vacuum-time=7d
```

---

## Building log analysis pipelines — interview patterns

**Pattern: error rate spike investigation**
```bash
# 1. When did it start?
grep "ERROR" app.log | awk '{print $1}' | uniq -c

# 2. What are the distinct error types?
grep "ERROR" app.log | awk -F'ERROR' '{print $2}' | sort | uniq -c | sort -rn

# 3. Which endpoints are affected?
grep "ERROR" app.log | awk '{print $7}' | sort | uniq -c | sort -rn

# 4. Is it correlated with a specific user or IP?
grep "ERROR" app.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

**Pattern: "Parse this log and show me errors per minute"**
```bash
grep "ERROR" app.log \
  | awk '{print $1, $2}' \
  | awk -F'[:.]' '{print $1":"$2}' \
  | sort | uniq -c
```

**Pattern: extract a metric from logs (e.g., latency percentiles)**
```bash
# Extract all response times, compute p50/p95/p99
awk '{print $NF}' app.log | sort -n | awk '
  BEGIN {lines=0}
  {data[lines++]=$1}
  END {
    print "p50:", data[int(lines*0.50)]
    print "p95:", data[int(lines*0.95)]
    print "p99:", data[int(lines*0.99)]
  }'
```

## Sources
*(none yet)*
