# Mapper Pattern

Mappers translate Prisma records ↔ domain entities. Static utility classes — no constructor, no state, no injection.

---

## File location

```
src/infra/database/prisma/mappers/<domain>/prisma-<entity>.mapper.ts
```

> **Multi-entity domains:** When domain uses subdomain folders (see `domain-organization.md`), mappers also use subdomain folders: `src/infra/database/prisma/mappers/<domain>/<subdomain>/`.

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

`toPrismaUpdate()` omits `id` and `createdAt` (immutable), returns `Prisma.<Entity>UncheckedUpdateInput`. Used by `save()` in Prisma repository.

---

## With relations

When Prisma query includes relation (e.g. `include: { role: true }`), declare composite type:

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

When entity holds WatchedList, mapper reconstructs it from included relation:

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

Prisma enums stored as strings. Cast explicitly — never trust implicit assignment:

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

- Always static class — never instantiated, never injected
- `toDomain()` always calls `<Entity>.reconstitute()` — never `<Entity>.create()` (emits spurious domain events)
- `toPrisma()` returns `Prisma.<Entity>UncheckedCreateInput` for creates
- `toPrismaUpdate()` returns `Prisma.<Entity>UncheckedUpdateInput` for updates — omits `id` and `createdAt`
- Null Prisma fields map to `undefined` in domain (`raw.field ?? undefined`) — domain uses `undefined`, not `null`
- FK IDs always wrapped in `new UniqueEntityID(raw.fieldId)` in `toDomain`
- FK IDs always `.toString()` in `toPrisma`
- WatchedList children **never** included in `toPrisma` — handled by join repository
- Relations included in `toDomain` composite type for read, never nested in `toPrisma` for write
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

## Testing

Mapper tests verify `toDomain()`, `toPrisma()`, roundtrip data preservation. Unit tests — no database required.

```ts
// src/infra/database/prisma/mappers/<domain>/__tests__/prisma-<entity>-mapper.spec.ts
import { describe, expect, it } from 'vitest'
import { Prisma<Entity>Mapper } from '../prisma-<entity>.mapper'
import { <Entity> } from '@/domain/<domain>/enterprise/entities/<entity>'
import { UniqueEntityID } from '@/core/entities/unique-entity-id'
import type { <Entity> as Prisma<Entity> } from '@/generated/prisma/client'

describe('Prisma<Entity>Mapper', () => {
  const now = new Date('2024-01-01T00:00:00.000Z')

  const prisma<Entity>: Prisma<Entity> = {
    id: '<entity>-1',
    name: 'Test Name',
    // ... all Prisma model fields
    createdAt: now,
    updatedAt: now,
  }

  describe('toDomain', () => {
    it('should map a Prisma <entity> to a domain <Entity> entity', () => {
      const entity = Prisma<Entity>Mapper.toDomain(prisma<Entity>)

      expect(entity).toBeInstanceOf(<Entity>)
      expect(entity.id.toString()).toBe('<entity>-1')
      expect(entity.name).toBe('Test Name')
      // ... assert all fields
    })
  })

  describe('toPrisma', () => {
    it('should map a domain <Entity> entity to Prisma input', () => {
      const entity = <Entity>.reconstitute(
        {
          name: 'Test Name',
          // ... all entity props
          createdAt: now,
        },
        new UniqueEntityID('<entity>-2'),
      )

      const result = Prisma<Entity>Mapper.toPrisma(entity)

      expect(result).toEqual({
        id: '<entity>-2',
        name: 'Test Name',
        // ... all Prisma fields
        createdAt: now,
      })
    })
  })

  describe('roundtrip', () => {
    it('should preserve data through toDomain → toPrisma', () => {
      const domain = Prisma<Entity>Mapper.toDomain(prisma<Entity>)
      const result = Prisma<Entity>Mapper.toPrisma(domain)

      expect(result.id).toBe(prisma<Entity>.id)
      expect(result.name).toBe(prisma<Entity>.name)
      // ... assert all fields match original
    })
  })
})
```

**Special cases — nullable JSON fields:**

When mapper handles nullable JSON fields (e.g. `changes` in AuditLog), test both value and null cases:

```ts
it('should handle null changes', () => {
  const prismaLog: PrismaAuditLog = {
    // ... other fields
    changes: null,
  }

  const auditLog = PrismaAuditLogMapper.toDomain(prismaLog)
  expect(auditLog.changes).toBeNull()
})

it('should map null changes to Prisma.JsonNull', () => {
  const auditLog = AuditLog.create({ /* ... */ changes: null })
  const result = PrismaAuditLogMapper.toPrisma(auditLog)
  expect(result.changes).toBe(Prisma.JsonNull)
})
```

**Rules:**
- Every mapper must have test file
- Always test all three directions: `toDomain`, `toPrisma`, `roundtrip`
- Use `reconstitute()` (not `create()`) when building entities for toPrisma tests — factories simulate existing records
- Test nullable/optional fields explicitly (null JSON, optional dates, etc.)
- Import Prisma types directly from `@prisma/client` for raw Prisma object shape
- Mapper tests are pure unit tests — no database, no DI container

---