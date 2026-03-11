# Store Pattern

Stores manage client-side UI state using Zustand. They handle user interactions only — dialogs, selections, local UI toggles. All API communication is server-side (API functions + server actions).

---

## File locations

```
src/domains/<domain>/store/<entity>-dialogs.store.ts     ← dialog/modal state
src/domains/<domain>/store/<entity>-selection.store.ts   ← selection state (e.g. multi-select)
src/domains/<domain>/store/index.ts                      ← barrel export
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), stores move under the subdomain: `src/domains/<domain>/<subdomain>/store/`.

One store per concern. One file per store.

---

## Structure

```ts
// src/domains/<domain>/store/<entity>-dialogs.store.ts
'use client'

import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface <Entity>DialogsState {
  archiveDialogOpen: string | null
  deleteDialogOpen: string | null
  setArchiveDialogOpen: (id: string | null) => void
  setDeleteDialogOpen: (id: string | null) => void
}

export const use<Entity>DialogsStore = create<<Entity>DialogsState>()(
  devtools(
    (set) => ({
      archiveDialogOpen: null,
      deleteDialogOpen: null,
      setArchiveDialogOpen: (id) => set({ archiveDialogOpen: id }),
      setDeleteDialogOpen: (id) => set({ deleteDialogOpen: id }),
    }),
    { name: '<Entity>Dialogs' },
  ),
)
```

Dialog state uses `string | null` — the value is the entity ID when open, `null` when closed. This allows the component to know *which* entity the dialog refers to.

For dialogs that don't target a specific entity (e.g. create dialogs), use `boolean` instead:

```ts
interface <Entity>DialogsState {
  createDialogOpen: boolean            // no entity ID needed
  deleteDialogOpen: string | null      // needs to know which entity
  setCreateDialogOpen: (open: boolean) => void
  setDeleteDialogOpen: (id: string | null) => void
}
```

---

## With persist middleware

Use `persist` when the state should survive page reloads — e.g. sidebar collapsed, user preferences, view mode.

```ts
// src/domains/<domain>/store/<entity>-preferences.store.ts
'use client'

import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

interface <Entity>PreferencesState {
  viewMode: 'table' | 'grid'
  setViewMode: (mode: 'table' | 'grid') => void
}

export const use<Entity>PreferencesStore = create<<Entity>PreferencesState>()(
  devtools(
    persist(
      (set) => ({
        viewMode: 'table',
        setViewMode: (mode) => set({ viewMode: mode }),
      }),
      { name: '<entity>-preferences' },
    ),
    { name: '<Entity>Preferences' },
  ),
)
```

---

## Barrel export

```ts
// src/domains/<domain>/store/index.ts
export { use<Entity>DialogsStore } from './<entity>-dialogs.store'
```

---

## Rules

- `'use client'` directive is mandatory — stores run on the client
- Always wrap with `devtools` middleware — enables Redux DevTools inspection during development
- `devtools` `name` option is required — identifies the store in DevTools
- Use `persist` only when state must survive page reloads (user preferences, view mode) — never for ephemeral UI state (dialogs, popovers)
- Stores hold UI state only — never cache server data in stores (use API functions with Next.js cache)
- Dialog state is `string | null` (entity ID when open, `null` when closed) for dialogs targeting a specific entity — use `boolean` for dialogs that don't target an entity (e.g. create)
- One store per concern — `dialogs`, `selection`, `preferences` are separate stores
- Setters follow `set<Property>` naming convention
- Every `store/` directory must have an `index.ts` barrel export

---

## Anti-patterns

```ts
// ❌ server data in store
const useProductStore = create((set) => ({
  products: [],
  fetchProducts: async () => { ... },  // never — use API functions + server actions
}))

// ❌ missing devtools middleware
export const useStore = create((set) => ({
  ...  // always wrap with devtools for DX
}))

// ❌ boolean for entity-targeting dialog state
archiveDialogOpen: false  // use string | null — need to know which entity
// ✅ boolean is correct for non-entity dialogs (create)
createDialogOpen: false   // no entity ID needed

// ❌ missing 'use client' directive
import { create } from 'zustand'  // will fail in server components

// ❌ multiple concerns in one store
const useProductStore = create((set) => ({
  archiveDialogOpen: null,   // dialog concern
  selectedIds: [],            // selection concern
  viewMode: 'table',          // preference concern
  // split into separate stores
}))

// ❌ persist on ephemeral state
persist(
  (set) => ({ archiveDialogOpen: null, ... }),  // dialogs should reset on reload
)
```
