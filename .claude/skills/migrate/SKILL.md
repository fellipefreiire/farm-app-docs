---
name: migrate
description: Guides a major dependency update through classification, impact analysis, codebase-wide migration, and validation. Automatically reverts if validation fails.
---

# Migrate

Handles major version updates for dependencies. Minor and patch updates are done freely during `/health-check` — this skill is for breaking changes that require codebase adaptation.

**Trigger:** Called manually by the user, or suggested by `/health-check` when a major update is detected and classified as non-architectural.

---

## When NOT to Use

- **Architectural breaking changes** (routing system overhaul, module system change, build pipeline change) — these should NOT be migrated through this skill. Create a GitHub Issue instead and plan a dedicated migration effort.
- **Patch/minor updates** — just update and run Phase 3. No skill needed.

---

## Process

### Phase 0 — Identification

1. Identify the dependency, current version, and target version
2. Identify which repo is affected (backend, frontend, or both)

```bash
cd <repo> && pnpm outdated <package-name>
```

### Phase 1 — Research

Read the changelog and migration guide before touching any code.

```bash
# Fetch the changelog or migration guide
```

Use `WebFetch` to read the official migration guide if available. If no migration guide exists, read the GitHub releases page.

**Extract and present to the user:**
- What changed (breaking changes list)
- What was deprecated and replaced
- What was removed entirely
- Any new peer dependencies required

### Phase 2 — Classification

Classify each breaking change:

| Type | Action |
|------|--------|
| **Architectural change** (routing, rendering, module system, build pipeline) | **Stop.** Revert any changes. Create GitHub Issue with full context. Do not proceed. |
| **API rename or signature change** (`func.exec()` → `func.init()`) | Proceed — search and replace all usages |
| **Deprecated function with replacement** | Proceed — replace all deprecated usages with new API |
| **New required configuration** | Proceed — add configuration following the migration guide |
| **Removed feature with no replacement** | **Stop.** Ask user how to handle — may need a workaround or alternative library. |

**Present the classification to the user and wait for approval before proceeding.**

If any single breaking change is architectural → abort the entire migration.

### Phase 3 — Impact Analysis

Search the entire codebase for all usages of the affected APIs:

```bash
# For each breaking change, find all files that use the old API
```

Use `Grep` to find every occurrence. Present a summary:

```
## Impact Analysis — <package>@<current> → <package>@<target>

### Breaking change 1: <description>
- Files affected: N
- <file1>:<line> — <usage context>
- <file2>:<line> — <usage context>

### Breaking change 2: <description>
- Files affected: N
- ...

### Total files to modify: N
```

**Wait for user approval before applying changes.**

### Phase 4 — Migration

1. **Create a snapshot** (commit current state if there are uncommitted changes):
   ```bash
   git stash   # if needed — to enable clean revert
   ```

2. **Update the dependency:**
   ```bash
   cd <repo> && pnpm update <package-name>@<target-version>
   ```

3. **Update peer dependencies** if the migration guide requires them:
   ```bash
   cd <repo> && pnpm add <peer-dep>@<version>
   ```

4. **Apply code changes** — for each breaking change, update all affected files:
   - API renames: find and replace all occurrences
   - Signature changes: update all call sites
   - Deprecated APIs: replace with new API
   - New configuration: add required config files or entries

5. **Update tests** — if test code uses the old API, update it too

### Phase 5 — Validation

Run full Phase 3 validation (from Shared Phases in CLAUDE.md):

```bash
# Backend
cd backend && pnpm test --coverage
cd backend && pnpm test:e2e
cd backend && tsc --noEmit

# Frontend
cd frontend && pnpm build
cd frontend && pnpm lint
cd frontend && tsc --noEmit
cd frontend && pnpm test:e2e
```

**If any validation fails:**

1. Attempt to fix the issue (max 3 attempts per failure)
2. If the fix is not straightforward → **revert the entire migration:**
   ```bash
   git checkout -- .    # revert all changes
   git stash pop        # restore stashed changes if any
   ```
3. Create a GitHub Issue documenting:
   - What was attempted
   - What failed
   - The specific error
   - Suggested approach for manual resolution

**Never leave the codebase in a broken state after a failed migration.**

### Phase 6 — Completion

If all validations pass:

1. Present results to the user:
   ```
   ## Migration Complete — <package>@<current> → <package>@<target>

   ### Changes applied
   - <file>: <what changed>

   ### Test results
   - Unit tests: X/X passing
   - E2E: X/X passing
   - Type check: clean
   - Build: passing

   ### Notes
   - <any caveats or follow-up items>
   ```

2. Wait for user approval
3. Commit with message: `chore(deps): migrate <package> from <old> to <new>`
4. Do NOT create a PR — the migration is typically part of a larger branch or done standalone

---

## Multiple Dependencies

If migrating multiple dependencies at once (e.g., from `/health-check` output):

- Migrate **one dependency at a time**
- Validate after each migration
- If one fails, revert only that one — previous successful migrations remain
- Present a summary at the end showing which succeeded and which failed

---

## Completion

After migration is complete:
1. If migrating from `/health-check`, return to the health-check report to address remaining items
2. If standalone, ask the user if they want to commit or continue with other work
3. Migrations do **not** increment the `health-check-counter`
