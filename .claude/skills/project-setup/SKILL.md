---
name: project-setup
description: One-time bootstrap for a new farm-app-style project. Fills base docs through guided conversation before any other work begins.
---

# Project Setup

Runs once, at the start of a new farm-app project. Populates base docs through
a guided conversation.

**Hard gate:** no other skill or implementation action until this skill completes.

## Process

Ask questions **one at a time**. Prefer multiple-choice when possible.

### 1. Configuration
- GitHub org/username
- Project name (used to build repo names: `<org>/<project>-docs`, `-backend`, `-frontend`)
- Package manager (pnpm, npm, yarn, bun — default pnpm)

Update `CLAUDE.md` Configuration section with the values.

### 2. `docs/product.md`
One question at a time:
- What does this project do? (one sentence)
- Who are the users? (roles, personas)
- Main problem solved?
- Key features for v1?
- Known constraints / non-goals?

### 3. `docs/architecture.md`
One question at a time:
- Main domains/modules (e.g. users, orders, inventory)
- How they relate (which communicate with which)
- External services / integrations (email, payment, storage)
- Database structure at a high level (main entities)

Generate a Mermaid diagram:
~~~mermaid
graph TD
  A[Module A] --> B[Module B]
~~~

### 4. `docs/flows/<domain>.md`
One question at a time:
- Most important user flow (step by step)
- Other key flows (authentication, main CRUD, critical business flows)

For each flow, choose the domain it belongs to and write it under `docs/flows/<domain>.md` (creating the file if needed). Use numbered narrative + Mermaid sequence diagram. See `docs/flows/_index.md` for the template and format conventions.

### 5. `docs/glossary.md`
- Domain-specific terms needing definition (business terms, abbreviations,
  Portuguese terms mapped to English code names)

Format:
```
**Term** — definition. Code: `EnglishName`
```

## Recovery

If the conversation is interrupted:
1. Check which base docs already have content: `product.md`, `architecture.md`, `flows/*.md` (any file in the flows folder), `glossary.md`
2. Skip populated sections — do not re-ask.
3. Resume from the first incomplete step.
4. If a doc is partially filled, present its current content and ask: "Continue from here or restart this section?"

## Completion

When all base docs are filled:

1. Present a summary of what was created
2. Ask for confirmation
3. If confirmed → inform the user that normal workflow begins via `superpowers:using-superpowers`
4. If corrections needed → adjust and confirm again

**Do not proceed to normal work until explicit user confirmation.** After
confirmation, if the user has a pending request, hand off to `superpowers:using-superpowers`.
Otherwise ask: "Setup complete. What would you like to build first?"
