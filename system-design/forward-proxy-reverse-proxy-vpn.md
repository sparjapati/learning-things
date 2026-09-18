# Forward Proxies, Reverse Proxies and VPNs

> "All problems in computer science can be solved by another level of indirection."
> — David Wheeler
>
> *These are three levels of indirection that look alike in a diagram and answer completely different questions.*

## The distinction in one line

All three sit in the middle of a conversation. What separates them is **whose agent they are** — and, for the VPN, **which layer they operate at**.

```
Forward proxy  →  acts for the CLIENT.  Hides the client from the server.
Reverse proxy  →  acts for the SERVER.  Hides the server from the client.
VPN            →  acts for the NETWORK. Moves your machine onto another network.
```

A forward proxy and a reverse proxy can be **the same software on the same box** — nginx does both. The difference isn't the technology, it's **which side configured it and which side it represents**.

```
FORWARD PROXY — the client knows about it; the server doesn't

  [ Client ] ──► [ Forward proxy ] ──► [ Any server on the internet ]
   configured                            sees the PROXY's IP,
   to use it                             not the client's

REVERSE PROXY — the client doesn't know about it; the server does

  [ Any client ] ──► [ Reverse proxy ] ──► [ Backend servers ]
   thinks it is                            hidden; the client never
   talking to the                          learns they exist
   real server

VPN — not in the middle of one conversation; your machine is now elsewhere

  [ Your machine ] ══encrypted tunnel══► [ VPN gateway ] ──► anything
    ALL IP traffic,                       you now appear to
    every app, every port                 be on that network
```

**Real-life analogy.** A **forward proxy** is a company mailroom: you hand it every outgoing letter, it re-addresses them with the company's return address and posts them (**the server sees the proxy's IP, not yours**), and it can refuse to send some (**egress filtering**). You had to agree to use it (**explicitly configured**). A **reverse proxy** is a company's front desk: the public writes to one published address, and the receptionist decides which of the dozen internal departments actually handles it (**routing to backends**) — callers never learn the internal extensions (**topology hidden**). A **VPN** is different in kind: not a mail service at all, but a **private tunnel connecting your home office to the company building**, so your desk behaves as if it were physically inside (**your machine joins the remote network**) — every kind of traffic goes through it, not just letters (**all IP traffic, not one protocol**).

---

## Forward proxy

**What:** a server that a client is configured to send its outbound requests through.

**Why organisations use one:**

| Purpose | How |
| --- | --- |
| **Egress control** | Allow-list which external hosts employees or servers can reach |
| **Content filtering** | Block categories of site at the network edge |
| **Caching** | One copy of a large download served to many clients |
| **Auditing** | A single log of everything leaving the network |
| **Anonymity / geo** | The destination sees the proxy's IP and location |
| **Package mirrors** | An internal Artifactory/Nexus proxying npm, Maven Central, PyPI |

### Explicit vs transparent

- **Explicit** — the client is told to use it (`HTTP_PROXY`, browser settings, a PAC file). The client knows.
- **Transparent / intercepting** — the network silently redirects port 80/443 to the proxy. The client doesn't know, and nothing was configured on it.

### The HTTPS problem, and `CONNECT`

A forward proxy can read and cache plain HTTP freely. HTTPS is different — it can't read what it can't decrypt. So the client asks for a blind tunnel:

```
CONNECT api.example.com:443 HTTP/1.1

→ proxy opens a TCP connection to that host and relays bytes in both directions
→ TLS is negotiated end-to-end between client and origin, THROUGH the proxy
```

The proxy therefore sees **the hostname** (from the `CONNECT` line, and from TLS SNI) and the byte volume — **but not the content**. That's why corporate filtering can block a domain but not inspect the request body.

Unless, that is, the organisation installs its **own root CA** on every machine. Then the proxy can terminate TLS, decrypt, inspect, and re-encrypt with a certificate it mints itself — a sanctioned man-in-the-middle. This is exactly how `mitmproxy` and Charles Proxy work for debugging, and exactly why corporate laptops trust a certificate you've never heard of.

### Where developers meet it

```bash
export HTTP_PROXY=http://proxy.corp.internal:8080
export HTTPS_PROXY=http://proxy.corp.internal:8080
export NO_PROXY=localhost,127.0.0.1,.internal    # exceptions matter as much as the proxy
```

The JVM has its own settings, which is a recurring source of "curl works but my app doesn't":

```bash
java -Dhttp.proxyHost=proxy.corp.internal -Dhttp.proxyPort=8080 \
     -Dhttps.proxyHost=proxy.corp.internal -Dhttps.proxyPort=8080 \
     -Dhttp.nonProxyHosts='localhost|127.0.0.1|*.internal' -jar app.jar
```

```kotlin
// Per-client proxy configuration, when a JVM-wide system property is too blunt.
private val proxy = Proxy(Proxy.Type.HTTP, InetSocketAddress("proxy.corp.internal", 8080))

val httpClient: HttpClient = HttpClient.newBuilder()
    .proxy(ProxySelector.of(InetSocketAddress("proxy.corp.internal", 8080)))
    .connectTimeout(Duration.ofSeconds(10))
    .build()
```

**The classic incident:** Maven/Gradle/npm hang or fail TLS validation inside a corporate network, because the build tool doesn't read `HTTP_PROXY`, or because the interception CA isn't in the **JVM's** truststore (it's in the OS truststore, which the JVM doesn't use by default).

---

## Reverse proxy

**What:** a server that clients talk to, believing it's the origin, which forwards to backends behind it.

**Why:** this is the workhorse of every production web stack.

| Purpose | Notes |
| --- | --- |
| **TLS termination** | Certificates live in one place — see [reverse-proxy-protocol-termination](reverse-proxy-protocol-termination.md) |
| **Load balancing** | Spread requests over a backend pool — see [load-balancing-algorithms](load-balancing-algorithms.md) |
| **Routing** | `/api` to one service, `/static` to another, by path, host or header |
| **Caching & compression** | Serve cached responses; gzip/brotli once at the edge |
| **Security** | WAF, rate limiting, request-size caps, hiding internal topology |
| **Protocol translation** | HTTP/3 or HTTP/2 outside, HTTP/1.1 inside |

Examples: nginx, HAProxy, Envoy, Traefik, an AWS ALB, a Kubernetes Ingress controller, Cloudflare in front of your origin.

**Reverse proxy vs load balancer:** a reverse proxy is a **position** (in front of servers, acting for them); load balancing is a **function** it usually performs. Nearly every reverse proxy can balance, and nearly every L7 load balancer is a reverse proxy. They're not alternatives.

**The header consequence worth remembering:** once traffic passes through a reverse proxy, your application's "client IP" is the proxy's. The real one arrives in `X-Forwarded-For` / `Forwarded`, and your app must be configured to trust it — but **only from known proxies**, since a header is trivially spoofable by a client. Rate limiting or geo-blocking on an untrusted `X-Forwarded-For` is a real vulnerability.

---

## VPN

**What:** a **Virtual Private Network** creates an encrypted tunnel and a **virtual network interface** on your machine, so your traffic is routed as if you were physically attached to the remote network.

**The distinction that matters: it works at a different layer.**

```
Proxy   →  Layer 7 (usually).  Per-application, per-protocol.
           Only apps configured to use it are affected.

VPN     →  Layer 3 (IP).  Per-machine.
           EVERY app, every port, every protocol — DNS, SSH, database
           connections, game traffic — without any app knowing.
```

That's why "a VPN is just a proxy for everything" is roughly right in effect but wrong in mechanism: a proxy *forwards requests*; a VPN *changes where your packets originate*.

### Types

| Type | Purpose |
| --- | --- |
| **Remote access** | One user's device joins a corporate network from home |
| **Site-to-site** | Two offices' networks permanently joined; users notice nothing |
| **Full tunnel** | *All* traffic goes through the VPN — including your Netflix |
| **Split tunnel** | Only corporate ranges route through; the rest goes direct |

Protocols: **WireGuard** (modern, small, fast, UDP), **IPsec** (the enterprise/site-to-site standard), **OpenVPN** (older, TLS-based, very portable).

### What a VPN does *not* do

Worth stating plainly, because consumer VPN marketing implies otherwise:

- **It does not make you anonymous.** It moves your traffic's visible origin to the VPN provider. You've **relocated trust from your ISP to your VPN provider** — not eliminated it.
- **It does not add encryption you already have.** HTTPS already encrypts content end-to-end. A VPN hides *which sites you visit* from the local network, not the contents from the site.
- **It does not protect against logged-in tracking.** Cookies and accounts identify you regardless of source IP.
- **It is not access control.** Historically a VPN granted *network* access and then everything inside was reachable — which is precisely the flat-network problem Zero Trust exists to fix.

### What replaced it in many organisations

The VPN model — "get onto the network, then you're trusted" — is being displaced by **Zero Trust / identity-aware proxies** (Google BeyondCorp, Cloudflare Access, Tailscale, AWS Verified Access). Each *application* is fronted by a reverse proxy that authenticates **identity and device posture per request**, so there's no implicit network trust and no perimeter to be inside of.

Note what that actually is: **a reverse proxy doing the job a VPN used to do.** The two topics converge.

---

## How reverse proxy, load balancer and rate limiter fit together

These three get listed as if they were peer components choosing between each other. They aren't — they're **a position, a function, and a policy**, and in most stacks all three are the *same process*.

```
Reverse proxy  →  a POSITION.  "I sit in front of the servers and act for them."
Load balancer  →  a FUNCTION.  "I choose which backend gets this request."
Rate limiter   →  a POLICY.    "I decide whether this request is allowed at all."
```

A reverse proxy is the *place*; load balancing and rate limiting are two of the *jobs* done there. Asking "should I use a reverse proxy or a load balancer?" is like asking whether to use a kitchen or cooking.

### Why they converge on one box

Each job needs to happen **before** a request reaches your application, and each needs the same thing: a component that terminates the client connection and can inspect the request. Once something is in that position, giving it all three responsibilities is nearly free — so nginx, Envoy, HAProxy, an AWS ALB and a Kubernetes ingress controller all do the lot.

The order they run in is what matters:

```
                    ┌──────────── EDGE (one process, several jobs) ────────────┐
[ Client ] ──TLS──► │ 1. terminate TLS                                         │
                    │ 2. RATE LIMIT      — reject early, cheapest possible no  │ ──► [ Backend pool ]
                    │ 3. route           — /api here, /static there            │
                    │ 4. LOAD BALANCE    — pick a healthy backend              │
                    └──────────────────────────────────────────────────────────┘
```

**Rate limiting comes before load balancing, and that ordering is the whole point.** A rejected request should cost you a TLS handshake and a `429` — not a backend connection, a thread, a database query and a timeout. Limiting *after* choosing a backend means the overload already happened.

### How they interact in practice

| Interaction | Consequence |
| --- | --- |
| **Rate limiting needs the real client IP** | Which the reverse proxy destroyed by terminating the connection. It must be recovered from `X-Forwarded-For` — and trusted only from known proxies, or the limit is trivially bypassed |
| **Rate limit state must be shared** | Two balanced proxy instances each enforcing "100 req/min" locally means an effective limit of 200. Distributed limits need shared counters (Redis), which is why [rate-limiters](rate-limiters.md) is a harder problem than it looks |
| **Health checks feed load balancing** | Ejecting a failing backend is a load-balancing decision made from health data the proxy collects |
| **Retries interact badly with rate limits** | The proxy retrying a failed request against another backend can multiply load precisely when the fleet is struggling — hence retry budgets |
| **Sticky sessions constrain balancing** | Session affinity (or a WebSocket) pins a client to one backend, overriding whatever algorithm you configured |
| **The limiter protects what the balancer can't** | A balancer spreads load evenly; it has no way to say "this is too much load in total". Only the limiter can refuse |

**The division of labour, in one line each:** the **reverse proxy** decides *whether this request is mine to handle*; the **rate limiter** decides *whether it's allowed right now*; the **load balancer** decides *which of my backends serves it*.

### Where they're separate components

They only split apart at scale or across trust boundaries:

- **An API gateway** (Kong, Apigee, AWS API Gateway) is a reverse proxy whose *primary* job is policy — auth, quotas, per-consumer rate limits, billing — with balancing as a side effect.
- **A CDN** rate limits and terminates TLS thousands of kilometres from your load balancer.
- **A service mesh** moves balancing, retries and limits into a sidecar next to *every* service, so the same three jobs happen at every hop rather than only at the edge.
- **Dedicated L4 in front of L7:** an AWS NLB (pure connection balancing) fronting a fleet of nginx instances (TLS, routing, limits, L7 balancing). Two balancers at two layers, doing different work — see [load-balancing-l4-vs-l7](load-balancing-l4-vs-l7.md).

## Side by side

| | Forward proxy | Reverse proxy | VPN |
| --- | --- | --- | --- |
| **Acts on behalf of** | The client | The server | The whole machine |
| **Who configures it** | The client | The server operator | The client (and the network operator) |
| **Who knows it exists** | The client (server doesn't) | The server (client doesn't) | Both ends, by design |
| **Layer** | 7 (or 4 via `CONNECT`) | 7 (or 4) | 3 (IP) |
| **Scope** | Configured apps only | All traffic to that origin | Every app on the device |
| **Hides** | The client from the server | The servers from the client | Traffic from the local network |
| **Typical owner** | Corporate IT, an ISP | The website operator | Corporate IT, a VPN vendor |
| **Examples** | Squid, corporate proxies, mitmproxy | nginx, Envoy, ALB, Cloudflare | WireGuard, IPsec, OpenVPN |
| **You meet it when** | Your build can't reach Maven Central | Every production request you've ever served | Connecting to a work network |

**The compact version:** *forward proxy protects/controls the client; reverse proxy protects/scales the server; VPN relocates the client.*

---

## How to decide

### Which one does this problem need?

| Question to ask | → Forward proxy | Real-life example | → Reverse proxy | Real-life example | → VPN | Real-life example |
|---|---|---|---|---|---|---|
| Whose behaviour are you controlling? | Outbound clients you own | Restricting which external APIs production servers may call | Inbound traffic to services you own | Terminating TLS and routing `/api` vs `/static` | A whole device's network position | A laptop needing to reach an internal database |
| Does it need to cover *every* protocol, not just HTTP? | No | Web and API traffic | No | HTTP(S) services | **Yes** | SSH, JDBC, SMB, internal DNS all at once |
| Should the far end see the real client IP? | No — that's the point | Egress via one auditable IP | Yes, via `X-Forwarded-For` | Logging and rate limiting per user | Depends on gateway NAT | — |
| Are you hiding clients or hiding servers? | Hiding clients | Corporate egress | Hiding servers | Not publishing backend addresses | Neither — joining networks | Branch-office link |
| Is per-request identity enough, or is network access needed? | — | — | **Per-request identity** — prefer an identity-aware proxy | Internal dashboards behind SSO | Genuine network-level access | Legacy systems with no auth of their own |

### Do we still need a VPN for this?

| Question | → Keep the VPN | Real-life example | → Use an identity-aware reverse proxy | Real-life example |
|---|---|---|---|---|
| Is the target an HTTP application? | No — it's raw TCP/UDP | A database port, SMB share, or legacy protocol | Yes | An internal admin UI or wiki |
| Can the app authenticate users itself? | No, it assumes a trusted network | An old appliance with no SSO | Yes, or the proxy can enforce SSO | Anything behind OIDC |
| Do you need the device *on* the network? | Yes | An engineer debugging network equipment | No — one app at a time is enough | Day-to-day staff access |
| Is lateral movement after compromise a concern? | It's the known weakness | Flat corporate network | **Yes — this is the fix** | Per-app authorisation, no network trust |

---

## Common mistakes

| Mistake | Consequence |
| --- | --- |
| Trusting `X-Forwarded-For` from any source | Trivially spoofed; breaks rate limiting, geo-blocking and audit logs |
| Forgetting `NO_PROXY` for internal hosts | Internal service calls loop out to the proxy and fail or slow down |
| Assuming the JVM reads `HTTP_PROXY` | It doesn't — it needs its own `-Dhttp.proxyHost` properties |
| Interception CA in the OS truststore only | `curl` works, the JVM app fails TLS validation |
| Expecting a forward proxy to inspect HTTPS content | With `CONNECT` it sees only hostname and volume, unless it's a sanctioned MITM |
| Thinking a VPN makes you anonymous | It relocates trust to the VPN provider; logged-in tracking is unaffected |
| Full-tunnel VPN for everything | All personal traffic through the corporate gateway: slow, costly, and a privacy problem |
| Treating "on the VPN" as authorisation | Flat network trust — one compromised laptop reaches everything |
| Confusing reverse proxy with load balancer | They're a position and a function, not competing choices |
| Exposing a backend directly *and* via the reverse proxy | The proxy's TLS, WAF and rate limits are bypassable through the open port |

## Key takeaways

1. **The question is whose agent it is.** Forward proxy → the client's. Reverse proxy → the server's.
2. **Same software, opposite role.** nginx is both; the topology and configuration decide which.
3. **Forward proxies are configured by clients; reverse proxies are invisible to them.**
4. **`CONNECT` means a forward proxy sees hostnames, not content** — unless it MITMs with a trusted corporate CA.
5. **A VPN is not a proxy.** It's Layer 3, per-machine and protocol-agnostic; a proxy is Layer 7 and per-application.
6. **A VPN relocates trust, it doesn't remove it**, and it grants network access rather than application authorisation.
7. **Zero Trust replaces the VPN with an identity-aware reverse proxy** — which is why these three topics converge.
8. **Behind any reverse proxy, client IP handling becomes a security decision**, not a detail.

## See also

- [reverse-proxy-protocol-termination](reverse-proxy-protocol-termination.md) — what "termination" means at the edge, and why HTTP/2 usually stops there.
- [load-balancing-l4-vs-l7](load-balancing-l4-vs-l7.md) — the layer a proxy operates at, and what it can see as a result.
- [load-balancing-algorithms](load-balancing-algorithms.md) — how a reverse proxy chooses which backend gets the request.
- [http-protocol-versions](http-protocol-versions.md) — why the protocol spoken to the client and to the backend often differ.
- [rate-limiters](rate-limiters.md) — commonly enforced at the reverse proxy, and dependent on trustworthy client-IP handling.
