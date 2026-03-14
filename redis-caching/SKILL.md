---
name: redis-caching
description: >
  Redis caching, distributed locking, and rate limiting patterns for .NET 10 with 
  StackExchange.Redis. Use this skill whenever the user asks about caching strategies, 
  cache invalidation, distributed locks, rate limiting with Redis, session management, 
  Redis pub/sub, or any Redis-related concern in .NET. Also trigger for "cache ekle", 
  "cache temizle", "distributed lock", "rate limit", "Redis", "StackExchange.Redis", 
  "IDistributedCache", "cache invalidation", "cache-aside", "write-through", "sliding 
  window", "token bucket", "Redis pub/sub", "cache key", or any performance optimization 
  involving in-memory data stores.
---

# Redis Caching & Distributed Patterns

Production patterns for Redis in .NET 10 with StackExchange.Redis.

## Reference Files — Read on Demand

| When the user asks about... | Read this file |
|----------------------------|---------------|
| Cache-aside, write-through, invalidation, warming | `./caching-patterns.md` |
| Distributed locks, rate limiting, pub/sub, counters | `./distributed-patterns.md` |
| Core setup (ICacheService, connection, keys) | `./setup.md` |

Read only the relevant file.

## Quick Reference

### Cache Key Convention
```
{module}:{entity}:{id}          → appointment:detail:550e8400-...
{module}:{entity}:list:{hash}   → appointment:list:a1b2c3d4
user:{userId}:{scope}           → user:550e8400:profile
```

### Cache Invalidation Rules
| Action | Invalidate |
|--------|-----------|
| Create | List + count caches for that type |
| Update | Entity cache + all list caches |
| Delete | Entity + list + count caches |
| Bulk | All caches for that type (`RemoveByPrefixAsync`) |

### Connection String
```
localhost:6379,abortConnect=false,connectTimeout=5000,syncTimeout=3000,asyncTimeout=3000
```
`abortConnect=false` = don't throw on startup if Redis is down.
