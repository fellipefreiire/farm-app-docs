---
name: commit
description: Opus-only commit skill. Updates documentation, runs final tests, commits, pushes, and optionally opens a PR. Called explicitly by the user — never triggered automatically.
---

# Commit

Opus-only. The final step of any feature cycle. Updates docs, verifies everything passes, commits, pushes, and opens a PR if configured.

**Only run this skill when explicitly requested by the user.** Never trigger automatically after validation.

---

## Step 0 — Read configuration

Read `docs/CLAUDE.md` configuration block. Show:
- `USE_GITHUB_ISSUES` value
- `PR_TARGET_BRANCH` value
- `PRODUCTION_BRANCH` value
- Current branch: `git branch --show-current` in affected repos

Read the plan file if provided (e.g., `/commit docs/plans/2026-04-06-feature-name.md`). If not provided, ask which feature is being committed.

---

## Step 1 — Documentation update (Phase 4)

Update only the docs that were actually affected by this implementation:

- `docs/architecture.md` → if a new domain or module was added
- `docs/api-reference.md` → if new endpoints were added
- `docs/flows.md` → if user flows were added or changed
- `docs/rules/<domain>.md` → if business rules were clarified during implementation
- `docs/reminders.md` → add new gotchas discovered; remove resolved ones
- `.env.example` → if new environment variables were added, add with placeholder values

Do not update docs that were not affected.

Delete the plan file after docs are updated:
```bash
rm docs/plans/YYYY-MM-DD-<feature-name>.md
```

---

## Step 2 — Final tests

Run fresh tests. Do not rely on previous runs.

```bash
cd backend && pnpm test && pnpm test:e2e
cd frontend && pnpm build && pnpm test:e2e
```

**Port check — verify no custom dev ports leaked into tracked files:**

| File | Check | Must NOT contain |
|------|-------|-----------------|
| `backend/docker-compose.yml` | Port defaults | `5433`, `6381` |
| `backend/.env.example` | Port values | `4333`, `5433`, `6381` |
| `frontend/.env.example` | API URL | `http://localhost:4333` |

If any tracked file contains custom dev ports → revert those values before committing.

If tests fail → stop. Report to user. Do not commit broken code.

---

## Step 3 — Commit

Stage specific files only. Never `git add -A` or `git add .`.

```bash
git add <specific files>
git commit -m "<type>(<domain>): <description>"
```

Commit type:
- `feat` — new feature
- `fix` — bug fix
- `refactor` — refactor
- `chore` — dependency update, config change

Do not add any Claude attribution to commit messages.

---

## Step 4 — Push and PR

```bash
git push -u origin <branch-name>
```

**If `USE_GITHUB_ISSUES = false`:** push only. Do not create PR automatically. Ask user: "Pushed. Do you want me to open a PR?"

**If `USE_GITHUB_ISSUES = true`:** create PR after push:

For backend-only or frontend-only:
```bash
gh pr create \
  --title "<type>(<domain>): <description>" \
  --body "..." \
  --base PR_TARGET_BRANCH
```

For fullstack (two PRs):
```bash
# Backend PR first
cd backend && git push -u origin <branch-name>
gh pr create --repo BACKEND_REPO \
  --title "<type>(<domain>): <description> [backend]" \
  --body "Refs DOCS_REPO#<issue-number>" \
  --base PR_TARGET_BRANCH

# Frontend PR second
cd frontend && git push -u origin <branch-name>
gh pr create --repo FRONTEND_REPO \
  --title "<type>(<domain>): <description> [frontend]" \
  --body "Closes DOCS_REPO#<issue-number>" \
  --base PR_TARGET_BRANCH
```

**Merge order: backend first, then frontend.**

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
Refs DOCS_REPO#<issue-number>    ← backend PR
Closes DOCS_REPO#<issue-number>  ← frontend PR

## Checklist
- [ ] All tests passing
- [ ] No type errors (tsc --noEmit)
- [ ] No console.log in production code
- [ ] Docs updated
- [ ] data-testid on all new frontend components
- [ ] Semantic HTML and keyboard navigation on new UI elements
- [ ] Semantic color tokens (no raw Tailwind colors)
- [ ] Target branch is PR_TARGET_BRANCH, never PRODUCTION_BRANCH
```

Do not reference Claude Code anywhere in the git workflow.

---

## Step 5 — Increment health-check counter

After PR is successfully created (or push confirmed if no PR):

Read `docs/reminders.md`, find the `health-check-counter` line, increment by 1, write back.

```
health-check-counter: N+1
```

Only increment after successful push. If push fails, do not increment.

Bug fixes and refactors do **not** increment the counter — skip this step for those.

---

## Completion

```
✓ /commit complete
Branch: <branch-name>
Commit: <commit hash>
PR: <PR URL> (or "pushed, no PR created")
health-check-counter: N+1
```
