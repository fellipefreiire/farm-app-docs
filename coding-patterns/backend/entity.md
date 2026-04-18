# Entity Pattern

Entities = core of domain layer. Encapsulate identity, state, business invariants. Never import infrastructure types.

---

## File location

```
src/domain/<domain>/enterprise/entities/<Entity>.ts
```

> **Multi-entity domains:** When domain uses subdomain folders (see `domain-organization.md`), entities move under subdomain: `src/domain/<domain>/<subdomain>/enterprise/entities/<Entity>.ts`.

---

## Base classes (src/core/)

Three base classes. Choose right one:

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

- Extend `AggregateRoot<Props>` if entity emits domain events (most cases)
- Extend `Entity<Props>` only if entity never emits events
- All props accessed through **getters only** — `props` is protected, never exposed
- `create()` used by use cases — always requires `actorId`, always emits `<Entity>CreatedEvent`
- `reconstitute()` used by mappers only — no `actorId`, never emits events
- Business mutations emit domain events with `actorId` and `previousData` for audit trail
- Always call `touch()` on any mutation to update `updatedAt`
- Always use `type` for Props — never `interface` (prevents declaration merging and accidental mutation of typings)
- **Never import** Prisma, NestJS, or any infrastructure type
- Use `UniqueEntityID` for IDs — never raw strings

---

## Value Objects

For attributes with own validation rules, extract Value Object:

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

Value objects are **immutable** — no setters, no mutation methods. Equality structural (by value), not by identity.

---

## Optional utility

Used in `create()` to make certain props optional (system-provided defaults):

```ts
// src/core/types/optional.ts
export type Optional<T, K extends keyof T> = Pick<Partial<T>, K> & Omit<T, K>
```

---

## Multi-entity domains

When domain has more than one entity, model relationship with **join entity** + **WatchedList**.

**Join entity** — represents relationship, extends `Entity<Props>` (not AggregateRoot):

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

**WatchedList** — tracks additions and removals without replacing full collection.

> WatchedList is parameterized over **join entity** (e.g. `ProductCategory`), not target entity (e.g. `Category`). Join entity holds FKs to both parent and target, and is what gets created/deleted in join table.

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

**Aggregate root holds WatchedList as prop:**

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
- Join entity extends `Entity`, not `AggregateRoot` — no domain events
- WatchedList tracks `getNewItems()` and `getRemovedItems()` — repository uses these to upsert/delete efficiently
- Related entities accessed through aggregate root only — never fetched independently by use case that already has aggregate
- `compareItems()` compare by identity (`a.id.equals(b.id)`) unless structural equality needed
- **Join entities never have FK setters** — parent ID must be pre-generated with `new UniqueEntityID()` in use case and passed to children at creation

---

## Delete patterns

Three concepts — which apply must be defined in `docs/rules/<domain>.md`:

| Pattern | When | Field | Use case behavior |
|---------|------|-------|-------------------|
| **Hard delete** | No domain interactions exist | — | `repository.delete(id)` — permanent removal, no domain event emitted |
| **Soft delete** | Domain interactions exist (orders, references) | `deletedAt?: Date` | `entity.softDelete(actorId)` + `repository.save(entity)` |
| **Toggle active** | Entity can be disabled and re-enabled | `active: boolean` | `entity.toggleActive(actorId)` + `repository.save(entity)` |

Hard delete vs soft delete decision belongs in delete use case, informed by domain rules. Use case checks for domain-specific interactions (external references, child records from other domains) and decides:
- No interactions → hard delete
- Has interactions → soft delete

What counts as "an interaction" is domain-specific — document in `docs/rules/<domain>.md`.

**Soft-deleted entities** excluded from all `list()` queries by default. `findById()` returns entity regardless of soft-delete status (needed for delete and audit). Use `findActiveById()` for standard reads that exclude soft-deleted records.

**Hard delete and audit:** hard deletes not audited via domain events — entity permanently removed, no side effects needed. If audit required for all deletions, use soft delete exclusively.

**Cascade on delete:** own children (e.g. prices, stock) deleted with parent via `onDelete: Cascade` in Prisma schema — no application code needed. External references (e.g. orders referencing product) block deletion and return domain error checked in use case.

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