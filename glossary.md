# Glossary

Project-specific terms and their definitions. This file ensures consistent language between the development team and Claude across all conversations.

**Why this matters:** When the team uses domain-specific terms (e.g. "batch", "tenant", "workflow"), Claude needs to understand exactly what they mean in the context of this project. Without a glossary, Claude may assume wrong meanings or ask repeatedly about the same terms.

---

## How to use

### For the developer

Add terms as the project evolves. Each term should have:
- **Term** — the word or phrase as used in conversations and code
- **Definition** — what it means in this project (1-2 sentences)
- **Context** *(optional)* — where it appears (code, UI, business rule) or an example

```markdown
### Batch
A group of products created together in a single production run. Each batch has a unique code and tracks quality metrics.
- **Context:** `Batch` entity in the `production` domain. UI shows batch list with status filters.
```

### For Claude

- **During setup:** After `product.md` and `architecture.md` are ready, prompt the developer to fill this glossary with key terms from those documents.
- **During development:** When you encounter an unfamiliar project-specific term in conversation, code, or docs — do not assume its meaning. Ask the developer for context, then add it here.
- **During code review (Phase 5):** Verify that variable names, entity names, and comments use terms consistent with this glossary. Flag any new domain term that appears in code or conversation but is not listed here.
- **When adding terms:** Use the format above. Keep definitions concise and project-specific.

---

## Terms

### Domain
A bounded context with its own entities, use cases, and repository. The primary organizational unit of the codebase (e.g., `orders`, `billing`, `users`).
- **Context:** Maps to a folder under `src/domain/<domain>/` in backend and `src/domains/<domain>/` in frontend.

### Module (NestJS)
A NestJS `@Module({})` class that wires controllers, providers, and imports. One domain typically has one or more NestJS modules.
- **Context:** Files like `<domain>-controllers.module.ts`, `<domain>-database.module.ts`, `<domain>-events.module.ts`.

### Module (architecture)
Synonym for "domain" in architectural discussions. Prefer "domain" for clarity.
- **Context:** Used in CLAUDE.md rules (e.g., "module boundaries", "cross-module communication"). Refers to domains, not NestJS modules.

<!-- Add project-specific terms below. -->
<!-- Example:

### Tenant
An organization that owns a workspace in the platform. Each tenant has isolated data and its own set of users.
- **Context:** `Tenant` entity in the `organization` domain. All queries are scoped by `tenantId`.

### Credit
A unit of consumption that limits how many operations a user can perform. Credits are purchased in packs and deducted per action.
- **Context:** `Credit` value object in the `billing` domain. Displayed in the dashboard header.

-->
