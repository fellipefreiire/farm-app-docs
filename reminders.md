# Reminders

Living notes for gotchas, pitfalls, and things that went wrong during development. This file prevents the same mistakes from happening twice.

---

## How Claude uses this document

- **At the start of every task:** Read gotchas to avoid repeating known mistakes.
- **After fixing a bug or discovering a pitfall:** Add it here immediately.
- **During Phase 4 (Documentation):** Add new gotchas discovered, remove resolved ones.

---

## Health check

health-check-counter: 2

The `health-check-counter` tracks how many features have been merged since the last `/health-check`. It is incremented by `/feature`, `/new-domain`, and `/small-change` after a PR is successfully created. When it reaches `HEALTH_CHECK_THRESHOLD` (defined in CLAUDE.md), Claude warns before starting the next task. It resets to 0 after `/health-check` completes.

---

## Maintenance

- Add gotchas as soon as they are discovered — do not wait until the end of a session
- Remove gotchas when the root cause is fixed or the information is no longer relevant
- Keep entries specific: include file path, library name, or exact scenario
- This file is read at the start of every implementation task — keep it concise

---

## Gotchas

<!-- Add gotchas discovered during development. Remove them when they are resolved or no longer relevant. -->
<!-- Format: one line per gotcha, prefixed with dash. Be specific — include file, library, or context. -->

- Phase 4 doc updates were skipped in past sessions, causing `api-reference.md`, `flows.md`, and `architecture.md` to go stale. Every implementation MUST update the docs listed in Phase 4 (`shared-phases.md` lines 88-95) — never skip even for small changes.
- `.env.test` must NOT contain `DATABASE_URL` — it breaks E2E tests because NestJS `ConfigModule` uses it instead of the unique schema URL set by `setup-e2e.ts`.
- Redis cache can serve stale data after schema migrations. When adding new fields to an entity, flush Redis (`redis-cli FLUSHDB`) after migration. The `findById` repository method caches full entities — cached entries from before the migration won't have the new field.
- Frontend toast/UI strings must be in Portuguese. Backend API responses stay in English (per CLAUDE.md rule). The translation happens in the frontend action files and components, not in the backend.
- Stryker requires `appendPlugins: ['@stryker-mutator/vitest-runner']` in `stryker.config.mjs` because pnpm isolates packages and Stryker's auto-discovery doesn't find the vitest-runner. Without this, Stryker silently fails with "Cannot find TestRunner plugin 'vitest'".
- Playwright E2E tests use port 3001 (not 3000) to avoid conflicts with the dev server. If tests fail on `page.goto()`, check for stale `.next/dev/lock` file and residual processes on port 3001.
- Playwright test assertions must use Portuguese text for toasts, validation messages, and status labels. English text causes silent assertion failures (timeout waiting for invisible text).

---

## Future improvements (post-MVP)

### Inventory
- Lot/batch tracking with expiry dates for inputs
- Minimum stock alerts (notify when input stock is low)
- Soft reservation: Schedule reserves stock for planning; alert when more product needs to be purchased to continue the plan
- Input enrichment: manufacturer, commercial code, concentration fields
- Unit price calculation from total purchase value (analytics/reporting)
- Automatic StockMovement (exit) triggered by FieldTicket finalization — implement when FieldTicket domain is built (MVP, not post-MVP)

### Supplier
- Price comparison across suppliers for the same input
- Purchase history and supplier performance tracking

