# API contract — blackboard-swe-coordination

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tickets` | `{ "title": String, "description": String, "submittedBy"?: String }` | `202 { "ticketId": String }` | `BoardEndpoint` → `TicketQueue` |
| `GET` | `/api/tickets/{id}` | — | `200 Ticket` / `404` | `BoardEndpoint` ← `TicketEntity` |
| `GET` | `/api/boards` | — | `200 [ BoardRow... ]` | `BoardEndpoint` ← `BlackboardView` |
| `GET` | `/api/boards?stage=…` | — | `200 [ BoardRow... ]` (filtered client-side) | `BoardEndpoint` ← `BlackboardView` |
| `GET` | `/api/boards/{ticketId}` | — | `200 BoardRow` / `404` | `BoardEndpoint` ← `BlackboardView` |
| `GET` | `/api/boards/sse` | — | `text/event-stream` (one event per stage change) | `BoardEndpoint` ← `BlackboardView` |
| `POST` | `/api/signoff/{ticketId}` | `{ "approved": boolean, "by": String, "notes"?: String }` | `200 SignoffDecision` / `404` | `BoardEndpoint` → `SignoffEntity`, resumes `SignoffWorkflow` |
| `GET` | `/api/signoff/{ticketId}` | — | `200 SignoffRecord` / `404` | `BoardEndpoint` ← `SignoffEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Ticket

```json
{
  "ticketId": "t-9b1…",
  "title": "Rate-limiter middleware",
  "description": "Add sliding-window rate limiting at 100 req/min per client.",
  "submittedBy": "ui",
  "status": "IN_PROGRESS",
  "createdAt": "2026-06-29T08:10:00Z",
  "completedAt": null
}
```

### BoardRow

```json
{
  "ticketId": "t-9b1…",
  "title": "Rate-limiter middleware",
  "stage": "CODED",
  "taskBreakdownSummary": "Implement a sliding-window rate-limiter as server-side middleware.",
  "archApproach": "Single-process middleware backed by an in-memory sliding counter.",
  "backendArtifactSummary": "Sliding-window counter and middleware wiring.",
  "backendFileCount": 2,
  "frontendArtifactSummary": "Response-header banner component.",
  "frontendFileCount": 1,
  "reviewApproved": null,
  "securityClearToMerge": null,
  "testsAllPassed": null,
  "integrationApproach": null,
  "stuckAlert": null,
  "createdAt": "2026-06-29T08:10:00Z",
  "mergedAt": null
}
```

Lifecycle fields are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### SignoffRecord

```json
{
  "ticketId": "t-9b1…",
  "request": {
    "requestedBy": "controller",
    "requestedAt": "2026-06-29T08:15:42Z"
  },
  "decision": {
    "by": "lead-engineer-1",
    "approved": true,
    "notes": "All checks green. Approved.",
    "decidedAt": "2026-06-29T08:16:30Z"
  }
}
```

### SSE event format

```
event: board-update
data: { "ticketId": "t-9b1…", "stage": "AWAITING_SIGNOFF", ... }
```

One event per stage transition. Clients reconcile by `ticketId` and update the pipeline display for that ticket.
