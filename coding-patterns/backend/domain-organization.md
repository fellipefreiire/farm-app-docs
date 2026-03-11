# Domain Organization Pattern

Domains are top-level folders under `src/domain/`. Each domain maps to a bounded context from `docs/rules/<domain>.md`. This document defines when and how to organize domains — including multi-entity domains that require subdomain folders.

---

## Decision: flat vs subdomain structure

A domain starts flat. When it grows to contain **two or more entities with full CRUD**, split into subdomain folders.

| Scenario | Structure | Example |
|----------|-----------|---------|
| Single entity or simple domain | Flat | `domain/audit-log/` |
| Multiple entities in the same bounded context | Subdomain folders | `domain/crop/crop-types/`, `domain/crop/varieties/` |

---

## Flat domain structure

```
src/domain/<domain>/
  enterprise/
    entities/
      <entity>.ts
    events/
      <entity>-created-event.ts
      <entity>-updated-event.ts
      <entity>-deleted-event.ts
  application/
    use-cases/
      create-<entity>.ts
      edit-<entity>.ts
      delete-<entity>.ts
      find-<entity>-by-id.ts
      list-<entity>s.ts
      __tests__/
        create-<entity>.spec.ts
        ...
      errors/
        <entity>-not-found-error.ts
        ...
    repositories/
      <entity>s-repository.ts
```

---

## Subdomain structure

When the domain has multiple entities, each entity gets its own subdomain folder mirroring the flat structure:

```
src/domain/<domain>/
  <entity-a>/
    enterprise/
      entities/
        <entity-a>.ts
      events/
        <entity-a>-created-event.ts
        <entity-a>-updated-event.ts
        <entity-a>-deleted-event.ts
    application/
      use-cases/
        create-<entity-a>.ts
        edit-<entity-a>.ts
        delete-<entity-a>.ts
        find-<entity-a>-by-id.ts
        list-<entity-a>s.ts
        __tests__/
          create-<entity-a>.spec.ts
          ...
        errors/
          <entity-a>-not-found-error.ts
          ...
      repositories/
        <entity-a>s-repository.ts
  <entity-b>/
    enterprise/
      entities/
      events/
    application/
      use-cases/
        __tests__/
        errors/
      repositories/
  <entity-c>/
    enterprise/
    application/
```

**Example:** `domain/crop/` with subdomains `crop-types/`, `varieties/`, `harvests/`.

---

## Infrastructure layer

The infrastructure layer mirrors the domain structure:

### Flat domain

```
src/infra/
  http/controllers/<domain>/
    create-<entity>.controller.ts
    <domain>-http.module.ts
  database/prisma/
    mappers/<domain>/
      prisma-<entity>.mapper.ts
    repositories/<domain>/
      prisma-<entity>s.repository.ts
      <domain>-database.module.ts
  events/<domain>/
    <domain>-events.module.ts
```

### Subdomain — infrastructure mirrors domain subdomains

The infrastructure layer also uses subdomain folders for controllers, mappers, and repositories. NestJS modules and error filters stay at the `<domain>/` level since they wire all entities together.

```
src/infra/
  http/
    controllers/<domain>/
      <subdomain-a>/
        create-<entity-a>.controller.ts
        edit-<entity-a>.controller.ts
        ...
      <subdomain-b>/
        create-<entity-b>.controller.ts
        ...
      <domain>-http.module.ts            ← single module at domain level
    presenters/<domain>/
      <entity-a>.presenter.ts            ← flat under domain (one file per entity)
      <entity-b>.presenter.ts
      <entity-c>.presenter.ts
    filters/
      <domain>-error.filter.ts           ← single filter at domain level
    dtos/
      error/<domain>/
        <subdomain-a>/
          <entity-a>-not-found.dto.ts
        <subdomain-b>/
          ...
      requests/<domain>/
        <subdomain-a>/
          create-<entity-a>-request.dto.ts
        <subdomain-b>/
          ...
      response/<domain>/
        <subdomain-a>/
          <entity-a>-response.dto.ts
        <subdomain-b>/
          ...
  database/prisma/
    mappers/<domain>/
      <subdomain-a>/
        prisma-<entity-a>.mapper.ts
      <subdomain-b>/
        prisma-<entity-b>.mapper.ts
    repositories/<domain>/
      <subdomain-a>/
        prisma-<entity-a>s.repository.ts
      <subdomain-b>/
        prisma-<entity-b>s.repository.ts
      <domain>-database.module.ts        ← single module at domain level
  events/<domain>/
    <domain>-events.module.ts            ← single module at domain level
```

**What stays flat (at domain level):**
- NestJS modules (`*-http.module.ts`, `*-database.module.ts`, `*-events.module.ts`) — they wire all entities together
- Error filters — they handle errors from all entities in the domain
- Presenters — one file per entity, flat under `presenters/<domain>/`

**What splits into subdomains:**
- Controllers, mappers, Prisma repositories, DTOs — these are per-entity and benefit from the same organization as the domain layer

---

## Test infrastructure

Test helpers mirror the domain structure:

### Flat domain

```
test/
  factories/
    make-<entity>.ts
  repositories/<domain>/
    in-memory-<entity>s-repository.ts
```

### Subdomain — test repos mirror domain, factories stay flat

In-memory repositories mirror the domain subdomain structure. Factories stay flat since they are simple one-file-per-entity helpers.

```
test/
  factories/
    make-<entity-a>.ts                        ← flat (one file per entity)
    make-<entity-b>.ts
    make-<entity-c>.ts
  repositories/<domain>/
    <subdomain-a>/
      in-memory-<entity-a>s-repository.ts
    <subdomain-b>/
      in-memory-<entity-b>s-repository.ts
    <subdomain-c>/
      in-memory-<entity-c>s-repository.ts
```

---

## Import paths

### Flat domain

```ts
import { AuditLog } from '@/domain/audit-log/enterprise/entities/audit-log'
import { CreateAuditLogUseCase } from '@/domain/audit-log/application/use-cases/create-audit-log'
```

### Subdomain

```ts
import { CropType } from '@/domain/crop/crop-types/enterprise/entities/crop-type'
import { CreateCropTypeUseCase } from '@/domain/crop/crop-types/application/use-cases/create-crop-type'

import { Variety } from '@/domain/crop/varieties/enterprise/entities/variety'
import { CreateVarietyUseCase } from '@/domain/crop/varieties/application/use-cases/create-variety'
```

---

## Cross-subdomain references

Entities within the same domain may reference each other. This is expected — they share a bounded context.

```ts
// domain/crop/varieties/application/use-cases/create-variety.ts
import { CropTypesRepository } from '@/domain/crop/crop-types/application/repositories/crop-types-repository'
```

```ts
// domain/crop/harvests/application/use-cases/create-harvest.ts
import { CropTypesRepository } from '@/domain/crop/crop-types/application/repositories/crop-types-repository'
import { VarietiesRepository } from '@/domain/crop/varieties/application/repositories/varieties-repository'
```

**Cross-domain references** (between different bounded contexts) use domain events, never direct imports.

---

## Subdomain naming

Use the **plural entity name** in kebab-case:

| Entity | Subdomain folder |
|--------|-----------------|
| CropType | `crop-types/` |
| Variety | `varieties/` |
| Harvest | `harvests/` |

---

## When to create a new subdomain

Add a new subdomain folder when the domain gains a new entity that has:

1. Its own aggregate root (entity with identity)
2. Its own repository interface
3. Its own use cases (create, edit, delete, list, find)
4. Its own domain events

Value objects used by an entity stay in that entity's `enterprise/entities/` folder.

---

## Migration: flat → subdomain

When a flat domain grows to need subdomains:

1. Create subdomain folders matching entity names
2. Move entity, events, use cases, errors, and repository interface into respective subdomains
3. Update all import paths across `src/` and `test/`
4. Infrastructure modules (`*-http.module.ts`, `*-database.module.ts`, `*-events.module.ts`) keep their flat structure — only update import paths to point at subdomain locations
5. Run `pnpm test` and `pnpm test:e2e` to verify no broken imports
