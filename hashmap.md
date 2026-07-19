# How a HashMap Works

The core trick: instead of searching through every entry to find a key (O(n)), a HashMap computes a number directly from the key that tells it exactly which slot to look in — turning lookup into, on average, a single array access, O(1).

![HashMap bucket array: apple and plum hash to the same bucket and get chained together; pear hashes to a different bucket](images/hashmap-buckets-and-collisions.png)

## The mechanism

1. **A backing array of "buckets"**: internally, a HashMap holds a fixed-size array. Each slot ("bucket") can hold one or more key-value pairs.
2. **The hash function picks a bucket**: when a key is inserted, its `hashCode()` — a deterministic integer derived from its contents — gets computed, then mixed/reduced down to an index within the array's current size (`hash & (capacity - 1)`, since capacity is kept a power of two). That index is the bucket this key-value pair lives in.
3. **Collisions — different keys, same bucket**: two different keys can hash to the same bucket. When that happens, the bucket holds more than one entry, historically as a small linked list. Java 8+ upgrades a bucket to a balanced red-black tree once it holds enough entries (8+) in a big enough table, bounding the worst case at O(log n) instead of O(n) — this specifically defends against maliciously crafted keys that all hash to the same bucket.
4. **Lookup walks the (usually tiny) bucket**: to fetch a value, hash the key, jump to that bucket, then compare each entry's key with `.equals()` until a match is found. Hash equality alone isn't sufficient proof of a match — since collisions are possible — `.equals()` is the actual tie-breaker.
5. **Resizing keeps buckets short**: the map tracks `size / capacity` (the load factor). Once that ratio passes a threshold (0.75 by default in Java), the backing array doubles in size, and every existing entry gets rehashed into the new array — bucket assignment depends on capacity, so old bucket indices aren't valid in the bigger array. This is the one relatively expensive operation, but it happens rarely enough that average-case performance stays close to O(1).
6. **The hashCode/equals contract**: two keys that are `.equals()` must produce the same `hashCode()` — otherwise the map would send them to different buckets and never realize they're "the same" key. The reverse isn't required: two unequal keys can share a hash (that's just a collision), which is exactly why `.equals()` still has to run inside the matching bucket.

## Real-life analogy

Think of a hotel's wall of numbered mail pigeonholes. Instead of a receptionist checking every pigeonhole for your name, they do quick math on your name (the hash function) and know instantly which numbered hole to check. Usually that's exact — one guest, one hole. Occasionally two different guests' names produce the same number (a collision), so that hole has a small stack of letters, and the receptionist flips through that short stack checking names properly to find yours. If the hotel gets too full (too many letters per hole on average), management gets a bigger wall of pigeonholes and re-sorts all the mail into it using the same math against the new, bigger count — that's a resize.
