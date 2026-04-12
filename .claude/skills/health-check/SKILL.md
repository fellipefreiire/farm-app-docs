---
name: health-check
description: Farm-app project health audit — tests, coverage, dependencies, architecture conformance, documentation gaps, memory hygiene. On-demand only.
---

# Health Check

Farm-app-specific project audit. Produces a prioritized report. **Runs on
demand only** — no auto-trigger, no counter (the previous counter lived in
`docs/reminders.md` which was removed in the harness rewrite).

## Process

Dispatch in parallel via `superpowers:dispatching-parallel-agents`. All shell
commands use `tokenzip` prefix — filters apply automatically.

### Agent 1 — Tests and coverage
```bash
cd backend && tokenzip pnpm test --coverage
cd backend && tokenzip pnpm test:e2e
cd frontend && tokenzip pnpm build
cd frontend && tokenzip pnpm test:e2e
```

### Agent 2 — Type check
```bash
cd backend && tokenzip tsc --noEmit
cd frontend && tokenzip tsc --noEmit
```

### Agent 3 — Dependencies
```bash
cd backend && tokenzip pnpm outdated
cd backend && tokenzip pnpm audit
cd frontend && tokenzip pnpm outdated
cd frontend && tokenzip pnpm audit
```

### Agent 4 — Architecture conformance
Scan `backend/src/domain/`, `backend/src/application/`, `backend/src/infra/`:
- Layer violations (wrong-direction imports)
- Module boundary violations (direct cross-domain imports — should be Domain Events or QueryBus)
- Endpoints without auth guards
- Endpoints without Zod validation

### Agent 5 — Documentation gaps
- `docs/rules/` — one file per domain in `backend/src/domain/`?
- `docs/flows/` — one file per domain in `backend/src/domain/`, with flows documented?
- Backend controllers — every `@Controller` has `@ApiTags` and every endpoint has `@ApiOperation` + `@ApiResponse` decorators (Swagger is canonical — no `api-reference.md` anymore)?
- `docs/coding-patterns/` — any file > 400 lines needing split?
- `docs/adr/` — are there architectural decisions from recent work that were not recorded as an ADR?

### Agent 6 — Memory hygiene (claude-mem)
- Project memory entries with `canonical: pending` older than 14 days
- Project memory entries whose canonical link is broken or stale
- Report findings, do not auto-delete

## Dependency Update Protocol

For each outdated dependency:

**Patch / Minor** — update freely, then run verification.

**Major** — classify first:

| Type | Action |
|------|--------|
| Architectural (routing system, rendering model, module system, build pipeline) | DO NOT update. Open GitHub Issue, add to pinned list. |
| Peer dependency conflict | DO NOT update. Add to pinned list with blocking peer. |
| LTS mismatch (e.g. `@types/node` ahead of Node LTS) | DO NOT update. Add to pinned list with LTS cycle date. |
| API rename / signature change | Proceed via `/migrate`. |
| Deprecated function with replacement | Proceed via `/migrate`. |

**Pinned dependencies** live in project-level `claude-mem`, tagged
`pinned-dependency`. Each entry: package, pinned-at version, reason,
re-evaluate-when. See `docs/memory-rules.md`.

## Output

```
## Health Check Report — <date>

### Tests
| Suite | Result | Coverage |
|---|---|---|
| Backend unit | X/X | domain+app X%, infra X% |
| Backend E2E | X/X | — |
| Frontend build | pass/fail | — |
| Frontend E2E (Playwright) | X/X | — |
| Type check (backend) | 0 / X errors | — |
| Type check (frontend) | 0 / X errors | — |

### Security
| Repo | Critical CVEs | High CVEs |
|---|---|---|
| Backend | 0 | 0 |
| Frontend | 0 | 0 |

### Dependencies
| Package | Current | Latest | Type | Action |
|---|---|---|---|---|

### Pinned dependencies re-evaluated
| Package | Pinned at | Latest | Still blocked? | Blocker |
|---|---|---|---|---|

### Architecture
- Layer violations: <list or "none">
- Module boundary violations: <list or "none">
- Endpoints without auth: <list or "none">
- Endpoints without validation: <list or "none">

### Documentation gaps
- Domains without rules file: <list>
- Endpoints without Swagger decorators: <list>
- Domains without flows file: <list>
- Coding patterns > 400 lines: <list>

### Memory hygiene
- Stale pending entries: <list>
- Broken canonical links: <list>

### Issues
**Critical** (block next feature): <list>
**Important** (add to backlog): <list>
**Minor** (fix when convenient): <list>

### Verdict
HEALTHY / NEEDS ATTENTION
```

## Completion

1. Open GitHub Issues for Critical and Important items not fixed immediately.
2. Report findings to the user.
3. If Critical issues exist: recommend fixing before next feature.
4. Ask: "Proceed with next feature?"
