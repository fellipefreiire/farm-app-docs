# Inventory — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Manage categories [MVP]

**Trigger:** User navigates to /categories
**Actor:** Farm owner, Farm manager
**Domain:** Inventory

**Happy path:**
1. List page shows categories with tabs (Todos/Ativos/Inativos) and search
2. Create: user clicks "Adicionar Categoria" → sheet opens → fills name → submits → toast "Categoria criada com sucesso."
3. Edit: actions popover → "Editar" → sheet opens → changes name → submits → toast "Categoria atualizada com sucesso."
4. Toggle status: actions popover → "Ativar/Desativar" → toast confirms
5. Delete: actions popover → "Excluir" → confirmation dialog → confirms → toast "Categoria excluída com sucesso."
6. Detail page: shows linked inputs list + audit logs

## Manage inputs (insumos) [MVP]

**Trigger:** User navigates to /inputs
**Actor:** Farm owner, Farm manager
**Domain:** Inventory

**Happy path:**
1. List page shows inputs with tabs (Todos/Ativos/Inativos), search, filter popover (category), and list/card view toggle
2. Card view shows stock level with progress bar (color by quantity: red ≤10, yellow ≤50, green >50)
3. Table view shows "Em estoque" column (e.g., "32 kg")
4. Create: user clicks "Adicionar Insumo" → sheet opens → fills name, selects category, selects unit → submits
5. Detail page: shows entries (purchases) and exits (stock movements) for this input, with "Ver todas" links that navigate to filtered list pages

## Register purchase (entrada) [MVP]

**Trigger:** User clicks "Nova Entrada" on /purchases or from input detail page
**Actor:** Farm owner, Farm manager
**Domain:** Inventory

**Happy path:**
1. List page shows purchases with search, filter popover (supplier, input)
2. Create: sheet opens → selects supplier, date → adds items (input, quantity, total value) → submits → toast "Entrada criada com sucesso."
3. Items container scrolls independently — submit button stays fixed at bottom
4. Edit: actions popover → "Editar" → sheet opens → fills edit reason → modifies fields → submits
5. Delete: actions popover → "Excluir" → confirmation dialog → confirms → redirects to /purchases
6. Detail page: shows items list (linked to inputs), supplier link, audit logs

## Register stock movement (saída) [MVP]

**Trigger:** User clicks "Nova Saída" on /stock-movements or from input detail page
**Actor:** Farm owner, Farm manager
**Domain:** Inventory

**Happy path:**
1. List page shows stock movements with tabs (Todos/Ativos/Cancelados), search, filter popover (input, reason)
2. Create: sheet opens → selects input, fills quantity, reason defaults to LOSS → submits → toast "Saída de estoque criada com sucesso."
3. Cancel: actions popover → "Cancelar saída" → toast confirms. Cancelled movements show "Cancelado" badge.
4. Detail page: shows input link, quantity, type, reason, audit logs

---
