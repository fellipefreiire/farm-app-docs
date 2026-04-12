# Event E2E Tests

> Part of the farm-app backend test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
