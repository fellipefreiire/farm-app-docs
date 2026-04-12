# Component — Listing Filters

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Listing pages use shared filter components that sync state with URL searchParams. The URL is the source of truth — each filter component reads its value from the URL and pushes changes via `useFilterNavigation`.

## Layout: TableToolbar

Container for filter components above a data table.

```tsx
// src/shared/components/table-toolbar.tsx
import type { ReactNode } from 'react'

type TableToolbarProps = {
  children: ReactNode
}

/** Layout container for filter components above a data table. */
export function TableToolbar({ children }: TableToolbarProps) {
  return (
    <div className="mb-6 flex items-center gap-2" data-testid="table-toolbar">
      {children}
    </div>
  )
}
```

## SearchInput

Debounced text input (300ms) that syncs with `?query=` by default. Resets page on change.

```tsx
// src/shared/components/search-input.tsx
<SearchInput placeholder="Pesquisar tipos de cultura..." />
<SearchInput placeholder="Pesquisar..." paramName="search" />  // custom param name
```

- Default `paramName`: `query`
- `data-testid`: `${paramName}-search-input`
- Includes search icon (magnifying glass) on the left

## FilterSelect

Select dropdown that syncs with a URL param. Selecting the placeholder ("Todos", "Status", etc.) clears the param.

```tsx
// src/shared/components/filter-select.tsx
<FilterSelect
  paramName="status"
  placeholder="Status"
  options={[
    { label: 'Ativo', value: 'active' },
    { label: 'Arquivado', value: 'archived' },
  ]}
/>
```

- `data-testid`: `${paramName}-filter-select`
- First option is always the placeholder (shows all, clears filter)

## SortableHeader

Clickable column header that toggles sort direction. Used inside column definitions.

```tsx
// src/shared/components/sortable-header.tsx
// In column definitions:
{
  header: () => <SortableHeader label="Nome" sortKey="name" />,
  accessorKey: 'name',
}
```

- Updates `?sort=<sortKey>&order=asc|desc`
- Shows arrow icon: `ArrowUpDown` (inactive), `ArrowUp` (asc), `ArrowDown` (desc)
- Uses `setParams` (not `setParam`) to update sort + order in a single navigation
- `data-testid`: `sort-${sortKey}`

## PaginationControls

Prev/next buttons with page indicator. Hidden when `totalPages <= 1`.

```tsx
// src/shared/components/pagination-controls.tsx
<PaginationControls
  currentPage={data.meta.page}
  totalPages={Math.ceil(data.meta.total / data.meta.perPage)}
/>
```

- `data-testid`: `pagination-controls`, `pagination-prev`, `pagination-next`
- Page 1 removes the `page` param from URL (cleaner URLs)
- Text: "Página X de Y"

## Composition in pages

```tsx
// Minimal — just search
<TableToolbar>
  <SearchInput placeholder="Pesquisar..." />
</TableToolbar>
<EntityTable data={data.data} />
<PaginationControls currentPage={...} totalPages={...} />

// With filters
<TableToolbar>
  <SearchInput placeholder="Pesquisar..." />
  <FilterSelect paramName="status" placeholder="Status" options={[...]} />
</TableToolbar>

// With tabs (below toolbar)
<TableToolbar>
  <SearchInput placeholder="Pesquisar..." />
</TableToolbar>
<TableTabs tabs={TABLE_TABS} />
```

**Rules:**
- All filter state lives in URL searchParams — never in React state or Zustand
- Filter components are shared (`src/shared/components/`) — not domain-specific
- Changing any filter resets page to 1 automatically
- `SortableHeader` `sortKey` values must match the backend's `z.enum([...])` for the `sort` query param
- Pages read searchParams in the Server Component and pass them to the API function — filter components only update the URL

---
