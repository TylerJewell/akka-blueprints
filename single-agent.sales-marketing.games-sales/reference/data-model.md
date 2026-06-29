# Data model — games-sales

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GamePreferences` | `query` | `String` | no | Free-text customer query. |
| | `platform` | `Optional<String>` | yes | e.g. `"PS5"`, `"Nintendo Switch"`. |
| | `genre` | `Optional<String>` | yes | e.g. `"action"`, `"RPG"`. |
| | `maxBudgetCents` | `Optional<Integer>` | yes | Maximum spend in cents. |
| | `sessionId` | `String` | no | UUID minted by `SalesEndpoint`. |
| | `customerId` | `String` | no | Customer identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GameSuggestion` | `catalogId` | `String` | no | MUST match an entry in `game-catalog.jsonl`. |
| | `title` | `String` | no | Human-readable game title. |
| | `platform` | `String` | no | Platform string, e.g. `"PS5"`. |
| | `priceCents` | `int` | no | Price in cents. |
| | `confidenceScore` | `double` | no | Match confidence in `[0.0, 1.0]`. |
| | `rationale` | `String` | no | 1–2 sentences explaining the match. |
| | `upsellNote` | `Optional<String>` | yes | Related product note, or absent. |
| `SalesRecommendation` | `suggestions` | `List<GameSuggestion>` | no | 1–5 entries, ranked best-first. |
| | `summary` | `String` | no | 1–2 sentence overview. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `FollowUpQuery` | `sessionId` | `String` | no | The session being refined. |
| | `refinementText` | `String` | no | Customer's refinement text. |
| | `submittedAt` | `Instant` | no | When the follow-up was submitted. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `preferences` | `Optional<GamePreferences>` | yes | Populated after `SessionStarted`. |
| | `recommendation` | `Optional<SalesRecommendation>` | yes | Populated after `RecommendationRecorded`. |
| | `followUpRecommendation` | `Optional<SalesRecommendation>` | yes | Populated after `FollowUpRecorded`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Session` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SessionStatus`: `QUERYING`, `RECOMMENDED`, `FOLLOW_UP_QUERYING`, `FOLLOW_UP_RECOMMENDED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `preferences` | → QUERYING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDED |
| `FollowUpStarted` | `refinementText: String` | → FOLLOW_UP_QUERYING |
| `FollowUpRecorded` | `followUpRecommendation` | → FOLLOW_UP_RECOMMENDED |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with all `Optional` fields as `Optional.empty()` and `status = QUERYING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session`. The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SessionTasks.java`)

```java
public final class SessionTasks {
  public static final Task<SalesRecommendation> RECOMMEND_GAMES = Task
      .name("Recommend games")
      .description("Read the customer query and preferences and produce a SalesRecommendation with up to 5 ranked suggestions")
      .resultConformsTo(SalesRecommendation.class);

  private SessionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Game catalog entry (JSONL schema)

Each line in `src/main/resources/sample-events/game-catalog.jsonl`:

```json
{
  "catalogId": "elden-ring-ps5",
  "title": "Elden Ring",
  "platform": "PS5",
  "genre": "action",
  "priceCents": 5999,
  "releaseYear": 2022,
  "rating": "M"
}
```

`RecommendationGuardrail` loads all `catalogId` values at startup into a `Set<String>` and validates every `GameSuggestion.catalogId` against this set.
