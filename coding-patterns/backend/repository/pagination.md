# Repository — Offset vs Cursor pagination

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

| Strategy | When to use | Examples |
|----------|------------|---------|
| **Offset** (`page/perPage`) | Admin panels, tables with page numbers, small/medium datasets, data that rarely changes between page loads | Product list, user management, category list |
| **Cursor** (`cursor/limit`) | Feeds, logs, timelines, infinite scroll, large datasets, data that changes frequently (inserts between reads cause offset to skip/duplicate) | Audit log, notifications, activity feed, chat messages |

**Rule of thumb:** if the user navigates by page number → offset. If the user scrolls or loads "more" → cursor.

Core types (defined in `src/core/repositories/pagination-params.ts`):

```ts
export type SortOrder = 'asc' | 'desc'

export type SortParams<TAllowedFields extends string = string> = {
  sort?: TAllowedFields
  order?: SortOrder
}

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

`SortParams` is generic — each domain restricts allowed sort fields via the type parameter (e.g. `SortParams<'name' | 'createdAt'>`). This prevents invalid sort fields at the type level.

---
