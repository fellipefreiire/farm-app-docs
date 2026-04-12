---
name: domain-discovery
description: Structured conversation to capture a domain's business rules, entities, use cases, and relationships into docs/rules/<domain>.md.
---

# Domain Discovery

Captures everything needed to implement a farm-app domain. Output is
`docs/rules/<domain>.md`, the authoritative reference for that domain.

Run this **before** any implementation skill if the rules file is missing.

## Process

Ask questions **one at a time**. Wait for the answer. Build the rules file
incrementally.

### 1. Domain Overview
- Name (English, singular, PascalCase — used as code name)
- Responsibility in one sentence
- Explicit out-of-scope boundaries

### 2. Entities
For the primary entity and each related one:
- Attributes (name, type, required/optional)
- Business invariants (rules that must always hold, e.g. "price must be positive")
- Identity (UUID system-generated vs externally provided)
- Status / lifecycle (state machine or none)
- Relationships (one-to-many, many-to-many, embedded) — including to other domains

### 3. Value Objects
- Attributes that represent concepts with their own validation
- Each with its validation rules (e.g. Email, CPF, Money, Coordinate)

### 4. Use Cases
For every operation the domain exposes:
- Trigger (user action, domain event, scheduled job)
- Inputs
- Business rules enforced
- Success result + all typed error cases
- Domain events emitted (if any)

### 5. Cross-Domain Relationships
- Domains this one depends on (by ID reference only — never direct imports)
- Domains consuming this one's events
- Cross-domain data reads that should go through QueryBus

### 6. Authorization
- Who can execute each use case (roles, ownership)
- Resource-level restrictions (e.g. "user sees only their own records")

### 7. Edge Cases and Constraints
- Known edge cases
- Uniqueness constraints (e.g. email unique per tenant)
- Soft-delete requirements
- Audit trail requirements

## Output

Write `docs/rules/<domain>.md`:

~~~markdown
# <Domain> — Business Rules

## Responsibility
<one sentence>

## Out of scope
<what this domain does NOT handle>

## Entities

### <Entity>
| Attribute | Type | Required | Rules |
|-----------|------|----------|-------|
| id | UUID | yes | system-generated |
| ... | | | |

**Invariants:**
- <rule>

**Lifecycle:** <states and transitions, or "no state machine">

## Value Objects
- **<ValueObject>** — what it represents, validation rules

## Use Cases

### <ActionEntity> (e.g. CreateOrder)
- **Trigger:** <what initiates>
- **Input:** <fields>
- **Rules:** <business rules enforced>
- **Success:** <what is returned>
- **Errors:** <typed error cases>
- **Events emitted:** <DomainEvent names, or "none">

## Cross-Domain Dependencies
- Depends on: <DomainA> (ID reference only)
- Emits events consumed by: <DomainB>
- Reads from: <DomainC> via QueryBus

## Authorization
- <UseCase>: <who can execute, ownership>

## Constraints
- <uniqueness, soft-delete, audit requirements>

## Edge Cases
- <known edge cases>

## Open Questions
- <anything undefined — resolve before implementation>
~~~

## Completion

Present the filled file to the user. Ask:
1. Is everything correct?
2. Are there open questions to resolve before implementation?

If confirmed → ask: "Rules are ready. Want to start implementation now, or save
for later?" If implementing now, hand off to `superpowers:brainstorming` for
the first feature. If saving, mention the domain name when ready.

Coding patterns for implementation live in `docs/coding-patterns/backend/_index.md`
and `docs/coding-patterns/frontend/_index.md`. They are read per-layer by the
implementing subagent, not loaded upfront.
