# Component — Forms with dynamic item lists (in sheets)

> Part of the farm-app frontend component pattern collection. Read `_index.md` first for shared rules and anti-patterns.

When a form has a dynamic list of items (e.g., purchase items), the form layout must prevent the submit button from being pushed off-screen:

```tsx
<form className="flex h-full min-h-0 flex-col p-4">
  {/* Fixed top — static fields + section header */}
  <div className="flex flex-col gap-4 pb-4">
    <SelectField ... />
    <InputField ... />
    <Separator />
    <div className="flex items-center justify-between">
      <h3 className="text-sm font-medium">Itens da Entrada</h3>
      <Button onClick={() => append(...)}>Adicionar item</Button>
    </div>
  </div>

  {/* Scrollable middle — dynamic item list */}
  <div className="flex min-h-0 flex-1 flex-col gap-4 overflow-y-auto">
    {fields.map((field) => (
      <div key={field.id} className="rounded-md border p-3">...</div>
    ))}
  </div>

  {/* Fixed bottom — submit button */}
  <div className="ml-auto flex gap-4 pt-4">
    <Button type="submit">Criar</Button>
  </div>
</form>
```

**Key classes:**
- `h-full min-h-0` on `<form>` — fills sheet height, allows children to shrink
- `flex-1 min-h-0 overflow-y-auto` on item container — scrolls only the items
- No `mt-auto` on submit button — it's naturally pinned by the flex layout

---
