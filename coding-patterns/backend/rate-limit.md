# Rate Limit Pattern

Per-IP rate limiting using `rate-limiter-flexible` with Redis (primary) and in-memory fallback (degraded mode). Fails open on unexpected errors — never blocks legitimate traffic due to infra issues.

---

## File locations

```
src/shared/rate-limit/rate-limit.decorator.ts   ← @RateLimit(points, duration) metadata decorator
src/shared/rate-limit/rate-limit.guard.ts       ← NestJS guard (reads Reflector, sets Retry-After)
src/shared/rate-limit/rate-limit.service.ts     ← Redis-backed limiter with in-memory fallback
src/shared/rate-limit/rate-limit.module.ts      ← NestJS module (imports CacheModule + EnvModule)
```

---

## Decorator

```ts
// src/shared/rate-limit/rate-limit.decorator.ts
import { SetMetadata } from '@nestjs/common'

export interface RateLimitOptions {
  points: number
  duration: number // in seconds
}

export function RateLimit(points: number, duration: number) {
  return SetMetadata('rateLimitOptions', { points, duration } as RateLimitOptions)
}
```

---

## Guard

```ts
// src/shared/rate-limit/rate-limit.guard.ts
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(
    private readonly rateLimitService: RateLimitService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest<Request>()
    const res = context.switchToHttp().getResponse<Response>()
    const ip = req.ip ?? 'unknown'

    const rateLimitOptions = this.reflector.get<RateLimitOptions>(
      'rateLimitOptions',
      context.getHandler(),
    )

    try {
      const result = await this.rateLimitService.consume(
        ip,
        rateLimitOptions?.points,
        rateLimitOptions?.duration,
      )

      if (!result.allowed) {
        res.setHeader('Retry-After', Math.ceil(result.retryAfter / 1000) || 1)
        throw new HttpException('Too many requests', HttpStatus.TOO_MANY_REQUESTS)
      }

      return true
    } catch (error) {
      if (error instanceof HttpException) throw error
      return true // fail open
    }
  }
}
```

---

## Service

```ts
// src/shared/rate-limit/rate-limit.service.ts
@Injectable()
export class RateLimitService {
  private limiter: RateLimiterAbstract
  private customLimiters = new Map<string, RateLimiterAbstract>()
  private readonly defaultPoints: number
  private readonly defaultDuration: number

  constructor(envService: EnvService, redisService: RedisService) {
    // Uses RateLimiterRedis if Redis is ready, RateLimiterMemory otherwise
    // Custom per-endpoint limiters are cached in the Map
  }

  async consume(key: string, points?: number, duration?: number): Promise<ConsumeResult> {
    // If custom points/duration, creates or reuses a custom limiter
    // Returns { allowed: true, source } or { allowed: false, retryAfter, source }
  }
}
```

---

## Module

```ts
// src/shared/rate-limit/rate-limit.module.ts
@Module({
  imports: [CacheModule, EnvModule],
  providers: [RateLimitService],
  exports: [RateLimitService],
})
export class RateLimitModule {}
```

---

## Usage in controllers

Apply `@UseGuards(RateLimitGuard)` on endpoints that need rate limiting. Use `@RateLimit(points, duration)` to override defaults per endpoint.

```ts
@Post()
@UseGuards(RateLimitGuard)
@RateLimit(5, 60) // 5 requests per 60 seconds (stricter than global)
async handle(@Body() body: SignInDTO) { ... }
```

Without `@RateLimit()`, the guard uses global defaults from env vars (`RATE_LIMIT_POINTS`, `RATE_LIMIT_DURATION`).

---

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RATE_LIMIT_POINTS` | `100` | Max requests per duration window |
| `RATE_LIMIT_DURATION` | `60` | Duration window in seconds |

---

## Rules

- Rate limiting is per-IP, keyed by `req.ip`
- Redis is the primary store; in-memory is the automatic fallback
- **Fail open** — if Redis is down and in-memory fails, allow the request
- Always set `Retry-After` header on 429 responses
- Use `@RateLimit()` for sensitive endpoints (login, registration) with stricter limits
- Global defaults are sufficient for most endpoints
- Rate limit lives in `src/shared/` (not `src/infra/`) because it's a cross-cutting concern

---

## Anti-patterns

```ts
// ❌ rate limiting in use case
export class AuthenticateUserUseCase {
  constructor(private rateLimiter: RateLimitService) {} // never — rate limit is infra

// ❌ failing closed on unexpected errors
catch (error) {
  throw new HttpException('Service unavailable', 503) // fail open instead

// ❌ no Retry-After header
throw new HttpException('Too many requests', 429) // always set header first

// ❌ hardcoding limits instead of using env vars or decorator
if (requestCount > 10) { ... } // use RateLimitService
```
