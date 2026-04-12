# Repository — N+1 query prevention

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
