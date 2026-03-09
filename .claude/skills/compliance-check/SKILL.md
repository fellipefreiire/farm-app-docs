---
name: compliance-check
description: Verifies that a specific file or module follows the coding patterns and architectural rules. Fast, targeted check — not a full audit.
---

# Compliance Check

Verifies that a specific file or module conforms to `docs/coding-patterns/` and the architectural rules in CLAUDE.md. Use this after implementing a feature to catch drift before code review, or when reviewing a PR.

For a full project-wide audit → use `/code-audit`.

---

## Scope

The user specifies what to check:
- A single file: `src/domain/orders/entities/Order.ts`
- A module: `src/domain/orders/`
- A layer: all use cases in `src/application/`

---

## Process

Launch an `Explore` agent (Haiku) to:

1. Read the target file(s)
2. Read the relevant coding pattern files from `docs/coding-patterns/`:
   - Entity → `docs/coding-patterns/backend/entity.md`
   - Use case → `docs/coding-patterns/backend/use-case.md`
   - Repository → `docs/coding-patterns/backend/repository.md`
   - Controller → `docs/coding-patterns/backend/controller.md`
   - Mapper → `docs/coding-patterns/backend/mapper.md`
   - Presenter → `docs/coding-patterns/backend/presenter.md`
   - Events → `docs/coding-patterns/backend/events.md`
   - Tests → `docs/coding-patterns/backend/tests.md`
   - Frontend schema → `docs/coding-patterns/frontend/schema.md`
   - Frontend api → `docs/coding-patterns/frontend/api.md`
   - Frontend action → `docs/coding-patterns/frontend/action.md`
   - Frontend store → `docs/coding-patterns/frontend/store.md`
   - Frontend component → `docs/coding-patterns/frontend/component.md`
   - Frontend page → `docs/coding-patterns/frontend/page.md`
   - Frontend design system → `docs/coding-patterns/frontend/design-system.md`
3. Compare the file against the pattern — check each rule explicitly

---

## Checklist

**Architecture:**
- [ ] File is in the correct layer (domain / application / infrastructure)
- [ ] No layer violations (domain doesn't import infrastructure, etc.)
- [ ] Module boundaries respected (no direct cross-domain imports)

**SOLID principles:**
- [ ] Single Responsibility — file has one reason to change (one use case, one entity, one controller)
- [ ] Dependency Inversion — use cases depend on abstract repository, never on Prisma directly
- [ ] Open/Closed — new behavior via new use cases, not by modifying existing ones

**Domain layer:**
- [ ] Entity has business invariants enforced in the constructor or methods
- [ ] No infrastructure types imported (no Prisma types in entities)
- [ ] Value objects are immutable
- [ ] Domain events are emitted on state changes

**Application layer:**
- [ ] Use case returns Either<DomainError, Result>
- [ ] No direct database calls (only via repository interface)
- [ ] No HTTP types (no Request, Response objects)
- [ ] Single responsibility (one use case per file)

**Infrastructure layer:**
- [ ] Mapper has toDomain and toPrisma methods
- [ ] Controller delegates to use case, never contains business logic
- [ ] Presenter formats output (no raw Prisma objects returned)
- [ ] Zod validation on all inputs

**Tests:**
- [ ] AAA pattern (Arrange, Act, Assert)
- [ ] InMemoryRepository used in unit tests (not real DB)
- [ ] No mocks of domain objects (only infrastructure)
- [ ] Each test has a single assertion focus

**Frontend:**
- [ ] Zod schema defined for API response
- [ ] `data-testid` on all interactive elements
- [ ] Semantic HTML elements used (`button`, `nav`, `main`, `section` — not generic `div` for interactive elements)
- [ ] Form inputs have associated `label` elements
- [ ] No `window`/`localStorage` access outside `useEffect`
- [ ] Server actions handle errors and return typed results
- [ ] Semantic color tokens used (no raw Tailwind colors like `red-500`, `blue-600`)
- [ ] Typography follows the defined scale (no arbitrary font sizes)
- [ ] Dark mode: every custom color has both `:root` and `.dark` values

---

## Architectural Rules Verification

When the scope is a module or layer (not a single file), run these automated checks using `Grep` to catch violations at scale:

**Auth enforcement — all controllers must use `@UseGuards`:**
- Grep `@Controller` in `src/infra/http/controllers/` → list of controller files
- Grep `@UseGuards` in each file found
- Any controller file missing `@UseGuards` is a violation — unless it's explicitly marked `@Public()`

**Either pattern — all use cases must return `Either`:**
- Grep `execute\(` in `src/domain/**/use-cases/` → list of use case files
- Grep `Either` in each file found
- Any use case without `Either` in its return type is a violation

**No console.log in production code:**
- Grep `console\.log` in `src/` — any match is a violation

**DomainEvents dispatch — all repositories with create/save must dispatch:**
- Grep `DomainEvents.dispatchEventsForAggregate` in `src/infra/database/prisma/repositories/`
- Cross-reference with files containing `async create` or `async save`
- Any create/save without dispatch is a violation

**No raw Prisma types in domain layer:**
- Grep `from '@prisma/client'` or `from '.prisma/client'` in `src/domain/` — any match is a violation

**Logging discipline — domain never logs:**
- Grep `LoggerService` or `logger` in `src/domain/` — any match is a violation (domain must not log)

---

## Output

Report findings grouped by severity:

```
## Compliance Check — <file or module>

### Violations
- [CRITICAL] <file>: <rule violated> — <what to fix>
- [IMPORTANT] <file>: <rule violated> — <what to fix>
- [MINOR] <file>: <rule violated> — <what to fix>

### Passing
- <rule>: ✓
- <rule>: ✓

### Summary
X violations found (Y critical, Z important, W minor)
```

If no violations → "All checks pass."

Present to the user. Do not auto-fix — let the user decide whether to fix now or log as issues.

---

## Completion

After presenting the report:
1. If violations found → ask: "Do you want me to fix these violations?"
2. If yes → fix in priority order, verify each fix against the pattern
3. If no → the check is informational only
4. If all checks pass → "All checks pass. Ready to proceed."
