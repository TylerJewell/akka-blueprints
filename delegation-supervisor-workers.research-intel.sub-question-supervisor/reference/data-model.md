# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## QuerySession (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `sessionId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The submitted question |
| `status` | `SessionStatus` | no | Lifecycle state |
| `decomposition` | `Optional<DecompositionPlan>` | yes | Supervisor's sub-question plan; null until `DecompositionReady` |
| `indexResults` | `Optional<List<IndexResult>>` | yes | Accumulated worker results; null until first `IndexResultAttached` |
| `combinedAnswer` | `Optional<CombinedAnswer>` | yes | Merged answer; null until `SessionSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `SessionBlocked` or `SessionPartial` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `SynthesisEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Session creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `QuerySessionView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QuestionRequest(String question, String requestedBy) {}

record SubQuestion(String text, String indexId) {}
record DecompositionPlan(List<SubQuestion> subQuestions, Instant decomposedAt) {}

record SourceRef(String title, String uri) {}
record IndexResult(String subQuestion, String indexId, String answer,
                   List<SourceRef> sourceRefs, Instant retrievedAt) {}

record CombinedAnswer(String summary, List<IndexResult> indexResults, Instant synthesisedAt) {}
```

## Status enum

```java
enum SessionStatus { DECOMPOSING, RETRIEVING, SYNTHESISED, PARTIAL, BLOCKED }
```

## Events

### QuerySessionEntity

| Event | Trigger |
|---|---|
| `SessionCreated` | Workflow creates the session (`createSession`) |
| `DecompositionReady` | Supervisor returns a `DecompositionPlan` |
| `IndexResultAttached` | One `IndexWorker` returns an `IndexResult` |
| `IndexCallRejected` | Before-tool-call guardrail blocked a sub-question |
| `SessionSynthesised` | Supervisor synthesis completes from full results |
| `SessionPartial` | One or more workers timed out; synthesised from partial results |
| `SessionBlocked` | All sub-questions were rejected by the guardrail |
| `SynthesisEvalScored` | `EvalSampler` recorded a 1–5 score |

### IndexCallQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, requestedBy)` from endpoint or simulator |

Fields: `{ sessionId, question, requestedBy, submittedAt }`.
