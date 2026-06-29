# Data model — location-discovery-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PlaceCandidate` | `placeId` | `String` | no | Stable identifier from the seeded dataset. |
| | `name` | `String` | no | Venue display name. |
| | `category` | `String` | no | Category label (e.g., "Coffee Shop", "Museum"). |
| | `distanceMeters` | `double` | no | Distance from the search origin, computed at sanitize time. |
| | `latitude` | `double` | no | Venue latitude (kept in entity; not sent to LLM). |
| | `longitude` | `double` | no | Venue longitude (kept in entity; not sent to LLM). |
| | `address` | `String` | no | Street address. |
| | `rating` | `Optional<Double>` | yes | Aggregate rating, if available. |
| | `reviewCount` | `Optional<Integer>` | yes | Number of reviews backing the rating. |
| `SearchRequest` | `searchId` | `String` | no | UUID minted by `SearchEndpoint`. |
| | `query` | `String` | no | User's natural-language search string. |
| | `rawLatitude` | `double` | no | Pre-sanitization latitude. Audit-only. |
| | `rawLongitude` | `double` | no | Pre-sanitization longitude. Audit-only. |
| | `radiusMeters` | `int` | no | Search radius: 500, 1000, or 2000. |
| | `categoryFilter` | `Optional<String>` | yes | Category constraint; `Optional.empty()` = all categories. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedCoordinate` | `boundingBoxToken` | `String` | no | Coarse bounding box, e.g., `"bbox:37.77-37.79:-122.42--122.40"`. |
| | `candidatePlaces` | `List<PlaceCandidate>` | no | Places within the requested radius; this is what the agent sees. |
| `PlaceResult` | `placeId` | `String` | no | MUST equal a `placeId` in `SanitizedCoordinate.candidatePlaces`. |
| | `name` | `String` | no | Copied from the candidate. |
| | `category` | `String` | no | Copied from the candidate. |
| | `distanceMeters` | `double` | no | Copied from the candidate. |
| | `relevanceScore` | `int` | no | Agent-assigned fit score, 1–10. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| `PlaceRecommendation` | `summary` | `String` | no | 1–3 sentence overview of the result set. |
| | `places` | `List<PlaceResult>` | no | Ordered by `relevanceScore` descending. |
| | `decidedAt` | `Instant` | no | When the agent returned the result. |
| `EvalResult` | `score` | `int` | no | 1–5 quality score. |
| | `rationale` | `String` | no | One sentence from `EvaluationScorer`. |
| | `evaluatedAt` | `Instant` | no | When `EvaluationScorer` finished. |
| `Search` (entity state) | `searchId` | `String` | no | — |
| | `request` | `Optional<SearchRequest>` | yes | Populated after `SearchSubmitted`. |
| | `sanitized` | `Optional<SanitizedCoordinate>` | yes | Populated after `CoordinateSanitized`. |
| | `recommendation` | `Optional<PlaceRecommendation>` | yes | Populated after `RecommendationRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `SearchStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SearchSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Search` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SearchStatus`: `SUBMITTED`, `COORDINATE_SANITIZED`, `DISCOVERING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`SearchEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SearchSubmitted` | `request` | → SUBMITTED |
| `CoordinateSanitized` | `sanitized` | → COORDINATE_SANITIZED |
| `DiscoveryStarted` | — | → DISCOVERING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDATION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `SearchFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Search.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SearchRow` mirrors `Search` minus `request.rawLatitude` and `request.rawLongitude` (the audit log keeps those). The UI fetches the raw coordinates on demand via `GET /api/searches/{id}` and reads `request.rawLatitude` / `request.rawLongitude` from the JSON.

The view declares ONE query: `getAllSearches: SELECT * AS searches FROM search_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SearchTasks.java`)

```java
public final class SearchTasks {
  public static final Task<PlaceRecommendation> DISCOVER_PLACES = Task
      .name("Discover places")
      .description("Read the attached candidate place list and produce a PlaceRecommendation ranked by relevance to the query")
      .resultConformsTo(PlaceRecommendation.class);

  private SearchTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
