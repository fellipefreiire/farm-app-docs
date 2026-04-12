# Page — Detail (sub-entity / submodule)

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
