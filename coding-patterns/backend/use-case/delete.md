# Use Case — Delete

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

The delete use case checks for domain interactions and decides between hard delete and soft delete. The decision rule is defined in `docs/rules/<domain>.md`.

```ts
type Delete<Entity>UseCaseRequest = {
  id: string
  actorId: string
}

type Delete<Entity>UseCaseResponse = Either<
  <Entity>NotFoundError,
  null
>

@Injectable()
export class Delete<Entity>UseCase {
  constructor(
    private <entity>Repository: <Entity>Repository,
    private <related>Repository: <Related>Repository, // injected to check interactions
  ) {}

  async execute({ id, actorId }: Delete<Entity>UseCaseRequest): Promise<Delete<Entity>UseCaseResponse> {
    const entity = await this.<entity>Repository.findById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    // Check for external references that block deletion
    // (what counts as "interaction" is defined per domain in docs/rules/<domain>.md)
    const hasExternalReferences = await this.<related>Repository.existsFor<Entity>(id)

    if (hasExternalReferences) {
      // Soft delete — entity must be preserved for audit/reporting
      entity.softDelete(actorId)
      await this.<entity>Repository.save(entity)
    } else {
      // Hard delete — no external references, safe to remove permanently
      // Own children (e.g. prices, stock) are deleted automatically via
      // onDelete: Cascade defined in the Prisma schema — no code needed here
      await this.<entity>Repository.delete(id)
    }

    return right(null)
  }
}
```

**Rules:**
- What counts as "an interaction" is domain-specific — document in `docs/rules/<domain>.md`
- Own children (prices, stock) are deleted automatically via `onDelete: Cascade` in the Prisma schema — no application code needed
- External references (orders, references from other domains) trigger soft delete instead
- Soft delete uses `entity.softDelete(actorId)` + `repository.save()` — same as any mutation
- Hard delete uses `repository.delete(id)` — no domain event needed (entity is gone)
- The user never sees the distinction — the endpoint always responds the same way

---

## Testing

The delete use case injects a second repository to check for external references (see `use-case.md` → Delete). The InMemory implementation of the related repository must include the `existsFor<Entity>()` method.

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { InMemory<Related>Repository } from 'test/repositories/<related-domain>/in-memory-<related>-repository'
import { Delete<Entity>UseCase } from '../delete-<entity>'
import { make<Entity> } from 'test/factories/make-<entity>'
import { <Entity>NotFoundError } from '../errors/<entity>-not-found-error'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let inMemory<Related>Repository: InMemory<Related>Repository
let sut: Delete<Entity>UseCase

describe('Delete <Entity>', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    inMemory<Related>Repository = new InMemory<Related>Repository()
    sut = new Delete<Entity>UseCase(
      inMemory<Entity>Repository,
      inMemory<Related>Repository,
    )
  })

  it('should hard delete when no external references exist', async () => {
    // Arrange
    const entity = make<Entity>()
    inMemory<Entity>Repository.items.push(entity)
    // inMemory<Related>Repository has no items → existsFor<Entity> returns false

    // Act
    const result = await sut.execute({
      id: entity.id.toString(),
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isRight()).toBe(true)
    expect(inMemory<Entity>Repository.items).toHaveLength(0)
  })

  it('should soft delete when external references exist', async () => {
    // Arrange
    const entity = make<Entity>()
    inMemory<Entity>Repository.items.push(entity)
    // Seed a related record so existsFor<Entity> returns true
    inMemory<Related>Repository.items.push(make<Related>({ <entity>Id: entity.id }))

    // Act
    const result = await sut.execute({
      id: entity.id.toString(),
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isRight()).toBe(true)
    expect(inMemory<Entity>Repository.items).toHaveLength(1)
    expect(inMemory<Entity>Repository.items[0].deletedAt).toBeTruthy()
  })

  it('should return error when <entity> does not exist', async () => {
    // Act
    const result = await sut.execute({
      id: 'non-existent',
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isLeft()).toBe(true)
    expect(result.value).toBeInstanceOf(<Entity>NotFoundError)
  })
})
```
