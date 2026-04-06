---
name: implement
description: Local model implementation skill. Reads the plan from docs/plans/ and implements it following the spec exactly. Never makes architectural decisions. Stops and reports if anything is unclear.
---

# Implement

Local model only. Reads a plan written by Opus and implements it step by step.

**This skill never makes architectural decisions.** If anything in the plan is missing, ambiguous, or conflicts with a coding pattern — stop immediately and report. Do not decide on your own.

---

## Step 0 — Resolve plan

If a plan path was provided (e.g., `/implement docs/plans/2026-04-06-feature-name.md`):
- Read that file

If no path was provided:
- List available plans: contents of `docs/plans/` excluding `SPEC_TEMPLATE.md`
- Ask the user which plan to implement

Read the plan completely before doing anything else.

---

## Step 1 — Read coding patterns

Read ONLY the files listed in the plan's **"Patterns to read"** section. Do not read any other documentation.

After reading each file, confirm briefly:
> "Read [file]: key constraint for this feature is [X]."

If the plan's "Patterns to read" section is empty or missing → stop:
> "⚠️ Plan is missing the 'Patterns to read' section. Send this plan back to Opus for correction."

---

## Step 2 — Validate plan completeness

Before writing any code, verify the plan has:
- [ ] Exact TypeScript contracts (interfaces, function signatures)
- [ ] Complete file list (create / modify / do not touch)
- [ ] Numbered implementation sequence
- [ ] Test scenarios with input/output
- [ ] Restrictions section

**If any item is missing:**
```
⚠️ PLAN GAP DETECTED
Missing: [what is missing]
Cannot proceed. Returning to Opus for correction.
```

Update `docs/plans/<plan-file>.md` — append to the bottom:
```markdown
## ⚠️ Implementation Blocked

**Gap detected in Step 2 validation:**
- [description of what is missing]

Returned to Opus for correction.
```

Then stop. Do not proceed.

---

## Step 3 — Implementation

Follow the implementation sequence in the plan exactly. Do not skip steps. Do not reorder steps.

**TDD is mandatory for every artifact:**

1. Write the failing test (RED) — show the test output confirming it fails
2. Write the minimum code to make it pass (GREEN)
3. Refactor if needed

**Do not write implementation code before showing failing test output.** This is a hard requirement.

After completing each wave or major step, update the progress section in the plan file:

```markdown
## Implementation Progress

✓ Step 1 — [description] (completed)
✓ Step 2 — [description] (completed)
→ Step 3 — [description] (in progress)
```

**If stuck on the same issue after 3 attempts:**

Stop immediately. Do not attempt a 4th fix. Update the plan file:

```markdown
## ⚠️ Implementation Blocked

**Stuck at:** Step N — [description]
**Problem:** [exact error or issue]
**Attempts:**
1. [what was tried and result]
2. [what was tried and result]
3. [what was tried and result]

Returning to Opus for diagnosis.
```

Then emit:
```
⚠️ BLOCKED after 3 attempts at [step description]
Error details written to docs/plans/<plan-file>.md
Send this file to Opus for diagnosis before continuing.
```

**If plan conflicts with a coding pattern:**
```
⚠️ CONFLICT DETECTED
Plan says: [X]
docs/coding-patterns/[file] requires: [Y]
Cannot resolve this without architectural decision.
Send to Opus for resolution.
```

Update plan file with the conflict details and stop.

---

## Step 4 — Prisma (if applicable)

If the plan includes schema changes, run after the schema is updated:

```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev
```

If migration fails → stop and report to user. Do not proceed without a working migration.

---

## Step 5 — Verification

Run the tests defined in the plan:

```bash
# Backend
cd backend && pnpm test --testPathPattern=<relevant-spec>
cd backend && pnpm test --coverage

# Frontend (if applicable)
cd frontend && pnpm build
cd frontend && pnpm lint
cd frontend && tsc --noEmit
```

Show the full output. Do not summarize — show actual test results.

Update the progress section:

```markdown
## Implementation Progress

✓ Step 1 — entity and value objects
✓ Step 2 — use case
✓ Step 3 — repository and mapper
✓ Step 4 — controller and presenter
✓ Step 5 — frontend slice
✓ Verification — X/X tests passing, coverage X%
```

---

## ⏸ Handoff

After verification passes:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /implement complete
Completed: all steps implemented, tests passing
Plan file: docs/plans/<plan-file>.md (progress recorded)
Next step: in the Opus terminal, run /validate docs/plans/<plan-file>.md
Waiting for: confirmation that validation is complete
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
