# Component — Table

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## Columns with image placeholder

```tsx
// src/domains/<domain>/components/table/<entity>-columns.tsx
'use client'

import { <Icon> } from 'lucide-react'
import type { CellContext, ColumnDef } from '@tanstack/react-table'

import { SortableHeader } from '@/shared/components/sortable-header'

import type { <Entity> } from '../../schemas'
import { <Entity>ActionsPopover } from '../ui/<entity>-actions-popover'

export const <entity>Columns: ColumnDef<<Entity>>[] = [
  {
    id: 'image',
    cell: ImageCell,
    meta: { className: 'w-10 pr-0' },
  },
  {
    header: () => <SortableHeader label="Nome" sortKey="name" />,
    accessorKey: 'name',
    meta: { grow: true },
  },
  {
    header: () => <SortableHeader label="Criado em" sortKey="createdAt" />,
    accessorKey: 'createdAt',
    cell: ({ row }) =>
      new Intl.DateTimeFormat('pt-BR', { dateStyle: 'medium' }).format(
        row.original.createdAt,
      ),
  },
  {
    id: 'Actions',
    cell: ActionsCell,
    meta: { isLast: true },
  },
]

function ImageCell() {
  return (
    <div
      className="bg-muted flex size-10 items-center justify-center rounded-[6px] border border-transparent group-hover/row:border-placeholder-icon"
      data-testid="<entity>-image"
    >
      <<Icon> className="text-placeholder-icon size-4" />
    </div>
  )
}

function ActionsCell(ctx: CellContext<<Entity>, unknown>) {
  const entity = ctx.row.original
  return <<Entity>ActionsPopover entity={entity} />
}
```

## Table component

```tsx
// src/domains/<domain>/components/table/<entity>-table.tsx
'use client'

import { useRouter } from 'next/navigation'
import { getCoreRowModel, useReactTable } from '@tanstack/react-table'

import { DataTable } from '@/shared/components/ui/data-table'

import type { <Entity> } from '../../schemas'
import { <entity>Columns } from './<entity>-columns'

type <Entity>TableProps = {
  data: <Entity>[]
}

/** Displays <entity>s in a sortable table. */
export function <Entity>Table({ data }: Readonly<<Entity>TableProps>) {
  const router = useRouter()

  const table = useReactTable({
    data,
    columns: <entity>Columns,
    getCoreRowModel: getCoreRowModel(),
  })

  function onRowClick(entity: <Entity>) {
    router.push(`/<entities>/${entity.id}`)
  }

  return <DataTable table={table} onRowClick={onRowClick} />
}
```

For sub-entity tables, accept the parent ID for correct navigation:

```tsx
type VarietyTableProps = {
  data: Variety[]
  cropTypeId: string  // parent ID for nested route
}

function onRowClick(variety: Variety) {
  router.push(`/crop-types/${cropTypeId}/varieties/${variety.id}`)
}
```

---
