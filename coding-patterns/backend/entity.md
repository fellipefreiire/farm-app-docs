# Entity Pattern

Entities are the core of the domain layer. They encapsulate identity, state, and business invariants. They must never import infrastructure types.

---

## File location

```
src/domain/<domain>/enterprise/entities/<Entity>.ts
```

---

## Base classes (src/core/)

The project uses three base classes. Choose the right one:

| Class | When to use |
|-------|------------|
| `Entity<Props>` | Simple entity with identity but no domain events |
| `AggregateRoot<Props>` | Entity that emits domain events (most entities) |
| `ValueObject<Props>` | Immutable concept with no identity (Email, Money, CPF) |

```ts
// src/core/entities/entity.ts
export abstract class Entity<Props> {
  private _id: UniqueEntityID
  protected props: Props

  get id() { return this._id }

  protected constructor(props: Props, id?: UniqueEntityID) {
    this.props = props
    this._id = id ?? new UniqueEntityID()
  }

  public equals(entity: Entity<unknown>) {
    return entity === this || entity.id === this._id
  }
}

// src/core/entities/aggregate-root.ts
export abstract class AggregateRoot<Props> extends Entity<Props> {
  private _domainEvents: DomainEvent[] = []

  get domainEvents(): DomainEvent[] { return this._domainEvents }

  protected addDomainEvent(domainEvent: DomainEvent): void {
    this._domainEvents.push(domainEvent)
    DomainEvents.markAggregateForDispatch(this)
  }

  public clearEvents(): void { this._domainEvents = [] }
}
```

---

## Structure

```ts
import { AggregateRoot } from '@/core/entities/aggregate-root'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import type { Optional } from '@/core/types/optional'
import { <Entity>CreatedEvent } from '../events/<entity>-created-event'
import { <Entity>UpdatedEvent } from '../events/<entity>-updated-event'
import { <Entity>ActiveStatusChangedEvent } from '../events/<entity>-active-status-changed-event'
import { <Entity>DeletedEvent } from '../events/<entity>-deleted-event'

export type <Entity>Props = {
  field: Type
  optionalField?: Type
  <related>Id: UniqueEntityID  // FK — always UniqueEntityID, never raw string
  active: boolean
  createdAt: Date
  updatedAt?: Date
  deletedAt?: Date    // present only if domain supports soft delete
}

export class <Entity> extends AggregateRoot<<Entity>Props> {
  // --- Getters (one per prop) ---

  get field(): Type {
    return this.props.field
  }

  get active(): boolean {
    return this.props.active
  }

  get createdAt(): Date {
    return this.props.createdAt
  }

  get updatedAt(): Date | undefined {
    return this.props.updatedAt
  }

  // --- Business methods ---

  update(data: { field: Type; optionalField?: Type }, actorId: string): void {
    const previousData = {
      field: this.props.field,
      optionalField: this.props.optionalField,
    }

    this.props.field = data.field
    this.props.optionalField = data.optionalField
    this.touch()

    this.addDomainEvent(new <Entity>UpdatedEvent(this, actorId, previousData))
  }

  toggleActive(actorId: string): void {
    this.props.active = !this.props.active
    this.touch()
    this.addDomainEvent(new <Entity>ActiveStatusChangedEvent(this, actorId))
  }

  softDelete(actorId: string): void {
    this.props.deletedAt = new Date()
    this.touch()
    this.addDomainEvent(new <Entity>DeletedEvent(this, actorId))
  }

  // --- Private helpers ---

  private touch(): void {
    this.props.updatedAt = new Date()
  }

  // --- Factory (use case) ---

  static create(
    props: Optional<<Entity>Props, 'createdAt' | 'active'>,
    actorId: string,
    id?: UniqueEntityID,
  ): <Entity> {
    // id parameter is for pre-generated IDs (e.g. WatchedList children), NOT for reconstitution
    const now = new Date()

    const entity = new <Entity>(
      {
        ...props,
        active: props.active ?? true,
        createdAt: props.createdAt ?? now,
        updatedAt: props.updatedAt ?? undefined,
      },
      id,
    )

    entity.addDomainEvent(new <Entity>CreatedEvent(entity, actorId))

    return entity
  }

  // --- Reconstitution (mapper only) ---

  static reconstitute(
    props: <Entity>Props,
    id: UniqueEntityID,
  ): <Entity> {
    return new <Entity>(props, id)
  }
}
```

---

## Rules

- Extend `AggregateRoot<Props>` if the entity emits domain events (most cases)
- Extend `Entity<Props>` only if the entity never emits events
- All props accessed through **getters only** — `props` is protected, never exposed
- `create()` is used by use cases — always requires `actorId`, always emits `<Entity>CreatedEvent`
- `reconstitute()` is used by mappers only — no `actorId`, never emits events
- Business mutations emit domain events with `actorId` and `previousData` for audit trail
- Always call `touch()` on any mutation to update `updatedAt`
- Always use `type` for Props — never `interface` (prevents declaration merging and accidental mutation of typings)
- **Never import** Prisma, NestJS, or any infrastructure type
- Use `UniqueEntityID` for IDs — never raw strings

---

## Value Objects

For attributes with their own validation rules, extract a Value Object:

```ts
// src/domain/<domain>/enterprise/value-objects/<ValueObject>.ts
import { ValueObject } from '@/core/entities/value-object'

export type <ValueObject>Props = {
  value: string
}

export class <ValueObject> extends ValueObject<<ValueObject>Props> {
  get value(): string {
    return this.props.value
  }

  private constructor(props: <ValueObject>Props) {
    super(props)
  }

  static create(value: string): <ValueObject> {
    if (/* invalid */) {
      throw new Error('<reason>')
    }
    return new <ValueObject>({ value: value.trim() })
  }
}
```

Value objects are **immutable** — no setters, no mutation methods. Equality is structural (by value), not by identity.

---

## Optional utility

Used in `create()` to make certain props optional (system-provided defaults):

```ts
// src/core/types/optional.ts
export type Optional<T, K extends keyof T> = Pick<Partial<T>, K> & Omit<T, K>
```

---

## Multi-entity domains

When a domain has more than one entity, model the relationship with a **join entity** and a **WatchedList**.

**Join entity** — represents the relationship, extends `Entity<Props>` (not AggregateRoot):

```ts
// src/domain/<domain>/enterprise/entities/<Parent><Child>.ts
import { Entity } from '@/core/entities/entity'
import type { UniqueEntityID } from '@/core/entities/unique-entity-id'

export type <Parent><Child>Props = {
  <parent>Id: UniqueEntityID
  <child>Id: UniqueEntityID
}

export class <Parent><Child> extends Entity<<Parent><Child>Props> {
  get <parent>Id() { return this.props.<parent>Id }
  get <child>Id() { return this.props.<child>Id }

  static create(props: <Parent><Child>Props, id?: UniqueEntityID): <Parent><Child> {
    return new <Parent><Child>(props, id)
  }
}
```

**WatchedList** — tracks additions and removals without replacing the full collection.

> The WatchedList is parameterized over the **join entity** (e.g. `ProductCategory`), not the target entity (e.g. `Category`). The join entity holds FKs to both parent and target, and is what gets created/deleted in the join table.

```ts
// src/domain/<domain>/enterprise/entities/<Parent><Child>List.ts
import { WatchedList } from '@/core/entities/watched-list'
import type { <Child> } from './<child>'

// <Child> here refers to the join entity (e.g. ProductCategory), not the target entity (e.g. Category)
export class <Parent><Child>List extends WatchedList<<Child>> {
  compareItems(a: <Child>, b: <Child>): boolean {
    return a.id.equals(b.id)
  }
}
```

**Aggregate root holds the WatchedList as a prop:**

```ts
export type <Parent>Props = {
  name: string
  <children>: <Parent><Child>List   // ← WatchedList, not a plain array
  // ...
}

export class <Parent> extends AggregateRoot<<Parent>Props> {
  get <children>() { return this.props.<children> }

  static create(
    props: Optional<<Parent>Props, 'createdAt' | '<children>'>,
    actorId: string,
    id?: UniqueEntityID,
  ): <Parent> {
    const entity = new <Parent>({
      ...props,
      <children>: props.<children> ?? new <Parent><Child>List(),
      createdAt: props.createdAt ?? new Date(),
      updatedAt: props.updatedAt ?? undefined,
    }, id)

    entity.addDomainEvent(new <Parent>CreatedEvent(entity, actorId))

    return entity
  }
}
```

**Rules:**
- The join entity extends `Entity`, not `AggregateRoot` — it has no domain events
- The WatchedList tracks `getNewItems()` and `getRemovedItems()` — the repository uses these to upsert/delete efficiently
- Related entities are always accessed through the aggregate root — never fetched independently by a use case that already has the aggregate
- `compareItems()` should compare by identity (`a.id.equals(b.id)`) unless structural equality is needed
- **Join entities never have FK setters** — the parent ID must be pre-generated with `new UniqueEntityID()` in the use case and passed to children at creation time

---

## Delete patterns

Three distinct concepts — which apply to a domain must be defined in `docs/rules/<domain>.md`:

| Pattern | When | Field | Use case behavior |
|---------|------|-------|-------------------|
| **Hard delete** | No domain interactions exist | — | `repository.delete(id)` — permanent removal, no domain event emitted |
| **Soft delete** | Domain interactions exist (orders, references) | `deletedAt?: Date` | `entity.softDelete(actorId)` + `repository.save(entity)` |
| **Toggle active** | Entity can be disabled and re-enabled | `active: boolean` | `entity.toggleActive(actorId)` + `repository.save(entity)` |

**Hard delete vs soft delete decision** belongs in the delete use case, informed by domain rules. The use case checks for domain-specific interactions (external references, child records from other domains) and decides:
- No interactions → hard delete
- Has interactions → soft delete

What counts as "an interaction" is domain-specific and must be documented in `docs/rules/<domain>.md`.

**Soft-deleted entities** are excluded from all `list()` queries by default. `findById()` returns the entity regardless of soft-delete status (needed for delete and audit). Use `findActiveById()` for standard reads that should exclude soft-deleted records.

**Hard delete and audit:** hard deletes are intentionally not audited via domain events — the entity is permanently removed and no side effects are needed. If audit is required for all deletions, the domain should use soft delete exclusively.

**Cascade on delete:** own children (e.g. prices, stock) are deleted together with the parent via `onDelete: Cascade` in the Prisma schema — no application code needed. External references (e.g. orders referencing a product) block deletion and return a domain error checked in the use case.

---

## Anti-patterns

```ts
// ❌ plain class with no base
class Order {
  id: string
  status: string
}

// ❌ infrastructure import in entity
import { PrismaClient } from '@/generated/prisma/client'

// ❌ mutating props directly from outside
order.props.status = 'cancelled'

// ❌ no domain event on state change
update(data) {
  this.props.name = data.name
  // missing: this.addDomainEvent(...)
}

// ❌ using create() in mapper (emits spurious event)
static toDomain(raw) {
  return <Entity>.create(props, 'system', new UniqueEntityID(raw.id)) // use reconstitute()
}

// ❌ using reconstitute() in use case (event never fires)
const entity = <Entity>.reconstitute(props, new UniqueEntityID()) // use create(props, actorId)

// ❌ FK setter on join entity
setProductId(id: UniqueEntityID) {
  this.props.productId = id  // pre-generate parent ID in use case instead
}
```
