# Data model — react-math-wiki

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Question` | `questionId` | `String` | no | UUID minted by `ResearchEndpoint`. |
| | `text` | `String` | no | User's question, unmodified. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCall` | `step` | `int` | no | 1-indexed position in the ReAct loop. |
| | `tool` | `ToolName` | no | Enum value. |
| | `argument` | `String` | no | Expression passed to `evaluate_math`, or query passed to `search_wikipedia`. |
| | `observation` | `String` | no | What the tool returned (numeric string or passage text). |
| `ResearchAnswer` | `answerText` | `String` | no | Direct answer to the question. |
| | `confidence` | `Confidence` | no | Enum value. |
| | `trace` | `List<ToolCall>` | no | All tool calls made, in order. May be empty if the agent answered from training data (eval will flag this). |
| | `stepCount` | `int` | no | `trace.size()`. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `AnswerEvaluator` finished. |
| `ResearchQuestion` (entity state) | `questionId` | `String` | no | — |
| | `question` | `Optional<Question>` | yes | Populated after `QuestionSubmitted`. |
| | `answer` | `Optional<ResearchAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `QuestionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuestionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ResearchQuestion` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToolName`: `EVALUATE_MATH`, `SEARCH_WIKIPEDIA`.
`Confidence`: `HIGH`, `MEDIUM`, `LOW`.
`QuestionStatus`: `SUBMITTED`, `RUNNING`, `ANSWERED`, `EVALUATED`, `FAILED`.

## Events (`ResearchEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuestionSubmitted` | `question` | → SUBMITTED |
| `RunStarted` | — | → RUNNING |
| `ToolCallRecorded` | `step`, `tool`, `argument`, `observation` | stays RUNNING (self-loop) |
| `AnswerRecorded` | `answer` | → ANSWERED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `QuestionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ResearchQuestion.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QuestionRow` mirrors `ResearchQuestion` with the trace truncated to 10 entries for the UI. The full trace is accessible via `GET /api/questions/{id}` which returns the raw entity state.

The view declares ONE query: `getAllQuestions: SELECT * AS questions FROM research_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ResearchTasks.java`)

```java
public final class ResearchTasks {
  public static final Task<ResearchAnswer> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Run a ReAct loop with evaluate_math and search_wikipedia to answer the question")
      .resultConformsTo(ResearchAnswer.class);

  private ResearchTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
