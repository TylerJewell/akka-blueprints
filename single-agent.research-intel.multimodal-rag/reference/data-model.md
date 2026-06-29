# Data model — multiformat-hybrid-rag

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChunkRecord` | `chunkId` | `String` | no | Stable id assigned when corpus is loaded. |
| | `sourceTitle` | `String` | no | Human-readable title of the source document. |
| | `sourceUri` | `String` | no | URI or file path of the source (for auditing). |
| | `format` | `SourceFormat` | no | TEXT / MARKDOWN / PDF_PAGE / IMAGE_CAPTION. |
| | `content` | `String` | no | Normalised plaintext ready for retrieval scoring. |
| | `rawContent` | `String` | no | Original before normalisation; stored for citation excerpts. |
| | `pageOrSequence` | `int` | no | Page number (PDF) or insertion order (others). |
| `RetrievedChunk` | `chunkId` | `String` | no | References a `ChunkRecord`. |
| | `sourceTitle` | `String` | no | Copied from `ChunkRecord`. |
| | `format` | `SourceFormat` | no | Copied from `ChunkRecord`. |
| | `excerpt` | `String` | no | First 300 chars of `ChunkRecord.content`. |
| | `relevanceScore` | `double` | no | 0..1 Jaccard score against the query question. |
| `Citation` | `chunkId` | `String` | no | MUST equal a `RetrievedChunk.chunkId` in the same query. |
| | `sourceTitle` | `String` | no | From the chunk the agent cited. |
| | `format` | `SourceFormat` | no | From the chunk the agent cited. |
| | `passageExcerpt` | `String` | no | Short verbatim passage used as evidence (≤ 80 words). |
| `ResearchAnswer` | `decision` | `AnswerDecision` | no | ANSWERED or NO_RESULT. |
| | `answerText` | `String` | yes | 2–6-sentence narrative. Null when `NO_RESULT`. |
| | `noResultReason` | `String` | yes | One-sentence rationale. Null when `ANSWERED`. |
| | `citations` | `List<Citation>` | no | Non-empty when `ANSWERED`; empty when `NO_RESULT`. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `question` | `String` | no | User-supplied research question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `RetrievalResult` | `chunks` | `List<RetrievedChunk>` | no | Top-K chunks, sorted descending by score. |
| | `totalChunksSearched` | `int` | no | Size of the full `ChunkStore` at retrieval time. |
| | `retrievedAt` | `Instant` | no | When `ChunkStore.topK` returned. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `retrieval` | `Optional<RetrievalResult>` | yes | Populated after `RetrievalCompleted`. |
| | `answer` | `Optional<ResearchAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SourceFormat`: `TEXT`, `MARKDOWN`, `PDF_PAGE`, `IMAGE_CAPTION`.
`AnswerDecision`: `ANSWERED`, `NO_RESULT`.
`QueryStatus`: `SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `NO_RESULT`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `RetrievalCompleted` | `retrieval` | → RETRIEVING |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWERED or NO_RESULT (resolved from `answer.decision`) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. The UI fetches the full query on demand via `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<ResearchAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached corpus chunks and return a ResearchAnswer with citations anchoring every factual claim")
      .resultConformsTo(ResearchAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
