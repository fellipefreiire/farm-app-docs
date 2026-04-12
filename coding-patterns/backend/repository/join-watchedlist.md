# Repository — Join (WatchedList domains)

> Part of the farm-app backend repository pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
