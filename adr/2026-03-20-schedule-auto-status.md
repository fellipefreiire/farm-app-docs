# 2026-03-20 — Automatic schedule status with review checkpoint

**Context:** After FieldTicket absorbed ScheduleOperation (2026-03-19), the Schedule entity is a lightweight grouping. Its 4-state manual status (PLANNED/ACTIVE/COMPLETED/CANCELLED with explicit Activate/Complete buttons) created a second control layer that duplicated what FieldTicket statuses already express. The original purpose of the status was to enforce "only 1 active schedule per field" — not to burden the user with manual lifecycle management.
**Decision:** Schedule status becomes automatic. New 5-state machine: PLANNED → ACTIVE → UNDER_REVIEW → COMPLETED, with CANCELLED from PLANNED or ACTIVE. The user only controls cancellation explicitly — all other transitions are automatic or review-driven.

**Status lifecycle:**
- **Creation:** Born ACTIVE if no active schedule exists for the field. Born PLANNED if field already has an active schedule.
- **ACTIVE → UNDER_REVIEW:** Automatic when all FieldTickets reach terminal state (COMPLETED or CANCELLED). Triggered by subscriber on FieldTicketCompletedEvent / FieldTicketCancelledEvent.
- **UNDER_REVIEW → COMPLETED:** User reviews the schedule (summary of operations, notes/observations field, confirmation). Upon confirming, system shows PLANNED schedules for the field — user chooses which to activate next, or creates a new one.
- **UNDER_REVIEW → ACTIVE:** If user adds a new FieldTicket during review (identified missing operations), schedule reverts to ACTIVE.
- **PLANNED/ACTIVE → CANCELLED:** Explicit user action with mandatory reason. Requires resolving PRINTED tickets first (finalize or cancel each one individually). DRAFT/REVIEWED tickets are auto-cancelled. COMPLETED tickets are preserved as historical record.
- **Post-cancellation:** User is offered to activate a PLANNED schedule, copy the cancelled one as a new schedule (all operations as DRAFT), or dismiss.

**Multiple PLANNED per field:** Allowed. User chooses which PLANNED schedule to promote during the review/cancellation flow — no automatic FIFO.

**PLANNED schedules can have FieldTickets:** Yes. Users can plan operations in advance on a PLANNED schedule before it becomes ACTIVE.

**Schedule entity changes:** New fields: `reviewNotes` (string, optional), `reviewedAt` (DateTime, optional). New enum value: `UNDER_REVIEW`.

**Harvest follows Schedule:** ACTIVE/UNDER_REVIEW → Harvest ACTIVE. COMPLETED → Harvest COMPLETED. CANCELLED → Harvest CANCELLED.

**Copy cancelled schedule:** CopySchedule works on any status (including CANCELLED). Copies ALL operations as DRAFT to the new schedule, regardless of their original status.

**Options considered:** (A) Remove PLANNED, keep 3 manual states / (B) Simplify to 2 states (OPEN/CLOSED) / (C) Automatic status with review checkpoint (chosen) / (D) Keep as-is
**Rationale:** Option C matches the real workflow — the user plans operations, they get executed, the system detects completion. The review checkpoint gives the user control at the right moment (when all work is done) instead of burdening them with status buttons throughout. The "resolve PRINTED before cancel" rule prevents data loss. The "choose next schedule" flow during review/cancellation naturally handles the multiple-PLANNED scenario.
**Consequences:** Remove `ActivateScheduleUseCase`, `CompleteScheduleUseCase`, and their controllers/tests. Add `ReviewScheduleUseCase` (confirms review, transitions to COMPLETED, promotes selected next schedule). Add `OnFieldTicketTerminalCheckSchedule` subscriber (auto-transition to UNDER_REVIEW). Modify `CancelScheduleUseCase` to enforce PRINTED resolution, auto-cancel DRAFT/REVIEWED tickets, and offer next-schedule selection. Modify `CreateScheduleUseCase` to check field for existing ACTIVE schedule. Remove "Ativar" and "Concluir" buttons from frontend. Add review screen with summary, notes, and next-schedule picker. Add PRINTED resolution flow in cancellation dialog. Update `ScheduleStatus` enum in Prisma schema (add UNDER_REVIEW). Migrate existing PLANNED schedules: if field has no ACTIVE schedule, promote the oldest PLANNED to ACTIVE.
