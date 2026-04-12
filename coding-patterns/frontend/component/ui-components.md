# Component — UI primitives

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## Actions popover

```tsx
// src/domains/<domain>/components/ui/<entity>-actions-popover.tsx
'use client'

import { useState } from 'react'
import { Ellipsis } from 'lucide-react'
import { usePathname, useRouter } from 'next/navigation'

import { Button } from '@/shared/components/ui/button'
import { Popover, PopoverContent, PopoverTrigger } from '@/shared/components/ui/popover'

import type { <Entity> } from '../../schemas'
import { use<Entity>DialogsStore } from '../../store'

type <Entity>ActionsPopoverProps = {
  entity: <Entity>
  showEdit?: boolean    // default true — set false when edit button is already visible
}

/** Contextual actions menu for a <entity> (edit, delete). */
export function <Entity>ActionsPopover({ entity, showEdit = true }: <Entity>ActionsPopoverProps) {
  const pathname = usePathname()
  const router = useRouter()
  const { setDeleteDialogOpen } = use<Entity>DialogsStore()
  const [open, setOpen] = useState(false)

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="outline" size="icon" className="size-7" data-testid={`<entity>-actions-${entity.id}`}>
          <Ellipsis />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-56 p-0.5" align="end">
        {showEdit && (
          <Button
            variant="ghost"
            className="w-full justify-start"
            onClick={() => { setOpen(false); router.push(`${pathname}?edit-<entity>=${entity.id}`) }}
            data-testid={`<entity>-edit-btn-${entity.id}`}
          >
            Editar <entity>
          </Button>
        )}
        <Button
          variant="ghost"
          className="w-full justify-start text-destructive"
          onClick={() => { setOpen(false); setDeleteDialogOpen(entity.id) }}
          data-testid={`<entity>-delete-btn-${entity.id}`}
        >
          Excluir <entity>
        </Button>
      </PopoverContent>
    </Popover>
  )
}
```

**Key points:**
- Trigger button: `size-7` (28×28)
- Content padding: `p-0.5` (2px)
- `showEdit` prop: set `false` on detail pages where edit button is already visible
- Destructive actions use `text-destructive` and always open a confirmation dialog
- All text in Portuguese

---
