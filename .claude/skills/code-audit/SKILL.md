---
name: code-audit
description: Full project-wide audit of architecture, code quality, security, and test coverage. Produces a prioritized report of issues.
---

# Code Audit

Performs a comprehensive, project-wide review of code quality, architectural conformance, security, and test coverage. Produces a prioritized report for the user to act on.

For a targeted single-file check → use `/compliance-check`.
For a full health check including dependencies and metrics → use `/health-check`.

---

## Scope

Covers both backend and frontend:
- Architectural conformance (layer violations, module boundaries)
- Coding pattern conformance (entities, use cases, controllers, etc.)
- Security (OWASP Top 10, input validation, auth coverage)
- Test quality (coverage, mutation score, AAA pattern)
- Code quality (cyclomatic complexity, dead code, console.logs)
- Frontend safety (SSR, data-testid, Zod schemas)

---

## Process

Launch parallel `Explore` agents (Haiku) to scan:

**Agent 1 — Backend architecture:**
- Check all files in `src/domain/` for layer violations, entity patterns, and use case patterns
- Check `src/infra/database/` for mapper and repository patterns
- Check `src/infra/http/` for controller and presenter patterns

**Agent 2 — Backend security:**
- Verify every HTTP endpoint has auth guard (except explicitly public)
- Verify every endpoint has Zod validation pipe
- Search for `console.log`, hardcoded credentials, unhandled promises
- Check `pnpm audit` output

**Agent 3 — Frontend:**
- Verify Zod schemas exist for all API responses
- Search for missing `data-testid` on interactive elements
- Search for `window`/`localStorage` outside `useEffect`
- Check `pnpm audit` output

**Agent 4 — Tests:**
- Run `pnpm test --coverage` and parse coverage report
- Identify files below threshold (domain+application < 100%, infra < 80%)
- Check for tests missing AAA pattern or using real DB in unit tests

---

## Output

```
## Code Audit Report — <date>

### Critical (must fix before next feature)
- [ARCH] src/application/orders/use-cases/CreateOrder.ts imports PrismaClient directly — layer violation
- [SEC] POST /payments has no auth guard
- [SEC] src/infra/http/controllers/UserController.ts — missing Zod validation on body

### Important (fix soon, add to backlog)
- [COV] src/domain/inventory/ — coverage 72%, below 100% threshold
- [TEST] test/use-cases/CreateOrder.spec.ts — no AAA structure, mixed assertions
- [FE] src/domains/orders/components/OrderForm.tsx — submit button missing data-testid

### Minor (log as issue, fix when convenient)
- [QUAL] src/application/reports/ — cyclomatic complexity > 10 in GenerateReport use case
- [FE] src/domains/users/components/UserCard.tsx — uses inline style instead of Tailwind class

### Metrics
| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
| Backend coverage (domain+app) | 87% | 100% | ❌ |
| Backend coverage (infra) | 83% | 80% | ✅ |
| Backend coverage (overall) | 85% | 85% | ✅ |
| Frontend coverage | 76% | 80% | ❌ |
| Security audit (backend) | 0 critical CVEs | 0 | ✅ |
| Security audit (frontend) | 2 high CVEs | 0 | ❌ |

### Recommended actions
1. Fix critical violations immediately
2. Open GitHub Issues for important items
3. Schedule minor items in backlog
```

Present the report to the user. Do not auto-fix — the user decides what to address and when.

---

## Completion

After presenting the report:
1. Ask: "Do you want me to fix any of these issues now?"
2. If yes → fix in priority order (critical first), re-run affected tests after each fix
3. Open GitHub Issues for items not immediately fixed
4. If no → the report is informational only
