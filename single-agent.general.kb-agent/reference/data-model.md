# Data model — kb-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `KbDocument` | `documentId` | `String` | no | Stable id supplied by the caller. |
| | `title` | `String` | no | Human-readable document name. |
| | `body` | `String` | no | Full document text, used for TF-IDF indexing. |
| | `collection` | `String` | no | Logical grouping (e.g., `"policies"`, `"product-faq"`). |
| | `indexedAt` | `Instant` | no | When `KbDocumentConsumer` finished indexing. |
| `Passage` | `passageId` | `String` | no | Stable id minted by the retriever (`documentId + "-p-" + chunkIndex`). |
| | `documentId` | `String` | no | Source document. |
| | `documentTitle` | `String` | no | Copied from the source `KbDocument`. |
| | `excerpt` | `String` | no | The passage text (sentence or sliding window) shown to the agent. |
| | `relevanceScore` | `double` | no | TF-IDF cosine similarity score (0.0–1.0). |
| `KbAnswer` | `answer` | `String` | no | Prose response, 1–5 sentences. |
| | `citations` | `List<Citation>` | no | One entry per passage drawn on; empty for NOT_FOUND. |
| | `answerStatus` | `AnswerStatus` | no | Enum value. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Citation` | `passageId` | `String` | no | MUST equal a `passageId` from the retrieved list. |
| | `documentTitle` | `String` | no | Copied from the matching passage. |
| | `excerpt` | `String` | no | The specific phrase the agent drew on. |
| `GroundednessResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `citedSentences` | `int` | no | Count of answer sentences with ≥ 1 passage overlap. |
| | `totalSentences` | `int` | no | Total sentences in the answer. |
| | `evaluatedAt` | `Instant` | no | When `GroundednessScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | The user's question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `passages` | `Optional<List<Passage>>` | yes | Populated after `PassagesAttached`. |
| | `answer` | `Optional<KbAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `groundedness` | `Optional<GroundednessResult>` | yes | Populated after `GroundednessScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AnswerStatus`: `GROUNDED`, `NOT_FOUND`, `PARTIAL`.
`QueryStatus`: `SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `questionText`, `submittedBy` | → SUBMITTED |
| `PassagesAttached` | `passages: List<Passage>` | → RETRIEVING |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer: KbAnswer` | → ANSWER_RECORDED |
| `GroundednessScored` | `groundedness: GroundednessResult` | → EVALUATED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` minus the full `KbDocument.body` text (the index holds that; the view holds passage excerpts only). The UI fetches a full query on demand via `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM kb_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`KbTasks.java`)

```java
public final class KbTasks {
  public static final Task<KbAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached passages and produce a KbAnswer to the question")
      .resultConformsTo(KbAnswer.class);

  private KbTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
