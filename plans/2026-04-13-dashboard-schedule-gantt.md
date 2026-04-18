# Dashboard Schedule Gantt — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a proportional Gantt visualization of all ACTIVE schedules to the Dashboard, grouped by field (talhao), with inline expand on row click.

**Architecture:** Dedicated backend endpoint `GET /v1/schedules/dashboard` returns ACTIVE schedules hydrated with harvest/field/cropType/variety data and field tickets aggregated into operations. Frontend renders a proportional date-axis Gantt on the Dashboard page with expandable rows. Cross-domain data fetched via QueryBus (existing `FindHarvestQuery`, `FindFieldTicketsByScheduleIdQuery`, plus new queries for field and cropType/variety names).

**Tech Stack:** NestJS, Prisma, Zod, Next.js (Server Components), Tailwind CSS, MSW, Vitest, Playwright

---

## File Structure

### Backend — New files

| File | Responsibility |
|---|---|
| `backend/src/domain/schedule/application/use-cases/get-dashboard-schedules.ts` | Use case: fetches ACTIVE schedules, hydrates via QueryBus, aggregates tickets into operations |
| `backend/src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts` | Unit test for use case |
| `backend/src/infra/http/controllers/schedule/get-dashboard-schedules.controller.ts` | HTTP controller for `GET /v1/schedules/dashboard` |
| `backend/src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts` | E2E test for controller |
| `backend/src/infra/http/presenters/schedule/dashboard-schedule.presenter.ts` | Transforms use case output to HTTP response |
| `backend/src/domain/field/application/queries/find-field.query.ts` | QueryBus query for field data |
| `backend/src/domain/field/application/handlers/find-field.handler.ts` | QueryBus handler returning field name |
| `backend/src/domain/crop/harvests/application/queries/find-harvest-with-crop.query.ts` | QueryBus query for harvest + cropType + variety names |
| `backend/src/domain/crop/harvests/application/handlers/find-harvest-with-crop.handler.ts` | QueryBus handler returning harvest with crop/variety names |

### Backend — Modified files

| File | Change |
|---|---|
| `backend/src/infra/http/controllers/schedule/schedule-http.module.ts` | Register new controller + use case |
| `backend/src/infra/query-bus/query-bus.module.ts` | Register new query handlers |

### Frontend — New files

| File | Responsibility |
|---|---|
| `frontend/src/domains/schedule/schemas/dashboard-schedule.schema.ts` | Zod schema for dashboard endpoint response |
| `frontend/src/domains/schedule/api/get-dashboard-schedules.ts` | API call to `GET /v1/schedules/dashboard` |
| `frontend/src/domains/schedule/components/dashboard/schedule-gantt-dashboard.tsx` | Container: fetches data, renders header + rows |
| `frontend/src/domains/schedule/components/dashboard/gantt-header.tsx` | Date axis with "today" marker |
| `frontend/src/domains/schedule/components/dashboard/gantt-row.tsx` | Info column + operation blocks |
| `frontend/src/domains/schedule/components/dashboard/gantt-row-detail.tsx` | Expanded content on click |
| `frontend/src/domains/schedule/components/dashboard/index.ts` | Barrel export |
| `frontend/tests/mocks/handlers/dashboard-schedule.handler.ts` | MSW handler for dashboard endpoint |

### Frontend — Modified files

| File | Change |
|---|---|
| `frontend/src/app/(private)/dashboard/page.tsx` | Import and render `ScheduleGanttDashboard` |
| `frontend/tests/mocks/handlers.ts` | Register dashboard schedule handler |

---

## Task 1: FindFieldQuery + Handler (Backend)

**Files:**
- Create: `backend/src/domain/field/application/queries/find-field.query.ts`
- Create: `backend/src/domain/field/application/handlers/find-field.handler.ts`
- Modify: `backend/src/infra/query-bus/query-bus.module.ts`

- [ ] **Step 1: Create the query class**

```typescript
// backend/src/domain/field/application/queries/find-field.query.ts
import type { Query } from '@/core/query-bus/query'

export type FindFieldQueryResult = {
  id: string
  name: string
} | null

export class FindFieldQuery implements Query {
  readonly queryName = 'FindFieldQuery'

  constructor(public readonly fieldId: string) {}
}
```

- [ ] **Step 2: Create the handler**

```typescript
// backend/src/domain/field/application/handlers/find-field.handler.ts
import { Injectable } from '@nestjs/common'
import { QueryBus } from '@/core/query-bus/query-bus'
import type { QueryHandler } from '@/core/query-bus/query-handler'
import { FieldsRepository } from '../repositories/fields-repository'
import {
  FindFieldQuery,
  type FindFieldQueryResult,
} from '../queries/find-field.query'

@Injectable()
export class FindFieldHandler
  implements QueryHandler<FindFieldQuery, FindFieldQueryResult>
{
  constructor(private fieldsRepository: FieldsRepository) {
    QueryBus.register('FindFieldQuery', this.handle.bind(this))
  }

  async handle(query: FindFieldQuery): Promise<FindFieldQueryResult> {
    const field = await this.fieldsRepository.findById(query.fieldId)

    if (!field) return null

    return {
      id: field.id.toString(),
      name: field.name,
    }
  }
}
```

- [ ] **Step 3: Register handler in QueryBusModule**

Add `FindFieldHandler` to the `providers` array in `backend/src/infra/query-bus/query-bus.module.ts`. Import `FieldDatabaseModule` in the `imports` array if not already present.

- [ ] **Step 4: Verify compilation**

Run: `cd backend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/field/application/queries/find-field.query.ts \
       backend/src/domain/field/application/handlers/find-field.handler.ts \
       backend/src/infra/query-bus/query-bus.module.ts
git commit -m "feat(field): add FindFieldQuery handler for cross-domain reads"
```

---

## Task 2: FindHarvestWithCropQuery + Handler (Backend)

The existing `FindHarvestQuery` returns only `{ id, fieldId, status, startDate, expectedEndDate }`. The dashboard needs cropType and variety names. Create a new query instead of modifying the existing one.

**Files:**
- Create: `backend/src/domain/crop/harvests/application/queries/find-harvest-with-crop.query.ts`
- Create: `backend/src/domain/crop/harvests/application/handlers/find-harvest-with-crop.handler.ts`
- Modify: `backend/src/infra/query-bus/query-bus.module.ts`

- [ ] **Step 1: Create the query class**

```typescript
// backend/src/domain/crop/harvests/application/queries/find-harvest-with-crop.query.ts
import type { Query } from '@/core/query-bus/query'

export type FindHarvestWithCropQueryResult = {
  id: string
  name: string
  fieldId: string
  cropTypeName: string
  varietyName: string
  startDate: Date
  expectedEndDate: Date
} | null

export class FindHarvestWithCropQuery implements Query {
  readonly queryName = 'FindHarvestWithCropQuery'

  constructor(public readonly harvestId: string) {}
}
```

- [ ] **Step 2: Create the handler**

The handler needs both HarvestsRepository and access to CropType/Variety. Since this is in the crop domain, it can access CropTypesRepository and VarietiesRepository directly.

```typescript
// backend/src/domain/crop/harvests/application/handlers/find-harvest-with-crop.handler.ts
import { Injectable } from '@nestjs/common'
import { QueryBus } from '@/core/query-bus/query-bus'
import type { QueryHandler } from '@/core/query-bus/query-handler'
import { HarvestsRepository } from '../repositories/harvests-repository'
import { CropTypesRepository } from '../../crop-types/application/repositories/crop-types-repository'
import { VarietiesRepository } from '../../varieties/application/repositories/varieties-repository'
import {
  FindHarvestWithCropQuery,
  type FindHarvestWithCropQueryResult,
} from '../queries/find-harvest-with-crop.query'

@Injectable()
export class FindHarvestWithCropHandler
  implements
    QueryHandler<FindHarvestWithCropQuery, FindHarvestWithCropQueryResult>
{
  constructor(
    private harvestsRepository: HarvestsRepository,
    private cropTypesRepository: CropTypesRepository,
    private varietiesRepository: VarietiesRepository,
  ) {
    QueryBus.register(
      'FindHarvestWithCropQuery',
      this.handle.bind(this),
    )
  }

  async handle(
    query: FindHarvestWithCropQuery,
  ): Promise<FindHarvestWithCropQueryResult> {
    const harvest = await this.harvestsRepository.findById(query.harvestId)
    if (!harvest) return null

    const [cropType, variety] = await Promise.all([
      this.cropTypesRepository.findById(harvest.cropTypeId.toString()),
      this.varietiesRepository.findById(harvest.varietyId.toString()),
    ])

    return {
      id: harvest.id.toString(),
      name: harvest.name,
      fieldId: harvest.fieldId.toString(),
      cropTypeName: cropType?.name ?? '',
      varietyName: variety?.name ?? '',
      startDate: harvest.startDate,
      expectedEndDate: harvest.expectedEndDate,
    }
  }
}
```

- [ ] **Step 3: Register handler in QueryBusModule**

Add `FindHarvestWithCropHandler` to providers in `backend/src/infra/query-bus/query-bus.module.ts`. Import `CropDatabaseModule` (includes harvests, crop-types, varieties) in `imports` if not already present.

- [ ] **Step 4: Verify compilation**

Run: `cd backend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/crop/harvests/application/queries/find-harvest-with-crop.query.ts \
       backend/src/domain/crop/harvests/application/handlers/find-harvest-with-crop.handler.ts \
       backend/src/infra/query-bus/query-bus.module.ts
git commit -m "feat(crop): add FindHarvestWithCropQuery for harvest + cropType + variety names"
```

---

## Task 3: GetDashboardSchedules Use Case — Unit Test (Backend)

**Files:**
- Create: `backend/src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// backend/src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { QueryBus } from '@/core/query-bus/query-bus'
import { InMemorySchedulesRepository } from 'test/repositories/schedule/in-memory-schedules-repository'
import { GetDashboardSchedulesUseCase } from '../get-dashboard-schedules'
import { makeSchedule } from 'test/factories/make-schedule'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'

let schedulesRepository: InMemorySchedulesRepository
let sut: GetDashboardSchedulesUseCase

describe('GetDashboardSchedulesUseCase', () => {
  beforeEach(() => {
    schedulesRepository = new InMemorySchedulesRepository()
    sut = new GetDashboardSchedulesUseCase(schedulesRepository)
    QueryBus.clearHandlers()
  })

  it('should return only ACTIVE schedules with hydrated data', async () => {
    const harvestId1 = new UniqueEntityID('harvest-1')
    const harvestId2 = new UniqueEntityID('harvest-2')

    const activeSchedule = makeSchedule(
      { harvestId: harvestId1, status: 'ACTIVE' },
      new UniqueEntityID('schedule-1'),
    )
    const plannedSchedule = makeSchedule(
      { harvestId: harvestId2, status: 'PLANNED' },
      new UniqueEntityID('schedule-2'),
    )

    schedulesRepository.items.push(activeSchedule, plannedSchedule)

    QueryBus.register('FindHarvestWithCropQuery', async () => ({
      id: 'harvest-1',
      name: 'Safra 2026',
      fieldId: 'field-1',
      cropTypeName: 'Soja',
      varietyName: 'Intacta',
      startDate: new Date('2026-04-06T00:00:00Z'),
      expectedEndDate: new Date('2026-04-20T00:00:00Z'),
    }))

    QueryBus.register('FindFieldQuery', async () => ({
      id: 'field-1',
      name: 'Talhao A1',
    }))

    QueryBus.register('FindFieldTicketsByScheduleIdQuery', async () => [
      {
        id: 'ticket-1',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-06T00:00:00Z'),
        operationType: 'SPRAYING',
        status: 'COMPLETED',
        inputs: [],
      },
      {
        id: 'ticket-2',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-07T00:00:00Z'),
        operationType: 'SPRAYING',
        status: 'COMPLETED',
        inputs: [],
      },
      {
        id: 'ticket-3',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-08T00:00:00Z'),
        operationType: 'FERTIGATION',
        status: 'DRAFT',
        inputs: [],
      },
    ])

    const result = await sut.execute()

    expect(result.isRight()).toBe(true)
    if (!result.isRight()) return

    const { data } = result.value
    expect(data).toHaveLength(1)

    const item = data[0]
    expect(item.scheduleId).toBe('schedule-1')
    expect(item.field.name).toBe('Talhao A1')
    expect(item.harvest.cropTypeName).toBe('Soja')
    expect(item.harvest.varietyName).toBe('Intacta')
    expect(item.operations).toHaveLength(2)
    expect(item.operations[0]).toEqual(
      expect.objectContaining({
        type: 'SPRAYING',
        label: 'P1',
        startDate: new Date('2026-04-06T00:00:00Z'),
        endDate: new Date('2026-04-07T00:00:00Z'),
        completed: true,
      }),
    )
    expect(item.operations[1]).toEqual(
      expect.objectContaining({
        type: 'FERTIGATION',
        label: 'F1',
        startDate: new Date('2026-04-08T00:00:00Z'),
        endDate: new Date('2026-04-08T00:00:00Z'),
        completed: false,
      }),
    )
  })

  it('should return empty data when no ACTIVE schedules exist', async () => {
    const result = await sut.execute()

    expect(result.isRight()).toBe(true)
    if (!result.isRight()) return
    expect(result.value.data).toHaveLength(0)
  })

  it('should group consecutive same-type tickets into one operation', async () => {
    const harvestId = new UniqueEntityID('harvest-1')
    const schedule = makeSchedule(
      { harvestId, status: 'ACTIVE' },
      new UniqueEntityID('schedule-1'),
    )
    schedulesRepository.items.push(schedule)

    QueryBus.register('FindHarvestWithCropQuery', async () => ({
      id: 'harvest-1',
      name: 'Safra 2026',
      fieldId: 'field-1',
      cropTypeName: 'Soja',
      varietyName: 'Intacta',
      startDate: new Date('2026-04-01T00:00:00Z'),
      expectedEndDate: new Date('2026-04-15T00:00:00Z'),
    }))

    QueryBus.register('FindFieldQuery', async () => ({
      id: 'field-1',
      name: 'Talhao B2',
    }))

    QueryBus.register('FindFieldTicketsByScheduleIdQuery', async () => [
      {
        id: 't1',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-01T00:00:00Z'),
        operationType: 'SPRAYING',
        status: 'COMPLETED',
        inputs: [],
      },
      {
        id: 't2',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-02T00:00:00Z'),
        operationType: 'SPRAYING',
        status: 'COMPLETED',
        inputs: [],
      },
      // gap day 3
      {
        id: 't3',
        scheduleId: 'schedule-1',
        fieldId: 'field-1',
        harvestId: 'harvest-1',
        date: new Date('2026-04-04T00:00:00Z'),
        operationType: 'SPRAYING',
        status: 'DRAFT',
        inputs: [],
      },
    ])

    const result = await sut.execute()
    expect(result.isRight()).toBe(true)
    if (!result.isRight()) return

    const ops = result.value.data[0].operations
    // D1-D2 consecutive SPRAYING = P1, D4 SPRAYING after gap = P2
    expect(ops).toHaveLength(2)
    expect(ops[0].label).toBe('P1')
    expect(ops[0].endDate).toEqual(new Date('2026-04-02T00:00:00Z'))
    expect(ops[0].completed).toBe(true)
    expect(ops[1].label).toBe('P2')
    expect(ops[1].startDate).toEqual(new Date('2026-04-04T00:00:00Z'))
    expect(ops[1].completed).toBe(false)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && contextzip pnpm vitest run src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts`
Expected: FAIL — `get-dashboard-schedules` module not found

---

## Task 4: GetDashboardSchedules Use Case — Implementation (Backend)

**Files:**
- Create: `backend/src/domain/schedule/application/use-cases/get-dashboard-schedules.ts`

- [ ] **Step 1: Implement the use case**

```typescript
// backend/src/domain/schedule/application/use-cases/get-dashboard-schedules.ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import { QueryBus } from '@/core/query-bus/query-bus'
import { SchedulesRepository } from '../repositories/schedules-repository'
import { FindHarvestWithCropQuery } from '@/domain/crop/harvests/application/queries/find-harvest-with-crop.query'
import type { FindHarvestWithCropQueryResult } from '@/domain/crop/harvests/application/queries/find-harvest-with-crop.query'
import { FindFieldQuery } from '@/domain/field/application/queries/find-field.query'
import type { FindFieldQueryResult } from '@/domain/field/application/queries/find-field.query'
import { FindFieldTicketsByScheduleIdQuery } from '@/domain/field-ticket/application/queries/find-field-tickets-by-schedule-id.query'
import type { FindFieldTicketsByScheduleIdQueryResult } from '@/domain/field-ticket/application/queries/find-field-tickets-by-schedule-id.query'

export type DashboardOperation = {
  type: string
  label: string
  startDate: Date
  endDate: Date
  completed: boolean
}

export type DashboardScheduleItem = {
  scheduleId: string
  status: string
  field: { id: string; name: string }
  harvest: {
    id: string
    name: string
    cropTypeName: string
    varietyName: string
    startDate: Date
    expectedEndDate: Date
  }
  operations: DashboardOperation[]
}

type GetDashboardSchedulesResponse = Either<
  null,
  { data: DashboardScheduleItem[] }
>

@Injectable()
export class GetDashboardSchedulesUseCase {
  constructor(private schedulesRepository: SchedulesRepository) {}

  async execute(): Promise<GetDashboardSchedulesResponse> {
    const schedules = await this.schedulesRepository.findMany({
      status: 'ACTIVE',
      page: 1,
      perPage: 100,
    })

    const data = await Promise.all(
      schedules.map(async (schedule) => {
        const harvestData =
          await QueryBus.execute<FindHarvestWithCropQueryResult>(
            new FindHarvestWithCropQuery(schedule.harvestId.toString()),
          )

        const fieldData = harvestData
          ? await QueryBus.execute<FindFieldQueryResult>(
              new FindFieldQuery(harvestData.fieldId),
            )
          : null

        const tickets =
          await QueryBus.execute<FindFieldTicketsByScheduleIdQueryResult>(
            new FindFieldTicketsByScheduleIdQuery(schedule.id.toString()),
          )

        const operations = this.groupTicketsIntoOperations(tickets ?? [])

        return {
          scheduleId: schedule.id.toString(),
          status: schedule.status,
          field: {
            id: fieldData?.id ?? '',
            name: fieldData?.name ?? '',
          },
          harvest: {
            id: harvestData?.id ?? '',
            name: harvestData?.name ?? '',
            cropTypeName: harvestData?.cropTypeName ?? '',
            varietyName: harvestData?.varietyName ?? '',
            startDate: harvestData?.startDate ?? new Date(),
            expectedEndDate: harvestData?.expectedEndDate ?? new Date(),
          },
          operations,
        }
      }),
    )

    return right({ data })
  }

  private groupTicketsIntoOperations(
    tickets: NonNullable<FindFieldTicketsByScheduleIdQueryResult>,
  ): DashboardOperation[] {
    if (tickets.length === 0) return []

    const sorted = [...tickets]
      .filter((t) => t.status !== 'CANCELLED')
      .sort((a, b) => new Date(a.date).getTime() - new Date(b.date).getTime())

    const operations: DashboardOperation[] = []
    const sprayingCount = { current: 0 }
    const fertigationCount = { current: 0 }

    let i = 0
    while (i < sorted.length) {
      const current = sorted[i]
      const startDate = new Date(current.date)
      let endDate = new Date(current.date)
      let allCompleted = current.status === 'COMPLETED'

      // Group consecutive days with same operation type
      let j = i + 1
      while (j < sorted.length) {
        const next = sorted[j]
        const nextDate = new Date(next.date)
        const prevDate = new Date(sorted[j - 1].date)
        const diffDays =
          (nextDate.getTime() - prevDate.getTime()) / (1000 * 60 * 60 * 24)

        if (next.operationType === current.operationType && diffDays === 1) {
          endDate = nextDate
          if (next.status !== 'COMPLETED') allCompleted = false
          j++
        } else {
          break
        }
      }

      const counter =
        current.operationType === 'SPRAYING' ? sprayingCount : fertigationCount
      counter.current++
      const prefix = current.operationType === 'SPRAYING' ? 'P' : 'F'

      operations.push({
        type: current.operationType,
        label: `${prefix}${counter.current}`,
        startDate,
        endDate,
        completed: allCompleted,
      })

      i = j
    }

    return operations
  }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `cd backend && contextzip pnpm vitest run src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts`
Expected: All 3 tests PASS

- [ ] **Step 3: Commit**

```bash
git add backend/src/domain/schedule/application/use-cases/get-dashboard-schedules.ts \
       backend/src/domain/schedule/application/use-cases/__tests__/get-dashboard-schedules.spec.ts
git commit -m "feat(schedule): add GetDashboardSchedules use case with operation grouping"
```

---

## Task 5: DashboardSchedulePresenter (Backend)

**Files:**
- Create: `backend/src/infra/http/presenters/schedule/dashboard-schedule.presenter.ts`

- [ ] **Step 1: Create the presenter**

```typescript
// backend/src/infra/http/presenters/schedule/dashboard-schedule.presenter.ts
import type { DashboardScheduleItem } from '@/domain/schedule/application/use-cases/get-dashboard-schedules'

export class DashboardSchedulePresenter {
  static toHTTP(item: DashboardScheduleItem) {
    return {
      scheduleId: item.scheduleId,
      status: item.status,
      field: {
        id: item.field.id,
        name: item.field.name,
      },
      harvest: {
        id: item.harvest.id,
        name: item.harvest.name,
        cropTypeName: item.harvest.cropTypeName,
        varietyName: item.harvest.varietyName,
        startDate: item.harvest.startDate,
        expectedEndDate: item.harvest.expectedEndDate,
      },
      operations: item.operations.map((op) => ({
        type: op.type,
        label: op.label,
        startDate: op.startDate,
        endDate: op.endDate,
        completed: op.completed,
      })),
    }
  }
}
```

- [ ] **Step 2: Verify compilation**

Run: `cd backend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add backend/src/infra/http/presenters/schedule/dashboard-schedule.presenter.ts
git commit -m "feat(schedule): add DashboardSchedulePresenter"
```

---

## Task 6: GetDashboardSchedulesController + E2E Test (Backend)

**Files:**
- Create: `backend/src/infra/http/controllers/schedule/get-dashboard-schedules.controller.ts`
- Create: `backend/src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts`
- Modify: `backend/src/infra/http/controllers/schedule/schedule-http.module.ts`

- [ ] **Step 1: Write the E2E test**

```typescript
// backend/src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts
import { describe, it, expect, beforeAll } from 'vitest'
import request from 'supertest'
import { type INestApplication } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/infra/app.module'
import { makeHarvestFactory } from 'test/factories/make-harvest'
import { makeScheduleFactory } from 'test/factories/make-schedule'
import { makeFieldFactory } from 'test/factories/make-field'
import { makeFieldTicketFactory } from 'test/factories/make-field-ticket'
import { makeCropTypeFactory } from 'test/factories/make-crop-type'
import { makeVarietyFactory } from 'test/factories/make-variety'
import { TokenService } from '@/infra/auth/token.service'
import { PrismaService } from '@/infra/database/prisma/prisma.service'

describe('GetDashboardSchedulesController (E2E)', () => {
  let app: INestApplication
  let tokenService: TokenService
  let prisma: PrismaService

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = moduleRef.createNestApplication()
    tokenService = moduleRef.get(TokenService)
    prisma = moduleRef.get(PrismaService)

    await app.init()
  })

  it('[GET] /v1/schedules/dashboard — returns ACTIVE schedules with hydrated data', async () => {
    // Setup: create field, cropType, variety, harvest, schedule, tickets
    const user = await prisma.user.findFirst()
    const token = await tokenService.generateAccessToken({
      sub: user!.id,
      role: user!.role,
    })

    const cropType = await makeCropTypeFactory(prisma, { name: 'Soja' })
    const variety = await makeVarietyFactory(prisma, {
      name: 'Intacta',
      cropTypeId: cropType.id,
    })
    const field = await makeFieldFactory(prisma, { name: 'Talhao A1' })
    const harvest = await makeHarvestFactory(prisma, {
      name: 'Safra 2026',
      cropTypeId: cropType.id,
      varietyId: variety.id,
      fieldId: field.id,
      startDate: new Date('2026-04-06T00:00:00Z'),
      expectedEndDate: new Date('2026-04-20T00:00:00Z'),
    })
    const schedule = await makeScheduleFactory(prisma, {
      harvestId: harvest.id,
      status: 'ACTIVE',
    })
    await makeFieldTicketFactory(prisma, {
      scheduleId: schedule.id,
      fieldId: field.id,
      harvestId: harvest.id,
      date: new Date('2026-04-06T00:00:00Z'),
      operationType: 'SPRAYING',
      status: 'COMPLETED',
    })
    await makeFieldTicketFactory(prisma, {
      scheduleId: schedule.id,
      fieldId: field.id,
      harvestId: harvest.id,
      date: new Date('2026-04-07T00:00:00Z'),
      operationType: 'FERTIGATION',
      status: 'DRAFT',
    })

    const response = await request(app.getHttpServer())
      .get('/v1/schedules/dashboard')
      .set('Authorization', `Bearer ${token}`)

    expect(response.status).toBe(200)
    expect(response.body.data).toHaveLength(1)

    const item = response.body.data[0]
    expect(item.field.name).toBe('Talhao A1')
    expect(item.harvest.cropTypeName).toBe('Soja')
    expect(item.harvest.varietyName).toBe('Intacta')
    expect(item.operations).toHaveLength(2)
    expect(item.operations[0].type).toBe('SPRAYING')
    expect(item.operations[0].label).toBe('P1')
    expect(item.operations[0].completed).toBe(true)
    expect(item.operations[1].type).toBe('FERTIGATION')
    expect(item.operations[1].label).toBe('F1')
    expect(item.operations[1].completed).toBe(false)
  })
})
```

> **Note:** The E2E test factory functions (`makeHarvestFactory`, `makeScheduleFactory`, etc.) follow the existing pattern where they create records directly via PrismaService. Verify exact factory signatures match the project's test factories before running. Adjust imports if factories use a different export pattern (e.g. `makeSchedule` with overrides object).

- [ ] **Step 2: Run E2E test to verify it fails**

Run: `cd backend && contextzip pnpm vitest run src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts`
Expected: FAIL — controller not found / 404

- [ ] **Step 3: Create the controller**

```typescript
// backend/src/infra/http/controllers/schedule/get-dashboard-schedules.controller.ts
import { Controller, Get, UseFilters, UseGuards } from '@nestjs/common'
import {
  ApiBearerAuth,
  ApiForbiddenResponse,
  ApiInternalServerErrorResponse,
  ApiOkResponse,
  ApiOperation,
  ApiTags,
  ApiUnauthorizedResponse,
} from '@nestjs/swagger'
import { GetDashboardSchedulesUseCase } from '@/domain/schedule/application/use-cases/get-dashboard-schedules'
import { ScheduleErrorFilter } from '../../filters/schedule-error.filter'
import { ServiceTag } from '@/infra/decorators/service-tag.decorator'
import { CaslAbilityGuard } from '@/infra/auth/casl/casl-ability.guard'
import { CheckPolicies } from '@/infra/auth/casl/check-policies.decorator'
import { DashboardSchedulePresenter } from '../../presenters/schedule/dashboard-schedule.presenter'
import { ForbiddenDto, InternalServerErrorDto } from '../../dtos/error/generic'

@UseFilters(ScheduleErrorFilter)
@ApiTags('Schedules')
@ApiBearerAuth()
@ServiceTag('schedule')
@Controller({ path: 'schedules', version: '1' })
export class GetDashboardSchedulesController {
  constructor(
    private getDashboardSchedules: GetDashboardSchedulesUseCase,
  ) {}

  @Get('dashboard')
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('list', 'Schedule'))
  @ApiOperation({ summary: 'Get dashboard schedule overview' })
  @ApiOkResponse({ description: 'Dashboard schedules returned' })
  @ApiUnauthorizedResponse({ description: 'Unauthorized' })
  @ApiForbiddenResponse({ type: ForbiddenDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle() {
    const result = await this.getDashboardSchedules.execute()

    if (result.isLeft()) {
      throw result.value
    }

    return {
      data: result.value.data.map(DashboardSchedulePresenter.toHTTP),
    }
  }
}
```

- [ ] **Step 4: Register in schedule-http.module.ts**

Add `GetDashboardSchedulesController` to the `controllers` array and `GetDashboardSchedulesUseCase` to the `providers` array in `backend/src/infra/http/controllers/schedule/schedule-http.module.ts`.

- [ ] **Step 5: Run E2E test to verify it passes**

Run: `cd backend && contextzip pnpm vitest run src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add backend/src/infra/http/controllers/schedule/get-dashboard-schedules.controller.ts \
       backend/src/infra/http/controllers/schedule/__tests__/get-dashboard-schedules.controller.e2e-spec.ts \
       backend/src/infra/http/controllers/schedule/schedule-http.module.ts
git commit -m "feat(schedule): add GET /v1/schedules/dashboard endpoint"
```

---

## Task 7: Frontend Schema + API (Frontend)

**Files:**
- Create: `frontend/src/domains/schedule/schemas/dashboard-schedule.schema.ts`
- Create: `frontend/src/domains/schedule/api/get-dashboard-schedules.ts`

- [ ] **Step 1: Create the Zod schema**

```typescript
// frontend/src/domains/schedule/schemas/dashboard-schedule.schema.ts
import { z } from 'zod'

const dashboardOperationSchema = z.object({
  type: z.enum(['SPRAYING', 'FERTIGATION']),
  label: z.string(),
  startDate: z.coerce.date(),
  endDate: z.coerce.date(),
  completed: z.boolean(),
})

const dashboardScheduleItemSchema = z.object({
  scheduleId: z.string().uuid(),
  status: z.string(),
  field: z.object({
    id: z.string().uuid(),
    name: z.string(),
  }),
  harvest: z.object({
    id: z.string().uuid(),
    name: z.string(),
    cropTypeName: z.string(),
    varietyName: z.string(),
    startDate: z.coerce.date(),
    expectedEndDate: z.coerce.date(),
  }),
  operations: z.array(dashboardOperationSchema),
})

export const dashboardSchedulesResponseSchema = z.object({
  data: z.array(dashboardScheduleItemSchema),
})

export type DashboardSchedulesResponse = z.infer<
  typeof dashboardSchedulesResponseSchema
>
export type DashboardScheduleItem = z.infer<typeof dashboardScheduleItemSchema>
export type DashboardOperation = z.infer<typeof dashboardOperationSchema>
```

- [ ] **Step 2: Create the API function**

```typescript
// frontend/src/domains/schedule/api/get-dashboard-schedules.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/handle-http-error'
import { dashboardSchedulesResponseSchema } from '../schemas/dashboard-schedule.schema'

export async function getDashboardSchedules() {
  try {
    const response = await api.get('v1/schedules/dashboard').json()
    return dashboardSchedulesResponseSchema.parse(response)
  } catch (error) {
    throw handleHttpError(error)
  }
}
```

- [ ] **Step 3: Verify compilation**

Run: `cd frontend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/domains/schedule/schemas/dashboard-schedule.schema.ts \
       frontend/src/domains/schedule/api/get-dashboard-schedules.ts
git commit -m "feat(schedule): add dashboard schedules schema and API function"
```

---

## Task 8: MSW Handler (Frontend)

**Files:**
- Create: `frontend/tests/mocks/handlers/dashboard-schedule.handler.ts`
- Modify: `frontend/tests/mocks/handlers.ts`

- [ ] **Step 1: Create the MSW handler**

```typescript
// frontend/tests/mocks/handlers/dashboard-schedule.handler.ts
import { http, HttpResponse } from 'msw'

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:4333'

const dashboardSchedules = [
  {
    scheduleId: 'schedule-1',
    status: 'ACTIVE',
    field: { id: 'field-1', name: 'Talhao A1' },
    harvest: {
      id: 'harvest-1',
      name: 'Safra 2026',
      cropTypeName: 'Soja',
      varietyName: 'Intacta',
      startDate: '2026-04-06T00:00:00.000Z',
      expectedEndDate: '2026-04-20T00:00:00.000Z',
    },
    operations: [
      {
        type: 'SPRAYING',
        label: 'P1',
        startDate: '2026-04-06T00:00:00.000Z',
        endDate: '2026-04-07T00:00:00.000Z',
        completed: true,
      },
      {
        type: 'FERTIGATION',
        label: 'F1',
        startDate: '2026-04-08T00:00:00.000Z',
        endDate: '2026-04-08T00:00:00.000Z',
        completed: true,
      },
      {
        type: 'SPRAYING',
        label: 'P2',
        startDate: '2026-04-10T00:00:00.000Z',
        endDate: '2026-04-11T00:00:00.000Z',
        completed: false,
      },
    ],
  },
  {
    scheduleId: 'schedule-2',
    status: 'ACTIVE',
    field: { id: 'field-2', name: 'Talhao B2' },
    harvest: {
      id: 'harvest-2',
      name: 'Safra 2026 B',
      cropTypeName: 'Milho',
      varietyName: 'Safrinha',
      startDate: '2026-04-04T00:00:00.000Z',
      expectedEndDate: '2026-04-25T00:00:00.000Z',
    },
    operations: [
      {
        type: 'SPRAYING',
        label: 'P1',
        startDate: '2026-04-04T00:00:00.000Z',
        endDate: '2026-04-06T00:00:00.000Z',
        completed: true,
      },
      {
        type: 'FERTIGATION',
        label: 'F1',
        startDate: '2026-04-08T00:00:00.000Z',
        endDate: '2026-04-09T00:00:00.000Z',
        completed: false,
      },
    ],
  },
]

export const dashboardScheduleHandlers = [
  http.get(`${API_URL}/v1/schedules/dashboard`, () => {
    return HttpResponse.json({ data: dashboardSchedules })
  }),
]
```

- [ ] **Step 2: Register in handlers.ts**

Add import and spread `dashboardScheduleHandlers` into the `handlers` array in `frontend/tests/mocks/handlers.ts`.

- [ ] **Step 3: Commit**

```bash
git add frontend/tests/mocks/handlers/dashboard-schedule.handler.ts \
       frontend/tests/mocks/handlers.ts
git commit -m "feat(schedule): add MSW handler for dashboard schedules endpoint"
```

---

## Task 9: GanttHeader Component (Frontend)

**Files:**
- Create: `frontend/src/domains/schedule/components/dashboard/gantt-header.tsx`

- [ ] **Step 1: Create the component**

This component renders the date axis. It receives the global date range and renders date labels at regular intervals, plus a "today" vertical marker.

```tsx
// frontend/src/domains/schedule/components/dashboard/gantt-header.tsx
'use client'

type GanttHeaderProps = {
  rangeStart: Date
  rangeEnd: Date
}

export function GanttHeader({ rangeStart, rangeEnd }: GanttHeaderProps) {
  const totalMs = rangeEnd.getTime() - rangeStart.getTime()
  const totalDays = Math.ceil(totalMs / (1000 * 60 * 60 * 24)) + 1

  const now = new Date()
  const todayUtc = Date.UTC(
    now.getUTCFullYear(),
    now.getUTCMonth(),
    now.getUTCDate(),
  )
  const todayOffset = todayUtc - rangeStart.getTime()
  const todayPercent = (todayOffset / totalMs) * 100

  // Generate date labels at regular intervals
  const labelCount = Math.min(totalDays, 10)
  const step = Math.max(1, Math.floor(totalDays / labelCount))
  const labels: { date: Date; percent: number }[] = []

  for (let i = 0; i < totalDays; i += step) {
    const date = new Date(rangeStart.getTime() + i * 24 * 60 * 60 * 1000)
    const percent = (i / (totalDays - 1)) * 100
    labels.push({ date, percent })
  }

  const formatDate = (d: Date) => {
    const day = String(d.getUTCDate()).padStart(2, '0')
    const month = String(d.getUTCMonth() + 1).padStart(2, '0')
    return `${day}/${month}`
  }

  return (
    <div className="relative grid grid-cols-[180px_1fr] border-b border-border">
      <div className="px-3 py-2 text-xs font-medium text-muted-foreground uppercase tracking-wide">
        Talhao / Safra
      </div>
      <div className="relative py-2">
        {labels.map(({ date, percent }) => (
          <span
            key={date.toISOString()}
            className="absolute text-[10px] text-muted-foreground -translate-x-1/2"
            style={{ left: `${percent}%` }}
          >
            {formatDate(date)}
          </span>
        ))}
        {todayPercent >= 0 && todayPercent <= 100 && (
          <div
            className="absolute top-0 bottom-0 w-0.5 bg-emerald-500 z-10"
            style={{ left: `${todayPercent}%` }}
            title="Hoje"
          />
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Verify compilation**

Run: `cd frontend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/domains/schedule/components/dashboard/gantt-header.tsx
git commit -m "feat(schedule): add GanttHeader component with date axis and today marker"
```

---

## Task 10: GanttRow Component (Frontend)

**Files:**
- Create: `frontend/src/domains/schedule/components/dashboard/gantt-row.tsx`

- [ ] **Step 1: Create the component**

```tsx
// frontend/src/domains/schedule/components/dashboard/gantt-row.tsx
'use client'

import type { DashboardScheduleItem } from '../../schemas/dashboard-schedule.schema'

type GanttRowProps = {
  item: DashboardScheduleItem
  rangeStart: Date
  rangeEnd: Date
  isExpanded: boolean
  onToggle: () => void
}

export function GanttRow({
  item,
  rangeStart,
  rangeEnd,
  isExpanded,
  onToggle,
}: GanttRowProps) {
  const totalMs = rangeEnd.getTime() - rangeStart.getTime()

  const now = new Date()
  const todayUtc = Date.UTC(
    now.getUTCFullYear(),
    now.getUTCMonth(),
    now.getUTCDate(),
  )
  const todayPercent = ((todayUtc - rangeStart.getTime()) / totalMs) * 100

  return (
    <div
      className="grid grid-cols-[180px_1fr] border-b border-border cursor-pointer hover:bg-muted/30 transition-colors"
      onClick={onToggle}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          e.preventDefault()
          onToggle()
        }
      }}
      aria-expanded={isExpanded}
    >
      <div className="px-3 py-2 border-r border-border">
        <div className="text-sm font-semibold">{item.field.name}</div>
        <div className="text-xs text-muted-foreground">
          {item.harvest.cropTypeName} {item.harvest.varietyName}
        </div>
      </div>
      <div className="relative h-8 my-auto">
        {todayPercent >= 0 && todayPercent <= 100 && (
          <div
            className="absolute top-0 bottom-0 w-0.5 bg-emerald-500/40 z-10"
            style={{ left: `${todayPercent}%` }}
          />
        )}
        {item.operations.map((op) => {
          const opStart = op.startDate.getTime() - rangeStart.getTime()
          const opEnd = op.endDate.getTime() - rangeStart.getTime()
          const oneDayMs = 24 * 60 * 60 * 1000
          const left = (opStart / totalMs) * 100
          const width = ((opEnd - opStart + oneDayMs) / totalMs) * 100

          const colorClass = op.completed
            ? op.type === 'SPRAYING'
              ? 'bg-emerald-700 text-emerald-200'
              : 'bg-blue-600 text-blue-200'
            : 'bg-muted text-muted-foreground'

          return (
            <div
              key={op.label}
              className={`absolute top-1 h-6 rounded flex items-center justify-center text-[10px] font-medium ${colorClass}`}
              style={{ left: `${left}%`, width: `${Math.max(width, 2)}%` }}
              title={`${op.label} — ${op.type === 'SPRAYING' ? 'Pulverizacao' : 'Fertirrigacao'} (${op.completed ? 'concluido' : 'pendente'})`}
            >
              {width > 3 ? op.label : ''}
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Verify compilation**

Run: `cd frontend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/domains/schedule/components/dashboard/gantt-row.tsx
git commit -m "feat(schedule): add GanttRow component with proportional operation blocks"
```

---

## Task 11: GanttRowDetail Component (Frontend)

**Files:**
- Create: `frontend/src/domains/schedule/components/dashboard/gantt-row-detail.tsx`

- [ ] **Step 1: Create the component**

```tsx
// frontend/src/domains/schedule/components/dashboard/gantt-row-detail.tsx
'use client'

import Link from 'next/link'
import type { DashboardScheduleItem } from '../../schemas/dashboard-schedule.schema'

type GanttRowDetailProps = {
  item: DashboardScheduleItem
}

export function GanttRowDetail({ item }: GanttRowDetailProps) {
  const formatDate = (d: Date) => {
    const day = String(d.getUTCDate()).padStart(2, '0')
    const month = String(d.getUTCMonth() + 1).padStart(2, '0')
    return `${day}/${month}`
  }

  return (
    <div className="border-b border-border bg-muted/20 px-4 py-3">
      <div className="grid grid-cols-[180px_1fr]">
        <div className="text-xs text-muted-foreground">
          <div>
            Safra: {item.harvest.name}
          </div>
          <div>
            Inicio: {formatDate(item.harvest.startDate)} — Fim previsto:{' '}
            {formatDate(item.harvest.expectedEndDate)}
          </div>
          <Link
            href={`/schedules/${item.scheduleId}`}
            className="text-primary hover:underline mt-1 inline-block"
          >
            Ver cronograma
          </Link>
        </div>
        <div className="flex flex-wrap gap-2">
          {item.operations.map((op) => (
            <div
              key={op.label}
              className="flex items-center gap-2 rounded border border-border px-2 py-1 text-xs"
            >
              <span
                className={`h-2 w-2 rounded-full ${
                  op.completed
                    ? op.type === 'SPRAYING'
                      ? 'bg-emerald-500'
                      : 'bg-blue-500'
                    : 'bg-muted-foreground'
                }`}
              />
              <span className="font-medium">{op.label}</span>
              <span className="text-muted-foreground">
                {formatDate(op.startDate)}
                {op.startDate.getTime() !== op.endDate.getTime() &&
                  ` — ${formatDate(op.endDate)}`}
              </span>
              <span
                className={
                  op.completed ? 'text-emerald-500' : 'text-muted-foreground'
                }
              >
                {op.completed ? 'Concluido' : 'Pendente'}
              </span>
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Verify compilation**

Run: `cd frontend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/domains/schedule/components/dashboard/gantt-row-detail.tsx
git commit -m "feat(schedule): add GanttRowDetail component for expanded view"
```

---

## Task 12: ScheduleGanttDashboard Container + Barrel Export (Frontend)

**Files:**
- Create: `frontend/src/domains/schedule/components/dashboard/schedule-gantt-dashboard.tsx`
- Create: `frontend/src/domains/schedule/components/dashboard/index.ts`

- [ ] **Step 1: Create the container component**

```tsx
// frontend/src/domains/schedule/components/dashboard/schedule-gantt-dashboard.tsx
'use client'

import { useState } from 'react'
import type { DashboardSchedulesResponse } from '../../schemas/dashboard-schedule.schema'
import { GanttHeader } from './gantt-header'
import { GanttRow } from './gantt-row'
import { GanttRowDetail } from './gantt-row-detail'

type ScheduleGanttDashboardProps = {
  data: DashboardSchedulesResponse['data']
}

export function ScheduleGanttDashboard({
  data,
}: ScheduleGanttDashboardProps) {
  const [expandedId, setExpandedId] = useState<string | null>(null)

  if (data.length === 0) {
    return (
      <div className="flex items-center justify-center py-12 text-sm text-muted-foreground">
        Nenhum cronograma ativo.
      </div>
    )
  }

  // Compute global date range across all schedules
  let rangeStart = Infinity
  let rangeEnd = -Infinity

  for (const item of data) {
    const start = item.harvest.startDate.getTime()
    const end = item.harvest.expectedEndDate.getTime()
    if (start < rangeStart) rangeStart = start
    if (end > rangeEnd) rangeEnd = end
  }

  const rangeStartDate = new Date(rangeStart)
  const rangeEndDate = new Date(rangeEnd)

  return (
    <div className="rounded-lg border border-border overflow-hidden">
      <GanttHeader rangeStart={rangeStartDate} rangeEnd={rangeEndDate} />
      {data.map((item) => (
        <div key={item.scheduleId}>
          <GanttRow
            item={item}
            rangeStart={rangeStartDate}
            rangeEnd={rangeEndDate}
            isExpanded={expandedId === item.scheduleId}
            onToggle={() =>
              setExpandedId((prev) =>
                prev === item.scheduleId ? null : item.scheduleId,
              )
            }
          />
          {expandedId === item.scheduleId && (
            <GanttRowDetail item={item} />
          )}
        </div>
      ))}
    </div>
  )
}
```

- [ ] **Step 2: Create barrel export**

```typescript
// frontend/src/domains/schedule/components/dashboard/index.ts
export { ScheduleGanttDashboard } from './schedule-gantt-dashboard'
```

- [ ] **Step 3: Verify compilation**

Run: `cd frontend && contextzip pnpm exec tsc --noEmit`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/domains/schedule/components/dashboard/
git commit -m "feat(schedule): add ScheduleGanttDashboard container component"
```

---

## Task 13: Dashboard Page Integration (Frontend)

**Files:**
- Modify: `frontend/src/app/(private)/dashboard/page.tsx`

- [ ] **Step 1: Update the dashboard page**

Replace the placeholder with the Gantt component. The page is a Server Component — it fetches data server-side and passes it to the client component.

```tsx
// frontend/src/app/(private)/dashboard/page.tsx
import { getDashboardSchedules } from '@/domains/schedule/api/get-dashboard-schedules'
import { ScheduleGanttDashboard } from '@/domains/schedule/components/dashboard'

export default async function DashboardPage() {
  const { data } = await getDashboardSchedules()

  return (
    <div
      className="container flex h-full flex-col gap-6 p-10"
      data-testid="dashboard-page"
    >
      <div>
        <h1 className="text-3xl font-bold tracking-tight">Dashboard</h1>
        <p className="mt-1 text-sm text-muted-foreground">
          Cronogramas Ativos
        </p>
      </div>
      <ScheduleGanttDashboard data={data} />
    </div>
  )
}
```

- [ ] **Step 2: Start dev server and verify visually**

Run: `cd frontend && contextzip pnpm dev`

Open `http://localhost:4000/dashboard` in the browser. Verify:
- Gantt renders with mock data (MSW)
- Date axis shows labels and "today" marker
- Operation blocks are positioned proportionally
- Rows are clickable and expand/collapse
- Colors: green = completed spraying, blue = completed fertigation, gray = pending

- [ ] **Step 3: Commit**

```bash
git add frontend/src/app/(private)/dashboard/page.tsx
git commit -m "feat(dashboard): integrate ScheduleGanttDashboard on dashboard page"
```

---

## Task 14: Playwright E2E Test (Frontend)

**Files:**
- Create: `frontend/tests/e2e/dashboard/dashboard-gantt.e2e-spec.ts`

- [ ] **Step 1: Create the E2E test**

```typescript
// frontend/tests/e2e/dashboard/dashboard-gantt.e2e-spec.ts
import { test, expect } from '@playwright/test'

test.describe('Dashboard Schedule Gantt', () => {
  test('should display active schedules as Gantt rows', async ({ page }) => {
    await page.goto('/dashboard')

    // Wait for dashboard to load
    await expect(page.getByTestId('dashboard-page')).toBeVisible()

    // Should show schedule gantt with field names from MSW mock
    await expect(page.getByText('Talhao A1')).toBeVisible()
    await expect(page.getByText('Talhao B2')).toBeVisible()

    // Should show crop info
    await expect(page.getByText('Soja Intacta')).toBeVisible()
    await expect(page.getByText('Milho Safrinha')).toBeVisible()
  })

  test('should expand row on click and show details', async ({ page }) => {
    await page.goto('/dashboard')

    await expect(page.getByTestId('dashboard-page')).toBeVisible()

    // Click on first row to expand
    await page.getByText('Talhao A1').click()

    // Should show expanded details
    await expect(page.getByText('Ver cronograma')).toBeVisible()

    // Click again to collapse
    await page.getByText('Talhao A1').click()

    // Details should be hidden
    await expect(page.getByText('Ver cronograma')).not.toBeVisible()
  })
})
```

- [ ] **Step 2: Run E2E test**

Run: `cd frontend && contextzip pnpm test:e2e tests/e2e/dashboard/dashboard-gantt.e2e-spec.ts`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add frontend/tests/e2e/dashboard/dashboard-gantt.e2e-spec.ts
git commit -m "test(dashboard): add Playwright E2E tests for schedule Gantt"
```
