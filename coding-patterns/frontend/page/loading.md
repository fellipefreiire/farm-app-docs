# Page — loading.tsx

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
