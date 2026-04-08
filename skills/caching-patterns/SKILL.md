---
name: caching-patterns
description: Backend caching strategies — Redis cache-aside, in-memory cache, and HTTP cache headers. Use when adding a cache layer to reduce DB load or API latency.
origin: EWE
---

# Caching Patterns

Caching strategies for NestJS backend services.

## When to Activate

- Adding a Redis cache layer to a repository
- Caching expensive database queries or external API calls
- Setting HTTP `Cache-Control` headers on GET responses
- Implementing in-memory caching for single-instance or development use

---

## Redis Cache-Aside

The most common pattern: read from cache first, fall back to the source, then write to cache.

```typescript
// modules/market/cachedMarket.repository.ts
@Injectable()
export class CachedMarketRepository implements IMarketRepository {
  constructor(
    private readonly baseRepo: MarketRepository,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  private key = (id: string) => `market:${id}`

  async findById(id: string): Promise<TMarket | null> {
    const cached = await this.redis.get(this.key(id))
    if (cached) return JSON.parse(cached)

    const market = await this.baseRepo.findById(id)
    if (market) await this.redis.setex(this.key(id), 300, JSON.stringify(market))

    return market
  }

  async invalidate(id: string) {
    await this.redis.del(this.key(id))
  }
}
```

Key points:
- TTL (seconds) as second arg to `setex` — always set a TTL, never cache indefinitely
- Call `invalidate` in write operations (update/delete) to prevent stale reads
- Use a consistent key naming convention: `<entity>:<id>`
- TTL guidance: 5–60s for frequently-updated data (leaderboards, counters); 5–30min for reference data (task configs, static lists)

### Redis Module Setup

```typescript
// app.module.ts
import { RedisModule } from '@nestjs-modules/ioredis'

RedisModule.forRoot({
  type: 'single',
  url:  process.env.REDIS_URL,
})
```

---

## In-Memory Cache (Single Instance / Dev)

Use when Redis is unavailable or for caching within a single process. Not suitable for multi-replica deployments.

```typescript
// common/cache/inMemoryCache.service.ts
@Injectable()
export class InMemoryCacheService {
  private readonly store = new Map<string, { value: unknown; expiresAt: number }>()

  get<T>(key: string): T | null {
    const entry = this.store.get(key)
    if (!entry) return null
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key)
      return null
    }
    return entry.value as T
  }

  set(key: string, value: unknown, ttlMs: number): void {
    this.store.set(key, { value, expiresAt: Date.now() + ttlMs })
  }

  delete(key: string): void {
    this.store.delete(key)
  }
}
```

Usage:
```typescript
const cacheKey = `user:${userId}:profile`
const cached = this.cache.get<TProfile>(cacheKey)
if (cached) return cached

const profile = await this.profileRepo.findByUserId(userId)
this.cache.set(cacheKey, profile, 60_000)  // 60 seconds
return profile
```

---

## HTTP Cache Headers

Add `Cache-Control` on GET endpoints that return stable data. Lets browsers and CDNs cache without hitting the server.

```typescript
// Controller — sets cache headers on the response
@Get(':id')
async getMarket(
  @Param('id') id: string,
  @Res({ passthrough: true }) res: Response,
): Promise<{ data: TMarket }> {
  const market = await this.marketService.findById(id)
  // Cache for 60s publicly; allow stale for 30s while revalidating
  res.setHeader('Cache-Control', 'public, max-age=60, stale-while-revalidate=30')
  return { data: market }
}
```

Common directives:

| Directive | Meaning |
|-----------|---------|
| `no-store` | Never cache (auth'd user data) |
| `no-cache` | Cache but always revalidate |
| `private, max-age=N` | Browser only, N seconds |
| `public, max-age=N` | CDN + browser, N seconds |
| `stale-while-revalidate=N` | Serve stale for N seconds while refreshing |

