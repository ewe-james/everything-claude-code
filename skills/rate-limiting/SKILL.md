---
name: rate-limiting
description: Rate limiting patterns for NestJS — in-memory sliding window, NestJS Throttler module, and IP/user-based limiting strategies.
origin: EWE
---

# Rate Limiting Patterns

Rate limiting strategies to protect NestJS API endpoints from abuse.

## When to Activate

- Protecting public endpoints from excessive requests
- Adding per-user or per-IP request quotas
- Setting up global or per-route throttling in NestJS

---

## NestJS Throttler (Recommended)

The `@nestjs/throttler` package integrates natively with NestJS guards.

```bash
bun add @nestjs/throttler
```

```typescript
// app.module.ts
ThrottlerModule.forRoot([
  { name: 'short', ttl: 1000,  limit: 5  },   // 5 req/s
  { name: 'long',  ttl: 60000, limit: 100 },  // 100 req/min
])

// Register guard globally
providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }]
```

Override per route:
```typescript
@Throttle({ short: { limit: 2, ttl: 1000 } })
@Post('login')
async login(@Body() body: unknown) { ... }

@SkipThrottle()
@Get('health')
health() { return 'ok' }
```

---

## In-Memory Sliding Window (No Dependencies)

Simple sliding-window implementation using a `Map`. Works for single-instance deployments or local dev. Not suitable for multi-replica — use Redis-backed throttling instead.

```typescript
// common/rateLimit/rateLimiter.service.ts
@Injectable()
export class RateLimiter {
  private readonly requests = new Map<string, number[]>()

  check(identifier: string, maxRequests: number, windowMs: number): boolean {
    const now    = Date.now()
    const recent = (this.requests.get(identifier) ?? []).filter((t) => now - t < windowMs)

    if (recent.length >= maxRequests) return false

    recent.push(now)
    this.requests.set(identifier, recent)
    return true
  }
}
```

Usage in a NestJS interceptor:
```typescript
@Injectable()
export class RateLimitInterceptor implements NestInterceptor {
  constructor(private readonly limiter: RateLimiter) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = ctx.switchToHttp().getRequest<Request>()
    const ip  = req.headers['x-forwarded-for'] as string ?? req.socket.remoteAddress ?? 'unknown'

    if (!this.limiter.check(ip, 100, 60_000)) {
      throw new HttpException('Rate limit exceeded', HttpStatus.TOO_MANY_REQUESTS)
    }

    return next.handle()
  }
}
```

Register globally or per-controller:
```typescript
// Global
providers: [{ provide: APP_INTERCEPTOR, useClass: RateLimitInterceptor }]

// Per-controller
@UseInterceptors(RateLimitInterceptor)
@Controller('public')
export class PublicController {}
```

---

## Redis-Backed Rate Limiting (Multi-Replica)

Use Redis `INCR` + `EXPIRE` for distributed rate limiting across multiple instances.

```typescript
@Injectable()
export class RedisRateLimiter {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async check(identifier: string, maxRequests: number, windowSec: number): Promise<boolean> {
    const key   = `rate:${identifier}`
    const count = await this.redis.incr(key)

    if (count === 1) {
      await this.redis.expire(key, windowSec)
    }

    return count <= maxRequests
  }
}
```

Usage:
```typescript
const allowed = await this.rateLimiter.check(`ip:${ip}`, 100, 60)
if (!allowed) throw new HttpException('Rate limit exceeded', HttpStatus.TOO_MANY_REQUESTS)
```

---

## Strategy Comparison

| Strategy | Multi-replica | Persistence | Complexity |
|----------|--------------|-------------|------------|
| `@nestjs/throttler` (in-memory) | No | No | Low |
| In-memory sliding window | No | No | Low |
| Redis `INCR` / `EXPIRE` | Yes | Yes | Medium |
| Redis sliding window (sorted set) | Yes | Yes | High |

For production multi-replica deployments, always use a Redis-backed strategy.
