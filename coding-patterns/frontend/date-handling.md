# Date Handling Pattern

## Rule

All dates from the backend are stored and transmitted in **UTC** (e.g., `2026-03-20T00:00:00.000Z`). The frontend must treat API dates as UTC when extracting date parts (year, month, day, weekday). Using local-time methods on UTC dates causes off-by-one-day bugs in negative UTC offset timezones (e.g., GMT-3).

## When to use UTC vs local time

| Context | Method | Example |
|---------|--------|---------|
| Extracting date parts from API data | `getUTC*()` | `d.getUTCDate()`, `d.getUTCMonth()`, `d.getUTCFullYear()`, `d.getUTCDay()` |
| Getting "today" for the user's perspective | `get*()` (local) | `new Date().getDate()`, `new Date().getFullYear()` |
| Creating a Date from a `YYYY-MM-DD` string for local display | Append `T12:00:00` | `new Date('2026-03-20T12:00:00')` — noon local avoids day shift |
| Comparing API dates to user-selected dates | Both sides must use the same convention | See examples below |

## Correct patterns

### Extracting ISO date string from an API date

```typescript
// CORRECT — uses UTC methods
function toISODateString(date: Date): string {
  const y = date.getUTCFullYear()
  const m = String(date.getUTCMonth() + 1).padStart(2, '0')
  const d = String(date.getUTCDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}
```

### Formatting a short date from an API date

```typescript
// CORRECT — uses UTC methods
function formatShortDate(date: Date): string {
  const dayNames = ['Dom', 'Seg', 'Ter', 'Qua', 'Qui', 'Sex', 'Sáb']
  const d = new Date(date)
  return `${dayNames[d.getUTCDay()]} ${String(d.getUTCDate()).padStart(2, '0')}/${String(d.getUTCMonth() + 1).padStart(2, '0')}`
}
```

### Getting today's date string (user perspective)

```typescript
// CORRECT — local time is appropriate for "today"
function getTodayISO(): string {
  const now = new Date()
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}-${String(now.getDate()).padStart(2, '0')}`
}
```

### Navigating dates from a user-selected base (e.g., day offset)

```typescript
// CORRECT — noon local anchor avoids day shift
function offsetDate(base: string, days: number): string {
  const d = new Date(base + 'T12:00:00')
  d.setDate(d.getDate() + days)
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
}
```

## Anti-patterns

```typescript
// WRONG — local time on an API date (bug in GMT-3: 2026-03-20T00:00:00Z → 2026-03-19T21:00 local)
const d = new Date(ticket.date)
const day = d.getDate()        // Returns 19 instead of 20 in GMT-3!
const month = d.getMonth()
const weekday = d.getDay()

// WRONG — using toLocaleDateString for filtering (locale-dependent format)
const dateStr = new Date(ticket.date).toLocaleDateString()

// WRONG — using toISOString().slice(0, 10) on a modified Date
// toISOString always returns UTC, but if the Date was created with local time it may shift
```

## Reference implementation

See `frontend/src/domains/schedule/components/week-view/date-utils.ts` for the canonical date utility functions used across the schedule domain. All functions use `getUTC*()` consistently.

## Checklist for code review

- [ ] Every `getFullYear()`, `getMonth()`, `getDate()`, `getDay()` on an API date is flagged — must be `getUTC*()`
- [ ] Date comparisons between API dates and user-selected dates use the same convention
- [ ] Date formatting for display uses UTC methods when the source is an API date
- [ ] `new Date(isoString)` is never followed by local-time extraction without explicit justification
