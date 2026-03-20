# Plan: FieldTicket Domain

**Issue:** fellipefreiire/farm-app-docs#24
**Branch:** FS-24/field-ticket (backend + frontend)
**Date:** 2026-03-19

---

## Overview

FieldTicket IS the operation. It replaces ScheduleOperation/ScheduleOperationInput. This plan covers:
1. Creating the FieldTicket domain (backend + frontend)
2. Removing ScheduleOperation/Input from Schedule domain
3. Refactoring Schedule to query FieldTickets via QueryBus
4. Building the execution panel (frontend)

---

## Backend Waves

### Wave 1 — Domain Layer (entities, events, errors) — NO DEPENDENCIES

**Entities:**
- `src/domain/field-ticket/enterprise/entities/field-ticket.ts`
- `src/domain/field-ticket/enterprise/entities/field-ticket-input.ts`

**Events (8):**
- `src/domain/field-ticket/enterprise/events/field-ticket-created-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-reviewed-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-updated-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-printed-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-completed-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-re-evaluated-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-cancelled-event.ts`
- `src/domain/field-ticket/enterprise/events/field-ticket-deleted-event.ts`

**Errors:**
- `src/domain/field-ticket/application/use-cases/errors/field-ticket-not-found-error.ts`
- `src/domain/field-ticket/application/use-cases/errors/invalid-status-transition-error.ts`
- `src/domain/field-ticket/application/use-cases/errors/cannot-cancel-completed-ticket-error.ts`
- `src/domain/field-ticket/application/use-cases/errors/operation-date-out-of-range-error.ts`
- `src/domain/field-ticket/application/use-cases/errors/schedule-not-found-error.ts` (local copy)
- `src/domain/field-ticket/application/use-cases/errors/schedule-not-editable-error.ts` (local copy)
- `src/domain/field-ticket/application/use-cases/errors/input-not-found-error.ts` (local copy)
- `src/domain/field-ticket/application/use-cases/errors/vehicle-not-found-error.ts` (local copy)
- `src/domain/field-ticket/application/use-cases/errors/implement-not-found-error.ts` (local copy)

### Wave 2 — Application Layer (repos, use cases, QueryBus) — DEPENDS ON W1

**Repository interfaces:**
- `src/domain/field-ticket/application/repositories/field-tickets-repository.ts`
- `src/domain/field-ticket/application/repositories/field-ticket-inputs-repository.ts`

**QueryBus (handler for Schedule to query FieldTickets):**
- `src/domain/field-ticket/application/queries/find-field-tickets-by-schedule-id.query.ts`
- `src/domain/field-ticket/application/handlers/find-field-tickets-by-schedule-id.handler.ts`

**Use cases (10):**
- `src/domain/field-ticket/application/use-cases/create-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/review-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/edit-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/print-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/finalize-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/re-evaluate-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/cancel-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/delete-field-ticket.ts`
- `src/domain/field-ticket/application/use-cases/list-field-tickets.ts`
- `src/domain/field-ticket/application/use-cases/find-field-ticket-by-id.ts`

### Wave 3 — Test Support + Unit Tests — DEPENDS ON W2

**Factories:**
- `test/factories/make-field-ticket.ts`
- `test/factories/make-field-ticket-input.ts`

**InMemory repos:**
- `test/repositories/field-ticket/in-memory-field-tickets-repository.ts`
- `test/repositories/field-ticket/in-memory-field-ticket-inputs-repository.ts`

**Unit tests (10):**
- `src/domain/field-ticket/application/use-cases/__tests__/create-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/review-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/edit-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/print-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/finalize-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/re-evaluate-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/cancel-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/delete-field-ticket.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/list-field-tickets.spec.ts`
- `src/domain/field-ticket/application/use-cases/__tests__/find-field-ticket-by-id.spec.ts`

### Wave 4 — Prisma + Infrastructure DB — DEPENDS ON W2

**Prisma schema changes:**
- Add `FieldTicket` model + enums (OperationType, FieldTicketStatus, BarType, TurbineType, GearType)
- Add `FieldTicketInput` model
- Remove `ScheduleOperation` model
- Remove `ScheduleOperationInput` model
- Migration: `prisma generate` + `prisma migrate dev`

**Mappers:**
- `src/infra/database/prisma/mappers/field-ticket/prisma-field-ticket.mapper.ts`
- `src/infra/database/prisma/mappers/field-ticket/prisma-field-ticket-input.mapper.ts`

**Prisma repos:**
- `src/infra/database/prisma/repositories/field-ticket/prisma-field-tickets.repository.ts`
- `src/infra/database/prisma/repositories/field-ticket/prisma-field-ticket-inputs.repository.ts`
- `src/infra/database/prisma/repositories/field-ticket/field-ticket-database.module.ts`

### Wave 5 — HTTP Layer — DEPENDS ON W4

**Controllers (11):**
- `src/infra/http/controllers/field-ticket/create-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/review-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/edit-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/print-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/finalize-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/re-evaluate-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/cancel-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/delete-field-ticket.controller.ts`
- `src/infra/http/controllers/field-ticket/list-field-tickets.controller.ts`
- `src/infra/http/controllers/field-ticket/find-field-ticket-by-id.controller.ts`
- `src/infra/http/controllers/field-ticket/list-field-ticket-audit-logs.controller.ts`

**Presenters:**
- `src/infra/http/presenters/field-ticket/field-ticket.presenter.ts`
- `src/infra/http/presenters/field-ticket/field-ticket-input.presenter.ts`

**Error filter:**
- `src/infra/http/filters/field-ticket-error.filter.ts`

**Module:**
- `src/infra/http/controllers/field-ticket/field-ticket-http.module.ts`

**Audit subscribers (8):**
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-created.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-reviewed.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-updated.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-printed.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-completed.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-re-evaluated.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-cancelled.ts`
- `src/domain/audit-log/application/subscribers/field-ticket/on-field-ticket-deleted.ts`

**Events module:**
- `src/infra/events/field-ticket/field-ticket-events.module.ts`

### Wave 6 — Schedule Refactoring — DEPENDS ON W5

**Delete (~80 files):**
- All ScheduleOperation/Input entities, events, use cases, tests, errors
- All ScheduleOperation/Input repos (interface + prisma + in-memory)
- All ScheduleOperation/Input mappers, controllers, presenters
- All ScheduleOperation/Input audit subscribers
- ScheduleOperation/Input test factories

**Modify:**
- `src/domain/schedule/application/use-cases/copy-schedule.ts` → use FieldTicket QueryBus
- `src/domain/schedule/application/use-cases/copy-schedule-operations-to-day.ts` → use FieldTicket QueryBus
- `src/domain/schedule/application/use-cases/get-schedule.ts` → include FieldTickets via QueryBus
- `src/domain/schedule/application/use-cases/delete-schedule.ts` → check FieldTickets instead of operations
- Schedule modules (DB, HTTP, Events) → remove Operation providers
- Schedule error filter → remove operation errors
- Update unit tests for modified use cases

### Wave 7 — E2E Controller Tests — DEPENDS ON W5, W6

- `src/infra/http/controllers/field-ticket/__tests__/create-field-ticket.controller.e2e-spec.ts`
- `src/infra/http/controllers/field-ticket/__tests__/list-field-tickets.controller.e2e-spec.ts`
- `src/infra/http/controllers/field-ticket/__tests__/find-field-ticket-by-id.controller.e2e-spec.ts`
- (more as needed)

---

## Frontend Waves

### Wave 8 — Schemas + API — NO DEPENDENCIES

**Schemas:**
- `src/domains/field-ticket/schemas/field-ticket.schema.ts`
- `src/domains/field-ticket/schemas/field-ticket-input.schema.ts`
- `src/domains/field-ticket/schemas/create-field-ticket.schema.ts`
- `src/domains/field-ticket/schemas/review-field-ticket.schema.ts`
- `src/domains/field-ticket/schemas/edit-field-ticket.schema.ts`
- `src/domains/field-ticket/schemas/finalize-field-ticket.schema.ts`
- `src/domains/field-ticket/schemas/list-field-tickets.schema.ts`
- `src/domains/field-ticket/schemas/index.ts`

**API:**
- `src/domains/field-ticket/api/create-field-ticket.ts`
- `src/domains/field-ticket/api/review-field-ticket.ts`
- `src/domains/field-ticket/api/edit-field-ticket.ts`
- `src/domains/field-ticket/api/print-field-ticket.ts`
- `src/domains/field-ticket/api/finalize-field-ticket.ts`
- `src/domains/field-ticket/api/re-evaluate-field-ticket.ts`
- `src/domains/field-ticket/api/cancel-field-ticket.ts`
- `src/domains/field-ticket/api/delete-field-ticket.ts`
- `src/domains/field-ticket/api/list-field-tickets.ts`
- `src/domains/field-ticket/api/find-field-ticket-by-id.ts`
- `src/domains/field-ticket/api/index.ts`

### Wave 9 — Actions + Store — DEPENDS ON W8

**Actions:**
- `src/domains/field-ticket/actions/create-field-ticket.ts`
- `src/domains/field-ticket/actions/review-field-ticket.ts`
- `src/domains/field-ticket/actions/edit-field-ticket.ts`
- `src/domains/field-ticket/actions/print-field-ticket.ts`
- `src/domains/field-ticket/actions/finalize-field-ticket.ts`
- `src/domains/field-ticket/actions/re-evaluate-field-ticket.ts`
- `src/domains/field-ticket/actions/cancel-field-ticket.ts`
- `src/domains/field-ticket/actions/delete-field-ticket.ts`
- `src/domains/field-ticket/actions/index.ts`

**Store:**
- `src/domains/field-ticket/store/field-ticket-dialogs.store.ts`
- `src/domains/field-ticket/store/index.ts`

### Wave 10 — Components — DEPENDS ON W9

**Execution panel:**
- `src/domains/field-ticket/components/execution-panel/awaiting-finalization-section.tsx`
- `src/domains/field-ticket/components/execution-panel/today-operations-section.tsx`
- `src/domains/field-ticket/components/execution-panel/other-days-section.tsx`
- `src/domains/field-ticket/components/execution-panel/ticket-card.tsx`
- `src/domains/field-ticket/components/execution-panel/stage-column.tsx`

**Forms & Sheets:**
- `src/domains/field-ticket/components/forms/create-field-ticket-form.tsx`
- `src/domains/field-ticket/components/forms/review-field-ticket-form.tsx`
- `src/domains/field-ticket/components/forms/finalize-field-ticket-form.tsx`
- `src/domains/field-ticket/components/sheets/create-field-ticket-sheet.tsx`
- `src/domains/field-ticket/components/sheets/review-field-ticket-sheet.tsx`
- `src/domains/field-ticket/components/sheets/finalize-field-ticket-sheet.tsx`
- `src/domains/field-ticket/components/sheets/field-ticket-detail-sheet.tsx`

**Dialogs:**
- `src/domains/field-ticket/components/dialogs/cancel-field-ticket-dialog.tsx`
- `src/domains/field-ticket/components/dialogs/re-evaluate-field-ticket-dialog.tsx`

**Barrel:**
- `src/domains/field-ticket/components/index.ts`

### Wave 11 — Pages + Schedule Refactoring — DEPENDS ON W10

**New pages:**
- `src/app/(private)/field-tickets/page.tsx`
- `src/app/(private)/field-tickets/loading.tsx`

**Schedule frontend refactoring:**
- Remove schedule operation schemas, API, actions (replaced by field-ticket)
- Refactor week-view/operation-card to use FieldTicket data
- Refactor add/edit operation sheets to call FieldTicket actions
- Update schedule-dialogs.store.ts

**Audit integration:**
- Update `action-label-map.ts` — add field-ticket actions, remove schedule-operation actions
- Update `limited-audit-logs.tsx` — add field-ticket action text
- Update `audit-diff-modal.tsx` — add field-ticket field labels

**Sidebar navigation:**
- Add "Painel de Execução" link below Dashboard

### Wave 12 — E2E Tests — DEPENDS ON W11

**MSW handlers:**
- `tests/mocks/handlers/field-ticket.handler.ts`
- Update `tests/mocks/handlers/schedule.handler.ts` (remove operation endpoints)

**Playwright tests:**
- `tests/field-ticket/execution-panel.e2e-spec.ts`
- Update `tests/schedule/` tests if they reference operations

---

## Parallel Execution Strategy

| Wave | Parallel agents | Model |
|------|----------------|-------|
| W1 | 2 agents (entities+events ∥ errors) | Sonnet |
| W2 | 3 agents (repos ∥ use cases batch 1 ∥ use cases batch 2) | Sonnet |
| W3 | 3 agents (factories+inmemory ∥ tests batch 1 ∥ tests batch 2) | Sonnet |
| W4 | 1 agent (prisma schema + migration), then 2 agents (mappers ∥ prisma repos) | Sonnet |
| W5 | 3 agents (controllers batch 1 ∥ controllers batch 2 ∥ presenters+filter+modules+audit) | Sonnet |
| W6 | 2 agents (delete files + modify schedule use cases ∥ update modules) | Sonnet |
| W7 | 1 agent (E2E tests) | Sonnet |
| W8-9 | 2 agents (schemas+api ∥ actions+store) | Sonnet |
| W10 | 2 agents (execution panel ∥ forms+sheets+dialogs) | Sonnet |
| W11 | 2 agents (new pages ∥ schedule refactoring + audit integration) | Sonnet |
| W12 | 1 agent (MSW + Playwright) | Sonnet |

---

## Notes

- User authorized data deletion — migration can drop ScheduleOperation tables
- FieldTicket is flat domain (2 entities, but FieldTicketInput is a child — no subdomains needed)
- Schedule frontend components (week-view, gantt-view) stay in schedule domain but call FieldTicket APIs
- Execution panel is the new `/field-tickets` page
- All enums (OperationType, FieldTicketStatus, BarType, TurbineType, GearType) defined in Prisma
