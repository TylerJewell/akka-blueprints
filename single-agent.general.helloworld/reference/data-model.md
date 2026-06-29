# Data model

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6); Akka serializes `Optional<T>` as the raw value or `null`.

## `Question` (entity state and View row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Question id (UUID). |
| `text` | `String` | no | The user's question, set at creation. |
| `status` | `QuestionStatus` | no | Lifecycle status. |
| `askedAt` | `Optional<Instant>` | yes | When the question was asked. |
| `answerText` | `Optional<String>` | yes | The recorded answer; absent until answered. |
| `confidence` | `Optional<Double>` | yes | Confidence in `[0,1]`; absent until answered. |
| `answeredAt` | `Optional<Instant>` | yes | When the answer was recorded. |
| `blockedReason` | `Optional<String>` | yes | Why the answer was blocked. |
| `blockedAt` | `Optional<Instant>` | yes | When the answer was blocked. |

`Question.initial(id)` returns a state with `text=""`, status pending, and every `Optional` empty. `applyEvent(QuestionEvent)` switches per variant. `emptyState()` calls `Question.initial("")` and never references `commandContext()` (Lesson 3).

## `QuestionStatus` (enum)

`ASKED · ANSWERED · BLOCKED`

## `Answer` (agent output)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `text` | `String` | no | The answer text. |
| `confidence` | `double` | no | Self-reported confidence in `[0,1]`. |

## `QuestionEvent` (sealed interface, 3 variants)

| Event | Trigger | Fields |
|---|---|---|
| `QuestionAsked` | `ask(text)` command | `questionId, text, timestamp` |
| `AnswerRecorded` | `recordAnswer(text, confidence)` after the check passes | `questionId, text, confidence, timestamp` |
| `AnswerBlocked` | `blockAnswer(reason)` after the before-agent-response check fails | `questionId, reason, timestamp` |

```java
sealed interface QuestionEvent permits QuestionAsked, AnswerRecorded, AnswerBlocked {
  Instant timestamp();
}
record QuestionAsked(String questionId, String text, Instant timestamp) implements QuestionEvent {}
record AnswerRecorded(String questionId, String text, double confidence, Instant timestamp) implements QuestionEvent {}
record AnswerBlocked(String questionId, String reason, Instant timestamp) implements QuestionEvent {}
```

## View row

`QuestionsView` uses `Question` directly as its row type. One query, `getAllQuestions` → `SELECT * AS questions FROM questions_view`. No `WHERE` on `status` (Lesson 2); status filtering is client-side in `AskEndpoint`. A `streamAllQuestions` query backs the SSE endpoint.
