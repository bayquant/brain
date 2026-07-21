---
tags: [caching, performance, architecture]
---
# Caching

Caching stores a copy of expensive-to-compute or expensive-to-fetch data somewhere faster to read, trading staleness risk for speed and reduced load on the source of truth.

---

## CORE CONCEPTS

```
Client
└── CDN / edge cache
    └── Application (in-memory / local cache)
        └── Distributed cache (Redis, Memcached)
            └── Database (source of truth)
```

- **Hit** — requested data was found in the cache
- **Miss** — requested data was not in the cache, must be fetched from the source
- **TTL (time to live)** — how long an entry is considered valid before it expires
- **Eviction** — removing entries to free space, independent of TTL expiry
- **Staleness** — serving cached data that no longer matches the source of truth

---

## CACHE LEVELS

| Level | Example | Scope | Typical lifetime |
|---|---|---|---|
| Browser / client | HTTP cache | single user | minutes–days |
| CDN / edge | Cloudflare, CloudFront | all users, geo-distributed | minutes–weeks |
| Application (local) | in-process dict/LRU | single process/instance | seconds–minutes |
| Distributed | Redis, Memcached | shared across instances | seconds–hours |
| Database | query cache, materialized view | shared, tied to schema | varies |

Each level trades consistency for speed — the closer to the client, the faster but the harder to invalidate reliably across all copies.

---

## EVICTION POLICIES

```
LRU  (Least Recently Used)   — evict the entry not accessed in the longest time
LFU  (Least Frequently Used) — evict the entry accessed the fewest times
FIFO (First In, First Out)   — evict the oldest entry regardless of access pattern
TTL-based                    — evict on expiry, independent of access pattern
Random                       — evict a random entry (cheap, surprisingly effective at scale)
```

LRU is the default choice for most general-purpose caches (Redis's `allkeys-lru`, most in-memory cache libraries) — it approximates "keep what's likely to be reused" cheaply.

---

## CACHING STRATEGIES

### CACHE-ASIDE (LAZY LOADING)

Application checks the cache first; on a miss, reads from the source and populates the cache.

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query_user(user_id)
        cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

Most common pattern — simple, and the cache only holds what's actually been requested. Downside: first request after a miss always pays the full latency (cold start).

### WRITE-THROUGH

Writes go to the cache and the source synchronously, together. Cache is always consistent with the source, at the cost of write latency.

```python
def update_user(user_id, data):
    db.update_user(user_id, data)                 # write to source
    cache.set(f"user:{user_id}", data)             # write to cache, same request
```

### WRITE-BEHIND (WRITE-BACK)

Writes go to the cache immediately and are flushed to the source asynchronously. Fast writes, but risks data loss if the cache fails before the flush completes.

```python
def update_user(user_id, data):
    cache.set(f"user:{user_id}", data)             # write to cache immediately, caller returns fast
    write_queue.enqueue(("update_user", user_id, data))  # flushed to db later by a background worker
```

### READ-THROUGH

The cache itself is responsible for fetching from the source on a miss — the application only ever talks to the cache, never the source directly. Functionally similar to cache-aside, but the fetch-on-miss logic lives inside the cache layer instead of the caller.

```python
# cache is configured with a loader function once, up front
cache = ReadThroughCache(loader=lambda user_id: db.query_user(user_id), ttl=300)

def get_user(user_id):
    return cache.get(f"user:{user_id}")            # cache handles the miss internally
```

---

## INVALIDATION

The hardest problem in caching — stale data is often worse than a cache miss.

```
TTL expiry           — simplest, but data can be stale for up to the full TTL window
Explicit invalidation — delete/update the cache entry when the source changes
Versioned keys        — key includes a version (user:123:v4); bump version instead of deleting
Event-driven          — source publishes a change event, cache subscribers invalidate on receipt
```

```python
# Explicit invalidation on write
def update_user(user_id, data):
    db.update_user(user_id, data)
    cache.delete(f"user:{user_id}")   # next read repopulates via cache-aside
```

---

## HTTP CACHING

See also [[API Design]] for the request/response contract these headers apply to.

```
Cache-Control: max-age=3600, public     # cacheable for 1 hour, by any cache (CDN, browser)
Cache-Control: max-age=0, private        # cacheable only by the browser, must revalidate
Cache-Control: no-store                 # never cache (sensitive data)

ETag: "33a64df551"                      # opaque fingerprint of the resource's current state
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT
```

```
# Conditional request using ETag — server returns 304 if unchanged, saving the body transfer
If-None-Match: "33a64df551"

304 Not Modified
```

---

## PATTERNS

### CACHE STAMPEDE (THUNDERING HERD)

When a hot key expires, many concurrent requests all miss simultaneously and hammer the source at once.

```python
# Mitigation: lock around the recompute so only one request repopulates the cache
def get_with_lock(key, compute_fn, ttl=300):
    value = cache.get(key)
    if value is not None:
        return value
    with cache.lock(f"lock:{key}", timeout=10):
        value = cache.get(key)          # re-check after acquiring lock
        if value is None:
            value = compute_fn()
            cache.set(key, value, ttl=ttl)
    return value
```

Alternative mitigations: stagger TTLs with random jitter so keys don't all expire at once, or serve stale data while recomputing in the background.

### NEGATIVE CACHING

Cache "not found" results too, with a short TTL — otherwise repeated lookups for a nonexistent key hit the source every time.

### CACHE WARMING

Pre-populate the cache (e.g. on deploy or on a schedule) for known hot keys, so the first real request isn't the one paying the miss cost.

---

## COMMON PITFALLS

- Caching mutable data without a clear invalidation path — leads to silent staleness that's hard to debug
- TTL too long for data that changes often, or too short for expensive-to-compute data that rarely changes
- Caching per-user data under a shared key (missing the user ID in the key) — leaks one user's data to another
- Not handling cache unavailability gracefully — a cache outage shouldn't take down the whole system, just slow it back down to source-of-truth speed
- Unbounded local/in-memory caches — no eviction policy means a slow memory leak under load
