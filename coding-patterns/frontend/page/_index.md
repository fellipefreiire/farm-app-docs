# Frontend Page Pattern — Index

Navigation hub for farm-app frontend page patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree** —
layer-scoping and context-mode sandboxing only pay off if this discipline is respected.

## Files

| Variant | File | When to read |
|---|---|---|
| List page | [list-page.md](list-page.md) | Implementing a top-level entity listing page |
| Detail page (top-level) | [detail-page.md](detail-page.md) | Implementing a top-level entity detail page |
| Detail page (sub-entity) | [detail-page-sub.md](detail-page-sub.md) | Detail page for a sub-entity / submodule |
| Sub-entity listing | [sub-entity-listing.md](sub-entity-listing.md) | Listing of sub-entities inside a parent detail |
| Audit page | [audit-page.md](audit-page.md) | Audit log page for an entity |
| Post-deletion redirect | [post-delete-redirect.md](post-delete-redirect.md) | Redirect logic after entity deletion |
| loading.tsx | [loading.md](loading.md) | Writing a Next.js loading skeleton |
| not-found.tsx | [not-found.md](not-found.md) | Writing a Next.js not-found page |

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
