# Schedule Automatic Status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace manual schedule status management (Activate/Complete buttons) with automatic status transitions driven by FieldTicket lifecycle, adding a review checkpoint before completion.

**Architecture:** Schedule status machine becomes: PLANNED → ACTIVE → UNDER_REVIEW → COMPLETED, with CANCELLED from PLANNED/ACTIVE. Auto-transitions triggered by domain event subscribers listening to FieldTicket events. New `ReviewScheduleUseCase` replaces manual `CompleteScheduleUseCase`. `CreateScheduleUseCase` checks field for existing ACTIVE schedule to determine initial status. `CancelScheduleUseCase` enforces PRINTED ticket resolution before allowing cancellation.

**Tech Stack:** NestJS, Prisma, Zod, Next.js App Router, Zustand, react-hook-form, shadcn/ui

**ADR:** `docs/architecture.md` — "2026-03-20 — Automatic schedule status with review checkpoint"
**Rules:** `docs/rules/schedule.md` (updated)

---

## File Map

### Files to DELETE

| File | Reason |
|------|--------|
| `backend/src/domain/schedule/application/use-cases/activate-schedule.ts` | Replaced by automatic activation |
| `backend/src/domain/schedule/application/use-cases/complete-schedule.ts` | Replaced by ReviewScheduleUseCase |
| `backend/src/domain/schedule/application/use-cases/__tests__/activate-schedule.spec.ts` | Use case deleted |
| `backend/src/domain/schedule/application/use-cases/__tests__/complete-schedule.spec.ts` | Use case deleted |
| `backend/src/domain/schedule/application/use-cases/errors/schedule-not-planned-error.ts` | Only used by activate |
| `backend/src/domain/schedule/application/use-cases/errors/schedule-not-active-error.ts` | Only used by complete |
| `backend/src/infra/http/controllers/schedule/activate-schedule.controller.ts` | Endpoint removed |
| `backend/src/infra/http/controllers/schedule/complete-schedule.controller.ts` | Endpoint removed |
| `frontend/src/domains/schedule/api/activate-schedule.ts` | Endpoint removed |
| `frontend/src/domains/schedule/api/complete-schedule.ts` | Endpoint removed |
| `frontend/src/domains/schedule/actions/activate-schedule.ts` | Endpoint removed |
| `frontend/src/domains/schedule/actions/complete-schedule.ts` | Endpoint removed |

### Files to CREATE

| File | Responsibility |
|------|---------------|
| `backend/src/domain/schedule/enterprise/events/schedule-under-review-event.ts` | Domain event for ACTIVE → UNDER_REVIEW |
| `backend/src/domain/schedule/enterprise/events/schedule-reviewed-event.ts` | Domain event for UNDER_REVIEW → COMPLETED |
| `backend/src/domain/schedule/application/use-cases/review-schedule.ts` | Confirm review + promote next schedule |
| `backend/src/domain/schedule/application/use-cases/__tests__/review-schedule.spec.ts` | Unit tests |
| `backend/src/domain/schedule/application/use-cases/errors/schedule-not-under-review-error.ts` | Error for review |
| `backend/src/domain/schedule/application/use-cases/errors/unresolved-printed-tickets-error.ts` | Error for cancel with PRINTED |
| `backend/src/domain/schedule/application/subscribers/on-field-ticket-terminal-check-schedule.ts` | Auto ACTIVE → UNDER_REVIEW |
| `backend/src/domain/schedule/application/subscribers/on-field-ticket-created-revert-review.ts` | Auto UNDER_REVIEW → ACTIVE |
| `backend/src/domain/schedule/application/subscribers/__tests__/on-field-ticket-terminal-check-schedule.spec.ts` | Subscriber unit tests |
| `backend/src/domain/schedule/application/subscribers/__tests__/on-field-ticket-created-revert-review.spec.ts` | Subscriber unit tests |
| `backend/src/domain/schedule/application/queries/find-active-schedule-by-field-id.query.ts` | Query contract |
| `backend/src/domain/schedule/application/handlers/find-active-schedule-by-field-id.handler.ts` | Query handler |
| `backend/src/domain/field-ticket/application/subscribers/on-schedule-cancelled-auto-cancel-tickets.ts` | Auto-cancel DRAFT/REVIEWED tickets on schedule cancel |
| `backend/src/infra/http/controllers/schedule/review-schedule.controller.ts` | PATCH /v1/schedules/:id/review |
| `backend/src/infra/http/controllers/schedule/__tests__/review-schedule.controller.e2e-spec.ts` | E2E controller test |
| `backend/src/domain/audit-log/application/subscribers/schedule/on-schedule-under-review.ts` | Audit log |
| `backend/src/domain/audit-log/application/subscribers/schedule/on-schedule-reviewed.ts` | Audit log |
| `backend/src/domain/crop/harvests/application/subscribers/on-schedule-completed.ts` | Harvest COMPLETED on review confirm |
| `frontend/src/domains/schedule/schemas/review-schedule.schema.ts` | Form + response schemas |
| `frontend/src/domains/schedule/api/review-schedule.ts` | API function |
| `frontend/src/domains/schedule/actions/review-schedule.ts` | Server action |
| `frontend/src/domains/schedule/components/dialogs/review-schedule-dialog.tsx` | Review UI |
| `frontend/src/domains/schedule/components/dialogs/resolve-printed-tickets-dialog.tsx` | PRINTED resolution UI |
| `frontend/src/domains/schedule/components/dialogs/select-next-schedule-dialog.tsx` | Next schedule picker |

### Files to MODIFY

| File | Change |
|------|--------|
| `backend/prisma/schema.prisma` | Add `UNDER_REVIEW` to ScheduleStatus enum, add `reviewNotes`/`reviewedAt` to Schedule model |
| `backend/src/domain/schedule/enterprise/entities/schedule.ts` | Add UNDER_REVIEW to type, new methods: `setUnderReview()`, `confirmReview()`, update `cancel()` guard |
| `backend/src/domain/schedule/application/use-cases/create-schedule.ts` | Check field for ACTIVE schedule, set initial status accordingly |
| `backend/src/domain/schedule/application/use-cases/__tests__/create-schedule.spec.ts` | Add tests for ACTIVE/PLANNED creation logic |
| `backend/src/domain/schedule/application/use-cases/cancel-schedule.ts` | Enforce PRINTED resolution, auto-cancel DRAFT/REVIEWED tickets |
| `backend/src/domain/schedule/application/use-cases/__tests__/cancel-schedule.spec.ts` | Add tests for PRINTED guard, auto-cancel |
| `backend/src/domain/schedule/application/repositories/schedules-repository.ts` | Add `findActiveByFieldId(fieldId)` method |
| `backend/src/infra/database/prisma/repositories/schedule/prisma-schedules.repository.ts` | Implement `findActiveByFieldId` |
| `backend/src/infra/database/prisma/mappers/schedule/prisma-schedule.mapper.ts` | Map `reviewNotes`/`reviewedAt` |
| `backend/test/repositories/schedule/in-memory-schedules-repository.ts` | Implement `findActiveByFieldId` |
| `backend/src/infra/http/controllers/schedule/schedule-http.module.ts` | Remove activate/complete, add review controller + use case |
| `backend/src/infra/http/filters/schedule-error.filter.ts` | Remove old errors, add new ones |
| `backend/src/infra/events/schedule/schedule-events.module.ts` | Remove old subscribers, add new ones |
| `backend/src/domain/audit-log/application/subscribers/schedule/on-schedule-activated.ts` | Update `from` to handle both PLANNED→ACTIVE |
| `backend/src/domain/crop/harvests/application/subscribers/on-schedule-activated.ts` | Keep — still reacts to ScheduleActivatedEvent |
| `backend/src/domain/crop/harvests/enterprise/entities/harvest.ts` | Verify/add `cancel()` method (ACTIVE → CANCELLED) |
| `backend/src/domain/crop/harvests/application/queries/find-harvest.query.ts` | Verify `fieldId` in query result (needed by CreateSchedule) |
| `backend/src/domain/audit-log/application/subscribers/schedule/on-schedule-completed.ts` | Update to listen to ScheduleReviewedEvent instead of ScheduleCompletedEvent |
| `frontend/src/domains/schedule/schemas/schedule.schema.ts` | Add UNDER_REVIEW to enum |
| `frontend/src/domains/schedule/components/ui/schedule-status-badge.tsx` | Add UNDER_REVIEW label + variant |
| `frontend/src/domains/schedule/components/ui/schedule-actions-popover.tsx` | Remove Ativar/Concluir, add Revisar, update guards |
| `frontend/src/domains/schedule/components/ui/schedule-detail-header.tsx` | Update canEdit logic |
| `frontend/src/domains/schedule/api/index.ts` | Remove activate/complete, add review |
| `frontend/src/domains/schedule/actions/index.ts` | Remove activate/complete, add review |
| `frontend/src/app/(private)/schedules/page.tsx` | Add UNDER_REVIEW tab |
| `frontend/src/domains/schedule/store/schedule-dialogs.store.ts` | Add state for 3 new dialogs (review, resolve-printed, select-next) |
| `frontend/src/app/(private)/schedules/[id]/page.tsx` | Render review/resolve/next-schedule dialogs |
| `frontend/tests/mocks/handlers/schedule.handler.ts` | Add PATCH /review handler, remove /activate /complete, add UNDER_REVIEW |
| `frontend/tests/schedule/schedule-status.e2e-spec.ts` | Rewrite for review flow (remove activate/complete tests) |
| `docs/api-reference.md` | Remove activate/complete endpoints, add review endpoint |
| `docs/flows.md` | Add review flow, update cancel flow |

---

## Task 1: Prisma Schema Migration

**Files:**
- Modify: `backend/prisma/schema.prisma`

- [ ] **Step 1: Add UNDER_REVIEW to ScheduleStatus enum and new fields to Schedule model**

In `backend/prisma/schema.prisma`, find the `ScheduleStatus` enum and `Schedule` model:

```prisma
enum ScheduleStatus {
  PLANNED
  ACTIVE
  UNDER_REVIEW
  COMPLETED
  CANCELLED
}

model Schedule {
  // ... existing fields ...
  reviewNotes String?   @map("review_notes")
  reviewedAt  DateTime? @map("reviewed_at")
  // ... existing relations ...
}
```

- [ ] **Step 2: Generate Prisma client and run migration**

```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev --name add-schedule-under-review-status
```

- [ ] **Step 3: Verify migration applied**

```bash
cd backend && pnpm prisma migrate status
```

Expected: All migrations applied, no pending.

---

## Task 2: Schedule Entity — New Status and Methods

**Files:**
- Modify: `backend/src/domain/schedule/enterprise/entities/schedule.ts`
- Create: `backend/src/domain/schedule/enterprise/events/schedule-under-review-event.ts`
- Create: `backend/src/domain/schedule/enterprise/events/schedule-reviewed-event.ts`

- [ ] **Step 1: Create ScheduleUnderReviewEvent**

```typescript
// backend/src/domain/schedule/enterprise/events/schedule-under-review-event.ts
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import { DomainEvent } from '@/core/events/domain-event'
import type { Schedule } from '../entities/schedule'

export class ScheduleUnderReviewEvent implements DomainEvent {
  occurredAt: Date
  schedule: Schedule
  actorId: string

  constructor(schedule: Schedule, actorId: string) {
    this.schedule = schedule
    this.actorId = actorId
    this.occurredAt = new Date()
  }

  getAggregateId(): UniqueEntityID {
    return this.schedule.id
  }
}
```

- [ ] **Step 2: Create ScheduleReviewedEvent**

```typescript
// backend/src/domain/schedule/enterprise/events/schedule-reviewed-event.ts
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import { DomainEvent } from '@/core/events/domain-event'
import type { Schedule } from '../entities/schedule'

export class ScheduleReviewedEvent implements DomainEvent {
  occurredAt: Date
  schedule: Schedule
  actorId: string
  previousData: { status: string }

  constructor(schedule: Schedule, actorId: string) {
    this.schedule = schedule
    this.actorId = actorId
    this.previousData = { status: 'UNDER_REVIEW' }
    this.occurredAt = new Date()
  }

  getAggregateId(): UniqueEntityID {
    return this.schedule.id
  }
}
```

- [ ] **Step 3: Update Schedule entity**

Add `'UNDER_REVIEW'` to the status type union. Add `reviewNotes` and `reviewedAt` to props. Add new methods:

- `setUnderReview(actorId)`: guards `status === 'ACTIVE'`, sets status to `UNDER_REVIEW`, fires `ScheduleUnderReviewEvent`
- `confirmReview(reviewNotes: string | undefined, actorId: string)`: guards `status === 'UNDER_REVIEW'`, sets status to `COMPLETED`, sets `reviewNotes`, `reviewedAt`, fires BOTH `ScheduleReviewedEvent` AND `ScheduleCompletedEvent` (audit + harvest subscribers listen to ScheduleCompletedEvent)
- Update `update()` guard: block UNDER_REVIEW alongside COMPLETED/CANCELLED
- Update `cancel()` guard: allow PLANNED and ACTIVE only (UNDER_REVIEW not cancellable per rules)
- Update `activate()` guard: allow BOTH `PLANNED` and `UNDER_REVIEW` (needed for promoting PLANNED → ACTIVE AND for reverting UNDER_REVIEW → ACTIVE when new ticket added)

- [ ] **Step 4: Run existing tests to verify no regressions**

```bash
cd backend && pnpm test -- --testPathPattern="schedule" --no-coverage
```

Expected: All existing schedule tests still pass.

- [ ] **Step 5: Commit**

```bash
git add backend/prisma backend/src/domain/schedule/enterprise
git commit -m "feat(schedule): add UNDER_REVIEW status, reviewNotes/reviewedAt fields, new domain events"
```

---

## Task 3: Repository — Add findActiveByFieldId

**Files:**
- Modify: `backend/src/domain/schedule/application/repositories/schedules-repository.ts`
- Modify: `backend/src/infra/database/prisma/repositories/schedule/prisma-schedules.repository.ts`
- Modify: `backend/src/infra/database/prisma/mappers/schedule/prisma-schedule.mapper.ts`
- Modify: `backend/test/repositories/schedule/in-memory-schedules-repository.ts`

- [ ] **Step 1: Add method to repository interface**

In `schedules-repository.ts`, add:

```typescript
abstract findActiveByFieldId(fieldId: string): Promise<Schedule | null>
```

- [ ] **Step 2: Update mapper for new fields**

In `prisma-schedule.mapper.ts`, add `reviewNotes` and `reviewedAt` to both `toDomain` and `toPrisma`/`toPrismaUpdate` methods.

- [ ] **Step 3: Implement in Prisma repository**

In `prisma-schedules.repository.ts`, implement `findActiveByFieldId`:

```typescript
async findActiveByFieldId(fieldId: string): Promise<Schedule | null> {
  const schedule = await this.prisma.schedule.findFirst({
    where: {
      status: 'ACTIVE',
      deletedAt: null,
      harvest: { fieldId },
    },
  })

  if (!schedule) return null

  return PrismaScheduleMapper.toDomain(schedule)
}
```

- [ ] **Step 4: Implement in InMemory repository**

In `in-memory-schedules-repository.ts`, implement `findActiveByFieldId`. This requires access to harvests to resolve `fieldId` — inject `InMemoryHarvestsRepository` or use a simpler approach with a stored mapping.

Note: Read the existing InMemory implementation to understand how cross-domain data is resolved in tests. The InMemory repo may need a `harvestsRepository` dependency or the test can use `makeSchedule` with a known harvestId linked to a known fieldId.

- [ ] **Step 5: Run tests**

```bash
cd backend && pnpm test -- --testPathPattern="schedule" --no-coverage
```

- [ ] **Step 6: Commit**

```bash
git add backend/src/domain/schedule/application/repositories backend/src/infra/database/prisma backend/test/repositories
git commit -m "feat(schedule): add findActiveByFieldId to repository and mapper support for reviewNotes/reviewedAt"
```

---

## Task 4: QueryBus — FindActiveScheduleByFieldId

**Files:**
- Create: `backend/src/domain/schedule/application/queries/find-active-schedule-by-field-id.query.ts`
- Create: `backend/src/domain/schedule/application/handlers/find-active-schedule-by-field-id.handler.ts`

- [ ] **Step 1: Create query contract**

```typescript
// find-active-schedule-by-field-id.query.ts
export class FindActiveScheduleByFieldIdQuery {
  static queryName = 'FindActiveScheduleByFieldIdQuery' as const

  constructor(public readonly fieldId: string) {}
}

export type FindActiveScheduleByFieldIdQueryResult = {
  id: string
  harvestId: string
  status: string
} | null
```

- [ ] **Step 2: Create handler**

```typescript
// find-active-schedule-by-field-id.handler.ts
import { Injectable } from '@nestjs/common'
import { QueryHandler } from '@/core/query-bus/query-handler'
import { SchedulesRepository } from '../repositories/schedules-repository'
import {
  FindActiveScheduleByFieldIdQuery,
  type FindActiveScheduleByFieldIdQueryResult,
} from '../queries/find-active-schedule-by-field-id.query'

@Injectable()
export class FindActiveScheduleByFieldIdHandler
  implements QueryHandler<FindActiveScheduleByFieldIdQuery, FindActiveScheduleByFieldIdQueryResult>
{
  queryName = FindActiveScheduleByFieldIdQuery.queryName

  constructor(private schedulesRepository: SchedulesRepository) {}

  async handle(query: FindActiveScheduleByFieldIdQuery): Promise<FindActiveScheduleByFieldIdQueryResult> {
    const schedule = await this.schedulesRepository.findActiveByFieldId(query.fieldId)

    if (!schedule) return null

    return {
      id: schedule.id.toString(),
      harvestId: schedule.harvestId.toString(),
      status: schedule.status,
    }
  }
}
```

- [ ] **Step 3: Register handler in QueryBusModule**

Add `FindActiveScheduleByFieldIdHandler` to `backend/src/infra/query-bus/query-bus.module.ts`.

- [ ] **Step 4: Commit**

```bash
git add backend/src/domain/schedule/application/queries backend/src/domain/schedule/application/handlers backend/src/infra/query-bus
git commit -m "feat(schedule): add FindActiveScheduleByFieldIdQuery for cross-domain field check"
```

---

## Task 5: Modify CreateScheduleUseCase — Auto-determine Initial Status

**Files:**
- Modify: `backend/src/domain/schedule/application/use-cases/create-schedule.ts`
- Modify: `backend/src/domain/schedule/application/use-cases/__tests__/create-schedule.spec.ts`

- [ ] **Step 0: Verify FindHarvestQueryResult includes fieldId**

Read `backend/src/domain/crop/harvests/application/queries/find-harvest.query.ts` and confirm `FindHarvestQueryResult` includes `fieldId`. If it doesn't, add `fieldId: string` to the result type and update the `FindHarvestHandler` to return it. This is required for the CreateScheduleUseCase to determine which field the harvest belongs to.

- [ ] **Step 1: Write failing tests**

Add 2 new test cases to `create-schedule.spec.ts`:

1. "should create schedule as ACTIVE when field has no active schedule" — mock `FindActiveScheduleByFieldIdQuery` to return `null`, verify schedule status is ACTIVE
2. "should create schedule as PLANNED when field already has an active schedule" — mock `FindActiveScheduleByFieldIdQuery` to return an existing schedule, verify status is PLANNED

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd backend && pnpm test -- --testPathPattern="create-schedule" --no-coverage
```

Expected: 2 new tests FAIL.

- [ ] **Step 3: Implement changes in CreateScheduleUseCase**

After validating harvest exists and is UNSCHEDULED:
1. Get harvest's `fieldId` from the `FindHarvestQuery` result (already fetched)
2. Query `FindActiveScheduleByFieldIdQuery` with `fieldId`
3. If no active schedule → call `Schedule.create(props, actorId)` then `schedule.activate(actorId)` → born ACTIVE
4. If active schedule exists → call `Schedule.create(props, actorId)` → born PLANNED (default)

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd backend && pnpm test -- --testPathPattern="create-schedule" --no-coverage
```

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/schedule/application/use-cases/create-schedule.ts backend/src/domain/schedule/application/use-cases/__tests__/create-schedule.spec.ts
git commit -m "feat(schedule): auto-determine initial status based on field's active schedule"
```

---

## Task 6: ReviewScheduleUseCase — New Use Case

**Files:**
- Create: `backend/src/domain/schedule/application/use-cases/review-schedule.ts`
- Create: `backend/src/domain/schedule/application/use-cases/__tests__/review-schedule.spec.ts`
- Create: `backend/src/domain/schedule/application/use-cases/errors/schedule-not-under-review-error.ts`

- [ ] **Step 1: Create error class**

```typescript
// schedule-not-under-review-error.ts
export class ScheduleNotUnderReviewError extends Error {
  constructor() {
    super('Schedule is not under review.')
  }
}
```

- [ ] **Step 2: Write failing tests**

Test cases for `review-schedule.spec.ts`:

1. "should confirm review and complete schedule" — UNDER_REVIEW schedule → COMPLETED, reviewNotes set, reviewedAt set
2. "should promote next schedule when nextScheduleId provided" — verify next schedule transitions to ACTIVE
3. "should fail if schedule is not UNDER_REVIEW" — ACTIVE schedule → ScheduleNotUnderReviewError
4. "should fail if schedule not found" — ScheduleNotFoundError
5. "should fail if nextScheduleId is not PLANNED" — error
6. "should complete without promoting if no nextScheduleId" — just COMPLETED

- [ ] **Step 3: Run tests to verify they fail**

```bash
cd backend && pnpm test -- --testPathPattern="review-schedule" --no-coverage
```

- [ ] **Step 4: Implement ReviewScheduleUseCase**

Input: `{ scheduleId, reviewNotes?, nextScheduleId?, actorId }`

Logic:
1. Find schedule, verify UNDER_REVIEW status
2. Call `schedule.confirmReview(reviewNotes, actorId)` → fires ScheduleReviewedEvent
3. Save schedule
4. If `nextScheduleId`:
   a. Find next schedule, verify PLANNED status
   b. Call `nextSchedule.activate(actorId)` → fires ScheduleActivatedEvent
   c. Save next schedule
5. Return `{ data: schedule }`

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd backend && pnpm test -- --testPathPattern="review-schedule" --no-coverage
```

- [ ] **Step 6: Commit**

```bash
git add backend/src/domain/schedule/application/use-cases/review-schedule.ts backend/src/domain/schedule/application/use-cases/__tests__/review-schedule.spec.ts backend/src/domain/schedule/application/use-cases/errors/schedule-not-under-review-error.ts
git commit -m "feat(schedule): add ReviewScheduleUseCase with next-schedule promotion"
```

---

## Task 7: Modify CancelScheduleUseCase — PRINTED Resolution + Auto-cancel

**Files:**
- Modify: `backend/src/domain/schedule/application/use-cases/cancel-schedule.ts`
- Modify: `backend/src/domain/schedule/application/use-cases/__tests__/cancel-schedule.spec.ts`
- Create: `backend/src/domain/schedule/application/use-cases/errors/unresolved-printed-tickets-error.ts`

- [ ] **Step 1: Create error class**

```typescript
// unresolved-printed-tickets-error.ts
export class UnresolvedPrintedTicketsError extends Error {
  public ticketIds: string[]

  constructor(ticketIds: string[]) {
    super('Resolve all printed tickets before cancelling.')
    this.ticketIds = ticketIds
  }
}
```

- [ ] **Step 2: Write failing tests**

New test cases:
1. "should fail if schedule has PRINTED tickets" — returns UnresolvedPrintedTicketsError with ticket IDs
2. "should auto-cancel DRAFT and REVIEWED tickets" — verify tickets cancelled via QueryBus
3. "should preserve COMPLETED tickets" — verify COMPLETED tickets untouched
4. "should promote next schedule when nextScheduleId provided" — like ReviewSchedule

- [ ] **Step 3: Run tests to verify they fail**

```bash
cd backend && pnpm test -- --testPathPattern="cancel-schedule" --no-coverage
```

- [ ] **Step 4: Implement changes**

Add `reason: string` to the request type (mandatory per rules). Add `nextScheduleId?: string` to the request type.

Before cancelling:
1. Query `FindFieldTicketsByScheduleIdQuery` to get all tickets
2. Filter PRINTED tickets — if any exist, return Left(UnresolvedPrintedTicketsError) with their IDs
3. Call `schedule.cancel(reason, actorId)` — the entity's `cancel()` method must accept a `reason` parameter
4. Save schedule — this fires `ScheduleCancelledEvent`
5. Auto-cancellation of DRAFT/REVIEWED tickets is handled by a **FieldTicket domain subscriber** (`OnScheduleCancelledAutoCancelTickets`) listening to `ScheduleCancelledEvent` — this preserves domain boundaries (Schedule does not directly cancel FieldTickets)
6. If `nextScheduleId` → promote as in ReviewSchedule

**Important:** Create the FieldTicket subscriber `on-schedule-cancelled-auto-cancel-tickets.ts` in `backend/src/domain/field-ticket/application/subscribers/`. It queries tickets by scheduleId, filters DRAFT/REVIEWED, and cancels each with reason "Cronograma cancelado". Register it in the FieldTicket events module.

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd backend && pnpm test -- --testPathPattern="cancel-schedule" --no-coverage
```

- [ ] **Step 6: Commit**

```bash
git add backend/src/domain/schedule/application/use-cases/cancel-schedule.ts backend/src/domain/schedule/application/use-cases/__tests__/cancel-schedule.spec.ts backend/src/domain/schedule/application/use-cases/errors
git commit -m "feat(schedule): enforce PRINTED resolution and auto-cancel DRAFT/REVIEWED on cancel"
```

---

## Task 8: Event Subscribers — Auto-transitions

**Files:**
- Create: `backend/src/domain/schedule/application/subscribers/on-field-ticket-terminal-check-schedule.ts`
- Create: `backend/src/domain/schedule/application/subscribers/on-field-ticket-created-revert-review.ts`

- [ ] **Step 0a: Write failing tests for OnFieldTicketTerminalCheckSchedule**

Create `backend/src/domain/schedule/application/subscribers/__tests__/on-field-ticket-terminal-check-schedule.spec.ts`. Test cases:
1. "should transition ACTIVE schedule to UNDER_REVIEW when all tickets are terminal"
2. "should NOT transition if schedule is not ACTIVE"
3. "should NOT transition if some tickets are still non-terminal"

Run tests, verify they fail.

- [ ] **Step 0b: Write failing tests for OnFieldTicketCreatedRevertReview**

Create `backend/src/domain/schedule/application/subscribers/__tests__/on-field-ticket-created-revert-review.spec.ts`. Test cases:
1. "should revert UNDER_REVIEW schedule to ACTIVE when new ticket created"
2. "should NOT revert if schedule is not UNDER_REVIEW"

Run tests, verify they fail.

- [ ] **Step 1: Write OnFieldTicketTerminalCheckSchedule subscriber**

Listens to: `FieldTicketCompletedEvent`, `FieldTicketCancelledEvent`

Logic:
1. Get `scheduleId` from event's FieldTicket
2. Find schedule — if not ACTIVE, return (only ACTIVE can transition)
3. Query all FieldTickets for this schedule via `FindFieldTicketsByScheduleIdQuery`
4. Check if ALL are terminal (COMPLETED or CANCELLED)
5. If yes → `schedule.setUnderReview(event.actorId)` → save

- [ ] **Step 2: Write OnFieldTicketCreatedRevertReview subscriber**

Listens to: `FieldTicketCreatedEvent`

Logic:
1. Get `scheduleId` from event's FieldTicket
2. Find schedule — if not UNDER_REVIEW, return
3. Call `schedule.activate(event.actorId)` → save

- [ ] **Step 3: Register subscribers in ScheduleEventsModule**

Add both subscribers to `backend/src/infra/events/schedule/schedule-events.module.ts` providers.

- [ ] **Step 4: Run all schedule tests**

```bash
cd backend && pnpm test -- --testPathPattern="schedule" --no-coverage
```

- [ ] **Step 5: Commit**

```bash
git add backend/src/domain/schedule/application/subscribers backend/src/infra/events/schedule
git commit -m "feat(schedule): add auto-transition subscribers for UNDER_REVIEW and revert-to-ACTIVE"
```

---

## Task 9: Delete Old Use Cases and Controllers

**Files:**
- Delete: `backend/src/domain/schedule/application/use-cases/activate-schedule.ts`
- Delete: `backend/src/domain/schedule/application/use-cases/complete-schedule.ts`
- Delete: `backend/src/domain/schedule/application/use-cases/__tests__/activate-schedule.spec.ts`
- Delete: `backend/src/domain/schedule/application/use-cases/__tests__/complete-schedule.spec.ts`
- Delete: `backend/src/domain/schedule/application/use-cases/errors/schedule-not-planned-error.ts`
- Delete: `backend/src/domain/schedule/application/use-cases/errors/schedule-not-active-error.ts`
- Delete: `backend/src/infra/http/controllers/schedule/activate-schedule.controller.ts`
- Delete: `backend/src/infra/http/controllers/schedule/complete-schedule.controller.ts`

- [ ] **Step 1: Delete all files listed above**

- [ ] **Step 2: Update schedule-http.module.ts**

Remove imports and registrations for `ActivateScheduleController`, `ActivateScheduleUseCase`, `CompleteScheduleController`, `CompleteScheduleUseCase`.

- [ ] **Step 3: Update schedule-error.filter.ts**

Remove `ScheduleNotPlannedError` and `ScheduleNotActiveError` imports and switch cases. Add `ScheduleNotUnderReviewError` (422) and `UnresolvedPrintedTicketsError` (422).

- [ ] **Step 4: Update schedule-events.module.ts**

Remove imports for old audit-log subscribers (`OnScheduleActivated` for audit if no longer needed — actually keep it, the event still fires from auto-activation). Update references to match the new event flow. Add the new audit subscribers for `ScheduleUnderReviewEvent` and `ScheduleReviewedEvent`.

- [ ] **Step 5: Run all backend tests**

```bash
cd backend && pnpm test --no-coverage
```

Expected: All tests pass, no broken imports.

- [ ] **Step 6: Commit**

```bash
git add -A backend/src
git commit -m "refactor(schedule): remove manual activate/complete use cases and controllers"
```

---

## Task 10: Review Schedule Controller

**Files:**
- Create: `backend/src/infra/http/controllers/schedule/review-schedule.controller.ts`
- Modify: `backend/src/infra/http/controllers/schedule/schedule-http.module.ts`

- [ ] **Step 1: Write E2E test (TDD)**

Create `backend/src/infra/http/controllers/schedule/__tests__/review-schedule.controller.e2e-spec.ts`. Test cases:
1. "should review and complete an UNDER_REVIEW schedule" — 200 OK
2. "should promote next schedule when nextScheduleId provided" — 200 OK, verify next schedule is ACTIVE
3. "should return 422 if schedule is not UNDER_REVIEW"
4. "should return 401 if not authenticated"

Run tests, verify they fail.

- [ ] **Step 2: Create controller**

```
PATCH /v1/schedules/:id/review
Body: { reviewNotes?: string, nextScheduleId?: string }
Auth: can('update', 'Schedule')
```

Follow `docs/coding-patterns/backend/controller.md`. Use `ZodValidationPipe` for body validation. Map errors via `ScheduleErrorFilter`.

- [ ] **Step 3: Register in module**

Add `ReviewScheduleController` and `ReviewScheduleUseCase` to `schedule-http.module.ts`.

- [ ] **Step 4: Run E2E tests and verify they pass**

```bash
cd backend && pnpm test:e2e -- --testPathPattern="review-schedule" --no-coverage
```

- [ ] **Step 5: Commit**

```bash
git add backend/src/infra/http/controllers/schedule
git commit -m "feat(schedule): add PATCH /v1/schedules/:id/review endpoint"
```

---

## Task 11: Harvest Subscriber Updates

**Files:**
- Modify: `backend/src/domain/crop/harvests/application/subscribers/on-schedule-cancelled.ts`
- Create: `backend/src/domain/crop/harvests/application/subscribers/on-schedule-completed.ts`

- [ ] **Step 1: Keep on-schedule-activated.ts as-is**

The `ScheduleActivatedEvent` still fires (from `CreateScheduleUseCase` when born ACTIVE, from `ReviewScheduleUseCase` when promoting next). The subscriber `OnScheduleActivatedUpdateHarvest` still calls `harvest.activate()`. No changes needed.

- [ ] **Step 2: Verify/add harvest.cancel() method**

Read `backend/src/domain/crop/harvests/enterprise/entities/harvest.ts`. Check if a `cancel()` method exists that transitions ACTIVE → CANCELLED. If it doesn't exist, add it following the existing pattern of `complete()`. This method should fire `HarvestCancelledEvent`.

- [ ] **Step 3: Update on-schedule-cancelled.ts**

Change from `harvest.revertToUnscheduled()` to `harvest.cancel()` — per the new rules, cancelled schedule → Harvest CANCELLED (not UNSCHEDULED).

- [ ] **Step 4: Create on-schedule-completed subscriber**

Listens to `ScheduleCompletedEvent` (fired by `confirmReview()` alongside `ScheduleReviewedEvent`).

Logic: Find harvest → `harvest.complete(actorId)` → save.

Note: Using `ScheduleCompletedEvent` (not `ScheduleReviewedEvent`) keeps the harvest subscriber decoupled from the review concept — it only cares that the schedule is COMPLETED.

- [ ] **Step 5: Update audit-log on-schedule-completed subscriber**

Update `backend/src/domain/audit-log/application/subscribers/schedule/on-schedule-completed.ts` to use correct `previousData` — change `from: 'ACTIVE'` to `from: 'UNDER_REVIEW'` since the transition is now UNDER_REVIEW → COMPLETED.

- [ ] **Step 6: Register new subscriber in CropEventsModule**

- [ ] **Step 7: Run all tests**

```bash
cd backend && pnpm test --no-coverage
```

- [ ] **Step 8: Commit**

```bash
git add backend/src/domain/crop backend/src/domain/audit-log
git commit -m "feat(crop): update harvest subscribers for automatic schedule lifecycle"
```

---

## Task 12: Frontend — Schema and API Updates

**Files:**
- Modify: `frontend/src/domains/schedule/schemas/schedule.schema.ts`
- Create: `frontend/src/domains/schedule/schemas/review-schedule.schema.ts`
- Modify: `frontend/src/domains/schedule/schemas/index.ts`
- Delete: `frontend/src/domains/schedule/api/activate-schedule.ts`
- Delete: `frontend/src/domains/schedule/api/complete-schedule.ts`
- Create: `frontend/src/domains/schedule/api/review-schedule.ts`
- Modify: `frontend/src/domains/schedule/api/index.ts`
- Delete: `frontend/src/domains/schedule/actions/activate-schedule.ts`
- Delete: `frontend/src/domains/schedule/actions/complete-schedule.ts`
- Create: `frontend/src/domains/schedule/actions/review-schedule.ts`
- Modify: `frontend/src/domains/schedule/actions/index.ts`

- [ ] **Step 1: Update schedule schema — add UNDER_REVIEW**

- [ ] **Step 2: Create review-schedule schema**

```typescript
// review-schedule.schema.ts
import { z } from 'zod'

export const reviewScheduleRequestSchema = z.object({
  reviewNotes: z.string().optional(),
  nextScheduleId: z.string().uuid().optional(),
})

export type ReviewScheduleRequest = z.infer<typeof reviewScheduleRequestSchema>
```

- [ ] **Step 3: Delete activate/complete API and action files, create review API and action**

Follow patterns from `docs/coding-patterns/frontend/api.md` and `docs/coding-patterns/frontend/action.md`. The API calls `PATCH v1/schedules/:id/review`. The action validates, calls API, invalidates `schedules` cache tag.

- [ ] **Step 4: Update barrel exports** (api/index.ts, actions/index.ts, schemas/index.ts)

- [ ] **Step 5: Verify frontend builds**

```bash
cd frontend && pnpm build
```

- [ ] **Step 6: Commit**

```bash
git add frontend/src/domains/schedule
git commit -m "feat(schedule): update frontend schemas, API, and actions for auto-status"
```

---

## Task 13: Frontend — UI Components

**Files:**
- Modify: `frontend/src/domains/schedule/components/ui/schedule-status-badge.tsx`
- Modify: `frontend/src/domains/schedule/components/ui/schedule-actions-popover.tsx`
- Modify: `frontend/src/domains/schedule/components/ui/schedule-detail-header.tsx`
- Modify: `frontend/src/app/(private)/schedules/page.tsx`
- Create: `frontend/src/domains/schedule/components/dialogs/review-schedule-dialog.tsx`
- Create: `frontend/src/domains/schedule/components/dialogs/resolve-printed-tickets-dialog.tsx`
- Create: `frontend/src/domains/schedule/components/dialogs/select-next-schedule-dialog.tsx`
- Modify: `frontend/src/app/(private)/schedules/[id]/page.tsx`

- [ ] **Step 1: Update schedule-status-badge.tsx**

Add `UNDER_REVIEW` → label `'Em Revisão'`, variant `'draft'` (or a new appropriate variant).

- [ ] **Step 2: Update schedule-actions-popover.tsx**

Remove "Ativar" and "Concluir" buttons and their handlers. Add "Revisar" button visible only when status is `UNDER_REVIEW`. Update cancel button guard to show for PLANNED and ACTIVE.

- [ ] **Step 3: Update schedule-detail-header.tsx**

Update `canEdit` to: `schedule.status === 'PLANNED' || schedule.status === 'ACTIVE'`. UNDER_REVIEW, COMPLETED, CANCELLED cannot edit.

- [ ] **Step 4: Update schedules list page — add UNDER_REVIEW tab**

Add new tab "Em Revisão" with filter `status=UNDER_REVIEW` between "Ativo" and "Concluído".

- [ ] **Step 5: Create review-schedule-dialog.tsx**

Dialog shown when user clicks "Revisar" on an UNDER_REVIEW schedule. Contains:
- Summary of completed/cancelled tickets count
- `reviewNotes` textarea (optional)
- "Confirmar Revisão" button
- On confirm → calls `reviewSchedule` action

Follow `docs/coding-patterns/frontend/component.md` for dialog pattern. All elements must have `data-testid`.

- [ ] **Step 6: Create resolve-printed-tickets-dialog.tsx**

Dialog shown when cancel is attempted and PRINTED tickets exist. Lists each PRINTED ticket with 2 options:
- "Finalizar" (was executed)
- "Cancelar" (was not executed)

Only allows proceeding when all PRINTED tickets are resolved. Each resolution calls the respective FieldTicket action (finalize or cancel).

- [ ] **Step 7: Create select-next-schedule-dialog.tsx**

Dialog shown after review confirmation or after cancellation. Lists PLANNED schedules for the same field. Options:
- Select one to activate (radio/click)
- "Criar novo cronograma" button (navigates to create schedule with field pre-selected)
- "Fechar" button (just dismiss)

On selection → passes `nextScheduleId` to the review/cancel action.

- [ ] **Step 8: Update useScheduleDialogsStore**

Add state and setters for the 3 new dialogs in `frontend/src/domains/schedule/store/schedule-dialogs.store.ts`:
- `reviewDialogOpen: string | null` (scheduleId)
- `resolvePrintedDialogOpen: string | null` (scheduleId)
- `selectNextScheduleDialogOpen: string | null` (scheduleId)

Plus their setters.

- [ ] **Step 9: Wire dialogs in schedule detail page**

Render the 3 new dialogs in `schedules/[id]/page.tsx`. Wire to `useScheduleDialogsStore`.

- [ ] **Step 10: Update Playwright MSW handler**

In `frontend/tests/mocks/handlers/schedule.handler.ts`:
- Add `UNDER_REVIEW` to `ScheduleStatus` type
- Add `reviewNotes`/`reviewedAt` to the Schedule interface
- Add `PATCH /v1/schedules/:id/review` handler
- Remove `/activate` and `/complete` handlers

- [ ] **Step 11: Update Playwright E2E spec**

In `frontend/tests/schedule/schedule-status.e2e-spec.ts`:
- Remove "activate a planned schedule" and "complete an active schedule" test cases
- Add "review an under-review schedule" test case
- Add "UNDER_REVIEW tab shows in schedule list" test case

- [ ] **Step 12: Verify frontend builds**

```bash
cd frontend && pnpm build
```

- [ ] **Step 13: Run Playwright tests**

```bash
cd frontend && pnpm test:e2e -- --grep "schedule"
```

- [ ] **Step 14: Commit**

```bash
git add frontend/src frontend/tests
git commit -m "feat(schedule): add review dialog, PRINTED resolution, next-schedule picker, remove Ativar/Concluir"
```

---

## Task 14: Documentation Updates

**Files:**
- Modify: `docs/api-reference.md`
- Modify: `docs/flows.md`

- [ ] **Step 1: Update api-reference.md**

Remove:
- `PATCH /v1/schedules/:id/activate`
- `PATCH /v1/schedules/:id/complete` (if they exist — check first)

Add:
- `PATCH /v1/schedules/:id/review` — Review and complete a schedule (UNDER_REVIEW → COMPLETED, optional nextScheduleId to promote)

- [ ] **Step 2: Update flows.md**

Add new flows:
- "Review completed schedule" — trigger: system auto-transitions to UNDER_REVIEW, user reviews, confirms, picks next
- "Cancel schedule with PRINTED resolution" — trigger: user cancels, system requires resolving PRINTED tickets first

- [ ] **Step 3: Commit**

```bash
git add docs/
git commit -m "docs: update api-reference and flows for automatic schedule status"
```

---

## Task 15: Final Validation

- [ ] **Step 1: Run all backend tests**

```bash
cd backend && pnpm test --no-coverage
```

- [ ] **Step 2: Run backend E2E tests**

```bash
cd backend && pnpm test:e2e --no-coverage
```

- [ ] **Step 3: Run frontend build**

```bash
cd frontend && pnpm build
```

- [ ] **Step 4: Run frontend lint**

```bash
cd frontend && pnpm lint
```

- [ ] **Step 5: Verify no broken imports across codebase**

```bash
cd backend && pnpm tsc --noEmit
```

- [ ] **Step 6: Final commit if any fixes needed**

---

## Summary

| Task | What | Type |
|------|------|------|
| 1 | Prisma migration (UNDER_REVIEW, reviewNotes, reviewedAt) | Schema |
| 2 | Schedule entity (new status, methods, events) | Domain |
| 3 | Repository (findActiveByFieldId, mapper updates) | Infra |
| 4 | QueryBus (FindActiveScheduleByFieldIdQuery) | Cross-domain |
| 5 | CreateScheduleUseCase (auto initial status) | Use case |
| 6 | ReviewScheduleUseCase (new) | Use case |
| 7 | CancelScheduleUseCase (PRINTED guard, auto-cancel) | Use case |
| 8 | Event subscribers (auto-transitions) | Domain events |
| 9 | Delete old use cases and controllers | Cleanup |
| 10 | Review controller (new endpoint) | HTTP |
| 11 | Harvest subscribers (complete, cancel) | Cross-domain |
| 12 | Frontend schemas, API, actions | Frontend |
| 13 | Frontend UI (dialogs, status badge, actions) | Frontend |
| 14 | Documentation | Docs |
| 15 | Final validation | QA |
