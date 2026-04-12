# Farm-App Verification Rules

Farm-app-specific quality gates and architectural conformance checks. Extracted
from the retired `docs/shared-phases.md` — this file owns everything that was
farm-app-specific in Phases 3-5 (validation, architectural checks, Stryker,
secrets, Playwright troubleshooting).

Invoked by:
- `/finish` — runs verification before allowing git integration
- `/health-check` — audits current state against these rules
- `/migrate` — runs validation after a major dependency update

Everything in this file is farm-app-specific. Generic verification discipline
(TDD, root cause analysis, evidence before assertions) lives in
`superpowers:verification-before-completion` and `superpowers:systematic-debugging`.

## Quality thresholds

All of these must pass before claiming completion or allowing a commit/PR:

| Metric | Minimum | Scope |
|---|---|---|
| Coverage — domain + application layer | 100% | Backend |
| Coverage — infrastructure layer | 80% | Backend |
| Coverage — overall | 85% | Backend |
| Mutation score — domain + application | 90% | Backend (changed files only) |
| Mutation score — infrastructure | 70% | Backend (changed files only) |
| Coverage | 80% | Frontend |
| Build | passing | Frontend |
| Lint | zero errors | Frontend |
| Playwright E2E | all passing | Frontend |
| Type check | zero errors | Both |
| Security audit | zero critical CVEs | Both |

**On threshold failure:**
- Coverage below minimum → write additional tests targeting uncovered lines. Re-run and verify.
- Mutation score below minimum → analyze surviving mutants, add assertions that kill them. Re-run and verify.
- Playwright failing → see troubleshooting section below.
- After 2 focused attempts still failing → present the gap to the user with specific uncovered lines or surviving mutants, ask whether to continue writing tests or accept the score with justification in the PR body.

**Never claim verification passed without fresh output in the current session.**

## Architectural conformance checklist

For every implementation that introduces or modifies use cases or controllers:

**Backend:**
- [ ] `Either<Error, { data }>` return type on all use cases
- [ ] `@UseGuards` on all controllers (unless explicitly `@Public()`)
- [ ] Zod validation pipes on all endpoint inputs
- [ ] Paginated results on all list use cases (offset or cursor)
- [ ] No N+1 queries in repositories (no queries inside loops)
- [ ] No logging in domain layer (`backend/src/domain/`)
- [ ] No cross-domain repository or error imports in use cases — cross-domain data goes through `QueryBus` (see `docs/coding-patterns/backend/query-bus.md`)
- [ ] Allowed cross-domain imports in use cases: query contracts (`application/queries/*.query.ts`) and `AuditLogRepository` (shared)
- [ ] Subscribers are **exempt** from the cross-domain import rule

**Frontend:**
- [ ] Compliance with `docs/coding-patterns/frontend/design-system.md`
- [ ] No raw Tailwind colors — must use semantic tokens
- [ ] Typography follows the defined scale
- [ ] Every custom color token has both `:root` and `.dark` values
- [ ] All interactive elements have `data-testid` attributes (required for Playwright)
- [ ] All dates from the API use `getUTC*()` methods (see `docs/coding-patterns/frontend/date-handling.md`)

## Stryker (mutation testing) — scope and rules

**Default scope** when running `stryker run` without flags:
- Mutates: `src/domain/`, `src/shared/`, `src/core/` (business logic)
- Does NOT mutate: `src/infra/` (infrastructure) — only on explicit `--mutate` flag

**Runs only unit tests.** Never E2E.

**Timeout policy:** if mutation testing exceeds 10 minutes, STOP the background
Stryker process and ask the user: *"Mutation testing is taking longer than
expected. Wait or skip?"* Never let Stryker run indefinitely.

If skipped, mark mutation score as `skipped (timeout)` in the review summary
and PR body. A skipped score does **not** block `/finish` — it's a documented gap.

**`--mutate` flag gotcha:** passing `--mutate 'src/domain/x/**/*.ts'`
**replaces** the entire `mutate` array from `stryker.config.mjs`, **including**
the `!src/**/__tests__/**` exclusion. Use specific paths like
`--mutate 'src/domain/x/**/use-cases/*.ts'` to avoid mutating test files.

See `docs/coding-patterns/backend/mutation-testing.md` for full configuration
and surviving-mutant analysis.

## Secrets check

On every verification run, search changed files for:
- `password =`, `secret =`, `token =`
- API keys hardcoded inline
- Connection strings with embedded credentials

Verify `.env` files are in `.gitignore`. If secrets are found, **remove them
immediately** and add the variable to `.env.example` with a placeholder value.

## Playwright troubleshooting

If Playwright tests fail, check in order:

1. **Stale dev server without MSW on E2E port** — E2E port is `DEV_PORT + 1`. Fix: `rm -f .next/dev/lock` and restart.
2. **MSW handler response shape doesn't match Zod schema** — compare with `src/domains/<domain>/schemas/`.
3. **Toast/status text in English instead of Portuguese** — the frontend uses Portuguese copy.
4. **Clicking `stacked-sheet-close` after form auto-closes** — the form already closed, the click targets a detached element.
5. **Turbopack cache corruption** — if you see panics like `range start index out of range for slice`, delete `frontend/.next/` entirely and re-run.

See `docs/coding-patterns/frontend/e2e-test.md` Troubleshooting section for the full list.

## Security — hard rule

Never commit `.env` files (must be in `.gitignore`). Never hardcode credentials,
tokens, or secrets in code. Environment variables are always read from `process.env`.
If a new env var is needed, add it to `.env.example` with a placeholder and
document it in the PR body.
