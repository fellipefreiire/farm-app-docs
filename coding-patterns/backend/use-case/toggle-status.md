# Use Case — Toggle Status

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
type Toggle<Entity>StatusUseCaseRequest = {
  id: string
  actorId: string
}

type Toggle<Entity>StatusUseCaseResponse = Either<
  <Entity>NotFoundError,
  {
    data: <Entity>
  }
>

/** Toggles the active status of a <entity>. */
@Injectable()
export class Toggle<Entity>StatusUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    id,
    actorId,
  }: Toggle<Entity>StatusUseCaseRequest): Promise<Toggle<Entity>StatusUseCaseResponse> {
    const entity = await this.<entity>Repository.findActiveById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    entity.toggleActive(actorId)

    await this.<entity>Repository.save(entity)

    return right({ data: entity })
  }
}
```

---

## Testing

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { Toggle<Entity>StatusUseCase } from '../toggle-<entity>-status'
import { make<Entity> } from 'test/factories/make-<entity>'
import { <Entity>NotFoundError } from '../errors/<entity>-not-found-error'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let sut: Toggle<Entity>StatusUseCase

describe('Toggle <Entity> Status', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    sut = new Toggle<Entity>StatusUseCase(inMemory<Entity>Repository)
  })

  it('should be able to toggle <entity> active status', async () => {
    // Arrange
    const entity = make<Entity>({ active: true })
    inMemory<Entity>Repository.items.push(entity)

    // Act
    const result = await sut.execute({
      id: entity.id.toString(),
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.data.active).toBe(false)
    }

    expect(inMemory<Entity>Repository.items[0].active).toBe(false)
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

**Rules:**
- Always use `InMemoryRepository` — never real database in unit tests
- AAA pattern: Arrange → Act → Assert
- One behavior per test — never assert multiple behaviors in one `it()`
- Variable name is `sut` (System Under Test) for the class being tested
- Use `result.isRight()` / `result.isLeft()` — never access `.value` before checking
- Assert both the return value AND the repository state when relevant
