# Backend Repository Pattern — Index

Navigation hub for farm-app backend repository patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree**.

## Files

| Variant | File | When to read |
|---|---|---|
| Interface (domain) | [interface.md](interface.md) | Writing the repository interface in the domain layer |
| Prisma implementation | [prisma.md](prisma.md) | Writing the Prisma implementation (infra layer) |
| InMemory implementation | [in-memory.md](in-memory.md) | Writing the InMemory implementation (test layer) |
| Pagination | [pagination.md](pagination.md) | Choosing offset vs cursor and wiring each |
| Transactions | [transactions.md](transactions.md) | When and how to use transactions |
| Join / WatchedList | [join-watchedlist.md](join-watchedlist.md) | Repositories for multi-entity (WatchedList) domains |
| N+1 prevention | [n-plus-one.md](n-plus-one.md) | How to avoid N+1 queries in repositories |

## File locations

```
src/domain/<domain>/application/repositories/<entity>-repository.ts   ← interface (abstract class)
src/infra/database/prisma/repositories/<domain>/prisma-<entity>.repository.ts  ← Prisma implementation
test/repositories/<domain>/in-memory-<entity>-repository.ts           ← in-memory implementation for tests
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), the repository interface moves under the subdomain: `src/domain/<domain>/<subdomain>/application/repositories/`. Infrastructure implementations and test repositories also use subdomain folders: `src/infra/database/prisma/repositories/<domain>/<subdomain>/` and `test/repositories/<domain>/<subdomain>/`.

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
