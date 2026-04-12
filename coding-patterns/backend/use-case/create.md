# Use Case — Create

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type Create<Entity>UseCaseRequest = {
  name: string
  optionalField?: string
  actorId: string
}

type Create<Entity>UseCaseResponse = Either<
  null,
  {
    data: <Entity>
  }
>

/** Creates a new <entity>. */
@Injectable()
export class Create<Entity>UseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    actorId,
    ...data
  }: Create<Entity>UseCaseRequest): Promise<Create<Entity>UseCaseResponse> {
    const entity = <Entity>.create(data, actorId)

    await this.<entity>Repository.create(entity)

    return right({ data: entity })
  }
}
```

---

## Testing

> Default test template for create use cases. The "edit variants" subsection covers edit use cases, which are structurally similar to create.

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { <Action><Entity>UseCase } from '../<action>-<entity>'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let sut: <Action><Entity>UseCase

describe('<Action> <Entity>', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    sut = new <Action><Entity>UseCase(inMemory<Entity>Repository)
  })

  it('should be able to <action> a <entity>', async () => {
    // Arrange
    const input = {
      name: 'test name',
      actorId: 'actor-1',
    }

    // Act
    const result = await sut.execute(input)

    // Assert
    expect(result.isRight()).toBe(true)
    expect(inMemory<Entity>Repository.items).toHaveLength(1)

    if (result.isRight()) {
      expect(result.value.data.name).toBe('test name')
    }
  })

  it('should return error when <entity> does not exist', async () => {
    // Act
    const result = await sut.execute({ id: 'non-existent', actorId: 'actor-1' })

    // Assert
    expect(result.isLeft()).toBe(true)
    expect(result.value).toBeInstanceOf(<Entity>NotFoundError)
  })
})
```

### Testing edit variants

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { Edit<Entity>UseCase } from '../edit-<entity>'
import { make<Entity> } from 'test/factories/make-<entity>'
import { <Entity>NotFoundError } from '../errors/<entity>-not-found-error'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let sut: Edit<Entity>UseCase

describe('Edit <Entity>', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    sut = new Edit<Entity>UseCase(inMemory<Entity>Repository)
  })

  it('should be able to edit a <entity>', async () => {
    // Arrange
    const entity = make<Entity>({ name: 'old name' })
    inMemory<Entity>Repository.items.push(entity)

    // Act
    const result = await sut.execute({
      id: entity.id.toString(),
      name: 'new name',
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.data.name).toBe('new name')
    }

    expect(inMemory<Entity>Repository.items[0].name).toBe('new name')
  })

  it('should return error when <entity> does not exist', async () => {
    // Act
    const result = await sut.execute({
      id: 'non-existent',
      name: 'new name',
      actorId: 'actor-1',
    })

    // Assert
    expect(result.isLeft()).toBe(true)
    expect(result.value).toBeInstanceOf(<Entity>NotFoundError)
  })
})
```
