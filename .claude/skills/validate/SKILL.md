---
name: validate
description: Opus-only validation skill. Runs tests, analyzes coverage and mutation scores, reviews code against patterns, and lists all problems. Never writes code — sends corrections back to the local model via the plan file.
---

# Validate

Opus-only. Runs all validation phases (Phase 3 + Phase 5) and produces a clear report. **Never writes implementation code.** If problems are found, they are described precisely so the local model can fix them.

---

## Step 0 — Resolve plan

If a plan path was provided (e.g., `/validate docs/plans/2026-04-06-feature-name.md`):
- Read that file — including the Implementation Progress section

If no path was provided:
- List files in `docs/plans/` and ask which one to validate

Read the current branch: `git branch --show-current`

---

## Step 1 — Run validation suite

Launch in parallel (background):

```bash
# Backend
cd backend && pnpm test --coverage
cd backend && pnpm test:e2e
cd backend && tsc --noEmit
cd backend && stryker run --mutate <changed-files>

# Frontend (if in scope)
cd frontend && pnpm build
cd frontend && pnpm lint
cd frontend && tsc --noEmit
cd frontend && pnpm test:e2e
```

Show full output for each command. Do not summarize failures — show the exact error.

**Security audit:**
```bash
cd backend && pnpm audit
cd frontend && pnpm audit
```

**Secrets check:** Search changed files for hardcoded credentials, tokens, or secrets:
```bash
git diff development --name-only
```
For each changed file, verify no `password =`, `secret =`, `token =`, API keys, or connection strings with embedded credentials.

---

## Step 2 — Quality thresholds

Check against these thresholds. All must pass before the skill can emit OK:

| Metric | Minimum | Scope |
|--------|---------|-------|
| Coverage — domain + application layer | 100% | Backend |
| Coverage — infrastructure layer | 80% | Backend |
| Coverage — overall | 85% | Backend |
| Mutation score — domain + application | 90% | Backend (changed files) |
| Mutation score — infrastructure | 70% | Backend (changed files) |
| Coverage | 80% | Frontend |
| Build | passing | Frontend |
| Lint | zero errors | Frontend |
| Playwright E2E | all passing | Frontend |
| Type check | zero errors | Both |
| Security audit | zero critical CVEs | Both |

---

## Step 3 — Architectural conformance

Verify for all new use cases and controllers:
- Either return type on all use cases
- `@UseGuards` on all controllers (unless `@Public()`)
- Zod validation pipes on all endpoint inputs
- Paginated results on all list use cases
- No N+1 queries in repository implementations (no queries inside loops)
- No logging in domain layer (`src/domain/`)
- No cross-domain repository imports in use cases — QueryBus only

---

## Step 4 — Code review against patterns

Launch an `Explore` agent to verify the implementation against the patterns listed in the plan's "Patterns to read" section.

Check specifically:
- For every use case that creates or mutates: domain event emitted and `dispatchEventsForAggregate` called after save
- For frontend: no raw Tailwind colors (semantic tokens only), typography scale followed, `data-testid` on all interactive elements

---

## Step 5 — Report

**This skill never writes code fixes.** Problems are described precisely so the local model knows exactly what to change.

### If all checks pass:

Present to user:

```
## Validation Report — <feature-name>

### Test results
- Unit tests: X/X passing, coverage: X%
- E2E (API): X/X passing
- Playwright: X/X passing
- Mutation score: X%
- Type check: clean
- Build: passing
- Lint: zero errors
- Security: zero critical CVEs

### Architectural conformance
- ✓ Either return types on all use cases
- ✓ Guards on all controllers
- [...]

### Code review
- ✓ Patterns followed correctly
- ✓ Domain events emitted
- ✓ No raw Tailwind colors

STATUS: ✅ APPROVED
```

Then emit HANDOFF (OK):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /validate approved
Completed: all thresholds met, code reviewed and approved
Next step: run /commit when you are ready to commit and push
Waiting for: explicit /commit invocation — do not proceed automatically
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### If problems are found:

List every problem with:
- File path and line number
- What is wrong
- What must be done to fix it (precise instruction, no code)

Then update the plan file — append:

```markdown
## ⚠️ Validation Failed — [date]

### Problems found

#### Problem 1 — [category: coverage / conformance / pattern / test]
- **File:** `path/to/file.ts:42`
- **Issue:** [what is wrong]
- **Fix required:** [exact instruction for local model — no code, just description]

#### Problem 2 — ...

**Return to local model: run /implement and address these problems before re-running /validate.**
```

Then emit HANDOFF (corrections):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /validate rejected
Completed: validation ran, N problems found
Next step: in the local model terminal, read the plan file and fix the listed problems
Plan file: docs/plans/<plan-file>.md (problems listed at the bottom)
Waiting for: confirmation that fixes are applied — then re-run /validate
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
