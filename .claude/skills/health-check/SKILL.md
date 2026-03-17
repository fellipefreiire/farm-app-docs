---
name: health-check
description: Full project health evaluation — tests, coverage, dependencies, architecture, documentation gaps. Run on demand or when health-check-counter reaches HEALTH_CHECK_THRESHOLD.
---

# Health Check

Comprehensive evaluation of project health across all dimensions: tests, coverage, mutation score, dependencies, architecture, security, and documentation. Produces a prioritized report and resets the `health-check-counter`.

---

## When to Run

- When `health-check-counter` in `docs/reminders.md` reaches `HEALTH_CHECK_THRESHOLD`
- Before starting a new domain (suggested)
- On demand by the user

---

## Process

Launch all agents in parallel (background):

**Agent 1 — Tests and coverage (Bash, Haiku):**
```bash
cd backend && pnpm test --coverage
cd backend && pnpm test:e2e
cd frontend && pnpm build
cd frontend && pnpm test:e2e
```

**Agent 2 — Type check (Bash, Haiku):**
```bash
cd backend && tsc --noEmit
cd frontend && tsc --noEmit
```

**Agent 3 — Dependencies (Bash, Haiku):**
```bash
cd backend && pnpm outdated
cd backend && pnpm audit
cd frontend && pnpm outdated
cd frontend && pnpm audit
```

**Agent 4 — Architecture conformance (Explore, Haiku):**
- Scan `src/domain/`, `src/application/`, `src/infra/` for layer violations
- Check module boundaries (no direct cross-domain imports)
- Verify all endpoints have auth guards and Zod validation

**Agent 5 — Documentation gaps (Explore, Haiku):**
- Check `docs/rules/` — is there a rules file for every domain in the codebase?
- Check `docs/api-reference.md` — does it cover all endpoints?
- Check `docs/flows.md` — are all major user flows documented?
- Check `docs/reminders.md` — are there unresolved items?

---

## Dependency Update Protocol

For each outdated dependency found:

**Patch / Minor versions:**
- Update freely: `pnpm update <package>`
- Run Phase 3 validation. If passing → done.

**Major versions:**
1. **Check `docs/reminders.md` → "Pinned dependency versions" table first.** If the package is listed there, do NOT flag it as outdated in the report — instead, re-evaluate the "Re-evaluate when" column to check if the blocker was resolved. If resolved, proceed with the update and remove the entry from the table. If still blocked, skip silently.
2. Read the changelog and migration guide before updating anything
3. Classify the breaking change:
   - **Architectural change** (routing system, rendering model, module system) → **do not update**. Open a GitHub Issue to plan a dedicated migration. When ready, use `/migrate` to execute. **Add an entry to the "Pinned dependency versions" table** in `docs/reminders.md` with the reason and re-evaluation condition.
   - **Peer dependency conflict** → **do not update**. **Add an entry to the pinned table** with the blocking peer and when to re-check.
   - **LTS mismatch** (e.g., `@types/node` ahead of Node LTS) → **do not update**. **Add an entry to the pinned table** with the LTS cycle date.
   - **API rename/signature change** → update all usages, run Phase 3
   - **Deprecated function with replacement** → replace all usages, run Phase 3
4. If in doubt whether a change is architectural → do not update, open issue, add to pinned table

---

## Output

```
## Health Check Report — <date>

### Test Results
| Suite | Result | Coverage |
|-------|--------|----------|
| Backend unit | X/X passing | domain+app: X%, infra: X%, overall: X% |
| Backend E2E | X/X passing | — |
| Frontend build | passing/failing | — |
| Frontend E2E (Playwright) | X/X passing | — |
| Type check (backend) | 0 errors / X errors | — |
| Type check (frontend) | 0 errors / X errors | — |

### Security
| Repo | Critical CVEs | High CVEs |
|------|--------------|-----------|
| Backend | 0 | 0 |
| Frontend | 0 | 0 |

### Outdated Dependencies
| Package | Current | Latest | Type | Action |
|---------|---------|--------|------|--------|
| <name> | X.Y.Z | A.B.C | patch | updated |
| <name> | X.Y.Z | A.B.C | major | issue opened |

### Pinned Dependencies (re-evaluated)
| Package | Pinned at | Latest | Still blocked? | Reason |
|---------|-----------|--------|----------------|--------|
| <name> | X.Y.Z | A.B.C | yes/no | <blocker status> |
<!-- Re-evaluate each entry from docs/reminders.md "Pinned dependency versions" table. If unblocked, update and remove from pinned table. -->

### Architecture
- [ ] Layer violations found: <list or "none">
- [ ] Module boundary violations: <list or "none">
- [ ] Endpoints without auth: <list or "none">
- [ ] Endpoints without validation: <list or "none">

### Documentation Gaps
- [ ] Domains without rules file: <list or "none">
- [ ] Endpoints not in api-reference.md: <list or "none">
- [ ] Flows not in flows.md: <list or "none">

### Issues Found
**Critical** (block next feature):
- <issue>

**Important** (add to backlog):
- <issue>

**Minor** (fix when convenient):
- <issue>

### Verdict
HEALTHY — project is in good shape, ready to continue.
NEEDS ATTENTION — address critical items before next feature.
```

---

## Completion

After presenting the report:

1. Open GitHub Issues for any important or critical items not immediately fixed
2. Reset `health-check-counter` in `docs/reminders.md` (reset happens after the report, regardless of whether fixes are applied — the counter tracks when to run the check, not fix completion):
   ```
   health-check-counter: 0
   ```
3. If major dependency updates were classified as non-architectural, suggest: "There are N major updates ready for migration. Do you want me to run `/migrate` for any of them?" If the user agrees, run `/migrate` once per dependency sequentially. After each migration, ask whether to continue with the next or stop.
4. If critical issues were found, recommend fixing them first: "There are N critical issues. I recommend addressing them before starting the next feature."
5. Ask the user: "Health check complete. Proceed with the next feature?"
