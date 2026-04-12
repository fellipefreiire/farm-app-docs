# Test Factory

> Part of the farm-app backend test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Factories create domain entities with sensible defaults using `faker`. Two forms:

**Function factory** — for unit tests (no DB):

```ts
// test/factories/make-<entity>.ts
import { faker } from '@faker-js/faker'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import { <Entity>, type <Entity>Props } from '@/domain/<domain>/enterprise/entities/<entity>'

export function make<Entity>(
  override: Partial<<Entity>Props> = {},
  id?: UniqueEntityID,
): <Entity> {
  return <Entity>.reconstitute(
    {
      name: faker.lorem.word(),
      active: true,
      createdAt: new Date(),
      ...override,
    },
    id ?? new UniqueEntityID(),
  )
}
```

Factories use `reconstitute()` — they simulate pre-existing DB records, not new creations. To test event emission, call the use case directly in the test.

**Class factory** — for E2E tests (persists to DB via Prisma):

```ts
import { Injectable } from '@nestjs/common'
import { PrismaService } from '@/infra/database/prisma/prisma.service'
import { Prisma<Entity>Mapper } from '@/infra/database/prisma/mappers/<domain>/prisma-<entity>.mapper'

@Injectable()
export class <Entity>Factory {
  constructor(private prisma: PrismaService) {}

  async makePrisma<Entity>(data: Partial<<Entity>Props> = {}): Promise<<Entity>> {
    const entity = make<Entity>(data)

    await this.prisma.<entity>.create({
      data: Prisma<Entity>Mapper.toPrisma(entity),
    })

    return entity
  }
}
```

---
