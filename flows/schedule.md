# Schedule — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Schedule auto-created with Harvest [MVP]

**Trigger:** User creates a Harvest — Schedule is auto-created via `HarvestCreatedEvent`
**Actor:** Farm owner, Farm manager (indirect — triggered by harvest creation)
**Domain:** Schedule (via `OnHarvestCreatedCreateSchedule` subscriber)

**Happy path:**
1. User creates a Harvest (see Harvest lifecycle above)
2. System emits `HarvestCreatedEvent` → `OnHarvestCreatedCreateSchedule` subscriber fires automatically
3. Schedule is created with status PLANNED or ACTIVE:
   - ACTIVE if the field has no other ACTIVE schedule
   - PLANNED if the field already has an ACTIVE schedule
4. User navigates to the schedule → adds operations day by day:
   - Selects operation type (spraying, fertigation, etc.)
   - Selects day in the schedule
   - Assigns inputs from inventory (product, dosage)
5. User repeats step 4 until schedule is complete

**Note:** There is no manual "Novo cronograma" button — schedules are always auto-created with their harvest.

**Error cases:**
- Input not in inventory → select only shows existing inventory items

---

## Edit schedule [MVP]

**Trigger:** User opens an existing schedule and modifies operations/inputs
**Actor:** Farm owner, Farm manager
**Domain:** Schedule

**Happy path:**
1. User opens schedule detail page
2. User adds, removes, or edits operations and inputs
3. System saves changes → changes reflect in pending field tickets

**Error cases:**
- Schedule already has finalized field tickets for edited days → warning before proceeding

---

## Copy schedule (wizard) [MVP]

**Trigger:** User clicks "Copiar cronograma" on a schedule's actions menu
**Actor:** Farm owner, Farm manager
**Domain:** Schedule

**4-step wizard flow:**

**Step 1 — Destino:**
1. Wizard opens with two tabs: "Safra existente" and "Nova safra"
2. **Existing harvest tab:** User selects any harvest via async search (not limited to PLANNED). Info badge shows harvest status.
3. **New harvest tab:** User fills inline form (name, crop type, variety, field, dates). Harvest is created via `POST /v1/harvests` when advancing.
4. If creation fails → stays on Step 1 with validation errors.

**Step 2 — Configuração:**
1. User selects date mode:
   - **Offset relativo** (default): Operations are mapped by day offset from source harvest start to target harvest start
   - **Datas exatas**: Operations keep their original dates
2. If target already has operations, conflict resolution appears:
   - **Adicionar** (default): New operations are added alongside existing ones
   - **Substituir**: Existing operations are soft-deleted before copying. Warning shows count of operations to be removed.

**Step 3 — Preview:**
1. System calls `GET /v1/schedules/:id/copy-preview` to generate mapping table
2. Table shows: source date → target date, operation type, input count, status (within range or out of range)
3. Summary shows: X operations to copy, Y to skip (out of target harvest period)
4. If replace mode: shows count of existing operations to be removed

**Step 4 — Confirmação:**
1. Summary card: target harvest (new/existing), date mode, operations to copy/skip
2. If replace: destructive warning about operations to be removed
3. User clicks "Copiar Cronograma" → `POST /v1/schedules/:id/copy` with `dateMode` and `conflictResolution`
4. Success → toast + redirect to new/updated schedule detail page

**Error cases:**
- Source schedule not found → 404
- Target harvest not found → 404
- All operations out of range → "Copiar" button disabled, user can go back and change config
- Network error on preview → error state with "Voltar" button
- User can cancel at any step → wizard closes without side effects

---

## Review completed schedule [MVP]

**Trigger:** System auto-transitions schedule to UNDER_REVIEW when all field tickets are terminal (COMPLETED/CANCELLED)
**Actor:** Farm owner
**Domain:** Schedule

**Happy path:**
1. All field tickets in the schedule reach terminal state (COMPLETED or CANCELLED)
2. System automatically transitions schedule to UNDER_REVIEW status
3. User sees "Em Revisão" badge on the schedule and a "Revisar Cronograma" button
4. User clicks "Revisar" → review dialog opens showing:
   - Summary of completed vs cancelled tickets
   - Optional notes/observations textarea
5. User fills notes (optional) → clicks "Confirmar Revisão"
6. System shows PLANNED schedules for the same field (if any) → user selects which to activate next
7. If no PLANNED schedules → option to create new or just close
8. Schedule → COMPLETED, Harvest → COMPLETED
9. Selected next schedule → ACTIVE, its Harvest → ACTIVE

**Error cases:**
- No field tickets in schedule → schedule never reaches UNDER_REVIEW (stays ACTIVE until cancelled)
- User adds new field ticket during review → schedule reverts to ACTIVE automatically

---

## Cancel schedule with PRINTED resolution [MVP]

**Trigger:** User clicks "Cancelar" on a PLANNED or ACTIVE schedule
**Actor:** Farm owner
**Domain:** Schedule

**Happy path:**
1. User clicks "Cancelar" on a schedule
2. If schedule has PRINTED tickets → system shows resolution dialog:
   - Lists each PRINTED ticket
   - For each: "Finalizar" (was executed) or "Cancelar" (was not)
   - User resolves all PRINTED tickets before proceeding
3. System shows confirmation dialog:
   - Alert: "Ao cancelar, todas as boletas não executadas serão canceladas. Boletas concluídas serão mantidas como histórico."
   - Mandatory reason field
4. User confirms → system:
   - Auto-cancels all DRAFT/REVIEWED tickets
   - Preserves COMPLETED tickets as historical record
   - Schedule → CANCELLED, Harvest → CANCELLED
5. System shows post-cancellation options:
   - "Ativar cronograma planejado" (if PLANNED schedules exist for the field)
   - "Criar novo baseado neste" (copies all operations as DRAFT)
   - "Fechar"

**Error cases:**
- PRINTED tickets not resolved → cancel button disabled until all resolved
- Reason not provided → validation error

---
