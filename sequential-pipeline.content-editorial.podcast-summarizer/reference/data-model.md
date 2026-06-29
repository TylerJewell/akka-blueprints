# Data model — podcast-summarizer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Quote` | `quoteId` | `String` | no | Short stable id (`q-<8 hex>`). |
| | `speaker` | `String` | no | Name of the speaker as identified in the transcript. |
| | `text` | `String` | no | Verbatim quoted passage. |
| | `timestampLabel` | `String` | no | Approximate episode position (e.g. `"12:34"`). |
| `QuoteSet` | `quotes` | `List<Quote>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `episodeTitle` | `String` | no | Carried from the submitted episode title. |
| | `extractedAt` | `Instant` | no | When the EXTRACT task returned. |
| `TopicCluster` | `clusterId` | `String` | no | Slug. MUST be unique within a `Segmentation`. |
| | `topicLabel` | `String` | no | 3–5-word human-readable label. |
| | `quoteIds` | `List<String>` | no | Each entry MUST equal a `Quote.quoteId` from the upstream `QuoteSet`. |
| `Segmentation` | `clusters` | `List<TopicCluster>` | no | Possibly empty. |
| | `segmentedAt` | `Instant` | no | When the SEGMENT task returned. |
| `SummarySection` | `clusterId` | `String` | no | MUST equal a `TopicCluster.clusterId` from the upstream `Segmentation`. |
| | `heading` | `String` | no | Section heading (typically the cluster's `topicLabel`). |
| | `body` | `String` | no | 2–4 sentences paraphrasing the cluster's quotes. |
| | `supportingQuoteIds` | `List<String>` | no | Non-empty (E1 rule 2). Every entry MUST equal a `Quote.quoteId` in the upstream `QuoteSet`. |
| `EpisodeSummary` | `episodeTitle` | `String` | no | Carried from the submitted episode title. |
| | `overallTakeaway` | `String` | no | 1-sentence overarching insight. |
| | `sections` | `List<SummarySection>` | no | `sections.size() == segmentation.clusters.size()` (E1 rule 4). |
| | `writtenAt` | `Instant` | no | When the SUMMARIZE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `FidelityScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `EXTRACT` / `SEGMENT` / `SUMMARIZE`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `PhaseGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `EpisodeRecord` (entity state) | `episodeId` | `String` | no | — |
| | `episodeTitle` | `Optional<String>` | yes | Populated after `EpisodeCreated`. |
| | `transcriptText` | `Optional<String>` | yes | Populated after `EpisodeCreated`. |
| | `quotes` | `Optional<QuoteSet>` | yes | Populated after `QuotesExtracted`. |
| | `segmentation` | `Optional<Segmentation>` | yes | Populated after `TopicsSegmented`. |
| | `summary` | `Optional<EpisodeSummary>` | yes | Populated after `SummaryWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `EpisodeStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `EpisodeCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `EpisodeRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`EpisodeStatus`: `CREATED`, `EXTRACTING`, `EXTRACTED`, `SEGMENTING`, `SEGMENTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `PhaseGuardrail`): `EXTRACT`, `SEGMENT`, `SUMMARIZE`.

## Events (`EpisodeEntity`)

| Event | Payload | Transition |
|---|---|---|
| `EpisodeCreated` | `episodeTitle: String, transcriptText: String` | → CREATED |
| `ExtractStarted` | — | → EXTRACTING |
| `QuotesExtracted` | `quotes: QuoteSet` | → EXTRACTED |
| `SegmentStarted` | — | → SEGMENTING |
| `TopicsSegmented` | `segmentation: Segmentation` | → SEGMENTED |
| `SummarizeStarted` | — | → SUMMARIZING |
| `SummaryWritten` | `summary: EpisodeSummary` | → SUMMARIZED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `EpisodeFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `EpisodeRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`EpisodeRow` mirrors `EpisodeRecord` exactly. The UI fetches the full row via `GET /api/episodes/{id}` and streams updates via `GET /api/episodes/sse`.

The view declares ONE query: `getAllEpisodes: SELECT * AS episodes FROM episode_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`TranscriptTasks.java`)

```java
public final class TranscriptTasks {
  public static final Task<QuoteSet> EXTRACT_QUOTES = Task
      .name("Extract quotes")
      .description("Pull representative quotes from the transcript using scanTranscript and fetchSpeakerTurn")
      .resultConformsTo(QuoteSet.class);

  public static final Task<Segmentation> SEGMENT_TOPICS = Task
      .name("Segment topics")
      .description("Cluster the extracted quotes into topic groups using clusterQuotes and labelCluster")
      .resultConformsTo(Segmentation.class);

  public static final Task<EpisodeSummary> WRITE_SUMMARY = Task
      .name("Write summary")
      .description("Draft summary sections for each topic cluster using draftSection and writeOverallTakeaway")
      .resultConformsTo(EpisodeSummary.class);

  private TranscriptTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ExtractTools`, `SegmentTools`, and `SummarizeTools` carries a `Phase` constant. `PhaseGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.
