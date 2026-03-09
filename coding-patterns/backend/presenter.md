# Presenter Pattern

Presenters format domain entities into HTTP response shapes. They are static utility classes — no constructor, no state, no injection.

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

When an entity carries a denormalized field from a relation (e.g. `roleName` loaded alongside `roleId`):

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

When the presenter needs data that isn't in the entity (e.g. actor name from a join), extend the entity type:

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

When related data is fetched separately and passed alongside the entity:

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

## Date-only fields

When a field stores only a date (no time) — e.g. `@db.Date` in Prisma — format as ISO date string to avoid timezone shifting on the frontend:

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

- Always a static class — never instantiated, never injected
- `id` is always `.toString()` — `UniqueEntityID` is not a plain string
- All FK IDs are always `.toString()`
- WatchedList items accessed via `.getItems()` — never access `.currentItems` directly
- Optional relations: use ternary — `entity.relatedId ? { id, name } : null`
- Never return raw Prisma objects — always pass through the entity
- Never add business logic — only shape transformation and formatting
- Monetary amounts stored as integers (cents) must be divided by 100 on output
- Date-only fields must be formatted as `YYYY-MM-DD` string to avoid timezone issues
- Optional domain fields (`undefined`) must be converted to `null` in `toHTTP()` using `?? null` — `JSON.stringify` omits `undefined` keys, breaking API contracts

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
