# Reminders

Living notes for gotchas, pitfalls, and things that went wrong during development. This file prevents the same mistakes from happening twice.

---

## How Claude uses this document

- **At the start of every task:** Read gotchas to avoid repeating known mistakes.
- **After fixing a bug or discovering a pitfall:** Add it here immediately.
- **During Phase 4 (Documentation):** Add new gotchas discovered, remove resolved ones.

---

## Health check

health-check-counter: 4

The `health-check-counter` tracks how many features have been merged since the last `/health-check`. It is incremented by `/feature`, `/new-domain`, and `/small-change` after a PR is successfully created. When it reaches `HEALTH_CHECK_THRESHOLD` (defined in CLAUDE.md), Claude warns before starting the next task. It resets to 0 after `/health-check` completes.

---

## Maintenance

- Add gotchas as soon as they are discovered — do not wait until the end of a session
- Remove gotchas when the root cause is fixed or the information is no longer relevant
- Keep entries specific: include file path, library name, or exact scenario
- This file is read at the start of every implementation task — keep it concise

---

## Gotchas

<!-- Add gotchas discovered during development. Remove them when they are resolved or no longer relevant. -->
<!-- Format: one line per gotcha, prefixed with dash. Be specific — include file, library, or context. -->

(none)

---

## Future improvements (post-MVP)

### Inventory
- Lot/batch tracking with expiry dates for inputs
- Minimum stock alerts (notify when input stock is low)
- Soft reservation: Schedule reserves stock for planning; alert when more product needs to be purchased to continue the plan
- Input enrichment: manufacturer, commercial code, concentration fields
- Unit price calculation from total purchase value (analytics/reporting)
- Automatic StockMovement (exit) triggered by FieldTicket finalization — implement when FieldTicket domain is built (MVP, not post-MVP)

### Supplier
- Price comparison across suppliers for the same input
- Purchase history and supplier performance tracking

