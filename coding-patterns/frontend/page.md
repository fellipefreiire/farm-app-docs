# Page Pattern

Pages are Server Components that compose domain components. Pages that fetch data must be async. They live in the Next.js App Router file structure and handle routing, data loading, search params, and error/loading states.

## Table of contents

- [File locations](#file-locations)
- [Route groups](#route-groups)
- [List page](#list-page)
  - [Status filtering — always use TableTabs](#status-filtering--always-use-tabletabs)
  - [List page filter architecture](#list-page-filter-architecture)
  - [PreserveParamsLink](#preserveparamslink)
  - [ViewToggle (list/card views)](#viewtoggle-listcard-views)
- [Detail page (top-level entity)](#detail-page-top-level-entity)
  - [Detail page sidebar rules](#detail-page-sidebar-rules)
  - [Detail page header actions](#detail-page-header-actions)
  - [Detail page linked fields](#detail-page-linked-fields)
  - [Related tables in detail pages](#related-tables-in-detail-pages)
- [Detail page (sub-entity / submodule)](#detail-page-sub-entity--submodule)
- [Sub-entity listing page](#sub-entity-listing-page)
- [Audit page](#audit-page)
- [Post-deletion redirect](#post-deletion-redirect)
- [loading.tsx](#loadingtsx)
- [not-found.tsx](#not-foundtsx)
- [Rules](#rules)
- [Anti-patterns](#anti-patterns)

---

## File locations

```
src/app/
  (public)/                          ← unauthenticated routes (sign-in, sign-up)
    page.tsx
  (private)/                         ← authenticated routes
    <domain>/
      page.tsx                       ← list page
      loading.tsx                    ← loading skeleton
      not-found.tsx                  ← 404 page
      [id]/
        page.tsx                     ← detail page
        loading.tsx
        not-found.tsx
        audit/
          page.tsx                   ← audit log page
          loading.tsx
        <sub-entity>/                ← sub-module listing (e.g., varieties)
          page.tsx
          loading.tsx
          [subId]/
            page.tsx                 ← sub-module detail page
            loading.tsx
            not-found.tsx
            audit/
              page.tsx
              loading.tsx
    layout.tsx                       ← shared layout (sidebar, nav)
  layout.tsx                         ← root layout (fonts, providers, Toaster)
  not-found.tsx                      ← global 404
```

### Routing hierarchy

Sub-entities that have a 1-N relationship with a parent entity are nested under the parent's route:

```
/crop-types                          ← list crop types
/crop-types/[id]                     ← crop type detail
/crop-types/[id]/audit               ← crop type audit logs
/crop-types/[id]/varieties           ← varieties for this crop type
/crop-types/[id]/varieties/[varietyId]       ← variety detail (submodule layout)
/crop-types/[id]/varieties/[varietyId]/audit ← variety audit logs
```

Sub-entities do NOT have standalone top-level routes. They only exist in the context of their parent.

---

## Route groups

Use `(public)` and `(private)` as the default route groups.

```
src/app/
  (public)/         ← no auth required
  (private)/        ← auth required (layout checks authentication)
```

---

## List page

```tsx
// src/app/(private)/<domain>/page.tsx
import { Plus, <Icon> } from 'lucide-react'
import Link from 'next/link'

import { list<Entity>s } from '@/domains/<domain>/api'
import { find<RelatedEntity>ById } from '@/domains/<related-domain>/api'
import {
  <Entity>Table,
  Create<Entity>Sheet,
  Edit<Entity>Sheet,
  Delete<Entity>Dialog,
  <Entity>Filters,
  <Entity>ActiveFilters,
} from '@/domains/<domain>/components'
import { EmptyState } from '@/shared/components/empty-state'
import { PaginationControls } from '@/shared/components/pagination-controls'
import { SearchInput } from '@/shared/components/search-input'
import { Button } from '@/shared/components/ui/button'
import { getSearchParam } from '@/shared/utils/search-params'

export default async function <Entity>sPage(props: NextPageProps) {
  const searchParams = await props.searchParams

  // Standard params (all list pages)
  const page = getSearchParam(searchParams.page)
  const query = getSearchParam(searchParams.query)
  const sort = getSearchParam(searchParams.sort)
  const order = getSearchParam(searchParams.order)

  // Domain-specific filter params — read from URL, pass to API
  const <filterParam> = getSearchParam(searchParams.<filterParam>)
  const active = getSearchParam(searchParams.active)

  const data = await list<Entity>s({
    page: page ? Number(page) : undefined,
    query,
    sort,
    order,
    // Domain-specific filters — convert URL strings to API types
    <filterParam>: <filterParam> ?? undefined,
    active: active !== undefined && active !== null ? active === 'true' : undefined,
  })

  // Count active filters for the badge
  const activeFilterCount = [<filterParam>, active].filter(Boolean).length

  // Resolve filter IDs to names for active filter pills (server-side)
  const activeFilters: { param: string; label: string; value: string }[] = []

  if (<filterParam>) {
    const entity = await find<RelatedEntity>ById(<filterParam>).catch(() => null)
    if (entity) activeFilters.push({ param: '<filterParam>', label: '<FilterLabel>', value: entity.name })
  }

  if (active) {
    activeFilters.push({ param: 'active', label: 'Status', value: active === 'true' ? 'Ativo' : 'Inativo' })
  }

  return (
    <>
      <div className="container flex h-full flex-col p-10" data-testid="<entity>s-page">
        <div className="mb-10 flex justify-between">
          <h1 className="text-2xl font-bold"><Entity>s</h1>
          <Button asChild>
            <Link href="/<entities>?create=<entity>" scroll={false} data-testid="<entity>-create-button">
              <Plus />
              Adicionar <Entity>
            </Link>
          </Button>
        </div>

        <div className="mb-6 flex flex-col gap-3">
          <div className="flex gap-2">
            <SearchInput placeholder="Pesquisar <entities>..." />
            <<Entity>Filters
              activeCount={activeFilterCount}
              current<FilterParam>={<filterParam> ?? undefined}
              currentActive={active ?? undefined}
            />
          </div>
          <<Entity>ActiveFilters filters={activeFilters} />
        </div>

        {data.data.length > 0 ? (
          <>
            <<Entity>Table data={data.data} />
            <PaginationControls
              currentPage={data.meta.page}
              totalPages={Math.ceil(data.meta.total / data.meta.perPage)}
            />
          </>
        ) : (
          <EmptyState
            icon={<Icon>}
            message="Nenhum(a) <entity> cadastrado(a)"
          />
        )}
      </div>

      {searchParams.create === '<entity>' && <Create<Entity>Sheet />}
      <Delete<Entity>Dialog />
    </>
  )
}
```

### Status filtering — always use `TableTabs`

When a list page has a status filter (e.g., `active`, `status`), it must use `TableTabs` — **never** inside the filter popover. Status is a primary navigation concern, not a secondary filter.

```tsx
import { TableTabs, type TabConfig } from '@/shared/components/table-tabs'

const TABLE_TABS: TabConfig[] = [
  { text: 'Todos', icon: 'ScrollText', tabKey: null, paramName: 'active' },
  { text: 'Ativos', icon: 'CircleCheck', tabKey: 'true', paramName: 'active' },
  { text: 'Inativos', icon: 'Archive', tabKey: 'false', paramName: 'active' },
]

// In the page JSX, tabs go above search/filters:
<div className="mb-4 flex justify-between border-b">
  <TableTabs tabs={TABLE_TABS} />
</div>
```

`TabConfig` does **not** include `href` — `TabItem` builds the URL dynamically from current searchParams, preserving all existing params (view, filters, etc.) and only changing the tab param.

### List page filter architecture

Filters use domain-specific components, **not** inline `FilterSelect` in the page. Each domain creates two components:

**1. `<Entity>Filters`** — Popover with a "Filtros" button, filter fields inside, "Aplicar" and "Limpar" buttons.

```
src/domains/<domain>/components/ui/<entity>-filters.tsx
```

- Uses `AsyncSelectField` for relationship filters (fetches options with search + infinite scroll)
- **Status/boolean filters go in `TableTabs`, never in the popover**
- Syncs with URL via `useFilterNavigation().setParams()`
- Shows active filter count as a badge on the trigger button
- Receives current filter values as props (from the server component page)

**Popover layout rules — max 3 rows per column:**

| Fields | Layout |
|--------|--------|
| 1–3    | Single column |
| 4      | 2×2 grid (2 columns, 2 rows) |
| 5–6    | 2 columns (3 rows max per column) |
| 7–9    | 3 columns (3 rows max per column) |

```tsx
// 1–3 fields: single column (default)
<div className="flex flex-col gap-4">
  <h3 className="text-sm font-bold">Filtrar <entities></h3>
  <Field1 />
  <Field2 />
  <Field3 />
  {/* Aplicar / Limpar */}
</div>

// 4 fields: 2×2 grid
<div className="flex flex-col gap-4">
  <h3 className="text-sm font-bold">Filtrar <entities></h3>
  <div className="grid grid-cols-2 gap-4">
    <Field1 />
    <Field2 />
    <Field3 />
    <Field4 />
  </div>
  {/* Aplicar / Limpar */}
</div>

// 5–6 fields: 2 columns, max 3 per column
<div className="flex flex-col gap-4">
  <h3 className="text-sm font-bold">Filtrar <entities></h3>
  <div className="grid grid-cols-2 gap-4">
    <div className="flex flex-col gap-4">
      <Field1 />
      <Field2 />
      <Field3 />
    </div>
    <div className="flex flex-col gap-4">
      <Field4 />
      <Field5 />
    </div>
  </div>
  {/* Aplicar / Limpar */}
</div>
```

Use `w-80` for single-column popovers, `w-[40rem]` for 2-column layouts.

**2. `<Entity>ActiveFilters`** — Removable pills showing currently active filters.

```
src/domains/<domain>/components/ui/<entity>-active-filters.tsx
```

- Each pill shows `label: value` with an X to remove
- Removing a filter calls `useFilterNavigation().setParams({ param: null })`
- If removing a parent filter should cascade (e.g., removing cropType clears variety), use a `cascadeRemovals` map
- The page resolves filter IDs to names **server-side** (e.g., `findCategoryById(categoryId)`) and passes the resolved `filters` array

**Page layout — full structure with tabs, search, and filters:**
```tsx
{/* 1. Tabs (status) */}
<div className="mb-4 flex justify-between border-b">
  <TableTabs tabs={TABLE_TABS} />
</div>

{/* 2. Search + Filter button */}
<div className="mb-6 flex flex-col gap-3">
  <div className="flex gap-2">
    <SearchInput placeholder="Pesquisar..." />
    <<Entity>Filters activeCount={count} current...={...} />
  </div>
  <<Entity>ActiveFilters filters={activeFilters} />
</div>
```

**Do NOT** use `FilterSelect` or `TableToolbar` for domain filters. `FilterSelect` exists as a shared primitive but the standard pattern for list pages is the Popover + ActiveFilters approach above.

**Rules:**
- Pages that fetch data are async Server Components — they fetch data directly, no `useEffect`
- Filters come from `searchParams` — the URL is the source of truth for listing state
- Parse searchParams using `getSearchParam()` from shared utils — never access raw values directly
- **Status filters always use `TableTabs`** — never in the filter popover
- **Every list page must include `<Entity>Filters` and `<Entity>ActiveFilters` components** for non-status filterable fields
- Filter popover fields must not exceed **3 rows per column** — use multi-column grid for 4+ fields
- Filter ID resolution (ID → name) happens **server-side** in the page, not in the client component
- Boolean filters (like `active`) must convert the URL string to boolean before passing to the API: `active === 'true'`
- **Links that open sheets must use `PreserveParamsLink`** — never use `<Link>` with hardcoded URLs for sheet-opening actions. `PreserveParamsLink` preserves all current URL params (view, filters, tabs) and only changes the specified ones. Always include `scroll={false}`.
- Pass `query`, `sort`, `order`, `page` from searchParams to the API function
- `SortableHeader` goes in the column definitions, not in the page — the page only passes sort params to the API
- Pass parsed data to domain components — pages compose, they don't render complex UI

### PreserveParamsLink

Use `PreserveParamsLink` instead of `<Link>` whenever a navigation must preserve the current URL state (filters, view, tabs). This is mandatory for all sheet-opening actions (create, edit).

```tsx
import { PreserveParamsLink } from '@/shared/components/preserve-params-link'

// Opens create sheet while preserving view=cards, active=true, categoryId=..., etc.
<PreserveParamsLink params={{ create: 'input' }} scroll={false} data-testid="input-create-button">
  Adicionar Insumo
</PreserveParamsLink>

// Opens edit sheet while preserving all params
<PreserveParamsLink params={{ 'edit-input': id }} scroll={false}>
  <Pencil />
</PreserveParamsLink>
```

The `StackedSheet` `closeAll` automatically removes `create` and `edit-*` params when closing, preserving everything else.

### ViewToggle (list/card views)

When a list page supports multiple views (table and cards), add a `ViewToggle` to the right side of the tabs bar. The view preference is stored in URL param `view`.

```tsx
import { ViewToggle } from '@/shared/components/view-toggle'

<div className="mb-4 flex justify-between border-b">
  <TableTabs tabs={TABLE_TABS} />
  <ViewToggle currentView={view} />
</div>

// Conditional rendering:
{view === 'cards' ? (
  <<Entity>CardGrid data={rows} />
) : (
  <<Entity>Table data={rows} />
)}
```

Default view is `list` (no `view` param in URL). Card view sets `view=cards`.

---

## Detail page (top-level entity)

Top-level entities use `DetailLayout` with a 12-column grid (9/3 split), `DetailHeader` with breadcrumbs, placeholder image, and actions.

```tsx
// src/app/(private)/<domain>/[id]/page.tsx
import { DetailLayout } from '@/shared/components/detail-layout'
import { DetailHeader } from '@/shared/components/detail-header'
import { DetailSection } from '@/shared/components/detail-section'
import { DetailField } from '@/shared/components/detail-field'

export default async function <Entity>DetailPage(
  props: NextPageProps<{ id: string }>,
) {
  const params = await props.params
  const searchParams = await props.searchParams
  const id = params.id

  const entity = await find<Entity>ById(id)

  return (
    <>
      <DetailLayout
        header={
          <DetailHeader
            backHref="/<entities>"
            backLabel="<Entities>"
            title={entity.name}
            icon={<LucideIcon>}
            actions={
              <>
                <Button variant="outline" size="sm" className="font-bold" asChild>
                  <Link href={`/<entities>/${id}?edit-<entity>=${id}`}>Editar <Entity></Link>
                </Button>
                <<Entity>ActionsPopover entity={entity} showEdit={false} />
              </>
            }
          />
        }
        main={/* 9 columns: tables, audit, etc. */}
        sidebar={
          <DetailSection
            title="Detalhes"
            actions={
              <Button variant="outline" size="icon" className="size-7" asChild>
                <Link href={`/<entities>/${id}?edit-<entity>=${id}`}><Pencil /></Link>
              </Button>
            }
          >
            <div className="flex flex-col gap-4">
              {/* Never include Nome (already in header title) or Status (use badge prop in DetailHeader) */}
              <DetailField label="Criado em" icon={Calendar}>{formattedDate}</DetailField>
            </div>
          </DetailSection>
        }
      />

      {isEditing && <Edit<Entity>Sheet entity={entity} />}
      <Delete<Entity>Dialog />
    </>
  )
}
```

### Detail page sidebar rules

**Never show in detail fields:**
- **Nome** — already displayed in the `DetailHeader` title. Repeating it in the sidebar is redundant.
- **Status** — must be shown as a `StatusBadge` next to the title in `DetailHeader` using the `badge` prop, not as a `DetailField` in the sidebar.

```tsx
// ✅ Status as badge in header (follows harvest pattern)
<DetailHeader
  title={entity.name}
  icon={<LucideIcon>}
  badge={
    <StatusBadge
      label={statusLabelMap[entity.status]}
      variant={statusVariantMap[entity.status]}
    />
  }
/>

// ❌ Status as DetailField in sidebar
<DetailField label="Status" icon={Package}>
  {entity.active ? 'Ativo' : 'Inativo'}
</DetailField>

// ❌ Nome as DetailField in sidebar
<DetailField label="Nome" icon={Package}>
  {entity.name}
</DetailField>
```

These rules apply to **all modules** — inventory, crop, field, etc.

### Detail page header actions

- "Editar {Nome do Módulo}" button with `font-bold`, `variant="outline"`, `size="sm"`
- Actions popover with `showEdit={false}` (edit is already visible as a button)
- Pencil edit button (`size-7`, 28×28) in sidebar "Detalhes" section title

### Detail page linked fields

When a field references another entity, use the `href` prop on `DetailField` to make it a link:

```tsx
<DetailField
  label="Tipo de Cultura"
  icon={Wheat}
  href={`/crop-types/${cropType.id}`}
  data-testid="variety-crop-type-link"
>
  {cropType.name}
</DetailField>
```

### Related tables in detail pages

When a detail page shows a related entity table (e.g., crop-type shows varieties):

- Add a `+` button (`size-7`, Plus icon) in the section title to create related entities inline
- "Examinar mais itens" link below the table points to the sub-entity listing page
- Creating from the detail page opens a `StackedSheet` with the parent entity pre-selected and disabled

---

## Detail page (sub-entity / submodule)

Sub-entities in a 1-N relationship use a simplified layout without the grid or sidebar. See `design-system.md` → "Submodule detail pages" for the full visual specification.

```tsx
// src/app/(private)/<parent>/[id]/<sub-entity>/[subId]/page.tsx
import { ChevronLeft } from 'lucide-react'

export default async function <SubEntity>DetailPage(
  props: NextPageProps<{ id: string; subId: string }>,
) {
  const params = await props.params
  const parentId = params.id
  const subId = params.subId

  const [parent, subEntity] = await Promise.all([
    findParentById(parentId),
    findSubEntityById(subId),
  ])

  return (
    <>
      <div className="container flex h-full flex-col p-10" data-testid="<sub-entity>-detail-page">
        <header className="flex shrink-0 items-end justify-between pb-2">
          <div className="flex flex-col gap-2">
            <Link
              href={`/<parents>/${parentId}/<sub-entities>`}
              className="flex items-center gap-1 text-muted-foreground hover:text-foreground transition-colors"
              data-testid="<sub-entity>-back-link"
            >
              <ChevronLeft size={12} />
              <span className="text-[13px] font-medium uppercase">
                <Sub-Entities>
              </span>
            </Link>
            <h1 className="text-[28px] font-bold leading-none">{subEntity.name}</h1>
          </div>
          <div className="flex items-end gap-2">{/* actions */}</div>
        </header>

        <Separator className="mb-4" />

        <div className="mb-8 flex items-start">
          {/* Inline fields with vertical separators (mx-5) between them */}
        </div>

        {/* Sections (auditoria, etc.) */}
      </div>
    </>
  )
}
```

The back-link uses `ChevronLeft` + uppercase plural label (e.g. "VARIEDADES") linking to the sub-entity listing page. This provides a clear navigation path back to the listing without using breadcrumbs.

---

## Sub-entity listing page

When a sub-entity has its own listing page under the parent, use `DetailHeader` with breadcrumbs. Omit the `icon` prop and add a `Separator` after the header — this gives the listing a clean, compact look distinct from detail pages:

```tsx
// src/app/(private)/<parent>/[id]/<sub-entities>/page.tsx
import { Separator } from '@/shared/components/ui/separator'

<DetailHeader
  breadcrumbs={[{ href: '/<parents>', label: '<Parents>' }]}
  backHref={`/<parents>/${id}`}
  backLabel={parent.name}
  title="<Sub-Entities>"
  className="pb-2"
  actions={<Button>Adicionar <Sub-Entity></Button>}
/>

<Separator className="mb-4" />
```

**Key patterns:**
- No `icon` prop — listing pages don't show the placeholder icon container
- `className="pb-2"` — tighter spacing between header and separator
- `<Separator className="mb-4" />` — visual break before table/content
- `DetailHeader` uses `items-end` alignment when no icon, so action buttons align to the bottom of the title
- Creating a sub-entity from this page always pre-selects the parent entity (disabled select)

---

## Audit page

```tsx
// src/app/(private)/<domain>/[id]/audit/page.tsx
<DetailHeader
  breadcrumbs={[{ href: '/<entities>', label: '<Entities>' }]}
  backHref={`/<entities>/${id}`}
  backLabel={entity.name}
  title="Auditoria"
  icon={<LucideIcon>}
/>
```

For sub-entity audit pages, include the full breadcrumb chain:

```tsx
breadcrumbs={[
  { href: '/<parents>', label: '<Parents>' },
  { href: `/<parents>/${parentId}`, label: parent.name },
  { href: `/<parents>/${parentId}/<sub-entities>`, label: '<Sub-Entities>' },
]}
```

---

## Post-deletion redirect

After successfully deleting an entity, redirect to the corresponding listing page:

```tsx
// In delete form onSubmit:
if (res.ok) {
  closeDialog()
  router.push('/<entities>')  // or parent listing for sub-entities
}
```

For sub-entities, extract the parent ID from the URL path:

```tsx
const pathname = usePathname()
const parentId = pathname.split('/<parents>/')[1]?.split('/')[0]
router.push(`/<parents>/${parentId}/<sub-entities>`)
```

---

## loading.tsx

Loading pages use skeletons that mirror the layout of the actual page.

```tsx
import { Skeleton } from '@/shared/components/ui/skeleton'

export default function Loading() {
  return (
    <div className="container flex h-full flex-col p-10" data-testid="loading-skeleton">
      <div className="mb-10 flex justify-between">
        <Skeleton className="h-8 w-48" />
        <Skeleton className="h-10 w-40" />
      </div>
      <Skeleton className="h-64 w-full rounded-md" />
    </div>
  )
}
```

**Rules:**
- Skeleton layout must match the actual page design — same spacing, same proportions
- Use `Skeleton` from `shared/ui/` — never use spinners or plain text

---

## not-found.tsx

```tsx
import Link from 'next/link'

export default function NotFound() {
  return (
    <div
      className="container flex h-full flex-col items-center justify-center gap-4 p-10"
      data-testid="not-found"
    >
      <h2 className="text-2xl font-semibold"><Entity> não encontrada</h2>
      <p className="text-sm text-muted-foreground">
        A <entity> que você está procurando não existe.
      </p>
      <Link
        href="/<entities>"
        className="rounded-md bg-primary px-4 py-2 text-sm text-primary-foreground"
      >
        Voltar para <Entities>
      </Link>
    </div>
  )
}
```

**Rules:**
- Every domain route segment should have a `not-found.tsx`
- Text in Portuguese (pt-BR)
- Always include a link back to the list page

---

## Rules

- Pages that fetch data are async Server Components — data fetching happens on the server, never in `useEffect`
- Filters and listing state are controlled via `searchParams` — the URL is always the source of truth
- Use `Promise.all` to fetch independent data in parallel — never sequential awaits for unrelated data
- Route groups: `(public)` and `(private)` as default
- Sub-entities are nested under their parent route — never standalone top-level routes
- Every domain route must have `loading.tsx` with skeletons matching the page design
- Every domain route must have `not-found.tsx` for 404 handling
- Pages compose domain components — they don't render complex UI inline
- `data-testid` on page root containers for Playwright tests
- All user-facing text in Portuguese (pt-BR)
- After deletion, redirect to the listing page

---

## Anti-patterns

```tsx
// ❌ client-side data fetching
'use client'
export default function Page() {
  const [data, setData] = useState([])
  useEffect(() => { fetchData().then(setData) }, [])  // fetch in Server Component
}

// ❌ sequential fetches for independent data
const entity = await findEntityById(id)
const logs = await listAuditLogs({ entityId: id })  // use Promise.all

// ❌ spinner instead of skeleton
export default function Loading() {
  return <Spinner />  // use Skeleton matching page layout
}

// ❌ sub-entity as standalone route
src/app/(private)/varieties/page.tsx  // should be under /crop-types/[id]/varieties
// CORRECT:
src/app/(private)/crop-types/[id]/varieties/page.tsx

// ❌ using DetailLayout for sub-entity detail
<DetailLayout header={...} main={...} sidebar={...} />
// sub-entities use simplified layout without grid/sidebar

// ❌ English text in UI
<h2>Variety Not Found</h2>
// CORRECT:
<h2>Variedade não encontrada</h2>

// ❌ no redirect after deletion
if (res.ok) { closeDialog() }  // must also router.push to listing page

// ❌ Nome in sidebar detail fields (already in header title)
<DetailField label="Nome">{entity.name}</DetailField>

// ❌ Status as detail field in sidebar
<DetailField label="Status">{entity.active ? 'Ativo' : 'Inativo'}</DetailField>
// CORRECT: use badge prop in DetailHeader
<DetailHeader title={entity.name} badge={<StatusBadge label="Ativo" variant="active" />} />
```
