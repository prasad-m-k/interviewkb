# SSL Certificate Expiry Checker

**Difficulty:** Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** Parallel I/O / subprocess / Network
**Companies:** [[sre/companies/apple]], [[sre/companies/meta]], [[sre/companies/google]]

## Problem

Write a script that, given a list of hostnames, **checks the SSL/TLS certificate expiry date for each host** and reports which certificates expire within a warning threshold (e.g., 30 days) or are already expired.

Input: a file or stdin with one `host[:port]` per line.
Output: a table of `host | expiry date | days remaining | status`.

```
host                    expiry              days    status
api.example.com         2026-05-10          18      WARN
db.internal             2025-12-31          -113    EXPIRED
cdn.example.com         2027-01-15          268     OK
```

## Why This Is Tricky

- Raw SSL connections require the `ssl` module — using `subprocess openssl` is easier in scripts but fragile on paths.
- Certificates on SNI-enabled servers require the `server_hostname` parameter — without it, you get the default cert (often wrong).
- Connections can time out on firewalled ports — must set a socket timeout.
- Parallelism matters for checking 500 hosts — sequential takes minutes; `ThreadPoolExecutor` brings it to seconds.

## Approach

1. For each host, open a TLS socket using Python's `ssl` module.
2. Set `server_hostname` for SNI support.
3. Extract `notAfter` from the peer certificate.
4. Compute days remaining.
5. Classify as `EXPIRED`, `WARN`, or `OK`.
6. Run checks in parallel with a thread pool (I/O-bound, safe to parallelize).

## Solution (Python)

```python
import ssl
import socket
import sys
import concurrent.futures
from datetime import datetime, timezone, timedelta

WARN_DAYS   = 30
TIMEOUT_SEC = 5

def check_cert(host_spec: str) -> dict:
    """Check SSL cert for 'host' or 'host:port'. Returns a result dict."""
    if ':' in host_spec:
        host, port_str = host_spec.rsplit(':', 1)
        port = int(port_str)
    else:
        host, port = host_spec, 443

    result = {"host": host_spec, "expiry": None, "days": None, "status": "ERROR", "error": ""}
    ctx = ssl.create_default_context()

    try:
        with socket.create_connection((host, port), timeout=TIMEOUT_SEC) as sock:
            with ctx.wrap_socket(sock, server_hostname=host) as ssock:
                cert = ssock.getpeercert()
                not_after = cert.get("notAfter", "")
                expiry_dt = datetime.strptime(not_after, "%b %d %H:%M:%S %Y %Z")
                expiry_dt = expiry_dt.replace(tzinfo=timezone.utc)
                now = datetime.now(tz=timezone.utc)
                days = (expiry_dt - now).days

                result["expiry"] = expiry_dt.strftime("%Y-%m-%d")
                result["days"]   = days
                if days < 0:
                    result["status"] = "EXPIRED"
                elif days <= WARN_DAYS:
                    result["status"] = "WARN"
                else:
                    result["status"] = "OK"

    except ssl.SSLCertVerificationError as e:
        result["error"] = f"cert invalid: {e.reason}"
    except (socket.timeout, ConnectionRefusedError, OSError) as e:
        result["error"] = str(e)

    return result

def main():
    hosts = [line.strip() for line in sys.stdin if line.strip() and not line.startswith('#')]

    print(f"{'HOST':<30} {'EXPIRY':<14} {'DAYS':>6}  STATUS")
    print("-" * 60)

    with concurrent.futures.ThreadPoolExecutor(max_workers=20) as pool:
        futures = {pool.submit(check_cert, h): h for h in hosts}
        for future in concurrent.futures.as_completed(futures):
            r = future.result()
            if r["error"]:
                print(f"{r['host']:<30} {'N/A':<14} {'N/A':>6}  {r['status']} ({r['error']})")
            else:
                print(f"{r['host']:<30} {r['expiry']:<14} {r['days']:>6}  {r['status']}")

if __name__ == "__main__":
    main()
```

## Usage

```bash
# Check a list of hosts
echo -e "api.example.com\ncdn.example.com:443\nmail.example.com:993" | python cert_checker.py

# Pipe from a file
cat hosts.txt | python cert_checker.py | grep -E "WARN|EXPIRED"

# Shell equivalent for a single host
echo | openssl s_client -servername api.example.com -connect api.example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate
```

## Complexity

- **Time:** O(H / T) where H = number of hosts, T = thread pool size. Each check is ~TIMEOUT_SEC worst case.
- **Space:** O(T) for active connections, O(H) for results.

## Key Insight

SSL expiry checks are purely I/O-bound (waiting for TCP + TLS handshake). `ThreadPoolExecutor` parallelizes I/O-bound work correctly in Python despite the GIL. For CPU-bound work, use `ProcessPoolExecutor` instead.

## Variants / Follow-ups

- "What if a cert uses a chain with an intermediate CA — do you check the full chain?" → `ssl.create_default_context()` validates the full chain automatically. To inspect intermediate certs, use `ssock.getpeercert(binary_form=True)` + the `cryptography` library.
- "How do you get the cert without connecting (e.g., from a file)?" → `ssl.PEM_cert_to_DER_cert` + `cryptography.x509.load_der_x509_certificate`.
- "Integrate with alerting" → Pipe `WARN` and `EXPIRED` lines to PagerDuty API or post to Slack webhook.
- "How do you schedule this for fleet-wide daily checks?" → Run via cron, store results in a time-series DB, alert on threshold crossing.
- "What about mTLS or client certs?" → Pass `ctx.load_cert_chain(certfile, keyfile)` before connecting.

## Sources

- [[sre/companies/apple]]
- [[sre/companies/meta]]
