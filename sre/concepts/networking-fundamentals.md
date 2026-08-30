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

## HTTP-Layer Handshake (ALPN, HTTP/2, HTTP/3/QUIC)

The TCP and TLS handshakes above get a connection open and encrypted — but a browser or client still needs to agree with the server on WHICH version of HTTP to speak, and (for HTTP/2+) exchange a few more framing details before real requests flow. This is the piece interviewers probe when they ask "what's the HTTP handshake" specifically, as distinct from TCP/TLS.

### ALPN — negotiating the HTTP version inside the TLS handshake itself

ALPN (Application-Layer Protocol Negotiation) is a TLS extension, not a separate round trip — it rides inside the same ClientHello/ServerHello messages already shown above, at zero extra RTT cost.

```
Client                                  Server
  │──── ClientHello ──────────────────▶│
  │     ALPN extension:                 │
  │     protocols = ["h2", "http/1.1"]  │
  │     (offered in preference order)   │
  │                                     │
  │◀─── ServerHello ────────────────────│
  │     ALPN extension:                 │
  │     protocol = "h2"  (server picks  │
  │     the highest one it supports)    │
  │                                     │
  │══ both sides now know: speak h2 ════│
```

If the server doesn't support HTTP/2, it omits/rejects the ALPN extension and the client falls back to HTTP/1.1 — no extra handshake needed either way. This is why upgrading a service to HTTP/2 is often "just" a server/load-balancer config change: ALPN negotiation means old HTTP/1.1-only clients keep working automatically.

### HTTP/1.1 — no extra handshake, just framing

```
GET /orders/456 HTTP/1.1
Host: api.example.com
Connection: keep-alive

  → plaintext request line + headers + optional body, sent directly
    over the now-open TCP(+TLS) connection. No additional handshake.

Connection: keep-alive keeps the TCP connection open for the NEXT
request too, avoiding a fresh TCP+TLS handshake per request — the
real-world reason browsers open 6-8 parallel persistent connections
per host rather than one connection per request.
```

### HTTP/2 — connection preface + SETTINGS exchange

Once ALPN has negotiated "h2", both sides must complete one more exchange before any request/response frames flow:

```
Client                                  Server
  │──── Connection Preface ───────────▶│  fixed 24-byte magic string:
  │     "PRI * HTTP/2.0\r\n\r\n         │  "PRI * HTTP/2.0\r\n\r\n
  │      SM\r\n\r\n"                    │   SM\r\n\r\n" — confirms
  │                                     │   both sides really mean h2
  │──── SETTINGS frame ───────────────▶│  client's params (max
  │                                     │  concurrent streams, initial
  │                                     │  window size, header table
  │                                     │  size for HPACK, ...)
  │                                     │
  │◀─── SETTINGS frame ─────────────────│  server's params
  │──── SETTINGS ACK ──────────────────▶│
  │◀─── SETTINGS ACK ────────────────────│
  │                                     │
  │══ binary-framed, multiplexed ═══════│  many concurrent streams,
  │   streams begin                     │  HPACK header compression,
  │                                     │  one TCP connection total
```

This SETTINGS exchange is why HTTP/2 needs a fully-established TLS connection first (there's no HTTP/2-over-plaintext in browsers, even though the spec allows "h2c" for internal/non-browser use) — the preface and SETTINGS frames themselves are the "handshake" at this layer, on top of TCP+TLS.

### HTTP/3 (QUIC) — the transport and TLS handshake become ONE

HTTP/3 doesn't run over TCP at all — QUIC runs over UDP and folds the transport-level handshake and the TLS 1.3 handshake into the same flight, instead of TCP's handshake completing first and THEN TLS starting on top of it:

```
TCP + TLS 1.3 (HTTP/1.1 or HTTP/2):        QUIC (HTTP/3):
  1. TCP SYN/SYN-ACK/ACK   (1 RTT)           1. QUIC Initial packet:
  2. TLS 1.3 ClientHello/                       transport params +
     ServerHello+Finished  (1 RTT)              TLS 1.3 ClientHello,
  ────────────────────────                      combined            (1 RTT)
  Total: 2 RTT before data                   2. Server responds with
                                                 transport params +
                                                 TLS ServerHello+
                                                 Finished, combined
                                              ────────────────────────
                                              Total: 1 RTT before data
                                              (0-RTT possible on
                                              resumption — same
                                              replay caveat as TLS
                                              1.3 0-RTT above)
```

This — not just per-stream loss recovery — is the other half of why HTTP/3 wins hardest on high-latency or lossy mobile networks: every RTT saved on connection setup matters most when RTTs are expensive to begin with. See [[solution-arch/concepts/network-architecture-fundamentals]] for the architecture-level "where do we actually deploy HTTP/3 first" decision.

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
3. TLS: ClientHello (with ALPN offering `["h2", "http/1.1"]`) → ServerHello+cert (ALPN picks `h2`) → key exchange → session keys
4. HTTP/2: connection preface + SETTINGS exchange, THEN `GET /` over the encrypted, multiplexed channel
5. Response flows back through the same layers

**Q: "Is there really a separate 'HTTP handshake', or is it just TCP + TLS?"**
There's a real third layer on top of TCP+TLS, and it differs by HTTP version: HTTP/1.1 has none (requests just start flowing after TLS); HTTP/2 requires a connection preface + SETTINGS frame exchange before any request/response frames are valid; HTTP/3 has no separate handshake at all because QUIC merges the transport and TLS 1.3 handshakes into one flight, so by the time the connection is "up," HTTP/3 is already past what TCP+TLS+H2-preface would still be doing across three separate steps.

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
