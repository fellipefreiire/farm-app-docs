# Events Pattern

Domain events are the fire-and-forget mechanism for cross-domain side effects. The entity emits the event; subscribers react to it in separate modules.

> **DomainEvents vs QueryBus:** DomainEvents are fire-and-forget (one-way, no return value) — used for side effects after mutations. QueryBus is request/reply (returns data) — used when a use case needs to read data from another domain. See `query-bus.md` for the full QueryBus pattern.

---

## File locations

```
src/domain/<domain>/enterprise/events/<entity>-<action>-event.ts    ← domain event (enterprise layer)
src/domain/<reacting-domain>/application/subscribers/<entity>/on-<entity>-<action>.ts  ← subscriber (application layer of the reacting domain)
src/infra/events/<domain>/<domain>-events.module.ts                  ← NestJS wiring
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), events move under the subdomain: `src/domain/<domain>/<subdomain>/enterprise/events/`. The infra events module stays flat.

---

## 1. Domain Event

```ts
// src/domain/<domain>/enterprise/events/<entity>-<action>-event.ts
import { DomainEvent } from '@/core/events/domain-event'
import type { <Entity> } from '../entities/<entity>'
import type { UniqueEntityID } from '@/core/entities/unique-entity-id'

export class <Entity><Action>Event implements DomainEvent {
  public occurredAt: Date
  public <entity>: <Entity>
  public actorId: string

  constructor(<entity>: <Entity>, actorId: string) {
    this.<entity> = <entity>
    this.actorId = actorId
    this.occurredAt = new Date()
  }

  getAggregateId(): UniqueEntityID {
    return this.<entity>.id
  }
}
```

For update events, include `previousData` for audit trail:

```ts
export class <Entity>UpdatedEvent implements DomainEvent {
  public occurredAt: Date
  public <entity>: <Entity>
  public actorId: string
  public previousData: {
    name: string
    optionalField?: string
  }

  constructor(
    <entity>: <Entity>,
    actorId: string,
    previousData: { name: string; optionalField?: string },
  ) {
    this.<entity> = <entity>
    this.actorId = actorId
    this.previousData = previousData
    this.occurredAt = new Date()
  }

  getAggregateId(): UniqueEntityID {
    return this.<entity>.id
  }
}
```

**Naming convention:**
- Created: `<Entity>CreatedEvent`
- Updated: `<Entity>UpdatedEvent`
- Deleted: `<Entity>DeletedEvent`
- Status toggle: `<Entity>ActiveStatusChangedEvent`
- Specific action: `<Entity><Action>Event` (e.g. `OrderCancelledEvent`)

---

## 2. Subscriber

```ts
// src/domain/<reacting-domain>/application/subscribers/<entity>/on-<entity>-<action>.ts
import { Injectable, Logger } from '@nestjs/common'
import type { EventHandler } from '@/core/events/event-handler'
import { DomainEvents } from '@/core/events/domain-events'
import { <Entity><Action>Event } from '@/domain/<domain>/enterprise/events/<entity>-<action>-event'
import { CreateAuditLogUseCase } from '@/domain/audit-log/application/use-cases/create-audit-log'

@Injectable()
export class On<Entity><Action> implements EventHandler {
  private readonly logger = new Logger(On<Entity><Action>.name)

  constructor(private createAuditLog: CreateAuditLogUseCase) {
    this.setupSubscriptions()
  }

  setupSubscriptions(): void {
    DomainEvents.register(
      this.handle.bind(this),
      <Entity><Action>Event.name,
    )
  }

  async handle(event: <Entity><Action>Event): Promise<void> {
    const { <entity>, actorId } = event

    try {
      await this.createAuditLog.execute({
        actorId,
        action: '<domain>:<action>',
        entity: '<ENTITY_CONSTANT>',
        entityId: <entity>.id.toString(),
        changes: null,
      })
    } catch (error) {
      // Subscribers are fire-and-forget — a failure here must never roll back
      // the original operation. Log and move on.
      this.logger.error(
        `Failed to handle ${<Entity><Action>Event.name} for ${<entity>.id.toString()}`,
        error,
      )
    }
  }
}
```

For update subscribers, pass `previousData` in `changes`:

```ts
async handle(event: <Entity>UpdatedEvent): Promise<void> {
  const { <entity>, actorId, previousData } = event

  try {
    await this.createAuditLog.execute({
      actorId,
      action: '<domain>:updated',
      entity: '<ENTITY_CONSTANT>',
      entityId: <entity>.id.toString(),
      changes: {
        name: { before: previousData.name, after: <entity>.name },
        optionalField: {
          before: previousData.optionalField,
          after: <entity>.optionalField,
        },
      },
    })
  } catch (error) {
    this.logger.error(
      `Failed to handle ${<Entity>UpdatedEvent.name} for ${<entity>.id.toString()}`,
      error,
    )
  }
}
```

---

## 2b. Subscriber with domain enrichment (resolving names at write time)

When a created event carries foreign key IDs (e.g. `cropTypeId` on a variety), the subscriber resolves them to human-readable names before storing the audit log. This makes the audit data self-contained — the read path doesn't need to re-resolve historical IDs.

```ts
@Injectable()
export class On<Entity>Created implements EventHandler {
  private readonly logger = new Logger(On<Entity>Created.name)

  constructor(
    private createAuditLog: CreateAuditLogUseCase,
    private <related>Repository: <Related>Repository,  // injected for name resolution
  ) {
    this.setupSubscriptions()
  }

  setupSubscriptions(): void {
    DomainEvents.register(this.handle.bind(this), <Entity>CreatedEvent.name)
  }

  async handle(event: <Entity>CreatedEvent): Promise<void> {
    const { <entity>, actorId } = event

    try {
      // Resolve foreign key to name
      const related = await this.<related>Repository.findById(
        <entity>.<related>Id.toString(),
      )

      await this.createAuditLog.execute({
        actorId,
        action: '<domain>:created',
        entity: '<ENTITY_CONSTANT>',
        entityId: <entity>.id.toString(),
        changes: {
          name: <entity>.name,
          <related>Id: <entity>.<related>Id.toString(),
          <related>: { from: null, to: related?.name ?? null },  // resolved name stored alongside ID
        },
      })
    } catch (error) {
      this.logger.error(
        `Failed to handle ${<Entity>CreatedEvent.name} for ${<entity>.id.toString()}`,
        error,
      )
    }
  }
}
```

**Rules for enrichment:**
- Always store the raw ID field (`cropTypeId`) alongside the resolved name field (`cropType: { from: null, to: "Soja" }`)
- Use `findById` for single lookups (created events have one value per FK)
- The resolved name is stored as a `{ from, to }` field — consistent with update events
- If resolution fails (entity deleted), store `null` — never block the audit log creation

---

## 3. Events Module (NestJS wiring)

```ts
// src/infra/events/<domain>/<domain>-events.module.ts
import { Module } from '@nestjs/common'
import { On<Entity>Created } from '@/domain/audit-log/application/subscribers/<entity>/on-<entity>-created'
import { On<Entity>Updated } from '@/domain/audit-log/application/subscribers/<entity>/on-<entity>-updated'
import { On<Entity>ActiveStatusChanged } from '@/domain/audit-log/application/subscribers/<entity>/on-<entity>-active-status-changed'
import { On<Entity>Deleted } from '@/domain/audit-log/application/subscribers/<entity>/on-<entity>-deleted'
import { CreateAuditLogUseCase } from '@/domain/audit-log/application/use-cases/create-audit-log'
import { AuditLogDatabaseModule } from '@/infra/database/prisma/repositories/audit-log/audit-log-database.module'

@Module({
  imports: [
    AuditLogDatabaseModule,
    // Import domain database modules when subscribers need to resolve foreign IDs
    // e.g. CropDatabaseModule when variety/harvest subscribers resolve cropTypeId → name
  ],
  providers: [
    On<Entity>Created,
    On<Entity>Updated,
    On<Entity>ActiveStatusChanged,
    On<Entity>Deleted,
    CreateAuditLogUseCase,
  ],
})
export class <Domain>EventsModule {}
```

---

## How dispatch works

1. Entity method calls `this.addDomainEvent(new <Entity><Action>Event(this, actorId))`
2. `AggregateRoot.addDomainEvent()` queues the event and marks the aggregate for dispatch
3. Repository calls `DomainEvents.dispatchEventsForAggregate(entity.id)` after `create()` or `save()`
4. `DomainEvents` finds all registered handlers for that event class name and calls them
5. Subscriber `handle()` executes the side effect (e.g. audit log)

---

## Rules

- Events are emitted by the entity — never by the use case or controller
- Events carry `actorId` for audit trail
- Update events carry `previousData` — the state before the change
- Subscribers live in the **application layer** of the domain that reacts (usually `audit-log`)
- The events module lives in `src/infra/events/<domain>/` — this is the NestJS wiring layer
- The `<domain>` in `src/infra/events/<domain>/` refers to the **emitting** domain. Subscribers imported live in the **reacting** domain.
- Each subscriber registers itself in `setupSubscriptions()` in the constructor
- Subscribers implement `EventHandler` interface
- `DomainEvents.register()` uses the event class **name** as the key — never a string literal
- Never call a use case from within another use case to react to events — use subscribers
- **Always wrap `handle()` in try/catch** — subscribers are fire-and-forget. A failure must never roll back the original operation
- **Always log errors in `handle()`** using NestJS `Logger` — never swallow silently, never rethrow
- **Hard deletes do not emit events** — `repository.delete()` permanently removes the entity without dispatching. Only soft deletes (`entity.softDelete()` + `repository.save()`) emit `<Entity>DeletedEvent`. If a domain requires audit for all deletions, use soft delete exclusively
- **Every new audit subscriber requires frontend translations** — when adding a new `action` string (e.g., `'input:created'`), you must also add translations in three frontend files (see checklist below)

### Audit event translation checklist

When creating audit subscribers for a new domain, **all three files must be updated** with translations for every `action` string the subscribers produce:

| File | What to add |
|------|-------------|
| `frontend/src/domains/audit-log/utils/action-label-map.ts` | `detailActionLabelMap` (e.g., `'input:created': 'Insumo criado'`) and `tableActionLabelMap` (e.g., `'input:created': 'Criado'`) |
| `frontend/src/domains/audit-log/components/limited-audit-logs.tsx` | `actionTextMap` entry with humanized text function (e.g., `` `Insumo ${v ?? ''} criado por ${a}` ``) |
| `frontend/src/domains/audit-log/components/audit-diff-modal.tsx` | `fieldLabelMap` for field names in changes (e.g., `unitOfMeasure: 'Unidade de Medida'`), `hiddenFields` for ID fields to hide (e.g., `categoryId`), and `valueLabelMap` for enum values (e.g., `LOSS: 'Perda'`) |

**Action string format:** `<entity>:<verb>` — e.g., `category:created`, `input:status-changed`, `stock-movement:cancelled`

**Without these translations, audit logs display raw English event keys like `input:created` instead of `Insumo criado`.**

---

## Anti-patterns

```ts
// ❌ emitting event in use case instead of entity
async execute() {
  const entity = <Entity>.create(...)
  this.eventBus.emit(new <Entity>CreatedEvent(entity, actorId)) // emit in entity
}

// ❌ no previousData in update event
export class <Entity>UpdatedEvent {
  constructor(public <entity>: <Entity>, public actorId: string) {
    // missing previousData — audit log can't show what changed
  }
}

// ❌ string literal as event key
DomainEvents.register(handler, '<Entity>CreatedEvent') // use <Entity>CreatedEvent.name

// ❌ subscriber in wrong layer
// src/infra/events/<domain>/on-<entity>-created.ts  ← wrong, subscriber is application layer
// src/domain/audit-log/application/subscribers/     ← correct

// ❌ no try/catch in handle() — failure propagates and may roll back the original operation
async handle(event: <Entity>CreatedEvent): Promise<void> {
  await this.createAuditLog.execute({ ... }) // unhandled rejection
}

// ❌ rethrowing in handle() — same problem as no try/catch
} catch (error) {
  throw error // never rethrow — log and move on
}
```
