# Data model — chroma-rag-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | User-supplied question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `RetrievedChunk` | `chunkId` | `String` | no | Stable identifier from the corpus JSONL. |
| | `documentTitle` | `String` | no | Title of the source document. |
| | `passage` | `String` | no | Verbatim text passage from the corpus. |
| | `score` | `double` | no | Cosine similarity 0..1 from ChromaDB. |
| `Citation` | `chunkId` | `String` | no | MUST equal a `chunkId` in the retrieved context. |
| | `documentTitle` | `String` | no | Copied from the matching `RetrievedChunk`. |
| | `passage` | `String` | no | Verbatim passage copied from the matching chunk. |
| `RagAnswer` | `responseText` | `String` | no | 1–4-sentence grounded answer. |
| | `citations` | `List<Citation>` | no | One entry per chunk used; may be empty for refusals. |
| | `chunksRetrieved` | `int` | no | Total chunks given to the agent (not the number cited). |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `GroundednessResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `GroundednessScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `retrievedChunks` | `Optional<List<RetrievedChunk>>` | yes | Populated after `ChunksRetrieved`. |
| | `answer` | `Optional<RagAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `groundedness` | `Optional<GroundednessResult>` | yes | Populated after `GroundednessScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `EVALUATED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `ChunksRetrieved` | `chunks: List<RetrievedChunk>` | → RETRIEVING |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWERED |
| `GroundednessScored` | `groundedness` | → EVALUATED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` minus the full `RetrievedChunk.passage` strings (the view stores only `chunkId` and `documentTitle` per chunk to keep row size manageable). The full passages are available via `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM rag_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RagTasks.java`)

```java
public final class RagTasks {
  public static final Task<RagAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Given the user question and retrieved chunks, produce a grounded RagAnswer with citations")
      .resultConformsTo(RagAnswer.class);

  private RagTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
