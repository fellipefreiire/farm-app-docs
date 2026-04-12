# Repository — Prisma implementation

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
// src/infra/database/prisma/repositories/<domain>/prisma-<entity>.repository.ts
import { Injectable } from '@nestjs/common'
import { DomainEvents } from '@/core/events/domain-events'
import type { Prisma } from '@/generated/prisma/client'
import {
  <Entity>Repository,
  type <Entity>ListParams,
  type <Entity>CursorParams,
} from '@/domain/<domain>/application/repositories/<entity>-repository'
import type { CursorPaginatedResult } from '@/core/repositories/pagination-params'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import { PrismaService } from '../../prisma.service'
import { Prisma<Entity>Mapper } from '../../mappers/<domain>/prisma-<entity>.mapper'

@Injectable()
export class Prisma<Entity>Repository implements <Entity>Repository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<<Entity> | null> {
    const record = await this.prisma.<entity>.findUnique({ where: { id } })
    if (!record) return null
    return Prisma<Entity>Mapper.toDomain(record)
  }

  async findActiveById(id: string): Promise<<Entity> | null> {
    const record = await this.prisma.<entity>.findUnique({
      where: { id, deletedAt: null },
    })
    if (!record) return null
    return Prisma<Entity>Mapper.toDomain(record)
  }

  async findManyByIds(ids: string[]): Promise<<Entity>[]> {
    const records = await this.prisma.<entity>.findMany({
      where: { id: { in: ids } },
      orderBy: { name: 'asc' },
    })
    return records.map(Prisma<Entity>Mapper.toDomain)
  }

  async list({
    page = 1,
    perPage = 20,
    sort = 'createdAt',
    order = 'desc',
    search,
    active,
    status,
    type,
    startDate,
    endDate,
  }: <Entity>ListParams): Promise<[<Entity>[], number]> {
    const where: Prisma.<Entity>WhereInput = {
      deletedAt: null,  // always exclude soft-deleted records
    }

    // boolean filter
    if (active !== undefined) where.active = active

    // free text search across multiple columns
    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { reference: { contains: search, mode: 'insensitive' } },
        // add any other searchable columns here
      ]
    }

    // enum filter (single)
    if (status) where.status = status

    // multi-value enum filter (array)
    if (type?.length) where.type = { in: type }

    // date range
    if (startDate || endDate) {
      where.createdAt = {
        ...(startDate && { gte: startDate }),
        ...(endDate && { lte: endDate }),
      }
    }

    const [records, total] = await this.prisma.$transaction([
      this.prisma.<entity>.findMany({
        where,
        orderBy: { [sort]: order },
        skip: (page - 1) * perPage,
        take: perPage,
      }),
      this.prisma.<entity>.count({ where }),
    ])

    return [records.map(Prisma<Entity>Mapper.toDomain), total]
  }

  async listWithCursor({
    cursor,
    limit = 20,
  }: <Entity>CursorParams): Promise<CursorPaginatedResult<<Entity>>> {
    const records = await this.prisma.<entity>.findMany({
      take: limit + 1,
      ...(cursor && {
        cursor: { id: cursor },
        skip: 1,
      }),
      where: { deletedAt: null },
      orderBy: { createdAt: 'desc' },
    })

    const hasNextPage = records.length > limit
    const items = hasNextPage ? records.slice(0, -1) : records

    return {
      items: items.map(Prisma<Entity>Mapper.toDomain),
      nextCursor: hasNextPage ? items[items.length - 1].id : null,
      count: items.length,
    }
  }

  async create(entity: <Entity>): Promise<void> {
    const data = Prisma<Entity>Mapper.toPrisma(entity)
    await this.prisma.<entity>.create({ data })
    DomainEvents.dispatchEventsForAggregate(entity.id)
  }

  async save(entity: <Entity>): Promise<void> {
    await this.prisma.<entity>.update({
      where: { id: entity.id.toString() },
      data: Prisma<Entity>Mapper.toPrismaUpdate(entity),
    })
    DomainEvents.dispatchEventsForAggregate(entity.id)
  }

  async delete(id: string): Promise<void> {
    await this.prisma.<entity>.delete({ where: { id } })
  }
}
```

**Rules:**
- Always call `DomainEvents.dispatchEventsForAggregate()` after `create()` and `save()`
- Use `$transaction([findMany, count])` for paginated list queries — avoids race conditions
- Pass data through the mapper — never build Prisma objects inline in the repository
- `save()` updates only mutable fields — never re-sends `id` or `createdAt`

---

## Testing

> Repository cache test is E2E because it exercises the Redis integration — keep it next to the Prisma implementation pattern.

Repository cache tests verify that the Prisma repository correctly populates and invalidates Redis cache. These are E2E tests that require a running database and Redis.

```ts
// src/infra/database/prisma/repositories/<domain>/__tests__/prisma-<entity>-repository.e2e-spec.ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import { <Entity>DatabaseModule } from '../<entity>-database.module'
import { CacheModule } from '@/infra/cache/cache.module'
import { <Entity>Factory } from 'test/factories/make-<entity>'
import { CacheRepository } from '@/infra/cache/cache.repository'
import { <Entity>sRepository } from '@/domain/<domain>/application/repositories/<entity>s-repository'
import { PrismaService } from '../../../prisma.service'

describe('Prisma <Entity>s Repository (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let <entity>Factory: <Entity>Factory
  let cacheRepository: CacheRepository
  let <entity>sRepository: <Entity>sRepository

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule, <Entity>DatabaseModule, CacheModule],
      providers: [<Entity>Factory],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    <entity>Factory = moduleRef.get(<Entity>Factory)
    cacheRepository = moduleRef.get(CacheRepository)
    <entity>sRepository = moduleRef.get(<Entity>sRepository)

    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "<table_name>" CASCADE')
  })

  it('should cache <entity> details on findById', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    // Act
    await <entity>sRepository.findById(id)

    // Assert
    const cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).not.toBeNull()
    expect(JSON.parse(cached!)).toEqual(
      expect.objectContaining({ id }),
    )
  })

  it('should return cached <entity> details on subsequent calls', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    // Act — first call (cache miss)
    let cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).toBeNull()

    await <entity>sRepository.findById(id)
    cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).not.toBeNull()

    // Act — second call (cache hit)
    const details = await <entity>sRepository.findById(id)

    // Assert
    expect(JSON.parse(cached!)).toEqual(
      expect.objectContaining({ id: details?.id.toString() }),
    )
  })

  it('should invalidate <entity> cache on save', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    await cacheRepository.set(
      `<entity>:${id}:details`,
      JSON.stringify({ empty: true }),
    )

    // Act
    await <entity>sRepository.save(entity)

    // Assert
    const cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).toBeNull()
  })
})
```

**Rules:**
- Test three cache behaviors: **populate on read**, **hit on subsequent read**, **invalidate on write**
- Test cache for each lookup method (`findById`, `findByEmail`, etc.)
- Use `CacheRepository` to verify cache state directly — don't rely on timing
- These are E2E tests (`.e2e-spec.ts`) — they require real DB and Redis
- Truncate tables in `beforeEach`

---
