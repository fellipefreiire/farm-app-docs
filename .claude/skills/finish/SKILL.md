---
name: finish
description: Prepare current branch for merge/PR. Runs farm-app doc checklist, delegates verification and git operations to superpowers. Explicit user invocation only — never auto-triggered.
---

# Finish

Called explicitly by the user when current work is ready to integrate.
**Never invoked automatically** by other skills. **Never commits, pushes, or
opens a PR without explicit user confirmation at each step.**

## Hard rules

1. No auto-commit
2. No auto-push
3. No auto-PR
4. Every git operation requires explicit user confirmation
5. The farm-app doc checklist (Step 2) is **mandatory** — never skipped

## Process

### 1. Show current state

```bash
contextzip git status
contextzip git diff --stat
contextzip git log main..HEAD --oneline
```

Present:
- Current branch
- Files modified / added / deleted
- Commits ahead of `main` / `development`

### 2. Farm-app doc checklist (MANDATORY)

For each item below, ask explicitly: **"Does this need updating? If yes, edit now."**

| Change in work | Review |
|---|---|
| Controller endpoint added/changed | Verify `@nestjs/swagger` decorators (`@ApiOperation`, `@ApiResponse`, DTO schemas). Swagger runtime at `/api-docs` is the canonical API reference — no docs file to update. |
| New or changed user/system flow | `docs/flows/<domain>.md` (per-domain file) |
| New or changed domain rule | `docs/rules/<domain>.md` |
| Architectural decision (any significant choice) | Write a new ADR in `docs/adr/YYYY-MM-DD-<slug>.md` and link from `_index.md`. Never edit an existing ADR — mark it superseded if it is replaced. |
| Architectural overview needs update (new domain, changed relationships) | `docs/architecture.md` |
| New coding pattern or pattern modified | `docs/coding-patterns/**` |
| Glossary term added/changed | `docs/glossary.md` |
| Decision worth remembering across sessions | project-level `claude-mem` (see `docs/memory-rules.md`) — every entry needs a `canonical:` link |

If any doc needs updating, edit it **now, before proceeding**. Do not defer to
"a later PR". Past sessions caused stale docs by deferring — this is a hard rule.

Once edits are done, run `contextzip git status` to confirm the doc changes show up.

### 3. Verification

Invoke `superpowers:verification-before-completion`. It runs the full quality
gate (tests, type check, lint, build). If anything fails, **STOP** — fix the
failure or mark the work as not ready and return to implementation.

Do not proceed to Step 4 with any failing check.

### 4. Delegate git flow to superpowers

Invoke `superpowers:finishing-a-development-branch`. That skill presents
integration options (merge, PR, cleanup). It does not auto-execute — the user
picks.

Farm-app-specific constraints to pass through to superpowers:
- **PR target:** `development` (never `main`)
- **Issue requirement:** `USE_GITHUB_ISSUES=true` — every PR needs a linked
  GitHub Issue in the docs repo
- **Branch naming:** `BE-{n}/name`, `FE-{n}/name`, `FS-{n}/name`, `hotfix/name`
- **Commit messages:** Conventional Commits (`feat(scope): description`).
  No Claude attribution in commit body.

After the user picks an option and superpowers executes it, report what was done:

```
## Finished

Branch: <branch>
Action: <merge / PR / stash / etc.>
Target: <development / main>
PR URL: <if created>
Commits: <list>
Docs updated: <list or "none">
```

## When not to invoke

- **Mid-implementation:** continue implementing first.
- **Failed tests:** fix before invoking. `/finish` will stop at Step 3 anyway.
- **Open questions:** resolve with the user before starting integration.
- **Already integrated:** if the work is already in `development`, there is
  nothing to finish.
