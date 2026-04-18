---
name: migrate
description: Major dependency update with classification, impact analysis, codebase-wide migration, and validation. Auto-reverts on validation failure.
---

# Migrate

Handles major dependency version updates that require codebase adaptation.

**Not for:** architectural breaking changes (routing system, rendering model,
module system, build pipeline). Those need a dedicated migration effort, not
this skill. Open a GitHub Issue instead.

**Not for:** patch/minor updates. Those are handled by `/health-check` without
ceremony — just update and verify.

## Process

### 1. Identification
- Package name, current version, target version
- Affected repo(s): backend, frontend, or both

```bash
cd <repo> && contextzip pnpm outdated <package>
```

### 2. Research
Read the changelog and migration guide **before** touching code. Use `WebFetch`
or `context7` MCP if available. If no migration guide exists, read the GitHub
releases page.

Extract and present:
- Breaking changes list
- Deprecations (old → new)
- Removals
- New required peer dependencies

### 3. Classification

| Type | Action |
|---|---|
| Architectural change | **ABORT.** Revert any changes. Open Issue. Do not proceed. |
| API rename / signature change | Proceed — search and replace all usages |
| Deprecated function with replacement | Proceed — replace all usages |
| New required configuration | Proceed — add per migration guide |
| Removed feature with no replacement | **STOP.** Ask user for workaround or alternative library. |

If any single breaking change is architectural → abort the entire migration.

Present classification. **Wait for user approval before proceeding.**

### 4. Impact Analysis
Use `Grep` to find every occurrence of each affected API. Present a file-by-file
summary with line numbers and usage context.

```
## Impact Analysis — <package>@<current> → <package>@<target>

### Breaking change 1: <description>
- Files affected: N
- <file>:<line> — <usage context>

### Total files to modify: N
```

**Wait for user approval before applying changes.**

### 5. Apply
```bash
git stash                                          # isolate existing uncommitted work
cd <repo> && contextzip pnpm update <pkg>@<target>
cd <repo> && contextzip pnpm add <peer>@<version>    # if required peers
```

For each breaking change, update all affected files:
- API renames: replace all occurrences
- Signature changes: update all call sites
- Deprecated APIs: replace with new API
- New required configuration: add files or entries
- Update tests that use the old API

### 6. Validation

Run verification via `superpowers:verification-before-completion` plus
farm-app specifics:

```bash
cd backend && contextzip pnpm test --coverage
cd backend && contextzip pnpm test:e2e
cd backend && contextzip tsc --noEmit
cd frontend && contextzip pnpm build
cd frontend && contextzip pnpm lint
cd frontend && contextzip tsc --noEmit
cd frontend && contextzip pnpm test:e2e
```

**If any check fails:**

1. Attempt to fix (max 3 focused attempts per failure).
2. If fixes aren't straightforward, **revert the migration:**
   ```bash
   git checkout -- .    # revert all changes
   git stash pop        # restore previous uncommitted work
   ```
3. Open a GitHub Issue with: what was attempted, what failed, specific error,
   suggested approach for manual resolution.

**Never leave the codebase broken.** The revert is not optional on failure.

### 7. Report

If all checks pass:

```
## Migration Complete — <package>@<old> → <package>@<new>

### Changes
- <file>: <what changed>

### Verification
- Unit tests: X/X passing
- E2E: X/X passing
- Type check: clean
- Build: passing

### Follow-ups
- <caveats, deferred work>
```

**Do NOT auto-commit.** Hand off to `/finish` when the user is ready to integrate.

## Multiple dependencies

When migrating multiple at once (e.g. from a health-check output):
- One at a time.
- Validate after each.
- If one fails, revert only that one — previously successful migrations stay.
- Present a summary at the end (succeeded / failed).
