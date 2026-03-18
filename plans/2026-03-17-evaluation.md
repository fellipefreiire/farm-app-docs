# Boilerplate Evaluation Report — 2026-03-17

## Overall score: 8.4/10

| # | Dimension | Score | Issues |
|---|-----------|-------|--------|
| 1 | Internal coherence | 8/10 | 0 critical, 3 important, 3 minor |
| 2 | Coding patterns completeness | 9/10 | 0 critical, 0 important, 3 minor |
| 3 | Skills coverage | 8/10 | 0 critical, 2 important, 4 minor |
| 4 | Template quality | 8/10 | 0 critical, 2 important, 5 minor |
| 5 | Security (settings.json) | 8/10 | 1 critical, 0 important, 1 minor |
| 6 | End-to-end flow | 9/10 | 0 critical, 1 important, 0 minor |
| 7 | Maintainability | 8/10 | 1 critical, 3 important, 3 minor |

---

## Critical (must fix)

- ~~[DIM-5] `docs/.claude/settings.json:56,76` — `rm -rf .next` in allow list is overridden by `rm -rf *` in deny list, making the allow rule ineffective~~ **FIXED** — Removed `Bash(rm -rf .next)` from allow list
- ~~[DIM-7] MEMORY.md line 25 — States "Redis 6380" but actual `.env` and `shared-phases.md` both use `6381`~~ **FIXED** — Updated MEMORY.md to say `6381`

## Important (should fix)

- ~~[DIM-1] `docs/coding-patterns/backend/use-case.md` — Rule 8 in CLAUDE.md says "cross-module" but all supporting docs (query-bus.md, shared-phases.md) use "cross-domain"~~ **FIXED** — Standardized CLAUDE.md Rule 8 to "cross-domain"
- ~~[DIM-1] `docs/shared-phases.md:75` — `AuditLogRepository` exemption mentioned in Phase 3 but not documented in `query-bus.md`~~ **FALSE POSITIVE** — Already documented at query-bus.md line 379 in module boundary rules table
- ~~[DIM-1] `docs/.claude/skills/small-change/SKILL.md` — Phase 2 does not mention `data-testid` requirement for frontend components~~ **FIXED** — Added `data-testid` requirement to Phase 2
- ~~[DIM-3] `docs/.claude/skills/bugfix/SKILL.md`, `docs/.claude/skills/refactor/SKILL.md` — Validation steps overlap Phase 3 but don't reference `shared-phases.md`~~ **FIXED** — Added note referencing shared-phases.md Phase 3 thresholds
- ~~[DIM-3] CLAUDE.md Entry Point — `/project-setup` is triggered by Step 0 but not listed in Step 3 routing table~~ **FIXED** — Added Project Setup row
- ~~[DIM-4] `docs/flows.md` — No instructions on how to document a new flow~~ **FIXED** — Added "How to add a new flow" preamble with format and guidelines
- ~~[DIM-4] `docs/api-reference.md` — No Swagger URL reference for full schema details~~ **FIXED** — Added `http://localhost:4333/api` reference (using custom dev port)
- ~~[DIM-6] `docs/.claude/skills/small-change/SKILL.md:85` — E2E test guidance "when in doubt, run E2E" contradicts the skip list~~ **FIXED** — Rewritten to make run-by-default explicit, skip only when strictly limited to unit-testable changes
- ~~[DIM-7] CLAUDE.md and docs/CLAUDE.md — Two identical 435-line files, changes must be synced manually~~ **FALSE POSITIVE** — Root `CLAUDE.md` is already a symlink to `docs/CLAUDE.md`
- [DIM-7] Branch creation instructions duplicated in 5+ skill files → **DEFERRED** — Each skill has contextual variations (branch prefix, description). Extracting adds indirection for minimal gain (2-3 lines).
- [DIM-7] Prisma migration commands duplicated in 3+ places → **DEFERRED** — Same reasoning; commands are short and contextual

## Minor (nice to have)

- ~~[DIM-1] `docs/coding-patterns/backend/events.md:3` — Opening paragraph doesn't use "fire-and-forget" terminology~~ **FIXED** — Added "fire-and-forget" to opening line
- ~~[DIM-1] `docs/shared-phases.md:111` — Phase 5 mentions `dispatchEventsForAggregate` but no coding pattern documents this method~~ **FALSE POSITIVE** — Already documented in repository.md (9 occurrences), events.md, and tests.md
- [DIM-1] No skill references CLAUDE.md "Dependency Management" section → **SKIPPED** — Very low impact; dependency management is triggered by `/health-check` and `/migrate`, not implementation skills
- ~~[DIM-2] `docs/coding-patterns/frontend/accessibility.md` — Uses "What NOT to do" instead of "Anti-patterns" section title~~ **FIXED** — Renamed to "Anti-patterns"
- [DIM-2] `docs/coding-patterns/frontend/audit-integration.md` — Minimal detail compared to other patterns → **ADEQUATE** — File is 129 lines with checklist, rules, examples, and pending improvements. Comprehensive for its scope.
- ~~[DIM-2] `docs/coding-patterns/backend/mutation-testing.md` — No explicit "Anti-patterns" section~~ **FALSE POSITIVE** — Anti-patterns section exists at line 187
- [DIM-3] `/feature`, `/new-domain`, `/small-change` — Model recommendations not linked to CLAUDE.md Model Selection table → **SKIPPED** — Model selection table is in CLAUDE.md which is always loaded; adding references in each skill adds noise for minimal value
- [DIM-3] Health-check counter gate not mentioned in skill frontmatter descriptions → **SKIPPED** — Frontmatter descriptions are for skill discovery; health-check gating is an operational detail documented in Phase 0 of each skill
- [DIM-3] "Phase 0 → 6" vs "Phases 3-6" terminology varies across skills → **NOT AN ISSUE** — "Phase 0 → 6" describes the full feature workflow; "Phases 3-6" describes the shared portion. Both are accurate in context.
- ~~[DIM-3] Evaluate skill Agent 3 is self-referential~~ **FIXED** — Added clarifying note that self-evaluation is intentional
- ~~[DIM-4] `docs/shared-phases.md` — Phase 4 and 5 lack "How Claude uses this phase" preamble~~ **FIXED** — Added preambles to both phases
- ~~[DIM-4] `docs/glossary.md` — System timing logic leaking into user-facing doc~~ **FALSE POSITIVE** — The "For Claude" section is legitimate guidance for Claude's glossary usage, not system timing logic
- ~~[DIM-4] `docs/product.md` — No "Why this matters" preamble~~ **FIXED** — Added preamble explaining the document's foundational role
- [DIM-5] `docs/.claude/settings.json:27` — `git checkout *` in allow is broader than needed → **ADEQUATE** — Dangerous variants (`git checkout .`, `git checkout -- .`) are explicitly denied. The broad allow covers legitimate branch switching.
- ~~[DIM-7] Large files without TOC: tests.md (1468 lines), controller.md (756 lines), page.md (732 lines)~~ **FIXED** — Added table of contents to all three files
- ~~[DIM-7] Port check table in shared-phases.md missing explicit custom→original mapping~~ **FIXED** — Added "Custom dev values (must NOT appear)" column

---

## Strengths

- All 11 Non-Negotiable Rules are enforced by at least one skill or pattern
- All 9 Entry Point tracks map to existing skills with proper transitions
- 35 coding pattern files cover every artifact type in the project structure
- QueryBus pattern well-documented with cross-references to architecture.md, use-case.md, events.md, tests.md, and shared-phases.md
- Phase gating is robust: Phase 3 gates 4, Phase 5 gates 6, with explicit failure handling
- Security deny list covers all critical destructive commands
- Health-check counter mechanism properly integrated across implementation skills
- Pinned dependency versions table prevents false positives in future health-checks
- No dead ends found in any workflow path

## Comparison with previous evaluation

No previous evaluation report found in `docs/plans/`.
