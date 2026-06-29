# Data model — support-next-steps

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TicketRequest` | `ticketId` | `String` | no | UUID minted by `TicketEndpoint`. |
| | `subject` | `String` | no | User-supplied subject line. |
| | `ticketBody` | `String` | no | Pre-sanitization ticket body. Audit-only. |
| | `productArea` | `String` | no | `Billing`, `Authentication`, or `Data Export`. |
| | `submittedBy` | `String` | no | Support agent identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedTicket` | `redactedBody` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","account-number","address","person-name"]`. |
| `ResolutionRef` | `caseId` | `String` | no | Past-case identifier from the resolution library. |
| | `summary` | `String` | no | One-sentence summary of the resolved case. |
| | `resolutionDate` | `String` | no | ISO-8601 date the case was resolved. |
| `RecommendedStep` | `stepNumber` | `int` | no | 1-based rank. |
| | `description` | `String` | no | Actionable verb-phrase. |
| | `actionType` | `ActionType` | no | Enum value. |
| | `confidence` | `Confidence` | no | Enum value. |
| | `resolutionRef` | `String` | no | `caseId` from resolution library, or empty string. |
| `RecommendationSet` | `overallConfidence` | `OverallConfidence` | no | Enum value. |
| | `rationale` | `String` | no | 1–2 sentences. |
| | `steps` | `List<RecommendedStep>` | no | 2–5 ranked steps. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `RecommendationScorer` finished. |
| `Ticket` (entity state) | `ticketId` | `String` | no | — |
| | `request` | `Optional<TicketRequest>` | yes | Populated after `TicketSubmitted`. |
| | `sanitized` | `Optional<SanitizedTicket>` | yes | Populated after `TicketSanitized`. |
| | `recommendation` | `Optional<RecommendationSet>` | yes | Populated after `RecommendationRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `TicketStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TicketSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Ticket` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ActionType`: `VERIFY`, `ESCALATE`, `CONFIGURE`, `INFORM`, `INVESTIGATE`.
`Confidence`: `HIGH`, `MEDIUM`, `LOW`.
`OverallConfidence`: `HIGH`, `MEDIUM`, `LOW`.
`TicketStatus`: `SUBMITTED`, `SANITIZED`, `ADVISING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`TicketEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TicketSubmitted` | `request` | → SUBMITTED |
| `TicketSanitized` | `sanitized` | → SANITIZED |
| `AdvisingStarted` | — | → ADVISING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDATION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `TicketFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Ticket.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`TicketRow` mirrors `Ticket` minus `request.ticketBody` (the audit log keeps that). The UI fetches the raw body on demand via `GET /api/tickets/{id}` and reads `request.ticketBody` from the JSON.

The view declares ONE query: `getAllTickets: SELECT * AS tickets FROM ticket_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TicketTasks.java`)

```java
public final class TicketTasks {
  public static final Task<RecommendationSet> ADVISE_TICKET = Task
      .name("Advise ticket")
      .description("Read the attached sanitized ticket and resolution library, then produce a RecommendationSet")
      .resultConformsTo(RecommendationSet.class);

  private TicketTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
