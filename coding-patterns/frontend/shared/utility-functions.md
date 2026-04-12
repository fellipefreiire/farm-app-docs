# Shared — Utility functions

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## getSearchParam

Extracts a single string value from Next.js searchParams, handling the `string | string[] | undefined` union safely.

```ts
// src/shared/utils/search-params.ts
export function getSearchParam(
  value: string | string[] | undefined,
): string | undefined {
  return typeof value === 'string' ? value : undefined
}
```

**Usage:**

```tsx
const searchParams = await props.searchParams
const page = getSearchParam(searchParams.page)
const search = getSearchParam(searchParams.search)
```

## renderToast

Dismisses a loading toast and shows a success or error toast based on the action result.

```ts
// src/shared/utils/toast.ts
import { toast } from 'sonner'

export function renderToast(
  ok: boolean,
  message: string,
  toastId: string | number,
) {
  if (ok) {
    toast.success(message, { id: toastId })
  } else {
    toast.error(message, { id: toastId })
  }
}
```

**Usage pattern (in form components):**

```tsx
const toastId = toast.loading('Creating entity...')
const res = await createEntityAction(data)
renderToast(res.ok, res.message, toastId)
```

The `id` parameter replaces the loading toast in-place — the user sees a smooth transition from loading to success/error.

## Input masks

Mask functions for numeric form fields. **Never use `type="number"`** — always use `type="text"` with a mask and the `numeric` prop on `InputField`.

```ts
// src/shared/utils/masks.ts

/** Allows only digits (integers). Use for year, quantity, etc. */
export function maskDigitsOnly(value: string): string {
  return value.replace(/\D/g, '')
}

/** Allows digits and a single decimal point (positive floats). Use for capacity, width, etc. */
export function maskPositiveFloat(value: string): string {
  const cleaned = value.replace(/[^\d.]/g, '')
  const parts = cleaned.split('.')
  if (parts.length <= 2) return cleaned
  return parts[0] + '.' + parts.slice(1).join('')
}
```

**Usage with InputField:**

```tsx
import { maskDigitsOnly, maskPositiveFloat } from '@/shared/utils/masks'

// Integer field (year)
<InputField name="year" inputMode="numeric" maxLength={4} mask={maskDigitsOnly} numeric data-testid="..." />

// Decimal field (capacity in liters)
<InputField name="tankCapacityL" inputMode="decimal" mask={maskPositiveFloat} numeric data-testid="..." />
```

The `numeric` prop on `InputField` converts the masked string to `Number` before passing to react-hook-form, so Zod schemas can keep `z.number()` validation.

---
