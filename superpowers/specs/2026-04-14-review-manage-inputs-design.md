# Manage Insumos During Ticket Review — Design

## Context

Today, when the operator reviews a field ticket in the Execution Panel, the configure sheet only lets them toggle `highlighted` on existing inputs. To add or remove an insumo they must go back to the DRAFT edit form. The user needs add/remove capability directly inside the review sheet so reviewing is a single action.

## Decisions

- **Status effect**: add/remove does NOT change the ticket status. Same as current review semantics.
- **API shape**: extend the existing `PATCH /v1/field-tickets/:id/review` to accept a full `inputs` array (diff-based). No new endpoints.
- **Allowed statuses**: widen from `DRAFT | REVIEWED` to "any non-COMPLETED" (DRAFT, REVIEWED, PRINTED).

## Backend changes

### 1. Schema — `backend/src/domains/field-ticket/http/schemas/review-field-ticket.schema.ts`

Replace `highlightedInputIds: z.array(z.string().uuid()).optional()` with:

```ts
inputs: z.array(z.object({
  inputId: z.string().uuid(),
  dosagePer100L: z.number().positive(),
  highlighted: z.boolean().default(false),
})).optional()
```

The `inputs` array is the authoritative list after review. Omitting it preserves current inputs (backward-compatible for the highlight-only workflow, though the frontend will always send the array).

### 2. Use case — `backend/src/domains/field-ticket/application/use-cases/review-field-ticket.use-case.ts`

Replace the highlight-flag loop (~lines 124–130) with a diff + apply block:

- Load existing `FieldTicketInput[]` for the ticket.
- Compute three sets by `inputId`:
  - **to add**: in payload, not in existing → create `FieldTicketInput`.
  - **to remove**: in existing, not in payload → delete.
  - **to update**: in both → patch `dosagePer100L` and `highlighted`.
- Persist via the existing `fieldTicketInputsRepository` (extend with `deleteMany(ids)` if missing).
- Emit the existing domain event unchanged.

Validation: if payload includes duplicate `inputId`s, reject with a domain error (`DuplicateInputError`). If payload `inputId` doesn't exist in the inputs catalog, reject (`InputNotFoundError`) — reuse existing check.

### 3. Controller status guard — `backend/src/domains/field-ticket/http/controllers/review-field-ticket.controller.ts`

Widen the allowed-status check from `['DRAFT', 'REVIEWED']` to `['DRAFT', 'REVIEWED', 'PRINTED']` (or however the current guard is expressed). Reject `COMPLETED` and `CANCELLED`.

### 4. Tests

- Unit test the use case: add-only, remove-only, update-only, mixed, duplicate-inputId rejection, unknown-input rejection. Use `InMemoryRepository`.
- E2E controller test (supertest): review with new inputs array on DRAFT, REVIEWED, and PRINTED tickets; reject on COMPLETED.

## Frontend changes

### 1. Sheet UI — `frontend/src/domains/field-ticket/components/sheets/configure-field-ticket-sheet.tsx`

Replace the current highlight-only list with the same add/remove pattern already used by the edit/create forms:

- Use `useFieldArray` on an `inputs` field in the sheet's React Hook Form schema.
- Add an "Adicionar insumo" row at the bottom: reuse the async input picker from `edit-field-ticket-form.tsx` (async popover via `useAsyncSelect` calling `listInputs()`, filtered to exclude already-added inputs) + a dosage numeric field (masked text per the `no-input-number` rule). On add, push to the array.
- Each row shows: input name, dosage field (editable), highlight toggle, remove button.
- No new components needed — extract the input picker and row renderer from the edit form into a shared `field-ticket-input-rows.tsx` under `components/forms/` if duplication gets painful. Otherwise import the existing handler logic directly.

### 2. Action — `frontend/src/domains/field-ticket/actions/review-field-ticket.ts`

Change payload from `{ ..., highlightedInputIds }` to `{ ..., inputs: [{ inputId, dosagePer100L, highlighted }] }`. Same shape as the create/edit form already produces.

### 3. API client — `frontend/src/domains/field-ticket/api/review-field-ticket.ts`

Mirror the backend schema change.

### 4. Execution panel status gate — `frontend/src/domains/field-ticket/components/execution-panel/execution-panel.tsx`

Widen the click-to-configure guard from `DRAFT | REVIEWED` to `DRAFT | REVIEWED | PRINTED`. Update any disabled-state rendering accordingly.

### 5. Playwright E2E

Extend the existing execution-panel spec to cover:
- Reviewing a DRAFT with a new insumo added (visible in ticket after close).
- Reviewing a REVIEWED ticket with an insumo removed.
- Reviewing a PRINTED ticket.
- COMPLETED ticket is not clickable.

## Non-goals

- No new endpoints.
- No schema migration in Prisma (inputs table already supports add/delete).
- No changes to the edit form, create form, or print layout.

## Verification

```
cd backend && contextzip pnpm test -- review-field-ticket
cd backend && contextzip pnpm test:e2e -- review-field-ticket
cd frontend && contextzip pnpm test:e2e -- execution-panel
```

Manually: open execution panel, click a DRAFT ticket, add 2 inputs via picker, remove 1, save, reopen — changes persist. Repeat on a PRINTED ticket.
