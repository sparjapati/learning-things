# Caching Fundamentals

## What a cache actually is

A cache is a smaller, faster store sitting in front of a slower, authoritative source of truth, holding copies of frequently-accessed data so most requests never reach the slow source. It trades a small staleness/memory cost for a large latency win.

**Analogy**: keeping frequently-used tools in your desk drawer instead of walking to the supply room every time. The supply room (database) has everything and is authoritative, but it's slow to reach. Your drawer (cache) only holds your most-used items — if something isn't there, you walk to the supply room and (usually) bring a copy back to your drawer for next time.

## Where caching sits — the layers, client to database

Caching isn't one thing in one place — it exists at every hop of a request path:

1. **Client/browser cache** — `Cache-Control`/`ETag` HTTP headers, local storage
2. **CDN / edge cache** — Cloudflare, Fastly, CloudFront — geographically close to the user
3. **Reverse proxy cache** — NGINX, Varnish, sitting in front of app servers
4. **Application/in-memory cache** — Caffeine, Guava — local to *one* server instance
5. **Distributed cache** — Redis, Memcached — shared across all app instances
6. **Database's own buffer pool** — e.g. MySQL's InnoDB buffer pool caches pages in its own memory

Each layer that hits (returns data without going further) short-circuits everything below it. Closer to the client = faster but smaller/more likely stale; closer to the DB = slower but more authoritative.

![Caching layers in a request path](images/caching-fundamentals-eraser.png)

## The core patterns: how reads and writes flow through a cache

| Pattern | How it works | Trade-off |
|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache → miss → reads DB → app populates cache | Simplest, most common. First request for any key is always a slow miss; concurrent misses on the same key risk a [thundering herd](thundering-herd-problem.md) |
| **Read-through** | Same idea, but the cache *itself* owns the DB fallback — app just asks the cache | Centralizes the loading logic (e.g. Guava `LoadingCache`, Spring `@Cacheable` + a loader) |
| **Write-through** | Every write updates cache **and** DB synchronously before returning | Cache and DB never diverge, but every write pays the cost of two systems |
| **Write-behind** (write-back) | Write hits the cache immediately; DB write is deferred/batched async | Great write latency, but a crash before the deferred write flushes loses data |
| **Write-around** | Writes go straight to the DB, bypassing the cache entirely; cache only fills on a later read | Avoids polluting the cache with data that's rarely re-read soon after writing |

### Decision checklist

| Question | Pattern | Real-life example |
|---|---|---|
| Is read latency the priority, and is an occasional DB-hit miss acceptable? | Cache-aside | Product catalog pages |
| Want the caching logic centralized out of app code? | Read-through | Guava `LoadingCache`, Spring `@Cacheable` with a `CacheLoader` |
| Must cache and DB *never* diverge, even briefly? | Write-through | A wallet/ledger balance shown right after a transaction |
| Is write throughput critical, and is losing the very latest write on crash tolerable? | Write-behind | View counts, high-volume metrics |
| Is written data rarely read again right away? | Write-around | Bulk imports, audit logs |

## Eviction policies — what gets thrown out when the cache is full

**Analogy**: a fridge with limited shelf space.

- **LRU (Least Recently Used)** — evict what hasn't been touched in the longest time. The default in Redis, Memcached, Caffeine.
- **LFU (Least Frequently Used)** — evict what's used least often *overall*, even if touched recently once.
- **FIFO** — evict whatever was inserted longest ago, regardless of usage.
- **TTL expiration** — evict on a fixed time budget regardless of usage, bounding staleness. Usually combined with LRU/LFU rather than used alone.

## Invalidation — the actually-hard part

- **TTL expiration** — simplest; accept staleness up to the TTL window.
- **Explicit/event-driven invalidation** — on every write to the source, also delete/update the matching cache entry. Precise, but requires remembering to do it at every write site.
- **Versioned keys** — change the key itself when data changes (`user:5:v3`, `app.a1b2c3.js`) so old entries just go stale/unreferenced rather than needing active deletion. Standard for CDN/static-asset caching.
- Write-through sidesteps invalidation entirely by keeping cache and DB in lockstep on every write.

## Where this connects to earlier notes

- **Hibernate's first-level (session) and second-level (`@Cacheable`) caches** are a specific instance of the "application-level cache" layer above — same core ideas (eviction, staleness, invalidation), just scoped to entity objects. See [spring-boot/jpa-hibernate-spring-data-stack.md](../spring-boot/jpa-hibernate-spring-data-stack.md).
- A hot cache key expiring under heavy concurrent traffic is exactly the [thundering herd / cache stampede problem](thundering-herd-problem.md).
- [Redis](redis-single-threaded.md) and Memcached are the most common distributed-cache technologies; see [redis-persistence-rdb-vs-aof.md](redis-persistence-rdb-vs-aof.md) for how a cache that also needs to survive a restart persists data.
- Rate limiters ([rate-limiters.md](rate-limiters.md)) often store their own counters/tokens in a cache like Redis.
