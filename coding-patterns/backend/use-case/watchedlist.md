# Use Case — Multi-entity with WatchedList

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

When an aggregate has a WatchedList, the use case only interacts with the **aggregate repository**. WatchedList sync (new items, removed items) is handled inside `repository.save()` — the use case never touches the join repository directly.

```ts
@Injectable()
export class Create<Entity>UseCase {
  constructor(
    private <entity>Repository: <Entity>Repository,
    // ← no join repository injection — sync is handled inside repository.save()
  ) {}

  async execute({ actorId, ...request }: Create<Entity>UseCaseRequest) {
    const entityId = new UniqueEntityID()  // pre-generate so children can reference it

    const relatedList = new <Entity><Related>List([
      <Related>.create({ ...request.related, <entity>Id: entityId }),
    ])

    const entity = <Entity>.create({
      ...request,
      relatedItems: relatedList,
    }, actorId, entityId)

    await this.<entity>Repository.create(entity)
    // join table rows are created inside create() — no extra call needed

    return right({ data: entity })
  }
}
```

**On edit** — reconstruct the WatchedList from the request, pass to aggregate, call `save()`:

```ts
const relatedList = new <Entity><Related>List(
  request.relatedItems.map((item) =>
    <Related>.create({ ...item, <entity>Id: entity.id })
  )
)

entity.update({ ...request, relatedItems: relatedList }, actorId)

await this.<entity>Repository.save(entity)
// save() syncs getNewItems() + getRemovedItems() atomically via $transaction — no extra call needed
```

---
