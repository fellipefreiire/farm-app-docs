# E2E — Adding tests for a new domain

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Every new domain **must** include Playwright E2E tests as part of the implementation — never defer to a future task.

1. **Create the handler** at `tests/mocks/handlers/<domain>.handler.ts`:
   - Define interfaces matching the real API response (check Zod schemas in `src/domains/<domain>/schemas/`)
   - Add seed data with valid UUIDs (use the next available prefix: 00=Field, 10=CropType, 20=Variety, 30=Harvest, 40=Supplier, 50=next)
   - Pre-seed audit logs for at least one entity
   - Implement all CRUD endpoints + toggle-status + audit log endpoint
   - Export seed IDs and reset function

2. **Register the handler** in `tests/mocks/handlers.ts` — add import and spread into the handlers array

3. **Create the test file** at `tests/<domain>/manage-<entities>.e2e-spec.ts`:
   - Import from `../fixtures/auth.fixture`
   - Use `test.describe.serial()`
   - Order: list → search → create → validation → edit → status changes → delete
   - Delete tests LAST (removes entities from shared state)
   - All toast messages and status labels must be in **Portuguese** (check server action messages)
   - After edit/create submit, do NOT click `stacked-sheet-close` — verify `not.toBeVisible()` directly
   - Use `getByRole('cell', { name: '...' })` when text might match both table cell and toast

4. **Add audit log test** in `tests/audit/view-audit-logs.e2e-spec.ts`:
   - Add a test for the new domain's audit page using the pre-seeded audit log

5. **Verify components have `data-testid`** — every interactive element in the frontend must have a `data-testid` attribute following the conventions above

---
