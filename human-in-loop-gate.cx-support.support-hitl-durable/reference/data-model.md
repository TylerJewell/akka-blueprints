# Data model

Every record, event, enum, and view row the generated system defines.

## `Ticket` (TicketEntity state + TicketsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Ticket id; equals the workflow id and entity id |
| `subject` | `Optional<String>` | yes | The submitted ticket subject line |
| `body` | `Optional<String>` | yes | The submitted ticket body (not persisted after triage) |
| `status` | `TicketStatus` | no | Lifecycle status |
| `submittedAt` | `Optional<Instant>` | yes | When the ticket was received |
| `triageCategory` | `Optional<String>` | yes | Classification: billing, technical, account, general |
| `triagePriority` | `Optional<String>` | yes | Urgency: low, medium, high |
| `triageSummary` | `Optional<String>` | yes | Sanitized issue description (no PII) |
| `approvedAt` | `Optional<Instant>` | yes | When the operator approved |
| `approvedBy` | `Optional<String>` | yes | Operator identity who approved |
| `operatorNote` | `Optional<String>` | yes | Operator's note at approval time |
| `escalatedAt` | `Optional<Instant>` | yes | When the ticket was escalated |
| `escalatedBy` | `Optional<String>` | yes | Operator identity who escalated |
| `escalationReason` | `Optional<String>` | yes | Reason for escalation |
| `resolvedAt` | `Optional<String>` | yes | ISO-8601 resolution time |
| `resolution` | `Optional<String>` | yes | Customer-facing response text |

Every nullable lifecycle field is `Optional<T>` because `Ticket` is the view row type (Lesson 6). `emptyState()` returns `Ticket.initial("")` with no `commandContext()` reference (Lesson 3).

## `TicketStatus` enum

`TRIAGED`, `APPROVED`, `ESCALATED`, `RESOLVED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `TicketTriaged` | `recordTriage` after TriageAgent returns | sets `submittedAt`, `triageCategory`, `triagePriority`, `triageSummary`, status → `TRIAGED` |
| `TicketApproved` | `approve` command (operator via API) | sets `approvedAt`, `approvedBy`, `operatorNote`, status → `APPROVED` |
| `TicketEscalated` | `escalate` command (operator via API) | sets `escalatedAt`, `escalatedBy`, `escalationReason`, status → `ESCALATED` |
| `TicketResolved` | `recordResolution` after ResolutionAgent returns | sets `resolvedAt`, `resolution`, status → `RESOLVED` |

## Domain records

- `TicketTriage(String category, String priority, String summary)` — TriageAgent result.
- `OperatorDecision(String approvedBy, String note)` — approve payload.
- `TicketResolution(String response, String resolvedAt)` — ResolutionAgent result.

## View row type

`TicketsView` rows are `Ticket`. One query: `getAllTickets` → `SELECT * AS tickets FROM tickets_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (SupportTasks.java)

- `TRIAGE` — `Task.name(...).resultConformsTo(TicketTriage.class)`.
- `RESOLVE` — `Task.name(...).resultConformsTo(TicketResolution.class)`.
