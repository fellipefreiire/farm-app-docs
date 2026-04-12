# Use Case — List (cursor pagination)

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import type { CursorPaginatedResult } from '@/core/repositories/pagination-params'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type List<Entity>sCursorUseCaseRequest = {
  cursor?: string
  limit?: number
}

type List<Entity>sCursorUseCaseResponse = Either<
  null,
  CursorPaginatedResult<<Entity>>
>

/** Lists <entity>s with cursor-based pagination. */
@Injectable()
export class List<Entity>sCursorUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    cursor,
    limit = 20,
  }: List<Entity>sCursorUseCaseRequest): Promise<List<Entity>sCursorUseCaseResponse> {
    const result = await this.<entity>Repository.listWithCursor({ cursor, limit })
    return right(result)
  }
}
```

---

## Testing

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { List<Entity>sCursorUseCase } from '../list-<entity>s-cursor'
import { make<Entity> } from 'test/factories/make-<entity>'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let sut: List<Entity>sCursorUseCase

describe('List <Entity>s (Cursor)', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    sut = new List<Entity>sCursorUseCase(inMemory<Entity>Repository)
  })

  it('should return first page with nextCursor when more items exist', async () => {
    // Arrange
    for (let i = 0; i < 25; i++) {
      inMemory<Entity>Repository.items.push(
        make<Entity>({ createdAt: new Date(2024, 0, 25 - i) }),
      )
    }

    // Act
    const result = await sut.execute({ limit: 20 })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.items).toHaveLength(20)
      expect(result.value.count).toBe(20)
      expect(result.value.nextCursor).not.toBeNull()
    }
  })

  it('should return items after cursor (second page)', async () => {
    // Arrange
    for (let i = 0; i < 25; i++) {
      inMemory<Entity>Repository.items.push(
        make<Entity>({ createdAt: new Date(2024, 0, 25 - i) }),
      )
    }

    const firstPage = await sut.execute({ limit: 20 })
    const cursor = firstPage.isRight() ? firstPage.value.nextCursor : null

    // Act
    const result = await sut.execute({ cursor: cursor!, limit: 20 })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.items).toHaveLength(5)
      expect(result.value.count).toBe(5)
      expect(result.value.nextCursor).toBeNull()
    }
  })

  it('should return null nextCursor on last page', async () => {
    // Arrange
    for (let i = 0; i < 10; i++) {
      inMemory<Entity>Repository.items.push(make<Entity>())

    // Act
    const result = await sut.execute({ limit: 20 })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.items).toHaveLength(10)
      expect(result.value.count).toBe(10)
      expect(result.value.nextCursor).toBeNull()
    }
  })
})
```
