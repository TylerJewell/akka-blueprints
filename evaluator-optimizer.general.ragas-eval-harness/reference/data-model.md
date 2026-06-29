# Data model — ragas-eval-harness

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Question` | `text` | `String` | no | The user-submitted question. |
| | `corpusTag` | `Optional<String>` | yes | Optional filter restricting retrieval to a corpus subset. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ContextChunk` | `chunkId` | `String` | no | Unique identifier of the corpus chunk. |
| | `sourceDoc` | `String` | no | Document the chunk was extracted from. |
| | `text` | `String` | no | The chunk text used in answer composition. |
| | `relevanceScore` | `double` | no | Retrieval relevance score in `[0.0, 1.0]`. |
| `AnswerRecord` | `text` | `String` | no | The composed answer. |
| | `retrievedChunks` | `List<ContextChunk>` | no | Chunks used; may be empty if no corpus match. |
| | `groundingConfidence` | `double` | no | Self-reported grounding confidence in `[0.0, 1.0]`. |
| | `answeredAt` | `Instant` | no | When the Retrieval agent returned the answer. |
| `GroundingVerdict` | `passed` | `boolean` | no | Whether the grounding check accepted the answer. |
| | `reasonCode` | `String` | no | `OK` or `BELOW_CONFIDENCE_FLOOR`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `RagasFeedback` | `failedMetrics` | `List<String>` | no | Names of metrics below threshold; empty on `PASS`. |
| | `metricScores` | `Map<String, Double>` | no | Always populated for all three metrics. |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `RagasScore` | `verdict` | `EvalVerdict` | no | `PASS` or `RETRY`. |
| | `feedback` | `RagasFeedback` | no | See above. |
| | `faithfulness` | `double` | no | `[0.0, 1.0]`; threshold ≥ 0.80. |
| | `answerRelevance` | `double` | no | `[0.0, 1.0]`; threshold ≥ 0.75. |
| | `contextPrecision` | `double` | no | `[0.0, 1.0]`; threshold ≥ 0.70. |
| | `evaluatedAt` | `Instant` | no | When the RAGAS Evaluator returned. |
| `EvalAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes grounding-blocked attempts). |
| | `answer` | `AnswerRecord` | no | The Retrieval agent's output for this attempt. |
| | `grounding` | `GroundingVerdict` | no | The deterministic check's verdict. |
| | `score` | `Optional<RagasScore>` | yes | Empty when the grounding check blocked; populated otherwise. |
| `EvalRun` (entity state) | `runId` | `String` | no | Unique id. |
| | `questionText` | `String` | no | Original question. |
| | `corpusTag` | `Optional<String>` | yes | Corpus filter; null when not provided. |
| | `maxAttempts` | `int` | no | Per-run retry ceiling (default 3). |
| | `status` | `EvalRunStatus` | no | See enum. |
| | `attempts` | `List<EvalAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `passedAttemptNumber` | `Optional<Integer>` | yes | Populated on `RunPassed`. |
| | `passedAnswerText` | `Optional<String>` | yes | Populated on `RunPassed`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `RunFailedFinal`. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |

## Enums

`EvalRunStatus`: `ANSWERING`, `EVALUATING`, `PASSED`, `FAILED_FINAL`.

`EvalVerdict`: `PASS`, `RETRY`.

## Events (`EvalRunEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RunCreated` | `questionText, corpusTag, maxAttempts, createdAt` | Workflow `startStep` | → `ANSWERING` |
| `AttemptAnswered` | `attemptNumber, answer: AnswerRecord` | After `answerStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGroundingVerdictRecorded` | `attemptNumber, verdict: GroundingVerdict` | After `groundingStep` | `passed=true` → `EVALUATING`; `passed=false` → `ANSWERING` (re-answer) |
| `AttemptScored` | `attemptNumber, score: RagasScore` | After `scoreStep` returns | (no status change; populates `attempts[n].score`) |
| `RunPassed` | `attemptNumber, passedAnswerText` | `RagasScore.verdict = PASS` | → `PASSED`, `finishedAt = now` |
| `RunFailedFinal` | `bestAttemptNumber, bestAnswerText, failureReason` | `attempts.size() == maxAttempts` AND last score is `RETRY`; OR `defaultStepRecovery` failover | → `FAILED_FINAL`, `finishedAt = now` |
| `ScoreRecorded` | `attemptNumber, verdict, faithfulness, answerRelevance, contextPrecision, ceilingExceeded, recordedAt` | `ScoreSampler` per scored attempt; workflow on terminal transition | (no status change; appends to an internal `scoreEvents[]` view-side projection) |

## Events (`QuestionQueue`)

| Event | Payload |
|---|---|
| `QuestionSubmitted` | `runId, text, corpusTag, submittedBy, submittedAt` |

## View row

`EvalRunRow` is structurally identical to `EvalRun` — the `attempts` list is bounded at `maxAttempts` (default 3) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `EvalRunEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRuns` returning the full list. Callers filter by `status` client-side. The CI gate endpoint uses the same query and computes the rolling pass rate in the endpoint handler.
