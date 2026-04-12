# Use Case — List (offset pagination)

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import type { PaginationMeta } from '@/core/repositories/pagination-params'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type List<Entity>sUseCaseRequest = {
  page?: number
  perPage?: number
  search?: string                        // free text — passed directly to repository
  active?: boolean
  status?: '<StatusA>' | '<StatusB>'    // domain-specific filters
  type?: ('<TypeA>' | '<TypeB>')[]      // always array — normalized in controller
  startDate?: Date
  endDate?: Date
}

type List<Entity>sUseCaseResponse = Either<
  null,
  {
    data: <Entity>[]
    meta: PaginationMeta
  }
>

/** Lists <entity>s with pagination and optional search. */
@Injectable()
export class List<Entity>sUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    page = 1,
    perPage = 20,
    search,
    active,
    status,
    type,
    startDate,
    endDate,
  }: List<Entity>sUseCaseRequest): Promise<List<Entity>sUseCaseResponse> {
    const [items, total] = await this.<entity>Repository.list({ page, perPage, search, active, status, type, startDate, endDate })

    const totalPages = Math.ceil(total / perPage)

    return right({
      data: items,
      meta: {
        total,
        perPage,
        totalPages,
        currentPage: page,
        nextPage: page < totalPages ? page + 1 : null,
        previousPage: page > 1 ? page - 1 : null,
      },
    })
  }
}
```

---

## Testing

```ts
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { List<Entity>sUseCase } from '../list-<entity>s'
import { make<Entity> } from 'test/factories/make-<entity>'

let inMemory<Entity>Repository: InMemory<Entity>Repository
let sut: List<Entity>sUseCase

describe('List <Entity>s', () => {
  beforeEach(() => {
    inMemory<Entity>Repository = new InMemory<Entity>Repository()
    sut = new List<Entity>sUseCase(inMemory<Entity>Repository)
  })

  it('should be able to list <entity>s with pagination', async () => {
    // Arrange
    for (let i = 0; i < 25; i++) {
      inMemory<Entity>Repository.items.push(make<Entity>())
    }

    // Act
    const result = await sut.execute({ page: 1, perPage: 20 })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.data).toHaveLength(20)
      expect(result.value.meta.total).toBe(25)
      expect(result.value.meta.totalPages).toBe(2)
      expect(result.value.meta.currentPage).toBe(1)
      expect(result.value.meta.nextPage).toBe(2)
    }
  })

  it('should filter by search term', async () => {
    // Arrange
    inMemory<Entity>Repository.items.push(make<Entity>({ name: 'Alpha' }))
    inMemory<Entity>Repository.items.push(make<Entity>({ name: 'Beta' }))

    // Act
    const result = await sut.execute({ search: 'Alpha' })

    // Assert
    expect(result.isRight()).toBe(true)

    if (result.isRight()) {
      expect(result.value.data).toHaveLength(1)
      expect(result.value.data[0].name).toBe('Alpha')
    }
  })
})
```
