# 2026-03-16 — Harvest status PLANNED replaced with UNSCHEDULED *(superseded by 2026-03-20)*

**Context:** The Schedule domain drives the Harvest lifecycle. A Harvest without a Schedule should reflect that state explicitly.
**Decision:** Replace Harvest status `PLANNED` with `UNSCHEDULED`. Harvest activation is no longer manual — it's triggered by ScheduleActivatedEvent. When a Schedule is cancelled, Harvest reverts to UNSCHEDULED.
**Options considered:** (A) Add UNSCHEDULED alongside PLANNED / (B) Replace PLANNED with UNSCHEDULED (chosen)
**Rationale:** Having both PLANNED and UNSCHEDULED creates ambiguity. UNSCHEDULED clearly communicates "no schedule exists yet". The Schedule domain has its own PLANNED status for "schedule exists but not yet active".
**Consequences:** Manual ActivateHarvest endpoint removed. Harvest activation only via Schedule domain events. All existing PLANNED data migrated to UNSCHEDULED.
**Note:** This decision was superseded on 2026-03-20 — UNSCHEDULED was eliminated and PLANNED was restored as the initial Harvest status, since Schedule is now auto-created and always exists.
