# Data model — basic-rag-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DocumentSource` | `docId` | `String` | no | Stable document identifier. |
| | `title` | `String` | no | Human-readable document title. |
| | `content` | `String` | no | Full document body text. |
| | `sourceUrl` | `String` | no | Canonical URL of the source document. |
| `IndexedChunk` | `chunkId` | `String` | no | Stable short id (`ck-<8 hex>`). |
| | `docId` | `String` | no | Parent document identifier. |
| | `text` | `String` | no | Chunk text (~200 words). |
| | `sourceUrl` | `String` | no | MUST equal the parent `DocumentSource.sourceUrl`. |
| | `chunkIndex` | `int` | no | Zero-based position of this chunk within the parent document. |
| `IndexedCorpus` | `corpusId` | `String` | no | Identifier of the loaded corpus (e.g., `"default"`). |
| | `chunks` | `List<IndexedChunk>` | no | All indexed chunks; possibly empty if corpus is empty. |
| | `indexedAt` | `Instant` | no | When the INGEST task returned. |
| `Citation` | `chunkId` | `String` | no | MUST equal an `IndexedChunk.chunkId` from the session's `IndexedCorpus`. |
| | `sourceUrl` | `String` | no | MUST equal the `sourceUrl` of the cited `IndexedChunk`. Checked by `AnswerGuardrail`. |
| | `excerpt` | `String` | no | Short quoted passage from the cited chunk. |
| `RagAnswer` | `question` | `String` | no | The original question. |
| | `answerText` | `String` | no | Grounded answer text. `"No relevant content found in the indexed corpus."` when no chunks were retrieved. |
| | `citations` | `List<Citation>` | no | Non-empty for substantive answers; empty only when `answerText` is the no-content sentinel. |
| | `answeredAt` | `Instant` | no | When the QUERY task returned. |
| `GuardrailOutcome` | `verdict` | `String` | no | `"PASSED"` or `"BLOCKED"`. |
| | `reason` | `String` | no | `"All citations grounded in indexed corpus."` on PASSED; structured rejection reason on BLOCKED. |
| | `evaluatedAt` | `Instant` | no | When `AnswerGuardrail` ran. |
| `RagSessionRecord` (entity state) | `sessionId` | `String` | no | — |
| | `question` | `Optional<String>` | yes | Populated after `SessionCreated`. |
| | `corpus` | `Optional<IndexedCorpus>` | yes | Populated after `CorpusIndexed`. |
| | `answer` | `Optional<RagAnswer>` | yes | Populated after `AnswerDrafted`. |
| | `guardrailOutcome` | `Optional<GuardrailOutcome>` | yes | Populated after `GuardrailApplied`. |
| | `status` | `RagSessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `RagSessionRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RagSessionStatus`: `CREATED`, `INDEXING`, `INDEXED`, `ANSWERING`, `ANSWERED`, `BLOCKED`, `FAILED`.

## Events (`RagSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `question: String` | → CREATED |
| `IngestStarted` | — | → INDEXING |
| `CorpusIndexed` | `corpus: IndexedCorpus` | → INDEXED |
| `QueryStarted` | — | → ANSWERING |
| `AnswerDrafted` | `answer: RagAnswer` | → ANSWERED |
| `GuardrailApplied` | `verdict, reason, evaluatedAt` | no status change (audit-only during retries; written alongside `AnswerDrafted` on the final accepted answer) |
| `AnswerBlocked` | `reason: String` | → BLOCKED (terminal on budget-exhaustion) |
| `SessionFailed` | `reason: String` | → FAILED (terminal on step error) |

`emptyState()` returns `RagSessionRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RagSessionRow` mirrors `RagSessionRecord` exactly. The UI fetches the full row via `GET /api/sessions/{id}` and streams updates via `GET /api/sessions/sse`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM rag_session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`RagTasks.java`)

```java
public final class RagTasks {
  public static final Task<IndexedCorpus> INGEST_CORPUS = Task
      .name("Ingest corpus")
      .description("Load documents from the default corpus and index them into the in-process vector store")
      .resultConformsTo(IndexedCorpus.class);

  public static final Task<RagAnswer> ANSWER_QUERY = Task
      .name("Answer query")
      .description("Retrieve relevant chunks for the question and compose a grounded answer with citations")
      .resultConformsTo(RagAnswer.class);

  private RagTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## In-process VectorStore

`VectorStore` is a singleton plain class (not an Akka primitive). It stores `List<IndexedChunk>` in memory and provides:

- `void store(List<IndexedChunk> chunks)` — replaces the current store; called by `IngestTools.indexChunks`.
- `List<IndexedChunk> retrieve(String question, int topK)` — tokenises `question`, scores each chunk by matching-token count, returns the top-K chunks by score. Deterministic given the same corpus and question.

Because `ingestStep` writes `CorpusIndexed` and the workflow advances to `queryStep` only after that write, there is no concurrency hazard between writes and reads.

## Citation integrity contract

`AnswerGuardrail` enforces: for every `citation` in `RagAnswer.citations`, `citation.sourceUrl` MUST appear in `IndexedCorpus.chunks[].sourceUrl` as recorded on `RagSessionEntity`. The check is a set-membership lookup, not a substring or fuzzy match.
