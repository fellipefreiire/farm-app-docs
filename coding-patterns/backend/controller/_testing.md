# Controller — E2E Test Pattern

> Generic E2E test pattern shared by all controller variants in this folder. Co-located here so controller subagents can load variant + test pattern with two reads.

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
