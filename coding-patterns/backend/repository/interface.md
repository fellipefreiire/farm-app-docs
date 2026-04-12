# Repository — Interface (domain layer)

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Defined as an `abstract class` — not an interface — so NestJS can use it as an injection token.

```ts
// src/domain/<domain>/application/repositories/<entity>-repository.ts
import type {
  PaginationParams,
  SortParams,
  CursorPaginationParams,
  CursorPaginatedResult,
} from '@/core/repositories/pagination-params'
import type { <Entity> } from '../../enterprise/entities/<entity>'

export type <Entity>SortField = 'name' | 'createdAt'  // only sortable columns — domain decides

export type <Entity>CursorParams = CursorPaginationParams & {
  // add domain-specific filters here if needed (same as offset variant)
}

export type <Entity>ListParams = PaginationParams &
  SortParams<<Entity>SortField> & {
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
