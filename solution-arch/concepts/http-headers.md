# HTTP Headers

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related:** [[solution-arch/concepts/http-status-codes]], [[solution-arch/concepts/rest-api-design-principles]], [[solution-arch/concepts/caching]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/topics/security-architecture]]

## What it is

Headers carry the metadata that lets generic HTTP infrastructure (browsers, caches, CDNs, proxies, gateways) make correct decisions WITHOUT understanding the request/response body at all. A status code says what happened; headers say how to interpret it, how to cache it, who's allowed to read it, and where it came from. Interviewers use headers to probe whether a candidate understands HTTP as a real protocol with a contract, not just "JSON over a URL."

## How it works

### Caching headers — full depth in [[solution-arch/concepts/caching]]

```
Cache-Control: public, max-age=86400        cacheable anywhere, 1 day
Cache-Control: private, max-age=0            browser-only, don't cache in CDN
Cache-Control: no-store                      never cache (sensitive data)
Cache-Control: stale-while-revalidate=300    serve stale 5min while refetching
```

Not re-derived here — see [[solution-arch/concepts/caching]] for the full directive set and CDN behavior.

### Conditional request headers — how caches revalidate without re-downloading

```
ETag: "abc123"
  Server-generated opaque fingerprint of a resource's current
  content (often a hash). Sent on every response for a cacheable
  resource.

If-None-Match: "abc123"
  Client's follow-up request sends back the ETag it has cached.
  If it still matches current content → server returns 304 Not
  Modified with NO body (see [[solution-arch/concepts/http-status-codes]]).
  If content changed → server returns 200 with the new body and a
  new ETag.

Last-Modified: Wed, 15 Aug 2026 10:00:00 GMT
If-Modified-Since: Wed, 15 Aug 2026 10:00:00 GMT
  A coarser, timestamp-based alternative to ETag/If-None-Match —
  same 304 mechanic, but only second-level precision and doesn't
  catch a change that doesn't update the timestamp. ETag is
  strictly more precise; Last-Modified is simpler to generate
  (no hashing) and still widely supported as a fallback.

Vary: Accept-Encoding, Authorization
  Tells caches which REQUEST headers change what response gets
  served — a cache must store a separate entry per distinct value
  of each header listed. Interview trap: forgetting to include
  Authorization (or an auth-bearing header) in Vary on a
  per-user-personalized response is a real security bug — it lets
  one user's cached response leak to another user via a shared
  cache that doesn't know the response was personalized.
```

### Content negotiation headers

```
Accept: application/json
  Client states what content type(s) it can parse, in preference
  order (can include quality weights: application/json;q=0.9,
  application/xml;q=0.5). Server that can't satisfy any listed
  type returns 406 Not Acceptable.

Content-Type: application/json; charset=utf-8
  What the BODY actually is, on both requests and responses. A
  request Content-Type that doesn't match the actual body format
  is a classic cause of a confusing 400/415 the client can't
  explain without inspecting this header.

Content-Encoding: gzip
Accept-Encoding: gzip, br
  Compression negotiation — client advertises what it can decode
  (Accept-Encoding), server states what it actually used
  (Content-Encoding). Distinct from Content-Type: encoding is a
  transport-level transformation, type is the underlying format.

Content-Length: 1024
  Byte size of the body — required for some servers/proxies to
  know when a non-chunked request/response is complete without
  waiting on connection close.
```

### CORS headers — the preflight mechanics interviewers actually probe

```
Origin: https://app.example.com
  Sent automatically by browsers on cross-origin requests — the
  requesting page's own origin. Not sendable/spoofable by JS itself
  (browser-controlled), which is WHY it's trustworthy as a security
  signal server-side.

Preflight (automatic browser-initiated OPTIONS request, sent BEFORE
the real request, for any "non-simple" cross-origin request — e.g.
one using a custom header, or a method other than GET/HEAD/POST
with a simple content type):

  Browser  ──── OPTIONS /orders ─────────────▶  Server
             Origin: https://app.example.com
             Access-Control-Request-Method: PUT
             Access-Control-Request-Headers: Authorization

  Server   ◀─── 204 No Content ─────────────────
             Access-Control-Allow-Origin: https://app.example.com
             Access-Control-Allow-Methods: GET, PUT, DELETE
             Access-Control-Allow-Headers: Authorization
             Access-Control-Max-Age: 600   (cache preflight 10 min)

  Browser then sends the REAL PUT request only if the preflight
  response permits it.

Access-Control-Allow-Origin: https://app.example.com
  MUST echo a specific allowed origin (or "*" for fully public,
  non-credentialed APIs) — the classic interview trap: "*" cannot
  be combined with Access-Control-Allow-Credentials: true; browsers
  reject that combination outright, because it would let ANY site
  make credentialed requests on a user's behalf.

Access-Control-Allow-Credentials: true
  Required in addition to a specific (non-wildcard) allow-origin
  value if the cross-origin request needs to carry cookies/auth.
```

### Security headers — response headers that constrain what the browser will do

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
  HSTS — tells the browser "always use HTTPS for this domain (and
  subdomains) for the next year, even if the user types http://".
  Defeats SSL-stripping downgrade attacks on subsequent visits;
  useless on the FIRST visit unless the domain is on the browser's
  HSTS preload list.

Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com
  CSP — an allowlist of where the browser may load scripts/styles/
  images/etc. from, enforced by the browser itself. The primary
  browser-side defense against XSS: even if an attacker injects a
  <script> tag, the browser refuses to execute it if the script's
  source isn't on the CSP allowlist.

X-Content-Type-Options: nosniff
  Stops the browser from "MIME-sniffing" (guessing a different
  content type than what Content-Type declares) — prevents a
  crafted file uploaded as, say, an image from being executed as
  script if a browser would otherwise have guessed HTML/JS.

X-Frame-Options: DENY
  Prevents the page from being embedded in an <iframe> on another
  site — the classic clickjacking defense (largely superseded by
  CSP's frame-ancestors directive, but still widely deployed as a
  simpler single-purpose header).

Referrer-Policy: strict-origin-when-cross-origin
  Controls how much of the current page's URL leaks to the NEXT
  site via the Referer header when a user clicks an outbound link
  — relevant when URLs contain sensitive query params (session
  tokens, search terms) that shouldn't leak to third-party sites.
```

### Authentication headers

```
Authorization: Bearer eyJhbGc...
  Carries the credential — a bearer token (JWT, OAuth access
  token), or "Basic base64(user:pass)" for basic auth. "Bearer"
  means literally that: whoever bears/holds the token can use it,
  no further proof of identity required — which is why bearer
  tokens must ONLY travel over TLS and be treated as a secret with
  a short expiry.

WWW-Authenticate: Bearer realm="api", error="invalid_token"
  Sent BACK on a 401 (see [[solution-arch/concepts/http-status-codes]])
  — tells the client HOW it's expected to authenticate and, often,
  WHY the current attempt failed (expired vs missing vs malformed
  token) so the client can react correctly (refresh token vs
  redirect to login).
```

### Proxy / forwarding headers — what a request "really" is behind a proxy chain

```
X-Forwarded-For: 203.0.113.5, 198.51.100.2
  Chain of client IPs as the request passed through proxies —
  leftmost is the ORIGINAL client; each proxy appends its view of
  the previous hop. Trusting this blindly is a spoofing risk: any
  client can SET this header itself unless the edge-most proxy
  strips/overwrites whatever the client sent before appending its
  own trusted value.

X-Forwarded-Proto: https
  What protocol the CLIENT actually used to reach the edge proxy —
  needed because TLS is usually terminated at the load balancer
  (see [[sre/concepts/networking-fundamentals]]'s reverse proxy
  section), so the backend only ever sees plain HTTP and would
  otherwise have no way to know the original request was HTTPS
  (important for generating correct absolute URLs, and for auth
  flows that require "was this request actually secure").

Forwarded: for=203.0.113.5;proto=https;by=203.0.113.43
  The standardized (RFC 7239) single-header replacement for the
  X-Forwarded-* family — less widely adopted in practice than the
  de facto X-Forwarded-* headers, but worth knowing it exists as
  "the header these were all informally standardizing toward."

Via: 1.1 proxy1.example.com, 1.1 proxy2.example.com
  Records which proxies/gateways a request passed through — mainly
  a diagnostic/loop-detection header, not a security control.
```

### Distributed tracing headers

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  W3C Trace Context standard — encodes a trace ID, the current
  span ID, and sampling flags in one header, propagated across
  every service hop so a request can be reassembled into a single
  distributed trace. Format: version-traceID-spanID-flags.

tracestate: congo=t61rcWkgMzE
  Vendor-specific extra tracing state riding alongside traceparent
  (e.g. an APM vendor's internal sampling metadata) — additive,
  doesn't replace traceparent.
```

### Idempotency and rate-limit headers — full depth elsewhere in this wiki

```
Idempotency-Key: 7b3e4f2a-...   → see [[solution-arch/concepts/idempotency]]
Retry-After: 30                  → see [[solution-arch/concepts/http-status-codes]]
                                    (429/503) and [[solution-arch/concepts/rate-limiting]]
RateLimit-Limit / -Remaining / -Reset → see [[solution-arch/concepts/rate-limiting]]
```

## Complexity

Not algorithmic. The cost of getting headers wrong is almost always a silent correctness or security gap rather than a crash: a missing `Vary: Authorization` leaks one user's cached response to another; a wildcard CORS origin combined with credentials gets silently rejected by the browser (fails "safe," but confusingly); a missing `X-Forwarded-Proto` produces subtly wrong absolute URLs in generated links. None of these throw an obvious error — they show up as hard-to-reproduce bugs or security findings.

## When to use

```
✅ Set Vary on ANY response whose content depends on a request
   header, even if you're not sure a shared cache sits in front of
   it today — assume one will eventually
✅ Never combine Access-Control-Allow-Origin: * with credentialed
   requests — pick a specific origin allowlist instead
✅ Set HSTS + CSP on every browser-facing response by default, not
   just after a security review flags their absence
✅ Trust X-Forwarded-* headers only from your OWN edge-most proxy;
   strip/overwrite them at ingress if there's any chance of a
   client setting them directly
```

## Common interview angles

```
Q: "Why does Access-Control-Allow-Origin: * break when credentials
    are involved?"
A: Browsers refuse to expose a credentialed cross-origin response
   to JavaScript if the server's Allow-Origin is a wildcard —
   otherwise ANY website could silently make authenticated requests
   using a logged-in user's cookies/session and read the response.
   The server must echo back a SPECIFIC allowed origin (not "*")
   alongside Access-Control-Allow-Credentials: true.

Q: "Your API caches responses at a shared CDN. Two different users
    are seeing each other's data. What header is most likely
    missing?"
A: Vary: Authorization (or whatever header carries the per-user
   identity) — without it, the CDN treats requests to the same URL
   from different users as cache-equivalent and serves one user's
   cached, personalized response to another.

Q: "A backend service keeps generating http:// links even though
    all traffic actually arrives over HTTPS. Why, and what's the
    fix?"
A: TLS is terminated at the load balancer/reverse proxy, so the
   backend only ever sees plain HTTP — it has no direct way to know
   the original request was HTTPS. Fix: the edge proxy sets
   X-Forwarded-Proto: https, and the backend reads that header
   (not the literal scheme of the connection it received) when
   constructing absolute URLs.

Q: "What's the practical difference between ETag/If-None-Match and
    Last-Modified/If-Modified-Since?"
A: ETag is a content fingerprint — precise, catches ANY change,
   typically requires hashing the content. Last-Modified is a
   timestamp — coarser (second-level precision), can miss a change
   that doesn't bump the timestamp, but cheaper to generate since
   it doesn't require hashing. ETag is preferred when available;
   Last-Modified is the fallback for resources where hashing is
   too expensive.

Q: "Why can't you fully trust X-Forwarded-For for security
    decisions (like IP-based rate limiting)?"
A: It's a header the CLIENT can set directly unless your edge-most
   proxy strips whatever the client sent and appends only its own
   observed value. If any hop in the chain doesn't sanitize it, a
   client can spoof an arbitrary "original" IP. Only trust it when
   you control and can verify the entire proxy chain up to the
   point you're reading it.
```

## Examples

```
A single request/response pair showing several header families at once:

Request:
  GET /orders/456 HTTP/2
  Host: api.example.com
  Authorization: Bearer eyJhbGc...
  Accept: application/json
  Accept-Encoding: gzip, br
  If-None-Match: "abc123"
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

Response (nothing changed since last fetch):
  HTTP/2 304
  ETag: "abc123"
  Cache-Control: private, max-age=60
  Vary: Authorization, Accept-Encoding
```

## Sources
- [[solution-arch/concepts/http-status-codes]]
- [[solution-arch/concepts/caching]]
- [[solution-arch/concepts/rate-limiting]]
- [[solution-arch/concepts/idempotency]]
- [[solution-arch/topics/security-architecture]]
- [[sre/concepts/networking-fundamentals]]
