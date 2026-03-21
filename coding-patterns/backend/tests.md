# Tests Pattern

Eleven types of tests: **unit tests** (use cases), **E2E tests** (HTTP controllers), **event subscriber unit tests** (domain event handlers), **event E2E tests** (full-stack event flow), **core primitive tests** (Either, DomainEvents, WatchedList), **mapper tests** (toDomain/toPrisma roundtrip), **presenter tests** (toHTTP formatting), **shared utils tests** (pure functions), **repository cache tests** (Redis cache integration), and **middleware tests** (Helmet, CORS). Each has its own location, tooling, and setup.

> This pattern also covers test Factories (`test/factories/`) — see the Factories section below.
>
> For mutation testing (Stryker) — see `mutation-testing.md`. All unit tests (`.spec.ts`) are subject to mutation testing during Phase 3 validation.

## Table of contents

- [File locations](#file-locations)
- [1. Unit Test (use case)](#1-unit-test-use-case)
  - [Edit use case test](#edit-use-case-test)
  - [List use case test](#list-use-case-test)
  - [List use case with cursor pagination test](#list-use-case-with-cursor-pagination-test)
  - [Delete use case test](#delete-use-case-test)
  - [Toggle status use case test](#toggle-status-use-case-test)
  - [Use case tests with QueryBus (cross-domain data)](#use-case-tests-with-querybus-cross-domain-data)
- [2. Factory](#2-factory)
- [3. E2E Test (controller)](#3-e2e-test-controller)
- [4. Event Subscriber Unit Test](#4-event-subscriber-unit-test)
- [4b. Event Test (E2E)](#4b-event-test-e2e)
- [5. Core Primitive Tests](#5-core-primitive-tests)
- [6. Mapper Test](#6-mapper-test)
- [7. Presenter Test](#7-presenter-test)
- [8. Shared Utils Test](#8-shared-utils-test)
- [9. Repository Cache Test (E2E)](#9-repository-cache-test-e2e)
- [10. Middleware E2E Test](#10-middleware-e2e-test)
- [Rules (all tests)](#rules-all-tests)
- [Anti-patterns](#anti-patterns)

---

## File locations

```
src/domain/<domain>/application/use-cases/__tests__/<action>-<entity>.spec.ts              ← unit tests (use cases)
src/infra/http/controllers/<domain>/__tests__/<action>-<entity>.controller.e2e-spec.ts     ← E2E tests (controllers)
src/domain/<domain>/application/subscribers/<source-domain>/__tests__/on-<entity>-<action>.spec.ts ← subscriber unit tests
src/infra/events/<domain>/__tests__/on-<entity>-<action>.e2e-spec.ts                       ← event E2E tests
src/core/__tests__/<primitive>.spec.ts                                                      ← core primitive tests
src/infra/database/prisma/mappers/<domain>/__tests__/prisma-<entity>-mapper.spec.ts        ← mapper tests
src/infra/http/presenters/__tests__/<entity>-presenter.spec.ts                              ← presenter tests
src/shared/utils/__tests__/<util-name>.spec.ts                                              ← shared utils tests
src/infra/database/prisma/repositories/<domain>/__tests__/prisma-<entity>-repository.e2e-spec.ts ← repository cache tests
test/e2e/<middleware>.e2e-spec.ts                                                           ← middleware tests
test/factories/make-<entity>.ts                                                             ← entity factories (shared)
test/repositories/<domain>/in-memory-<entity>-repository.ts                                ← InMemory repos (shared)
test/utils/wait-for.ts                                                                      ← async polling helper
```

---

## 1. Unit Test (use case)

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

### Edit use case test

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

### List use case test

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

### List use case with cursor pagination test

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

### Delete use case test

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

### Toggle status use case test

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

### Use case tests with QueryBus (cross-domain data)

When a use case calls `QueryBus.execute()` instead of injecting a cross-domain repository, tests use `InMemoryQueryBus` to register static responses:

```ts
import { InMemoryQueryBus } from 'test/query-bus/in-memory-query-bus'

let queryBus: InMemoryQueryBus

describe('Use Case with QueryBus', () => {
  beforeEach(() => {
    queryBus = new InMemoryQueryBus()
    // Register default response for cross-domain query
    queryBus.register('FindHarvestQuery', {
      id: 'harvest-1',
      status: 'PLANNED',
      startDate: new Date('2026-01-01'),
      expectedEndDate: new Date('2026-06-30'),
    })

    sut = new MyUseCase(myRepository)  // no cross-domain repo in constructor
  })

  afterEach(() => {
    queryBus.clear()
  })

  it('should return error when cross-domain entity not found', async () => {
    queryBus.register('FindHarvestQuery', null)  // override for this test
    const result = await sut.execute({ ... })
    expect(result.isLeft()).toBe(true)
  })
})
```

For tests querying **multiple entities by ID** (e.g., copy-schedule with source + target harvests), register a dynamic handler:

```ts
import { QueryBus } from '@/core/query-bus/query-bus'

const harvestsMap = new Map<string, FindHarvestQueryResult>()

beforeEach(() => {
  QueryBus.register('FindHarvestQuery', async (query: any) => {
    return harvestsMap.get(query.harvestId) ?? null
  })
})

afterEach(() => {
  harvestsMap.clear()
  QueryBus.clearHandlers()
})
```

**Key differences from repository tests:**
- No `InMemory<Entity>Repository` for the cross-domain entity — use `InMemoryQueryBus` instead
- Constructor args decrease (cross-domain repos removed)
- Response data is a plain object, not a domain entity
- Error imports change to local (`./errors/`) instead of cross-domain paths

See `query-bus.md` for the full testing patterns and handler test examples.

---

## 2. Factory

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

## 3. E2E Test (controller)

```ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import request from 'supertest'
import { AppModule } from '@/infra/app.module'
import { PrismaService } from '@/infra/database/prisma/prisma.service'
import { <Entity>Factory } from 'test/factories/make-<entity>'
import { <Entity>DatabaseModule } from '@/infra/database/prisma/repositories/<domain>/<entity>-database.module'
import { UserFactory } from 'test/factories/make-user'
import { TokenService } from '@/infra/auth/jwt/token.service'
import { CryptographyModule } from '@/infra/cryptography/cryptography.module'
import { randomUUID } from 'node:crypto'
import { UserDatabaseModule } from '@/infra/database/prisma/repositories/user/user-database.module'
import type { User } from '@/domain/user/enterprise/entities/user'

describe('<Action> <Entity> (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let <entity>Factory: <Entity>Factory
  let userFactory: UserFactory
  let tokenService: TokenService
  let user: User
  let accessToken: { token: string; expiresIn: number }

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [
        AppModule,
        <Entity>DatabaseModule,
        CryptographyModule,
        UserDatabaseModule,
      ],
      providers: [<Entity>Factory, UserFactory, TokenService],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    <entity>Factory = moduleRef.get(<Entity>Factory)
    userFactory = moduleRef.get(UserFactory)
    tokenService = moduleRef.get(TokenService)

    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "<table_name>" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "users" CASCADE')

    user = await userFactory.makePrismaUser({ role: 'ADMIN' })

    accessToken = await tokenService.generateAccessToken({
      sub: user.id.toString(),
      role: user.role,
      jti: randomUUID(),
    })
  })

  describe('[POST] /v1/<entities>', () => {
    it('[201] should create a <entity>', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/<entities>')
        .set('Authorization', `Bearer ${accessToken.token}`)
        .send({ name: 'test name' })

      expect(response.statusCode).toBe(201)
      expect(response.body.data).toEqual(
        expect.objectContaining({ name: 'test name' }),
      )
    })

    it('[401] should return 401 when not authenticated', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/<entities>')
        .send({ name: 'test name' })

      expect(response.statusCode).toBe(401)
    })

    it('[422] should return 422 when body is invalid', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/<entities>')
        .set('Authorization', `Bearer ${accessToken.token}`)
        .send({}) // missing required fields

      expect(response.statusCode).toBe(422)
    })

    it('[404] should return 404 when dependency does not exist', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/<entities>')
        .set('Authorization', `Bearer ${accessToken.token}`)
        .send({ name: 'test', dependencyId: '00000000-0000-0000-0000-000000000000' })

      expect(response.statusCode).toBe(404)
    })
  })
})
```

**Rules:**
- `beforeAll` — spin up NestJS app once per file
- `afterAll` — close the app
- `beforeEach` — `TRUNCATE` all tables touched by the test, create fresh user + token
- Always test the 401 case for protected endpoints
- Always test the 422 case for invalid input
- Always test the 404 case for missing dependencies
- Use `expect.objectContaining({})` for partial response assertions
- Use `Bearer ${accessToken.token}` for authenticated requests

---

## 4. Event Subscriber Unit Test

Subscriber unit tests verify that a domain event subscriber calls the correct use case when an event is dispatched. They use InMemory repositories and `DomainEvents.dispatchEventsForAggregate()` directly — no HTTP layer.

```ts
// src/domain/<subscriber-domain>/application/subscribers/<source-domain>/__tests__/on-<entity>-<action>.spec.ts
import { vi } from 'vitest'
import { waitFor } from 'test/utils/wait-for'
import { make<Entity> } from 'test/factories/make-<entity>'
import { DomainEvents } from '@/core/events/domain-events'
import { On<Entity><Action> } from '@/domain/<subscriber-domain>/application/subscribers/<source-domain>/on-<entity>-<action>'
import { InMemory<Entity>sRepository } from 'test/repositories/<source-domain>/in-memory-<entity>s-repository'
import { InMemory<SideEffect>Repository } from 'test/repositories/<subscriber-domain>/in-memory-<side-effect>-repository'
import { <SideEffect>UseCase } from '@/domain/<subscriber-domain>/application/use-cases/<side-effect-action>'

let inMemory<Entity>sRepository: InMemory<Entity>sRepository
let inMemory<SideEffect>Repository: InMemory<SideEffect>Repository
let <sideEffect>UseCase: <SideEffect>UseCase
let useCaseSpy: ReturnType<typeof vi.spyOn>

describe('On <Entity> <Action> (subscriber)', () => {
  beforeEach(() => {
    inMemory<Entity>sRepository = new InMemory<Entity>sRepository()
    inMemory<SideEffect>Repository = new InMemory<SideEffect>Repository()
    <sideEffect>UseCase = new <SideEffect>UseCase(inMemory<SideEffect>Repository)
    useCaseSpy = vi.spyOn(<sideEffect>UseCase, 'execute')

    new On<Entity><Action>(<sideEffect>UseCase)
  })

  it('should create <side-effect> when <entity> is <action>', async () => {
    // Arrange
    const entity = make<Entity>()
    inMemory<Entity>sRepository.create(entity)

    // Act — trigger the domain event
    entity.<action>(entity.id.toString())
    DomainEvents.dispatchEventsForAggregate(entity.id)

    // Assert
    await waitFor(() => {
      expect(useCaseSpy).toHaveBeenCalled()
    })

    expect(inMemory<SideEffect>Repository.items).toHaveLength(1)
    expect(inMemory<SideEffect>Repository.items[0]).toEqual(
      expect.objectContaining({
        action: '<source-domain>:<action>',
        entityId: entity.id.toString(),
        actorId: entity.id.toString(),
      }),
    )
  })
})
```

**Rules:**
- Use `vi.spyOn` on the use case's `execute` method — never mock the entire use case
- Use `waitFor()` for assertions — subscribers are async
- Trigger events via entity methods + `DomainEvents.dispatchEventsForAggregate()` (not via HTTP)
- File location: `src/domain/<subscriber-domain>/application/subscribers/<source-domain>/__tests__/`
- These are unit tests (`.spec.ts`) — InMemory repos, no database

---

## 4b. Event Test (E2E)

Event E2E tests verify the full stack: HTTP request → use case → entity event → subscriber → side effect. They live in `src/infra/events/<domain>/__tests__/`.

```ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import request from 'supertest'
import { AppModule } from '@/infra/app.module'
import { PrismaService } from '@/infra/database/prisma/prisma.service'
import { UserFactory } from 'test/factories/make-user'
import { <Entity>Factory } from 'test/factories/make-<entity>'
import { TokenService } from '@/infra/auth/jwt/token.service'
import { CryptographyModule } from '@/infra/cryptography/cryptography.module'
import { UserDatabaseModule } from '@/infra/database/prisma/repositories/user/user-database.module'
import { <Entity>DatabaseModule } from '@/infra/database/prisma/repositories/<domain>/<entity>-database.module'
import { DomainEvents } from '@/core/events/domain-events'
import { waitFor } from 'test/utils/wait-for'
import { randomUUID } from 'node:crypto'

describe('<Entity> <Action> Event (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let userFactory: UserFactory
  let <entity>Factory: <Entity>Factory
  let tokenService: TokenService

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule, CryptographyModule, UserDatabaseModule, <Entity>DatabaseModule],
      providers: [UserFactory, <Entity>Factory, TokenService],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    userFactory = moduleRef.get(UserFactory)
    <entity>Factory = moduleRef.get(<Entity>Factory)
    tokenService = moduleRef.get(TokenService)

    DomainEvents.shouldRun = true  // events are disabled by default in tests

    await app.init()
  })

  afterAll(async () => {
    DomainEvents.shouldRun = false
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "<table_name>" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "users" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "audit_logs" CASCADE')
  })

  it('[EVENT] → should create an audit log when <entity> is <action>', async () => {
    // Arrange
    const user = await userFactory.makePrismaUser({ role: 'ADMIN' })
    const accessToken = await tokenService.generateAccessToken({
      sub: user.id.toString(),
      role: user.role,
      jti: randomUUID(),
    })

    // Act — trigger the event via HTTP (use case → entity.create() → event → subscriber)
    const response = await request(app.getHttpServer())
      .post('/v1/<entities>')
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({ name: 'test name' })

    expect(response.statusCode).toBe(201)

    // Assert — subscriber runs asynchronously, poll until side effect appears
    await waitFor(async () => {
      const auditLog = await prisma.auditLog.findFirst({
        where: { actorId: user.id.toString(), action: '<domain>:<action>' },
      })

      expect(auditLog).not.toBeNull()
      expect(auditLog).toMatchObject({
        actorId: user.id.toString(),
        action: '<domain>:<action>',
        entity: '<ENTITY_CONSTANT>',
      })
    })
  })
})
```

**`waitFor` utility** — polls assertions until they pass or timeout (default 1000ms):

```ts
// test/utils/wait-for.ts
export async function waitFor(
  assertions: () => Promise<void> | void,
  maxDuration = 1000,
): Promise<void> {
  return new Promise((resolve, reject) => {
    let elapsedTime = 0
    const interval = setInterval(async () => {
      elapsedTime += 10
      try {
        await assertions()
        clearInterval(interval)
        resolve()
      } catch (err) {
        if (elapsedTime >= maxDuration) reject(err)
      }
    }, 10)
  })
}
```

**Rules:**
- `DomainEvents.shouldRun = true` in `beforeAll` — domain events are disabled by default in the test environment
- Always use `waitFor()` to assert subscriber side effects — subscribers are async
- Trigger the event via HTTP (not by calling the use case directly) — tests the full stack
- Truncate all tables the test touches in `beforeEach`, including the side effect table (e.g. `audit_logs`)
- One event scenario per `it()` — never assert multiple events in one test
- File naming: `on-<entity>-<action>.e2e-spec.ts`

---

## 5. Core Primitive Tests

Core primitives (`Either`, `DomainEvents`, `WatchedList`, `UniqueEntityID`) are foundational building blocks. Each must have a dedicated test file in `src/core/__tests__/`.

### Either test

```ts
// src/core/__tests__/either.spec.ts
import { type Either, left, right } from '../either'

function doSomething(shouldSuccess: boolean): Either<string, number> {
  if (shouldSuccess) {
    return right(10)
  }

  return left('error')
}

test('success', () => {
  const result = doSomething(true)

  expect(result.isRight()).toBe(true)
  expect(result.isLeft()).toBe(false)
})

test('error', () => {
  const result = doSomething(false)

  expect(result.isRight()).toBe(false)
  expect(result.isLeft()).toBe(true)
})
```

### DomainEvents test

Tests the register → dispatch → callback cycle. Creates inline aggregate and event classes for isolation.

```ts
// src/core/__tests__/domain-events.spec.ts
import { DomainEvent } from '../events/domain-event'
import { UniqueEntityID } from '../entities/unique-entity-id'
import { AggregateRoot } from '../entities/aggregate-root'
import { DomainEvents } from '@/core/events/domain-events'
import { vi } from 'vitest'

class CustomAggregateCreated implements DomainEvent {
  public occurredAt: Date
  private aggregate: CustomAggregate

  constructor(aggregate: CustomAggregate) {
    this.aggregate = aggregate
    this.occurredAt = new Date()
  }

  public getAggregateId(): UniqueEntityID {
    return this.aggregate.id
  }
}

class CustomAggregate extends AggregateRoot<null> {
  static create() {
    const aggregate = new CustomAggregate(null)
    aggregate.addDomainEvent(new CustomAggregateCreated(aggregate))
    return aggregate
  }
}

describe('domain events', () => {
  it('should be able to dispatch and listen to events', async () => {
    const callbackSpy = vi.fn()

    // Register subscriber
    DomainEvents.register(callbackSpy, CustomAggregateCreated.name)

    // Create aggregate (event queued, not dispatched)
    const aggregate = CustomAggregate.create()
    expect(aggregate.domainEvents).toHaveLength(1)

    // Dispatch (simulates repository save)
    DomainEvents.dispatchEventsForAggregate(aggregate.id)

    expect(callbackSpy).toHaveBeenCalled()
    expect(aggregate.domainEvents).toHaveLength(0)
  })
})
```

### WatchedList test

Tests add, remove, update tracking, and re-add/re-remove edge cases.

```ts
// src/core/__tests__/watched-list.spec.ts
import { WatchedList } from '@/core/entities/watched-list'

class NumberWatchedList extends WatchedList<number> {
  compareItems(a: number, b: number): boolean {
    return a === b
  }
}

describe('watched list', () => {
  it('should be able to create a watched list with initial items', () => {
    const list = new NumberWatchedList([1, 2, 3])
    expect(list.currentItems).toHaveLength(3)
  })

  it('should be able to add new items to the list', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.add(4)
    expect(list.currentItems).toHaveLength(4)
    expect(list.getNewItems()).toEqual([4])
  })

  it('should be able to remove items from the list', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.remove(2)
    expect(list.currentItems).toHaveLength(2)
    expect(list.getRemovedItems()).toEqual([2])
  })

  it('should be able to add an item even if it was removed before', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.remove(2)
    list.add(2)
    expect(list.currentItems).toHaveLength(3)
    expect(list.getRemovedItems()).toEqual([])
    expect(list.getNewItems()).toEqual([])
  })

  it('should be able to remove an item even if it was added before', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.add(4)
    list.remove(4)
    expect(list.currentItems).toHaveLength(3)
    expect(list.getRemovedItems()).toEqual([])
    expect(list.getNewItems()).toEqual([])
  })

  it('should be able to update watched list items', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.update([1, 3, 5])
    expect(list.getRemovedItems()).toEqual([2])
    expect(list.getNewItems()).toEqual([5])
  })
})
```

**Rules:**
- One test file per core primitive
- Core primitive tests are pure unit tests — no database, no DI
- DomainEvents test must use inline aggregate/event classes (never import real domain entities)
- WatchedList test must use a concrete subclass (e.g. `NumberWatchedList`)
- Every new core primitive must have a corresponding test

---

## 6. Mapper Test

Mapper tests verify `toDomain()`, `toPrisma()`, and roundtrip data preservation. They are unit tests — no database required.

```ts
// src/infra/database/prisma/mappers/<domain>/__tests__/prisma-<entity>-mapper.spec.ts
import { describe, expect, it } from 'vitest'
import { Prisma<Entity>Mapper } from '../prisma-<entity>.mapper'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import type { <Entity> as Prisma<Entity> } from '@/generated/prisma/client'

describe('Prisma<Entity>Mapper', () => {
  const now = new Date('2024-01-01T00:00:00.000Z')

  const prisma<Entity>: Prisma<Entity> = {
    id: '<entity>-1',
    name: 'Test Name',
    // ... all Prisma model fields
    createdAt: now,
    updatedAt: now,
  }

  describe('toDomain', () => {
    it('should map a Prisma <entity> to a domain <Entity> entity', () => {
      const entity = Prisma<Entity>Mapper.toDomain(prisma<Entity>)

      expect(entity).toBeInstanceOf(<Entity>)
      expect(entity.id.toString()).toBe('<entity>-1')
      expect(entity.name).toBe('Test Name')
      // ... assert all fields
    })
  })

  describe('toPrisma', () => {
    it('should map a domain <Entity> entity to Prisma input', () => {
      const entity = <Entity>.reconstitute(
        {
          name: 'Test Name',
          // ... all entity props
          createdAt: now,
        },
        new UniqueEntityID('<entity>-2'),
      )

      const result = Prisma<Entity>Mapper.toPrisma(entity)

      expect(result).toEqual({
        id: '<entity>-2',
        name: 'Test Name',
        // ... all Prisma fields
        createdAt: now,
      })
    })
  })

  describe('roundtrip', () => {
    it('should preserve data through toDomain → toPrisma', () => {
      const domain = Prisma<Entity>Mapper.toDomain(prisma<Entity>)
      const result = Prisma<Entity>Mapper.toPrisma(domain)

      expect(result.id).toBe(prisma<Entity>.id)
      expect(result.name).toBe(prisma<Entity>.name)
      // ... assert all fields match original
    })
  })
})
```

**Special cases — nullable JSON fields:**

When a mapper handles nullable JSON fields (e.g. `changes` in AuditLog), test both the value and null cases:

```ts
it('should handle null changes', () => {
  const prismaLog: PrismaAuditLog = {
    // ... other fields
    changes: null,
  }

  const auditLog = PrismaAuditLogMapper.toDomain(prismaLog)
  expect(auditLog.changes).toBeNull()
})

it('should map null changes to Prisma.JsonNull', () => {
  const auditLog = AuditLog.create({ /* ... */ changes: null })
  const result = PrismaAuditLogMapper.toPrisma(auditLog)
  expect(result.changes).toBe(Prisma.JsonNull)
})
```

**Rules:**
- Every mapper must have a test file
- Always test all three directions: `toDomain`, `toPrisma`, `roundtrip`
- Use `reconstitute()` (not `create()`) when building entities for toPrisma tests — factories simulate existing records
- Test nullable/optional fields explicitly (null JSON, optional dates, etc.)
- Import Prisma types directly from `@prisma/client` for the raw Prisma object shape
- Mapper tests are pure unit tests — no database, no DI container

---

## 7. Presenter Test

Presenter tests verify `toHTTP()` formatting and field exclusion (e.g. password). They are unit tests — no database required.

```ts
// src/infra/http/presenters/__tests__/<entity>-presenter.spec.ts
import { describe, expect, it } from 'vitest'
import { <Entity>Presenter } from '../<entity>.presenter'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'

describe('<Entity>Presenter', () => {
  it('should format a <entity> entity to HTTP response', () => {
    // Arrange
    const createdAt = new Date('2024-01-01T00:00:00.000Z')
    const updatedAt = new Date('2024-01-02T00:00:00.000Z')

    const entity = <Entity>.reconstitute(
      {
        name: 'Test Name',
        // ... all entity props
        createdAt,
        updatedAt,
      },
      new UniqueEntityID('<entity>-1'),
    )

    // Act
    const result = <Entity>Presenter.toHTTP(entity)

    // Assert
    expect(result).toEqual({
      id: '<entity>-1',
      name: 'Test Name',
      // ... all HTTP response fields
      createdAt,
      updatedAt,
    })
  })

  it('should not expose sensitive fields', () => {
    // Arrange
    const entity = <Entity>.reconstitute(
      {
        name: 'Test',
        password: 'secret-password',
        // ... other props
      },
      new UniqueEntityID('<entity>-2'),
    )

    // Act
    const result = <Entity>Presenter.toHTTP(entity)

    // Assert
    expect(result).not.toHaveProperty('password')
  })
})
```

**Rules:**
- Every presenter must have a test file
- Always test that sensitive fields (password, tokens, secrets) are excluded
- Use `reconstitute()` to build entities in presenter tests
- Presenter tests are pure unit tests — no database, no DI container
- Test nullable fields: verify `undefined` is converted to `null` when applicable

---

## 8. Shared Utils Test

Shared utility functions (`src/shared/utils/`) are pure functions. Each must have a dedicated test file.

```ts
// src/shared/utils/__tests__/<util-name>.spec.ts
import { describe, expect, it } from 'vitest'
import { <utilFunction> } from '../<util-name>'

describe('<utilFunction>', () => {
  it('should <expected behavior on success>', async () => {
    const result = await <utilFunction>(/* valid input */)
    expect(result).toBe(/* expected */)
  })

  it('should <expected behavior on failure>', async () => {
    await expect(<utilFunction>(/* failing input */)).rejects.toThrow(/* error */)
  })
})
```

### withTimeout test

```ts
// src/shared/utils/__tests__/with-timeout.spec.ts
describe('withTimeout', () => {
  it('should resolve if the promise finishes before timeout', async () => {
    const result = await withTimeout(
      new Promise((resolve) => setTimeout(() => resolve('ok'), 50)),
      100,
    )
    expect(result).toBe('ok')
  })

  it('should reject if the promise takes longer than the timeout', async () => {
    await expect(
      withTimeout(
        new Promise((resolve) => setTimeout(() => resolve('too late'), 150)),
        50,
      ),
    ).rejects.toThrowError('Timeout')
  })
})
```

### retryWithBackoff test

```ts
// src/shared/utils/__tests__/retry-with-backoff.spec.ts
describe('retryWithBackoff', () => {
  it('should resolve successfully on first attempt', async () => {
    const fn = vi.fn().mockResolvedValue('success')
    const result = await retryWithBackoff(fn)
    expect(result).toBe('success')
    expect(fn).toHaveBeenCalledTimes(1)
  })

  it('should retry on failure and eventually succeed', async () => {
    const fn = vi.fn()
      .mockRejectedValueOnce(new Error('fail'))
      .mockResolvedValue('success')
    const onRetry = vi.fn()
    const result = await retryWithBackoff(fn, { retries: 3, initialDelayMs: 10, onRetry })
    expect(result).toBe('success')
    expect(fn).toHaveBeenCalledTimes(2)
    expect(onRetry).toHaveBeenCalledWith(expect.any(Error), 1)
  })

  it('should throw if all retries fail', async () => {
    const fn = vi.fn().mockRejectedValue(new Error('fail'))
    await expect(
      retryWithBackoff(fn, { retries: 2, initialDelayMs: 10, onRetry: vi.fn() }),
    ).rejects.toThrow('fail')
  })
})
```

### sanitize test

```ts
// src/shared/utils/__tests__/sanitize-html.spec.ts
describe('sanitize', () => {
  it('should remove HTML tags', () => {
    expect(sanitize('<script>alert("xss")</script><b>bold</b>')).toBe('bold')
  })

  it('should preserve plain text', () => {
    expect(sanitize('hello world')).toBe('hello world')
  })

  it('should strip attributes and tags', () => {
    expect(sanitize('<a href="https://example.com">link</a>')).toBe('link')
  })

  it('should return empty string if only HTML tags are passed', () => {
    expect(sanitize('<div><img src="x"/></div>')).toBe('')
  })
})
```

**Rules:**
- Every shared util must have a corresponding test file
- Test both success and failure paths
- Use small `initialDelayMs` values in retry tests to avoid slow test suites
- Pure function tests — no database, no DI, no mocking (except for retry callbacks)

---

## 9. Repository Cache Test (E2E)

Repository cache tests verify that the Prisma repository correctly populates and invalidates Redis cache. These are E2E tests that require a running database and Redis.

```ts
// src/infra/database/prisma/repositories/<domain>/__tests__/prisma-<entity>-repository.e2e-spec.ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import { <Entity>DatabaseModule } from '../<entity>-database.module'
import { CacheModule } from '@/infra/cache/cache.module'
import { <Entity>Factory } from 'test/factories/make-<entity>'
import { CacheRepository } from '@/infra/cache/cache.repository'
import { <Entity>sRepository } from '@/domain/<domain>/application/repositories/<entity>s-repository'
import { PrismaService } from '../../../prisma.service'

describe('Prisma <Entity>s Repository (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let <entity>Factory: <Entity>Factory
  let cacheRepository: CacheRepository
  let <entity>sRepository: <Entity>sRepository

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule, <Entity>DatabaseModule, CacheModule],
      providers: [<Entity>Factory],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    <entity>Factory = moduleRef.get(<Entity>Factory)
    cacheRepository = moduleRef.get(CacheRepository)
    <entity>sRepository = moduleRef.get(<Entity>sRepository)

    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "<table_name>" CASCADE')
  })

  it('should cache <entity> details on findById', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    // Act
    await <entity>sRepository.findById(id)

    // Assert
    const cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).not.toBeNull()
    expect(JSON.parse(cached!)).toEqual(
      expect.objectContaining({ id }),
    )
  })

  it('should return cached <entity> details on subsequent calls', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    // Act — first call (cache miss)
    let cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).toBeNull()

    await <entity>sRepository.findById(id)
    cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).not.toBeNull()

    // Act — second call (cache hit)
    const details = await <entity>sRepository.findById(id)

    // Assert
    expect(JSON.parse(cached!)).toEqual(
      expect.objectContaining({ id: details?.id.toString() }),
    )
  })

  it('should invalidate <entity> cache on save', async () => {
    // Arrange
    const entity = await <entity>Factory.makePrisma<Entity>({})
    const id = entity.id.toString()

    await cacheRepository.set(
      `<entity>:${id}:details`,
      JSON.stringify({ empty: true }),
    )

    // Act
    await <entity>sRepository.save(entity)

    // Assert
    const cached = await cacheRepository.get(`<entity>:${id}:details`)
    expect(cached).toBeNull()
  })
})
```

**Rules:**
- Test three cache behaviors: **populate on read**, **hit on subsequent read**, **invalidate on write**
- Test cache for each lookup method (`findById`, `findByEmail`, etc.)
- Use `CacheRepository` to verify cache state directly — don't rely on timing
- These are E2E tests (`.e2e-spec.ts`) — they require real DB and Redis
- Truncate tables in `beforeEach`

---

## 10. Middleware E2E Test

Middleware tests verify that Express middleware (Helmet, CORS, cookie-parser) is correctly applied to the NestJS application. They live in `test/e2e/`.

```ts
// test/e2e/<middleware>.e2e-spec.ts
import request from 'supertest'
import { INestApplication } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import helmet from 'helmet'

describe('Helmet Middleware (E2E)', () => {
  let app: INestApplication

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = moduleRef.createNestApplication()
    app.use(helmet())
    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  it('should apply security headers via Helmet', async () => {
    const response = await request(app.getHttpServer()).get('/health')

    expect(response.statusCode).toBe(200)
    expect(response.headers['x-content-type-options']).toBe('nosniff')
    expect(response.headers['x-frame-options']).toBe('SAMEORIGIN')
    expect(response.headers['strict-transport-security']).toContain('max-age=')
    expect(response.headers['content-security-policy']).toContain('default-src')
  })
})
```

**Rules:**
- Apply the middleware in `beforeAll` exactly as it is applied in `main.ts`
- Test specific header values — not just presence
- Use a public endpoint (e.g. `/health`) to avoid authentication complexity
- One test file per middleware concern (Helmet, CORS, etc.)

---

## Rules (all tests)

- **Every new use case must have a unit test** — create `<action>-<entity>.spec.ts` in the use case's `__tests__/` directory. No exceptions.
- **Every new controller must have an E2E test** — create `<action>-<entity>.controller.e2e-spec.ts` in the controller's `__tests__/` directory. No exceptions. The test infrastructure (Prisma test DB, factories, supertest) works in isolation — no external database needed.
- Unit tests: always use `InMemory` repositories — never real DB
- E2E tests: always use `@Injectable()` factories with Prisma — never raw `prisma.create()` inline
- Always follow AAA pattern: Arrange, Act, Assert
- One behavior per `it()` — keep assertions focused
- Test error paths: not found, unauthorized, validation failure, business rule violation
- Never use `any` in test setup or assertions
- Unit test file suffix: `.spec.ts` — E2E and event test file suffix: `.e2e-spec.ts`
- `.env.test` must NOT contain `DATABASE_URL` — the E2E setup (`test/setup-e2e.ts`) creates a unique schema URL from `.env`'s `DATABASE_URL`. If `.env.test` overrides it, NestJS `ConfigModule` uses the override instead of the isolated schema, and tests run against the development database

---

## Anti-patterns

```ts
// ❌ real database in unit test
const repo = new PrismaFieldRepository(prisma) // always InMemoryRepository

// ❌ no AAA structure
it('test', async () => {
  const sut = new UseCase(repo)
  const r = await sut.execute({ name: 'x' })
  expect(r.isRight()).toBe(true)
  expect(r.value.data.name).toBe('x') // no separation, mixed concerns
})

// ❌ accessing .value before checking Either
expect(result.value.data.name).toBe('x') // check isRight() first

// ❌ no table truncation in E2E beforeEach
beforeEach(async () => {
  // missing TRUNCATE — test state bleeds between runs
})

// ❌ multiple behaviors in one test
it('should create and list', async () => { ... }) // split into two tests
```
