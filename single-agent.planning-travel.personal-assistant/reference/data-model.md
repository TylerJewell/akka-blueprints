# Data model — personal-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ContextEvent` | `eventId` | `String` | no | Stable id supplied by the caller. |
| | `title` | `String` | no | Human-readable event name. |
| | `date` | `LocalDate` | no | Calendar date of the event. |
| | `time` | `Optional<LocalTime>` | yes | Start time; absent for all-day items. |
| | `location` | `String` | no | Location string (may be redacted by sanitizer). |
| `ContextTask` | `taskId` | `String` | no | Stable id supplied by the caller. |
| | `title` | `String` | no | Task description. |
| | `completed` | `boolean` | no | Completion flag. |
| | `dueDate` | `Optional<LocalDate>` | yes | Optional due date. |
| | `priority` | `Priority` | no | Enum value. |
| `ContextSnapshot` | `events` | `List<ContextEvent>` | no | Current calendar events. |
| | `tasks` | `List<ContextTask>` | no | Current task list. |
| | `timezone` | `String` | no | IANA timezone of the user (e.g. `Europe/Berlin`). |
| `SanitizedContext` | `redactedSnapshot` | `String` | no | JSON string with PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","address","person-name"]`. |
| `ChangeField` | `field` | `String` | no | Name of the field being changed (e.g. `"title"`, `"date"`, `"completed"`). |
| | `oldValue` | `Optional<String>` | yes | Previous value; absent when creating a new item. |
| | `newValue` | `String` | no | Value to write. Always non-empty (guardrail enforces this). |
| `AssistantAction` | `actionType` | `ActionType` | no | Enum value. |
| | `explanation` | `String` | no | 1–2 sentences from the agent. |
| | `changeSet` | `List<ChangeField>` | no | One entry per changed field. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `ActionScorer` finished. |
| `AssistantRequest` | `requestId` | `String` | no | UUID minted by `AssistantEndpoint`. |
| | `naturalLanguageRequest` | `String` | no | User-supplied free-text request. |
| | `rawContext` | `ContextSnapshot` | no | Pre-sanitization context. Audit-only. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AssistantState` (entity state) | `requestId` | `String` | no | — |
| | `request` | `Optional<AssistantRequest>` | yes | Populated after `RequestSubmitted`. |
| | `sanitized` | `Optional<SanitizedContext>` | yes | Populated after `ContextSanitized`. |
| | `action` | `Optional<AssistantAction>` | yes | Populated after `ActionRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `AssistantState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Priority`: `LOW`, `NORMAL`, `HIGH`.
`ActionType`: `CREATE_EVENT`, `CREATE_TASK`, `UPDATE_TASK`, `COMPLETE_TASK`, `SET_REMINDER`, `NO_ACTION`.
`RequestStatus`: `SUBMITTED`, `SANITIZED`, `ACTING`, `ACTION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`AssistantEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RequestSubmitted` | `request` | → SUBMITTED |
| `ContextSanitized` | `sanitized` | → SANITIZED |
| `ActionStarted` | — | → ACTING |
| `ActionRecorded` | `action` | → ACTION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AssistantState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AssistantRow` mirrors `AssistantState` minus `request.rawContext` (the audit log keeps that). The UI fetches the raw context on demand via `GET /api/requests/{id}` and reads `request.rawContext` from the JSON.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM assistant_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AssistantTasks.java`)

```java
public final class AssistantTasks {
  public static final Task<AssistantAction> HANDLE_REQUEST = Task
      .name("Handle assistant request")
      .description("Read the attached context snapshot and produce an AssistantAction satisfying the user's natural-language request")
      .resultConformsTo(AssistantAction.class);

  private AssistantTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
