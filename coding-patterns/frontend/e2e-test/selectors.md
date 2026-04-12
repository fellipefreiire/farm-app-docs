# E2E — Selector conventions

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Actual `data-testid` patterns used in the codebase:

| Element | Pattern | Example |
|---|---|---|
| Page container | `<domain>-page` | `fields-page`, `harvests-page` |
| Data table | `data-table` | `data-table` |
| Empty state | `empty-state` | `empty-state` |
| Search input | `query-search-input` | `query-search-input` |
| Create button | `<entity>-create-button` | `field-create-button` |
| Create form | `<entity>-create-form` | `field-create-form` |
| Create submit | `<entity>-create-submit` | `field-create-submit` |
| Edit form | `<entity>-edit-form` | `field-edit-form` |
| Edit submit | `<entity>-edit-submit` | `field-edit-submit` |
| Delete dialog | `<entity>-delete-dialog` | `field-delete-dialog` |
| Delete form | `<entity>-delete-form` | `field-delete-form` |
| Delete confirm | `<entity>-delete-confirm` | `field-delete-confirm` |
| Actions popover | `<entity>-actions-<id>` | `field-actions-00000...` |
| Edit button | `<entity>-edit-btn-<id>` | `field-edit-btn-00000...` |
| Delete button | `<entity>-delete-btn-<id>` | `field-delete-btn-00000...` |
| Toggle status | `<entity>-toggle-status-btn-<id>` | `field-toggle-status-btn-00000...` |
| Stacked sheet | `stacked-sheet` | `stacked-sheet` |
| Sheet close | `stacked-sheet-close` | `stacked-sheet-close` |
| Status cell | `<entity>-status-<id>` | `harvest-status-30000...` |
| Image cell | `<entity>-image` | `field-image`, `crop-type-image` |
| Detail page | `<entity>-detail-page` | `variety-detail-page` |
| Detail edit btn | `<entity>-edit-detail-btn` | `variety-edit-detail-btn` |

---
