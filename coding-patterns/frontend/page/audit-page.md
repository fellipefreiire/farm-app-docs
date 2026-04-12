# Page — Audit

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
