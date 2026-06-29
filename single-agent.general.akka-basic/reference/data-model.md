# Data model — akka-basic

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PromptRequest` | `runId` | `String` | no | UUID minted by `PromptEndpoint`. |
| | `promptText` | `String` | no | User-supplied text to process. |
| | `categoryHint` | `CategoryHint` | no | Enum value from the user. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `PromptResult` | `outputText` | `String` | no | Non-empty processed output. |
| | `confidenceScore` | `double` | no | In `[0.0, 1.0]`. |
| | `category` | `ResultCategory` | no | Resolved category enum value. |
| | `producedAt` | `Instant` | no | When the agent returned. |
| `Run` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<PromptRequest>` | yes | Populated after `RunSubmitted`. |
| | `result` | `Optional<PromptResult>` | yes | Populated after `ResultRecorded`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Run` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`CategoryHint`: `SUMMARISE`, `EXTRACT`, `CLASSIFY`, `AUTO`.  
`ResultCategory`: `SUMMARISE`, `EXTRACT`, `CLASSIFY`.  
`RunStatus`: `SUBMITTED`, `PROCESSING`, `COMPLETED`, `FAILED`.

## Events (`PromptEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunSubmitted` | `request` | → SUBMITTED |
| `RunStarted` | — | → PROCESSING |
| `ResultRecorded` | `result` | → COMPLETED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Run.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `Run`. The view declares ONE query: `getAllRuns: SELECT * AS runs FROM prompt_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`PromptTasks.java`)

```java
public final class PromptTasks {
  public static final Task<PromptResult> PROCESS_PROMPT = Task
      .name("Process prompt")
      .description("Read the submitted prompt and return a PromptResult with outputText, confidenceScore, and category")
      .resultConformsTo(PromptResult.class);

  private PromptTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
