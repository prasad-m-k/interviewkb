---
tags:
  - sre
  - networking
  - security
  - interview-prep
---

# Concept: Networking Fundamentals

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/networking-troubleshooting]], [[sre/concepts/load-balancers]], [[devops/topics/security-devsecops]]

---

## TCP Three-Way Handshake

Before any data flows, TCP establishes a connection. This costs **one full round-trip** before the first byte of application data.

```
Client                              Server
  │                                   │
  │──── SYN  (seq=100) ──────────────▶│  "I want to connect, my seq starts at 100"
  │                                   │
  │◀─── SYN-ACK (seq=200, ack=101) ───│  "OK, my seq starts at 200; got your 100"
  │                                   │
  │──── ACK  (ack=201) ──────────────▶│  "Got it. Connection open."
  │                                   │
  │══════════ data flows ══════════════│
```

**Cost:** 1 RTT before data. TLS adds 1–2 more RTTs on top (TLS 1.2 = 2 RTT, TLS 1.3 = 1 RTT).

### TCP four-way close

```
Client                              Server
  │──── FIN ────────────────────────▶│  "Done sending"
  │◀─── ACK ──────────────────────── │  "Got your FIN"
  │◀─── FIN ──────────────────────── │  "I'm done too"
  │──── ACK ────────────────────────▶│  Both ends closed
```

### Key TCP flags

| Flag | Meaning |
|---|---|
| `SYN` | Synchronize sequence numbers (connection open) |
| `ACK` | Acknowledge received data |
| `FIN` | Finished sending (graceful close) |
| `RST` | Reset — abrupt close, no further communication |
| `PSH` | Push data to application immediately, don't buffer |

### TIME_WAIT

After closing, the initiator holds the port for **2×MSL (~60s)** to absorb delayed packets from the network. Under high connection churn (thousands of short-lived connections/sec), you can exhaust local ports. Fixes: `SO_REUSEADDR`, `tcp_tw_reuse` sysctl, or connection pooling.

---

## TLS Handshake (Encryption at L7)

TLS authenticates the server, negotiates cipher suite, and derives symmetric session keys. Runs above TCP.

### TLS 1.2 — 2 RTTs after TCP

```
Client                                  Server
  │──── ClientHello ──────────────────▶│
  │     supported versions,             │
  │     cipher suites, random_C         │
  │                                     │
  │◀─── ServerHello + Certificate ──────│
  │     chosen cipher, random_S,        │
  │     server's cert chain             │
  │                                     │
  │  [Client verifies cert against      │
  │   OS/browser trust store]           │
  │                                     │
  │──── ClientKeyExchange ─────────────▶│  pre-master secret, encrypted with server pubkey
  │──── ChangeCipherSpec + Finished ───▶│  "switching to symmetric encryption"
  │                                     │
  │◀─── ChangeCipherSpec + Finished ────│
  │                                     │
  │══════════ encrypted data ═══════════│
```

### TLS 1.3 — 1 RTT (0-RTT for resumption)

```
Client                                  Server
  │──── ClientHello ──────────────────▶│
  │     key_share (DH public key),      │
  │     supported groups, random_C      │
  │                                     │
  │◀─── ServerHello + Cert + Finished ──│  everything in one flight
  │                                     │
  │──── Finished ──────────────────────▶│
  │                                     │
  │══════════ encrypted data ═══════════│
```

**TLS 1.3 improvements:**
- 1 RTT instead of 2 (50% fewer round trips)
- Forward secrecy is mandatory — ephemeral Diffie-Hellman only
- Removed weak ciphers: RC4, 3DES, SHA-1, RSA key exchange
- 0-RTT session resumption (caution: replay attacks on non-idempotent endpoints)

### How session keys are derived

```
Client random + Server random + Pre-master secret
              │
              ▼  PRF (TLS 1.2) / HKDF (TLS 1.3)
         Master Secret
              │
              ▼
  ┌───────────────────────────┐
  │ client_write_key  (C→S)   │  symmetric encryption
  │ server_write_key  (S→C)   │  symmetric encryption
  │ client_write_MAC          │  integrity
  │ server_write_MAC          │  integrity
  └───────────────────────────┘
```

The server's **private key** is used only to authenticate during the handshake. Data is encrypted with fast symmetric keys (AES-GCM). This is why **forward secrecy** matters — if the private key leaks later, past sessions (which used ephemeral DH keys) cannot be decrypted.

### Certificate chain of trust

```
Root CA  (self-signed; offline; e.g. DigiCert Root G4)
  └── Intermediate CA  (online; signs end-entity certs)
          └── Leaf Cert  (api.example.com; valid 90 days)
```

A browser verifies: leaf cert → signed by intermediate → signed by root → root in OS trust store. A self-signed cert breaks the chain → browser warning. Corporate environments push an enterprise root CA via MDM/GPO so custom certs are trusted.

---

## Encryption at Different Network Layers

```
OSI Layer    Protocol           What's encrypted              Visible to middleboxes
──────────────────────────────────────────────────────────────────────────────────────
L7 App       HTTPS/TLS          HTTP headers + body           SNI hostname, cert CN
L4 Transport WireGuard / DTLS   TCP/UDP payload               Src/dst IP, port
L3 Network   IPsec (ESP)        Full IP payload incl. headers Src/dst IP only
L2 Data      MACsec             Ethernet frame payload        MAC addresses only
```

### Comparison table

| | HTTPS (TLS) | VPN (WireGuard/IPsec) | mTLS (service mesh) |
|---|---|---|---|
| **Layer** | L7 | L3/L4 | L7 |
| **Authenticates** | Server (+ optional client) | Both peers | Both sides (mutual cert) |
| **Encrypts** | HTTP body + headers | All IP traffic | Service-to-service traffic |
| **Use case** | Web/API traffic | Remote access, site-to-site | Microservices zero trust |
| **Termination** | At the server / LB | At the VPN endpoint | At the sidecar proxy |

---

## Proxies

### Forward Proxy

Sits between client and internet. Client sends requests to the proxy, which forwards them onward.

```
                  ┌──────────────────┐
Client ──────────▶│  Forward Proxy   │──────────▶  Internet / Servers
                  │  (Squid, Zscaler)│
                  └──────────────────┘
                         ▲
               Client knows about it
               (explicit config) OR
               traffic is intercepted
               (transparent proxy)
```

**Use cases:** corporate egress control (block social media, log all traffic), caching (Squid), anonymization, egress IP normalization (all traffic exits from proxy IP).

**Explicit vs transparent:**
- **Explicit:** client configured with `http_proxy=http://proxy:3128`
- **Transparent:** iptables/TPROXY intercepts traffic — no client config needed

### Reverse Proxy

Sits in front of servers. Clients talk to the proxy; proxy routes to backends. Clients never see backend addresses.

```
              ┌──────────────────────┐
              │    Reverse Proxy     │──▶ Backend 1  :8080
Internet ────▶│  (nginx, HAProxy,    │──▶ Backend 2  :8080
              │   Envoy, Cloudflare) │──▶ Backend 3  :8080
              └──────────────────────┘
```

**What a reverse proxy provides:**

| Feature | Detail |
|---|---|
| Load balancing | Round-robin, least-conn, IP hash across backends |
| TLS termination | Proxy handles TLS; backends receive plain HTTP |
| Caching | Cache static assets, API responses |
| Compression | gzip/brotli at the proxy layer |
| Auth gateway | Validate JWT/OAuth before requests reach backends |
| Rate limiting | Per IP, per API key |
| Health checking | Remove unhealthy backends automatically |
| HTTP/2 + HTTP/3 | Upgrade client connections; backends can stay HTTP/1.1 |

**nginx reverse proxy example:**
```nginx
upstream backend {
    server app1:8080;
    server app2:8080;
    server app3:8080;
    keepalive 32;        # reuse connections to backends
}

server {
    listen 443 ssl http2;
    ssl_certificate     /etc/nginx/tls/cert.pem;
    ssl_certificate_key /etc/nginx/tls/key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass         http://backend;
        proxy_set_header   X-Forwarded-For  $remote_addr;
        proxy_set_header   X-Forwarded-Proto https;
        proxy_set_header   Host             $host;
    }
}
```

### CONNECT Tunneling (HTTPS through HTTP proxy)

```
Client ──── CONNECT api.example.com:443 HTTP/1.1 ────▶ Proxy
Proxy opens TCP tunnel to api.example.com:443
Client and server do TLS directly through the tunnel

Client ══════════ TLS (opaque to proxy) ══════════ api.example.com
         Proxy sees: destination hostname + byte count only
         Proxy CANNOT decrypt (end-to-end TLS)
```

### API Gateway vs Reverse Proxy

| | Reverse Proxy | API Gateway |
|---|---|---|
| **Focus** | Traffic routing | API lifecycle management |
| **Auth** | Optional, basic | First-class (JWT, OAuth 2.0, API keys) |
| **Rate limiting** | Per IP | Per client / per route / per plan |
| **Transforms** | Minimal | Request/response transformation, protocol bridging |
| **Developer portal** | No | Yes (Kong, Apigee, AWS API GW) |
| **Examples** | nginx, HAProxy, Envoy | Kong, AWS API GW, Apigee, Traefik |

---

## Modern Zero Trust: Zscaler / SASE

### The old model (castle-and-moat)

```
┌──────────────── Corporate Network ────────────────┐
│  "Inside = trusted"                               │
│  Laptop ─────────────────────── App Server        │
└───────────────────────────────────────────────────┘
         ▲
   VPN grants full network access
   Compromise 1 laptop = lateral movement to everything
```

**Problem:** remote work dissolved the perimeter. VPN hairpins all traffic through HQ. A compromised device inside the VPN can reach all internal resources.

### Zero Trust principles

```
1. Never trust, always verify   — identity, not network location
2. Least privilege              — access only the specific app needed
3. Assume breach                — verify explicitly; log everything
4. Device posture               — check device health before granting access
```

### Zscaler architecture (SASE = Secure Access Service Edge)

```
Old (VPN hairpin):
  Remote user ──VPN──▶ Corporate HQ ──▶ Internet
                           ▲ All traffic backhauled; latency + bottleneck

Zscaler (cloud-native):
  Remote user ──▶ Nearest Zscaler PoP (150+ globally) ──▶ App / Internet
                       ▲ Security inline at the edge; no backhaul
```

### Zscaler Internet Access (ZIA) — traffic flow

```
User's laptop
  │ (Zscaler Client Connector installed — tunnels all traffic)
  │ IPsec / GRE tunnel to nearest PoP
  ▼
Zscaler PoP
  ├─ SSL Inspection   (decrypt TLS, scan content, re-encrypt)
  ├─ URL Filtering    (category-based allow/block)
  ├─ DLP             (detect PII, credentials in outbound traffic)
  ├─ CASB            (shadow IT detection; block unapproved SaaS)
  ├─ Malware sandbox (detonate suspicious files)
  └─ Threat intel    (block known-bad IPs/domains)
  │
  ▼
Internet / SaaS (Office 365, Salesforce, etc.)
```

### How SSL Inspection works (it's a MITM proxy)

```
User ─────── TLS (cert signed by Zscaler Enterprise CA) ──────── Zscaler PoP
                                                                       │
                                                          TLS (real server cert)
                                                                       │
                                                              api.example.com

The user's device is configured (via MDM / GPO) to trust the
Zscaler Enterprise CA. So the forged cert appears valid.

Zscaler sees: plaintext HTTP. Scans it. Re-encrypts onward.
```

### Zscaler Private Access (ZPA) — replaces VPN

```
Old VPN:
  User ──VPN──▶ Corporate network ──▶ App Server
                  (user gets full L3 network access)

ZPA:
  User ──▶ Zscaler Cloud ◀──── App Connector (on-prem/cloud)
                                    │
                               App Server (port 443 only)

Key differences:
  ✓ User never gets network access — app-level access only
  ✓ App Connector makes outbound connections to Zscaler (no inbound firewall rules)
  ✓ Policy: "this user + this device posture → this app + this port only"
  ✓ Lateral movement impossible — users can't reach other internal resources
```

### Device posture check (before access is granted)

```
Is Zscaler agent running?           ─┐
Is disk encrypted?                   │  All must pass
Is OS patched within 30 days?        │  → Access granted
Is device enrolled in MDM?           │
Is AV running and up-to-date?       ─┘

Fail any check → access denied or quarantine VLAN
```

### Zscaler vs VPN comparison

| | VPN | Zscaler ZPA |
|---|---|---|
| **Access model** | Full L3 network | Per-app only |
| **Lateral movement** | Easy once inside | Impossible by design |
| **Routing** | Hairpin through HQ | Direct to nearest PoP |
| **Visibility** | None after tunnel | Full traffic inspection |
| **Scale** | VPN concentrator limit | Cloud-scale |
| **Zero day exposure** | Large attack surface | App connectors make outbound only |

### Other SASE / Zero Trust tools

| Tool | Category | What it does |
|---|---|---|
| **Cloudflare Access** | ZTNA | Identity-based app access; replaces VPN |
| **CrowdStrike Falcon** | EDR | Endpoint detection; feeds device posture |
| **Okta / Azure AD** | IdP | Identity source for all access decisions |
| **Netskope** | CASB/SWG | SaaS data protection, shadow IT |
| **Palo Alto Prisma** | SASE | Full SASE (Zscaler competitor) |
| **BeyondCorp (Google)** | ZTNA | Google's internal model; origin of zero trust |

---

## Quick Reference: Ports and Protocols

```
Protocol     Port(s)   Notes
──────────────────────────────────────────────────────────
HTTP          80       Plaintext — never use for auth or sensitive data
HTTPS         443      TLS — always verify cert + check SNI
SSH           22       Encrypted shell; prefer pubkey auth
DNS           53       UDP (queries) + TCP (zone transfers); plaintext by default
DNS-over-HTTPS 443     Encrypted DNS — Cloudflare 1.1.1.1, Google 8.8.8.8
SMTP          587      Email submission with STARTTLS
PostgreSQL   5432      TLS optional but recommended
Redis        6379      No auth by default — never expose to internet
Kubernetes   6443      API server TLS + cert-based auth
Prometheus   9090      Metrics scrape endpoint
gRPC         varies    HTTP/2; TLS strongly recommended
WireGuard   51820 UDP  Modern VPN; minimal attack surface
IPsec ESP     50       L3 encryption
```

---

## Common Interview Questions

**Q: "What happens at each layer when you type https://example.com?"**
1. DNS: resolve `example.com` → IP (UDP port 53)
2. TCP: 3-way handshake to port 443
3. TLS: ClientHello → ServerHello+cert → key exchange → session keys
4. HTTP/2: GET / over encrypted channel
5. Response flows back through the same layers

**Q: "What's the difference between a forward proxy and a reverse proxy?"**
Forward = protects/controls clients (egress). Reverse = protects/controls servers (ingress). The "direction" refers to where the anonymization happens.

**Q: "How does a corporate proxy inspect HTTPS traffic?"**
SSL/TLS inspection: proxy acts as MITM, presents its own cert (signed by an enterprise CA the device trusts), decrypts, scans, re-encrypts to the real server. Requires installing the enterprise CA cert on all managed devices.

**Q: "What is forward secrecy and why does it matter?"**
Even if the server's private key is later compromised, past sessions cannot be decrypted because session keys were derived from ephemeral DH parameters (discarded after the handshake). Without forward secrecy (old RSA key exchange), recording encrypted traffic today and stealing the private key tomorrow decrypts everything.

## Sources
- [[sre/concepts/networking-troubleshooting]]
- [[sre/concepts/load-balancers]]
- [[devops/topics/security-devsecops]]
- [[devops/concepts/service-mesh]]
