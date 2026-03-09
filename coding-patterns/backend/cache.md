# Cache Pattern

Redis cache for frequently accessed data. Cache is infrastructure — domain and application layers never reference it directly.

---

## File locations

```
src/infra/cache/cache.repository.ts                ← abstract cache interface
src/infra/cache/redis/redis.service.ts             ← Redis client (extends ioredis)
src/infra/cache/redis/redis-cache-repository.ts    ← Redis implementation
src/infra/cache/cache.module.ts                    ← NestJS module
test/cache/fake-cache.ts                           ← test stub (FakeCacheService)
```

---

## Cache interface

```ts
// src/infra/cache/cache.repository.ts
export abstract class CacheRepository {
  abstract set(key: string, value: string, ttlInSeconds?: number): Promise<void>
  abstract get(key: string): Promise<string | null>
  abstract del(key: string | string[]): Promise<number>
  abstract keys(pattern: string): Promise<string[]>
}
```

---

## Redis service

```ts
// src/infra/cache/redis/redis.service.ts
import { Injectable, OnModuleDestroy } from '@nestjs/common'
import { Redis } from 'ioredis'
import { EnvService } from '@/infra/env/env.service'

@Injectable()
export class RedisService extends Redis implements OnModuleDestroy {
  constructor(envService: EnvService) {
    super({
      host: envService.get('REDIS_HOST'),
      port: envService.get('REDIS_PORT'),
      db: envService.get('REDIS_DB'),
      connectTimeout: envService.get('REDIS_COMMAND_TIMEOUT'),
    })
    // error logging
  }

  onModuleDestroy() {
    return this.disconnect()
  }
}
```

---

## Redis implementation

```ts
// src/infra/cache/redis/redis-cache-repository.ts
import { Injectable } from '@nestjs/common'
import { CacheRepository } from '../cache.repository'
import { RedisService } from './redis.service'

@Injectable()
export class RedisCacheRepository implements CacheRepository {
  private EXPIRATION_SECONDS = 60 * 10 // 10 minutes

  constructor(private redis: RedisService) {}

  async set(key: string, value: string, ttlInSeconds?: number): Promise<void> {
    await this.redis.set(key, value, 'EX', ttlInSeconds || this.EXPIRATION_SECONDS)
  }

  async get(key: string): Promise<string | null> {
    return this.redis.get(key)
  }

  async del(key: string | string[]): Promise<number> {
    const keys = Array.isArray(key) ? key : [key]
    if (keys.length === 0) return 0
    return this.redis.del(...keys)
  }

  async keys(pattern: string): Promise<string[]> {
    return this.redis.keys(pattern)
  }
}
```

---

## Test stub

```ts
// test/cache/fake-cache.ts
import { CacheRepository } from '@/infra/cache/cache.repository'

export class FakeCacheService implements CacheRepository {
  private store = new Map<string, { value: string; expiresAt?: number }>()

  async set(key: string, value: string, ttlInSeconds?: number): Promise<void> {
    this.store.set(key, {
      value,
      expiresAt: ttlInSeconds ? Date.now() + ttlInSeconds * 1000 : undefined,
    })
  }

  async get(key: string): Promise<string | null> {
    const entry = this.store.get(key)
    if (!entry) return null
    if (entry.expiresAt && Date.now() > entry.expiresAt) {
      this.store.delete(key)
      return null
    }
    return entry.value
  }

  async del(key: string | string[]): Promise<number> {
    const keys = Array.isArray(key) ? key : [key]
    let count = 0
    for (const k of keys) {
      if (this.store.delete(k)) count++
    }
    return count
  }

  async keys(pattern: string): Promise<string[]> {
    const regex = new RegExp(
      '^' + pattern.replace(/[.+^${}()|[\]\\]/g, '\\$&').replace(/\*/g, '.*').replace(/\?/g, '.') + '$',
    )
    return Array.from(this.store.keys()).filter((k) => regex.test(k))
  }
}
```

---

## Cache module

```ts
// src/infra/cache/cache.module.ts
import { Module } from '@nestjs/common'
import { EnvModule } from '@/infra/env/env.module'
import { CacheRepository } from './cache.repository'
import { RedisService } from './redis/redis.service'
import { RedisCacheRepository } from './redis/redis-cache-repository'

@Module({
  imports: [EnvModule],
  providers: [
    RedisService,
    { provide: CacheRepository, useClass: RedisCacheRepository },
  ],
  exports: [CacheRepository, RedisService],
})
export class CacheModule {}
```

---

## Usage in repositories

Cache is consumed by Prisma repositories, not by use cases:

```ts
// src/infra/database/prisma/repositories/<domain>/prisma-<entity>.repository.ts
@Injectable()
export class Prisma<Entity>Repository implements I<Entity>Repository {
  constructor(
    private prisma: PrismaService,
    private cache: CacheRepository,
  ) {}

  async findById(id: string): Promise<Entity | null> {
    const cacheKey = `entity:${id}`
    const cached = await this.cache.get(cacheKey)

    if (cached) {
      return <Entity>Mapper.toDomain(JSON.parse(cached))
    }

    const raw = await this.prisma.<entity>.findUnique({ where: { id } })
    if (!raw) return null

    await this.cache.set(cacheKey, JSON.stringify(raw), 3600) // 1 hour TTL

    return <Entity>Mapper.toDomain(raw)
  }

  async save(entity: Entity): Promise<void> {
    const data = <Entity>Mapper.toPrisma(entity)
    await this.prisma.<entity>.upsert({
      where: { id: entity.id.toString() },
      update: data,
      create: data,
    })

    // invalidate cache after mutation
    await this.cache.del(`entity:${entity.id.toString()}`)
  }
}
```

---

## Cache key conventions

| Pattern | Example | Usage |
|---------|---------|-------|
| `entity:{id}` | `order:550e8400...` | Single entity lookup |
| `list:{domain}:{hash}` | `list:orders:abc123` | Paginated list (hash of query params) |
| `count:{domain}` | `count:orders` | Total count for pagination |

---

## Rules

- Cache is infrastructure — use cases and domain layer never reference `CacheRepository`
- Always use the abstract `CacheRepository` interface, never import Redis directly
- Always set a TTL — never cache indefinitely (default: 1 hour for entities)
- Invalidate cache on mutation — `del` the key after `save`, `update`, or `delete`
- Cache serialization is JSON — use `JSON.stringify`/`JSON.parse`
- In tests, use `FakeCacheService` — never connect to Redis in unit tests
- Cache miss is not an error — always fall back to database

---

## Anti-patterns

```ts
// ❌ cache in use case
export class FindOrderUseCase {
  constructor(private cache: CacheRepository) {}  // never — cache is infra
}

// ❌ no TTL
await this.cache.set(key, value)  // always set TTL

// ❌ no invalidation after mutation
await this.prisma.order.update({ ... })
// missing: await this.cache.del(`order:${id}`)

// ❌ importing Redis directly in repository
import { Redis } from 'ioredis'  // use CacheRepository interface

// ❌ caching in domain layer
export class Order extends Entity<OrderProps> {
  cache() { ... }  // domain never knows about cache
}
```
