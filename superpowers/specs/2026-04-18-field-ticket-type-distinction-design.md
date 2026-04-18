# Field Ticket Type Distinction — Design Spec

**Date:** 2026-04-18
**Status:** Approved

## Overview

Distinguish SPRAYING (Pulverização) and FERTIGATION (Fertirrigação) field tickets with type-specific validation, forms, and print layouts. No schema changes — all required fields already exist as nullable columns.

## Backend

### Approach
Single use cases with type-discriminated validation (Approach A). No new use cases, controllers, or repositories.

### ReviewFieldTicket — DRAFT → REVIEWED transition

| Type | Required fields |
|------|----------------|
| SPRAYING | `vehicleId` + `implementId` (existing behavior) |
| FERTIGATION | none — review exists only to edit inputs |

Remove the global invariant that requires `vehicleId` + `implementId` for all tickets. Replace with conditional check by `operationType`.

### FinalizeFieldTicket — PRINTED → COMPLETED transition

| Type | Required fields |
|------|----------------|
| SPRAYING | `hourmeterStart` + `hourmeterEnd` + `startTime` + `endTime` + `operatorName` |
| FERTIGATION | `startTime` + `endTime` + `operatorName` |

Add hourmeter validation for SPRAYING (currently not enforced).

### FieldTicket entity
- Update invariant: `vehicleId + implementId` required only when `operationType === SPRAYING`
- No new fields, no migration needed

## Frontend

### Review form
- **SPRAYING:** current form unchanged — vehicle, implement, waterL, bar, turbine, nozzleCount, pressure, gearNumber, gearType, pH
- **FERTIGATION:** inputs list only — no equipment section

### Finalization form
- **SPRAYING:** startTime, endTime, hourmeterStart, hourmeterEnd, "Tratorista"
- **FERTIGATION:** startTime, endTime, "Irrigante" — no hourmeter fields

### Labels
| Field | SPRAYING | FERTIGATION |
|-------|----------|-------------|
| `operatorName` | Tratorista | Irrigante |

### Print layout
- **SPRAYING:** 6 tickets per A4 landscape (existing)
- **FERTIGATION:** 8 tickets per A4 landscape — compact layout, no equipment block

## Testing

### Backend unit
- `ReviewFieldTicket`: FERTIGATION passes without vehicleId/implementId; SPRAYING fails without them
- `FinalizeFieldTicket`: SPRAYING fails without hourmeterStart/End; FERTIGATION passes without hourmeter

### Backend E2E
- Review controller: both types, conditional validation
- Finalize controller: both types

### Frontend Playwright
- Review FERTIGATION: no equipment section, edit inputs, confirm
- Review SPRAYING: no regression
- Finalize FERTIGATION: correct fields, "Irrigante" label
- Finalize SPRAYING: correct fields, "Tratorista" label, hourmeter required
- Print FERTIGATION: 8 per page

## Out of scope
- New operation types beyond SPRAYING / FERTIGATION
- Type-specific input units or dosage validation
- Separate print templates per type beyond density difference
