# HTTP Status Codes

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related:** [[solution-arch/concepts/rest-api-design-principles]], [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/http-headers]], [[solution-arch/concepts/caching]]

## What it is

The status code is the single most load-bearing 3 digits in an HTTP response: every generic HTTP-aware component in the path (browsers, CDNs, load balancers, retry libraries, monitoring/alerting) makes real behavioral decisions based on it — cache or don't, retry or don't, alert or don't. Getting the code wrong (e.g. `200 OK` with an error payload) silently breaks all of that tooling even though the response body might be perfectly readable to a human.

## How it works

### 1xx — Informational (rarely hand-written, but interviewers probe them)

```
100 Continue
  Client sent "Expect: 100-continue" with a large request body; server
  says "go ahead and send the body" before the client wastes bandwidth
  uploading a body the server would reject based on headers alone
  (e.g. a 401 on the Authorization header, or a body too large per
  Content-Length). Saves bandwidth on rejected large uploads.
  Example: a video upload API checking auth/size limits before
  accepting a multi-GB request body.

101 Switching Protocols
  Response to an "Upgrade" request header — used for the WebSocket
  handshake: client sends "Upgrade: websocket", server replies 101
  and the connection becomes a raw WebSocket connection, no longer
  HTTP request/response.
  Example: GET /chat with Upgrade: websocket → 101 → bidirectional
  WebSocket frames from here on.
```

### 2xx — Success (the response body's contract still matters)

```
200 OK
  Generic success. GET returning data, PUT/PATCH returning the
  updated resource.
  Example: GET /users/123 → 200 { "id": 123, "name": "Ada" }

201 Created
  POST successfully created a new resource. MUST include a
  Location header pointing to the new resource's URI.
  Example: POST /orders → 201, Location: /orders/456,
  body: { "orderId": 456, "status": "pending" }

202 Accepted
  Request accepted for ASYNC processing — not yet complete, and the
  response can't promise a final outcome. Common for long-running
  jobs, webhooks, and hot-path-first async writes (see
  [[solution-arch/patterns/hot-path-first-design]]).
  Example: POST /video-transcode-jobs → 202,
  body: { "jobId": "j_789", "status": "queued",
          "statusUrl": "/jobs/j_789" }

203 Non-Authoritative Information
  Response was modified by a transforming proxy (rare in interviews,
  but distinguishes "the origin said 200" from "a proxy in between
  changed something and is telling you so").

204 No Content
  Success, but there is deliberately no response body. Classic for
  DELETE, or a PUT that doesn't need to echo the resource back.
  Example: DELETE /orders/456/items/2 → 204 (empty body)

206 Partial Content
  Response to a Range request — used for resumable downloads and
  video streaming (seeking to a timestamp requests a byte range,
  not the whole file).
  Example: GET /video.mp4 with Range: bytes=1000000-2000000 → 206,
  Content-Range: bytes 1000000-2000000/5000000
```

### 3xx — Redirection (the semantic difference is a real interview trap)

```
301 Moved Permanently
  Resource permanently moved. Clients/search engines SHOULD update
  their stored URL (e.g. bookmarks, SEO link equity transfers to
  the new URL). Cacheable by default.
  Example: GET /old-product-page → 301, Location: /product-page

302 Found (a historically overloaded, ambiguous code)
  Temporary redirect — but browsers HISTORICALLY changed the method
  to GET on a 302 for a POST request, which is NOT what the spec
  intended. This ambiguity is exactly why 303 and 307 were added
  later to be unambiguous.

303 See Other
  Explicitly: "the result of your request is available at a
  different URI, fetch it with GET" — used after a POST to redirect
  to a status/result page without resubmitting the POST body on
  refresh (the Post/Redirect/Get pattern — prevents duplicate form
  submission on browser back/refresh).
  Example: POST /orders → 303, Location: /orders/456
  (browser then does GET /orders/456 to show the confirmation page)

304 Not Modified
  Response to a conditional GET (If-None-Match / If-Modified-Since)
  — tells the client its cached copy is still valid; NO body sent,
  saving bandwidth. See [[solution-arch/concepts/http-headers]] for
  the ETag/If-None-Match mechanics this depends on.
  Example: GET /style.css, If-None-Match: "abc123" → 304 (no body;
  browser uses its cached copy)

307 Temporary Redirect
  Like 302 but UNAMBIGUOUS: the method and body MUST be preserved
  on the redirected request (a POST stays a POST). Use this, not
  302, when redirecting a non-GET request temporarily.

308 Permanent Redirect
  Like 301 but with the same "preserve method and body" guarantee
  as 307. The modern, unambiguous replacement for 301 when the
  original request wasn't a GET.
```

### 4xx — Client Error (the client should NOT blindly retry as-is)

```
400 Bad Request
  Malformed request the server can't even parse/validate — bad
  JSON, missing required field, wrong data type.
  Example: POST /orders { "quantity": "two" } → 400,
  { "error": { "code": "INVALID_TYPE", "field": "quantity" } }

401 Unauthorized  (really means UN-AUTHENTICATED — a famous naming
                    mistake baked permanently into the spec)
  No valid credentials presented, or they've expired. MUST include
  a WWW-Authenticate header telling the client HOW to authenticate.
  Example: GET /orders/456 with no/expired token → 401,
  WWW-Authenticate: Bearer realm="api", error="invalid_token"

403 Forbidden
  Credentials WERE valid, but the authenticated identity doesn't
  have permission for this resource/action. The key interview
  distinction from 401: 401 = "who are you?", 403 = "I know who you
  are, and the answer is no."
  Example: authenticated as a normal user, GET /admin/users → 403

404 Not Found
  Resource doesn't exist at this URI. Also sometimes deliberately
  used INSTEAD of 403 to avoid leaking that a resource exists to an
  unauthorized caller (a private security-through-obscurity choice,
  worth naming explicitly if asked "401 vs 403 vs 404 for a
  resource the caller isn't allowed to know exists").
  Example: GET /orders/999999 (doesn't exist) → 404

405 Method Not Allowed
  The URI exists, but not for this HTTP method. MUST include an
  Allow header listing the methods that ARE supported.
  Example: DELETE /orders (collection, not deletable) → 405,
  Allow: GET, POST

406 Not Acceptable
  Server can't produce a response matching the client's Accept
  header (content negotiation failure).
  Example: GET /report with Accept: application/xml, but the
  server only produces JSON → 406

408 Request Timeout
  Server gave up waiting for the client to send a complete request
  (slow/stalled upload) — distinct from 504, which is the SERVER
  timing out waiting on an UPSTREAM dependency.

409 Conflict
  The request conflicts with the current state of the resource —
  classic case: optimistic concurrency version mismatch (two
  clients tried to update the same resource; the second one's
  version token is stale).
  Example: PUT /orders/456 with If-Match: "v3" but current version
  is "v4" → 409, { "error": { "code": "VERSION_CONFLICT" } }

410 Gone
  Like 404, but STRONGER: the resource used to exist and is
  PERMANENTLY removed (not "maybe it moved," not "maybe it never
  existed") — lets caches/crawlers purge it confidently rather than
  re-checking.

411 Length Required
  Server requires a Content-Length header and the client didn't
  send one (common when a client tries chunked transfer against a
  server that doesn't support it for that endpoint).

413 Payload Too Large
  Request body exceeds a size limit — e.g. an upload endpoint
  rejecting a file above its configured max.

415 Unsupported Media Type
  The Content-Type of the request body isn't one the server can
  process (e.g. sending XML to a JSON-only endpoint).
  Example: POST /orders, Content-Type: application/xml → 415

422 Unprocessable Entity
  Well-FORMED (valid JSON, right types) but semantically invalid —
  the key distinction from 400. A 400 is "I can't even parse this";
  a 422 is "I parsed it fine, but the business rules reject it."
  Example: POST /orders { "quantity": -5 } → 422 (valid JSON/type,
  but a negative quantity violates a business rule)

429 Too Many Requests
  Rate limited. SHOULD include Retry-After (and often
  RateLimit-* headers). See [[solution-arch/concepts/rate-limiting]]
  and [[solution-arch/concepts/http-headers]] for the full header
  contract.
  Example: 429, Retry-After: 30, RateLimit-Remaining: 0
```

### 5xx — Server Error (the client MAY retry, ideally with backoff)

```
500 Internal Server Error
  Generic, unexpected failure — the catch-all when nothing more
  specific applies. A well-designed API minimizes RAW 500s in favor
  of more specific codes wherever the failure mode is anticipated.

501 Not Implemented
  Server doesn't support the functionality needed to fulfill the
  request (e.g. an HTTP method the server has never implemented,
  distinct from 405 which means the method exists elsewhere on the
  API but not this specific resource).

502 Bad Gateway
  A gateway/proxy/load balancer got an INVALID response from an
  upstream server it was trying to fulfill the request through —
  the upstream is reachable but misbehaving (crashed mid-response,
  returned garbage).
  Example: nginx reverse-proxying to a backend that just crashed
  mid-request → 502 from nginx, not 500 from the backend (the
  backend never even sent a valid response)

503 Service Unavailable
  Server is temporarily unable to handle the request — overloaded,
  in maintenance, or a circuit breaker (see
  [[solution-arch/patterns/circuit-breaker]]) has tripped open.
  SHOULD include Retry-After if the downtime duration is known.
  Example: 503, Retry-After: 120 during a planned maintenance window

504 Gateway Timeout
  A gateway/proxy/load balancer's upstream dependency didn't
  respond IN TIME — the upstream may still be alive, just too slow.
  The key distinction from 502: 502 = upstream responded badly;
  504 = upstream didn't respond at all within the deadline.

507 Insufficient Storage
  Server can't store the representation needed to complete the
  request (out of disk space) — rare outside of WebDAV/file-storage
  APIs, but a good "do you know what's beyond the common ones"
  signal if it comes up.
```

## Complexity

Not algorithmic. The real cost of getting this wrong is systemic: every retry policy, cache layer, alerting rule, and client SDK in the ecosystem is written assuming standard HTTP semantics. A custom convention (always returning 200, encoding the "real" status in a body field) forces every one of those components to be reimplemented or bypassed.

## When to use

```
✅ Return the MOST SPECIFIC code that's accurate — 422 over 400 when
   the request parsed fine but violated a business rule; 409 over
   400 for a concurrency conflict; 403 over 404 unless deliberately
   hiding resource existence
✅ Always pair 3xx/429/503 with the headers that make them
   actionable (Location, Retry-After) — a status code without the
   header that explains what to do next is only half the contract
❌ Never invent a "success wrapper" that returns 200 with an
   { "success": false } body — see the "success theater" trap below
```

## Common interview angles

```
Q: "What's the difference between 401 and 403?"
A: 401 = the server doesn't know who you are (missing/invalid/
   expired credentials) — MUST include WWW-Authenticate. 403 = the
   server knows exactly who you are, and that identity is not
   permitted. Conflating them is a common junior mistake interviewers
   listen for.

Q: "What's the difference between 400 and 422?"
A: 400 = malformed at the SYNTAX level (invalid JSON, wrong type).
   422 = well-formed but invalid at the BUSINESS RULE level (valid
   JSON, correct types, but violates a domain constraint like a
   negative quantity or an already-cancelled order being modified).

Q: "What's the difference between 502 and 504 from a load balancer?"
A: 502 = the upstream responded, but the response was invalid or
   the connection was reset mid-response — the upstream is reachable
   but broken. 504 = the upstream never responded within the
   gateway's timeout — could be alive-but-slow, or fully hung; the
   gateway genuinely doesn't know which.

Q: "Why is returning 200 OK with an error in the body ('success
    theater') a real architectural problem, not just a style
    nitpick?"
A: Every generic HTTP-aware component in the path — CDN/cache
   layers, retry libraries, uptime monitors, API gateways — makes
   real decisions keyed off the status code, not the body. A CDN
   will happily CACHE a 200 response even if the body says
   "error: insufficient inventory"; a retry library won't retry a
   200 even if it represents a transient failure that should be
   retried. The status code IS the contract; the body is
   supplementary.

Q: "A client got a 409 on PUT /orders/456. What should it do?"
A: NOT blindly retry — a 409 means the request conflicts with
   current state (likely a stale version in an optimistic-
   concurrency check). The client should re-fetch the current
   resource state, reconcile/re-apply its intended change against
   the LATEST version, and resubmit — not resend the identical
   request, which would conflict again.

Q: "Why does the spec define both 307/308 when 302/301 already
    exist?"
A: 301/302 have an ambiguous, historically inconsistent rule about
   whether the method and body are preserved on redirect (browsers
   have historically downgraded a redirected POST to a GET on 302).
   307/308 exist purely to remove that ambiguity: they GUARANTEE the
   method and body are preserved. Use 307/308 whenever redirecting a
   non-GET request.
```

## Examples

```
A single resource's lifecycle through status codes:

POST   /orders             { "sku": "ABC", "qty": 2 }
       → 201 Created, Location: /orders/456

GET    /orders/456
       → 200 OK, { "id": 456, "status": "pending", "version": 1 }

PUT    /orders/456         If-Match: "v1"  { "status": "shipped" }
       → 200 OK   (or 409 Conflict if another client updated it
                    first and the If-Match version no longer matches)

GET    /orders/456         If-None-Match: "v2"
       → 304 Not Modified  (client's cached copy is still current)

DELETE /orders/456
       → 204 No Content

GET    /orders/456
       → 404 Not Found     (or 410 Gone, if the API distinguishes
                             "never existed" from "deleted on purpose")
```

## Sources
- [[solution-arch/concepts/rest-api-design-principles]]
- [[solution-arch/concepts/http-headers]]
- [[solution-arch/concepts/idempotency]]
- [[solution-arch/concepts/rate-limiting]]
- [[solution-arch/patterns/circuit-breaker]]
