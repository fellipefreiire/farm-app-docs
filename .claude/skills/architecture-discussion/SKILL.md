---
name: architecture-discussion
description: Facilitates a structured discussion about a significant architectural decision. Documents the outcome as an ADR and updates docs/architecture.md if needed.
---

# Architecture Discussion

Facilitates a structured discussion for significant architectural decisions: choosing a pattern, introducing a new integration, changing module boundaries, selecting a library, or defining a cross-cutting concern.

The output is a documented decision — either an ADR (Architecture Decision Record) or an update to `docs/architecture.md`.

---

## When to Use

- Introducing a new external service or integration
- Changing how modules communicate (direct call vs events vs shared DB)
- Choosing between two implementation approaches with trade-offs
- Defining a cross-cutting concern (auth, caching, logging, multi-tenancy)
- Any decision that affects more than one domain or layer

---

## Process

### 1. Define the Problem

Ask:
- What decision needs to be made?
- What is the current state? (what exists today)
- What is the desired outcome? (what problem are we solving)
- What is the consequence of not deciding now?

### 2. Identify Options

For each option the user or Claude proposes:
- What is it?
- How does it work in this specific context?
- What are the trade-offs? (pros and cons)
- Does it conflict with existing architecture decisions?
- Does it affect future multi-tenancy or scalability?

Use Opus model for this phase — architectural decisions require deep reasoning.

### 3. Evaluate Against Constraints

Check each option against:
- Non-Negotiable Rules in CLAUDE.md (Clean Architecture, SOLID, DDD boundaries)
- `docs/architecture.md` — existing decisions and module relationships
- `docs/rules/<domain>.md` — domain-specific constraints if relevant
- Performance requirements (async I/O, no N+1, caching strategy)
- Security requirements (OWASP, auth, input validation)

### 4. Recommendation

Present a clear recommendation with reasoning:
- Which option and why
- What trade-offs are accepted
- What is deferred or out of scope

Ask the user to confirm or redirect.

---

## Output

Update `docs/architecture.md` directly:

1. Add or update the relevant section (module, relationship, constraint, cross-cutting concern)
2. Update the Mermaid diagram if module relationships changed
3. Add a **Decision Log** entry at the bottom of the file:

```markdown
## Decision Log

### YYYY-MM-DD — <Title>

**Context:** <what problem prompted this decision>
**Decision:** <what was decided>
**Options considered:** <Option A (chosen) / Option B / Option C>
**Rationale:** <why Option A was chosen>
**Consequences:** <what changes as a result>
```

This keeps all architectural knowledge in one file instead of scattered across separate ADR files.

---

## Completion

Present the ADR or architecture update to the user and ask:
1. Does this accurately capture the decision?
2. Are there missing consequences or constraints?

If confirmed → the decision is recorded and implementation can proceed.
