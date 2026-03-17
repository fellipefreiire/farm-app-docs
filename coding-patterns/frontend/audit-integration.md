# Audit Log Integration — Frontend

When integrating audit logs into a new domain's detail page, **3 frontend files must be updated** with mappings for every action the backend emits.

## Checklist

### 1. Find all backend actions

Search the backend subscribers for the domain:

```bash
grep -r "action: '<domain>:" backend/src/domain/audit-log/application/subscribers/<domain>/
```

This gives you every `action` string and the `changes` object structure.

### 2. Update `action-label-map.ts`

**File:** `frontend/src/domains/audit-log/utils/action-label-map.ts`

Add entries to **both** maps:

- `detailActionLabelMap` — full description (e.g., `'schedule:created': 'Cronograma criado'`)
- `tableActionLabelMap` — compact label (e.g., `'schedule:created': 'Criado'`)

### 3. Update `limited-audit-logs.tsx`

**File:** `frontend/src/domains/audit-log/components/limited-audit-logs.tsx`

Add entries to `actionTextMap` — humanized text with actor name:

```typescript
'schedule:created': (_v, a) => `Cronograma criado por ${a}`,
```

### 4. Update `audit-diff-modal.tsx`

**File:** `frontend/src/domains/audit-log/components/audit-diff-modal.tsx`

Update **three** maps:

- `fieldLabelMap` — translate field names to Portuguese (e.g., `dosagePer100L: 'Dosagem (mL/100L)'`)
- `valueLabelMap` — translate enum values (e.g., `SPRAYING: 'Pulverização'`, `PLANNED: 'Planejado'`)
- `hiddenFields` — add all ID/FK fields that should never be shown to the user

## Rules

### Never show UUIDs

The `hiddenFields` set must include **every field that contains a UUID**. Users should never see raw IDs in the audit diff modal. This includes:

- Foreign key fields (`harvestId`, `scheduleId`, `operationId`, `inputId`, etc.)
- Internal reference IDs (`operationInputId`, etc.)

If an ID field references another entity (e.g., `inputId` references an Input), the backend subscriber should include the entity's **name** as a separate human-readable field in `changes` instead of (or in addition to) the raw ID. The frontend then shows the name and hides the ID.

### Backend subscriber rule: always include human-readable names

When a backend audit subscriber records `changes`, it must **never** send only UUIDs for referenced entities. Instead:

- Inject the relevant repository into the subscriber
- Resolve the entity name/label before recording
- Include the resolved name as a readable field alongside (or instead of) the raw ID

**Example — what NOT to do:**
```typescript
changes: {
  inputId: operationInput.inputId.toString(),  // UUID — useless to the user
}
```

**Example — correct approach:**
```typescript
// Inject InputsRepository in the subscriber constructor
const input = await this.inputsRepository.findById(operationInput.inputId)

changes: {
  inputId: operationInput.inputId.toString(),  // hidden by frontend
  input: input?.name ?? 'Desconhecido',        // shown to the user
}
```

### Composite actions must show all related data

When an action involves creating/adding something with related entities, the `changes` object must include **all meaningful information**, not just the parent entity.

**Example — adding an operation with inputs:**

The `schedule:operation-added` subscriber should include:
- `type` — the operation type (e.g., `FERTIGATION`), mapped by frontend to "Fertirrigação"
- `date` — the operation date
- `inputs` — list of input names and dosages that were added with the operation

This means the subscriber may need to resolve input names from the repository before saving the audit log.

### When backend data is not available yet

If the backend subscriber doesn't currently send resolved names or composite data, document it as a pending improvement. On the frontend:
1. Hide the UUID fields (add to `hiddenFields`)
2. Show whatever human-readable fields are available (`type`, `date`, `dosagePer100L`, etc.)
3. Create a backend task to enrich the audit changes with resolved names

## Example: Schedule domain

Backend emits 13 actions:

| Action | Visible fields | Hidden fields |
|---|---|---|
| `schedule:created` | status | harvestId |
| `schedule:updated` | — | harvestId |
| `schedule:activated` | status | — |
| `schedule:completed` | status | — |
| `schedule:cancelled` | status | — |
| `schedule:deleted` | status | — |
| `schedule:operation-added` | date, type | operationId |
| `schedule:operation-updated` | date, type | operationId |
| `schedule:operation-removed` | — | operationId |
| `schedule:operation-input-added` | dosagePer100L | operationInputId, operationId, inputId |
| `schedule:operation-input-updated` | dosagePer100L | operationInputId, operationId, inputId |
| `schedule:operation-input-removed` | — | operationInputId |

### Pending improvements (schedule)

The following subscribers need backend changes to include resolved names:

- `on-schedule-operation-added.ts` — should include input names when operation is created with inputs
- `on-schedule-operation-input-added.ts` — should resolve `inputId` to input name via `InputsRepository`
- `on-schedule-operation-input-updated.ts` — same as above
