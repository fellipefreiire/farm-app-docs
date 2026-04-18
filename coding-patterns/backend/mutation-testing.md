Compressing inline (no file path provided, applying rules directly):

---

# Mutation Testing Pattern

Stryker Mutator with Vitest runner. Validates tests detect bugs — not just execute code. Runs only against unit tests (never E2E).

---

## File locations

```
stryker.config.mjs               ← Stryker configuration
vitest.config.mjs                ← Vitest config used by Stryker (unit tests only)
reports/mutation/                 ← HTML report output (gitignored)
```

---

## Configuration

```js
// stryker.config.mjs
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
export default {
  appendPlugins: ['@stryker-mutator/vitest-runner'],
  testRunner: 'vitest',
  vitest: {
    configFile: 'vitest.config.mjs',
  },
  mutate: [
    'src/domain/**/*.ts',
    'src/shared/**/*.ts',
    'src/core/**/*.ts',
    '!src/**/__tests__/**',
    '!src/**/*.spec.ts',
    '!src/**/*.e2e-spec.ts',
  ],
  reporters: ['clear-text', 'progress', 'html'],
  coverageAnalysis: 'perTest',
  thresholds: { high: 90, low: 70, break: 70 },
}
```

### What each option does

| Option | Value | Why |
|--------|-------|-----|
| `appendPlugins` | `['@stryker-mutator/vitest-runner']` | **Required with pnpm.** pnpm isolates packages — Stryker auto-discovery only finds plugins in its own `node_modules`, excluding vitest-runner. Explicitly loads it. |
| `testRunner` | `vitest` | Matches project test framework |
| `vitest.configFile` | `vitest.config.mjs` | Unit test config — excludes E2E tests |
| `mutate` | domain + shared + core | Default scope — where business logic lives |
| `!__tests__`, `!*.spec.ts`, `!*.e2e-spec.ts` | excluded | Never mutate test files |
| `reporters` | clear-text, progress, html | Terminal output + HTML report for analysis |
| `coverageAnalysis` | `perTest` | Only runs tests covering mutated code (faster) |
| `thresholds` | high: 90, low: 70, break: 70 | Global minimum — per-layer thresholds enforced in Phase 3 |

---

## Scope

### Default scope (automatic)

`mutate` array covers:
- `src/domain/**/*.ts` — entities, use cases, events, subscribers, errors
- `src/shared/**/*.ts` — cryptography abstractions, utilities (sanitize, withTimeout, retryWithBackoff)
- `src/core/**/*.ts` — Either, Entity, AggregateRoot, WatchedList, DomainEvents

Tested by **unit tests only** (`.spec.ts`). Stryker never runs E2E tests.

### Infrastructure scope (on demand)

`src/infra/` not in default `mutate` scope. Run explicitly when needed:

```bash
stryker run --mutate 'src/infra/auth/**/*.ts'
stryker run --mutate 'src/infra/cryptography/**/*.ts'
stryker run --mutate 'src/infra/http/presenters/**/*.ts'
stryker run --mutate 'src/infra/http/pipes/**/*.ts'
stryker run --mutate 'src/infra/http/filters/**/*.ts'
```

Use for:
- New infra code with unit tests
- `/health-check` or `/code-audit` when mutation score requested for infra
- Targeted analysis of specific infra modules

---

## Running

```bash
# Full default scope (domain + shared + core)
cd backend && stryker run

# Specific changed files (used in Phase 3)
cd backend && stryker run --mutate 'src/domain/user/application/use-cases/create-user.ts'

# Multiple files
cd backend && stryker run --mutate 'src/domain/user/**/*.ts'

# Infrastructure on demand
cd backend && stryker run --mutate 'src/infra/http/presenters/**/*.ts'
```

---

## Quality thresholds (Phase 3)

Per-layer thresholds enforced during Phase 3 validation:

| Layer | Minimum mutation score |
|-------|----------------------|
| Domain + application (`src/domain/`) | 90% |
| Shared + core (`src/shared/`, `src/core/`) | 90% |
| Infrastructure (`src/infra/`) — when tested | 70% |

`break: 70` is global safety net. Per-layer thresholds above are stricter — enforced by verification process in `docs/verification-rules.md`, invoked by `/finish` before integration.

---

## Reading the report

After run, open `reports/mutation/mutation.html` for detailed analysis.

### Mutant statuses

| Status | Meaning | Action |
|--------|---------|--------|
| **Killed** | Test caught mutation | Good — no action needed |
| **Survived** | No test caught mutation | Add or strengthen assertions |
| **No coverage** | No test covers this code | Write test for this code path |
| **Timeout** | Mutant caused infinite loop | Usually fine — counts as detected |
| **Runtime error** | Mutant broke compilation | Usually fine — counts as detected |

### Fixing surviving mutants

Surviving mutant = tests still pass if that line changed. Common fixes:

```ts
// Surviving mutant: changed `>` to `>=` in pagination
// Fix: add boundary test
it('should return empty when page exceeds total', async () => {
  const result = await sut.execute({ page: 999 })
  expect(result.isRight()).toBe(true)
  if (result.isRight()) {
    expect(result.value.data).toHaveLength(0)
  }
})

// Surviving mutant: removed `throw` in error path
// Fix: explicitly test the error case
it('should return error when entity not found', async () => {
  const result = await sut.execute({ id: 'non-existent' })
  expect(result.isLeft()).toBe(true)
  expect(result.value).toBeInstanceOf(EntityNotFoundError)
})

// Surviving mutant: changed string literal in event
// Fix: assert the exact event payload
expect(inMemoryAuditLogRepository.items[0]).toEqual(
  expect.objectContaining({
    action: 'user:created',  // assert exact string, not just presence
  }),
)
```

---

## Timeout handling

- **Per-mutant timeout:** Stryker default (`timeoutMS`). Hanging mutant killed, marked timeout (counts as detected).
- **Total process timeout:** 10 minutes, enforced by `/finish` verification. If exceeded, user asked to wait or skip. See `docs/verification-rules.md` (Stryker section) for full timeout policy.

---

## Rules

- Stryker runs **only unit tests** — never E2E (`.e2e-spec.ts`)
- Default scope: `src/domain/`, `src/shared/`, `src/core/` — business logic only
- `src/infra/` mutated **on demand** via `--mutate` flag
- All non-E2E test files (`.spec.ts`) must score above threshold
- Use `--mutate <changed-files>` in Phase 3 to scope to current change
- HTML report in `reports/mutation/` (gitignored)
- Surviving mutant = missing/weak assertion — fix test, not code
- Never disable or skip mutators to game score

---

## Anti-patterns

```ts
// WRONG: running Stryker against E2E tests
stryker run --mutate 'src/infra/http/controllers/**/*.ts'
// E2E tests are too slow for mutation testing — use unit tests only

// WRONG: no --mutate flag in Phase 3
stryker run  // runs against ALL default files — scope to changed files

// WRONG: weakening assertions to kill mutants
expect(result).toBeTruthy()  // too weak — mutant changing return value still passes
// use: expect(result.isRight()).toBe(true)

// WRONG: adding Stryker ignore comments to skip mutations
// @stryker-disable-next-line
if (entity.ownerId !== actorId) ...  // never skip — strengthen the test instead

// WRONG: testing implementation details to kill mutants
expect(spy).toHaveBeenCalledTimes(1)  // fragile — test behavior, not call count
// use: expect(repository.items).toHaveLength(1)  // assert the outcome
```