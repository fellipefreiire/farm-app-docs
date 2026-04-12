# Page — Detail (top-level entity)

> Part of the farm-app frontend page pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Top-level entities use `DetailLayout` with a 12-column grid (9/3 split), `DetailHeader` with breadcrumbs, placeholder image, and actions.

```tsx
// src/app/(private)/<domain>/[id]/page.tsx
import { DetailLayout } from '@/shared/components/detail-layout'
import { DetailHeader } from '@/shared/components/detail-header'
import { DetailSection } from '@/shared/components/detail-section'
import { DetailField } from '@/shared/components/detail-field'

export default async function <Entity>DetailPage(
  props: NextPageProps<{ id: string }>,
) {
  const params = await props.params
  const searchParams = await props.searchParams
  const id = params.id

  const entity = await find<Entity>ById(id)

  return (
    <>
      <DetailLayout
        header={
          <DetailHeader
            backHref="/<entities>"
            backLabel="<Entities>"
            title={entity.name}
            icon={<LucideIcon>}
            actions={
              <>
                <Button variant="outline" size="sm" className="font-bold" asChild>
                  <Link href={`/<entities>/${id}?edit-<entity>=${id}`}>Editar <Entity></Link>
                </Button>
                <<Entity>ActionsPopover entity={entity} showEdit={false} />
              </>
            }
          />
        }
        main={/* 9 columns: tables, audit, etc. */}
        sidebar={
          <DetailSection
            title="Detalhes"
            actions={
              <Button variant="outline" size="icon" className="size-7" asChild>
                <Link href={`/<entities>/${id}?edit-<entity>=${id}`}><Pencil /></Link>
              </Button>
            }
          >
            <div className="flex flex-col gap-4">
              {/* Never include Nome (already in header title) or Status (use badge prop in DetailHeader) */}
              <DetailField label="Criado em" icon={Calendar}>{formattedDate}</DetailField>
            </div>
          </DetailSection>
        }
      />

      {isEditing && <Edit<Entity>Sheet entity={entity} />}
      <Delete<Entity>Dialog />
    </>
  )
}
```

## Detail page sidebar rules

**Never show in detail fields:**
- **Nome** — already displayed in the `DetailHeader` title. Repeating it in the sidebar is redundant.
- **Status** — must be shown as a `StatusBadge` next to the title in `DetailHeader` using the `badge` prop, not as a `DetailField` in the sidebar.

```tsx
// ✅ Status as badge in header (follows harvest pattern)
<DetailHeader
  title={entity.name}
  icon={<LucideIcon>}
  badge={
    <StatusBadge
      label={statusLabelMap[entity.status]}
      variant={statusVariantMap[entity.status]}
    />
  }
/>

// ❌ Status as DetailField in sidebar
<DetailField label="Status" icon={Package}>
  {entity.active ? 'Ativo' : 'Inativo'}
</DetailField>

// ❌ Nome as DetailField in sidebar
<DetailField label="Nome" icon={Package}>
  {entity.name}
</DetailField>
```

These rules apply to **all modules** — inventory, crop, field, etc.

## Detail page header actions

- "Editar {Nome do Módulo}" button with `font-bold`, `variant="outline"`, `size="sm"`
- Actions popover with `showEdit={false}` (edit is already visible as a button)
- Pencil edit button (`size-7`, 28×28) in sidebar "Detalhes" section title

## Detail page linked fields

When a field references another entity, use the `href` prop on `DetailField` to make it a link:

```tsx
<DetailField
  label="Tipo de Cultura"
  icon={Wheat}
  href={`/crop-types/${cropType.id}`}
  data-testid="variety-crop-type-link"
>
  {cropType.name}
</DetailField>
```

## Related tables in detail pages

When a detail page shows a related entity table (e.g., crop-type shows varieties):

- Add a `+` button (`size-7`, Plus icon) in the section title to create related entities inline
- "Examinar mais itens" link below the table points to the sub-entity listing page
- Creating from the detail page opens a `StackedSheet` with the parent entity pre-selected and disabled

---
