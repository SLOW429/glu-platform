# ADR-008: Redis as Cache, Session Store, and Pub/Sub

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU needs fast ephemeral storage for multiple concerns: session management, response caching, rate limiting counters, BullMQ queue backend, real-time pub/sub for notifications, and feature flag caching. These are distinct use cases with a common requirement: sub-millisecond latency with simple data structures.

---

## Decision

**Use Redis 7 as the unified store for caching, sessions, rate limiting, pub/sub, and BullMQ.**

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **Redis** | In-memory key-value store, rich data structures, pub/sub, persistence |
| **Memcached** | Pure cache, simpler, no persistence, no data structures |
| **Dragonfly** | Redis-compatible, multithreaded, drop-in replacement |
| **KeyDB** | Multi-threaded Redis fork |
| **Valkey** | Linux Foundation Redis fork (post license change) |
| **Database-backed sessions** | PostgreSQL session table |
| **In-process cache** | node-cache, no-op cache — process-local only |

---

## Reasons for Redis

### 1. Multiple data structures in one service

Redis is not just a cache. GLU uses:

- **Strings**: cached API responses, session tokens, feature flags
- **Hashes**: user session data, rate limit counters per endpoint
- **Sorted Sets**: leaderboards, trending books by score
- **Lists**: notification queues (complementing BullMQ)
- **Sets**: unique readers per book, blocked users list
- **Pub/Sub**: real-time notification delivery via SSE
- **Streams**: audit event streaming (future)
- **Bitmaps**: reading streak tracking (daily active per user in O(1))

Memcached supports only strings. Using PostgreSQL for these patterns would require expensive queries where Redis delivers microsecond latency.

### 2. BullMQ requires Redis

BullMQ uses Redis sorted sets and hashes to store job state. Redis is already mandatory for the queue. Not using Redis for other concerns would mean running two caching systems.

### 3. Rate limiting precision

`@nestjs/throttler` with the Redis store uses atomic `INCR` + `EXPIRE` commands. This is the standard, race-condition-free approach to distributed rate limiting. Every application instance shares the same counters, so rate limits are accurate across horizontal scaling.

### 4. Pub/Sub for real-time features

Redis Pub/Sub channels deliver notification events from any API instance to any subscriber. The SSE endpoint subscribes to `notifications:{userId}` channel. When any service publishes a notification event, all connected clients for that user receive it immediately.

### 5. Persistence options

- **RDB snapshots**: periodic full dataset dumps to disk
- **AOF (Append-Only File)**: log every write for minimal data loss
- **Hybrid**: both (recommended for production)

Redis can recover from restarts with data intact. Session loss on restart is avoided.

### 6. Atomic operations

Redis operations are single-threaded and atomic. `INCR` on a rate limit counter never causes a race condition. `SETNX` (set if not exists) is a distributed lock primitive. These atomicity guarantees simplify business logic.

### 7. Cluster and Sentinel

For high availability:
- **Redis Sentinel**: automatic failover (replica promoted to primary on failure). Simple setup.
- **Redis Cluster**: horizontal scaling (sharding) for extremely large datasets.

Both options are transparent to NestJS via `ioredis`.

---

## Redis Key Strategy

```
Naming convention: {module}:{entity}:{id}:{field}

Examples:
  sessions:{userId}:{deviceId}          → session data (2h TTL)
  cache:books:featured                   → featured books list (60m TTL)
  cache:books:{id}                       → individual book (10m TTL)
  cache:search:{queryHash}               → search results (5m TTL)
  cache:flags:{flagKey}                  → feature flag value (1m TTL)
  throttle:{userId}:{endpoint}           → rate limit counter (60s TTL)
  throttle:ip:{ip}:{endpoint}            → IP rate limit counter (60s TTL)
  pub:notifications:{userId}             → notification pub/sub channel
  streak:{userId}:{date}                 → reading streak bitmap
  ai:embeddings:{textHash}              → cached embedding (7d TTL)
  ai:budget:{userId}                     → daily token counter (reset midnight)
```

All keys have explicit TTLs. No key is permanent (except intentional long-lived config values).

---

## Pros

- In-memory performance (microsecond latency)
- Rich data structures (strings, hashes, sets, sorted sets, pub/sub, streams)
- Required by BullMQ — no additional infrastructure for caching
- Atomic operations (race-condition-free counters, locks)
- Persistence via RDB + AOF
- Cluster and Sentinel for high availability
- Excellent NestJS ecosystem (`@nestjs/cache-manager`, `ioredis`, `@nestjs/throttler`)
- Sub-millisecond response times observed in production

## Cons

- Entire dataset must fit in memory (mitigated by TTLs, eviction policies)
- Data is lost if Redis crashes without persistence configured (mitigated by AOF)
- Single-threaded command execution (mitigated by Dragonfly if needed)
- No complex queries — not a database replacement
- Memory cost ($$$) if used incorrectly (large payloads stored long-term)

---

## Consequences

- Redis 7 runs as a Docker service (port 6379)
- AOF persistence enabled (`appendonly yes`) in production
- `maxmemory-policy: allkeys-lru` — evict least-recently-used keys when memory is full (safe for cache; sessions must be excluded via separate Redis instance if needed at scale)
- `CacheModule` wraps all cache access — no service imports `ioredis` directly
- BullMQ and `@nestjs/throttler` use the same Redis instance (separate databases: `db: 0` for cache, `db: 1` for queues)
- Key naming conventions enforced by CacheService factory methods
- Redis memory monitored in Grafana — alert when > 80% used
