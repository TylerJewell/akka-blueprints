# Data model — java-bug-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CandidateFix` | `fixId` | `String` | no | Sequential label minted by the agent (e.g. `"fix-1"`). |
| | `confidenceScore` | `int` | no | Agent-assigned confidence 0–100. |
| | `description` | `String` | no | Actionable verb-phrase describing the fix. |
| | `sourceRef` | `String` | no | Ticket id or KB entry id the fix is derived from. |
| `SearchResult` | `ticketId` | `String` | no | Stable id in the ticket store. |
| | `summary` | `String` | no | One-line description of the ticket. |
| | `ticketStatus` | `TicketStatus` | no | Enum value. |
| | `resolution` | `String` | no | How the ticket was resolved; empty string for open tickets. |
| `BugReport` | `bugId` | `String` | no | UUID minted by `BugEndpoint`. |
| | `title` | `String` | no | User-supplied label. |
| | `rawReport` | `String` | no | Pre-normalization report body. Audit-only. |
| | `category` | `BugCategory` | no | Enum value; `UNKNOWN` triggers auto-detect. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `NormalizedReport` | `cleanedReport` | `String` | no | Credentials, internal hostnames, and tokens redacted; this is what the agent sees. |
| | `redactionCategories` | `List<String>` | no | e.g. `["credential","internal-hostname","stack-trace-token"]`. |
| `ResolutionRecommendation` | `action` | `ResolutionAction` | no | Enum value. |
| | `rationale` | `String` | no | 2–4 sentences; at least 20 characters. |
| | `candidateFixes` | `List<CandidateFix>` | no | Ranked, highest confidence first; non-empty. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `RecommendationScorer` finished. |
| `Bug` (entity state) | `bugId` | `String` | no | — |
| | `report` | `Optional<BugReport>` | yes | Populated after `BugSubmitted`. |
| | `normalized` | `Optional<NormalizedReport>` | yes | Populated after `BugNormalized`. |
| | `searchResults` | `Optional<List<SearchResult>>` | yes | Populated after `SearchResultsAttached`. |
| | `recommendation` | `Optional<ResolutionRecommendation>` | yes | Populated after `RecommendationRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `BugStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BugSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Bug` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TicketStatus`: `OPEN`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`, `WONT_FIX`.
`BugCategory`: `RUNTIME_ERROR`, `NETWORK_FAILURE`, `RESOURCE_EXHAUSTION`, `UNKNOWN`.
`ResolutionAction`: `RESOLVED`, `NEEDS_INVESTIGATION`, `ESCALATE`.
`BugStatus`: `SUBMITTED`, `NORMALIZED`, `SEARCHING`, `SEARCH_COMPLETE`, `RESOLVING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`BugEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BugSubmitted` | `report` | → SUBMITTED |
| `BugNormalized` | `normalized` | → NORMALIZED |
| `SearchStarted` | — | → SEARCHING |
| `SearchResultsAttached` | `searchResults` | → SEARCH_COMPLETE |
| `ResolutionStarted` | — | → RESOLVING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDATION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `BugFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Bug.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BugRow` mirrors `Bug` minus `report.rawReport` (the audit log keeps that). Engineers who need the raw submission fetch `GET /api/bugs/{id}` and read `report.rawReport` from the JSON.

The view declares ONE query: `getAllBugs: SELECT * AS bugs FROM bug_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BugTasks.java`)

```java
public final class BugTasks {
  public static final Task<ResolutionRecommendation> RESOLVE_BUG = Task
      .name("Resolve bug")
      .description("Read the attached normalized bug report and search results, then produce a ResolutionRecommendation with ranked candidate fixes")
      .resultConformsTo(ResolutionRecommendation.class);

  private BugTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
