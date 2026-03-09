# Page Pattern

Pages are Server Components that compose domain components. Pages that fetch data must be async. They live in the Next.js App Router file structure and handle routing, data loading, search params, and error/loading states.

---

## File locations

```
src/app/
  (public)/                          ← unauthenticated routes (sign-in, sign-up)
    page.tsx
  (private)/                         ← authenticated routes
    dashboard/
      <domain>/
        page.tsx                     ← list page
        [id]/
          page.tsx                   ← detail page
        loading.tsx                  ← loading skeleton
        not-found.tsx                ← 404 page
      layout.tsx                     ← shared layout (sidebar, nav)
  layout.tsx                         ← root layout (fonts, providers, Toaster)
  not-found.tsx                      ← global 404
```

---

## Route groups

Use `(public)` and `(private)` as the default route groups. Add more groups as needed by the project (e.g. `(onboarding)`, `(admin)`).

```
src/app/
  (public)/         ← no auth required
  (private)/        ← auth required (layout checks authentication)
```

---

## layout.tsx vs template.tsx

- **layout.tsx** — persists across navigations within the same segment. Use for shared UI that should not remount (sidebar, navigation, providers). State is preserved between page transitions.
- **template.tsx** — re-renders on every navigation. Use when you need fresh state on each page visit (e.g. auth check on every navigation, analytics tracking).

When in doubt, read the [Next.js documentation](https://nextjs.org/docs/app/api-reference/file-conventions/layout) to verify which one fits the use case.

---

## List page

```tsx
// src/app/(private)/dashboard/<domain>/page.tsx
import { list<Entity>s } from '@/domains/<domain>/api'
import { <Entity>Table } from '@/domains/<domain>/components'
import { getSearchParam } from '@/shared/utils/search-params'

export default async function <Entity>sPage(props: NextPageProps) {
  const searchParams = await props.searchParams

  const page = getSearchParam(searchParams.page)
  const search = getSearchParam(searchParams.search)
  const active = getSearchParam(searchParams.active)

  const data = await list<Entity>s({
    page: page ? Number(page) : undefined,
    search,
    active: active !== undefined ? active === 'true' : undefined,
  })

  return (
    <div className="container flex h-full flex-col p-10" data-testid="<entity>s-page">
      <div className="mb-10 flex justify-between">
        <h1 className="text-2xl font-bold"><Entity>s</h1>
      </div>
      <<Entity>Table data={data.data} />
    </div>
  )
}
```

**Rules:**
- Pages that fetch data are async Server Components — they fetch data directly, no `useEffect`
- Filters come from `searchParams` — the URL is the source of truth for listing state
- Parse searchParams using `getSearchParam()` from shared utils — never access raw values directly
- Pass parsed data to domain components — pages compose, they don't render complex UI

---

## Detail page

```tsx
// src/app/(private)/dashboard/<domain>/[id]/page.tsx
import { find<Entity>ById } from '@/domains/<domain>/api'

export default async function <Entity>DetailPage(
  props: NextPageProps<{ id: string }>,
) {
  const params = await props.params
  const id = params.id

  const entity = await find<Entity>ById(id)

  return (
    <div className="container flex h-full flex-col p-10" data-testid="<entity>-detail-page">
      <header className="flex items-start justify-between pb-6">
        <h1 className="text-2xl font-bold">{entity.name}</h1>
      </header>
      <main className="grid grid-cols-6 gap-12">
        <div className="col-span-4">
          {/* Main content */}
        </div>
        <div className="col-span-2">
          {/* Sidebar details */}
        </div>
      </main>
    </div>
  )
}
```

When the detail page needs multiple data sources, fetch them in parallel:

```tsx
const [entity, relatedData] = await Promise.all([
  find<Entity>ById(id),
  listRelatedItems({ entityId: id }),
])
```

---

## loading.tsx

Loading pages use skeletons that mirror the layout of the actual page. This gives the user a visual preview of the content structure while data loads.

```tsx
// src/app/(private)/dashboard/<domain>/loading.tsx
import { Skeleton } from '@/shared/components/ui/skeleton'

export default function Loading() {
  return (
    <div className="container flex h-full flex-col p-10">
      <div className="mb-10 flex justify-between">
        <Skeleton className="h-8 w-48" />
        <Skeleton className="h-10 w-40" />
      </div>
      <div className="flex flex-col gap-2">
        {Array.from({ length: 10 }).map((_, i) => (
          <Skeleton key={i} className="h-12 w-full" />
        ))}
      </div>
    </div>
  )
}
```

**Rules:**
- Skeleton layout must match the actual page design — same spacing, same proportions
- Use `Skeleton` from `shared/ui/` — never use spinners or plain text
- Keep it simple — approximate the shape, don't replicate every detail

---

## not-found.tsx

Displayed when the page route doesn't exist or when the backend returns 404 (via `notFound()` in `handleHttpError`).

```tsx
// src/app/(private)/dashboard/<domain>/not-found.tsx
import Link from 'next/link'

import { Button } from '@/shared/components/ui/button'

export default function NotFound() {
  return (
    <div
      className="flex h-full flex-col items-center justify-center gap-4"
      data-testid="<entity>-not-found"
    >
      <h1 className="text-2xl font-bold"><Entity> not found</h1>
      <p className="text-gray-500">
        The <entity> you are looking for does not exist or has been removed.
      </p>
      <Button asChild>
        <Link href="/dashboard/<entities>">Back to <entities></Link>
      </Button>
    </div>
  )
}
```

**Rules:**
- Every domain route segment should have a `not-found.tsx`
- API functions call `notFound()` on 404 responses (via `handleHttpError`) — this triggers the nearest `not-found.tsx`
- Always include a link back to the list page

---

## Rules

- Pages that fetch data are async Server Components — data fetching happens on the server, never in `useEffect`. Pages without data fetching do not need to be async
- Filters and listing state are controlled via `searchParams` — the URL is always the source of truth
- Use `Promise.all` to fetch independent data in parallel — never sequential awaits for unrelated data
- Route groups: `(public)` and `(private)` as default — add more as needed
- Use `layout.tsx` for persistent shared UI (sidebar, nav) — use `template.tsx` when fresh state is needed on every navigation
- Every domain route must have `loading.tsx` with skeletons matching the page design
- Every domain route must have `not-found.tsx` for 404 handling
- Pages compose domain components — they don't render complex UI inline
- `data-testid` on page root containers for Playwright tests

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

// ❌ missing not-found.tsx
// API returns 404 → generic Next.js error page
// always add not-found.tsx per domain route

// ❌ hardcoded filter state
const [active, setActive] = useState(true)  // use searchParams

// ❌ complex UI inline in page
export default async function Page() {
  return (
    <div>
      <table>
        <tr>...</tr>  // extract to domain component
      </table>
    </div>
  )
}

// ❌ wrong route group
src/app/dashboard/sign-in/page.tsx  // should be in (public)

// ❌ using layout.tsx when template.tsx is needed
// auth check in layout won't re-run on navigation within the segment
// use template.tsx for checks that must run on every navigation
```
