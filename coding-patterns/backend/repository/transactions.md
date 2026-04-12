# Repository — Transaction rules

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
