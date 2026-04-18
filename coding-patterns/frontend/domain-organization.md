# Domain Organization Pattern

Domains = top-level folders under `src/domains/`. Each domain maps to bounded context from `docs/rules/<domain>.md`. Defines when/how to organize domains — including multi-entity domains needing subdomain folders.

---

## Decision: flat vs subdomain structure

Domain starts flat. When grows to **two or more entities with full CRUD**, split into subdomain folders.

| Scenario | Structure | Example |
|----------|-----------|---------|
| Single entity or simple domain | Flat | `domains/audit-log/` |
| Multiple entities in same bounded context | Subdomain folders | `domains/crop/crop-types/`, `domains/crop/varieties/` |

**Rule of thumb:** domain >~30 files across actions/api/schemas/components → needs subdomain folders. Split early.

---

## Flat domain structure

Single entity or small set of closely related ops.

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

**Example:** `domains/audit-log/` — single entity, append-only, no UI mutations.

---

## Subdomain structure

Multiple entities in same bounded context (share business rules, reference each other).

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

Each subdomain mirrors flat structure exactly — same dirs, same naming, same barrel exports.

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

No root-level `index.ts` re-exporting everything — import directly from subdomain. Keeps deps explicit, tree-shaking effective.

---

## Cross-subdomain imports

Entities within same domain may reference each other. Expected, allowed — share bounded context.

```ts
// domains/crop/varieties/components/forms/create-variety-form.tsx
import { listCropTypes } from '@/domains/crop/crop-types/api'
```

```ts
// domains/crop/harvests/components/forms/create-harvest-form.tsx
import { listCropTypes } from '@/domains/crop/crop-types/api'
import { listVarieties } from '@/domains/crop/varieties/api'
```

**Rule:** Cross-domain imports (different bounded contexts) go through API functions or events — never direct entity imports.

---

## Subdomain naming

Use **plural entity name** as subdomain folder, matching URL segment:

| Entity | Subdomain folder | URL |
|--------|-----------------|-----|
| CropType | `crop-types/` | `/crop-types` |
| Variety | `varieties/` | `/varieties` |
| Harvest | `harvests/` | `/harvests` |

---

## Pages (App Router)

Pages stay at top level under `app/(private)/`, not nested under domain folder. Route structure mirrors subdomain naming:

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

Every detail page follows `DetailLayout` pattern with three slots:

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

1. **Relationship tables** in main limited to 5 items via API call (`limit: 5` or `perPage: 5`)
2. **Audit preview** uses `LimitedAuditLogs` (list format), never `AuditLogTable` — table only for dedicated `/audit` sub-page
3. **`LimitedAuditLogs`** includes own "Examinar mais itens" link — don't add `linkHref`/`linkLabel` to wrapping `DetailSection`
4. **Relationship sections** use `DetailSection` with `linkHref` and `linkLabel="Examinar mais itens"` pointing to filtered list page
5. **Sidebar** shows entity details with `DetailField` components — reference entities are links
6. **Data fetching** uses `Promise.all` for parallel independent fetches
7. **Audit API call** always uses `listAuditLogs({ entity: '<ENTITY>', entityId: id, limit: 5 })`

---

## Audit sub-page structure

Each entity has audit sub-page at `/<entities>/[id]/audit`:

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

Audit sub-page uses `AuditLogTable` (table format) with cursor pagination.

---

## When to create new subdomain

Add subdomain folder when domain gains entity with:

1. Own CRUD ops (create, edit, delete, list, find)
2. Own schemas, actions, API functions
3. Own components (forms, dialogs, table)

New concept = value object or simple enum used by existing entities → keep in parent entity's files. No subdomain.

---

## Migration: flat → subdomain

1. Create subdomain folders matching entity names
2. Move files into respective subdomain folders
3. Update all import paths across codebase
4. Update barrel exports (`index.ts`) in each subdomain
5. Remove old root-level barrel export
6. Run `pnpm build` to verify no broken imports