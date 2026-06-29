# Data model — bm25-rag-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PassageRef` | `passageId` | `String` | no | Stable corpus id (e.g. `p-014`). |
| | `title` | `String` | no | Section heading from the corpus entry. |
| | `snippet` | `String` | no | First 200 chars of the passage body. |
| | `bm25Score` | `float` | no | BM25 relevance score from this query. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | User-supplied natural-language question. |
| | `topK` | `int` | no | How many passages to retrieve (1–10). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `RetrievedPassages` | `passages` | `List<PassageRef>` | no | Ordered by bm25Score descending. |
| | `totalDocsScanned` | `int` | no | Total corpus size at search time. |
| `Citation` | `passageId` | `String` | no | MUST equal a passageId in the retrieved set. |
| | `quotedFragment` | `String` | no | Verbatim excerpt from the passage body. |
| `QueryAnswer` | `answerType` | `AnswerType` | no | Enum value. |
| | `answerText` | `String` | no | 1–4 sentence synthesis. |
| | `citations` | `List<Citation>` | no | Empty for NO_ANSWER; ≥ 1 for DIRECT/PARTIAL. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `GroundingResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `scoredAt` | `Instant` | no | When `GroundingScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `passages` | `Optional<RetrievedPassages>` | yes | Populated after `PassagesAttached`. |
| | `answer` | `Optional<QueryAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `grounding` | `Optional<GroundingResult>` | yes | Populated after `GroundingScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AnswerType`: `DIRECT`, `PARTIAL`, `NO_ANSWER`.
`QueryStatus`: `SUBMITTED`, `PASSAGES_ATTACHED`, `ANSWERING`, `ANSWER_RECORDED`, `SCORED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `PassagesAttached` | `passages` | → PASSAGES_ATTACHED |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWER_RECORDED |
| `GroundingScored` | `grounding` | → SCORED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. The passage bodies are not stored in the view row — only the `PassageRef` list (id, title, snippet, bm25Score). Full passage bodies are loaded from the `CorpusIndex` by `GroundingScorer` at scoring time and are accessible to callers by fetching `GET /api/queries/{id}`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<QueryAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached passage documents and produce a QueryAnswer citing only the provided passages")
      .resultConformsTo(QueryAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## CorpusIndex entry schema (JSONL)

Each line in `src/main/resources/corpus/passages.jsonl` is:

```json
{
  "passageId": "p-014",
  "title": "Scaled Dot-Product Attention",
  "body": "We scale the dot products by 1/sqrt(d_k)...",
  "url": "https://example.org/transformer-docs/attention"
}
```

`body` is the full passage text indexed by BM25. `snippet` in `PassageRef` is `body.substring(0, min(200, body.length()))`.
