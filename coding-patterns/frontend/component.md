# Component Pattern

Domain components are organized by type in subdirectories. They consume server actions, stores, and shared UI primitives. All interactive elements must include `data-testid` attributes for Playwright tests.

---

## File locations

```
src/domains/<domain>/components/
  <group>/                            ← subdirectory per category (see grouping rules below)
    <entity>-<name>.tsx
  index.ts                            ← barrel export
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), components move under the subdomain: `src/domains/<domain>/<subdomain>/components/`.

### Grouping rules

Components are grouped into subdirectories when **two or more components share the same category**. Groups are not fixed — they emerge from the domain's needs. Name the directory after what the components have in common.

Common groups that appear in most domains:

```
table/      ← columns + table component
forms/      ← create, edit, confirmation forms
dialogs/    ← dialog wrappers (archive, delete, etc.)
ui/         ← small domain-specific UI (badges, popovers, cards)
containers/ ← composition components (detail view, edit view)
```

These are examples, not mandatory. A domain might have `charts/`, `filters/`, `wizards/`, or any other grouping that makes sense. The rule is: **when components belong to the same category, create a directory with a descriptive name.**

A component that doesn't fit any group stays at the `components/` root.

---

## data-testid convention

All interactive elements must include `data-testid` for Playwright tests. Never use CSS selectors or text content in tests.

**Naming format:** `<entity>-<component>-<element>`

```tsx
// Buttons
<Button data-testid="<entity>-create-submit">Create</Button>
<Button data-testid="<entity>-edit-submit">Save</Button>
<Button data-testid="<entity>-delete-confirm">Delete</Button>
<Button data-testid="<entity>-archive-confirm">Archive</Button>

// Form fields
<InputField data-testid="<entity>-name-input" ... />
<SelectField data-testid="<entity>-status-select" ... />

// Table rows — use dynamic IDs
<TableRow data-testid={`<entity>-row-${entity.id}`} ... />

// Actions
<Button data-testid={`<entity>-actions-${entity.id}`} ... />

// Dialogs
<DialogContent data-testid="<entity>-archive-dialog" ... />
<DialogContent data-testid="<entity>-delete-dialog" ... />
```

---

## Table

### Columns

```tsx
// src/domains/<domain>/components/table/<entity>-columns.tsx
'use client'

import type { CellContext, ColumnDef } from '@tanstack/react-table'

import type { <Entity> } from '../../schemas'
import { <Entity>ActionsPopover } from '../ui/<entity>-actions-popover'

export const <entity>Columns: ColumnDef<<Entity>>[] = [
  {
    header: 'Name',
    accessorKey: 'name',
    meta: { grow: true },
  },
  {
    header: 'Status',
    cell: ({ row }) => {
      const entity = row.original
      return (
        <span data-testid={`<entity>-status-${entity.id}`}>
          {entity.active ? 'Active' : 'Archived'}
        </span>
      )
    },
  },
  {
    id: 'Actions',
    cell: ActionsCell,
    meta: { isLast: true },
  },
]

function ActionsCell(ctx: CellContext<<Entity>, unknown>) {
  const entity = ctx.row.original
  return <<Entity>ActionsPopover entity={entity} />
}
```

### Table component

```tsx
// src/domains/<domain>/components/table/<entity>-table.tsx
'use client'

import { DataTable } from '@/shared/components/data-table'
import { getCoreRowModel, useReactTable } from '@tanstack/react-table'

import type { <Entity> } from '../../schemas'
import { <entity>Columns } from './<entity>-columns'

type <Entity>TableProps = {
  data: <Entity>[]
}

/** Displays a list of <entity>s in a table with sortable columns. */
export function <Entity>Table({ data }: Readonly<<Entity>TableProps>) {
  const table = useReactTable({
    data,
    columns: <entity>Columns,
    getCoreRowModel: getCoreRowModel(),
  })

  return <DataTable table={table} />
}
```

---

## Forms

### Create form

```tsx
// src/domains/<domain>/components/forms/create-<entity>-form.tsx
'use client'

import { usePathname, useRouter } from 'next/navigation'
import { useForm } from 'react-hook-form'
import { toast } from 'sonner'

import { InputField } from '@/shared/components/fields/input-field'
import { Button } from '@/shared/components/ui/button'
import { Form } from '@/shared/components/ui/form'
import { renderToast } from '@/shared/utils/toast'
import { zodResolver } from '@hookform/resolvers/zod'

import { create<Entity>Action } from '../../actions'
import {
  type Create<Entity>Request,
  create<Entity>Schema,
} from '../../schemas'

/** Form for creating a new <entity>. */
export function Create<Entity>Form() {
  const pathname = usePathname()
  const { push } = useRouter()

  const form = useForm<Create<Entity>Request>({
    resolver: zodResolver(create<Entity>Schema),
    defaultValues: {
      name: '',
    },
  })

  async function onSubmit(data: Create<Entity>Request) {
    const toastId = toast.loading('Creating <entity>...')

    const res = await create<Entity>Action(data)

    renderToast(res.ok, res.message, toastId)

    if (res.ok) {
      push(pathname)
    }
  }

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit(onSubmit)}
        className="flex h-full flex-col gap-4 p-4"
        data-testid="<entity>-create-form"
      >
        <InputField
          control={form.control}
          label="Name"
          name="name"
          data-testid="<entity>-name-input"
        />
        <div className="mt-auto ml-auto flex gap-4">
          <Button variant="outline" data-testid="<entity>-create-cancel">
            Cancel
          </Button>
          <Button type="submit" data-testid="<entity>-create-submit">
            Create <entity>
          </Button>
        </div>
      </form>
    </Form>
  )
}
```

### Edit form

```tsx
// src/domains/<domain>/components/forms/edit-<entity>-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { toast } from 'sonner'

import { InputField } from '@/shared/components/fields/input-field'
import { Button } from '@/shared/components/ui/button'
import { Form } from '@/shared/components/ui/form'
import { renderToast } from '@/shared/utils/toast'
import { zodResolver } from '@hookform/resolvers/zod'

import { edit<Entity>Action } from '../../actions'
import {
  type <Entity>,
  type Edit<Entity>Request,
  edit<Entity>Schema,
} from '../../schemas'

type Edit<Entity>FormProps = {
  entity: <Entity>
  onSuccess?: () => void
  onCancel?: () => void
}

/** Form for editing an existing <entity>. */
export function Edit<Entity>Form({
  entity,
  onSuccess,
  onCancel,
}: Edit<Entity>FormProps) {
  const form = useForm<Edit<Entity>Request>({
    resolver: zodResolver(edit<Entity>Schema),
    defaultValues: {
      name: entity.name,
    },
  })

  async function onSubmit(data: Edit<Entity>Request) {
    const toastId = toast.loading('Saving <entity>...')

    const res = await edit<Entity>Action(entity.id, data)

    renderToast(res.ok, res.message, toastId)

    if (res.ok) {
      onSuccess?.()
    }
  }

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit(onSubmit)}
        className="flex h-full flex-col gap-4 p-4"
        data-testid="<entity>-edit-form"
      >
        <InputField
          control={form.control}
          label="Name"
          name="name"
          data-testid="<entity>-name-input"
        />
        <div className="mt-auto ml-auto flex gap-4">
          <Button
            type="button"
            variant="outline"
            onClick={onCancel}
            data-testid="<entity>-edit-cancel"
          >
            Cancel
          </Button>
          <Button type="submit" data-testid="<entity>-edit-submit">
            Save changes
          </Button>
        </div>
      </form>
    </Form>
  )
}
```

### Confirmation form (archive/delete)

Confirmation forms are simple — no fields, just a message and action buttons. They call the server action directly.

```tsx
// src/domains/<domain>/components/forms/archive-<entity>-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { toast } from 'sonner'

import { Button } from '@/shared/components/ui/button'
import { Form } from '@/shared/components/ui/form'
import { renderToast } from '@/shared/utils/toast'

import { toggle<Entity>StatusAction } from '../../actions'

type Archive<Entity>FormProps = {
  entityId: string
  closeDialog: () => void
}

/** Confirmation form for archiving a <entity>. */
export function Archive<Entity>Form({
  entityId,
  closeDialog,
}: Archive<Entity>FormProps) {
  const form = useForm()

  async function onSubmit() {
    const toastId = toast.loading('Archiving <entity>...')

    const res = await toggle<Entity>StatusAction(entityId)

    renderToast(res.ok, res.message, toastId)

    if (res.ok) {
      closeDialog()
    }
  }

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit(onSubmit)}
        className="flex flex-col gap-4"
        data-testid="<entity>-archive-form"
      >
        <p className="text-sm">
          Are you sure you want to archive this <entity>? It will no longer
          appear in default listings.
        </p>
        <div className="mt-auto ml-auto flex gap-4">
          <Button
            variant="outline"
            onClick={closeDialog}
            data-testid="<entity>-archive-cancel"
          >
            Cancel
          </Button>
          <Button type="submit" data-testid="<entity>-archive-confirm">
            Archive
          </Button>
        </div>
      </form>
    </Form>
  )
}
```

---

## Dialogs

Dialogs are thin wrappers around forms. They own the open/close state via the store and render the form inside.

```tsx
// src/domains/<domain>/components/dialogs/archive-<entity>-dialog.tsx
'use client'

import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/shared/components/ui/dialog'

import { use<Entity>DialogsStore } from '../../store'
import { Archive<Entity>Form } from '../forms/archive-<entity>-form'

type Archive<Entity>DialogProps = {
  entityId: string
}

/** Dialog for confirming <entity> archival. */
export function Archive<Entity>Dialog({ entityId }: Archive<Entity>DialogProps) {
  const { archiveDialogOpen, setArchiveDialogOpen } = use<Entity>DialogsStore()

  function closeDialog() {
    setArchiveDialogOpen(null)
  }

  return (
    <Dialog
      open={archiveDialogOpen === entityId}
      onOpenChange={() => setArchiveDialogOpen(null)}
    >
      <DialogContent data-testid="<entity>-archive-dialog">
        <DialogHeader>
          <DialogTitle>Archive <entity></DialogTitle>
        </DialogHeader>
        <Archive<Entity>Form entityId={entityId} closeDialog={closeDialog} />
      </DialogContent>
    </Dialog>
  )
}
```

---

## UI components

Small, focused components for domain-specific UI elements.

### Actions popover

```tsx
// src/domains/<domain>/components/ui/<entity>-actions-popover.tsx
'use client'

import { Ellipsis } from 'lucide-react'
import Link from 'next/link'
import { usePathname } from 'next/navigation'

import { Button } from '@/shared/components/ui/button'
import { Popover, PopoverContent, PopoverTrigger } from '@/shared/components/ui/popover'

import type { <Entity> } from '../../schemas'
import { use<Entity>DialogsStore } from '../../store'

type <Entity>ActionsPopoverProps = {
  entity: <Entity>
}

/** Contextual actions menu for a <entity> (edit, archive). */
export function <Entity>ActionsPopover({ entity }: <Entity>ActionsPopoverProps) {
  const pathname = usePathname()
  const { setArchiveDialogOpen } = use<Entity>DialogsStore()

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          size="icon"
          data-testid={`<entity>-actions-${entity.id}`}
        >
          <Ellipsis />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-56" align="end">
        <Button
          asChild
          variant="ghost"
          className="w-full justify-start"
          data-testid={`<entity>-edit-link-${entity.id}`}
        >
          <Link href={`${pathname}?edit-<entity>=${entity.id}`}>
            Edit <entity>
          </Link>
        </Button>
        <Button
          variant="ghost"
          className="w-full justify-start"
          onClick={() => setArchiveDialogOpen(entity.id)}
          data-testid={`<entity>-archive-btn-${entity.id}`}
        >
          {entity.active ? 'Archive' : 'Activate'}
        </Button>
      </PopoverContent>
    </Popover>
  )
}
```

---

## Barrel export

```ts
// src/domains/<domain>/components/index.ts
export { <entity>Columns } from './table/<entity>-columns'
export { <Entity>Table } from './table/<entity>-table'
export { Create<Entity>Form } from './forms/create-<entity>-form'
export { Edit<Entity>Form } from './forms/edit-<entity>-form'
export { Archive<Entity>Form } from './forms/archive-<entity>-form'
export { Archive<Entity>Dialog } from './dialogs/archive-<entity>-dialog'
export { <Entity>ActionsPopover } from './ui/<entity>-actions-popover'
```

---

## Rules

- Components are grouped into subdirectories by category — groups emerge from the domain's needs, not from a fixed list
- `'use client'` directive on all components that use hooks, state, or event handlers
- Every interactive element must have a `data-testid` attribute — never rely on CSS selectors or text content in tests
- `data-testid` naming: `<entity>-<component>-<element>`, dynamic IDs use template literals
- Every component must have a one-line JSDoc comment describing what it does
- Forms use `react-hook-form` + `zodResolver` — validation is always schema-driven
- Forms show `toast.loading` before the action, `renderToast` after — never silent mutations
- Dialogs are thin wrappers — they manage open/close via the store and render a form inside
- Confirmation actions (archive, delete) use a form component even without fields — keeps the pattern consistent
- Columns are defined in a separate file from the table component
- Every `components/` directory must have an `index.ts` barrel export
- Shared UI primitives (Button, Input, Dialog, etc.) come from `shared/ui/` — never redefine
- Shared form fields (InputField, SelectField, etc.) come from `shared/components/fields/`

---

## Anti-patterns

```tsx
// ❌ missing data-testid on interactive element
<Button type="submit">Save</Button>
// always: <Button type="submit" data-testid="<entity>-edit-submit">Save</Button>

// ❌ flat component directory when components share a category
components/
  create-form.tsx
  edit-form.tsx
  archive-form.tsx
  delete-form.tsx
// group related components: forms/create-form.tsx, forms/edit-form.tsx, etc.

// ❌ dialog with inline form logic
export function ArchiveDialog() {
  async function handleArchive() { ... }  // extract to a form component
  return <Dialog>...<Button onClick={handleArchive}>...</Dialog>
}

// ❌ no toast feedback on mutation
async function onSubmit(data) {
  await createAction(data)  // missing toast.loading + renderToast
}

// ❌ missing JSDoc
export function EntityTable() { ... }  // add: /** Displays entities in a table. */

// ❌ using CSS selectors in tests
await page.locator('.submit-btn').click()  // use data-testid

// ❌ duplicating shared UI primitives
function MyButton() { ... }  // use <Button> from shared/ui/

// ❌ missing 'use client' on component with hooks
import { useState } from 'react'
export function MyComponent() { ... }  // needs 'use client'
```
