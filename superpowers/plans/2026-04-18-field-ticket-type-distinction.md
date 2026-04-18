# Field Ticket Type Distinction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add type-discriminated validation and UI for SPRAYING vs FERTIGATION field tickets, with different review/finalization requirements and a compact 8-per-page print layout for FERTIGATION.

**Architecture:** Single use cases with `if (operationType === 'SPRAYING')` guards for validation; no new entities, repositories, or schema migrations. Frontend sheets become type-aware via `ticket.operationType`. Print layout groups tickets by type and renders appropriate grid density.

**Tech Stack:** NestJS + Vitest (backend), Next.js + React Hook Form + Zod (frontend), Playwright + MSW (E2E)

---

## File Map

### Backend — create
- `backend/src/domain/field-ticket/application/use-cases/errors/missing-equipment-for-spraying-error.ts`
- `backend/src/domain/field-ticket/application/use-cases/errors/missing-hourmeter-for-spraying-error.ts`
- `backend/src/infra/http/controllers/field-ticket/__tests__/review-field-ticket.controller.e2e-spec.ts`
- `backend/src/infra/http/controllers/field-ticket/__tests__/finalize-field-ticket.controller.e2e-spec.ts`

### Backend — modify
- `backend/src/domain/field-ticket/enterprise/entities/field-ticket.ts` — `finalize()` hourmeter params optional
- `backend/src/domain/field-ticket/application/use-cases/review-field-ticket.ts` — SPRAYING requires vehicleId + implementId
- `backend/src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts` — type-discriminated validation
- `backend/src/domain/field-ticket/application/use-cases/__tests__/review-field-ticket.spec.ts` — new cases
- `backend/src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts` — new cases
- `backend/src/infra/http/filters/field-ticket-error.filter.ts` — register new errors

### Frontend — modify
- `frontend/src/domains/field-ticket/components/sheets/configure-field-ticket-sheet.tsx` — hide equipment for FERTIGATION
- `frontend/src/domains/field-ticket/components/sheets/finalize-field-ticket-sheet.tsx` — operationType state, conditional hourmeter, dynamic label
- `frontend/src/domains/field-ticket/components/print/print-field-ticket-layout.tsx` — 8/page FERTIGATION layout

### Frontend — create
- `frontend/tests/field-ticket/review-finalize-field-ticket.e2e-spec.ts`
- `frontend/tests/mocks/handlers/field-ticket.handler.ts` (or extend existing)

### Docs — modify
- `docs/rules/field-ticket.md` — update invariants section

---

## Task 1: Make `finalize()` accept optional hourmeter

**Files:**
- Modify: `backend/src/domain/field-ticket/enterprise/entities/field-ticket.ts:265-327`

- [ ] **Step 1: Write the failing test**

Add to `backend/src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts`:

```ts
it('should finalize a FERTIGATION ticket without hourmeter', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'PRINTED', operationType: 'FERTIGATION' })
  fieldTicketsRepository.items.push(ticket)

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    startTime: '08:00',
    endTime: '12:00',
    operatorName: 'Maria',
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isRight()).toBe(true)
  if (result.isRight()) {
    expect(result.value.data.status).toBe('COMPLETED')
    expect(result.value.data.hourmeterStart).toBeUndefined()
    expect(result.value.data.hourmeterEnd).toBeUndefined()
  }
})
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd backend && contextzip pnpm test finalize-field-ticket.spec.ts
```

Expected: FAIL — TypeScript error: `hourmeterStart` is required in `finalize()`.

- [ ] **Step 3: Update entity `finalize()` signature**

In `backend/src/domain/field-ticket/enterprise/entities/field-ticket.ts`, change:

```ts
// BEFORE (lines 265-282)
finalize(
  data: {
    startTime: string
    endTime: string
    hourmeterStart: number
    hourmeterEnd: number
    // ...
  },
```

```ts
// AFTER
finalize(
  data: {
    startTime: string
    endTime: string
    hourmeterStart?: number
    hourmeterEnd?: number
    // ...
  },
```

Also update the body (lines 303-306): keep the conditional assignments:
```ts
this.props.startTime = data.startTime
this.props.endTime = data.endTime
if (data.hourmeterStart !== undefined) this.props.hourmeterStart = data.hourmeterStart
if (data.hourmeterEnd !== undefined) this.props.hourmeterEnd = data.hourmeterEnd
```

And update `previousData` snapshot (lines 285-301) — hourmeterStart/End are already optional in the snapshot so no change needed there.

- [ ] **Step 4: Fix the use case call site**

In `backend/src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts`, change lines 119-123:

```ts
// BEFORE
ticket.finalize(
  {
    startTime: startTime!,
    endTime: endTime!,
    hourmeterStart: hourmeterStart!,
    hourmeterEnd: hourmeterEnd!,
```

```ts
// AFTER
ticket.finalize(
  {
    startTime: startTime!,
    endTime: endTime!,
    hourmeterStart,
    hourmeterEnd,
```

- [ ] **Step 5: Run tests to verify pass**

```bash
cd backend && contextzip pnpm test finalize-field-ticket.spec.ts
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add backend/src/domain/field-ticket/enterprise/entities/field-ticket.ts \
        backend/src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts \
        backend/src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts
git commit -m "refactor(field-ticket): make hourmeter optional in finalize() for fertigation support"
```

---

## Task 2: Add new domain errors

**Files:**
- Create: `backend/src/domain/field-ticket/application/use-cases/errors/missing-equipment-for-spraying-error.ts`
- Create: `backend/src/domain/field-ticket/application/use-cases/errors/missing-hourmeter-for-spraying-error.ts`

- [ ] **Step 1: Create MissingEquipmentForSprayingError**

```ts
// missing-equipment-for-spraying-error.ts
import { BaseError } from '@/core/errors/base-error'

export class MissingEquipmentForSprayingError extends BaseError {
  constructor() {
    super(
      'SPRAYING tickets require vehicleId and implementId before review.',
      'MissingEquipmentForSprayingError',
    )
  }
}
```

- [ ] **Step 2: Create MissingHourmeterForSprayingError**

```ts
// missing-hourmeter-for-spraying-error.ts
import { BaseError } from '@/core/errors/base-error'

export class MissingHourmeterForSprayingError extends BaseError {
  constructor() {
    super(
      'SPRAYING tickets require hourmeterStart and hourmeterEnd for finalization.',
      'MissingHourmeterForSprayingError',
    )
  }
}
```

- [ ] **Step 3: Register in error filter**

In `backend/src/infra/http/filters/field-ticket-error.filter.ts`, add:

```ts
import { MissingEquipmentForSprayingError } from '@/domain/field-ticket/application/use-cases/errors/missing-equipment-for-spraying-error'
import { MissingHourmeterForSprayingError } from '@/domain/field-ticket/application/use-cases/errors/missing-hourmeter-for-spraying-error'
```

And in `mapDomainErrorToStatus`:
```ts
case MissingEquipmentForSprayingError.name:
  return 422
case MissingHourmeterForSprayingError.name:
  return 422
```

- [ ] **Step 4: Run full test suite to verify no breakage**

```bash
cd backend && contextzip pnpm test
```

Expected: all existing tests PASS

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/field-ticket/application/use-cases/errors/missing-equipment-for-spraying-error.ts \
        backend/src/domain/field-ticket/application/use-cases/errors/missing-hourmeter-for-spraying-error.ts \
        backend/src/infra/http/filters/field-ticket-error.filter.ts
git commit -m "feat(field-ticket): add MissingEquipmentForSprayingError and MissingHourmeterForSprayingError"
```

---

## Task 3: ReviewFieldTicket — SPRAYING requires vehicleId + implementId

**Files:**
- Modify: `backend/src/domain/field-ticket/application/use-cases/review-field-ticket.ts`
- Modify: `backend/src/domain/field-ticket/application/use-cases/__tests__/review-field-ticket.spec.ts`

- [ ] **Step 1: Write failing tests**

Add to `review-field-ticket.spec.ts`:

```ts
import { MissingEquipmentForSprayingError } from '../errors/missing-equipment-for-spraying-error'

// Add to describe block:

it('should review a FERTIGATION ticket without vehicleId/implementId', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'DRAFT', operationType: 'FERTIGATION' })
  fieldTicketsRepo.items.push(ticket)

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isRight()).toBe(true)
  if (result.isRight()) {
    expect(result.value.data.status).toBe('REVIEWED')
  }
})

it('should return error when SPRAYING ticket reviewed without vehicleId', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'DRAFT', operationType: 'SPRAYING' })
  fieldTicketsRepo.items.push(ticket)

  queryBus.register('FindActiveImplementQuery', { id: 'impl-1', active: true })

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    implementId: 'impl-1',
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isLeft()).toBe(true)
  expect(result.value).toBeInstanceOf(MissingEquipmentForSprayingError)
})

it('should return error when SPRAYING ticket reviewed without implementId', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'DRAFT', operationType: 'SPRAYING' })
  fieldTicketsRepo.items.push(ticket)

  queryBus.register('FindActiveVehicleQuery', { id: 'vehicle-1', active: true })

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    vehicleId: 'vehicle-1',
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isLeft()).toBe(true)
  expect(result.value).toBeInstanceOf(MissingEquipmentForSprayingError)
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd backend && contextzip pnpm test review-field-ticket.spec.ts
```

Expected: 3 new tests FAIL

- [ ] **Step 3: Add SPRAYING guard to use case**

In `review-field-ticket.ts`, after the status check (line ~96), add:

```ts
import { MissingEquipmentForSprayingError } from './errors/missing-equipment-for-spraying-error'

// After the status guard, before vehicle validation:
if (ticket.operationType === 'SPRAYING' && (!vehicleId || !implementId)) {
  return left(new MissingEquipmentForSprayingError())
}
```

Also update the `ReviewFieldTicketResponse` type union:
```ts
type ReviewFieldTicketResponse = Either<
  | FieldTicketNotFoundError
  | InvalidStatusTransitionError
  | VehicleNotFoundError
  | ImplementNotFoundError
  | MissingEquipmentForSprayingError,
  { data: FieldTicket }
>
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd backend && contextzip pnpm test review-field-ticket.spec.ts
```

Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/field-ticket/application/use-cases/review-field-ticket.ts \
        backend/src/domain/field-ticket/application/use-cases/__tests__/review-field-ticket.spec.ts
git commit -m "feat(field-ticket): enforce vehicleId+implementId for SPRAYING review"
```

---

## Task 4: FinalizeFieldTicket — type-discriminated hourmeter validation

**Files:**
- Modify: `backend/src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts`
- Modify: `backend/src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts`

- [ ] **Step 1: Write failing tests**

Add to `finalize-field-ticket.spec.ts`:

```ts
import { MissingHourmeterForSprayingError } from '../errors/missing-hourmeter-for-spraying-error'

it('should return error when SPRAYING ticket finalized without hourmeterStart', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'PRINTED', operationType: 'SPRAYING' })
  fieldTicketsRepository.items.push(ticket)

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    startTime: '08:00',
    endTime: '12:00',
    hourmeterEnd: 104,
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isLeft()).toBe(true)
  expect(result.value).toBeInstanceOf(MissingHourmeterForSprayingError)
})

it('should return error when SPRAYING ticket finalized without hourmeterEnd', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'PRINTED', operationType: 'SPRAYING' })
  fieldTicketsRepository.items.push(ticket)

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    startTime: '08:00',
    endTime: '12:00',
    hourmeterStart: 100,
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isLeft()).toBe(true)
  expect(result.value).toBeInstanceOf(MissingHourmeterForSprayingError)
})

it('should finalize SPRAYING ticket with all required fields', async () => {
  // Arrange
  const ticket = makeFieldTicket({ status: 'PRINTED', operationType: 'SPRAYING' })
  fieldTicketsRepository.items.push(ticket)

  // Act
  const result = await sut.execute({
    fieldTicketId: ticket.id.toString(),
    startTime: '08:00',
    endTime: '12:00',
    hourmeterStart: 100,
    hourmeterEnd: 104,
    operatorName: 'João',
    actorId: 'actor-1',
  })

  // Assert
  expect(result.isRight()).toBe(true)
  if (result.isRight()) {
    expect(result.value.data.hourmeterStart).toBe(100)
    expect(result.value.data.hourmeterEnd).toBe(104)
  }
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd backend && contextzip pnpm test finalize-field-ticket.spec.ts
```

Expected: 2 new error tests FAIL (no guard yet)

- [ ] **Step 3: Add type-discriminated guard to use case**

In `finalize-field-ticket.ts`, after the `ticket.status !== 'PRINTED'` check, add:

```ts
import { MissingHourmeterForSprayingError } from './errors/missing-hourmeter-for-spraying-error'

// After status guard, before vehicle validation:
if (
  ticket.operationType === 'SPRAYING' &&
  (hourmeterStart === undefined || hourmeterStart === null ||
   hourmeterEnd === undefined || hourmeterEnd === null)
) {
  return left(new MissingHourmeterForSprayingError())
}
```

Also update the `FinalizeFieldTicketResponse` type union:
```ts
type FinalizeFieldTicketResponse = Either<
  | FieldTicketNotFoundError
  | InvalidStatusTransitionError
  | VehicleNotFoundError
  | ImplementNotFoundError
  | MissingHourmeterForSprayingError,
  { data: FieldTicket }
>
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd backend && contextzip pnpm test finalize-field-ticket.spec.ts
```

Expected: all tests PASS

- [ ] **Step 5: Run full backend test suite**

```bash
cd backend && contextzip pnpm test
```

Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add backend/src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts \
        backend/src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts
git commit -m "feat(field-ticket): require hourmeter for SPRAYING finalization"
```

---

## Task 5: E2E — ReviewFieldTicket controller

**Files:**
- Create: `backend/src/infra/http/controllers/field-ticket/__tests__/review-field-ticket.controller.e2e-spec.ts`

- [ ] **Step 1: Create the E2E spec**

```ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import { hash } from 'bcryptjs'
import request from 'supertest'
import { UserDatabaseModule } from '@/infra/database/prisma/repositories/user/user-database.module'
import { CryptographyModule } from '@/infra/cryptography/cryptography.module'
import { UserFactory } from 'test/factories/make-user'
import { FieldTicketFactory } from 'test/factories/make-field-ticket'
import { PrismaService } from '@/infra/database/prisma/prisma.service'
import { TokenService } from '@/infra/auth/jwt/token.service'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'

const endpoint = (id: string) => `/v1/field-tickets/${id}/review`

describe('Review Field Ticket (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let userFactory: UserFactory
  let fieldTicketFactory: FieldTicketFactory
  let tokenService: TokenService
  let accessToken: { token: string; expiresIn: number }
  let scheduleId: string
  let fieldId: string
  let harvestId: string

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule, UserDatabaseModule, CryptographyModule],
      providers: [UserFactory, FieldTicketFactory, TokenService],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    userFactory = moduleRef.get(UserFactory)
    fieldTicketFactory = moduleRef.get(FieldTicketFactory)
    tokenService = moduleRef.get(TokenService)

    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "field_tickets" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "schedules" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "harvests" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "fields" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "users" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "audit_logs" CASCADE')

    const actor = await userFactory.makePrismaUser({
      email: 'admin@example.com',
      password: await hash('123456', 8),
      role: 'ADMIN',
    })

    accessToken = await tokenService.generateAccessToken({
      sub: actor.id.toString(),
      role: 'ADMIN',
    })

    // Seed supporting records
    const cropType = await prisma.cropType.create({ data: { name: 'Soja' } })
    const variety = await prisma.variety.create({ data: { name: 'Variedade X', cropTypeId: cropType.id } })
    const field_ = await prisma.field.create({ data: { name: 'A1', totalArea: 10, cultivatedArea: 8 } })
    const harvest = await prisma.harvest.create({
      data: {
        fieldId: field_.id,
        varietyId: variety.id,
        startDate: new Date('2026-01-01'),
        expectedEndDate: new Date('2026-12-31'),
      },
    })
    const schedule = await prisma.schedule.create({ data: { harvestId: harvest.id } })

    fieldId = field_.id
    harvestId = harvest.id
    scheduleId = schedule.id
  })

  it('should review a FERTIGATION ticket without vehicleId/implementId', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'FERTIGATION',
      status: 'DRAFT',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({})

    // Assert
    expect(res.statusCode).toBe(200)
    expect(res.body.data.status).toBe('REVIEWED')
  })

  it('should return 422 when SPRAYING ticket reviewed without vehicleId', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'SPRAYING',
      status: 'DRAFT',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({ implementId: '00000000-0000-4000-8000-000000000001' })

    // Assert
    expect(res.statusCode).toBe(422)
  })

  it('should return 422 when SPRAYING ticket reviewed without implementId', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'SPRAYING',
      status: 'DRAFT',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({ vehicleId: '00000000-0000-4000-8000-000000000001' })

    // Assert
    expect(res.statusCode).toBe(422)
  })
})
```

- [ ] **Step 2: Run E2E to verify**

```bash
cd backend && contextzip pnpm test:e2e review-field-ticket.controller.e2e-spec.ts
```

Expected: all 3 tests PASS

- [ ] **Step 3: Commit**

```bash
git add backend/src/infra/http/controllers/field-ticket/__tests__/review-field-ticket.controller.e2e-spec.ts
git commit -m "test(field-ticket): E2E tests for review controller type-discriminated validation"
```

---

## Task 6: E2E — FinalizeFieldTicket controller

**Files:**
- Create: `backend/src/infra/http/controllers/field-ticket/__tests__/finalize-field-ticket.controller.e2e-spec.ts`

- [ ] **Step 1: Create the E2E spec**

```ts
import { INestApplication, VersioningType } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import { hash } from 'bcryptjs'
import request from 'supertest'
import { UserDatabaseModule } from '@/infra/database/prisma/repositories/user/user-database.module'
import { CryptographyModule } from '@/infra/cryptography/cryptography.module'
import { UserFactory } from 'test/factories/make-user'
import { FieldTicketFactory } from 'test/factories/make-field-ticket'
import { PrismaService } from '@/infra/database/prisma/prisma.service'
import { TokenService } from '@/infra/auth/jwt/token.service'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'

const endpoint = (id: string) => `/v1/field-tickets/${id}/finalize`

describe('Finalize Field Ticket (E2E)', () => {
  let app: INestApplication
  let prisma: PrismaService
  let userFactory: UserFactory
  let fieldTicketFactory: FieldTicketFactory
  let tokenService: TokenService
  let accessToken: { token: string; expiresIn: number }
  let scheduleId: string
  let fieldId: string
  let harvestId: string

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule, UserDatabaseModule, CryptographyModule],
      providers: [UserFactory, FieldTicketFactory, TokenService],
    }).compile()

    app = moduleRef.createNestApplication()
    app.enableVersioning({ type: VersioningType.URI })

    prisma = moduleRef.get(PrismaService)
    userFactory = moduleRef.get(UserFactory)
    fieldTicketFactory = moduleRef.get(FieldTicketFactory)
    tokenService = moduleRef.get(TokenService)

    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  beforeEach(async () => {
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "field_tickets" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "schedules" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "harvests" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "fields" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "users" CASCADE')
    await prisma.$executeRawUnsafe('TRUNCATE TABLE "audit_logs" CASCADE')

    const actor = await userFactory.makePrismaUser({
      email: 'admin@example.com',
      password: await hash('123456', 8),
      role: 'ADMIN',
    })

    accessToken = await tokenService.generateAccessToken({
      sub: actor.id.toString(),
      role: 'ADMIN',
    })

    const cropType = await prisma.cropType.create({ data: { name: 'Soja' } })
    const variety = await prisma.variety.create({ data: { name: 'Variedade X', cropTypeId: cropType.id } })
    const field_ = await prisma.field.create({ data: { name: 'A1', totalArea: 10, cultivatedArea: 8 } })
    const harvest = await prisma.harvest.create({
      data: {
        fieldId: field_.id,
        varietyId: variety.id,
        startDate: new Date('2026-01-01'),
        expectedEndDate: new Date('2026-12-31'),
      },
    })
    const schedule = await prisma.schedule.create({ data: { harvestId: harvest.id } })

    fieldId = field_.id
    harvestId = harvest.id
    scheduleId = schedule.id
  })

  it('should finalize a FERTIGATION ticket without hourmeter', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'FERTIGATION',
      status: 'PRINTED',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({
        startTime: '08:00',
        endTime: '12:00',
        operatorName: 'Maria',
      })

    // Assert
    expect(res.statusCode).toBe(200)
    expect(res.body.data.status).toBe('COMPLETED')
  })

  it('should return 422 when SPRAYING ticket finalized without hourmeter', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'SPRAYING',
      status: 'PRINTED',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({
        startTime: '08:00',
        endTime: '12:00',
        operatorName: 'João',
      })

    // Assert
    expect(res.statusCode).toBe(422)
  })

  it('should finalize a SPRAYING ticket with all required fields', async () => {
    // Arrange
    const ticket = await fieldTicketFactory.makePrismaFieldTicket({
      scheduleId: new UniqueEntityID(scheduleId),
      fieldId: new UniqueEntityID(fieldId),
      harvestId: new UniqueEntityID(harvestId),
      date: new Date('2026-06-15'),
      operationType: 'SPRAYING',
      status: 'PRINTED',
    })

    // Act
    const res = await request(app.getHttpServer())
      .patch(endpoint(ticket.id.toString()))
      .set('Authorization', `Bearer ${accessToken.token}`)
      .send({
        startTime: '08:00',
        endTime: '12:00',
        hourmeterStart: 100,
        hourmeterEnd: 104,
        operatorName: 'João',
      })

    // Assert
    expect(res.statusCode).toBe(200)
    expect(res.body.data.status).toBe('COMPLETED')
  })
})
```

- [ ] **Step 2: Run E2E to verify**

```bash
cd backend && contextzip pnpm test:e2e finalize-field-ticket.controller.e2e-spec.ts
```

Expected: all 3 tests PASS

- [ ] **Step 3: Commit**

```bash
git add backend/src/infra/http/controllers/field-ticket/__tests__/finalize-field-ticket.controller.e2e-spec.ts
git commit -m "test(field-ticket): E2E tests for finalize controller type-discriminated validation"
```

---

## Task 7: Frontend — ConfigureFieldTicketSheet: hide equipment for FERTIGATION

**Files:**
- Modify: `frontend/src/domains/field-ticket/components/sheets/configure-field-ticket-sheet.tsx`

- [ ] **Step 1: Wrap equipment section in operationType guard**

The equipment section runs from the `<InputField name="waterL" .../>` block through `<InputField name="ph" .../>` (approximately lines 232–314 in the JSX). Wrap it:

```tsx
{ticket.operationType === 'SPRAYING' && (
  <>
    <InputField
      control={form.control}
      name="waterL"
      label="Volume de Água (L)"
      inputMode="decimal"
      placeholder="0"
      data-testid="configure-water"
    />

    <div className="grid grid-cols-2 gap-4">
      <SelectField
        control={form.control}
        name="bar"
        label="Barra"
        placeholder="Selecione..."
        options={[
          { label: 'Alta', value: 'ALTA' },
          { label: 'Baixa', value: 'BAIXA' },
        ]}
        data-testid="configure-bar"
      />
      <SelectField
        control={form.control}
        name="turbine"
        label="Turbina"
        placeholder="Selecione..."
        options={[
          { label: 'Com', value: 'COM' },
          { label: 'Sem', value: 'SEM' },
        ]}
        data-testid="configure-turbine"
      />
    </div>

    <div className="grid grid-cols-2 gap-4">
      <InputField
        control={form.control}
        name="nozzleCount"
        label="Bicos"
        inputMode="numeric"
        placeholder="0"
        data-testid="configure-nozzle-count"
      />
      <InputField
        control={form.control}
        name="pressure"
        label="Pressão"
        inputMode="decimal"
        placeholder="0.0"
        data-testid="configure-pressure"
      />
    </div>

    <div className="grid grid-cols-2 gap-4">
      <InputField
        control={form.control}
        name="gearNumber"
        label="Marcha"
        inputMode="numeric"
        placeholder="0"
        data-testid="configure-gear-number"
      />
      <SelectField
        control={form.control}
        name="gearType"
        label="Tipo de Marcha"
        placeholder="Selecione..."
        options={[
          { label: 'Simples', value: 'SIMPLE' },
          { label: 'Reduzida', value: 'REDUCED' },
        ]}
        data-testid="configure-gear-type"
      />
    </div>

    <InputField
      control={form.control}
      name="ph"
      label="pH"
      inputMode="decimal"
      placeholder="0.0"
      data-testid="configure-ph"
    />

    <Separator />
  </>
)}
```

Remove the standalone `<Separator />` that currently sits between equipment and inputs (it's now inside the SPRAYING block above).

- [ ] **Step 2: Build frontend to check for type errors**

```bash
cd frontend && contextzip pnpm build
```

Expected: build PASS with no TypeScript errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/domains/field-ticket/components/sheets/configure-field-ticket-sheet.tsx
git commit -m "feat(field-ticket): hide equipment section in review form for FERTIGATION tickets"
```

---

## Task 8: Frontend — FinalizeFieldTicketSheet: operationType-aware

**Files:**
- Modify: `frontend/src/domains/field-ticket/components/sheets/finalize-field-ticket-sheet.tsx`

- [ ] **Step 1: Add operationType state and fetch it from ticket**

After the `const [operatorOptions, ...]` line, add:

```ts
const [operationType, setOperationType] = useState<'SPRAYING' | 'FERTIGATION'>('SPRAYING')
```

In the `useEffect` that calls `findFieldTicketById`, update to capture `operationType`:

```ts
Promise.all([
  findFieldTicketById(finalizeId),
  listInputs({ active: true, perPage: 500 }),
  listPositions({ query: ticketOpType === 'FERTIGATION' ? 'irrigante' : 'tratorista', perPage: 100 }),
])
```

But `operationType` isn't known until `findFieldTicketById` resolves. Use a two-step approach — first get the ticket, then list employees:

```ts
useEffect(() => {
  if (!finalizeId) return
  let cancelled = false

  Promise.all([
    findFieldTicketById(finalizeId),
    listInputs({ active: true, perPage: 500 }),
  ])
    .then(async ([ticketData, inputsData]) => {
      if (cancelled) return

      const opType = ticketData.operationType as 'SPRAYING' | 'FERTIGATION'
      setOperationType(opType)

      const ticketInputs = ticketData.inputs ?? []
      setInputs(ticketInputs)

      const names: Record<string, string> = {}
      const units: Record<string, string> = {}
      for (const inv of inputsData.data) {
        names[inv.id] = inv.name
        units[inv.id] = inv.unitOfMeasure
      }
      setInputNames(names)
      setInputUnits(units)

      const dosages: Record<string, string> = {}
      for (const inp of ticketInputs) {
        dosages[inp.inputId] = String(inp.dosagePer100L)
      }
      setExecutedDosages(dosages)

      const positionQuery = opType === 'FERTIGATION' ? 'irrigante' : 'tratorista'
      const positionsData = await listPositions({ query: positionQuery, perPage: 100 })
      if (cancelled) return

      const posId = positionsData.data[0]?.id
      if (posId) {
        const employeesData = await listEmployees({
          positionId: posId,
          status: 'ACTIVE',
          perPage: 500,
        })
        if (!cancelled) {
          setOperatorOptions(
            employeesData.data.map((e) => ({ value: e.name, label: e.name })),
          )
        }
      }
    })
    .catch(() => {
      if (cancelled) return
    })

  return () => {
    cancelled = true
    setInputs([])
    setInputNames({})
    setInputUnits({})
    setExecutedDosages({})
    setOperatorOptions([])
    setOperationType('SPRAYING')
  }
}, [finalizeId])
```

- [ ] **Step 2: Update Zod schema to make hourmeter optional**

Replace the current `schema` definition (lines 35–65):

```ts
const schema = z
  .object({
    operatorName: z.string().min(1, 'Campo obrigatório'),
    startTime: z.string().min(1, 'Hora de início é obrigatória'),
    endTime: z.string().min(1, 'Hora de término é obrigatória'),
    hourmeterStart: z.string().optional(),
    hourmeterEnd: z.string().optional(),
  })
  .refine(
    (data) => {
      if (!data.startTime || !data.endTime) return true
      return data.endTime >= data.startTime
    },
    {
      message: 'Hora de término deve ser posterior ao início',
      path: ['endTime'],
    },
  )
  .refine(
    (data) => {
      const start = data.hourmeterStart ? Number(data.hourmeterStart) : undefined
      const end = data.hourmeterEnd ? Number(data.hourmeterEnd) : undefined
      if (start === undefined || end === undefined) return true
      return end >= start
    },
    {
      message: 'Horímetro fim deve ser maior ou igual ao início',
      path: ['hourmeterEnd'],
    },
  )
```

- [ ] **Step 3: Add runtime hourmeter validation in onSubmit (SPRAYING)**

In `onSubmit`, before calling `finalizeFieldTicket`:

```ts
async function onSubmit(data: FormData) {
  if (operationType === 'SPRAYING') {
    if (!data.hourmeterStart || data.hourmeterStart.trim() === '') {
      form.setError('hourmeterStart', { message: 'Horímetro início é obrigatório' })
      return
    }
    if (!data.hourmeterEnd || data.hourmeterEnd.trim() === '') {
      form.setError('hourmeterEnd', { message: 'Horímetro fim é obrigatório' })
      return
    }
  }

  const toastId = toast.loading('Finalizando boleta...')
  // ...rest unchanged
```

- [ ] **Step 4: Render operator label and hourmeter fields conditionally**

Change the operator `SelectField`/`InputField` labels:

```tsx
const operatorLabel = operationType === 'FERTIGATION' ? 'Irrigante' : 'Operador'
const operatorPlaceholder = operationType === 'FERTIGATION'
  ? 'Selecione o irrigante'
  : 'Selecione o tratorista'
const operatorFallbackPlaceholder = operationType === 'FERTIGATION'
  ? 'Nome do irrigante'
  : 'Nome do operador'

// In JSX replace:
// label="Operador" → label={operatorLabel}
// placeholder="Selecione o tratorista" → placeholder={operatorPlaceholder}
// placeholder="Nome do operador" → placeholder={operatorFallbackPlaceholder}
```

Wrap the hourmeter block in a SPRAYING guard:

```tsx
{operationType === 'SPRAYING' && (
  <div className="grid grid-cols-2 gap-4">
    <InputField
      control={form.control}
      name="hourmeterStart"
      label="Horímetro início"
      inputMode="decimal"
      placeholder="0.0"
      data-testid="finalize-hourmeter-start"
    />
    <InputField
      control={form.control}
      name="hourmeterEnd"
      label="Horímetro fim"
      inputMode="decimal"
      placeholder="0.0"
      data-testid="finalize-hourmeter-end"
    />
  </div>
)}
```

- [ ] **Step 5: Build frontend**

```bash
cd frontend && contextzip pnpm build
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add frontend/src/domains/field-ticket/components/sheets/finalize-field-ticket-sheet.tsx
git commit -m "feat(field-ticket): type-aware finalization — Irrigante label + no hourmeter for FERTIGATION"
```

---

## Task 9: Frontend — Print layout: 8/page for FERTIGATION

**Files:**
- Modify: `frontend/src/domains/field-ticket/components/print/print-field-ticket-layout.tsx`

- [ ] **Step 1: Add FertigationTicketLayout component**

After the existing `TicketLayout` component (around line 405), add a new compact layout:

```tsx
function FertigationTicketLayout({
  ticket,
  fieldName,
  inputNames,
  inputUnits,
  harvestStart,
}: {
  ticket: TicketWithInputs
  fieldName: string
  inputNames: Record<string, string>
  inputUnits: Record<string, string>
  harvestStart?: Date
}) {
  const days = harvestStart ? calcDays(ticket.date, harvestStart) : null
  const ticketDate = new Date(ticket.date)

  return (
    <div
      style={{
        border: '1px solid #000',
        height: '100%',
        display: 'flex',
        flexDirection: 'column',
        fontFamily: 'Arial, sans-serif',
        fontSize: '8pt',
        boxSizing: 'border-box',
        pageBreakInside: 'avoid',
        overflow: 'hidden',
      }}
    >
      {/* Title */}
      <div
        style={{
          background: '#2a6496',
          color: '#fff',
          textAlign: 'center',
          fontWeight: 'bold',
          fontSize: '7pt',
          padding: '1px 4px',
          letterSpacing: '1px',
          textTransform: 'uppercase',
          borderBottom: '1px solid #000',
        }}
      >
        FERTIRRIGAÇÃO
      </div>

      {/* Row: PARCELA | DATA | DIAS */}
      <div
        style={{
          display: 'grid',
          gridTemplateColumns: '1fr 1fr 0.8fr',
          borderBottom: '1px solid #000',
        }}
      >
        <div style={cell({ borderRight: '1px solid #000' })}>
          <div style={labelStyle}>Parcela</div>
          <div
            style={{
              background: '#f5c842',
              fontSize: '12pt',
              fontWeight: 'bold',
              textAlign: 'center',
              padding: '1px 4px',
              lineHeight: 1.1,
            }}
          >
            {fieldName || '—'}
          </div>
        </div>
        <div style={cell({ borderRight: '1px solid #000' })}>
          <div style={labelStyle}>Data</div>
          <div
            style={{
              fontSize: '12pt',
              fontWeight: 'bold',
              textAlign: 'center',
              padding: '1px 4px',
              lineHeight: 1.1,
            }}
          >
            {formatTicketDate(ticketDate)}
          </div>
        </div>
        <div>
          <div style={labelStyle}>Dias</div>
          <div
            style={{
              fontSize: '9pt',
              fontWeight: 'bold',
              textAlign: 'center',
              padding: '0 4px',
            }}
          >
            {days ? `${days.phase} ${days.day}` : '—'}
          </div>
        </div>
      </div>

      {/* Inputs */}
      <div style={{ flex: 1, overflow: 'hidden', minHeight: 0 }}>
        {ticket.inputs.length === 0 ? (
          <div style={{ padding: '4px', color: '#999', fontSize: '7pt' }}>Sem insumos</div>
        ) : (
          ticket.inputs.map((input, idx) => {
            const unit = inputUnits[input.inputId] ?? ''
            return (
              <div
                key={input.id}
                style={{
                  display: 'flex',
                  justifyContent: 'space-between',
                  alignItems: 'baseline',
                  padding: '1px 5px',
                  borderBottom: idx < ticket.inputs.length - 1 ? '1px dotted #999' : undefined,
                  color: input.highlighted ? '#cc0000' : '#000',
                  fontWeight: input.highlighted ? 'bold' : 'normal',
                  fontSize: '8pt',
                }}
              >
                <span style={{ marginRight: '6px', overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap' }}>
                  {inputNames[input.inputId] ?? '—'}
                </span>
                <span style={{ flexShrink: 0, whiteSpace: 'nowrap' }}>
                  {input.dosagePer100L}
                  {unit && <span style={{ fontSize: '6.5pt' }}>{` ${unit}/100L`}</span>}
                </span>
              </div>
            )
          })
        )}
      </div>

      {/* Footer: Hora | Irrigante */}
      <div style={{ borderTop: '1px solid #000', display: 'flex' }}>
        <div style={{ flex: 1, borderRight: '1px solid #000', display: 'flex', flexDirection: 'column' }}>
          <div style={sectionLabelStyle}>Hora</div>
          <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr' }}>
            <div style={cell({ borderRight: '1px solid #ccc' })}>
              <div style={labelStyle}>Início</div>
              <div style={{ padding: '2px 4px', fontSize: '9pt', fontWeight: 'bold', lineHeight: 1.1 }}>
                {ticket.startTime ?? '\u00A0'}
              </div>
            </div>
            <div>
              <div style={labelStyle}>Fim</div>
              <div style={{ padding: '2px 4px', fontSize: '9pt', fontWeight: 'bold', lineHeight: 1.1 }}>
                {ticket.endTime ?? '\u00A0'}
              </div>
            </div>
          </div>
        </div>
        <div style={{ flex: 0.7, display: 'flex', flexDirection: 'column', overflow: 'hidden' }}>
          <div style={labelStyle}>Irrigante</div>
          <div
            style={{
              flex: 1,
              padding: '2px 4px',
              fontSize: '9pt',
              fontWeight: 'bold',
              lineHeight: 1.1,
              overflow: 'hidden',
              textOverflow: 'ellipsis',
              whiteSpace: 'nowrap',
            }}
          >
            {ticket.operatorName ?? '\u00A0'}
          </div>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Update PrintFieldTicketLayout to group by type and use per-type page size**

Replace the current `PrintFieldTicketLayout` function body:

```tsx
export function PrintFieldTicketLayout({
  tickets,
  fieldNames,
  inputNames,
  inputUnits,
  vehicleCodes = {},
  implementCodes = {},
  harvestStartDates = {},
}: Props) {
  const spraying = tickets.filter((t) => t.operationType === 'SPRAYING')
  const fertigation = tickets.filter((t) => t.operationType === 'FERTIGATION')

  const sprayingPages: TicketWithInputs[][] = []
  for (let i = 0; i < spraying.length; i += 6) {
    sprayingPages.push(spraying.slice(i, i + 6))
  }

  const fertigationPages: TicketWithInputs[][] = []
  for (let i = 0; i < fertigation.length; i += 8) {
    fertigationPages.push(fertigation.slice(i, i + 8))
  }

  return (
    <>
      <style>{`
        @media screen {
          body { background: #f0f0f0; }
          [data-print-page] {
            width: 297mm;
            min-height: 210mm;
            margin: 16px auto;
            background: white;
            box-shadow: 0 2px 8px rgba(0,0,0,0.15);
          }
        }
        @media print {
          @page { size: A4 landscape; margin: 0; }
          * {
            -webkit-print-color-adjust: exact !important;
            print-color-adjust: exact !important;
            color-adjust: exact !important;
          }
          html, body { margin: 0 !important; padding: 0 !important; }
          body * { visibility: hidden; }
          [data-print-root], [data-print-root] * { visibility: visible; }
          [data-print-root] { position: fixed; top: 0; left: 0; width: 100%; }
          [data-print-page] {
            width: 100%;
            height: 210mm;
            min-height: 0 !important;
            padding: 8mm !important;
            page-break-after: always;
            box-shadow: none !important;
            margin: 0 !important;
          }
          [data-print-page]:last-child { page-break-after: avoid; }
        }
      `}</style>
      <div data-print-root>
        {sprayingPages.map((page, pageIdx) => (
          <div
            key={`spraying-${pageIdx}`}
            data-print-page
            style={{
              display: 'grid',
              gridTemplateColumns: 'repeat(3, 1fr)',
              gridTemplateRows: 'repeat(2, 1fr)',
              gap: '4px',
              padding: '8px',
              boxSizing: 'border-box',
              minHeight: '210mm',
            }}
          >
            {page.map((ticket) => (
              <TicketLayout
                key={ticket.id}
                ticket={ticket}
                fieldName={fieldNames[ticket.fieldId] ?? '—'}
                inputNames={inputNames}
                inputUnits={inputUnits}
                vehicleCode={ticket.vehicleId ? vehicleCodes[ticket.vehicleId] : undefined}
                implementCode={ticket.implementId ? implementCodes[ticket.implementId] : undefined}
                harvestStart={harvestStartDates[ticket.harvestId]}
              />
            ))}
          </div>
        ))}
        {fertigationPages.map((page, pageIdx) => (
          <div
            key={`fertigation-${pageIdx}`}
            data-print-page
            style={{
              display: 'grid',
              gridTemplateColumns: 'repeat(4, 1fr)',
              gridTemplateRows: 'repeat(2, 1fr)',
              gap: '4px',
              padding: '8px',
              boxSizing: 'border-box',
              minHeight: '210mm',
            }}
          >
            {page.map((ticket) => (
              <FertigationTicketLayout
                key={ticket.id}
                ticket={ticket}
                fieldName={fieldNames[ticket.fieldId] ?? '—'}
                inputNames={inputNames}
                inputUnits={inputUnits}
                harvestStart={harvestStartDates[ticket.harvestId]}
              />
            ))}
          </div>
        ))}
      </div>
    </>
  )
}
```

- [ ] **Step 3: Build frontend**

```bash
cd frontend && contextzip pnpm build
```

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add frontend/src/domains/field-ticket/components/print/print-field-ticket-layout.tsx
git commit -m "feat(field-ticket): compact FERTIGATION print layout — 8 per A4 landscape"
```

---

## Task 10: Playwright E2E — review and finalization flows

**Files:**
- Create: `frontend/tests/field-ticket/review-finalize-field-ticket.e2e-spec.ts`
- Create or extend: `frontend/tests/mocks/handlers/field-ticket.handler.ts`

> Before writing tests, read the E2E pattern files:
> - `docs/coding-patterns/frontend/e2e-test/test-structure.md`
> - `docs/coding-patterns/frontend/e2e-test/handler.md`
> - `docs/coding-patterns/frontend/e2e-test/auth-fixture.md`

- [ ] **Step 1: Add field-ticket MSW handler**

Check `frontend/tests/mocks/handlers/handlers.ts` for existing handler. If not present, create `frontend/tests/mocks/handlers/field-ticket.handler.ts` and register it in `handlers.ts`.

The handler needs endpoints:
- `GET /v1/field-tickets/:id` — returns ticket with operationType + inputs
- `PATCH /v1/field-tickets/:id/review` — returns 200 + updated ticket
- `PATCH /v1/field-tickets/:id/finalize` — returns 200 + updated ticket

```ts
import { http, HttpResponse } from 'msw'

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:4333'

const SPRAYING_TICKET = {
  id: '00000000-0000-4000-8000-000000000010',
  scheduleId: '00000000-0000-4000-8000-000000000020',
  fieldId: '00000000-0000-4000-8000-000000000030',
  harvestId: '00000000-0000-4000-8000-000000000040',
  date: '2026-06-15T00:00:00.000Z',
  operationType: 'SPRAYING',
  status: 'DRAFT',
  vehicleId: null,
  implementId: null,
  waterL: null,
  bar: null,
  turbine: null,
  nozzleCount: null,
  pressure: null,
  gearNumber: null,
  gearType: null,
  ph: null,
  operatorName: null,
  startTime: null,
  endTime: null,
  hourmeterStart: null,
  hourmeterEnd: null,
  cancelReason: null,
  createdAt: '2026-06-01T00:00:00.000Z',
  updatedAt: null,
  inputs: [],
}

const FERTIGATION_TICKET = {
  ...SPRAYING_TICKET,
  id: '00000000-0000-4000-8000-000000000011',
  operationType: 'FERTIGATION',
}

export const fieldTicketHandlers = [
  http.get(`${API_URL}/v1/field-tickets/:id`, ({ params }) => {
    const id = params.id as string
    if (id === SPRAYING_TICKET.id) return HttpResponse.json({ data: SPRAYING_TICKET })
    if (id === FERTIGATION_TICKET.id) return HttpResponse.json({ data: FERTIGATION_TICKET })
    return new HttpResponse(null, { status: 404 })
  }),

  http.patch(`${API_URL}/v1/field-tickets/:id/review`, ({ params }) => {
    const id = params.id as string
    const ticket = id === SPRAYING_TICKET.id ? SPRAYING_TICKET : FERTIGATION_TICKET
    return HttpResponse.json({ data: { ...ticket, status: 'REVIEWED' } })
  }),

  http.patch(`${API_URL}/v1/field-tickets/:id/finalize`, ({ params }) => {
    const id = params.id as string
    const ticket = id === SPRAYING_TICKET.id ? SPRAYING_TICKET : FERTIGATION_TICKET
    return HttpResponse.json({ data: { ...ticket, status: 'COMPLETED' } })
  }),
]
```

- [ ] **Step 2: Write E2E test file**

```ts
import { test, expect } from '@playwright/test'
import { createAuthenticatedFixture } from '../fixtures/auth.fixture'

const authenticatedTest = createAuthenticatedFixture()

authenticatedTest.describe.serial('SPRAYING review form', () => {
  authenticatedTest('shows equipment section for SPRAYING ticket', async ({ page }) => {
    // Arrange — navigate to execution panel or schedule page where configure sheet is triggered
    // (Adjust path to match actual route that opens the configure sheet)
    await page.goto('/execution')

    // Open configure sheet for SPRAYING ticket (assumes it's visible in execution panel)
    await page.getByTestId(`configure-sheet-trigger-00000000-0000-4000-8000-000000000010`).click()
    await page.getByTestId('field-ticket-configure-form').waitFor()

    // Assert equipment fields are visible
    await expect(page.getByTestId('configure-water')).toBeVisible()
    await expect(page.getByTestId('configure-bar')).toBeVisible()
    await expect(page.getByTestId('configure-turbine')).toBeVisible()
    await expect(page.getByTestId('configure-nozzle-count')).toBeVisible()
    await expect(page.getByTestId('configure-pressure')).toBeVisible()
    await expect(page.getByTestId('configure-gear-number')).toBeVisible()
    await expect(page.getByTestId('configure-ph')).toBeVisible()
  })
})

authenticatedTest.describe.serial('FERTIGATION review form', () => {
  authenticatedTest('hides equipment section for FERTIGATION ticket', async ({ page }) => {
    // Arrange
    await page.goto('/execution')
    await page.getByTestId(`configure-sheet-trigger-00000000-0000-4000-8000-000000000011`).click()
    await page.getByTestId('field-ticket-configure-form').waitFor()

    // Assert equipment fields are NOT present
    await expect(page.getByTestId('configure-water')).not.toBeVisible()
    await expect(page.getByTestId('configure-bar')).not.toBeVisible()
    await expect(page.getByTestId('configure-turbine')).not.toBeVisible()
    await expect(page.getByTestId('configure-ph')).not.toBeVisible()

    // Insumos section still visible
    await expect(page.getByTestId('configure-pending-input-select')).toBeVisible()
  })
})

authenticatedTest.describe.serial('SPRAYING finalization form', () => {
  authenticatedTest('shows Operador label and hourmeter fields', async ({ page }) => {
    // Arrange — open finalize sheet for SPRAYING ticket
    await page.goto('/execution')
    await page.getByTestId(`finalize-sheet-trigger-00000000-0000-4000-8000-000000000010`).click()
    await page.getByTestId('field-ticket-finalize-form').waitFor()

    // Assert
    await expect(page.getByLabel('Operador')).toBeVisible()
    await expect(page.getByTestId('finalize-hourmeter-start')).toBeVisible()
    await expect(page.getByTestId('finalize-hourmeter-end')).toBeVisible()
  })

  authenticatedTest('requires hourmeter for SPRAYING', async ({ page }) => {
    await page.goto('/execution')
    await page.getByTestId(`finalize-sheet-trigger-00000000-0000-4000-8000-000000000010`).click()
    await page.getByTestId('field-ticket-finalize-form').waitFor()

    // Fill name and time but not hourmeter
    await page.getByTestId('finalize-operator-name').fill('João')
    await page.getByTestId('finalize-start-time').fill('08:00')
    await page.getByTestId('finalize-end-time').fill('12:00')

    await page.getByTestId('finalize-submit').click()

    // Assert validation error
    await expect(page.getByText('Horímetro início é obrigatório')).toBeVisible()
  })
})

authenticatedTest.describe.serial('FERTIGATION finalization form', () => {
  authenticatedTest('shows Irrigante label and hides hourmeter', async ({ page }) => {
    await page.goto('/execution')
    await page.getByTestId(`finalize-sheet-trigger-00000000-0000-4000-8000-000000000011`).click()
    await page.getByTestId('field-ticket-finalize-form').waitFor()

    // Assert label is Irrigante
    await expect(page.getByLabel('Irrigante')).toBeVisible()

    // Assert hourmeter not visible
    await expect(page.getByTestId('finalize-hourmeter-start')).not.toBeVisible()
    await expect(page.getByTestId('finalize-hourmeter-end')).not.toBeVisible()
  })

  authenticatedTest('submits FERTIGATION without hourmeter', async ({ page }) => {
    await page.goto('/execution')
    await page.getByTestId(`finalize-sheet-trigger-00000000-0000-4000-8000-000000000011`).click()
    await page.getByTestId('field-ticket-finalize-form').waitFor()

    await page.getByTestId('finalize-operator-name').fill('Maria')
    await page.getByTestId('finalize-start-time').fill('08:00')
    await page.getByTestId('finalize-end-time').fill('12:00')

    await page.getByTestId('finalize-submit').click()

    await expect(page.getByText('Boleta finalizada com sucesso.')).toBeVisible()
  })
})
```

> **Note:** The actual route and trigger test-ids depend on how the execution panel opens these sheets. Adjust the navigation path and trigger test-ids to match the real implementation. If `configure-sheet-trigger-{id}` doesn't exist as a data-testid, add it to the trigger button in the execution panel component.

- [ ] **Step 3: Run Playwright tests**

```bash
cd frontend && contextzip pnpm test:e2e tests/field-ticket/review-finalize-field-ticket.e2e-spec.ts
```

Expected: all tests PASS (adjust trigger test-ids if needed)

- [ ] **Step 4: Commit**

```bash
git add frontend/tests/field-ticket/review-finalize-field-ticket.e2e-spec.ts \
        frontend/tests/mocks/handlers/field-ticket.handler.ts \
        frontend/tests/mocks/handlers/handlers.ts
git commit -m "test(field-ticket): Playwright E2E for SPRAYING/FERTIGATION review and finalization flows"
```

---

## Task 11: Update docs/rules/field-ticket.md

**Files:**
- Modify: `docs/rules/field-ticket.md`

- [ ] **Step 1: Read the current invariants section**

Read `docs/rules/field-ticket.md` — focus on the Entities > FieldTicket > Invariants block.

- [ ] **Step 2: Update invariants**

Replace the current invariant:
> Cannot transition to REVIEWED without vehicleId and implementId

With:
> Cannot transition to REVIEWED without vehicleId and implementId **when operationType is SPRAYING**. FERTIGATION tickets can be reviewed with no equipment fields.

Add below the existing hourmeter-related invariants:
> SPRAYING finalization requires hourmeterStart and hourmeterEnd. FERTIGATION finalization does not.

- [ ] **Step 3: Update Print Layout section**

Add to the Printing Layout section:
> FERTIGATION tickets use a compact layout: 8 per A4 landscape. No equipment block. Irrigante label replaces Tratorista.

- [ ] **Step 4: Commit**

```bash
git add docs/rules/field-ticket.md
git commit -m "docs(field-ticket): update invariants and print layout for SPRAYING/FERTIGATION distinction"
```
