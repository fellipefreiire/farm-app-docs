# Page — Post-deletion redirect

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
