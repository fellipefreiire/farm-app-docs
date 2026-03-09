# Health Check Pattern

`@nestjs/terminus`-based health check endpoint. Public, no auth required. Checks database and Redis connectivity.

---

## File locations

```
src/infra/http/controllers/health.controller.ts                 ← GET /health endpoint
src/infra/http/indicators/prisma-health.indicator.ts            ← database connectivity check
src/infra/http/indicators/redis-health.indicator.ts             ← Redis connectivity check
src/infra/http/dtos/common/health-check-response.dto.ts         ← Swagger response DTO
```

---

## Health controller

```ts
// src/infra/http/controllers/health.controller.ts
@ApiTags('Health')
@ServiceTag('health')
@Public()
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prisma: PrismaHealthIndicator,
    private redis: RedisHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  @ApiOperation({ summary: 'Check application health and its dependent services.' })
  @ApiOkResponse({ description: 'All services are operational.', type: HealthCheckResponseDto })
  @ApiServiceUnavailableResponse({ description: 'One or more services are unavailable.', type: HealthCheckResponseDto })
  check(): Promise<HealthCheckResult> {
    return this.health.check([
      () => this.prisma.isHealthy('database'),
      () => this.redis.isHealthy('redis'),
    ])
  }
}
```

---

## Health indicators

Each indicator implements an `isHealthy(key)` method that returns `{ [key]: { status: 'up' | 'down' } }`.

```ts
// src/infra/http/indicators/prisma-health.indicator.ts
@Injectable()
export class PrismaHealthIndicator {
  constructor(private prisma: PrismaService) {}

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      await this.prisma.$queryRawUnsafe('SELECT 1')
      return { [key]: { status: 'up' } }
    } catch {
      return { [key]: { status: 'down' } }
    }
  }
}
```

```ts
// src/infra/http/indicators/redis-health.indicator.ts
@Injectable()
export class RedisHealthIndicator {
  constructor(private readonly redisService: RedisService) {}

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      await this.redisService.ping()
      return { [key]: { status: 'up' } }
    } catch {
      return { [key]: { status: 'down' } }
    }
  }
}
```

---

## Response shape

```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  }
}
```

When a service is down, `status` is `"error"` and the failing service appears in the `error` object.

---

## Rules

- Health endpoint is always `@Public()` — monitoring tools must access it without auth
- Use `@ServiceTag('health')` for log filtering
- No versioning — `/health`, not `/v1/health`
- Each dependency gets its own health indicator class
- Indicators catch errors internally — never let a health check crash the app
- When adding a new dependency (e.g., external API), create a new indicator and add it to the `check()` array
- Uses `@nestjs/terminus` — dependency must be in `package.json`
