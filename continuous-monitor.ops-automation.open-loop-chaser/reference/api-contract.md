# API contract — open-loop-chaser

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/items` | — | `200 [ ActionItemRow... ]` (sorted newest-first). Optional `?status=…&sourceType=…` | `ActionItemEndpoint` ← `ActionItemView` |
| `GET` | `/api/items/{id}` | — | `200 ActionItemRow` / `404` | `ActionItemEndpoint` ← `ActionItemView` |
| `POST` | `/api/items/{id}/confirm-owner` | `{ "confirmedOwner": String, "confirmedBy": String }` | `200 ActionItemRow` (ownerConfirmation populated) | `ActionItemEndpoint` → `ActionItemEntity` |
| `POST` | `/api/items/{id}/close` | `{ "closedBy": String, "closureNote": String }` | `200 ActionItemRow` (status now CLOSED) | `ActionItemEndpoint` → `ActionItemEntity` |
| `POST` | `/api/items/{id}/snooze` | `{ "snoozedBy": String, "durationMinutes": Integer }` | `200 ActionItemRow` (status now SNOOZED) | `ActionItemEndpoint` → `ActionItemEntity` |
| `GET` | `/api/items/sse` | — | `text/event-stream` | `ActionItemEndpoint` ← `ActionItemView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### ActionItemRow

```json
{
  "itemId": "ai-4c7a…",
  "sourceEventId": "se-9f1b…",
  "sourceType": "MEETING_NOTES",
  "sanitized": {
    "redactedText": "Send the updated specification document.",
    "redactedAuthor": "[REDACTED-AUTHOR]",
    "piiCategoriesFound": ["name", "email"]
  },
  "extracted": {
    "description": "Send the updated specification document.",
    "suggestedOwner": "[REDACTED-PERSON-1]",
    "dueDate": "2026-07-04",
    "sourceEventId": "se-9f1b…"
  },
  "ownerConfirmation": {
    "confirmedOwner": "alex.kim",
    "confirmedBy": "operator-7",
    "confirmedAt": "2026-06-28T09:12:00Z"
  },
  "lastNudge": {
    "recipientHint": "team lead",
    "subject": "Reminder: Send the updated specification document.",
    "body": "This item has been open for 35 minutes…",
    "composedAt": "2026-06-28T09:45:00Z"
  },
  "snooze": null,
  "closure": null,
  "nudgeCount": 1,
  "status": "NUDGED",
  "detectedAt": "2026-06-28T08:45:00Z",
  "finishedAt": null
}
```

### SSE event format

```
event: item-update
data: { "itemId": "ai-4c7a…", "status": "NUDGED", "nudgeCount": 1, … }
```

One event per state transition. Clients reconcile by `itemId`.

### confirm-owner body

```json
{ "confirmedOwner": "alex.kim", "confirmedBy": "operator-7" }
```

### close body

```json
{ "closedBy": "operator-7", "closureNote": "Completed in sprint review." }
```

### snooze body

```json
{ "snoozedBy": "operator-7", "durationMinutes": 120 }
```
