# HTTP Headers: The Common Ones and What They Actually Do

> "The header is the part of the message that talks about the message."
>
> *Almost every cross-cutting concern in a web stack — caching, auth, compression, tracing, CORS, rate limiting, proxying — is implemented as a header, which is why header bugs look like bugs in five unrelated systems.*

A header is `Name: value` metadata attached to a request or response. Names are **case-insensitive** (HTTP/2 and /3 require them lowercase on the wire), values are text, and the same name can repeat.

See also: [http-protocol-versions](http-protocol-versions.md) for how headers are framed and compressed in each HTTP version.

---

## Part 1 — Request headers

### Routing and identity

| Header | What it does |
| --- | --- |
| **`Host`** | Which virtual host you want. **Mandatory in HTTP/1.1** — it's what lets one IP serve a thousand sites. Becomes the `:authority` pseudo-header in HTTP/2 |
| **`User-Agent`** | Client software string. Historically a mess of compatibility lies (`Mozilla/5.0` on everything) |
| **`Referer`** | The page that linked here. **Misspelled in the original spec (RFC 1945) and never fixed** — the response-side policy header `Referrer-Policy` spells it correctly, which is a permanent trap |
| **`Origin`** | Scheme + host + port of the calling page. Sent on CORS requests and all non-GET requests. The basis of CORS and a useful CSRF signal |

### Content negotiation

The client states preferences; the server picks. `q=` values (0–1) express relative preference:

```http
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: en-GB,en;q=0.9,hi;q=0.8
```

| Header | Negotiates |
| --- | --- |
| **`Accept`** | Media type of the response |
| **`Accept-Encoding`** | Compression the client understands (`gzip`, `br`) |
| **`Accept-Language`** | Natural language |

Whatever the server varies on **must** be echoed in the response's `Vary` header, or caches will serve the wrong variant to the wrong client.

### Describing the body

| Header | Meaning |
| --- | --- |
| **`Content-Type`** | The body's media type — `application/json`, `application/x-www-form-urlencoded`, `multipart/form-data; boundary=...`. May carry `; charset=utf-8` |
| **`Content-Length`** | Exact body size in bytes |
| **`Transfer-Encoding: chunked`** | Body sent in self-delimiting chunks when the length isn't known up front |
| **`Expect: 100-continue`** | "Will you accept this 2 GB upload?" — lets the server reject before the body is sent |

> **Security note:** if a request carries **both** `Content-Length` and `Transfer-Encoding`, and a front-end proxy and back-end server disagree about which to honour, you get **HTTP request smuggling** — an attacker prepends bytes to the *next* user's request. This is why proxies should reject requests containing both.

### Authentication and state

| Header | Notes |
| --- | --- |
| **`Authorization`** | `Bearer <jwt>`, `Basic <base64 user:pass>`. Not sent cross-origin by default unless CORS credentials are enabled |
| **`Cookie`** | All cookies matching the domain/path, sent automatically — which is precisely why CSRF exists |
| **`Proxy-Authorization`** | Credentials for the **proxy**, not the origin. Hop-by-hop |

### Caching and conditional requests

The heart of HTTP caching — see [caching-fundamentals](caching-fundamentals.md).

| Header | Asks |
| --- | --- |
| **`If-None-Match: "abc123"`** | "Send it only if the ETag differs" → `304 Not Modified` if unchanged |
| **`If-Modified-Since: <date>`** | Same idea, by timestamp (1-second resolution, so weaker) |
| **`If-Match: "abc123"`** | "Apply this write **only if** the resource is still at this version" → `412 Precondition Failed`. This is **optimistic concurrency control over HTTP** |
| **`If-Unmodified-Since`** | Timestamp equivalent of `If-Match` |
| **`Range: bytes=1000-`** | Partial fetch → `206 Partial Content`. Resumable downloads, video seeking |
| **`Cache-Control: no-cache`** | Request-side directive: revalidate before using a cached copy |

The `If-Match` pattern is worth internalising — it's a lost-update guard that needs no locks:

```http
GET /orders/42          →  200 OK, ETag: "v7"
PUT /orders/42          →  If-Match: "v7"
                           412 Precondition Failed if someone else already wrote "v8"
```

### Connection management (hop-by-hop)

| Header | Meaning |
| --- | --- |
| **`Connection: keep-alive` / `close`** | Whether to reuse the TCP connection |
| **`Connection: Upgrade`** + **`Upgrade: websocket`** | Request a **protocol switch** on this connection — see [websockets](websockets.md) |
| **`Keep-Alive: timeout=5, max=1000`** | Reuse hints |
| **`TE` / `Trailer`** | Transfer-encoding negotiation and trailing headers |

These are **hop-by-hop**: they describe *this one connection*, so a proxy must consume them rather than forward them. **HTTP/2 and /3 forbid them entirely** — connection management moved into the protocol itself.

### Proxy and infrastructure headers

Added by whatever sits in front of your app ([forward-proxy-reverse-proxy-vpn](forward-proxy-reverse-proxy-vpn.md)):

| Header | Carries |
| --- | --- |
| **`X-Forwarded-For`** | Original client IP, then each proxy: `client, proxy1, proxy2` |
| **`X-Forwarded-Proto`** | `https` — how the *client* connected, after TLS was terminated at the edge |
| **`X-Forwarded-Host`** | The `Host` the client originally asked for |
| **`Forwarded`** | RFC 7239's standard replacement: `Forwarded: for=1.2.3.4; proto=https; host=example.com` |

**These are only as trustworthy as the hop that set them.** A client can send any `X-Forwarded-For` it likes; if your app believes it unconditionally, IP-based rate limiting, geo-blocking and audit logs are all forgeable. Trust them **only from known proxy addresses**.

In Spring Boot that's a config decision, not application code:

```properties
# Honour X-Forwarded-* so request.scheme/host/remoteAddr reflect the client, not the proxy.
# Only enable when something trusted is definitely in front.
server.forward-headers-strategy=framework
```

`X-Forwarded-Proto` matters more than it looks: without it, an app behind a TLS-terminating proxy thinks every request is plain HTTP and generates `http://` redirect URLs — producing redirect loops.

### Tracing and correlation

| Header | Purpose |
| --- | --- |
| **`traceparent`** | W3C Trace Context: `00-<trace-id>-<span-id>-<flags>`. The modern standard, understood by OpenTelemetry |
| **`tracestate`** | Vendor-specific trace data |
| **`X-Request-ID`** | Ad-hoc correlation ID, usually minted at the edge and logged everywhere |
| **`Idempotency-Key`** | Client-generated key so a retried request isn't applied twice — see [idempotency](idempotency.md) |

### Fetch metadata (browser-set, useful for defence)

```http
Sec-Fetch-Site: same-origin      # same-origin | same-site | cross-site | none
Sec-Fetch-Mode: cors             # navigate | cors | no-cors | same-origin
Sec-Fetch-Dest: empty            # document | image | script | empty ...
```

Browsers set these and scripts **cannot forge them** (the `Sec-` prefix is protected), which makes them a strong CSRF signal: reject state-changing requests where `Sec-Fetch-Site: cross-site`.

---

## Part 2 — Response headers worth knowing

### Caching

| Header | Meaning |
| --- | --- |
| **`Cache-Control: public, max-age=3600, immutable`** | The authority on cacheability. `no-store` (never cache), `no-cache` (cache but revalidate), `private` (browser only, not CDNs), `s-maxage` (shared caches only) |
| **`ETag: "abc123"`** | Version identifier. `W/"abc"` marks a *weak* tag (semantically but not byte-identical) |
| **`Last-Modified`** | Timestamp counterpart |
| **`Vary: Accept-Encoding, Authorization`** | **Which request headers changed this response.** Get it wrong and a cache serves a gzipped body to a client that can't decompress it, or one user's authenticated page to another |
| **`Age`** | Seconds the response has sat in a cache |

### Content and redirects

| Header | Meaning |
| --- | --- |
| **`Content-Encoding: gzip`** | How the body was compressed |
| **`Content-Disposition: attachment; filename="report.pdf"`** | Download rather than render |
| **`Location`** | Target of a 3xx redirect, or the URL of the resource a `201 Created` just made |
| **`Retry-After: 120`** | Seconds (or an HTTP date) to wait — pair it with `429` and `503` |

### Cookies

```http
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600
```

| Attribute | Effect |
| --- | --- |
| **`HttpOnly`** | JavaScript cannot read it — the main XSS mitigation for session cookies |
| **`Secure`** | HTTPS only |
| **`SameSite`** | `Lax` (default in modern browsers), `Strict`, or `None` (requires `Secure`). The built-in CSRF defence |
| **`Domain` / `Path`** | Scope |
| **`Max-Age` / `Expires`** | Lifetime; omit both for a session cookie |

### CORS

| Header | Meaning |
| --- | --- |
| **`Access-Control-Allow-Origin`** | Which origin may read the response. `*` is **incompatible with credentials** |
| **`Access-Control-Allow-Methods` / `-Headers`** | Answers to a preflight `OPTIONS` |
| **`Access-Control-Allow-Credentials: true`** | Permit cookies/auth on cross-origin requests |
| **`Access-Control-Max-Age`** | How long a preflight result may be cached |

CORS is a **browser** restriction, not server security — `curl` ignores it entirely.

### Security headers

| Header | Protects against |
| --- | --- |
| **`Strict-Transport-Security: max-age=31536000; includeSubDomains`** | Downgrade and SSL-strip attacks — the browser refuses plain HTTP thereafter |
| **`Content-Security-Policy`** | XSS, by restricting where scripts/styles may load from |
| **`X-Content-Type-Options: nosniff`** | Browsers guessing a different content type than you declared |
| **`X-Frame-Options`** / CSP `frame-ancestors` | Clickjacking |
| **`Referrer-Policy`** | Leaking URLs (and their query strings) to third parties |
| **`Permissions-Policy`** | Access to camera, mic, geolocation |

### Rate limiting

```http
RateLimit-Limit: 100
RateLimit-Remaining: 7
RateLimit-Reset: 43
Retry-After: 43
```

The `X-RateLimit-*` spellings are the older de facto convention; the unprefixed ones are the IETF draft standard. See [rate-limiters](rate-limiters.md).

---

## Things that trip people up

**The `X-` prefix is deprecated.** RFC 6648 retired the convention — the problem was that `X-Forwarded-For` and friends became de facto standards and were then stuck with a prefix meaning "experimental". Don't invent new `X-` headers; pick a clear, unprefixed name.

**Hop-by-hop vs end-to-end.** Hop-by-hop headers (`Connection`, `Keep-Alive`, `Transfer-Encoding`, `TE`, `Trailer`, `Upgrade`, `Proxy-Authorization`, `Proxy-Authenticate`) describe a single connection and must be stripped by proxies. Everything else is end-to-end and travels intact.

**HTTP/2 and /3 changed the rules:**
- Header names **must be lowercase**.
- Method, scheme, path and authority became **pseudo-headers** (`:method`, `:scheme`, `:path`, `:authority`) that sort before regular ones.
- Headers are compressed with **HPACK** (H2) or **QPACK** (H3) rather than resent as text.
- Hop-by-hop headers are **forbidden**.

**Header size limits are real.** Servers cap total header size (nginx `large_client_header_buffers`, Tomcat `maxHttpHeaderSize`, commonly 8 KB). Oversized cookies or a giant JWT produce `431` or `400` — a classic "works locally, fails behind the gateway" bug, since each proxy adds more headers.

### Reading them in Spring

```kotlin
@GetMapping("/orders/{id}")
fun getOrder(
    @PathVariable id: String,
    @RequestHeader(HttpHeaders.IF_NONE_MATCH) ifNoneMatch: String?,   // nullable: header may be absent
    @RequestHeader("X-Request-ID") requestId: String?,
): ResponseEntity<Order> {
    val order = orderService.find(id)
    val etag = "\"${order.version}\""

    // Conditional GET: skip serialising the body entirely when the client's copy is current.
    if (etag == ifNoneMatch) return ResponseEntity.status(HttpStatus.NOT_MODIFIED).eTag(etag).build()

    return ResponseEntity.ok()
        .eTag(etag)
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)).cachePrivate())
        .body(order)
}
```

---

## How to decide

### `ETag` or `Last-Modified` for conditional requests?

| Question | → `ETag` | Real-life example | → `Last-Modified` | Real-life example |
|---|---|---|---|---|
| Can the resource change more than once per second? | Yes | A frequently-edited document | No | A nightly-generated report |
| Do you have a cheap version identifier already? | Yes | A JPA `@Version` column or row hash | No, but you have `updatedAt` | A CMS page |
| Do you need optimistic concurrency on writes? | **Yes — `If-Match` is the mechanism** | Preventing lost updates on `PUT` | No | Read-only content |

### `Cookie` or `Authorization` for API auth?

| Question | → Cookie | Real-life example | → `Authorization: Bearer` | Real-life example |
|---|---|---|---|---|
| Is the client a browser on your own domain? | Yes | A server-rendered app or same-site SPA | Not necessarily | A mobile app or service-to-service call |
| Do you want the browser to attach it automatically? | Yes — but that's also what enables CSRF | Session cookies with `SameSite=Lax` | No — explicit is safer cross-origin | A public API |
| Is it cross-origin? | Painful — needs `SameSite=None; Secure` + CORS credentials | A separately-hosted SPA | Straightforward | Any third-party API |

### `X-Forwarded-For` or `Forwarded`?

| Question | → `X-Forwarded-*` | Real-life example | → `Forwarded` | Real-life example |
|---|---|---|---|---|
| What does your existing infrastructure emit? | These — near-universal | ALB, nginx, Cloudflare defaults | Only if you control the whole chain | A greenfield internal mesh |
| Do you need proto and host as well as IP? | Three separate headers | Standard setup | One structured header | Cleaner, but less widely supported |

---

## Common mistakes

| Mistake | Consequence |
| --- | --- |
| Trusting `X-Forwarded-For` from any source | Forgeable client IP: broken rate limiting, geo-blocking and audit trails |
| Forgetting `X-Forwarded-Proto` behind a TLS-terminating proxy | The app generates `http://` redirects → redirect loops |
| Omitting `Vary: Accept-Encoding` | A cache serves a gzipped body to a client that can't decompress it |
| Omitting `Vary` on an auth-dependent response | A shared cache serves one user's page to another — a real data leak |
| Accepting both `Content-Length` and `Transfer-Encoding` | HTTP request smuggling |
| Session cookie without `HttpOnly` | One XSS becomes full session theft |
| `Access-Control-Allow-Origin: *` with credentials | Browsers reject it; attempts to force it are a serious hole |
| Treating CORS as server-side security | It restricts *browsers*; `curl` ignores it |
| Giant JWTs in cookies | `431`/`400` once a gateway adds its own headers |
| Inventing new `X-` headers | Deprecated by RFC 6648 |
| Capitalised header names in HTTP/2 code | Protocol error — must be lowercase |
| Forwarding hop-by-hop headers through a proxy | Breaks connection handling and keep-alive |

## Key takeaways

1. **Headers carry the cross-cutting concerns** — caching, auth, tracing, CORS, compression — which is why one wrong header looks like a bug in several systems at once.
2. **`Host` is what makes virtual hosting work**, and became `:authority` in HTTP/2.
3. **Conditional requests (`ETag` + `If-None-Match`) save bandwidth; `If-Match` gives you optimistic concurrency** with no locking.
4. **`Vary` is the most commonly forgotten header** and its failure mode is serving the wrong user's content.
5. **Proxy headers are hints, not facts** — trust them only from known proxies.
6. **Hop-by-hop headers stop at each proxy** and are banned outright in HTTP/2+.
7. **Cookie attributes are the security story**: `HttpOnly`, `Secure`, `SameSite`.
8. **CORS is a browser policy, not access control.**

## See also

- [http-protocol-versions](http-protocol-versions.md) — how headers are framed, compressed and renamed across HTTP/1.1, 2 and 3.
- [websockets](websockets.md) — where `Connection: Upgrade` and `Upgrade: websocket` actually lead.
- [caching-fundamentals](caching-fundamentals.md) — what `Cache-Control`, `ETag` and `Vary` are driving.
- [idempotency](idempotency.md) — the concern behind `Idempotency-Key`.
- [forward-proxy-reverse-proxy-vpn](forward-proxy-reverse-proxy-vpn.md) — who adds the `X-Forwarded-*` headers and why trusting them blindly is unsafe.
- [rate-limiters](rate-limiters.md) — what `RateLimit-*` and `Retry-After` communicate.
