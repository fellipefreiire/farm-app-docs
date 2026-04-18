# Presenter Pattern

Presenters format domain entities into HTTP response shapes. Static utility classes — no constructor, no state, no injection.

---

## File location

```
src/infra/http/presenters/<entity>.presenter.ts
```

---

## Structure — simple entity

```ts
import type { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'

export class <Entity>Presenter {
  static toHTTP(entity: <Entity>) {
    return {
      id: entity.id.toString(),
      name: entity.name,
      optionalField: entity.optionalField ?? null,
      active: entity.active,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt ?? null,
    }
  }
}
```

---

## Structure — with WatchedList (nested collection)

```ts
import type { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'

export class <Entity>Presenter {
  static toHTTP(entity: <Entity>) {
    return {
      id: entity.id.toString(),
      name: entity.name,
      active: entity.active,
      <children>: entity.<children>.getItems().map((item) => ({
        id: item.id.toString(),
        <child>Id: item.<child>Id.toString(),
        field: item.field,
      })),
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt ?? null,
    }
  }
}
```

---

## Structure — with optional relation (denormalized read)

Entity carries denormalized field from relation (e.g. `roleName` loaded alongside `roleId`):

```ts
export class <Entity>Presenter {
  static toHTTP(entity: <Entity>) {
    return {
      id: entity.id.toString(),
      name: entity.name,
      <related>: entity.<related>Id
        ? { id: entity.<related>Id.toString(), name: entity.<related>Name }
        : null,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt ?? null,
    }
  }
}
```

---

## Structure — with augmented type (extra data not in entity)

Presenter needs data not in entity (e.g. actor name from join) — extend entity type:

```ts
import type { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'

type <Entity>WithExtra = <Entity> & {
  actorName: string
  actorEmail: string
}

export class <Entity>Presenter {
  static toHTTP(entity: <Entity>WithExtra) {
    return {
      id: entity.id.toString(),
      actor: {
        id: entity.actorId.toString(),
        name: entity.actorName,
        email: entity.actorEmail,
      },
      action: entity.action,
      createdAt: entity.createdAt,
    }
  }
}
```

---

## Structure — with extra params (optional injected data)

Related data fetched separately, passed alongside entity:

```ts
export class <Entity>Presenter {
  static toHTTP(
    entity: <Entity>,
    related?: <Related>[],
    extra?: { fieldName?: string },
  ) {
    return {
      id: entity.id.toString(),
      name: entity.name,
      relatedName: extra?.fieldName ?? null,
      items: related
        ? related.map((r) => ({ id: r.id.toString(), name: r.name }))
        : entity.items.getItems().map((r) => ({ id: r.id.toString(), name: r.name })),
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt ?? null,
    }
  }
}
```

---

## Structure — audit log presenter (with ID resolution)

Audit log presenter accepts extra params for actor name and resolved names map. Enriches `changes` fields by replacing raw IDs with human-readable names.

```ts
import type { AuditLog } from '@/domain/audit-log/enterprise/entities/audit-log'
import { isFromToField } from '@/shared/utils/is-from-to-field'

export class AuditLogPresenter {
  static toHTTP(
    log: AuditLog,
    actorName?: string,
    resolvedNames?: Map<string, string>,
  ) {
    return {
      id: log.id.toString(),
      actorId: log.actorId,
      actorName: actorName ?? null,
      action: log.action,
      entity: log.entity,
      entityId: log.entityId,
      changes: resolvedNames
        ? AuditLogPresenter.enrichChanges(log.changes, resolvedNames)
        : log.changes,
      createdAt: log.createdAt,
    }
  }

  private static enrichChanges(
    changes: Record<string, unknown> | null,
    resolvedNames: Map<string, string>,
  ): Record<string, unknown> | null {
    if (!changes) return null

    const enriched: Record<string, unknown> = {}

    for (const [key, value] of Object.entries(changes)) {
      if (isFromToField(value)) {
        enriched[key] = {
          from: typeof value.from === 'string'
            ? resolvedNames.get(value.from) ?? value.from
            : value.from,
          to: typeof value.to === 'string'
            ? resolvedNames.get(value.to) ?? value.to
            : value.to,
        }
      } else {
        enriched[key] = typeof value === 'string'
          ? resolvedNames.get(value) ?? value
          : value
      }
    }

    return enriched
  }
}
```

Controller passes `actorNames.get(log.actorId)` and full `resolvedNames` map. Presenter tries to resolve every string value — falls back to raw value if not in map.

---

## Date-only fields

Field stores only date (no time) — e.g. `@db.Date` in Prisma — format as ISO date string to avoid timezone shifting on frontend:

```ts
function formatDateOnly(date: Date): string {
  return new Date(date).toISOString().split('T')[0]  // "YYYY-MM-DD"
}

// in toHTTP:
startDate: formatDateOnly(entity.startDate),
```

---

## Monetary values

Values stored in cents (integer) must be converted to decimal on output:

```ts
amount: entity.amount / 100,  // stored as 1999, returned as 19.99
```

---

## Rules

- Always static class — never instantiated, never injected
- `id` always `.toString()` — `UniqueEntityID` not plain string
- All FK IDs always `.toString()`
- WatchedList items via `.getItems()` — never access `.currentItems` directly
- Optional relations: ternary — `entity.relatedId ? { id, name } : null`
- Never return raw Prisma objects — always pass through entity
- Never add business logic — only shape transformation and formatting
- Monetary amounts stored as integers (cents) must divide by 100 on output
- Date-only fields must format as `YYYY-MM-DD` string to avoid timezone issues
- Optional domain fields (`undefined`) must convert to `null` in `toHTTP()` via `?? null` — `JSON.stringify` omits `undefined` keys, breaking API contracts

---

## Anti-patterns

```ts
// ❌ returning UniqueEntityID directly
id: entity.id  // always: entity.id.toString()

// ❌ accessing WatchedList items directly
items: entity.items.currentItems  // always: entity.items.getItems()

// ❌ business logic in presenter
price: entity.price < 0 ? 0 : entity.price  // belongs in entity

// ❌ returning a Prisma record
static toHTTP(raw: PrismaEntity) { ... }  // always accept domain entity

// ❌ monetary value in cents without conversion
amount: entity.amount  // if stored as cents: entity.amount / 100
```

## Testing

Presenter tests verify `toHTTP()` formatting and field exclusion (e.g. password). Unit tests — no database.

```ts
// src/infra/http/presenters/__tests__/<entity>-presenter.spec.ts
import { describe, expect, it } from 'vitest'
import { <Entity>Presenter } from '../<entity>.presenter'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'

describe('<Entity>Presenter', () => {
  it('should format a <entity> entity to HTTP response', () => {
    // Arrange
    const createdAt = new Date('2024-01-01T00:00:00.000Z')
    const updatedAt = new Date('2024-01-02T00:00:00.000Z')

    const entity = <Entity>.reconstitute(
      {
        name: 'Test Name',
        // ... all entity props
        createdAt,
        updatedAt,
      },
      new UniqueEntityID('<entity>-1'),
    )

    // Act
    const result = <Entity>Presenter.toHTTP(entity)

    // Assert
    expect(result).toEqual({
      id: '<entity>-1',
      name: 'Test Name',
      // ... all HTTP response fields
      createdAt,
      updatedAt,
    })
  })

  it('should not expose sensitive fields', () => {
    // Arrange
    const entity = <Entity>.reconstitute(
      {
        name: 'Test',
        password: 'secret-password',
        // ... other props
      },
      new UniqueEntityID('<entity>-2'),
    )

    // Act
    const result = <Entity>Presenter.toHTTP(entity)

    // Assert
    expect(result).not.toHaveProperty('password')
  })
})
```

**Rules:**
- Every presenter must have test file
- Always test sensitive fields (password, tokens, secrets) excluded
- Use `reconstitute()` to build entities in presenter tests
- Presenter tests pure unit tests — no database, no DI container
- Test nullable fields: verify `undefined` converts to `null` when applicable