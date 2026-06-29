# Data model — llm-judge-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Question` | `questionText` | `String` | no | The user-submitted question. |
| | `domainTag` | `String` | no | Subject domain (default `"general"`). |
| | `scoreThreshold` | `int` | no | Minimum judge score required for acceptance. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `GeneratedAnswer` | `text` | `String` | no | The answer itself. |
| | `tokenCount` | `int` | no | Approximate token count of `text`. |
| | `generatedAt` | `Instant` | no | When the Generator returned the answer. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the answer. |
| | `reasonCode` | `String` | no | `OK`, `STRUCTURAL_FAIL`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `JudgeFeedback` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Judgment` | `verdict` | `JudgeVerdict` | no | `ACCEPT` or `REVISE`. |
| | `feedback` | `JudgeFeedback` | no | See above. |
| | `score` | `int` | no | 1–5 rubric (minimum of four dimensions). |
| | `evaluatedAt` | `Instant` | no | When the Judge returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `answer` | `GeneratedAnswer` | no | The Generator's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The structural check's verdict. |
| | `judgment` | `Optional<Judgment>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Evaluation` (entity state) | `evaluationId` | `String` | no | Unique id. |
| | `questionText` | `String` | no | Original question. |
| | `domainTag` | `String` | no | Subject domain. |
| | `scoreThreshold` | `int` | no | Minimum score for acceptance. |
| | `maxAttempts` | `int` | no | Per-evaluation retry ceiling (default 4). |
| | `status` | `EvaluationStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `EvaluationAccepted`. |
| | `acceptedText` | `Optional<String>` | yes | Populated on `EvaluationAccepted`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `EvaluationRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `EvaluationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the evaluation reached a terminal state. |

## Enums

`EvaluationStatus`: `GENERATING`, `JUDGING`, `ACCEPTED`, `REJECTED_FINAL`.

`JudgeVerdict`: `ACCEPT`, `REVISE`.

## Events (`EvaluationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `EvaluationCreated` | `questionText, domainTag, scoreThreshold, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, answer: GeneratedAnswer` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `JUDGING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptJudged` | `attemptNumber, judgment: Judgment` | After `judgeStep` returns | (no status change; populates `attempts[n].judgment`) |
| `EvaluationAccepted` | `attemptNumber, acceptedText` | `Judgment.verdict = ACCEPT` | → `ACCEPTED`, `finishedAt = now` |
| `EvaluationRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last judgment is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `JudgmentRecorded` | `attemptNumber, verdict, score, guardrailFailed, recordedAt` | `JudgeSampler` per judged attempt; workflow on terminal transition | (no status change; appends to an internal `judgmentEvents[]` view-side projection) |

## Events (`QuestionQueue`)

| Event | Payload |
|---|---|
| `QuestionSubmitted` | `evaluationId, questionText, domainTag, scoreThreshold, submittedBy, submittedAt` |

## View row

`EvaluationRow` is structurally identical to `Evaluation` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `EvaluationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllEvaluations` returning the full list. Callers filter by `status` client-side.
