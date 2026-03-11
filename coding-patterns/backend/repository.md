# Repository Pattern

Repositories abstract persistence. The domain layer defines the interface (contract); infrastructure implements it. Use cases only depend on the interface — never on Prisma directly.

> For Prisma schema changes and migration workflow, see the "Database Migrations" section in CLAUDE.md.

---

## File locations

```
src/domain/<domain>/application/repositories/<entity>-repository.ts   ← interface (abstract class)
src/infra/database/prisma/repositories/<domain>/prisma-<entity>.repository.ts  ← Prisma implementation
test/repositories/<domain>/in-memory-<entity>-repository.ts           ← in-memory implementation for tests
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), the repository interface moves under the subdomain: `src/domain/<domain>/<subdomain>/application/repositories/`. Infrastructure implementations and test repositories also use subdomain folders: `src/infra/database/prisma/repositories/<domain>/<subdomain>/` and `test/repositories/<domain>/<subdomain>/`.

---

## 1. Repository interface (domain layer)

Defined as an `abstract class` — not an interface — so NestJS can use it as an injection token.

```ts
// src/domain/<domain>/application/repositories/<entity>-repository.ts
import type { PaginationParams, CursorPaginationParams, CursorPaginatedResult } from '@/core/repositories/pagination-params'
import type { <Entity> } from '../../enterprise/entities/<entity>'

export type <Entity>CursorParams = CursorPaginationParams & {
  // add domain-specific filters here if needed (same as offset variant)
}

export type <Entity>ListParams = PaginationParams & {
  search?: string        // free text — repository defines which columns are searched
  active?: boolean
  status?: '<StatusA>' | '<StatusB>'   // enum filter
  type?: ('<TypeA>' | '<TypeB>')[]     // multi-value enum — always array after normalization
  startDate?: Date
  endDate?: Date
  // add domain-specific filters here — each domain declares only what it needs
}

export abstract class <Entity>Repository {
  abstract findById(id: string): Promise<<Entity> | null>
  abstract findActiveById(id: string): Promise<<Entity> | null>
  abstract findManyByIds(ids: string[]): Promise<<Entity>[]>
  abstract list(params: <Entity>ListParams): Promise<[<Entity>[], number]>
  abstract listWithCursor(params: <Entity>CursorParams): Promise<CursorPaginatedResult<<Entity>>>
  abstract create(entity: <Entity>): Promise<void>
  abstract save(entity: <Entity>): Promise<void>
  abstract delete(id: string): Promise<void>  // hard delete — only when domain rules allow
}
```

**Rules:**
- Use `abstract class`, not `interface` — required for NestJS DI token
- Methods accept and return **domain entities** only — never Prisma types
- `list()` always returns a tuple `[items[], total]` — never an unbounded array
- `ListParams` extends `PaginationParams` and adds domain-specific filters
- Use `type` for `ListParams` — never `interface`

---

## 2. Prisma implementation (infrastructure layer)

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
        orderBy: { createdAt: 'desc' },
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

## 3. InMemory implementation (test layer)

```ts
// test/repositories/<domain>/in-memory-<entity>-repository.ts
import { DomainEvents } from '@/core/events/domain-events'
import type {
  <Entity>Repository,
  <Entity>ListParams,
  <Entity>CursorParams,
} from '@/domain/<domain>/application/repositories/<entity>-repository'
import type { CursorPaginatedResult } from '@/core/repositories/pagination-params'
import type { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'

export class InMemory<Entity>Repository implements <Entity>Repository {
  public items: <Entity>[] = []

  async findById(id: string): Promise<<Entity> | null> {
    return this.items.find((item) => item.id.toString() === id) ?? null
  }

  async findActiveById(id: string): Promise<<Entity> | null> {
    return this.items.find((item) => item.id.toString() === id && !item.deletedAt) ?? null
  }

  async findManyByIds(ids: string[]): Promise<<Entity>[]> {
    return this.items.filter((item) => ids.includes(item.id.toString()))
  }

  async list({
    page = 1,
    perPage = 20,
    search,
    active,
    status,
    type,
    startDate,
    endDate,
  }: <Entity>ListParams): Promise<[<Entity>[], number]> {
    let filtered = this.items.filter((i) => !i.deletedAt)  // always exclude soft-deleted

    if (active !== undefined) filtered = filtered.filter((i) => i.active === active)
    if (search) filtered = filtered.filter((i) =>
      i.name.toLowerCase().includes(search.toLowerCase())
      // mirror the same columns as the Prisma implementation
    )
    if (status) filtered = filtered.filter((i) => i.status === status)
    if (type?.length) filtered = filtered.filter((i) => type.includes(i.type))
    if (startDate) filtered = filtered.filter((i) => i.createdAt >= startDate)
    if (endDate) filtered = filtered.filter((i) => i.createdAt <= endDate)

    const total = filtered.length
    const items = filtered
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .slice((page - 1) * perPage, page * perPage)

    return [items, total]
  }

  async listWithCursor({
    cursor,
    limit = 20,
  }: <Entity>CursorParams): Promise<CursorPaginatedResult<<Entity>>> {
    let filtered = this.items
      .filter((i) => !i.deletedAt)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())

    if (cursor) {
      const cursorIndex = filtered.findIndex((i) => i.id.toString() === cursor)
      filtered = cursorIndex >= 0 ? filtered.slice(cursorIndex + 1) : []
    }

    const hasNextPage = filtered.length > limit
    const items = filtered.slice(0, limit)

    return {
      items,
      nextCursor: hasNextPage ? items[items.length - 1].id.toString() : null,
      count: items.length,
    }
  }

  async create(entity: <Entity>): Promise<void> {
    this.items.push(entity)
    DomainEvents.dispatchEventsForAggregate(entity.id)
  }

  async save(entity: <Entity>): Promise<void> {
    const index = this.items.findIndex((item) => item.id.equals(entity.id))
    this.items[index] = entity
    DomainEvents.dispatchEventsForAggregate(entity.id)
  }

  async delete(id: string): Promise<void> {
    this.items = this.items.filter((item) => item.id.toString() !== id)
  }
}
```

**Rules:**
- `items` is `public` — tests read it directly to assert state
- Must call `DomainEvents.dispatchEventsForAggregate()` — same as Prisma implementation
- Implement the same filtering logic as the Prisma version — tests should reflect real behavior
- Never use a real database in unit tests — always use InMemory
- ID comparison: use `.toString()` when comparing against a `string` parameter, use `.equals()` when comparing two `UniqueEntityID` instances

> `findById()` returns the entity regardless of soft-delete status — use for delete and audit. `findActiveById()` excludes soft-deleted records — use for standard reads, listings, edit, and toggle operations.

---

## Offset vs Cursor pagination — when to use each

| Strategy | When to use | Examples |
|----------|------------|---------|
| **Offset** (`page/perPage`) | Admin panels, tables with page numbers, small/medium datasets, data that rarely changes between page loads | Product list, user management, category list |
| **Cursor** (`cursor/limit`) | Feeds, logs, timelines, infinite scroll, large datasets, data that changes frequently (inserts between reads cause offset to skip/duplicate) | Audit log, notifications, activity feed, chat messages |

**Rule of thumb:** if the user navigates by page number → offset. If the user scrolls or loads "more" → cursor.

Core cursor types (defined in `src/core/repositories/pagination-params.ts` alongside existing offset types):

```ts
export type CursorPaginationParams = {
  cursor?: string
  limit?: number
}

export type CursorPaginatedResult<T> = {
  items: T[]
  nextCursor: string | null
  count: number
}
```

---

## Transaction rules

**Rule: any operation touching more than one entity MUST be wrapped in `$transaction`.**

| Scenario | Pattern |
|----------|---------|
| Batch operations on the same table (e.g. upsertMany) | `$transaction` inside the repository method |
| Paginated list (findMany + count) | `$transaction([findMany, count])` inside the repository |
| Delete with own children (prices, stock) | `onDelete: Cascade` in Prisma schema — atomic at DB level |
| Cross-domain coordination | Not recommended — model as domain events or accept eventual consistency |

**Own children cascade — Prisma schema:**
```prisma
model Price {
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  productId String
}
```

With `onDelete: Cascade` defined in the schema, deleting the parent automatically deletes all children in the same DB transaction — no application code needed.

**Note on cross-domain transactions:** If two aggregates from different domains truly require atomic coordination, introduce a `TransactionManager` abstract class in `src/core/database/`. This is an advanced pattern — evaluate carefully before adding. Most cross-domain cases are better modeled as domain events with compensating actions.

---

## Join repository (WatchedList domains)

When a domain uses a WatchedList, the **main repository** is responsible for syncing the join table inside `save()`. The join repository exists only for reads.

**Join repository — reads only:**

```ts
// Abstract class — read only
export abstract class <Parent><Child>Repository {
  abstract findManyBy<Parent>Id(<parent>Id: string): Promise<<Child>[]>
}

// Prisma implementation
@Injectable()
export class Prisma<Parent><Child>Repository implements <Parent><Child>Repository {
  constructor(private prisma: PrismaService) {}

  async findManyBy<Parent>Id(<parent>Id: string): Promise<<Child>[]> {
    const records = await this.prisma.<parentChild>.findMany({
      where: { <parent>Id },
    })
    return records.map(Prisma<Child>Mapper.toDomain)
  }
}

// InMemory implementation
export class InMemory<Parent><Child>Repository implements <Parent><Child>Repository {
  public items: <Child>[] = []

  async findManyBy<Parent>Id(<parent>Id: string): Promise<<Child>[]> {
    return this.items.filter((i) => i.<parent>Id.toString() === <parent>Id)
  }
}
```

**Main repository — injects join repository, owns WatchedList sync:**

```ts
@Injectable()
export class Prisma<Parent>Repository implements <Parent>Repository {
  constructor(
    private prisma: PrismaService,
    private <parent><child>Repository: <Parent><Child>Repository,  // for reads
  ) {}

  // findById/findActiveById must include the children relation for WatchedList reconstruction
  async findById(id: string): Promise<<Parent> | null> {
    const record = await this.prisma.<parent>.findUnique({
      where: { id },
      include: { <children>: true },
    })
    if (!record) return null
    return Prisma<Parent>Mapper.toDomain(record)
  }

  async findActiveById(id: string): Promise<<Parent> | null> {
    const record = await this.prisma.<parent>.findUnique({
      where: { id, deletedAt: null },
      include: { <children>: true },
    })
    if (!record) return null
    return Prisma<Parent>Mapper.toDomain(record)
  }

  // save() syncs the WatchedList inside a single $transaction
  async save(entity: <Parent>): Promise<void> {
    const newItems = entity.<children>.getNewItems()
    const removedItems = entity.<children>.getRemovedItems()

    await this.prisma.$transaction([
      this.prisma.<parent>.update({
        where: { id: entity.id.toString() },
        data: Prisma<Parent>Mapper.toPrismaUpdate(entity),
      }),
      ...newItems.map((item) =>
        this.prisma.<parentChild>.upsert({
          where: { id: item.id.toString() },
          create: Prisma<Child>Mapper.toPrisma(item),
          update: Prisma<Child>Mapper.toPrismaUpdate(item),
        })
      ),
      ...removedItems.map((item) =>
        this.prisma.<parentChild>.delete({
          where: { id: item.id.toString() },
        })
      ),
    ])

    DomainEvents.dispatchEventsForAggregate(entity.id)
  }
}
```

**InMemory main repository — mirrors the same sync logic:**

```ts
export class InMemory<Parent>Repository implements <Parent>Repository {
  public items: <Parent>[] = []

  constructor(public <parent><child>Repository: InMemory<Parent><Child>Repository) {}

  async save(entity: <Parent>): Promise<void> {
    const index = this.items.findIndex((i) => i.id.equals(entity.id))
    this.items[index] = entity

    const newItems = entity.<children>.getNewItems()
    const removedItems = entity.<children>.getRemovedItems()

    // mirror Prisma sync — add new, remove deleted
    this.<parent><child>Repository.items = [
      ...this.<parent><child>Repository.items.filter(
        (i) => !removedItems.some((r) => r.id.equals(i.id))
      ),
      ...newItems,
    ]

    DomainEvents.dispatchEventsForAggregate(entity.id)
  }
}
```

**Rules:**
- Join repository exposes reads only — never write methods
- `save()` in the main repository always uses `$transaction` to keep parent update + child sync atomic
- `getNewItems()` and `getRemovedItems()` come from the WatchedList — never sync the full collection
- InMemory must mirror the same sync logic as Prisma — tests must reflect real behavior
- The join repository is injected into the main repository for reads (e.g. loading attachments when fetching an entity)

---

## N+1 query prevention

N+1 happens when code fetches a list of entities and then queries a relation for each one individually. In Prisma, prevent this by loading relations in the original query.

**Correct — use `include` to load relations in a single query:**

```ts
// Loading parent with children — 1 query
const records = await this.prisma.order.findMany({
  where,
  include: { items: true },
})
```

**Correct — batch load related entities with `findMany` + `in`:**

```ts
// When relations are in a different repository, batch by IDs — 2 queries total
const orders = await this.prisma.order.findMany({ where })
const customerIds = [...new Set(orders.map((o) => o.customerId))]
const customers = await this.prisma.customer.findMany({
  where: { id: { in: customerIds } },
})
```

**Anti-patterns:**

```ts
// ❌ N+1 — querying inside a loop
const orders = await this.prisma.order.findMany({ where })
for (const order of orders) {
  const customer = await this.prisma.customer.findUnique({
    where: { id: order.customerId },  // 1 query per order
  })
}

// ❌ N+1 — fetching relation separately for each item
const products = await this.prisma.product.findMany({ where })
const result = await Promise.all(
  products.map(async (p) => ({
    ...p,
    prices: await this.prisma.price.findMany({ where: { productId: p.id } }),
  }))
)

// ✅ fix — use include
const products = await this.prisma.product.findMany({
  where,
  include: { prices: true },
})
```

**Rules:**
- Never query inside a loop — use `include` or batch with `findMany` + `in`
- Use `include` when the relation belongs to the same aggregate
- Use `findMany` + `in` when loading from a different repository or domain
- `$transaction([findMany, count])` for paginated lists is not N+1 — it's 2 queries by design

---

## Anti-patterns

```ts
// ❌ interface instead of abstract class
export interface <Entity>Repository { ... }  // can't be used as NestJS DI token

// ❌ returning Prisma types from repository
async findById(id: string): Promise<PrismaEntity | null>  // always return domain entity

// ❌ building Prisma objects inline in repository
await this.prisma.<entity>.create({
  data: { name: entity.name, ... }  // use mapper instead
})

// ❌ forgetting DomainEvents dispatch
async create(entity): Promise<void> {
  await this.prisma.<entity>.create({ data })
  // missing: DomainEvents.dispatchEventsForAggregate(entity.id)
}

// ❌ unbounded list
abstract findAll(): Promise<<Entity>[]>  // always paginate

// ❌ multi-entity operations without $transaction
async upsertMany(items: <Child>[]): Promise<void> {
  for (const item of items) {
    await this.prisma.<child>.upsert({ ... })  // no transaction — partial failure leaves inconsistent state
  }
}

// ❌ relying on application-level cascade instead of Prisma schema
async delete(id: string): Promise<void> {
  await this.prisma.price.deleteMany({ where: { productId: id } })  // should be onDelete: Cascade
  await this.prisma.product.delete({ where: { id } })
}
```
