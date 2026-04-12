# Page — not-found.tsx

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
