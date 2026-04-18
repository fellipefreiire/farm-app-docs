# Domain Organization Pattern

Domains are top-level folders under `src/domains/`. Each domain maps to a bounded context from `docs/rules/<domain>.md`. This document defines when and how to organize domains — including multi-entity domains that require subdomain folders.

---

## Decision: flat vs subdomain structure

A domain starts flat. When it grows to contain **two or more entities with full CRUD**, split into subdomain folders.

| Scenario | Structure | Example |
|----------|-----------|---------|
| Single entity or simple domain | Flat | `domains/audit-log/` |
| Multiple entities in the same bounded context | Subdomain folders | `domains/crop/crop-types/`, `domains/crop/varieties/` |

**Rule of thumb:** if the domain has more than ~30 files across actions/api/schemas/components, it needs subdomain folders. Don't wait until it's unmanageable — split early.

---

## Flat domain structure

Used when the domain has a single entity or a small set of closely related operations.

```
src/domains/<domain>/
  actions/
    create-<entity>.ts
    edit-<entity>.ts
    delete-<entity>.ts
    index.ts
  api/
    list-<entity>s.ts
    find-<entity>-by-id.ts
    create-<entity>.ts
    edit-<entity>.ts
    delete-<entity>.ts
    index.ts
  components/
    table/
    forms/
    dialogs/
    sheets/
    ui/
    index.ts
  schemas/
    <entity>.schema.ts
    create-<entity>.schema.ts
    edit-<entity>.schema.ts
    list-<entity>s.schema.ts
    index.ts
  store/
    <entity>-dialogs.store.ts
    index.ts
  index.ts                          ← optional, re-exports from subdirectories
```

**Example:** `domains/audit-log/` — single entity, append-only, no mutations from UI.

---

## Subdomain structure

Used when the domain contains multiple entities that belong to the same bounded context (share business rules, reference each other).

```
src/domains/<domain>/
  <entity-a>/
    actions/
      create-<entity-a>.ts
      edit-<entity-a>.ts
      delete-<entity-a>.ts
      index.ts
    api/
      list-<entity-a>s.ts
      find-<entity-a>-by-id.ts
      create-<entity-a>.ts
      edit-<entity-a>.ts
      delete-<entity-a>.ts
      index.ts
    components/
      table/
      forms/
      dialogs/
      sheets/
      ui/
      index.ts
    schemas/
      <entity-a>.schema.ts
      create-<entity-a>.schema.ts
      edit-<entity-a>.schema.ts
      list-<entity-a>s.schema.ts
      index.ts
    store/
      <entity-a>-dialogs.store.ts
      index.ts
  <entity-b>/
    actions/
    api/
    components/
    schemas/
    store/
  <entity-c>/
    actions/
    api/
    components/
    schemas/
    store/
```

Each subdomain folder mirrors the flat domain structure exactly — same directories, same naming conventions, same barrel exports.

**Example:** `domains/crop/` with subdomains `crop-types/`, `varieties/`, `harvests/`.

---

## Import paths

### Flat domain

```ts
import { listAuditLogs } from '@/domains/audit-log/api'
import { AuditLogTable } from '@/domains/audit-log/components'
import type { AuditLog } from '@/domains/audit-log/schemas'
```

### Subdomain

```ts
import { listCropTypes } from '@/domains/crop/crop-types/api'
import { CropTypeTable } from '@/domains/crop/crop-types/components'
import type { CropType } from '@/domains/crop/crop-types/schemas'

import { listVarieties } from '@/domains/crop/varieties/api'
import { VarietyTable } from '@/domains/crop/varieties/components'

import { createHarvestAction } from '@/domains/crop/harvests/actions'
```

No root-level `index.ts` re-exporting everything — import directly from the subdomain to keep dependencies explicit and tree-shaking effective.

---

## Cross-subdomain imports

Entities within the same domain may reference each other. This is expected and allowed — they share a bounded context.

```ts
// domains/crop/varieties/components/forms/create-variety-form.tsx
import { listCropTypes } from '@/domains/crop/crop-types/api'
```

```ts
// domains/crop/harvests/components/forms/create-harvest-form.tsx
import { listCropTypes } from '@/domains/crop/crop-types/api'
import { listVarieties } from '@/domains/crop/varieties/api'
```

**Rule:** Cross-domain imports (between different bounded contexts) go through API functions or events, never direct entity imports.

---

## Subdomain naming

Use the **plural entity name** as the subdomain folder name, matching the URL segment:

| Entity | Subdomain folder | URL |
|--------|-----------------|-----|
| CropType | `crop-types/` | `/crop-types` |
| Variety | `varieties/` | `/varieties` |
| Harvest | `harvests/` | `/harvests` |

---

## Pages (App Router)

Pages remain at the top level under `app/(private)/`, not nested under the domain folder. The route structure mirrors the subdomain naming:

```
src/app/(private)/
  crop-types/
    page.tsx                    ← list
    loading.tsx
    [id]/
      page.tsx                  ← detail
      loading.tsx
      not-found.tsx
      audit/
        page.tsx                ← audit sub-page
        loading.tsx
  varieties/
    page.tsx
    loading.tsx
    [id]/
      page.tsx
      loading.tsx
      not-found.tsx
      audit/
        page.tsx
        loading.tsx
  harvests/
    page.tsx
    loading.tsx
    [id]/
      page.tsx
      loading.tsx
      not-found.tsx
      audit/
        page.tsx
        loading.tsx
```

---

## Detail page structure

Every detail page follows the `DetailLayout` pattern with three slots:

```tsx
<DetailLayout
  header={<DetailHeader ... />}
  main={
    <>
      {/* Relationship sections — limited to 5 items */}
      <DetailSection title="Related Entities">
        {data.length > 0 ? (
          <EntityTable data={data} />
        ) : (
          <EmptyState icon={Icon} message="No items" />
        )}
      </DetailSection>

      {/* Audit section — always last in main */}
      <DetailSection title="Auditoria">
        {auditLogs.data.length > 0 ? (
          <LimitedAuditLogs
            logs={auditLogs.data}
            moreHref={`/<entities>/${id}/audit`}
          />
        ) : (
          <EmptyState icon={Icon} message="Nenhum registro de auditoria" />
        )}
      </DetailSection>
    </>
  }
  sidebar={
    <DetailSection title="Detalhes">
      <div className="flex flex-col gap-4">
        <DetailField label="Field" icon={Icon}>value</DetailField>
        {/* Linked references use Link component */}
        <DetailField label="Related Entity" icon={Icon}>
          <Link href={`/<related>/${entity.relatedId}`}
            className="text-primary underline-offset-4 hover:underline">
            {relatedName}
          </Link>
        </DetailField>
      </div>
    </DetailSection>
  }
/>
```

### Rules

1. **Relationship tables** in main are limited to 5 items via the API call (`limit: 5` or `perPage: 5`)
2. **Audit preview** uses `LimitedAuditLogs` (list format), never `AuditLogTable` — the table is only for the dedicated `/audit` sub-page
3. **`LimitedAuditLogs`** includes its own "Examinar mais itens" link — do not add `linkHref`/`linkLabel` to the wrapping `DetailSection`
4. **Relationship sections** use `DetailSection` with `linkHref` and `linkLabel="Examinar mais itens"` pointing to the filtered list page
5. **Sidebar** shows entity details with `DetailField` components — reference entities are links
6. **Data fetching** uses `Promise.all` for parallel independent fetches
7. **Audit API call** always uses `listAuditLogs({ entity: '<ENTITY>', entityId: id, limit: 5 })`

---

## Audit sub-page structure

Each entity has an audit sub-page at `/<entities>/[id]/audit`:

```tsx
// src/app/(private)/<entities>/[id]/audit/page.tsx
export default async function EntityAuditPage(props: NextPageProps<{ id: string }>) {
  const params = await props.params
  const searchParams = await props.searchParams
  const id = params.id
  const cursor = getSearchParam(searchParams['cursor'])

  const auditLogs = await listAuditLogs({
    entity: '<ENTITY>',
    entityId: id,
    limit: 20,
    cursor: cursor ?? undefined,
  })

  return (
    <DetailLayout
      header={<DetailHeader backHref={`/<entities>/${id}`} backLabel="Entity Name" title="Auditoria" icon={Icon} />}
      main={
        <DetailSection title="Registros de Auditoria">
          <AuditLogTable data={auditLogs.data} />
          {auditLogs.meta.nextCursor && (
            <Link href={`/<entities>/${id}/audit?cursor=${auditLogs.meta.nextCursor}`}>
              Carregar mais
            </Link>
          )}
        </DetailSection>
      }
    />
  )
}
```

The audit sub-page uses `AuditLogTable` (table format) with cursor pagination.

---

## When to create a new subdomain

Add a new subdomain folder when the domain gains a new entity that has:

1. Its own CRUD operations (create, edit, delete, list, find)
2. Its own schemas, actions, and API functions
3. Its own components (forms, dialogs, table)

If the new concept is just a value object or a simple enum used by existing entities, keep it in the parent entity's files — don't create a subdomain for it.

---

## Migration: flat → subdomain

When a flat domain grows to need subdomains:

1. Create subdomain folders matching entity names
2. Move files into their respective subdomain folders
3. Update all import paths across the codebase
4. Update barrel exports (`index.ts`) in each subdomain
5. Remove the old root-level barrel export
6. Run `pnpm build` to verify no broken imports
