# Networking Troubleshooting

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/patterns/troubleshooting-framework]], [[sre/concepts/slo-sli-sla]]

## The 7-layer mental model for interviews
When a service is unreachable, walk the stack from bottom up:

```
7. Application  — HTTP 5xx? Bad request routing? App bug?
6. Presentation — TLS cert expired? Cipher mismatch?
5. Session      — connection refused? timeout?
4. Transport    — TCP port open? firewall blocking? SYN/ACK?
3. Network      — routing issue? wrong subnet? ICMP blocked?
2. Data link    — ARP? VLAN?
1. Physical     — cable? NIC? (rare in cloud)
```

In practice: ping → traceroute → port check → curl → logs.

---

## ping — basic reachability

```bash
ping -c 4 google.com           # 4 packets
ping -i 0.2 -c 20 host         # flood test (fast)

# ICMP blocked? Try TCP ping:
hping3 -S -p 443 google.com
```

---

## dig / nslookup — DNS

```bash
# Resolve a hostname
dig google.com

# Query specific record type
dig google.com A
dig google.com AAAA
dig google.com MX
dig google.com NS
dig google.com TXT

# Use a specific DNS server
dig @8.8.8.8 google.com

# Reverse lookup (PTR)
dig -x 8.8.8.8

# Short output
dig +short google.com

# Trace full resolution path
dig +trace google.com

# Check for DNS propagation
dig @8.8.8.8 example.com   # Google DNS
dig @1.1.1.1 example.com   # Cloudflare DNS
```

**Interview question**: "Your app can't reach the database by hostname but can by IP." → DNS resolution issue. Check `/etc/resolv.conf`, `/etc/hosts`, DNS server availability, TTL caching.

---

## curl — HTTP testing

```bash
# Basic GET
curl https://api.example.com/health

# Show response headers and body
curl -i https://api.example.com/health

# Verbose (shows TLS handshake, headers)
curl -v https://api.example.com/health

# POST with JSON body
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"value"}' https://api.example.com/endpoint

# Follow redirects
curl -L https://example.com

# Resolve timing breakdown
curl -w "\ntotal: %{time_total}s\ndns: %{time_namelookup}s\nconnect: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\n" \
  -o /dev/null -s https://api.example.com

# Test with a specific Host header (bypass DNS)
curl -H "Host: api.example.com" http://10.0.0.5/health

# Test TLS cert
curl -vI https://example.com 2>&1 | grep -E "expire|issuer|subject"
```

---

## ss / netstat — connections and ports

```bash
# All listening ports (TCP + UDP)
ss -tulnp
# t=TCP u=UDP l=listening n=numeric p=process

# All established TCP connections
ss -tnp state established

# Connections to a specific port
ss -tnp dst :443

# Count connections per state
ss -s

# Legacy netstat (same flags)
netstat -tulnp
netstat -an | grep ESTABLISHED | wc -l

# How many connections per remote IP?
ss -tn | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

**TIME_WAIT accumulation**: many connections in TIME_WAIT means clients are initiating closes and ports are in cooldown (60s). Under high load this exhausts ephemeral ports. Fix: enable `SO_REUSEADDR`, reduce `tcp_fin_timeout`, or use connection pooling.

---

## traceroute / mtr — path and latency

```bash
traceroute google.com
traceroute -T -p 443 google.com   # TCP traceroute (bypasses ICMP blocks)

# mtr: continuous, combines ping + traceroute
mtr google.com
mtr --report google.com           # 10-packet report
```

**Reading traceroute**: each hop is a router. A `* * *` means ICMP is filtered at that hop (not necessarily a problem). Latency spike at hop N and stays high = bottleneck at hop N. Latency spike at hop N then drops = that router de-prioritizes ICMP.

---

## tcpdump — packet capture

```bash
# Capture on interface eth0
tcpdump -i eth0

# Capture HTTP traffic
tcpdump -i eth0 port 80

# Capture traffic to/from a host
tcpdump -i eth0 host 10.0.0.5

# Write to file for Wireshark
tcpdump -i eth0 -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Filter: TCP SYN packets (connection attempts)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Show packet contents (ASCII)
tcpdump -i eth0 -A port 8080
```

---

## iptables — firewall rules

```bash
# List all rules
iptables -L -n -v

# Allow inbound on port 8080
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Block an IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Delete a rule (same syntax as -A but use -D)
iptables -D INPUT -s 1.2.3.4 -j DROP

# Flush all rules (dangerous — removes all rules)
iptables -F
```

---

## Common networking interview scenarios

**"Service is unreachable from outside the cluster"**
```bash
# Step 1: Is it listening?
ss -tulnp | grep :8080

# Step 2: Can I reach it locally?
curl localhost:8080/health

# Step 3: Firewall blocking?
iptables -L -n | grep 8080
# or cloud: check security group / VPC rules

# Step 4: Is the hostname resolving correctly?
dig service.internal

# Step 5: Routing?
traceroute 10.0.1.5
```

**"Intermittent timeouts to the database"**
```bash
# Check packet loss along path
mtr --report db.internal

# Connection queue full?
ss -s | grep -i listen

# TCP retransmits
netstat -s | grep retransmit
cat /proc/net/netstat | grep TcpExt

# Is it DNS? Time each resolution
for i in $(seq 1 10); do time dig db.internal; done
```

**"TLS certificate error"**
```bash
# Check cert expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check cert chain
curl -vI https://example.com 2>&1 | grep -E "expire|issuer|verify"

# Check which cert is being served
openssl s_client -connect example.com:443 -servername example.com
```

## Sources
*(none yet)*
