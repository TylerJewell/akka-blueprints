# Data model — multi-step-query-engine

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SubQuestion` | `subQuestionId` | `String` | no | Short stable slug, e.g. `sq-compute`. Unique within a `DecomposedQuestion`. |
| | `text` | `String` | no | Focused reformulation of one facet of the original question. |
| | `priority` | `int` | no | 1 = most important. Used by `rankSubQuestions` to order retrieval. |
| `DecomposedQuestion` | `subQuestions` | `List<SubQuestion>` | no | Ordered by priority. Possibly 1-4 entries. |
| | `decomposedAt` | `Instant` | no | When the DECOMPOSE task returned. |
| `Passage` | `passageId` | `String` | no | Short stable id (`p-<8 hex>`). Unique within an `EvidenceSet`. |
| | `subQuestionId` | `String` | no | MUST equal a `SubQuestion.subQuestionId` from the upstream `DecomposedQuestion`. |
| | `source` | `String` | no | Short name of the originating source. |
| | `url` | `String` | no | Canonical url of the source entry. |
| | `text` | `String` | no | Passage text, a direct excerpt or paraphrase. |
| | `relevanceScore` | `float` | no | 0.0–1.0. Computed by `searchPassages`. |
| `EvidenceSet` | `passages` | `List<Passage>` | no | Possibly empty; J4 demonstrates the empty path. |
| | `retrievedAt` | `Instant` | no | When the RETRIEVE task returned. |
| `AnswerSection` | `subQuestionId` | `String` | no | MUST equal a `SubQuestion.subQuestionId` from the upstream `DecomposedQuestion`. |
| | `heading` | `String` | no | Section heading derived from the sub-question text. |
| | `body` | `String` | no | 2–4 sentences synthesising the passage evidence. |
| | `citedPassageIds` | `List<String>` | no | Each entry MUST equal a `Passage.passageId` from the upstream `EvidenceSet`. Non-empty for HIGH/MEDIUM confidence. |
| `QueryAnswer` | `question` | `String` | no | The original user question, echoed for reader context. |
| | `summary` | `String` | no | 1-paragraph summary of the answer. |
| | `confidence` | `ConfidenceLevel` | no | HIGH / MEDIUM / LOW / INSUFFICIENT based on evidence density. |
| | `sections` | `List<AnswerSection>` | no | `sections.size() == subQuestions.size()` (E1 rule 4). Empty when `confidence = INSUFFICIENT`. |
| | `synthesizedAt` | `Instant` | no | When the SYNTHESIZE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `StoppingEvaluator` finished. |
| `QueryRecord` (entity state) | `queryId` | `String` | no | — |
| | `question` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `decomposition` | `Optional<DecomposedQuestion>` | yes | Populated after `QuestionDecomposed`. |
| | `evidence` | `Optional<EvidenceSet>` | yes | Populated after `EvidenceRetrieved`. |
| | `answer` | `Optional<QueryAnswer>` | yes | Populated after `AnswerSynthesized`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `AnswerEvaluated`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `QueryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `CREATED`, `DECOMPOSING`, `DECOMPOSED`, `RETRIEVING`, `RETRIEVED`, `SYNTHESIZING`, `SYNTHESIZED`, `EVALUATED`, `FAILED`.

`ConfidenceLevel`: `HIGH`, `MEDIUM`, `LOW`, `INSUFFICIENT`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `question: String` | → CREATED |
| `DecomposeStarted` | — | → DECOMPOSING |
| `QuestionDecomposed` | `decomposition: DecomposedQuestion` | → DECOMPOSED |
| `RetrieveStarted` | — | → RETRIEVING |
| `EvidenceRetrieved` | `evidence: EvidenceSet` | → RETRIEVED |
| `SynthesizeStarted` | — | → SYNTHESIZING |
| `AnswerSynthesized` | `answer: QueryAnswer` | → SYNTHESIZED |
| `AnswerEvaluated` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryRecord` exactly. The UI fetches the full row via `GET /api/queries/{id}` and streams updates via `GET /api/queries/sse`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<DecomposedQuestion> DECOMPOSE_QUESTION = Task
      .name("Decompose question")
      .description("Break the research question into 2-4 focused sub-questions")
      .resultConformsTo(DecomposedQuestion.class);

  public static final Task<EvidenceSet> RETRIEVE_EVIDENCE = Task
      .name("Retrieve evidence")
      .description("Find relevant passages for each sub-question using searchPassages and fetchPassage")
      .resultConformsTo(EvidenceSet.class);

  public static final Task<QueryAnswer> SYNTHESIZE_ANSWER = Task
      .name("Synthesize answer")
      .description("Compose a QueryAnswer whose sections mirror the sub-questions one-to-one")
      .resultConformsTo(QueryAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## StoppingEvaluator checks

Each check contributes one point on a base of 1:

| Check | Rule | Failure rationale prefix |
|---|---|---|
| Sub-question coverage | Every `subQuestion.subQuestionId` has ≥ 1 passage in `evidence.passages` with matching `subQuestionId`. | `"Sub-question coverage failed: no passage for sub-question 'X'."` |
| Citation provenance | Every `passageId` in any `section.citedPassageIds` appears in `evidence.passages[].passageId`. | `"Citation provenance failed: section 'X' cites passageId 'Y' which does not appear in the retrieved EvidenceSet."` |
| Confidence calibration | HIGH requires ≥ 2 passages per sub-question; MEDIUM requires ≥ 1; LOW/INSUFFICIENT are self-consistent with any count. | `"Confidence calibration failed: confidence HIGH requires ≥ 2 passages per sub-question, saw N."` |
| Section parity | `answer.sections.size() == decomposition.subQuestions.size()`. | `"Section parity failed: N sections for M sub-questions."` |
