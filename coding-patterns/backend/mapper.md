# Mapper Pattern

Mappers translate between Prisma records and domain entities. They are static utility classes — no constructor, no state, no injection.

---

## File location

```
src/infra/database/prisma/mappers/<domain>/prisma-<entity>.mapper.ts
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), mappers also use subdomain folders: `src/infra/database/prisma/mappers/<domain>/<subdomain>/`.

---

## Structure

```ts
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import type { Prisma, <Entity> as Prisma<Entity> } from '@/generated/prisma/client'

export class Prisma<Entity>Mapper {
  static toDomain(raw: Prisma<Entity>): <Entity> {
    return <Entity>.reconstitute(
      {
        name: raw.name,
        optionalField: raw.optionalField ?? undefined,
        active: raw.active,
        createdAt: raw.createdAt,
        updatedAt: raw.updatedAt ?? undefined,
        deletedAt: raw.deletedAt ?? undefined,
      },
      new UniqueEntityID(raw.id),
    )
  }

  static toPrisma(entity: <Entity>): Prisma.<Entity>UncheckedCreateInput {
    return {
      id: entity.id.toString(),
      name: entity.name,
      optionalField: entity.optionalField,
      active: entity.active,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    }
  }

  static toPrismaUpdate(entity: <Entity>): Prisma.<Entity>UncheckedUpdateInput {
    return {
      name: entity.name,
      optionalField: entity.optionalField,
      active: entity.active,
      updatedAt: entity.updatedAt,
      deletedAt: entity.deletedAt,
      // never include: id, createdAt — immutable fields
    }
  }
}
```

`toPrismaUpdate()` omits `id` and `createdAt` (immutable) and returns `Prisma.<Entity>UncheckedUpdateInput`. Used by `save()` in the Prisma repository.

---

## With relations

When the Prisma query includes a relation (e.g. `include: { role: true }`), declare a composite type:

```ts
import type {
  <Entity> as Prisma<Entity>,
  <Related> as Prisma<Related>,
  Prisma,
} from '@/generated/prisma/client'

type Prisma<Entity>With<Related> = Prisma<Entity> & {
  <related>?: Prisma<Related> | null
}

export class Prisma<Entity>Mapper {
  static toDomain(raw: Prisma<Entity>With<Related>): <Entity> {
    return <Entity>.reconstitute(
      {
        name: raw.name,
        <related>Id: new UniqueEntityID(raw.<related>Id),
        <related>Name: raw.<related>?.name ?? undefined,  // denormalized read model
        createdAt: raw.createdAt,
        updatedAt: raw.updatedAt ?? undefined,
      },
      new UniqueEntityID(raw.id),
    )
  }

  static toPrisma(entity: <Entity>): Prisma.<Entity>UncheckedCreateInput {
    return {
      id: entity.id.toString(),
      name: entity.name,
      <related>Id: entity.<related>Id.toString(),  // FK only — never nest relations in write
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    }
  }
}
```

---

## With WatchedList (nested collection)

When the entity holds a WatchedList, the mapper reconstructs it from the included relation:

```ts
// <Child> here refers to the join entity (e.g. ProductCategory), not the target entity
import type { <Entity> as Prisma<Entity>, <Child> as Prisma<Child> } from '@/generated/prisma/client'
import { <Parent><Child>List } from '@/domain/<domain>/enterprise/entities/<parent>-<child>-list'
import { Prisma<Child>Mapper } from './prisma-<child>.mapper'

type Prisma<Entity>With<Children> = Prisma<Entity> & {
  <children>?: Prisma<Child>[]
}

export class Prisma<Entity>Mapper {
  static toDomain(raw: Prisma<Entity>With<Children>): <Entity> {
    const children = raw.<children>
      ? raw.<children>.map(Prisma<Child>Mapper.toDomain)
      : []

    return <Entity>.reconstitute(
      {
        name: raw.name,
        <children>: new <Parent><Child>List(children),
        createdAt: raw.createdAt,
        updatedAt: raw.updatedAt ?? undefined,
      },
      new UniqueEntityID(raw.id),
    )
  }

  static toPrisma(entity: <Entity>): Prisma.<Entity>UncheckedCreateInput {
    return {
      id: entity.id.toString(),
      name: entity.name,
      // never include <children> here — handled by join repository
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    }
  }
}
```

---

## With enums

Prisma enums are stored as strings. Cast explicitly — never trust implicit assignment:

```ts
static toDomain(raw: Prisma<Entity>): <Entity> {
  return <Entity>.reconstitute(
    {
      status: raw.status as 'ACTIVE' | 'INACTIVE' | 'ARCHIVED',
      type: raw.type as '<TypeA>' | '<TypeB>',
      // ...
    },
    new UniqueEntityID(raw.id),
  )
}
```

---

## Rules

- Always a static class — never instantiated, never injected
- `toDomain()` always calls `<Entity>.reconstitute()` — never `<Entity>.create()` (which would emit spurious domain events)
- `toPrisma()` returns `Prisma.<Entity>UncheckedCreateInput` for creates
- `toPrismaUpdate()` returns `Prisma.<Entity>UncheckedUpdateInput` for updates — omits `id` and `createdAt`
- Null Prisma fields map to `undefined` in domain (`raw.field ?? undefined`) — domain uses `undefined`, not `null`
- FK IDs are always wrapped in `new UniqueEntityID(raw.fieldId)` in `toDomain`
- FK IDs are always `.toString()` in `toPrisma`
- WatchedList children are **never** included in `toPrisma` — they are handled by the join repository
- Relations are included in `toDomain` composite type for read, but never nested in `toPrisma` for write
- Enum casts must be explicit (`as 'A' | 'B'`) — never rely on implicit type compatibility

---

## Anti-patterns

```ts
// ❌ null in domain entity
optionalField: raw.optionalField ?? null  // use undefined, not null

// ❌ raw string ID in domain
fieldId: raw.fieldId  // always wrap: new UniqueEntityID(raw.fieldId)

// ❌ writing nested relations in toPrisma
static toPrisma(entity) {
  return {
    ...
    children: { create: [...] }  // never — join repository handles this
  }
}

// ❌ implicit enum assignment
status: raw.status  // always cast explicitly: raw.status as 'ACTIVE' | 'INACTIVE'

// ❌ using toPrisma() in save() instead of toPrismaUpdate()
await prisma.<entity>.update({ data: Prisma<Entity>Mapper.toPrisma(entity) })
// toPrisma() includes id and createdAt — Prisma rejects immutable fields in update
```
