# Shared — Global types

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
// src/shared/types/global.d.ts
type SearchParams = Promise<Record<string, string | string[] | undefined>>

type NextPageProps<TParams extends Record<string, string> = never> = {
  searchParams: SearchParams
} & ([TParams] extends [never]
  ? Record<never, never>
  : { params: Promise<TParams> })
```

**Usage:**

```tsx
// List page — only searchParams
export default async function EntitiesPage(props: NextPageProps) { ... }

// Detail page — searchParams + route params
export default async function EntityDetailPage(
  props: NextPageProps<{ id: string }>,
) { ... }
```

`NextPageProps` wraps Next.js App Router page props with proper typing for `searchParams` and optional `params`.

---
