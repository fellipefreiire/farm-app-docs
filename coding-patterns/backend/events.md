# Events Pattern

Domain events = fire-and-forget for cross-domain side effects. Entity emits; subscribers react in separate modules.

> **DomainEvents vs QueryBus:** DomainEvents fire-and-forget (one-way, no return) — side effects after mutations. QueryBus request/reply (returns data) — when use case needs data from another domain. See `query-bus.md` for full QueryBus pattern.

---

## File locations

```
src/domain/<domain>/enterprise/events/<entity>-<action>-event.ts    ← domain event (enterprise layer)
src/domain/<reacting-domain>/application/subscribers/<entity>/on-<entity>-<action>.ts  ← subscriber (application layer of the reacting domain)
src/infra/events/<domain>/<domain>-events.module.ts                  ← NestJS wiring
```

> **Multi-entity domains:** When domain uses subdomain folders (see `domain-organization.md`), events move under subdomain: `src/domain/<domain>/<subdomain>/enterprise/events/`. Infra events module stays flat.

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

Update events include `previousData` for audit trail:

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

Update subscribers pass `previousData` in `changes`:

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

When created event carries FK IDs (e.g. `cropTypeId` on variety), subscriber resolves to human-readable names before storing audit log. Makes audit data self-contained — read path needn't re-resolve historical IDs.

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

**Enrichment rules:**
- Always store raw ID field (`cropTypeId`) alongside resolved name field (`cropType: { from: null, to: "Soja" }`)
- Use `findById` for single lookups (created events have one value per FK)
- Resolved name stored as `{ from, to }` — consistent with update events
- Resolution fails (entity deleted) → store `null` — never block audit log creation

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
2. `AggregateRoot.addDomainEvent()` queues event, marks aggregate for dispatch
3. Repository calls `DomainEvents.dispatchEventsForAggregate(entity.id)` after `create()` or `save()`
4. `DomainEvents` finds all registered handlers for that event class name, calls them
5. Subscriber `handle()` executes side effect (e.g. audit log)

---

## Rules

- Events emitted by entity — never by use case or controller
- Events carry `actorId` for audit trail
- Update events carry `previousData` — state before change
- Subscribers live in **application layer** of reacting domain (usually `audit-log`)
- Events module lives in `src/infra/events/<domain>/` — NestJS wiring layer
- `<domain>` in `src/infra/events/<domain>/` = **emitting** domain. Imported subscribers live in **reacting** domain.
- Each subscriber registers in `setupSubscriptions()` in constructor
- Subscribers implement `EventHandler` interface
- `DomainEvents.register()` uses event class **name** as key — never string literal
- Never call use case from within another use case to react to events — use subscribers
- **Always wrap `handle()` in try/catch** — subscribers fire-and-forget. Failure must never roll back original operation
- **Always log errors in `handle()`** using NestJS `Logger` — never swallow silently, never rethrow
- **Hard deletes don't emit events** — `repository.delete()` removes entity without dispatching. Only soft deletes (`entity.softDelete()` + `repository.save()`) emit `<Entity>DeletedEvent`. Domains requiring audit for all deletions must use soft delete exclusively
- **Every new audit subscriber requires frontend translations** — when adding new `action` string (e.g., `'input:created'`), add translations in three frontend files (see checklist below)

### Audit event translation checklist

When creating audit subscribers for new domain, **all three files must be updated** with translations for every `action` string subscribers produce:

| File | What to add |
|------|-------------|
| `frontend/src/domains/audit-log/utils/action-label-map.ts` | `detailActionLabelMap` (e.g., `'input:created': 'Insumo criado'`) and `tableActionLabelMap` (e.g., `'input:created': 'Criado'`) |
| `frontend/src/domains/audit-log/components/limited-audit-logs.tsx` | `actionTextMap` entry with humanized text function (e.g., `` `Insumo ${v ?? ''} criado por ${a}` ``) |
| `frontend/src/domains/audit-log/components/audit-diff-modal.tsx` | `fieldLabelMap` for field names in changes (e.g., `unitOfMeasure: 'Unidade de Medida'`), `hiddenFields` for ID fields to hide (e.g., `categoryId`), and `valueLabelMap` for enum values (e.g., `LOSS: 'Perda'`) |

**Action string format:** `<entity>:<verb>` — e.g., `category:created`, `input:status-changed`, `stock-movement:cancelled`

**Without these translations, audit logs show raw English event keys like `input:created` instead of `Insumo criado`.**

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