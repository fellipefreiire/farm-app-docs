# Plan: Playwright E2E Tests with MSW Mocks

**Issue:** fellipefreiire/farm-app-docs#12
**Branch:** FE-12/playwright-e2e-msw

## Existing Infrastructure
- `instrumentation.ts` — already configured, starts MSW on `NODE_ENV=test`
- `tests/mocks/server.ts` — MSW setupServer
- `tests/mocks/handlers.ts` — handler aggregator (only auth)
- `tests/mocks/handlers/auth.handler.ts` — auth endpoints mocked
- `tests/fixtures/auth.fixture.ts` — authenticated page fixture
- `tests/auth/sign-in.e2e-spec.ts` — 5 existing tests
- `playwright.config.ts` — configured with webServer
- `docs/coding-patterns/frontend/e2e-test.md` — already complete

## Implementation Plan

### Wave 1 — Handlers (parallelizable)

Create MSW handlers with in-memory state for each domain:

| File | Endpoints |
|------|-----------|
| `tests/mocks/handlers/field.handler.ts` | POST, GET list, GET :id, PUT :id, DELETE :id, PATCH toggle, GET audit-logs |
| `tests/mocks/handlers/crop-type.handler.ts` | POST, GET list, GET :id, PUT :id, DELETE :id, GET audit-logs |
| `tests/mocks/handlers/variety.handler.ts` | POST, GET list, GET :id, PUT :id, DELETE :id, GET audit-logs |
| `tests/mocks/handlers/harvest.handler.ts` | POST, GET list, GET :id, PUT :id, DELETE :id, PATCH activate/complete/cancel, GET active, GET audit-logs |
| `tests/mocks/handlers/audit-log.handler.ts` | Shared audit log factory used by domain handlers |
| Update `tests/mocks/handlers.ts` | Aggregate all handlers |

Each handler:
- Maintains an in-memory array (reset between tests via exported `reset()`)
- Validates required fields (returns 422 on invalid)
- Returns responses matching real API contract (Zod schemas)
- Supports pagination (offset-based for lists, cursor-based for audit logs)

### Wave 2 — Update auth fixture

- Hardcode credentials (`test@example.com` / `password123`) instead of env vars
- Add `user.handler.ts` for user-related MSW responses if needed

### Wave 3 — Test files (parallelizable per domain)

| File | Tests |
|------|-------|
| `tests/field/manage-fields.e2e-spec.ts` | Create, edit, toggle, delete, list with pagination/search |
| `tests/crop/manage-crop-types.e2e-spec.ts` | Create, edit, delete, list |
| `tests/crop/manage-varieties.e2e-spec.ts` | Create, edit, delete, list |
| `tests/crop/manage-harvests.e2e-spec.ts` | Create, activate, complete, cancel, list |
| `tests/audit/view-audit-logs.e2e-spec.ts` | View audit logs for field, crop type, variety, harvest |

### Wave 4 — Verify and fix

- Run all tests together
- Fix flaky tests
- Verify sign-in tests still pass

## Files to create/modify

**Create:**
- tests/mocks/handlers/field.handler.ts
- tests/mocks/handlers/crop-type.handler.ts
- tests/mocks/handlers/variety.handler.ts
- tests/mocks/handlers/harvest.handler.ts
- tests/mocks/handlers/audit-log.handler.ts
- tests/field/manage-fields.e2e-spec.ts
- tests/crop/manage-crop-types.e2e-spec.ts
- tests/crop/manage-varieties.e2e-spec.ts
- tests/crop/manage-harvests.e2e-spec.ts
- tests/audit/view-audit-logs.e2e-spec.ts

**Modify:**
- tests/mocks/handlers.ts (add new handlers)
- tests/fixtures/auth.fixture.ts (hardcode credentials)
