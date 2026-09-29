# Authentication Mechanisms and How They Evolved

> "Every authentication mechanism is an answer to one question — *how do I prove who I am without handing over something that can be stolen and reused?*"
>
> *The history of auth is the history of shrinking what a thief gets when they intercept the proof: from the password itself, to a session ID, to a short-lived token, to a signature that is worthless anywhere but the one site it was made for.*

See also: [passkeys](../general/passkeys.md) and [totp](../general/totp.md) for deep dives on two of the mechanisms at the end of this story.

## The one thing they all do

Authentication (**authN**) answers *"who are you?"*. Authorization (**authZ**) answers *"what are you allowed to do?"*. They're constantly confused because the same protocol often carries both — OAuth 2.0 is genuinely an *authorization* protocol that the industry spent a decade misusing for login, which is exactly why OIDC had to be invented (below).

Every mechanism in this note is some variation on presenting a **credential** — and the whole evolution is driven by three weaknesses of whatever credential was current at the time:

| Weakness | The question it raises | What it eventually forced |
| --- | --- | --- |
| **Replayable** — whoever intercepts it can reuse it | Is the proof tied to *this* request? | Digest auth, HMAC signing, DPoP |
| **Shared** — the server stores the same secret you hold | Does a server breach hand over every account? | Asymmetric keys: passkeys, mTLS, signed JWTs |
| **Unbound** — it works anywhere it's typed | Does the proof know *which site* it was made for? | WebAuthn origin binding (phishing resistance) |

Keep those three columns in mind; each step below fixes one and usually exposes the next.

![Authentication mechanisms grouped into five eras — password (Basic, Digest), session (form login plus session store), token (opaque, JWT), delegation (OAuth 2.0, OIDC) and phishing-resistant (second factor, passkeys) — with every arrow labelled by the weakness that forced the next step](images/auth-mechanisms-evolution-ladder-eraser.png)

---

# Track A — how humans log in

## 1. HTTP Basic auth (1996)

The browser sends `Authorization: Basic base64(user:password)` on **every single request**.

```http
GET /account HTTP/1.1
Host: example.com
Authorization: Basic YWxpY2U6aHVudGVyMg==
```

Base64 is *encoding*, not encryption — it's the password in plaintext for anyone on the wire. And because it's re-sent constantly, there are hundreds of chances per session to intercept it.

**Analogy:** shouting your house key's cut pattern (**the password**) to a doorman at every doorway in the building (**every request**), while everyone in the corridor (**anyone on the network path**) listens. Even if this doorman is honest, you've now said it a hundred times.

**What it still gets used for:** internal tools and machine endpoints behind TLS, where simplicity beats everything.

## 2. Digest auth (1997) — the first "don't send the secret" idea

Instead of sending the password, the client sends `MD5(password + server_nonce + request_details)`. The server, which also knows the password, computes the same thing and compares.

This fixed **replay** (the nonce ties the proof to one challenge) but not **sharing** — the server still stores the password to be able to verify — and MD5 is long dead. It's historically important as the first mainstream attempt at *proving knowledge of a secret without transmitting it*, an idea that resurfaces in HMAC signing and SCRAM.

## 3. Form login + server-side sessions (mid-1990s onward)

The dominant shape for two decades, and still the right default for a classic web app:

1. You POST your password **once**.
2. The server verifies it, creates a **session record** in memory/Redis/DB, and returns a random session ID in a cookie.
3. Every later request carries only that cookie; the server looks the ID up to find who you are.

**Analogy:** a cloakroom. You hand over your coat once (**the password, presented a single time**), and get a numbered ticket (**the session ID cookie**). The ticket means nothing by itself — all the real information sits in the cloakroom's own ledger (**the server-side session store**). Lose the ticket and a thief can collect your coat, but the attendant can also strike your number out of the ledger instantly (**server-side revocation**), and the ticket tells a pickpocket nothing about you (**opaque identifier, no data inside**).

What this bought: the password is exposed once instead of constantly, the credential is revocable server-side, and it can expire independently.

What it then broke, as apps grew:

- **Scaling** — the session lives on one server, so you need sticky sessions or a shared store (Redis). State in the request path is exactly what horizontal scaling hates.
- **CSRF** — the browser attaches the cookie *automatically*, even on a request forged by another site. This entire attack class exists only because cookies are ambient. (Mitigations: `SameSite` cookies, CSRF tokens.)
- **Non-browser clients** — mobile apps and SPAs calling multiple backends don't fit a cookie scoped to one domain.
- **Third parties** — no way to let another company act on your behalf without giving them your password.

Those last two are what the next two sections solve.

## 4. Token-based auth — opaque tokens, then JWTs (2010s)

Move the credential out of the cookie jar and into an explicit header the client controls:

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

Two very different things hide behind "token":

| | **Opaque token** | **JWT (self-contained)** |
| --- | --- | --- |
| What it is | A random string | Base64url `header.payload.signature` |
| To validate | Look it up in the issuer's store | Verify the signature locally |
| Revocation | Immediate — delete the row | Hard — it's valid until it expires |
| Network cost | A lookup per request (or introspection call) | None |

A **JWT** ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) carries its claims in the payload and is *signed*, so any service holding the issuer's public key can verify it without calling anyone.

**Analogy:** a festival wristband with your name and tier printed on it and a tamper-evident seal (**the signature**). Any bartender can read it and check the seal is intact (**local verification with the public key — no call to the box office**), which is exactly what makes the gates fast. But nothing on that wristband can be un-printed: if you're thrown out, every bar keeps honouring it until it physically expires (**no revocation without adding back the state you removed**). Note the seal only proves *the festival issued it*, not that it's *yours* — that's the bearer problem below.

This is the trade the whole industry argues about: you removed the session-store lookup, and paid for it with **no revocation**. The standard patch is a **short-lived access token** (5–15 min, self-contained) plus a **long-lived refresh token** (opaque, stored server-side, revocable) — which quietly reintroduces the session store, but only on the refresh path instead of every request.

```kotlin
// Session model: every request costs a lookup, but logout is instant.
fun authenticate(sessionId: String): User? =
    sessionStore.find(sessionId)?.takeIf { it.expiresAt > now() }?.user

// JWT model: no lookup at all — and no way to say "this one is cancelled".
fun authenticate(token: String): User? =
    Jwts.parser().verifyWith(issuerPublicKey).build()
        .parseSignedClaims(token).payload
        .let { User(id = it.subject, roles = it["roles"] as List<String>) }
```

Two things people get wrong here, both worth knowing cold:

- **A JWT is not encrypted.** Anyone can read the payload; the signature only proves it wasn't *altered*. Never put anything secret in it.
- **`alg: none` and algorithm confusion.** Early libraries trusted the token's own header to say how to verify it — so an attacker could set `alg` to `none`, or swap `RS256` for `HS256` and sign with the public key as the HMAC secret. Always pin the expected algorithm server-side rather than reading it from the token.

## 5. Delegation: OAuth 1.0 → OAuth 2.0 (2007 → 2012)

The problem OAuth exists for is **not login**. It's: *how does a photo-printing site fetch your Flickr photos without you giving it your Flickr password?* The pre-OAuth answer was literally "type your password into the third-party site" — the **password anti-pattern**, which gives away everything permanently.

OAuth 2.0 ([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)) splits the parties: the **resource owner** (you), the **client** (the printing site), the **authorization server** (Flickr's login), and the **resource server** (Flickr's API). You authenticate to the authorization server directly — the client never sees your password — and it gets back a **scoped, expiring access token**.

**Analogy:** the valet key. You hand the parking attendant (**the third-party client**) a key that starts the engine and opens the door but won't open the boot or the glovebox (**scopes**), and stops working at the end of the day (**token expiry**). Your real key (**your password**) never leaves your pocket, and you can tell the dealer to void the valet key without re-cutting your own (**revocation without a password reset**).

Evolution inside OAuth itself is worth knowing, because interviews probe it:

- **OAuth 1.0** required every request to be cryptographically signed by the client. Secure without TLS, but painful to implement correctly. OAuth 2.0 dropped signing and leaned on TLS instead — a deliberate simplicity-for-ubiquity trade.
- **The implicit grant** returned the token straight in the URL fragment for browser apps that couldn't keep a secret. It leaks tokens into browser history, referrers and logs, and is now **discouraged by the OAuth Security BCP**.
- **PKCE** ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)) replaced it: the client invents a random `code_verifier`, sends only its hash up front, and must present the original when redeeming the code — so an intercepted authorization code is useless to anyone else. Originally for mobile, now recommended for *all* clients including server-side ones.
- **The password grant** (client collects your username/password and exchanges it) re-created the exact anti-pattern OAuth was invented to kill, and is removed in OAuth 2.1.

The surviving default is **authorization code + PKCE**.

## 6. OIDC (2014) — putting identity back on top of OAuth

Because OAuth returns an access token and nothing about *who you are*, everyone bolted login onto it by calling some `/me` endpoint with the token — which is insecure (an access token from a different app can be replayed at yours: the "confused deputy" problem) and non-standard.

**OpenID Connect** is a thin identity layer on OAuth 2.0 that adds an **ID token**: a JWT *about the user*, with an audience claim naming the client it was minted for, so it can't be reused elsewhere.

**Analogy:** the access token is a **visa** — it says what you may do at this border. The ID token is a **passport** — it says who you are, issued by an authority both sides trust, and stamped with the specific country that requested it (**the `aud` claim**). OAuth alone hands out visas; people kept trying to infer identity from the visa, and OIDC finally issued the passport.

This is what's behind every "Sign in with Google/Apple/Microsoft" button.

![Sequence diagram of the OAuth 2.0 authorization code flow with PKCE: the client generates a code_verifier and its challenge, the user authenticates and consents at the authorization server, the returned code is redeemed together with the verifier (a stolen code without it gets invalid_grant), and the client validates the OIDC ID token before calling the resource server](images/auth-oauth2-pkce-oidc-flow-eraser.png)

Editable version (Eraser.io): [OAuth 2.0 + PKCE with OIDC ID token](https://app.eraser.io/workspace/JLgRjFjapzOnrAqixpQO?diagram=NKwpa0qKQF1UZL3i2AN9&layout=canvas).

## 7. Enterprise SSO: Kerberos → SAML (1988 → 2005)

Running in parallel, and *older* than most of the above, is the enterprise lineage:

- **Kerberos** (MIT, 1980s) introduced the ticket-granting model — authenticate once to a central authority, receive time-limited tickets for individual services. Still the engine inside Windows/Active Directory domain login.
- **SAML 2.0** (2005) carried the same idea to the browser with XML assertions passed between an **Identity Provider** (your company's Okta/AD FS) and a **Service Provider** (Salesforce, Workday). One corporate login, many SaaS apps.

SAML and OIDC solve the *same* problem with different eras' technology: XML + browser POST bindings versus JSON/JWT + REST. SAML predates OIDC by roughly a decade, which is why it's entrenched in large enterprises and why so much of "B2B SSO" work is still SAML integration.

## 8. Second factors: SMS OTP → TOTP → push → phishing-resistant

Every mechanism so far shares one flaw: the credential is **something you know**, so a phisher who gets you to type it wins. Multi-factor adds **something you have**.

| Factor | How it broke |
| --- | --- |
| **SMS OTP** | SIM-swap, SS7 interception, carrier social engineering. Widely deprecated for high-value accounts |
| **TOTP** ([totp](../general/totp.md)) | No carrier involved and works offline, but the code is still a string you can be tricked into typing — real-time relay phishing (Evilginx) defeats it |
| **Push approval** | Removed typing, introduced **MFA fatigue**: spam the prompt at 3am until someone taps "approve". Patched with number matching |
| **Hardware/WebAuthn** | The first factor that is *origin-bound* — see below |

## 9. Passwordless and phishing-resistance: U2F → WebAuthn → passkeys (2014 → now)

**Passkeys** ([passkeys](../general/passkeys.md)) are the end state of all three weaknesses at once:

- **Not shared** — your device keeps the private key; the server stores only a public key, so a breach of the server yields nothing usable.
- **Not replayable** — each login signs a fresh server challenge.
- **Not unbound** — the browser includes the **actual origin** in what gets signed, so a signature produced at `examp1e.com` cannot be presented at `example.com`. This is the key property: it doesn't *educate* the user out of phishing, it makes the phished artifact worthless.

**Analogy:** a lock cut for exactly one building's door (**the origin binding**), with a key that lives in a sealed keychain that only opens for your fingerprint (**the device's secure element + biometric unlock**). A fake lookalike door next door physically won't accept the key — the user can fall for the fake and still lose nothing, which no amount of "check the URL" training ever achieved.

Also in this lane: **magic links** (email a one-time login URL) — genuinely passwordless, but they inherit whatever security the mailbox has, and they're phishable.

## 10. Where it's heading

- **Sender-constrained tokens** — **DPoP** ([RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)) and mTLS-bound tokens make a stolen token useless without the matching private key, killing the "bearer" weakness that JWTs inherited from cookies.
- **Device-bound sessions** — the same idea applied to browser session cookies, so malware can't just copy a cookie off disk and resume your session elsewhere.
- **Continuous / risk-based auth** — instead of one decision at login, re-evaluate on signal changes (new device, impossible travel, privilege escalation). "Zero trust" in practice.

---

# Track B — how services authenticate to each other

Same three weaknesses, different constraints: no human, no browser, no phishing — but secrets now sit in config files, CI logs and container images, and they live for years.

## 1. Static API keys

One long random string per caller, sent in a header.

**Analogy:** a master key photocopied for every contractor who ever worked on the building (**the same secret pasted into every service, CI job and `.env`**). It works, nobody knows who actually holds a copy, and if one leaks you have to re-cut every lock (**rotation means coordinated redeployment everywhere**).

Weaknesses: replayable if intercepted, shared (the server stores something verifiable), long-lived, and usually all-or-nothing in scope. Still ubiquitous for low-risk public APIs because it's trivial.

## 2. HMAC request signing (AWS SigV4 style)

The caller signs a canonical form of the request — method, path, headers, body hash, timestamp — with a shared secret, and sends the signature rather than the secret.

```kotlin
val canonical = "$method\n$path\n$timestamp\n${sha256(body)}"
val signature = hmacSha256(key = secret, data = canonical).toHex()
// Header: Authorization: HMAC-SHA256 Credential=<id>, Signature=<signature>
```

This fixes **replay** — the signature covers a timestamp and the body, so it's valid for one request within a few minutes, and altering a single byte of the payload invalidates it. The secret itself never crosses the wire. It's still symmetric (both sides hold the secret), and implementing canonicalization identically on both sides is famously fiddly.

## 3. OAuth 2.0 client credentials / service tokens

The machine equivalent of the human flow: a service presents its client ID + secret to an authorization server and receives a short-lived, **scoped** JWT.

```http
POST /oauth2/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=billing-service&client_secret=s3cr3t&scope=orders.read
```

The gain over an API key: the long-lived secret is used only at one endpoint, and everything downstream sees a token that expires in minutes and names exactly what it may do. The remaining problem is the bootstrap — you still have a static client secret sitting somewhere.

## 4. mTLS and workload identity

In mutual TLS both sides present certificates, so the *transport itself* authenticates the caller — no application-level credential at all.

**Analogy:** ordinary TLS is a shop showing you its trading licence while you stay anonymous (**server-only certificate**); mTLS is both parties showing ID before the conversation starts (**client certificate too**), with a shared registrar vouching for both (**the CA**). Nothing secret is ever handed over — each side only proves it holds a private key matching a certificate the registrar signed.

**SPIFFE/SPIRE** standardized this as **workload identity**: a workload gets a short-lived cryptographic identity document (SVID) issued automatically from what it *is* (which cluster, namespace, service account), not from a secret someone pasted into config. A service mesh (Istio, Linkerd) typically does the certificate rotation for you as sidecar plumbing.

## 5. Federated, zero-static-secret credentials

The current best practice: **no long-lived secret exists anywhere**. The workload proves its identity with a platform-signed OIDC token and exchanges it for short-lived cloud credentials — GitHub Actions → AWS/GCP without a stored access key, or Kubernetes service account tokens → cloud IAM roles.

This closes the loop: it's the same authorization-code-for-token exchange from Track A, with the platform standing in for the human.

---

## The two decisions people actually argue about

### Server-side sessions vs stateless JWTs

| Question to ask | → Sessions | Real-life example (sessions) | → JWTs | Real-life example (JWTs) |
| --- | --- | --- | --- | --- |
| **Must "log out everywhere" be instant?** | Yes — you can delete the record | A bank's net-banking portal: a fraud report has to kill every live session *now*, not in 15 minutes | Can tolerate a short window | A news site's reading session — nothing catastrophic happens if a token stays valid 10 more minutes |
| **Who validates the credential?** | One app (or a few) sharing a store | A Spring Boot monolith with Spring Session + Redis — the lookup is one hop on the same network | Many independent services, possibly across teams | A microservice mesh where the payments service must validate a caller without calling the auth service on every request |
| **Is it a browser-only app?** | Yes — cookies are free and `HttpOnly` protects against XSS theft | A server-rendered admin dashboard, no mobile client | No — mobile/SPA/CLI/third-party | A mobile banking app and a partner API consuming the same backend |
| **What's the per-request latency budget?** | A store lookup is acceptable | Internal HR tool at a few hundred RPS | Must avoid a network hop per request | An ad-serving API at 50k RPS where an extra Redis round trip per call is a real cost |
| **How large is the claim set?** | Any size — it's server-side | A user with 200 fine-grained permissions; none of it travels | Small and stable | `sub`, `roles`, `tenant` — a few hundred bytes on every request header |

The honest answer for most products is **both**: an opaque, revocable refresh credential held server-side, and short-lived self-contained access tokens for the request path.

### OIDC vs SAML for SSO

| Question to ask | → OIDC | Real-life example (OIDC) | → SAML | Real-life example (SAML) |
| --- | --- | --- | --- | --- |
| **Who are the users?** | Consumers, or modern workforce IdPs | A consumer app offering "Sign in with Google" | Enterprise workforce behind an existing IdP | Selling to a bank that runs AD FS and will not add a new protocol for you |
| **What are the clients?** | Mobile apps, SPAs, APIs | A React SPA + iOS app sharing one identity provider | Browser-based web apps only | An internal HR SaaS accessed only from a corporate laptop browser |
| **What does the buyer's security team mandate?** | No constraint | A startup choosing freely — Auth0/Cognito/Keycloak, JSON all the way | A procurement checklist that literally says "SAML 2.0" | Enterprise deals where SAML support is a line item blocking the contract |
| **Greenfield or integration?** | Greenfield | New B2C product in 2026 — no reason to write XML | Integrating with what the customer already has | Onboarding a 50k-employee customer whose IdP has worked since 2009 |

### Service-to-service: which credential?

| Question to ask | Answer → mechanism | Real-life example |
| --- | --- | --- |
| Public-ish API, low blast radius, needs to be trivially adoptable? | **API key** | A weather API where a leaked key costs you rate-limit abuse |
| Untrusted network path, and payload integrity matters? | **HMAC signing** | Payment-gateway webhooks — the receiver must prove the body wasn't tampered with in flight |
| Many internal services, need scoping and expiry, already run an auth server? | **OAuth client credentials** | An internal platform where each service gets `orders.read`-style scopes from Keycloak |
| Same trust domain, want identity with zero app-level secrets? | **mTLS / SPIFFE** | Pod-to-pod calls inside a Kubernetes cluster running Istio |
| Crossing a platform boundary and want no stored secret at all? | **OIDC federation** | GitHub Actions deploying to AWS by assuming a role via its OIDC token, no `AWS_SECRET_ACCESS_KEY` in repo secrets |

---

## Common mistakes

- **Using OAuth 2.0 as a login protocol.** An access token proves *someone granted access*, not *who is sitting there*. Use OIDC and validate the `aud`, `iss` and `exp` claims of the ID token.
- **Putting secrets in a JWT payload.** It's signed, not encrypted — treat it as public text.
- **Trusting the JWT's own `alg` header.** Pin the algorithm and key server-side.
- **Long-lived access tokens because revocation was hard.** That makes the theft window worse, not better. Short expiry + revocable refresh is the whole point.
- **Storing tokens in `localStorage` "because cookies have CSRF".** You traded CSRF for XSS token theft. `HttpOnly` + `SameSite` cookies, or keep the token in memory only.
- **Treating MFA as binary.** SMS OTP and a passkey are both "MFA" and differ by orders of magnitude against phishing.
- **Rolling your own session/token format.** Every mechanism above exists because a previous generation's homemade scheme was broken in a way that wasn't obvious for years.

## See also

- [passkeys](../general/passkeys.md) — WebAuthn registration/login in detail, and synced vs device-bound credentials.
- [totp](../general/totp.md) — how the rotating 6-digit code is derived, and why it isn't phishing-resistant.
- [http-headers](http-headers.md) — where `Authorization`, `Cookie` and `Set-Cookie` sit among the rest.
- [sql-injection-protection](../spring-boot/sql-injection-protection.md) — the other half of "don't trust input", on the authorization-adjacent side.
