# Component — Dialogs

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Dialogs are thin wrappers around forms. They own the open/close state via the store.

```tsx
// src/domains/<domain>/components/dialogs/delete-<entity>-dialog.tsx
'use client'

import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/shared/components/ui/dialog'

import { use<Entity>DialogsStore } from '../../store'
import { Delete<Entity>Form } from '../forms/delete-<entity>-form'

/** Dialog for confirming <entity> deletion. */
export function Delete<Entity>Dialog() {
  const { deleteDialogOpen, setDeleteDialogOpen } = use<Entity>DialogsStore()

  function closeDialog() { setDeleteDialogOpen(null) }

  return (
    <Dialog open={!!deleteDialogOpen} onOpenChange={() => setDeleteDialogOpen(null)}>
      <DialogContent data-testid="<entity>-delete-dialog">
        <DialogHeader>
          <DialogTitle>Excluir <Entity></DialogTitle>
        </DialogHeader>
        {deleteDialogOpen && (
          <Delete<Entity>Form entityId={deleteDialogOpen} closeDialog={closeDialog} />
        )}
      </DialogContent>
    </Dialog>
  )
}
```

---
