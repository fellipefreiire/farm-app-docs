# Repository — InMemory implementation (tests)

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
    sort = 'createdAt',
    order = 'desc',
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

    filtered.sort((a, b) => {
      const aVal = a[sort]
      const bVal = b[sort]

      if (aVal instanceof Date && bVal instanceof Date) {
        return order === 'asc'
          ? aVal.getTime() - bVal.getTime()
          : bVal.getTime() - aVal.getTime()
      }

      const aStr = String(aVal)
      const bStr = String(bVal)
      return order === 'asc'
        ? aStr.localeCompare(bStr)
        : bStr.localeCompare(aStr)
    })

    const items = filtered.slice((page - 1) * perPage, page * perPage)

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
