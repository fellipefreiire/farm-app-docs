# Page Pattern

Pages are Server Components that compose domain components. Pages that fetch data must be async. They live in the Next.js App Router file structure and handle routing, data loading, search params, and error/loading states.

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
import {
  <Entity>Table,
  Create<Entity>Sheet,
  Edit<Entity>Sheet,
  Delete<Entity>Dialog,
} from '@/domains/<domain>/components'
import { EmptyState } from '@/shared/components/empty-state'
import { PaginationControls } from '@/shared/components/pagination-controls'
import { SearchInput } from '@/shared/components/search-input'
import { TableToolbar } from '@/shared/components/table-toolbar'
import { Button } from '@/shared/components/ui/button'
import { getSearchParam } from '@/shared/utils/search-params'

export default async function <Entity>sPage(props: NextPageProps) {
  const searchParams = await props.searchParams

  const page = getSearchParam(searchParams.page)
  const query = getSearchParam(searchParams.query)
  const sort = getSearchParam(searchParams.sort)
  const order = getSearchParam(searchParams.order)

  const data = await list<Entity>s({
    page: page ? Number(page) : undefined,
    query,
    sort,
    order,
  })

  return (
    <>
      <div className="container flex h-full flex-col p-10" data-testid="<entity>s-page">
        <div className="mb-10 flex justify-between">
          <h1 className="text-2xl font-bold"><Entity>s</h1>
          <Button asChild>
            <Link href="/<entities>?create=<entity>" data-testid="<entity>-create-button">
              <Plus />
              Adicionar <Entity>
            </Link>
          </Button>
        </div>

        <TableToolbar>
          <SearchInput placeholder="Pesquisar <entities>..." />
        </TableToolbar>

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

**Rules:**
- Pages that fetch data are async Server Components — they fetch data directly, no `useEffect`
- Filters come from `searchParams` — the URL is the source of truth for listing state
- Parse searchParams using `getSearchParam()` from shared utils — never access raw values directly
- Pass `query`, `sort`, `order`, `page` from searchParams to the API function
- Use `TableToolbar` + `SearchInput` for search, `FilterSelect` for dropdowns, `PaginationControls` for pagination
- `SortableHeader` goes in the column definitions, not in the page — the page only passes sort params to the API
- Pass parsed data to domain components — pages compose, they don't render complex UI

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
              <DetailField label="Nome" icon={<LucideIcon>}>{entity.name}</DetailField>
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
            <div className="flex items-center gap-2">
              <Rows4 size={12} className="text-muted-foreground" />
              <span className="text-[13px] font-medium uppercase text-muted-foreground">
                <Sub-Entity Type>
              </span>
            </div>
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

---

## Sub-entity listing page

When a sub-entity has its own listing page under the parent, use `DetailHeader` with breadcrumbs:

```tsx
// src/app/(private)/<parent>/[id]/<sub-entities>/page.tsx
<DetailHeader
  breadcrumbs={[{ href: '/<parents>', label: '<Parents>' }]}
  backHref={`/<parents>/${id}`}
  backLabel={parent.name}
  title="<Sub-Entities>"
  icon={<LucideIcon>}
  actions={<Button>Adicionar <Sub-Entity></Button>}
/>
```

Creating a sub-entity from this page always pre-selects the parent entity (disabled select).

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
```
