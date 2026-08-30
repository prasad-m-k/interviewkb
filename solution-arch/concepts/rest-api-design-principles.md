# REST API Design Principles

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/patterns/hot-path-first-design]], [[solution-arch/concepts/http-status-codes]], [[solution-arch/concepts/http-headers]]

## What it is

REST (REpresentational State Transfer) is an architectural style for designing networked APIs around **resources** (nouns, not actions) manipulated through a small, uniform set of HTTP verbs. It's not a protocol or a standard — it's a set of constraints that, followed consistently, make an API predictable, cacheable, and evolvable without breaking clients. Most "REST APIs" in production only follow a subset of these constraints (pragmatic REST) — an SA should know the full model to make an informed choice about which constraints to keep and which to relax.

## How it works

### Resources, Not Actions

```
BAD (RPC-style, verbs in the URL):
  POST /getUser?id=123
  POST /createOrder
  POST /cancelOrder?id=456

GOOD (resource-oriented, verb is the HTTP method):
  GET    /users/123
  POST   /orders
  DELETE /orders/456          (or PATCH /orders/456 {status: "cancelled"}
                                if cancellation is a state transition,
                                not a deletion — model it as what it
                                actually is)
```

```
Resource naming conventions:
  Collection:        /orders               (plural noun)
  Specific resource:  /orders/456
  Sub-collection:     /orders/456/items
  Sub-resource:       /orders/456/items/2

  Avoid verbs in paths. When an action doesn't map cleanly to CRUD
  (e.g. "send password reset email"), model it as a sub-resource
  representing the ACTION's result, not the verb itself:
    POST /password-reset-requests     (creates a new reset request)
  rather than:
    POST /users/123/sendPasswordReset
```

### HTTP Methods: Safety and Idempotency Matter Architecturally

```
┌─────────┬────────┬────────────┬──────────────────────────────────┐
│ Method  │ Safe?  │ Idempotent?│ Meaning                             │
├─────────┼────────┼────────────┼──────────────────────────────────┤
│ GET     │ Yes    │ Yes        │ Read a resource; no side effects     │
│ HEAD    │ Yes    │ Yes        │ Like GET, headers only                │
│ PUT     │ No     │ Yes        │ Replace a resource entirely            │
│ DELETE  │ No     │ Yes        │ Remove a resource                       │
│ PATCH   │ No     │ No*        │ Partial update (*can be made idempotent│
│         │        │            │  with a full replacement patch document)│
│ POST    │ No     │ No         │ Create a new resource / non-idempotent  │
│         │        │            │  action                                  │
└─────────┴────────┴────────────┴──────────────────────────────────┘

Safe = doesn't change server state (a load balancer/CDN/browser can
       retry or prefetch it freely)
Idempotent = calling it N times has the SAME EFFECT as calling it once
             (a client can safely retry after a timeout without
             double-applying the operation)
```

**Architectural consequence — this is not a pedantic distinction:** a client that times out on a `POST /orders` genuinely doesn't know if the order was created. Retrying blindly can double-charge the customer. Retrying a `PUT /orders/456` is always safe by definition. This is exactly why idempotency KEYS exist for POST (see below) — they retrofit the idempotency guarantee onto an inherently non-idempotent verb. Full mechanics: [[solution-arch/concepts/idempotency]].

```
Idempotency key pattern for POST:
  POST /orders
  Idempotency-Key: 7b3e4f2a-...   (client-generated UUID, same value
                                    on retry)

  Server: if this key was already processed, return the SAME
  response as the original call (don't re-create the order) —
  requires the server to persist (key → response) for a bounded
  window (e.g. 24h).
```

### HTTP Status Codes: Say What Actually Happened

Full reference with examples for every commonly-asked code (1xx–5xx, including the 3xx redirect-semantics traps and the 401-vs-403/400-vs-422/502-vs-504 distinctions interviewers probe): [[solution-arch/concepts/http-status-codes]].

**Interview trap:** returning `200 OK` with an error message in the body ("success theater") breaks every generic HTTP-aware client (caches, retries, monitoring) that keys behavior off the status code. Status codes are part of the API's real contract, not decoration.

### Richardson Maturity Model — How "RESTful" Is This API, Really?

```
Level 0: The Swamp of POX
  One URI, one HTTP method (usually POST), verbs and everything
  else encoded in the body — this is RPC-over-HTTP, not REST.

Level 1: Resources
  Multiple URIs, one per resource — but still mostly one HTTP verb.
  POST /getUser, POST /updateUser (verb still in the URL/body)

Level 2: HTTP Verbs
  Proper use of GET/POST/PUT/DELETE mapped to resource semantics
  and status codes used correctly. This is where MOST production
  "REST APIs" actually sit — pragmatic REST.

Level 3: HATEOAS (Hypermedia as the Engine of Application State)
  Responses include links describing what the client can do NEXT,
  so the client doesn't need to hardcode URI structure:

    { "orderId": 456, "status": "pending",
      "_links": {
        "self":   { "href": "/orders/456" },
        "cancel": { "href": "/orders/456/cancel" },
        "pay":    { "href": "/orders/456/payment" }
      }
    }

  Rarely fully implemented in practice — the discoverability benefit
  is real (clients can adapt to server-side workflow changes without
  a new client release) but the engineering cost is high and most
  internal/mobile-client APIs don't need it. Know it exists and why
  it's uncommon, rather than pretending every API should reach Level 3.
```

### Versioning Strategies

```
URI versioning:        /v1/orders, /v2/orders
  + Simple, visible, cacheable per version
  - URI is supposed to identify a resource, not a version of an API

Header versioning:     Accept: application/vnd.company.v2+json
  + Keeps the URI as the resource's true identity
  - Less discoverable; harder to test with a browser/curl casually

Query param versioning: /orders?version=2
  + Simple to add
  - Easy to omit accidentally; less explicit than a header/path

Practical default: URI versioning for public/partner APIs (simplicity,
cacheability, and clarity in logs/docs win over purity); consider
never breaking a MAJOR version without a documented deprecation
window — treat a version bump like the zero-drift contract discipline
this KB's PNC context applies to pagination params (nullable additive
fields > breaking changes).
```

### Pagination

```
Offset-based:
  GET /orders?offset=100&limit=20
  + Simple, supports jumping to arbitrary pages
  - Expensive at large offsets (DB still scans/skips prior rows);
    inconsistent if rows are inserted/deleted between page requests
    (a row can shift, causing skip/duplicate)

Cursor-based (keyset pagination):
  GET /orders?after=cursor_abc123&limit=20
  + O(1) regardless of position — uses an indexed WHERE id > :cursor
  + Stable under concurrent inserts/deletes
  - No "jump to page 50" — only forward/backward traversal

Practical default: cursor-based for any collection that's large,
frequently mutated, or infinite-scroll UX; offset-based is fine for
small, mostly-static admin lists where "jump to page N" matters more
than consistency under mutation.
```

### Filtering, Sorting, Field Selection

```
GET /products?category=electronics&sort=-price&fields=id,name,price

  category=...   → filter (equality; range filters: price_min/price_max
                    or a query-language param for complex cases)
  sort=-price     → descending price (- prefix convention); sort=price
                    for ascending
  fields=...      → partial response (sparse fieldsets) — reduces
                    payload size for high-volume, bandwidth-sensitive
                    clients (mobile) — same instinct as truncating
                    LLM context in [[solution-arch/topics/llm-application-architecture]]:
                    don't transfer more than the caller needs
```

### Error Response Format — Standardize It Once, Everywhere

```
{
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "Requested quantity exceeds available stock",
    "details": [
      { "field": "quantity", "issue": "max_available: 3" }
    ],
    "correlation_id": "req-9f3e2a1b"
  }
}
```

`code` is a stable, machine-parseable identifier the CALLER'S code can branch on (never parse `message` text — that's for humans and can change wording without notice). `correlation_id` ties the error back to server-side logs/traces — same discipline as the request tracing in [[solution-arch/topics/microservices]].

## Complexity

Not algorithmic. The architectural cost is contract stability: every relaxation of REST's constraints (skipping HATEOAS, adding RPC-style action endpoints, loose versioning) trades short-term development speed for long-term client coupling — clients hardcode assumptions the API can no longer safely change.

## When to use

```
REST is a strong default for:
  ✅ Public/partner-facing APIs — wide client tooling support (every
     language has an HTTP client), cacheable GETs, well-understood
     status code semantics
  ✅ Resource-oriented domains (CRUD-shaped: users, orders, products)

Consider gRPC/GraphQL instead when:
  ✅ Internal service-to-service calls needing lower latency/binary
     framing and strict typed contracts → gRPC (see
     [[solution-arch/topics/integration-patterns]])
  ✅ Clients need to fetch deeply nested, variably-shaped data in one
     round trip (avoiding REST's over-fetching/under-fetching and
     N+1 problem — see [[solution-arch/patterns/api-composition]]) → GraphQL
```

## Common interview angles

```
Q: "Is your API actually RESTful?"
A: Most production APIs sit at Richardson Maturity Level 2 (proper
   verbs + status codes) and skip Level 3 (HATEOAS) deliberately —
   discoverability benefit rarely justifies the engineering cost for
   internal or mobile-client APIs where the client is versioned and
   deployed alongside the API anyway.

Q: "A client's POST /orders call times out — did the order get
    created or not, and how do you design for that?"
A: The client can't know from a timeout alone. Require an
   Idempotency-Key header on the request; the server persists
   (key → response) so a retry with the same key returns the
   original result instead of creating a duplicate order. See
   [[solution-arch/concepts/idempotency]].

Q: "Why is PATCH not idempotent by default, and when would you want
    it to be?"
A: A PATCH like {"$inc": {"quantity": 1}} applied twice increments
   twice — not idempotent. A PATCH that fully replaces a field's
   value ({"quantity": 5}) IS idempotent, since applying it N times
   yields the same end state. Model the patch document based on
   whether the client's intent is "apply this delta" or "set this
   value."

Q: "How would you paginate a live feed that's being written to
    constantly, without showing duplicates or gaps as users page
    through?"
A: Cursor-based (keyset) pagination anchored to a stable, monotonic
   field (e.g. an auto-incrementing ID or a (timestamp, id) compound
   cursor) — offset-based pagination shifts under concurrent writes,
   causing skipped or duplicated rows across page boundaries.

Q: "How does REST API design connect to the 'hot path first'
    system design methodology?"
A: Endpoint classification (which endpoints are read-heavy/display
   vs write-heavy/decision-critical) is itself partly a REST API
   design decision — GET endpoints are naturally the cacheable,
   horizontally-scalable read path; POST/PUT/PATCH/DELETE are the
   correctness-critical write path. See
   [[solution-arch/patterns/hot-path-first-design]] for the full
   read/write separation methodology this maps onto.
```

## Examples

```
GET    /orders?status=pending&sort=-created_at&limit=20&after=c_abc
POST   /orders                       (Idempotency-Key required)
GET    /orders/456
PATCH  /orders/456                   { "status": "cancelled" }
GET    /orders/456/items
POST   /orders/456/items             { "sku": "ABC123", "qty": 2 }
DELETE /orders/456/items/2
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
