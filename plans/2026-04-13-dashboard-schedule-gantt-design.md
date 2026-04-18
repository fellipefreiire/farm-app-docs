# Dashboard Schedule Gantt — Design Spec

## Summary

Grouped Gantt visualization of active schedules on the Dashboard page, organized by field (talhão). Each field appears as a row with operations positioned proportionally on a real-date axis. A vertical "today" marker shows current progress.

## Decisions

| Decision | Choice |
|---|---|
| Location | Dashboard (`/dashboard`) |
| Layout | Gantt proportional with real dates |
| Interaction | Expand inline on row click |
| Filter | Only ACTIVE schedules |
| Backend | Dedicated endpoint `GET /schedules/dashboard` |
| Frontend | New component `ScheduleGanttDashboard` |
| Pagination | None (few active schedules expected) |

## Backend

### Endpoint

`GET /schedules/dashboard` — no query params, returns all ACTIVE schedules hydrated with Field, Harvest, and operations.

### Use Case

`GetDashboardSchedules` — queries schedules with status ACTIVE, joins Harvest → Field and aggregates FieldTickets into operations grouped by day.

### Response

```json
{
  "data": [
    {
      "scheduleId": "uuid",
      "status": "ACTIVE",
      "field": {
        "id": "uuid",
        "name": "Talhão A1"
      },
      "harvest": {
        "id": "uuid",
        "name": "Safra 2026",
        "cropType": "Soja",
        "variety": "Intacta",
        "startDate": "2026-04-06T00:00:00Z"
      },
      "totalDays": 14,
      "currentDay": 5,
      "operations": [
        {
          "day": 1,
          "type": "SPRAYING",
          "label": "P1",
          "spanDays": 2,
          "completed": true
        },
        {
          "day": 3,
          "type": "FERTIGATION",
          "label": "F1",
          "spanDays": 1,
          "completed": true
        }
      ]
    }
  ]
}
```

- `currentDay`: calculated on backend as diff between today (UTC) and `harvest.startDate`
- `operations[].day`: 1-indexed day relative to `harvest.startDate`
- `operations[].completed`: true if all FieldTickets for that operation are FINALIZED
- `operations[].label`: sequential label by type (P1, P2... for SPRAYING, F1, F2... for FERTIGATION)

### Data path

`Schedule → harvestId → Harvest → fieldId → Field`
`Schedule → FieldTicket[] → grouped by day + operationType`

## Frontend

### Components

All in `frontend/src/domains/schedule/components/dashboard/`:

- `schedule-gantt-dashboard.tsx` — container, fetches data via `useQuery`, renders header + rows
- `gantt-header.tsx` — date axis with "today" marker, calculated from global date range across all schedules
- `gantt-row.tsx` — info column (field name, crop, variety, status) + operation blocks positioned proportionally
- `gantt-row-detail.tsx` — expanded content shown below row on click, displays operations list with completion status from the same data (no extra fetch)

### Positioning logic

- Horizontal axis = global date range (earliest `startDate` to latest `startDate + totalDays`) across all active schedules
- Each operation block: `left = (day - 1) / totalRange * 100%`, `width = spanDays / totalRange * 100%`
- "Today" vertical line: same calculation using current date offset from range start

### Colors

- Green (`#1a6b3c`) — Spraying, completed
- Blue (`#2563eb`) — Fertigation, completed
- Gray (`#374151`) — Any type, pending

### Expand inline

Clicking a row toggles `gantt-row-detail.tsx` below it. Shows the operations list with type, day, span, and completed status. Data comes from the already-fetched response. Includes a "Ver cronograma" link to `/schedules/[id]`.

### Dashboard page

`ScheduleGanttDashboard` replaces the current placeholder content. Title: "Cronogramas Ativos".

## Out of scope

- Filters by crop type, variety, or harvest
- Drag-and-drop operations
- Export/print
- PLANNED or UNDER_REVIEW schedules
- Pagination
- Lazy-loaded boleta details in expand
