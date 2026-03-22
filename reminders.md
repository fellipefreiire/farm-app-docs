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

## Pinned dependency versions

Dependencies listed here were intentionally kept at their current version during a health-check. They must **not** be flagged as outdated in future health-checks — but they **must** be re-evaluated each time to check if the blocker was resolved (e.g., peer dep updated, migration completed).

| Package | Repo | Pinned at | Latest skipped | Reason | Re-evaluate when |
|---------|------|-----------|----------------|--------|------------------|
| `class-validator` | backend | ^0.14.4 | 0.15.1 | Peer dependency conflict: `@nestjs/mapped-types` requires `^0.13.0 \|\| ^0.14.0` | `@nestjs/mapped-types` releases a version supporting `^0.15` |
| `@types/node` | backend + frontend | ^24.x | 25.x | Node 24 is LTS; @types/node must match runtime LTS version | Node 26 becomes LTS (October 2026) — then update to `@types/node@^26` |
| `eslint` | frontend | ^9.x | 10.x | Architectural breaking change: removes `.eslintrc` config system entirely. Dedicated migration needed (fellipefreiire/farm-app-docs#22) | Issue #22 is completed |

---

## Gotchas

<!-- Add gotchas discovered during development. Remove them when they are resolved or no longer relevant. -->
<!-- Format: one line per gotcha, prefixed with dash. Be specific — include file, library, or context. -->

- **Turbopack cache corruption**: If Playwright tests hang with `turbo-persistence` panics (`range start index out of range for slice`), delete `frontend/.next/` entirely and re-run. The turbopack persistent cache can become corrupt after branch switches or large file changes.
- **Stryker CLI --mutate overrides config exclusions**: When using `--mutate 'src/domain/x/**/*.ts'` it replaces the entire `mutate` array from `stryker.config.mjs`, including `!src/**/__tests__/**` exclusions. Use specific paths like `--mutate 'src/domain/x/**/use-cases/*.ts'` to avoid mutating test files.
- **Prisma migrate dev shadow DB enum issue**: `prisma migrate dev` fails with `P3006` on migrations that add enum values and use them in the same transaction. Workaround: use `prisma db push` + manual migration file + `prisma migrate resolve --applied`.

---

## Future improvements (post-MVP)

### Inventory
- Lot/batch tracking with expiry dates for inputs
- Minimum stock alerts (notify when input stock is low)
- Soft reservation: Schedule reserves stock for planning; alert when more product needs to be purchased to continue the plan
- Input enrichment: manufacturer, commercial code, concentration fields
- Unit price calculation from total purchase value (analytics/reporting)
- Automatic StockMovement (exit) triggered by FieldTicket finalization — implement when FieldTicket domain is built (MVP, not post-MVP)

### Schedule
- Melhorar "Copiar Operações" — copiar operações entre dias ainda usa dialog simples, pode evoluir para wizard com preview (pendente desde 2026-03-16)

### Supplier
- Price comparison across suppliers for the same input
- Purchase history and supplier performance tracking

### Fleet
- Maintenance scheduling (preventive and corrective)
- Fuel tracking and consumption per vehicle
- Operational hours tracking
- Cost per vehicle/implement
- Operator assignment (link to User)
- Additional VehicleType values (MOTORCYCLE, CAR, TRUCK)
- Vehicle-implement compatibility matrix
- Vehicle inspection checklists

