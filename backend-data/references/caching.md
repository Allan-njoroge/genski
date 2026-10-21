# Caching

A cache stores copies of data so it can be served faster or with less load. Caches can make systems dramatically faster, and are also a classic source of bugs: stale data, data leaking between users, and failures when the cache is cold.

## Contents

1. When to cache
2. What to cache
3. Caching patterns
4. Invalidation
5. Cache keys
6. Failure and load problems
7. HTTP caching

## 1. When to cache

Add a cache only when:

- There's a **measured** performance or load problem, and
- Simpler fixes (better queries, indexes, fewer requests) aren't enough, and
- The data can tolerate being slightly out of date, or you have a reliable way to invalidate it.

A cache is an additional system to run, monitor, and reason about. Treat adding one as a deliberate decision (see `decisions`).

## 2. What to cache

Good candidates:

- Data read often and changed rarely: configuration, reference data, product catalogues.
- Expensive computations: aggregated reports, rendered content.
- Responses from slow or rate-limited external services.

Poor candidates:

- Data that must always be current: account balances, stock levels during checkout, permissions just after they change.
- Data that's cheap to fetch already.
- Highly personalised data with little reuse.

## 3. Caching patterns

**Cache-aside (lazy loading),** the most common:

```
value = cache.get(key)
if value is missing:
    value = database.load(...)
    cache.set(key, value, ttl)
return value
```

The application manages the cache. Only requested data is cached. The first request after expiry is slower.

**Write-through:** writes go to the cache and the database together, so the cache is always current for written data. Adds latency to writes.

**Read-only reference data in memory:** small, rarely changing data (such as a list of countries) can be loaded into application memory at startup and refreshed periodically. Remember that each application instance has its own copy.

**Precomputed results:** expensive results are computed by a background job and stored, rather than computed on demand.

## 4. Invalidation

Deciding when cached data stops being valid is the hard part.

- **Time-based expiry (TTL):** every entry expires after a set time. Simple, and bounds staleness. Choose the TTL from the question "how out of date may this data be?"
- **Explicit invalidation:** when data changes, delete or update its cache entry. Do this after the database transaction commits, so the cache isn't cleared and then refilled with the old value before the commit.
- **Versioned keys:** include a version in the key (`product:42:v7`); changing the data changes the version, so old entries are simply no longer used.

Use a TTL even with explicit invalidation, as a safety net for missed invalidations.

Prefer deleting cache entries on change over updating them in place: concurrent updates can leave the wrong value in the cache.

## 5. Cache keys

- **Include everything that affects the value:** IDs, parameters, language, version, and the user or tenant for user-specific or tenant-specific data.
- **A key that omits the user or tenant** can serve one user's data to another. This is a security bug, not just a correctness bug.
- Use a consistent, readable key format: `tenant:{tenantId}:product:{productId}`.
- Keep permission checks outside the cache: cache the data, and check access on every request.

## 6. Failure and load problems

- **The cache will fail sometimes.** The application should fall back to the source of truth (more slowly), not stop working. Set short timeouts on cache calls.
- **Cache stampede:** when a popular entry expires, many requests miss at once and all hit the database. Mitigate with request coalescing (only one request recomputes; others wait), early background refresh, or randomised TTLs.
- **Cold start:** after a restart or deployment, an empty cache sends all traffic to the database. Make sure the database can survive this, or warm critical entries.
- **Memory limits:** configure eviction policies and monitor the cache's hit rate and memory use.
- **Never treat a cache as the only copy** of important data, unless it's explicitly configured and backed up as a durable store.

## 7. HTTP caching

For APIs and web content, HTTP caching headers let clients and intermediate caches reuse responses (see `backend-api-design`):

- `Cache-Control: max-age=...` for responses that can be reused for a time.
- `ETag` with conditional requests, so unchanged resources return `304 Not Modified` without a body.
- `Cache-Control: private` for user-specific responses, so shared caches don't store them, and `no-store` for sensitive data.