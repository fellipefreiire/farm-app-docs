# Component — Forms

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## Create form

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

        {/* Numeric fields: NEVER use type="number". Use type="text" with mask + numeric prop. */}
        <InputField
          control={form.control}
          label="Ano"
          name="year"
          inputMode="numeric"
          maxLength={4}
          mask={maskDigitsOnly}
          numeric
          placeholder="Ex: 2024"
          data-testid="<entity>-year-input"
        />

        {/* Decimal fields: use inputMode="decimal" with maskPositiveFloat */}
        <InputField
          control={form.control}
          label="Capacidade (L)"
          name="capacity"
          inputMode="decimal"
          mask={maskPositiveFloat}
          numeric
          placeholder="Ex: 2000"
          data-testid="<entity>-capacity-input"
        />
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

## Confirmation form (delete)

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
