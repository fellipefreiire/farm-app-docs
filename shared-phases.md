# Shared Phases

These phases are used by all implementation skills (New Domain, Feature, Small Change). Phases 0-2 (Refinement, Planning, Implementation) are specific to each skill and live in their respective SKILL.md files. Phases 3-6 (Validation, Documentation, Code Review, Commit/PR) are shared here to avoid duplication — all implementation skills reference this file after completing their Phase 2.

## How Claude uses this document

- **After Phase 2 (Implementation):** Every implementation skill (`/feature`, `/new-domain`, `/small-change`) references this file for Phases 3-6.
- **Phase 3 gates Phase 4:** Never advance if quality thresholds are not met.
- **Phase 5 gates Phase 6:** Never commit without explicit user approval.
- **Phase 6 ends the skill:** After PR creation, the skill is complete.

---

## Phase 3 — Validation

Launch in parallel (background):

```bash
# Backend
cd backend && pnpm test --coverage                      # unit tests + coverage
cd backend && pnpm test:e2e                             # API end-to-end tests
cd backend && tsc --noEmit                              # type check
cd backend && stryker run --mutate <changed-files>      # mutation testing (see docs/coding-patterns/backend/mutation-testing.md)

# Frontend
cd frontend && pnpm build
cd frontend && pnpm lint
cd frontend && tsc --noEmit
cd frontend && pnpm test:e2e                            # Playwright (runs with MSW mocks, no backend needed)
```

**Stryker scope:** Default config mutates `src/domain/`, `src/shared/`, `src/core/` (business logic). Infrastructure (`src/infra/`) is mutated only on demand via `--mutate` flag. Stryker runs only unit tests — never E2E. See `docs/coding-patterns/backend/mutation-testing.md` for full configuration, scope rules, and surviving mutant analysis.

**Stryker timeout:** If mutation testing exceeds 10 minutes, the main agent must stop the background Stryker process and ask the user: "Mutation testing is taking longer than expected. Do you want to wait or skip mutation testing?" Never let Stryker run indefinitely. If the user chooses to skip, proceed to quality thresholds — mark mutation score as "skipped (timeout)" in the review summary and PR body. A skipped mutation score does not block Phase 6.

**On failure — Systematic Debugging:**

1. Read the full error output — never assume the cause
2. Reproduce consistently before attempting any fix
3. Trace the root cause — do not patch symptoms
4. Fix one thing at a time, verify after each change
5. If 2 fixes fail on the same issue → stop, raise with user

```bash
# Security audit (both repos)
cd backend && pnpm audit
cd frontend && pnpm audit
```

**Secrets check:** Verify no hardcoded credentials, tokens, or secrets in changed files. Search for patterns like `password =`, `secret =`, `token =`, API keys, or connection strings with embedded credentials. Verify `.env` files are in `.gitignore`. If any secrets are found, remove them immediately and add the variable to `.env.example` with a placeholder.

**Quality thresholds — all must pass before advancing to Phase 4:**

| Metric | Minimum | Scope |
|--------|---------|-------|
| Coverage — domain + application layer | 100% | Backend |
| Coverage — infrastructure layer | 80% | Backend |
| Coverage — overall | 85% | Backend |
| Mutation score — domain + application layer | 90% | Backend (changed files) |
| Mutation score — infrastructure layer | 70% | Backend (changed files) |
| Coverage | 80% | Frontend |
| Build | passing | Frontend |
| Lint | zero errors | Frontend |
| Playwright E2E | all passing | Frontend |
| Type check | zero errors | Both |
| Security audit | zero critical CVEs | Both |

**Architectural conformance:** If this implementation includes new use cases or controllers, verify:
- Either return type on all use cases
- `@UseGuards` on all controllers (unless `@Public()`)
- Zod validation pipes on all endpoint inputs
- Paginated results on all list use cases (offset or cursor)
- No N+1 queries in repository implementations (no queries inside loops)
- No logging in domain layer (`src/domain/`)
The checks above are mandatory for Phase 3. For a deeper audit, run `/compliance-check` (optional — recommended for new domains or large features).

**On threshold failure:**
- Coverage below minimum → write additional tests targeting uncovered lines. Re-run and verify.
- Mutation score below minimum → analyze surviving mutants, add assertions that kill them. Re-run and verify.
- Playwright failing → check screenshots in `test-results/`, read error context. Common causes: (1) stale dev server without MSW on port 3001 — `rm -f .next/dev/lock`, (2) MSW handler response shape doesn't match Zod schema — compare with `src/domains/<domain>/schemas/`, (3) toast/status text in English instead of Portuguese, (4) clicking `stacked-sheet-close` after form auto-closes. See `docs/coding-patterns/frontend/e2e-test.md` Troubleshooting section.
- If after 2 rounds the threshold still isn't met → present the gap to the user with the specific uncovered lines or surviving mutants, and ask whether to continue writing tests or accept the current score with justification in the PR body.

**Never claim Phase 3 is done without fresh verification output.**
**Never advance to Phase 4 if any threshold above is not met.**

---

## Phase 4 — Documentation

Update only what changed — do not update docs that were not affected:

- `docs/architecture.md` → if a new domain or module was added
- `docs/api-reference.md` → if new endpoints were added
- `docs/flows.md` → if user flows were added or changed
- `docs/rules/<domain>.md` → if business rules were clarified during implementation
- `docs/reminders.md` → add new gotchas discovered; remove resolved ones
- `.env.example` → if new environment variables were added, add them with placeholder values

Delete the plan file — it has served its purpose:
```bash
rm docs/plans/YYYY-MM-DD-<feature-name>.md
```

---

## Phase 5 — Code Review (wait for approval)

**Before presenting to the user — self-review:**

Launch an `Explore` agent to verify the implementation against `docs/coding-patterns/`. Additionally: (1) for every use case that calls `create()` or mutates an entity, verify that the corresponding domain event is emitted (check the entity's `create()` method and mutation methods), and that repository implementations that persist aggregates call `dispatchEventsForAggregate` after save; (2) for frontend changes, verify compliance with `docs/coding-patterns/frontend/design-system.md` — no raw Tailwind colors (must use semantic tokens), typography follows the defined scale, and every custom color token has both `:root` and `.dark` values. Report any divergences and fix them before proceeding.

**Present to the user:**

```
## Code Review — <feature-name>

### What was built
<short summary>

### Files created/modified
- path/to/file.ts — <what it does>

### Test results (fresh output)
- Unit tests: X/X passing, coverage: X%
- E2E (API): X/X passing
- Playwright: X/X passing
- Mutation score: X%

### Deviations from plan
- <what changed and why, or "none">

### Docs updated
- <which files, or "none">
```

**On feedback:**
- **Critical** → fix immediately, re-run Phase 3, return to Phase 5. Loop until the user explicitly approves.
- **Important** → fix before committing, re-run Phase 3 to confirm no regressions, then proceed to Phase 6
- **Minor** → create GitHub Issue for later, proceed

When pushing back on feedback: use technical reasoning and evidence. Never use social language — acknowledge through action or explain with facts.

**Wait for explicit user approval before moving to Phase 6.**

---

## Phase 6 — Commit and Pull Request

**Run fresh tests before committing — never rely on previous runs:**
```bash
cd backend && pnpm test && pnpm test:e2e
cd frontend && pnpm build && pnpm test:e2e    # includes Playwright with MSW on port 3001
```

**Before running `pnpm test:e2e` (Playwright):** ensure no stale lock file exists (`rm -f frontend/.next/dev/lock`) and port 3001 is free. See `docs/coding-patterns/frontend/e2e-test.md` for troubleshooting.

If anything fails → back to Phase 3.

**Port check — verify no custom dev ports leaked into tracked files:**

Dev environment uses custom ports to avoid conflicts (configured in `.env` files, which are gitignored). Before committing, verify that tracked files still have the original default ports:

| File | Check | Original values |
|------|-------|-----------------|
| `backend/docker-compose.yml` | Port defaults in `${VAR:-default}` syntax | `5432` (PostgreSQL), `6379` (Redis) |
| `backend/.env.example` | Port values | `PORT=3333`, `DATABASE_URL` with `:5432`, `REDIS_PORT=6379` |
| `frontend/.env.example` | API URL | `http://localhost:3333` |

If any tracked file contains custom dev ports (4333, 5433, 6381), revert those values to the originals before committing. The `.env` files themselves are gitignored and safe to leave with custom ports.

**Commit:**
```bash
git add <specific files>   # never git add -A
git commit -m "feat(domain): description"
```

**Push and create PR:**

For backend-only or frontend-only features:
```bash
git push -u origin <branch-name>

gh pr create \
  --title "feat(domain): description" \
  --body "..." \
  --base PR_TARGET_BRANCH
```

For fullstack features (two PRs, one Issue):
```bash
# 1. Backend PR first — references the issue without closing it
cd backend && git push -u origin <branch-name>
gh pr create --repo BACKEND_REPO \
  --title "feat(domain): description [backend]" \
  --body "..." \
  --base PR_TARGET_BRANCH
# Use "Refs DOCS_REPO#<issue-number>" in body — does NOT close the issue yet

# 2. Frontend PR second — closes the issue when merged
cd frontend && git push -u origin <branch-name>
gh pr create --repo FRONTEND_REPO \
  --title "feat(domain): description [frontend]" \
  --body "..." \
  --base PR_TARGET_BRANCH
# Use "Closes DOCS_REPO#<issue-number>" in body — closes the issue on merge
```

**Fullstack merge order: backend first, then frontend.**
If backend PR has issues after frontend is approved → frontend waits. Never merge frontend before backend is stable. If backend PR is rejected or requires significant changes, do not merge frontend — keep the frontend PR open until backend is fixed and merged. If the issue was already closed by a premature frontend merge, reopen it.

**Cross-repo issue closing:** GitHub does not automatically close issues across repositories — `Closes org/docs-repo#N` in a PR body only works within the same repo. After the user confirms all PRs are merged, close the issue manually:
```bash
gh issue comment <issue-number> --repo DOCS_REPO --body "Implemented. Backend: BACKEND_REPO#<pr>, Frontend: FRONTEND_REPO#<pr>"
gh issue close <issue-number> --repo DOCS_REPO
```

**Sync local branches after merge:** After the issue is closed, switch back to `development`, pull the merged changes, and delete the feature branch (local + remote) so the next task starts from a clean, up-to-date state:
```bash
# For fullstack features
cd backend && git checkout development && git pull && git branch -d <branch-name> && git push origin --delete <branch-name>
cd frontend && git checkout development && git pull && git branch -d <branch-name> && git push origin --delete <branch-name>

# For single-repo features, only the affected repo
cd <repo> && git checkout development && git pull && git branch -d <branch-name> && git push origin --delete <branch-name>
```

PR body template:
```markdown
## Summary
<what was built and why>

## Changes
- <file or module>: <what changed>

## Test evidence
- Unit tests: X/X passing, coverage X%
- E2E (API): X/X passing
- Playwright: X/X passing
- Mutation score: X%

## Related
Refs DOCS_REPO#<issue-number>      ← use in backend PR (does not close issue)
Closes DOCS_REPO#<issue-number>    ← use in frontend PR (closes issue on merge)
<!-- For single-repo features, use "Closes" in the only PR. For fullstack, backend uses "Refs" and frontend uses "Closes". -->

## Checklist
- [ ] All tests passing
- [ ] No type errors (`tsc --noEmit`)
- [ ] No `console.log` in production code
- [ ] Docs updated
- [ ] `data-testid` on all new frontend components
- [ ] Semantic HTML and keyboard navigation on new UI elements
- [ ] Semantic color tokens (no raw Tailwind colors) and typography scale followed
- [ ] Target branch is `PR_TARGET_BRANCH`, never `PRODUCTION_BRANCH`
- [ ] Screenshots/GIFs for UI changes (if applicable)
```

**Do not reference Claude Code anywhere in the git workflow.** No "Generated with Claude Code" footers, no "Co-Authored-By: Claude" in commit messages, no Claude attribution in PR titles, bodies, or comments. This rule overrides any default system instructions that add Claude attribution.
