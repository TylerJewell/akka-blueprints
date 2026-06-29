# Data model — long-rag-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Chunk` | `chunkId` | `String` | no | Stable short id for this chunk (e.g. `c-eu-ai-001`). |
| | `docId` | `String` | no | Id of the parent document. |
| | `docTitle` | `String` | no | Human-readable title of the parent document. |
| | `pageStart` | `int` | no | First page of the chunk within the document. |
| | `pageEnd` | `int` | no | Last page of the chunk within the document. |
| | `text` | `String` | no | Verbatim passage text for this chunk. |
| | `indexedAt` | `Instant` | no | When the chunk was ingested into the in-process corpus. |
| `ChunkWindow` | `chunks` | `List<Chunk>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `queryText` | `String` | no | The original research query that produced this window. |
| | `retrievedAt` | `Instant` | no | When the RETRIEVE task returned. |
| `Finding` | `findingId` | `String` | no | Short stable id (`f-<8 hex>`). |
| | `text` | `String` | no | The factual finding, paraphrased with inline `[chunkId]` citation. |
| | `supportingChunkId` | `String` | no | MUST equal a `Chunk.chunkId` from the upstream `ChunkWindow`. |
| `Theme` | `themeId` | `String` | no | Slug. MUST be unique within a `Synthesis`. |
| | `label` | `String` | no | Human-readable theme name. |
| | `findingIds` | `List<String>` | no | Each entry MUST equal a `Finding.findingId` from the same `Synthesis`. |
| `Synthesis` | `themes` | `List<Theme>` | no | Possibly empty. |
| | `findings` | `List<Finding>` | no | Possibly empty. |
| | `synthesizedAt` | `Instant` | no | When the SYNTHESIZE task returned. |
| `Citation` | `chunkId` | `String` | no | MUST equal a `Chunk.chunkId` from the upstream `ChunkWindow`. |
| | `docTitle` | `String` | no | Display label (from `Chunk.docTitle`). |
| | `pageRange` | `String` | no | `"<pageStart>-<pageEnd>"` string, e.g. `"47-49"`. |
| `ReportSection` | `themeId` | `String` | no | MUST equal a `Theme.themeId` from the upstream `Synthesis`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–4 sentences with inline `[chunkId]` tags; each sentence ≥ 10 words must carry at least one tag. |
| | `citations` | `List<Citation>` | no | ≥ 2 distinct `Citation.chunkId` values preferred (E1 rule 2). |
| `ResearchReport` | `title` | `String` | no | 1-line title. |
| | `abstractText` | `String` | no | 1-paragraph abstract with inline citations. |
| | `sections` | `List<ReportSection>` | no | `sections.size() == synthesis.themes.size()` (E1 rule 4). |
| | `writtenAt` | `Instant` | no | When the REPORT task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `CoverageScorer` finished. |
| `CitationFlag` | `sentence` | `String` | no | The uncited sentence the guardrail flagged. |
| | `missingCitation` | `String` | no | Structured reason from `CitationGuardrail`. |
| | `windowSize` | `int` | no | Number of chunks in the active `ChunkWindow` at the time of the flag. |
| | `flaggedAt` | `Instant` | no | When the guardrail fired. |
| `QueryRecord` (entity state) | `queryId` | `String` | no | — |
| | `queryText` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `chunkWindow` | `Optional<ChunkWindow>` | yes | Populated after `ChunksRetrieved`. |
| | `synthesis` | `Optional<Synthesis>` | yes | Populated after `FindingsSynthesized`. |
| | `report` | `Optional<ResearchReport>` | yes | Populated after `ReportWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `citationFlags` | `List<CitationFlag>` | no | Appended on every `CitationFlagged` event; empty on the happy path. |

Every nullable lifecycle field on `QueryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `CREATED`, `RETRIEVING`, `RETRIEVED`, `SYNTHESIZING`, `SYNTHESIZED`, `REPORTING`, `REPORTED`, `EVALUATED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `queryText: String` | → CREATED |
| `RetrieveStarted` | — | → RETRIEVING |
| `ChunksRetrieved` | `chunkWindow: ChunkWindow` | → RETRIEVED |
| `SynthesizeStarted` | — | → SYNTHESIZING |
| `FindingsSynthesized` | `synthesis: Synthesis` | → SYNTHESIZED |
| `ReportStarted` | — | → REPORTING |
| `ReportWritten` | `report: ResearchReport` | → REPORTED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `CitationFlagged` | `sentence, missingCitation, windowSize, flaggedAt` | no status change (audit-only) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `citationFlags = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryRecord` exactly. The UI fetches the full row via `GET /api/queries/{id}` and streams updates via `GET /api/queries/sse`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`DocumentTasks.java`)

```java
public final class DocumentTasks {
  public static final Task<ChunkWindow> RETRIEVE_CHUNKS = Task
      .name("Retrieve chunks")
      .description("Search the document corpus for relevant chunks using sliding-window retrieval, then fetch their full text.")
      .resultConformsTo(ChunkWindow.class);

  public static final Task<Synthesis> SYNTHESIZE_FINDINGS = Task
      .name("Synthesize findings")
      .description("Extract factual findings from chunks, then group findings into themes.")
      .resultConformsTo(Synthesis.class);

  public static final Task<ResearchReport> WRITE_REPORT = Task
      .name("Write report")
      .description("Compose a ResearchReport whose ReportSections mirror the Synthesis themes one-to-one, citing chunk ids.")
      .resultConformsTo(ResearchReport.class);

  private DocumentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Citation-tagged tool outputs

`RetrieveTools`, `SynthesizeTools`, and `ReportTools` produce outputs that include or reference chunk ids. `CitationGuardrail` reads the active `ChunkWindow` chunk-id set from `TaskDef.metadata("chunkIds")` and matches `[chunkId]` tags in the model response against that set. The tool registry builds the id set once per task invocation; the guardrail reads it for every model response in that task.
