# Component Pattern

Domain components are organized by type in subdirectories. They consume server actions, stores, and shared UI primitives. All interactive elements must include `data-testid` attributes for Playwright tests. All user-facing text is in Portuguese (pt-BR).

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

Components are grouped into subdirectories when **two or more components share the same category**. Groups are not fixed — they emerge from the domain's needs.

Common groups:

```
table/      ← columns + table component
forms/      ← create, edit, confirmation forms
dialogs/    ← dialog wrappers (archive, delete, cancel)
sheets/     ← sheet wrappers (create, edit)
ui/         ← small domain-specific UI (badges, popovers, cards)
```

---

## data-testid convention

All interactive elements must include `data-testid` for Playwright tests. Never use CSS selectors or text content in tests.

**Naming format:** `<entity>-<component>-<element>`

```tsx
// Buttons
<Button data-testid="<entity>-create-submit">Criar <entity></Button>
<Button data-testid="<entity>-edit-submit">Salvar alterações</Button>
<Button data-testid="<entity>-delete-confirm">Excluir <Entity></Button>
<Button data-testid="<entity>-delete-cancel">Voltar</Button>

// Form fields
<InputField data-testid="<entity>-name-input" ... />
<SelectField data-testid="<entity>-status-select" ... />

// Actions
<Button data-testid={`<entity>-actions-${entity.id}`} ... />
<Button data-testid={`<entity>-edit-btn-${entity.id}`} ... />
<Button data-testid={`<entity>-delete-btn-${entity.id}`} ... />

// Dialogs
<DialogContent data-testid="<entity>-delete-dialog" ... />

// Pages
<div data-testid="<entity>s-page" ... />
<div data-testid="<entity>-detail-page" ... />
```

---

## Table

### Columns with image placeholder

```tsx
// src/domains/<domain>/components/table/<entity>-columns.tsx
'use client'

import { <Icon> } from 'lucide-react'
import type { CellContext, ColumnDef } from '@tanstack/react-table'

import type { <Entity> } from '../../schemas'
import { <Entity>ActionsPopover } from '../ui/<entity>-actions-popover'

export const <entity>Columns: ColumnDef<<Entity>>[] = [
  {
    id: 'image',
    cell: ImageCell,
    meta: { className: 'w-10 pr-0' },
  },
  {
    header: 'Name',
    accessorKey: 'name',
    meta: { grow: true },
  },
  {
    header: 'Created at',
    accessorKey: 'createdAt',
    cell: ({ row }) =>
      new Intl.DateTimeFormat('en-US', { dateStyle: 'medium' }).format(
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

### Table component

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

## Forms

### Create form

```tsx
// src/domains/<domain>/components/forms/create-<entity>-form.tsx
'use client'

import { usePathname, useRouter } from 'next/navigation'
import { useForm } from 'react-hook-form'
import { toast } from 'sonner'
import { zodResolver } from '@hookform/resolvers/zod'

import { InputField } from '@/shared/components/fields/input-field'
import { Button } from '@/shared/components/ui/button'
import { Form } from '@/shared/components/ui/form'
import { renderToast } from '@/shared/utils/toast'

import { create<Entity>Action } from '../../actions'
import { type Create<Entity>Request, create<Entity>Schema } from '../../schemas'

/** Form for creating a new <entity>. */
export function Create<Entity>Form() {
  const pathname = usePathname()
  const { push } = useRouter()

  const form = useForm<Create<Entity>Request>({
    resolver: zodResolver(create<Entity>Schema),
    defaultValues: { name: '' },
  })

  async function onSubmit(data: Create<Entity>Request) {
    const toastId = toast.loading('Criando <entity>...')
    const res = await create<Entity>Action(data)
    renderToast(res.ok, res.message, toastId)
    if (res.ok) { push(pathname) }
  }

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit(onSubmit)}
        className="flex h-full flex-col gap-4 p-4"
        data-testid="<entity>-create-form"
      >
        <InputField control={form.control} label="Nome" name="name" data-testid="<entity>-name-input" />
        <div className="mt-auto ml-auto flex gap-4">
          <Button type="submit" data-testid="<entity>-create-submit">
            Criar <entity>
          </Button>
        </div>
      </form>
    </Form>
  )
}
```

### Confirmation form (delete)

Destructive actions always use a confirmation dialog. The form has no fields — just a warning message and action buttons.

```tsx
// src/domains/<domain>/components/forms/delete-<entity>-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { useRouter } from 'next/navigation'
import { toast } from 'sonner'

import { Button } from '@/shared/components/ui/button'
import { Form } from '@/shared/components/ui/form'
import { renderToast } from '@/shared/utils/toast'

import { delete<Entity> } from '../../actions'

type Delete<Entity>FormProps = {
  entityId: string
  closeDialog: () => void
}

/** Confirmation form for deleting a <entity>. */
export function Delete<Entity>Form({ entityId, closeDialog }: Delete<Entity>FormProps) {
  const form = useForm()
  const router = useRouter()

  async function onSubmit() {
    const toastId = toast.loading('Excluindo <entity>...')
    const res = await delete<Entity>(entityId)
    renderToast(res.ok, res.message, toastId)
    if (res.ok) {
      closeDialog()
      router.push('/<entities>')
    }
  }

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit(onSubmit)}
        className="flex flex-col gap-4"
        data-testid="<entity>-delete-form"
      >
        <p className="text-sm">
          Tem certeza que deseja excluir esta <entity>? Esta ação não pode ser desfeita.
        </p>
        <div className="mt-auto ml-auto flex gap-2">
          <Button type="button" variant="outline" onClick={closeDialog} data-testid="<entity>-delete-cancel">
            Voltar
          </Button>
          <Button type="submit" variant="destructive" data-testid="<entity>-delete-confirm">
            Excluir <Entity>
          </Button>
        </div>
      </form>
    </Form>
  )
}
```

**Key points:**
- Button gap is `gap-2` (8px) in confirmation modals
- Cancel button label: "Voltar"
- Delete button: `variant="destructive"`, label: "Excluir {Nome do Módulo}"
- After deletion, redirect to listing page via `router.push`

---

## Dialogs

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

## UI components

### Actions popover

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

## Barrel export

```ts
// src/domains/<domain>/components/index.ts
export * from './table/<entity>-columns'
export * from './table/<entity>-table'
export * from './forms/create-<entity>-form'
export * from './forms/edit-<entity>-form'
export * from './forms/delete-<entity>-form'
export * from './dialogs/delete-<entity>-dialog'
export * from './sheets/create-<entity>-sheet'
export * from './sheets/edit-<entity>-sheet'
export * from './ui/<entity>-actions-popover'
```

---

## Rules

- Components are grouped into subdirectories by category — groups emerge from the domain's needs
- `'use client'` directive on all components that use hooks, state, or event handlers
- Every interactive element must have a `data-testid` attribute
- Every component must have a one-line JSDoc comment describing what it does
- Forms use `react-hook-form` + `zodResolver` — validation is always schema-driven
- Forms show `toast.loading` before the action, `renderToast` after — never silent mutations
- Toast messages in Portuguese: "Criando...", "Excluindo...", "Salvando..."
- Dialogs are thin wrappers — they manage open/close via the store and render a form inside
- Destructive actions always require a confirmation dialog with `variant="destructive"` button
- Destructive option text uses `text-destructive` in popovers
- After deletion, redirect to the listing page
- Columns are defined in a separate file from the table component
- Every `components/` directory must have an `index.ts` barrel export
- All user-facing text in Portuguese (pt-BR)

---

## Anti-patterns

```tsx
// ❌ missing data-testid on interactive element
<Button type="submit">Save</Button>
// ✅ <Button type="submit" data-testid="<entity>-edit-submit">Salvar alterações</Button>

// ❌ destructive action without confirmation dialog
onClick={() => deleteEntity(id)}
// ✅ onClick={() => setDeleteDialogOpen(id)}

// ❌ destructive option without red text
<Button variant="ghost">Excluir</Button>
// ✅ <Button variant="ghost" className="text-destructive">Excluir <entity></Button>

// ❌ English text in UI
<Button>Delete</Button>
// ✅ <Button>Excluir Variedade</Button>

// ❌ no redirect after deletion
if (res.ok) { closeDialog() }
// ✅ if (res.ok) { closeDialog(); router.push('/<entities>') }

// ❌ no toast feedback on mutation
await createAction(data)
// ✅ const toastId = toast.loading('Criando...'); ... renderToast(...)

// ❌ popover trigger without size-7
<Button variant="outline" size="icon">
// ✅ <Button variant="outline" size="icon" className="size-7">

// ❌ showEdit=true on detail page where edit button exists
<ActionsPopover entity={entity} />
// ✅ <ActionsPopover entity={entity} showEdit={false} />
```
