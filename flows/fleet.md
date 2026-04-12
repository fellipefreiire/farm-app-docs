# Fleet — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Manage vehicles [MVP]

**Trigger:** User navigates to /vehicles
**Actor:** Farm owner, Farm manager
**Domain:** Fleet (Vehicles subdomain)

**Happy path:**
1. List page shows vehicles with tabs (Todos/Ativos/Inativos) and search
2. Create: user clicks "Adicionar Veículo" → sheet opens → fills code, name, type (select), optional fields (placa, marca, modelo, ano) → submits → toast "Veículo criado com sucesso."
3. Edit: actions popover → "Editar" → sheet opens → changes fields → submits → toast "Veículo atualizado com sucesso."
4. Toggle status: actions popover → "Ativar/Desativar" → toast confirms
5. Delete: actions popover → "Excluir" → confirmation dialog → confirms → toast "Veículo excluído com sucesso."
6. Detail page: shows vehicle details sidebar + audit logs

**Error cases:**
- Duplicate code → inline error: "Já existe um veículo com esse código"
- Missing required fields → inline validation

---

## Manage implements [MVP]

**Trigger:** User navigates to /implements
**Actor:** Farm owner, Farm manager
**Domain:** Fleet (Implements subdomain)

**Happy path:**
1. List page shows implements with tabs (Todos/Ativos/Inativos), search, and type filter
2. Create: user clicks "Adicionar Implemento" → sheet opens → fills code, name, type (select), optional fields (marca, modelo, ano, capacidade do tanque, largura de trabalho) → submits → toast "Implemento criado com sucesso."
3. Edit: actions popover → "Editar" → sheet opens → changes fields → submits → toast "Implemento atualizado com sucesso."
4. Toggle status: actions popover → "Ativar/Desativar" → toast confirms
5. Delete: actions popover → "Excluir" → confirmation dialog → confirms → toast "Implemento excluído com sucesso."
6. Detail page: shows implement details sidebar + audit logs

**Error cases:**
- Duplicate code → inline error: "Já existe um implemento com esse código"
- Missing required fields → inline validation

---
