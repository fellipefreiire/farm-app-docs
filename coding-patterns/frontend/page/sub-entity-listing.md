# Page — Sub-entity listing

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

When a sub-entity has its own listing page under the parent, use `DetailHeader` with breadcrumbs. Omit the `icon` prop and add a `Separator` after the header — this gives the listing a clean, compact look distinct from detail pages:

```tsx
// src/app/(private)/<parent>/[id]/<sub-entities>/page.tsx
import { Separator } from '@/shared/components/ui/separator'

<DetailHeader
  breadcrumbs={[{ href: '/<parents>', label: '<Parents>' }]}
  backHref={`/<parents>/${id}`}
  backLabel={parent.name}
  title="<Sub-Entities>"
  className="pb-2"
  actions={<Button>Adicionar <Sub-Entity></Button>}
/>

<Separator className="mb-4" />
```

**Key patterns:**
- No `icon` prop — listing pages don't show the placeholder icon container
- `className="pb-2"` — tighter spacing between header and separator
- `<Separator className="mb-4" />` — visual break before table/content
- `DetailHeader` uses `items-end` alignment when no icon, so action buttons align to the bottom of the title
- Creating a sub-entity from this page always pre-selects the parent entity (disabled select)

---
