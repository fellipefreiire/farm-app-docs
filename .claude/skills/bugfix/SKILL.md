---
name: bugfix
description: Diagnoses and fixes a bug. Focuses on root cause analysis, never patches symptoms. Requires a failing test before the fix.
---

# Bug Fix

Diagnoses and fixes a bug in any layer. The fix must be proven by a test — either an existing failing test or a new one written to reproduce the bug before fixing it.

---

## Phase 0 — Diagnosis

**Never assume the cause. Read the evidence first.**

1. Read the full error output, stack trace, or bug description from the issue
2. Read in parallel:
   - `docs/reminders.md` — check for known gotchas related to this area
   - The affected files (controller, use case, entity, frontend component)
3. Reproduce the bug consistently before attempting any fix:
   - Run the relevant test or E2E flow and confirm the failure
4. Identify the root cause — trace the error to its origin, not just where it surfaces
5. Present the diagnosis to the user: root cause, affected files, proposed fix

**If the root cause is unclear after reading:** ask the user for more context before proceeding.

---

## Branch

```bash
cd backend && git checkout development && git pull && git checkout -b BE-{issue-number}/fix-{description}
# or frontend / fullstack as needed
```

**If branch creation fails:** check the error — if the branch already exists (`git checkout` to it), if the working tree is dirty (`git stash` or ask the user). Do not proceed to Fix without a clean branch.

---

## Fix

**Write the failing test first** (if none exists):
- Unit test for the specific scenario that triggered the bug
- E2E test if the bug is at the API boundary

Then fix the root cause. One change at a time.

If the fix requires a Prisma schema change:
```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev
```

---

## Validation

```bash
# Confirm the specific bug test now passes
cd backend && pnpm test --testPathPattern=<relevant-spec>

# Run full suite to check for regressions
cd backend && pnpm test --coverage
cd backend && tsc --noEmit

# Frontend if affected
cd frontend && pnpm build && pnpm lint && tsc --noEmit
```

Run E2E if the bug was at the API or UI boundary:
```bash
cd backend && pnpm test:e2e
cd frontend && pnpm test:e2e
```

**Systematic debugging rules:**
1. Read the full error — never assume the cause
2. Reproduce consistently before fixing
3. Trace to root cause — never patch symptoms
4. Fix one thing at a time, verify after each change
5. If 3 fixes fail on the same issue → stop, raise with user

---

## Regression Check

After the fix is verified, search the codebase for the same pattern that caused the bug. Use `Grep` to find similar code in other files.

If the same pattern exists elsewhere, report to the user:
> "The same pattern that caused this bug exists in N other file(s): [list]. Do you want me to fix those too or create a GitHub Issue for later?"

- **If the user chooses to fix:** apply the same fix to all affected files, re-run the full test suite to verify no regressions, then proceed to Code Review. Include all fixed files in the review summary. If tests pass for the original fix but fail for some additional files, ask the user: "The original fix is solid, but the additional fix in `<file>` has test failures. Do you want to investigate that separately or revert those additional fixes?"
- **If the user chooses to create an issue:** create a GitHub Issue in DOCS_REPO describing the pattern and affected files, then proceed to Code Review with only the original fix.

If no similar patterns are found, proceed to Code Review.

---

## Code Review

Present to the user:

```
## Bug Fix — <description>

### Root cause
<one paragraph: what was wrong and why>

### Fix
<what was changed>

### Test added
- <test file and scenario>

### Test results
- Bug test: passing
- Full suite: X/X passing
- No regressions
```

**Wait for explicit user approval before committing.**

---

## Commit and Pull Request

```bash
git add <specific files>
git commit -m "fix(<domain>): <description>"
```

```bash
gh issue comment <issue-number> --repo DOCS_REPO --body "Fixed. Root cause: <description>"
gh issue close <issue-number> --repo DOCS_REPO
```

```bash
git push -u origin <branch-name>
gh pr create \
  --title "fix(<domain>): <description>" \
  --body "..." \
  --base PR_TARGET_BRANCH \
  --repo BACKEND_REPO  # or FRONTEND_REPO
```

PR body template:
```markdown
## Summary
<root cause and what was fixed>

## Changes
- <file or module>: <what changed>

## Test evidence
- Bug test: passing
- Full suite: X/X passing, coverage X%
- No regressions

## Related
Closes DOCS_REPO#<issue-number>

## Checklist
- [ ] All tests passing
- [ ] No type errors (tsc --noEmit)
- [ ] No console.log in production code
- [ ] Target branch is PR_TARGET_BRANCH, never PRODUCTION_BRANCH
```

---

## Completion

After PR is created:
1. Confirm the PR URL with the user
2. Add the gotcha to `docs/reminders.md` if the bug was caused by a non-obvious pattern that could recur
3. Bug fixes do **not** increment the `health-check-counter`
