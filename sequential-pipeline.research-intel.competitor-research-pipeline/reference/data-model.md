# Data model — competitor-research-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SearchResult` | `title` | `String` | no | Page title from the web result. |
| | `url` | `String` | no | Canonical URL of the source page. |
| | `excerpt` | `String` | no | Quoted passage retrieved by the SEARCH phase. |
| | `fetchedAt` | `Instant` | no | When the SEARCH phase recorded the result. |
| `SearchResultSet` | `results` | `List<SearchResult>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `searchedAt` | `Instant` | no | When the SEARCH task returned. |
| `ProfileField` | `fieldName` | `String` | no | One of: `pricing_model`, `primary_use_case`, `notable_integrations`, `known_differentiators`, `data_residency_stance`. |
| | `value` | `String` | no | Extracted text. `"(not found)"` or `"(no results found)"` when no evidence exists. |
| | `supportingUrl` | `String` | no | MUST equal a `SearchResult.url` from the upstream `SearchResultSet` (or a placeholder slug in the no-results case). |
| `ProfileSummary` | `fields` | `List<ProfileField>` | no | Exactly five entries, one per declared dimension. |
| | `domainClassification` | `String` | no | One of: `analytics`, `dataops`, `ml-platform`, `ai-app-framework`, `observability`, `other`. |
| | `summarizedAt` | `Instant` | no | When the SUMMARIZE task returned. |
| `NotionPageRef` | `pageId` | `String` | no | Notion internal page ID. Synthetic slug in mock mode. |
| | `pageUrl` | `String` | no | Public URL of the Notion page. Synthetic URL in mock mode. |
| | `publishedAt` | `Instant` | no | When `writeToNotion` returned. |
| `CompetitorProfile` | `competitorId` | `String` | no | Matches the `CompetitorEntity` key. |
| | `name` | `String` | no | Competitor display name. |
| | `oneLiner` | `String` | no | One-sentence synthesis from the `ProfileSummary`. |
| | `fields` | `List<ProfileField>` | no | Mirrors `ProfileSummary.fields`. |
| | `notionRef` | `NotionPageRef` | no | Non-null; set by `writeToNotion`. |
| | `profiledAt` | `Instant` | no | When the PUBLISH task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `PublishQualityScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `SEARCH` / `SUMMARIZE` / `PUBLISH`. |
| | `tool` | `String` | no | Name of the rejected tool call. |
| | `reason` | `String` | no | Structured reason from `NotionWriteGuardrail` (`phase-violation:`, `schema-violation:`, or `scope-violation:`). |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `CompetitorRecord` (entity state) | `competitorId` | `String` | no | — |
| | `name` | `Optional<String>` | yes | Populated after `CompetitorCreated`. |
| | `searchResults` | `Optional<SearchResultSet>` | yes | Populated after `ResultsFetched`. |
| | `summary` | `Optional<ProfileSummary>` | yes | Populated after `SummaryProduced`. |
| | `profile` | `Optional<CompetitorProfile>` | yes | Populated after `ProfilePublished`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `PublishEvaluated`. |
| | `status` | `CompetitorStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CompetitorCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `CompetitorRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`CompetitorStatus`: `CREATED`, `SEARCHING`, `SEARCHED`, `SUMMARIZING`, `SUMMARIZED`, `PUBLISHING`, `PUBLISHED`, `EVALUATED`, `FAILED`.

`ResearchPhase` (used by `@FunctionTool` annotations and `NotionWriteGuardrail`): `SEARCH`, `SUMMARIZE`, `PUBLISH`.

## Events (`CompetitorEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CompetitorCreated` | `name: String` | → CREATED |
| `SearchStarted` | — | → SEARCHING |
| `ResultsFetched` | `searchResults: SearchResultSet` | → SEARCHED |
| `SummarizeStarted` | — | → SUMMARIZING |
| `SummaryProduced` | `summary: ProfileSummary` | → SUMMARIZED |
| `PublishStarted` | — | → PUBLISHING |
| `ProfilePublished` | `profile: CompetitorProfile` | → PUBLISHED |
| `PublishEvaluated` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `CompetitorFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CompetitorRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CompetitorRow` mirrors `CompetitorRecord` exactly. The UI fetches the full row via `GET /api/competitors/{id}` and streams updates via `GET /api/competitors/sse`.

The view declares ONE query: `getAllCompetitors: SELECT * AS competitors FROM competitor_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`ResearchTasks.java`)

```java
public final class ResearchTasks {
  public static final Task<SearchResultSet> SEARCH_COMPETITOR = Task
      .name("Search competitor")
      .description("Gather raw web results about a competitor by calling searchCompetitor and fetchPage")
      .resultConformsTo(SearchResultSet.class);

  public static final Task<ProfileSummary> SUMMARIZE_FINDINGS = Task
      .name("Summarize findings")
      .description("Extract structured profile fields and classify the competitor's domain from raw results")
      .resultConformsTo(ProfileSummary.class);

  public static final Task<CompetitorProfile> PUBLISH_PROFILE = Task
      .name("Publish profile")
      .description("Build a Notion page from the profile summary and write it to the Notion competitor database")
      .resultConformsTo(CompetitorProfile.class);

  private ResearchTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `SearchTools`, `SummarizeTools`, and `PublishTools` carries a `ResearchPhase` constant. `NotionWriteGuardrail` reads this constant before the tool body runs and applies the per-status accept matrix (see eval-matrix.yaml G1). For `writeToNotion` calls that pass the phase gate, the guardrail additionally runs the Notion schema and scope checks against `NotionSchema` constants. The tool registry is built once at startup; the guardrail reads it for every call.

## NotionSchema constants

```java
public final class NotionSchema {
  public static final Set<String> REQUIRED_FIELDS = Set.of(
      "Name", "Pricing Model", "Primary Use Case", "Domain", "Source URL");

  public static final Map<String, String> FIELD_TYPES = Map.of(
      "Name",             "title",
      "Pricing Model",    "rich_text",
      "Primary Use Case", "rich_text",
      "Domain",           "select",
      "Source URL",       "url");

  // Resolved from ${?NOTION_DATABASE_ID} at startup; "mock-db-id" in mock mode.
  public static final String ALLOWED_DATABASE_ID = /* resolved at startup */;

  private NotionSchema() {}
}
```
