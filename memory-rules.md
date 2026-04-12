# Project Memory Rules — farm-app

Rules for what goes into `claude-mem` at the **project level** (farm-app-scoped,
stored under the project's memory namespace). These rules complement (do not
replace) the user-level rules in `~/.claude/projects/.../memory/MEMORY.md`.

## Governing principle

**Canonical docs are the source of truth. Memory is navigation + context, never
compression of canonical content.**

Every memory entry either (a) points at a canonical location where the real
answer lives, or (b) captures knowledge that has no canonical home (decisions,
discoveries, hard-learned lessons). Nothing in between.

---

## What goes in memory ✅

### 1. Dated architectural decisions
Format: date + decision + reason + link to canonical.
```
2026-03-20 — FieldTicket absorbed ScheduleOperation entity.
Reason: ScheduleOperation had 1:1 relationship with FieldTicket,
no independent lifecycle, added indirection with no value.
Canonical: docs/architecture.md#scheduling
Status: active as of <last verification date>
```

### 2. Discovery findings that don't yet live in docs
When `/domain-discovery` or `/flow-discovery` produces a finding that is
still being refined and hasn't been committed to `docs/rules/<domain>.md`
or `docs/flows.md` yet.

### 3. Hard-learned lessons from production incidents
Bug classes that bit the project and would bite again without a guardrail.
```
2026-02-14 — getDate() in UTC-stored dates caused off-by-one in GMT-3.
Rule: all API dates in frontend must use getUTC*() methods.
Canonical: docs/coding-patterns/frontend/date-handling.md
```

### 4. "Why not X" notes for paths not taken
When a reasonable-looking alternative was considered and rejected, with the
rejection reason. Prevents the next AI session from proposing the same thing.
```
2026-03-05 — Considered: splitting Inventory into Compras + Saídas + Ajustes subdomains.
Rejected: UI labels differ but code paths converge; splitting would duplicate
validation without removing duplication. Kept as Inventory with UI relabeling only.
```

### 5. Terminology bridges
When the UI says one thing and the code says another and the mapping is
not obvious. (Pointer, not the mapping itself — the mapping lives in glossary.)
```
Inventory UI ↔ code terminology: see docs/glossary.md#inventory-terms
```

---

## What does NOT go in memory ❌

1. **Coding patterns in any form.** Canonical files in `docs/coding-patterns/**`
   are the source. Don't compress them into memory — they change, memory will
   go stale, the model will act on stale rules.

2. **Business rules.** `docs/rules/<domain>.md` is canonical. Memory may link
   to it, never summarize it.

3. **Session state** ("I was working on feature X, phase 2 done"). This is
   **context-mode**'s job (SQLite session restore).

4. **Ephemeral task details** (current branch, active PR, in-flight test failures).
   These belong in `git`, `docs/plans/<feature>.md`, or the current conversation.

5. **Conversation history.** `claude-mem` already indexes conversations
   automatically via `mcp-search`. No need to duplicate.

6. **Architecture.md content.** If it fits in architecture.md, put it there,
   not in memory.

7. **"Reminders" about how to work.** Those are feedback, which is user-level
   memory, not project-level.

---

## Reciprocity rule

**Every project memory entry carries a link to its canonical home.** If the
canonical home doesn't exist yet, the entry is marked `canonical: pending`
and the next `/domain-discovery` / `/flow-discovery` run must promote it.

Before the model acts on a memory entry:
1. Read the canonical link.
2. Verify the memory still reflects canonical truth.
3. If they diverge: **canonical wins**, update or delete the memory entry.
4. Only then use the information.

This is the only defense against staleness.

---

## Write discipline

When a skill or conversation produces a new piece of knowledge, the model asks:

1. **Does it have a canonical home?** → Write to the canonical file first.
2. **Is it a decision / discovery / lesson / rejection?** → Write to memory
   with a link to the canonical (or `pending`).
3. **Is it ephemeral?** → Do not write to memory.

If the answer to (1) is yes, memory is **optional** and only useful if the
model would need to find this entry faster than it would find the canonical
file (rare — the canonical file is usually findable).

---

## Read discipline

When starting a session or a skill, the model:

1. Loads user-level memory (auto, already happens).
2. Loads project-level memory **only for entries tagged with domains / topics
   relevant to the current task**. No "dump everything" reads.
3. For each relevant entry, verifies against canonical before acting.

---

## Maintenance

- A memory entry with `canonical: pending` that stays pending for > 14 days
  is reviewed: either promoted to canonical doc, or deleted.
- `/health-check` (the slimmed skill) includes a memory-audit pass: list any
  project memory entries whose canonical link is broken or whose date is > 90 days
  old and ask the user to review.
- Never delete a memory entry automatically. Flag, ask, confirm, then delete.
