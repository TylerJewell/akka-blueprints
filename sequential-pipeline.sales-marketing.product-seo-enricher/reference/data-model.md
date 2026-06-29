# Data model — product-seo-enricher

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SerpEntry` | `title` | `String` | no | Page title from the SERP listing. |
| | `url` | `String` | no | Canonical url of the search result. |
| | `position` | `int` | no | 1-based rank on the SERP. |
| | `snippet` | `String` | no | Quoted passage from the SERP snippet. |
| | `fetchedAt` | `Instant` | no | When the FETCH phase recorded this entry. |
| `SerpResult` | `entries` | `List<SerpEntry>` | no | Possibly empty; J5 demonstrates the empty path. |
| | `fetchedAt` | `Instant` | no | When the FETCH task returned. |
| `KeywordCandidate` | `keyword` | `String` | no | The candidate keyword term. |
| | `frequency` | `int` | no | Number of SERP entries containing this keyword. |
| | `sourceUrl` | `String` | no | MUST equal a `SerpEntry.url` from the upstream `SerpResult`. |
| `CompetitorSignal` | `domain` | `String` | no | Registrable domain (e.g. `example.com`). |
| | `title` | `String` | no | Page title of the competitor's SERP entry. |
| | `position` | `int` | no | Best SERP rank for this domain. |
| | `angle` | `String` | no | ≤ 10-word description of the competitor's positioning. |
| `SerpAnalysis` | `keywords` | `List<KeywordCandidate>` | no | Possibly empty. |
| | `competitors` | `List<CompetitorSignal>` | no | Possibly empty. |
| | `analyzedAt` | `Instant` | no | When the ANALYZE task returned. |
| `RankedKeyword` | `keyword` | `String` | no | MUST equal a `KeywordCandidate.keyword` from the upstream `SerpAnalysis`. |
| | `relevanceScore` | `double` | no | 0.0–1.0 scale; frequency / max_frequency. |
| | `rationale` | `String` | no | One clause naming the backing evidence. |
| `ProductEnrichment` | `productName` | `String` | no | Product name passed at submission. |
| | `rankedKeywords` | `List<RankedKeyword>` | no | Possibly empty (J5). |
| | `competitorSummary` | `String` | no | Prose summary referencing ≥ 1 `CompetitorSignal.domain`. |
| | `recommendedMetaDescription` | `String` | no | ≤ 160 characters (QualityScorer rule 2). |
| | `writtenAt` | `Instant` | no | When the ENRICH task returned. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `FETCH` / `ANALYZE` / `ENRICH`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `BrowserGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `EnrichmentRecord` (entity state) | `enrichmentId` | `String` | no | — |
| | `productName` | `Optional<String>` | yes | Populated after `EnrichmentCreated`. |
| | `serpResult` | `Optional<SerpResult>` | yes | Populated after `SerpFetched`. |
| | `serpAnalysis` | `Optional<SerpAnalysis>` | yes | Populated after `SerpAnalyzed`. |
| | `enrichment` | `Optional<ProductEnrichment>` | yes | Populated after `EnrichmentWritten`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityScored`. |
| | `status` | `EnrichmentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `EnrichmentCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `EnrichmentRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`EnrichmentStatus`: `CREATED`, `FETCHING`, `FETCHED`, `ANALYZING`, `ANALYZED`, `ENRICHING`, `ENRICHED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `BrowserGuardrail`): `FETCH`, `ANALYZE`, `ENRICH`.

## Events (`EnrichmentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `EnrichmentCreated` | `productName: String` | → CREATED |
| `FetchStarted` | — | → FETCHING |
| `SerpFetched` | `serpResult: SerpResult` | → FETCHED |
| `AnalyzeStarted` | — | → ANALYZING |
| `SerpAnalyzed` | `serpAnalysis: SerpAnalysis` | → ANALYZED |
| `EnrichStarted` | — | → ENRICHING |
| `EnrichmentWritten` | `enrichment: ProductEnrichment` | → ENRICHED |
| `QualityScored` | `quality: QualityResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `EnrichmentFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `EnrichmentRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`EnrichmentRow` mirrors `EnrichmentRecord` exactly. The UI fetches the full row via `GET /api/enrichments/{id}` and streams updates via `GET /api/enrichments/sse`.

The view declares ONE query: `getAllEnrichments: SELECT * AS enrichments FROM enrichment_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`SeoTasks.java`)

```java
public final class SeoTasks {
  public static final Task<SerpResult> FETCH_SERP = Task
      .name("Fetch SERP")
      .description("Retrieve top search-engine result entries for a product by calling fetchSerpPage and fetchResultSnippet")
      .resultConformsTo(SerpResult.class);

  public static final Task<SerpAnalysis> ANALYZE_SERP = Task
      .name("Analyze SERP")
      .description("Extract keyword candidates and identify competitor signals from the SERP entries")
      .resultConformsTo(SerpAnalysis.class);

  public static final Task<ProductEnrichment> WRITE_ENRICHMENT = Task
      .name("Write enrichment")
      .description("Rank keyword candidates by relevance and compose a recommended meta description")
      .resultConformsTo(ProductEnrichment.class);

  private SeoTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `FetchTools`, `AnalyzeTools`, and `EnrichTools` carries a `Phase` constant. `BrowserGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.

## QualityScorer rules

Four checks, one point per check satisfied, starting from a base of 1:

1. **Keyword coverage** — every `RankedKeyword.keyword` appears in `SerpAnalysis.keywords[].keyword`.
2. **Meta-description length** — `ProductEnrichment.recommendedMetaDescription.length() ≤ 160`.
3. **Competitor grounding** — `competitorSummary` references at least one `CompetitorSignal.domain`.
4. **Keyword-list non-empty** — `rankedKeywords.size() > 0`.

Score range 1–5. Rationale names the largest gap. `QualityScorer` is deterministic — the same enrichment always scores the same.
