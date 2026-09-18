# WebSockets and the `Upgrade` Handshake

> "HTTP is a conversation where you may only speak when spoken to."
>
> *WebSocket is the escape hatch: it spends one HTTP request buying the right to stop speaking HTTP.*

## The problem it solves

HTTP is **request/response**: the client asks, the server answers. The server has no way to say something first. For chat, live prices, notifications, collaborative editing or progress updates, that's backwards — and the workarounds are all bad:

| Workaround | Why it hurts |
| --- | --- |
| **Polling** — ask every 5 s | Mostly empty responses; latency up to the interval; load scales with clients × frequency |
| **Long polling** — hold the request open until something happens | Better latency, but a fresh request (and headers, and TLS state) per message, and connections held hostage |

**WebSocket** replaces all of it with a single, persistent, **full-duplex** connection: either side may send at any time, with framing overhead of a few bytes instead of a full header block.

## The handshake: how HTTP turns into not-HTTP

A WebSocket connection begins life as an ordinary HTTP/1.1 GET:

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

The server agrees with a status code you'll rarely see anywhere else:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After that `101`, **the connection is no longer HTTP.** The same TCP (and TLS) connection now carries WebSocket frames in both directions. No further headers, no request/response pairing — see [http-headers](http-headers.md) for the `Upgrade`/`Connection` pair as hop-by-hop headers.

### Why `Sec-WebSocket-Accept` exists

The server takes the client's `Sec-WebSocket-Key`, appends the fixed GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`, SHA-1 hashes it, and base64-encodes the result.

This proves nothing about security — the GUID is public. Its job is to prove the server **actually understands WebSocket** rather than being a naive server or cache that echoed the request back. Without it, an attacker could trick an intermediary into treating attacker-controlled bytes as a cached HTTP response. It's an anti-confusion check, not authentication.

### Why client frames are masked

Every frame a **client** sends is XOR-masked with a random 32-bit key (servers don't mask). This isn't encryption — the key travels in the frame.

It exists because of **broken transparent proxies**: without masking, an attacker could craft WebSocket payloads that look like a valid HTTP request to an intermediary that doesn't understand WebSocket, poisoning its cache. Masking makes the bytes on the wire unpredictable, so they can't be shaped into something a confused proxy would parse as HTTP.

**Real-life analogy.** HTTP is a formal correspondence: you post a letter, you get a reply, and the office cannot write to you unprompted (**request/response only**). Polling is posting "anything for me?" every five minutes (**wasteful and always slightly stale**). WebSocket is using that formal channel *once* to request a phone line be installed (**the `Upgrade` handshake over HTTP**) — and once the operator confirms (**`101 Switching Protocols`**), you stop writing letters entirely and simply talk, either party speaking whenever they like (**full duplex, no more HTTP**). The confirmation code the operator reads back proves they're a real operator and not an answering machine repeating your words (**`Sec-WebSocket-Accept`**).

## Frames, not requests

After the handshake, data moves as frames with a 2–14 byte header:

| Opcode | Meaning |
| --- | --- |
| `0x1` | Text frame (UTF-8) |
| `0x2` | Binary frame |
| `0x8` | Close |
| `0x9` / `0xA` | **Ping / Pong** — heartbeats |

Ping/pong matters more than it sounds: **idle TCP connections get killed** by load balancers, NAT gateways and proxies, typically after 30–120 seconds of silence. A periodic ping keeps the connection alive and detects a peer that has vanished without a FIN. Most "the WebSocket randomly disconnects after a minute" bugs are a missing heartbeat or an LB idle timeout shorter than it.

## What this does to your infrastructure

This is where WebSockets stop being a client-side topic.

**Connections are long-lived and stateful.** A WebSocket pins a client to **one specific server instance** for the life of the connection. That breaks the stateless assumption most web architecture rests on:

- **Deploys disconnect everyone.** Rolling a deployment drops every connection on that instance; clients must reconnect with backoff and jitter, or you get a [thundering-herd-problem](thundering-herd-problem.md) at restart.
- **Autoscaling is lumpy.** Scaling in kills active connections; scaling out gains you nothing until clients reconnect.
- **Load balancing shifts meaning.** Round robin balances *connections*, and connections last hours — so connection fairness says nothing about message-rate fairness. This is exactly the failure mode in [load-balancing-algorithms](load-balancing-algorithms.md): balance by active connections or least-load, not round robin.
- **The LB must support upgrades.** An L7 proxy has to be configured to pass `Upgrade`/`Connection` through, and its idle timeout raised above your heartbeat interval. An L4 balancer passes it through naturally.

**Fan-out needs a backplane.** If user A is connected to instance 1 and user B to instance 2, instance 1 cannot deliver a message to B directly. You need a pub/sub layer — Redis pub/sub, Kafka, or a dedicated broker — that every instance subscribes to.

```
            ┌── instance 1 ── [ user A ]
[ Redis ] ──┤
  pub/sub   └── instance 2 ── [ user B ]

  A's message → instance 1 → publish → both instances → deliver to their own connections
```

**You're now sizing for connection count, not request rate.** 100k idle WebSocket connections cost memory and file descriptors but almost no CPU — a completely different capacity model than requests/second. This is the classic C10K problem, and it's why event-loop servers (Netty, and Spring WebFlux on top of it) suit WebSockets far better than thread-per-request ones.

## Security

- **Use `wss://`**, always. `ws://` is plaintext, and many proxies mangle it anyway.
- **Check `Origin` yourself.** This is the big one: **the same-origin policy does not apply to WebSockets.** Any site can open a WebSocket to your server, and the browser will attach cookies. If you authenticate by cookie and don't validate `Origin`, you have **Cross-Site WebSocket Hijacking** — an attacker's page opens an authenticated socket as your logged-in user.
- **Authenticate at the handshake**, since there's no per-message auth. A token in the URL query string works but leaks into logs; a short-lived ticket fetched over HTTPS and sent as the first frame is cleaner.
- **Validate and rate-limit messages.** A connection that's authenticated once can then send anything, as fast as it likes.

## Alternatives — usually the better default

WebSocket is not the answer to "the server needs to push". Most of the time it's overkill:

| Option | Direction | Runs over | Notes |
| --- | --- | --- | --- |
| **Polling** | Client pulls | HTTP | Fine for slow-changing data. Genuinely underrated |
| **Server-Sent Events (SSE)** | **Server → client only** | Plain HTTP | Auto-reconnect built in, works through any proxy, trivially simple |
| **WebSocket** | **Bidirectional** | Its own protocol after upgrade | Most capable, most infrastructure cost |
| **HTTP/2 or /3 streaming** | Server → client | HTTP/2+ | gRPC server streaming |

**If the client only needs to receive, use SSE.** It's ordinary HTTP — it passes through every proxy, reconnects automatically with `Last-Event-ID`, needs no upgrade support, and survives HTTP/2 multiplexing. Reach for WebSocket when the **client also needs to send frequently** — chat, collaborative editing, live cursors, gaming.

### On HTTP/2 and /3

WebSocket was designed for HTTP/1.1, where a connection could be wholly repurposed. Over HTTP/2 that's wrong — the connection is shared by many streams — so **RFC 8441** defines Extended CONNECT, upgrading a single *stream* via the `:protocol` pseudo-header. Support is uneven, and many stacks still negotiate HTTP/1.1 specifically for the WebSocket endpoint.

## How to decide

### Which push mechanism?

| Question to ask | → Polling | Real-life example | → SSE | Real-life example | → WebSocket | Real-life example |
|---|---|---|---|---|---|---|
| How often does the data change? | Slowly, predictably | An order status checked every 30 s | Frequently, one way | A live deployment log or price ticker | Constantly, both ways | A chat room or collaborative document |
| Does the client need to send frequently? | No | — | **No — this is the deciding question** | Dashboard updates | **Yes** | Typing indicators, live cursors |
| How much latency is acceptable? | Seconds | Background sync | Sub-second | Notifications | Milliseconds | Multiplayer gameplay |
| Must it traverse awkward proxies? | Trivially | Any corporate network | Yes — it's plain HTTP | Enterprise environments | Needs upgrade support and raised timeouts | Networks you control |
| Can you run stateful, long-lived connections? | No state needed | Serverless functions | Minimal | Mostly fine | Requires sticky instances + a pub/sub backplane | A dedicated realtime tier |

**The rule of thumb: polling until it hurts, SSE if the server just needs to talk, WebSocket only when the conversation is genuinely two-way.**

## Common mistakes

| Mistake | Consequence |
| --- | --- |
| No ping/pong heartbeat | Connections silently dropped by LB/NAT idle timeouts |
| LB idle timeout shorter than the heartbeat interval | Same, but harder to diagnose |
| Not validating `Origin` | Cross-Site WebSocket Hijacking — the same-origin policy won't save you |
| Round robin over WebSocket connections | Balances connections, not load; one instance ends up carrying the traffic |
| No pub/sub backplane | Messages only reach users on the same instance |
| No reconnect backoff + jitter | Every client reconnects simultaneously after a deploy — a thundering herd |
| Using WebSocket for server→client only | SSE would have been simpler, proxy-friendly and auto-reconnecting |
| Thread-per-connection server model | Falls over at a few thousand connections; use an event-loop stack |
| Assuming the reverse proxy passes upgrades by default | `400`/`426` at handshake until `Upgrade`/`Connection` are explicitly forwarded |
| Auth token in the WebSocket URL | Leaks into access logs and referrers |

## Key takeaways

1. **The handshake is HTTP; everything after `101 Switching Protocols` is not.**
2. **`Connection: Upgrade` + `Upgrade: websocket`** request the switch; both are hop-by-hop, so every proxy in the path must be configured to pass them.
3. **`Sec-WebSocket-Accept` proves the server understands the protocol**, not that anyone is authenticated.
4. **Client frames are masked to protect broken proxies**, not for confidentiality.
5. **Heartbeats are mandatory in practice** — idle connections get culled by infrastructure.
6. **A WebSocket pins a client to one instance**, which breaks stateless deploys, autoscaling and round-robin balancing.
7. **Cross-instance delivery needs a pub/sub backplane.**
8. **The same-origin policy does not apply** — validate `Origin` yourself.
9. **If the flow is one-way, use SSE.** WebSocket earns its cost only when both sides talk.

## See also

- [http-headers](http-headers.md) — `Upgrade`, `Connection` and why hop-by-hop headers are stripped by proxies.
- [http-protocol-versions](http-protocol-versions.md) — why WebSocket's connection-takeover model fits HTTP/1.1 and needed redesigning for HTTP/2.
- [load-balancing-algorithms](load-balancing-algorithms.md) — why round robin is the wrong choice for long-lived connections.
- [http-connections-and-tomcat-threading](../spring-boot/http-connections-and-tomcat-threading.md) — why an event loop, not a thread per connection, is what makes 100k open sockets affordable.
- [thundering-herd-problem](thundering-herd-problem.md) — what a mass reconnect after a deploy does, and why backoff needs jitter.
