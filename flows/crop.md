# Crop — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Manage crop types [MVP]

**Trigger:** User manages crop types (create, edit, delete)
**Actor:** Farm owner, Farm manager
**Domain:** Crop (CropType subdomain)

**Create:**
1. User clicks "Novo tipo de cultura" → form opens
2. User fills: name → submits
3. System creates crop type → success toast → list refreshes → audit log records creation

**Edit/Delete:** Same pattern as field management — edit inline, delete with confirmation.

**Error cases:**
- Missing required fields → inline validation

---

## Manage varieties [MVP]

**Trigger:** User manages varieties within a crop type
**Actor:** Farm owner, Farm manager
**Domain:** Crop (Variety subdomain)

**Create:**
1. User navigates to a crop type → clicks "Nova variedade"
2. User fills: name → submits
3. System creates variety linked to crop type → success toast → audit log records creation

**Edit/Delete:** Same pattern as field management.

**Error cases:**
- Missing required fields → inline validation

---

## Harvest lifecycle [MVP]

**Trigger:** User creates and manages a harvest through its lifecycle
**Actor:** Farm owner, Farm manager
**Domain:** Crop (Harvest subdomain)

**Listing:**
1. User navigates to "Colheitas" → paginated table with columns: Nome, Status (badge), Tipo de Cultura, Variedade, Talhão, Data de Início, Previsão de Término
2. Tabs filter by status: Todos, Planejadas, Ativas, Em Revisão, Concluídas, Canceladas
3. Search input filters by nome (debounced, server-side)
4. Filters popover allows filtering by Tipo de Cultura, Variedade (dependent on crop type), Talhão — active filters shown as removable tags below the search bar

**Create:**
1. User clicks "Adicionar Colheita" → sheet opens
2. User fills: nome, tipo de cultura (async select), variedade (async select, dependent), talhão (async select), data de início, previsão de término → submits
3. System validates (variety belongs to crop type, no active harvest on field, valid date range) → creates harvest with status `PLANNED` → audit log records creation

**Edit:**
1. User opens harvest detail → clicks "Editar Colheita" → sheet opens
2. User can edit: nome, tipo de cultura, variedade, talhão, datas
3. System updates harvest → audit log records all changed fields (name, cropTypeId, varietyId, fieldId, dates)
4. Sheet closes automatically on success

**Detail page:**
- Header: nome + status badge
- Sidebar "Detalhes" (collapsible, shows first 4): Tipo de Cultura (link), Variedade (link), Talhão (link), Data de Início, Previsão de Término, Data de Término (if completed), Criado em, Atualizado em
- Main: Auditoria (last 5 logs with diff modal)

**Activate:**
- Activation is automatic — driven by the Schedule domain when a schedule for this harvest becomes ACTIVE (via `ScheduleActivatedEvent`). There is no manual "Iniciar" button for harvests.
- A linked Schedule is also auto-created when the Harvest is created (via `HarvestCreatedEvent` → `OnHarvestCreatedCreateSchedule` subscriber).

**Complete:**
- Completion is automatic — driven by the Schedule domain when the review is confirmed (via `ScheduleCompletedEvent`). There is no manual "Finalizar" button for harvests.

**Cancel:**
1. User clicks "Cancelar" on a planned or active harvest
2. Confirmation dialog → system changes status to `CANCELLED` → audit log records cancellation

**Error cases:**
- Already active harvest on the same field → error: "Já existe uma colheita ativa neste talhão"
- Invalid status transition (e.g., complete a cancelled harvest) → error

---
