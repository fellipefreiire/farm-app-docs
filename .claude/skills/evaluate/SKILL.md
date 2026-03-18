---
name: evaluate
description: Evaluates the boilerplate framework itself — CLAUDE.md, skills, coding patterns, docs, and settings. Produces a scored report with findings and offers to fix issues.
---

# Evaluate

Performs a comprehensive evaluation of the boilerplate framework: rules, skills, coding patterns, doc templates, security settings, and overall flow coherence. This is a **meta-audit** — it evaluates the tools that guide development, not the project code itself.

For code quality audits → use `/code-audit`.
For project health (tests, deps, coverage) → use `/health-check`.

---

## When to Run

- After making changes to the boilerplate (new skills, new patterns, rule changes)
- Before adapting the boilerplate to a new project
- On demand by the user
- Periodically to catch drift or inconsistencies

---

## Process

Launch 7 `Explore` agents in parallel (Haiku — conformance checks).

**Mandatory verification rule for ALL agents:** Before reporting any finding, the agent MUST read the actual file and confirm the issue exists. Never report a finding based on assumption or naming convention alone (e.g., don't flag "logger.md missing" without checking if `logging.md` exists). If a finding cannot be confirmed by reading the file, discard it. False positives undermine the evaluation — precision is more important than recall.

**Agent 1 — Internal Coherence:**

Check that rules in CLAUDE.md are enforced by the skills and patterns:

1. Extract all Non-Negotiable Rules from CLAUDE.md (the numbered table)
2. For each implementation skill (`/feature`, `/new-domain`, `/small-change`): verify the skill references or enforces each rule. Flag rules that no skill checks.
3. Extract all Anti-Hallucination rules from CLAUDE.md — **note:** these are behavioral guidelines enforced by Claude's execution behavior, not by skill checks or automated verification. Do not flag them as unenforced — they are procedural by nature. Only flag if a skill or pattern *contradicts* an anti-hallucination rule.
4. For each coding pattern file: check if it contradicts any CLAUDE.md rule
5. Check that `docs/shared-phases.md` is consistent with references in CLAUDE.md and skill files
6. Verify terminology is consistent across all files (same terms for same concepts)

Score criteria:
- 10: Zero contradictions, all rules enforced by at least one skill
- 8-9: Minor gaps (a rule is implied but not explicitly referenced)
- 6-7: Some rules have no enforcement point
- <6: Contradictions found between files

**Agent 2 — Coding Patterns Completeness:**

1. Read the project structure defined in CLAUDE.md (`backend/src/` and `frontend/src/` sections)
2. List every artifact type mentioned (entity, use-case, repository, controller, mapper, presenter, events, tests, action, api, component, page, schema, store, shared)
3. For each artifact type, check if a corresponding file exists in `docs/coding-patterns/backend/` or `docs/coding-patterns/frontend/`
4. For each coding pattern file, verify it contains: purpose description, code template, rules/constraints, "what NOT to do" section
5. Check if any coding pattern references a dependency or method that should be verified against `package.json`. **Note:** In a boilerplate (before project setup), `package.json` does not exist yet — dependency verification is the responsibility of `/project-setup` and `/health-check`, not individual patterns. Do not flag this as a gap in a boilerplate context.

Score criteria:
- 10: Every artifact type has a pattern, all patterns have complete sections
- 8-9: All critical patterns exist, minor sections missing
- 6-7: 1-2 artifact types without patterns
- <6: Multiple artifact types without patterns

**Agent 3 — Skills Coverage:**

1. Read the Entry Point routing table (Step 2 → Step 3) in CLAUDE.md
2. For each track, verify the mapped skill(s) exist in `docs/.claude/skills/`
3. For each skill file, verify it has: `name` and `description` in frontmatter, clear phases/process, explicit completion criteria
4. For each implementation skill (`/feature`, `/new-domain`, `/small-change`): verify it references the Shared Phases (Phase 3→6)
5. Check for orphan skills (exist in the directory but are not referenced by any routing table or other skill)
6. Verify the model recommendations in skills match the Model Selection table in CLAUDE.md

> **Note:** Agent 3 evaluates all skills including `/evaluate` itself. This is intentional — the evaluate skill should also follow the structural standards it checks in others.

Score criteria:
- 10: Every track has skills, all skills are well-structured, no orphans
- 8-9: All tracks covered, minor structural gaps in skill files
- 6-7: A track is missing a skill, or a skill has incomplete phases
- <6: Multiple tracks without skills

**Agent 4 — Template Quality:**

Evaluate each doc template in `docs/`:
- `product.md`
- `architecture.md`
- `flows.md`
- `glossary.md`
- `api-reference.md`
- `reminders.md`
- `shared-phases.md`

For each template, check:
1. Has a purpose description (why this doc matters)
2. Has instructions for how to fill it
3. Has at least one example (commented or inline)
4. Has a "How Claude uses this document" section (or equivalent)
5. Sections are sufficient for the doc's purpose (no obvious gaps)
6. Cross-references to other docs are correct (linked files exist)

**Note:** Templates in a boilerplate are expected to have empty or placeholder content sections. Only flag missing structure (sections, instructions, format examples), not missing project-specific content. Commented-out examples count as valid examples — they show the format without polluting a clean project with fake data.

Score criteria:
- 10: All templates have all 6 qualities
- 8-9: Most templates complete, 1-2 missing a section
- 6-7: Several templates missing instructions or examples
- <6: Templates are stubs without guidance

**Agent 5 — Security (settings.json):**

1. Read `docs/.claude/settings.json`
2. Check the deny list against known dangerous commands:
   - `git reset --hard`, `git push --force`, `git push -f`, `git rebase`, `git clean`
   - `git checkout .`, `git checkout -- .`, `git restore .`
   - `rm -rf`, `rm -r`
   - `prisma migrate reset`
   - Any command that could delete data or overwrite history
3. Check the allow list for overly broad patterns (e.g., `Bash(*)` would be too permissive). **Exceptions (intentionally broad):** `Read(*)`, `Write(*)`, `Edit(*)`, `Glob(*)`, `Grep(*)` are file operation tools that need project-wide access. `Bash(pnpm *)` and `Bash(git push origin *)` are broad but their dangerous variants are explicitly denied — this is acceptable when deny rules provide coverage.
4. Verify that the allow list doesn't accidentally override a deny list entry
5. Check if there are dangerous commands that should be in the deny list but aren't

Score criteria:
- 10: All known dangerous commands blocked, allow list is scoped, no overrides
- 8-9: Minor gap (a rare dangerous command not blocked)
- 6-7: A common dangerous command is not blocked
- <6: Allow list is too broad or deny list has critical gaps

**Agent 6 — End-to-End Flow:**

Simulate the full workflow for each track to find gaps:

1. **New project:** Entry Point Step 0 (product.md missing) → `/project-setup` → Step 1 (load context) → Step 2 (classify). Verify every step has explicit instructions for what to do and what to do when things go wrong.

2. **Feature track:** User says "add X" → Step 2 classifies as Feature → Step 3 invokes `/feature` → Phase 0 (refinement) → Branch → Phase 1 (planning) → Phase 2 (implementation) → Phase 3 (validation) → Phase 4 (docs) → Phase 5 (review) → Phase 6 (commit/PR). Check for:
   - Dead ends (a phase says "if X → ?" with no answer)
   - Missing error handling (what if a phase fails?)
   - Unclear transitions (when exactly does one phase end and the next begin?)

3. **Bug fix track:** User says "bug in X" → `/bugfix` → diagnosis → branch → fix → validation → regression check → review → commit. Same checks.

4. **Cross-skill transitions:** After `/domain-discovery` completes, does it clearly hand off to `/new-domain`? After `/health-check` finds a major dep update, does it hand off to `/migrate`?

Score criteria:
- 10: Every path has clear transitions, error handling, and no dead ends
- 8-9: Minor gaps in error handling for edge cases
- 6-7: A common path has an unclear transition
- <6: Dead ends found in main paths

**Agent 7 — Maintainability:**

1. Measure file sizes — flag any file over 300 lines as potentially needing extraction. **Exception:** coding pattern files (`docs/coding-patterns/`) may exceed 300 lines when they cover multiple variants of the same artifact (e.g., repository with interface + Prisma + in-memory + pagination + transactions). Do not flag these if the content is cohesive and covers a single artifact type.
2. Search for duplicated text across files (same paragraph appearing in CLAUDE.md and a skill file, or in multiple skill files)
3. Verify all cross-file references are valid (if a file says "see docs/shared-phases.md", that file must exist)
4. Check for stale references (mentions of files, sections, or concepts that no longer exist)
5. Check if the overall file count and structure is manageable — is it easy to understand where things live?

**Note:** Empty doc templates (e.g., `flows.md`, `glossary.md` with only commented examples) are intentional in a boilerplate — they are ready-to-fill, not incomplete. Do not flag these as gaps.

Score criteria:
- 10: No large files, no duplication, all references valid, clean structure
- 8-9: Minor duplication (intentional for context), all references valid
- 6-7: Some stale references or significant duplication
- <6: Large files that need extraction, broken references

---

## Output

Save the report to `docs/plans/YYYY-MM-DD-evaluation.md` AND present it in the chat.

```markdown
# Boilerplate Evaluation Report — YYYY-MM-DD

## Overall score: X.X/10

| # | Dimension | Score | Issues |
|---|-----------|-------|--------|
| 1 | Internal coherence | X/10 | N critical, N important, N minor |
| 2 | Coding patterns completeness | X/10 | N critical, N important, N minor |
| 3 | Skills coverage | X/10 | N critical, N important, N minor |
| 4 | Template quality | X/10 | N critical, N important, N minor |
| 5 | Security (settings.json) | X/10 | N critical, N important, N minor |
| 6 | End-to-end flow | X/10 | N critical, N important, N minor |
| 7 | Maintainability | X/10 | N critical, N important, N minor |

---

## Critical (must fix)
- [DIM-N] <file>:<line> — <what is wrong> → <suggested fix>

## Important (should fix)
- [DIM-N] <file>:<line> — <what is wrong> → <suggested fix>

## Minor (nice to have)
- [DIM-N] <file>:<line> — <what is wrong> → <suggested fix>

## Strengths
- <what is working well>

## Comparison with previous evaluation
- <if a previous report exists in docs/plans/, compare scores and note improvements or regressions>
```

Each finding uses the format `[DIM-N]` where N is the dimension number (1-7) for traceability.

---

## Completion

Present the report to the user, then ask:

> "I found N issues (X critical, Y important, Z minor). Do you want me to fix them?"

Options:
- **Fix all** — fix critical and important immediately, create GitHub Issues for minor
- **Fix critical only** — fix critical, create Issues for important and minor
- **Don't fix** — the report is informational only
- **Cherry-pick** — user selects which findings to fix

For each fix:
1. Apply the change
2. Verify it doesn't break anything (read affected files, check cross-references)
3. Present the change to the user before moving to the next

After all fixes are applied, delete the evaluation file from `docs/plans/` — it has served its purpose.

If no fixes are needed (all scores ≥ 9), congratulate and delete the file.

If the overall score is below 5/10 or more than 5 critical issues are found, warn the user:
> "The boilerplate has significant issues that could impact development quality. I recommend fixing all critical items before starting any implementation work."
