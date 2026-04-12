# Page — List

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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

## Status filtering — always use `TableTabs`

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

## List page filter architecture

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

## PreserveParamsLink

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

## ViewToggle (list/card views)

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
