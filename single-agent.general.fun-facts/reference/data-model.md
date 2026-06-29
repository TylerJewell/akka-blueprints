# Data model — fun-facts

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `FactRequest` | `requestId` | `String` | no | UUID minted by `FactEndpoint`. |
| | `topic` | `String` | no | User-supplied topic string. Validated non-blank at endpoint. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `Fact` | `area` | `String` | no | Topical sub-area (e.g., "cognition", "anatomy", "history"). |
| | `statement` | `String` | no | 1–2 sentence fact statement. |
| | `confidence` | `Confidence` | no | Enum value. |
| `FactCollection` | `headlineFact` | `String` | no | One-sentence headline fact. |
| | `facts` | `List<Fact>` | no | 4–6 supporting facts. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `FactRequestState` (entity state) | `requestId` | `String` | no | — |
| | `request` | `Optional<FactRequest>` | yes | Populated after `FactRequested`. |
| | `collection` | `Optional<FactCollection>` | yes | Populated after `CollectionGenerated`. |
| | `status` | `FactRequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `FactRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `FactRequestState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Confidence`: `HIGH`, `MEDIUM`, `LOW`.
`FactRequestStatus`: `PENDING`, `GENERATING`, `GENERATED`, `FAILED`.

## Events (`FactRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `FactRequested` | `request` | → PENDING |
| `GenerationStarted` | — | → GENERATING |
| `CollectionGenerated` | `collection` | → GENERATED (terminal happy) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `FactRequestState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = PENDING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`FactRequestRow` mirrors `FactRequestState`. The full `collection` is included — there is no sensitive field to exclude (unlike blueprints that withhold raw document text from the view). The UI fetches individual requests via `GET /api/fact-requests/{id}` for the detail panel but can also read the full row from the list.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM fact_request_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`FactTasks.java`)

```java
public final class FactTasks {
  public static final Task<FactCollection> GENERATE_FACTS = Task
      .name("Generate facts")
      .description("Generate a structured collection of interesting facts about the given topic")
      .resultConformsTo(FactCollection.class);

  private FactTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
