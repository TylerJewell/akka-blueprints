# Data model — rag-baseline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Passage` | `passageId` | `String` | no | Stable id from the corpus JSONL. |
| | `documentTitle` | `String` | no | Source document label. |
| | `text` | `String` | no | Full passage body (2–4 sentences). |
| | `similarityScore` | `double` | no | Cosine similarity to the query, 0..1. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | User-submitted natural-language question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `RetrievedPassages` | `passages` | `List<Passage>` | no | Top-K passages from the vector store (0–N). |
| | `topK` | `int` | no | Retrieval parameter (fixed at 5). |
| `Citation` | `passageId` | `String` | no | MUST equal an id from `RetrievedPassages.passages`. |
| | `passageExcerpt` | `String` | no | Verbatim excerpt from the cited passage text. |
| | `claimSupported` | `String` | no | Part of the answer this passage anchors. |
| `GroundedAnswer` | `answerText` | `String` | no | Prose answer, 1–4 sentences. |
| | `citations` | `List<Citation>` | no | May be empty when no evidence found. |
| | `usedPassages` | `int` | no | Must equal `citations.size()`. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5 groundedness score. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `GroundednessEvaluator` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `retrieved` | `Optional<RetrievedPassages>` | yes | Populated after `PassagesRetrieved`. |
| | `answer` | `Optional<GroundedAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `SUBMITTED`, `PASSAGES_RETRIEVED`, `ANSWERING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `PassagesRetrieved` | `retrieved` | → PASSAGES_RETRIEVED |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWER_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. No fields are stripped — questions and answers are not sensitive in the baseline corpus. A deployer with sensitive questions may choose to omit `request.questionText` from the view and serve it only on the full entity GET.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<GroundedAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the attached passages and produce a GroundedAnswer anchored in retrieved evidence")
      .resultConformsTo(GroundedAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## VectorStore indexing

`VectorStore` loads `src/main/resources/corpus/passages.jsonl` at startup. Each entry `{passageId, documentTitle, text}` is tokenised by whitespace; a TF-IDF-style vector over the vocabulary is stored per passage. `search(String query, int topK)` tokenises the query, computes cosine similarity against all stored vectors, and returns the top-K results sorted by descending similarity score. No external model call; no HTTP. This is intentionally rudimentary — the blueprint's point is the Akka orchestration, not embedding quality.
