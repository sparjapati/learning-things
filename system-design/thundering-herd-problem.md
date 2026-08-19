# Thundering Herd Problem (Cache Stampede)

> "Men, it has been well said, think in herds; it will be seen that they go mad in herds, while they only recover their senses slowly, and one by one."
> — Charles Mackay, *Extraordinary Popular Delusions and the Madness of Crowds*

## What it is

The **thundering herd problem** (also called a **cache stampede**) happens when a shared resource that many concurrent requests depend on — most commonly a cache entry — becomes unavailable all at once (it expires, or the cache/server restarts), and *every* one of those waiting requests rushes to regenerate it **simultaneously**, overwhelming the backend the cache was protecting in the first place.

See [caching-fundamentals.md](caching-fundamentals.md) for the broader caching mental model (layers, patterns, eviction, invalidation) this problem is a specific failure mode of.

The term originally comes from OS/networking: multiple processes blocked waiting on the same event (e.g. all `accept()`-ing on the same socket) get woken up together when it fires, even though only one of them can actually make progress — the rest just burned CPU for nothing. The caching usage is the same shape, just with a database instead of a socket.

## Real-life analogy: an actual herd stampede

The name is literal: a herd of animals grazing calmly reacts to a single stimulus (a clap of thunder, a predator) and all bolt through the same narrow passage at the exact same instant — trampling each other and jamming a passage that could easily handle them if they trickled through one at a time. Nothing about the passage changed; what changed is that everyone hit it in the same instant instead of being spread out.

## The concrete scenario

1. A hot cache key — a celebrity's profile, a homepage feed — has a 60-second TTL.
2. Traffic is high (viral moment, flash sale). At the exact second the TTL expires, thousands of requests arrive.
3. **All of them miss the cache at the same instant** and go straight to the database (or an expensive computation, or a third-party API) to regenerate the same value — redundantly, thousands of times over.
4. The database, only ever sized to handle occasional cache-refill traffic, gets hit with thousands of *identical* queries at once → connection pool exhaustion → rising latency → timeouts → retries pile on more load → the system can cascade into a full outage. It's effectively a self-inflicted DDoS.

![Thundering herd: without vs with mitigation](images/thundering-herd-problem-eraser.png)

## Where else it shows up

- **Retry storms** — a transient failure causes every client to retry after the same fixed backoff, so they all hammer the server again in sync.
- **Distributed cron** — many servers scheduled to run a job at the exact same wall-clock time, all hitting a shared resource together.
- **DNS TTL expiry** — many clients re-resolving the same record at once when it expires.

## Mitigation techniques

| Technique | What it does | Real-life example |
|---|---|---|
| **Single-flight lock (mutex per key)** | Only one request recomputes the value; others wait briefly or get the stale value while it refreshes | Go's `singleflight` package; NGINX's `proxy_cache_lock` |
| **Stale-while-revalidate** | Serve everyone the stale value immediately while exactly one background request refreshes it | HTTP `Cache-Control: stale-while-revalidate`, supported by Cloudflare/Fastly |
| **Jittered TTL** | Add randomness to expiration (`60s ± 10s`) so keys don't all expire at the same instant | Standard cache library option (e.g. Redis client-side TTL jitter) |
| **Probabilistic early expiration** | Recompute *before* actual expiry, with probability rising as expiry nears — spreads refreshes out over time | Facebook's "XFetch" algorithm |
| **Proactive background refresh** | A scheduled job refreshes known hot keys before they'd ever expire under load | Cache-warming jobs for a homepage/trending feed |
| **Exponential backoff + jitter (client side)** | Prevents synchronized retry storms after a transient failure | AWS SDK's default retry policy |

## Does Spring's `@Cacheable` protect against this?

Not by default. Plain `@Cacheable` has no per-key locking — on expiry, every concurrent caller misses independently and hits the underlying method (and therefore the DB) at the same instant.

Spring does offer an opt-in single-flight lock: `@Cacheable(value = "users", key = "#id", sync = true)`. This switches to the cache's `get(key, Callable)` method, which blocks concurrent callers for the *same key* on whichever thread got there first — exactly the single-flight pattern above, built in.

**The catch: it's scoped to one JVM.** `sync = true` fully collapses the herd within a single running instance, but replicas don't coordinate with each other — a fleet of 5 replicas can still produce up to 5 concurrent DB calls for the same key at the same instant. It also can't be combined with multiple cache names or with `unless` (only `condition` is allowed). Eliminating the herd fleet-wide needs an actual distributed lock (e.g. a Redis `SETNX`-based mutex around the load step), not something `@Cacheable` provides on its own. See [caching-fundamentals.md](caching-fundamentals.md#which-pattern-does-springs-caching-abstraction-use) for how `@Cacheable`/`@CachePut`/`@CacheEvict` map to the caching patterns overall.

## Decision: which mitigation fits your system?

| Question | If yes → | Real-life example |
|---|---|---|
| Is briefly-stale data acceptable while refreshing? | Stale-while-revalidate / background refresh | Trending topics shown a few seconds stale is fine |
| Must the value always be exactly fresh? | Single-flight lock (others wait, never served stale) | Bank account balance — staleness unacceptable, but still don't want N duplicate DB hits |
| Is the hot key predictable in advance (known celebrity/campaign)? | Proactive cache warming before expiry | Pre-refreshing a flash-sale product page before the sale starts |
| Is the storm coming from your *clients'* retries, not your own cache? | Exponential backoff + jitter on the client, rate limiting as a backstop on the server | API clients retrying after a 429/503 |
| Just want a cheap, low-effort default with no code complexity? | Jittered TTL | Any cache with per-key TTLs — spreads expiry automatically |

## Code shape: single-flight lock

```
function getValue(key):
    value = cache.get(key)
    if value exists: return value

    if not acquireLock(key):              // someone else is already recomputing
        sleep briefly, then retry cache.get(key)   // or: return stale value if allowed
    else:
        try:
            value = computeExpensiveValue(key)
            cache.set(key, value, ttl = 60 + randomJitter())
            return value
        finally:
            releaseLock(key)
```

See [system-design/rate-limiters.md](rate-limiters.md) for how rate limiting acts as a backstop once a storm is already underway, and [system-design/redis-single-threaded.md](redis-single-threaded.md) for why a cache like Redis being hit with duplicate concurrent queries is itself a bottleneck worth avoiding.
