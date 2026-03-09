# Loading States

Loading states provide visual feedback while content is being fetched. The primary pattern is **skeleton screens** that replicate the layout of the actual page, preventing layout shift and giving users a sense of the content structure.

---

## Route-level loading (`loading.tsx`)

Next.js App Router automatically wraps `page.tsx` in a Suspense boundary when `loading.tsx` exists in the same route segment. The loading UI is shown instantly while the page's async data fetching completes.

```tsx
import { Skeleton } from '@/shared/components/ui/skeleton'

export default function Loading() {
  return (
    <div data-testid="loading-skeleton" className="space-y-6">
      {/* Page header — matches the real page header layout */}
      <div className="flex items-center justify-between">
        <Skeleton className="h-8 w-48" />       {/* Title */}
        <Skeleton className="h-10 w-32" />       {/* Action button */}
      </div>

      {/* Table skeleton — matches the real table layout */}
      <div className="space-y-3">
        <Skeleton className="h-10 w-full" />     {/* Table header */}
        <Skeleton className="h-12 w-full" />     {/* Row 1 */}
        <Skeleton className="h-12 w-full" />     {/* Row 2 */}
        <Skeleton className="h-12 w-full" />     {/* Row 3 */}
        <Skeleton className="h-12 w-full" />     {/* Row 4 */}
        <Skeleton className="h-12 w-full" />     {/* Row 5 */}
      </div>

      {/* Pagination skeleton */}
      <div className="flex justify-end">
        <Skeleton className="h-10 w-64" />
      </div>
    </div>
  )
}
```

**Rules:**
- The skeleton **must replicate the layout of the real page** — same structure, same spacing, same approximate dimensions
- Use the `Skeleton` component from shadcn/ui (`shared/components/ui/skeleton`)
- Always include `data-testid="loading-skeleton"` on the root container
- Comment each skeleton to indicate what it represents (title, table, button, etc.)
- No animated text like "Loading..." — skeletons with pulse animation are the standard

---

## Skeleton design principles

| Principle | Rule |
|-----------|------|
| **Mirror the layout** | Match the real page's structure exactly — same grid, same flex layout, same gaps |
| **Match dimensions** | Width and height of each skeleton should approximate the real content |
| **No layout shift** | When real content loads, it should occupy the same space as the skeleton |
| **Keep it simple** | Don't replicate every tiny detail — focus on the major content blocks |

---

## Placement

Every route that fetches data server-side must have a `loading.tsx`:

```
src/app/(private)/
  dashboard/
    loading.tsx                 ← dashboard skeleton
    <domain>/
      loading.tsx               ← list page skeleton
      [id]/
        loading.tsx             ← detail page skeleton
```

---

## Action loading (mutations)

For mutations triggered by user actions (create, edit, delete), use the `toast.loading` pattern already defined in `action.md`:

```tsx
const toastId = toast.loading('Creating animal...')
const result = await createAnimalAction(data)
renderToast(result, toastId)
```

This is handled by server actions — no skeleton needed for mutations.

---

## What NOT to do

- Never use a generic spinner as the only loading indicator for page loads — always use skeletons
- Never show an empty page while loading — the skeleton should appear instantly
- Never create skeletons that don't match the real page layout — this causes layout shift
- Never use `useEffect` + loading state for initial page data — use Server Components + `loading.tsx`
