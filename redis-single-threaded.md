# How Redis Works, Why It's Called Single-Threaded, and Common Use Cases

## How Redis works

Redis is an in-memory key-value data store. It keeps the entire dataset in RAM, which is why reads/writes are extremely fast (microsecond-level) compared to disk-backed databases like MySQL/MongoDB.

- **Data structures, not just strings**: values can be strings, hashes, lists, sets, sorted sets, etc. — used for caching, session storage, leaderboards, rate limiting, pub/sub messaging.
- **Persistence is optional but available**: Redis can snapshot to disk (RDB) or append every write to a log (AOF) so data survives a restart, even though the hot path for serving requests is all RAM. See [[redis-persistence-rdb-vs-aof]] for details.
- **Client-server model**: clients open a TCP connection and send commands (`GET`, `SET`, `LPUSH`, etc.); Redis processes them and replies.

## Why it's called "single-threaded"

Redis's core command execution — the part that actually reads/writes data — runs on a single thread. This is a deliberate design choice:

1. **No locks needed** — only one thread touches the data at a time, so there's no race between clients modifying the same key, and no need for locks/mutexes.
2. **Commands are atomic by default** — e.g. `INCR counter` is guaranteed atomic since commands execute one at a time with no concurrent access.
3. **CPU usually isn't the bottleneck** — operations are simple (hash lookups, list pushes) and RAM access is already fast; the real bottleneck is typically network I/O (waiting on clients), not CPU computation. A single thread can still deliver very high throughput.
4. **Avoids context-switching/cache-invalidation overhead** — multiple threads fighting over shared memory would add synchronization costs that a single thread sidesteps entirely.

### Nuance: not *entirely* single-threaded anymore

Since Redis 6+, reading data off the network socket and writing responses back (I/O threading) can be parallelized across threads. But the actual command execution against the data structures remains single-threaded. So "single-threaded" specifically describes **the part that touches the data**, not the whole process.

This splits into two separate jobs per request:

1. **Network I/O** — accepting the connection, reading raw bytes off the socket, parsing them into a command, and later writing the response bytes back. This part *can* run on multiple threads since Redis 6.
2. **Command execution against the data structures** — actually looking up the key in Redis's internal hash table/list/set/etc., reading or mutating it, and producing the result. This part always runs on exactly one thread, one command at a time.

Example: if 4 clients send `GET a`, `GET b`, `SET c 1`, `INCR d` at nearly the same instant, 4 I/O threads can read/parse all 4 requests off their sockets in parallel — but none of those commands can actually run against the dataset at the same instant. The single execution thread still picks them up one at a time (e.g. `GET a`, then `GET b`, then `SET c 1`, then `INCR d`) before moving to the next. Requests arrive and get parsed in parallel, but the "touch the data" step stays serialized — which is exactly what preserves the no-locks-needed, atomic-by-default guarantees above. If execution itself were multi-threaded, two threads could try to mutate the same key at once and Redis would need locks again.

### Example: why `INCR` specifically is atomic

`INCR` bundles "read current value → add 1 → write back" into a single command that runs start-to-finish on the single thread with no interruption. Two clients calling `INCR counter` never interleave:

```
Client A: INCR counter   → Redis internally does read(5)+1=6, write(6), returns 6 — one command
Client B: INCR counter   → Redis internally does read(6)+1=7, write(7), returns 7 — one command
```

Compare this to replicating the same logic manually with separate `GET`/`SET` commands from application code — that's NOT atomic, because the two round-trips can interleave:

```
Client A: GET counter        → returns 5
Client B: GET counter        → returns 5   (before A's write happens)
Client A: SET counter 6      → writes 6
Client B: SET counter 6      → writes 6   (overwrites A's increment — one increment lost)
```

Both clients meant to increment once each (5 → 7), but the final value ends up 6, not 7, because `GET`+`SET` are two separate commands rather than one atomic unit. To replicate this safely across multiple commands, you'd need `MULTI`/`EXEC` transactions or a Lua script, which bundle multiple commands into one atomic unit.

## Analogy

Like a single cashier at a counter: every customer (command) is served one at a time, so there's never a scramble over the same till (no locks needed), and each transaction fully completes before the next starts (atomicity). As long as each transaction is fast (in-memory ops are), a single cashier keeps up with high throughput.

## Use cases beyond caching

Redis is usually introduced as "just a cache," but its data structures and speed make it useful for much more:

1. **Session storage** — store login/cart state externally so any server in a load-balanced cluster can read the same session (avoids "sticky sessions").
2. **Rate limiting / throttling** — `INCR` + `EXPIRE` on a key like `rate_limit:user123` counts requests per time window atomically, no separate counter service needed.
3. **Leaderboards / ranking** — the sorted set (`ZSET`) keeps members ordered by score automatically; `ZADD`/`ZREVRANGE` gives top-N without manual sorting.
4. **Pub/Sub messaging** — `PUBLISH`/`SUBSCRIBE` channels deliver messages to subscribers instantly; used for lightweight real-time features (chat notifications, live updates).
5. **Distributed locks** — `SET key value NX EX 10` is atomic set-if-not-exists-with-expiry, letting multiple app instances coordinate (e.g. ensure only one server runs a scheduled job at a time — the "Redlock" pattern).
6. **Message/task queues** — `LPUSH`/`BRPOP` (list push / blocking pop) act as a simple job queue; simpler than RabbitMQ/Kafka for lightweight background job processing.
7. **Real-time analytics / counters** — view counts, like counts, or unique-visitor counts (via `HyperLogLog`, a probabilistic set for approximate unique counts) are cheap in Redis vs. repeated `COUNT DISTINCT` queries on SQL.
8. **Geospatial queries** — built-in geo commands (`GEOADD`, `GEOSEARCH`) store coordinates and answer "find all points within X km" (e.g. nearby drivers/restaurants).

**Why caching dominates in practice**: most apps already have a source-of-truth database (MySQL/Mongo), and the cheapest win is putting Redis in front of slow, repeated reads. The other use cases require making Redis the primary store for that data, which is a bigger architectural commitment — so teams reach for it there less often, even though Redis is fully capable.
