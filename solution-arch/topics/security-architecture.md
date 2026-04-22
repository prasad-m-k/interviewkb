# Security Architecture

**Related:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/api-gateway]], [[solution-arch/topics/nfr-quality-attributes]]

---

## Zero Trust Architecture

"Never trust, always verify." No implicit trust based on network location.

```
Traditional (Castle-and-Moat):           Zero Trust:
┌─────────────────────────────┐          Every request is verified:
│  Trusted Internal Network   │          ┌──────────────────────────────────┐
│  ┌──────┐ ┌──────┐          │          │  Identity  +  Device  +  Context │
│  │Svc A │ │Svc B │◀─trusted─│          │   (Who?)      (Clean?)  (Normal?)│
│  └──────┘ └──────┘          │          └───────────────┬──────────────────┘
└──────────────┬──────────────┘                          │ ✓ all factors
               │ firewall                                ▼
         Internet                               Grant minimal access
                                                for this specific request
```

**Pillars:**
1. Verify explicitly (every request authenticated + authorised)
2. Least privilege (minimal access, time-limited)
3. Assume breach (segment networks, monitor internally)

---

## Authentication (AuthN) Patterns

### JWT (JSON Web Token)
```
User ──login──▶ Auth Server ──▶ Signs JWT with private key
                                    │
                              JWT = header.payload.signature
                              Payload: { sub, role, exp }

User ──request + JWT──▶ Service
Service verifies signature with public key (no DB call needed)
```

Stateless; scalable. Risk: revocation is hard before expiry.

### OAuth 2.0 / OIDC Flow
```
User ──▶ App ──▶ Auth Server (Google, Auth0, Okta)
                      │
              User consents
                      │
              App gets: access_token (API calls)
                        id_token (user identity, OIDC)
                        refresh_token (get new tokens)
```

### mTLS (Mutual TLS)
Both sides present certificates. Used for service-to-service auth.
```
Service A ──presents cert──▶ Service B
Service A ◀──presents cert── Service B
Both verified; channel encrypted.
```
Service mesh (Istio, Linkerd) handles mTLS automatically via sidecar.

---

## Authorization (AuthZ) Models

| Model | How it works | Best for |
|-------|-------------|---------|
| **RBAC** (Role-Based) | User → Role → Permissions | Simple, most common |
| **ABAC** (Attribute-Based) | Policy: if user.dept == resource.dept | Fine-grained, complex rules |
| **ReBAC** (Relation-Based) | Google Zanzibar: user has relation to object | Social graphs, docs (Google Drive) |
| **OPA** (Open Policy Agent) | Policy as code (Rego language) | Microservices, Kubernetes |

---

## Encryption

```
In Transit:      Client ══TLS 1.3══▶ Server (HTTPS, gRPCS, MQTTS)
At Rest:         Database columns / disk encrypted (AES-256)
In Use:          Homomorphic encryption (emerging), SGX enclaves
Key Management:  HSM / KMS (never store keys next to data)
```

**Key rotation:** rotate encryption keys regularly. Old data re-encrypted on schedule.

---

## Threat Modeling (STRIDE)

| Threat | What | Mitigation |
|--------|------|-----------|
| **S**poofing | Fake identity | AuthN, mTLS |
| **T**ampering | Modify data | Integrity checks, HMAC |
| **R**epudiation | Deny action | Audit logs, signing |
| **I**nformation disclosure | Data leak | Encryption, access control |
| **D**enial of Service | Overwhelm system | Rate limiting, WAF, CDN |
| **E**levation of privilege | Gain higher access | Least privilege, RBAC |

---

## API Security Checklist

```
✅ Use HTTPS everywhere (TLS 1.2 minimum, 1.3 preferred)
✅ Authenticate every request (JWT / API key / OAuth)
✅ Authorise at the resource level, not just the endpoint
✅ Rate limit per client / IP (prevent abuse + DDoS)
✅ Validate and sanitise all input (prevent injection)
✅ Return minimal data (don't expose internals in errors)
✅ Use correlation IDs for tracing (not stack traces in prod)
✅ Rotate secrets / API keys regularly
✅ Log access without logging sensitive data (PII, tokens)
✅ CORS policy: allowlist specific origins
```

---

## Defence in Depth

```
Internet
    │
    ▼
WAF (Web Application Firewall)   ← blocks SQLi, XSS, known attack patterns
    │
    ▼
DDoS Protection / CDN            ← absorbs volumetric attacks
    │
    ▼
API Gateway                      ← AuthN, rate limiting, routing
    │
    ▼
Service Mesh (mTLS)              ← service-to-service security
    │
    ▼
Application                      ← input validation, AuthZ, business logic
    │
    ▼
Database                         ← encryption at rest, minimal DB user privileges
```

Multiple independent layers — compromise of one layer doesn't compromise all.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
