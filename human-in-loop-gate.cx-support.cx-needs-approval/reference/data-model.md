# Data model

Every record, event, enum, and view row the generated system defines.

## `Ticket` (TicketEntity state + TicketsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Ticket id; equals the workflow id and entity id |
| `customerMessage` | `Optional<String>` | yes | The submitted customer support message |
| `status` | `TicketStatus` | no | Lifecycle status |
| `triagedAt` | `Optional<Instant>` | yes | When the triage draft was recorded |
| `responseSummary` | `Optional<String>` | yes | One-sentence issue and resolution summary |
| `responseBody` | `Optional<String>` | yes | Full drafted response body |
| `approvedAt` | `Optional<Instant>` | yes | When the supervisor approved |
| `approvedBy` | `Optional<String>` | yes | Supervisor identity |
| `approverNote` | `Optional<String>` | yes | Supervisor note at approval time |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Supervisor identity at rejection |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `resolvedAt` | `Optional<String>` | yes | ISO-8601 resolution delivery time |
| `confirmationId` | `Optional<String>` | yes | Delivery confirmation reference (e.g. `RES-482910`) |

Every nullable lifecycle field is `Optional<T>` because `Ticket` is the view row type (Lesson 6). `emptyState()` returns `Ticket.initial("")` with no `commandContext()` reference (Lesson 3).

## `TicketStatus` enum

`TRIAGED`, `APPROVED`, `REJECTED`, `RESOLVED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `TicketTriaged` | `recordTriage` after TriageAgent returns | sets `triagedAt`, `responseSummary`, `responseBody`, status → `TRIAGED` |
| `TicketApproved` | `approve` command (supervisor via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `TicketRejected` | `reject` command (supervisor via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `TicketResolved` | `recordResolution` after ResolutionAgent returns | sets `resolvedAt`, `confirmationId`, status → `RESOLVED` |

## Domain records

- `DraftResponse(String summary, String body)` — TriageAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `DeliveredResolution(String confirmationId, String deliveredAt)` — ResolutionAgent result.

## View row type

`TicketsView` rows are `Ticket`. One query: `getAllTickets` → `SELECT * AS tickets FROM tickets_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (SupportTasks.java)

- `TRIAGE` — `Task.name(...).resultConformsTo(DraftResponse.class)`.
- `RESOLVE` — `Task.name(...).resultConformsTo(DeliveredResolution.class)`.
