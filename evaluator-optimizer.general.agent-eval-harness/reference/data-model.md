# Data model — agent-eval-harness

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TestCase` | `caseId` | `String` | no | Unique identifier for the case within the suite. |
| | `input` | `String` | no | The prompt or question sent to the target agent. |
| | `expectedAnswer` | `String` | no | The reference answer the judge scores against. |
| | `rubricHint` | `String` | no | Guidance for the judge on what to check. Empty string if not specified. |
| `CaseResponse` | `caseId` | `String` | no | Matches the originating `TestCase.caseId`. |
| | `rawOutput` | `String` | no | The exact text returned by the target agent. |
| | `respondedAt` | `Instant` | no | When the `EvaluatorAgent` returned the response. |
| `JudgingNotes` | `bullets` | `List<String>` | no | 0–2 specific deficiency bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Judgment` | `verdict` | `JudgeVerdict` | no | `PASS` or `FAIL`. |
| | `notes` | `JudgingNotes` | no | See above. |
| | `confidenceScore` | `double` | no | 0.0–1.0 confidence in the verdict. |
| | `judgedAt` | `Instant` | no | When the `JudgeAgent` returned. |
| `CaseResult` | `caseId` | `String` | no | Matches `TestCase.caseId`. |
| | `response` | `CaseResponse` | no | The raw output from the evaluator step. |
| | `judgment` | `Judgment` | no | The scoring from the judge step. |
| `EvalRun` (entity state) | `runId` | `String` | no | Unique run id. |
| | `suiteName` | `String` | no | Name of the suite being evaluated. |
| | `totalCases` | `int` | no | Number of cases in the suite at run start. |
| | `passingThreshold` | `double` | no | Accuracy fraction required to pass the CI gate. |
| | `status` | `RunStatus` | no | See enum. |
| | `results` | `List<CaseResult>` | no | Accumulated case results; bounded at `maxCasesPerRun` (default 20). |
| | `passRate` | `Optional<Double>` | yes | Populated on `RunPassed` or `RunFailed`. |
| | `passCount` | `Optional<Integer>` | yes | Populated on terminal transition. |
| | `failCount` | `Optional<Integer>` | yes | Populated on terminal transition. |
| | `failureReason` | `Optional<String>` | yes | Populated on `RunFailed`. |
| | `startedAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |
| `Suite` | `suiteName` | `String` | no | Unique suite name. |
| | `cases` | `List<TestCase>` | no | Test cases; non-empty. |
| | `rubric` | `String` | no | Default rubric string for the judge. |
| | `defaultPassingThreshold` | `double` | no | Default accuracy threshold for runs on this suite. |

## Enums

`RunStatus`: `PENDING`, `RUNNING`, `PASSED`, `FAILED`.

`JudgeVerdict`: `PASS`, `FAIL`.

## Events (`EvalRunEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RunCreated` | `runId, suiteName, totalCases, passingThreshold, startedAt` | Workflow `startStep` | → `PENDING` |
| `RunStarted` | `runId, startedAt` | Workflow begins iterating cases | → `RUNNING` |
| `CaseEvaluated` | `caseId, response: CaseResponse, judgment: Judgment` | After `judgeStep` returns for each case | (no status change; appends to `results[]`) |
| `GateFailed` | `passRate, threshold, failCount` | `aggregateStep` finds `passRate < passingThreshold` | (no status change; precedes `RunFailed`) |
| `RunPassed` | `passRate, passCount, finishedAt` | `passStep` after gate check succeeds | → `PASSED`, `finishedAt = now` |
| `RunFailed` | `passRate, failCount, failureReason, finishedAt` | `failStep` after gate fires or `defaultStepRecovery` failover | → `FAILED`, `finishedAt = now` |
| `AccuracySnapshot` | `runId, passRate, passCount, failCount, suiteName, recordedAt` | `AccuracySampler` per completed run; workflow on terminal transition | (no status change; final accuracy record) |

## Events (`SuiteRegistry`)

| Event | Payload |
|---|---|
| `SuiteRegistered` | `suiteName, cases, rubric, defaultPassingThreshold, registeredAt` |
| `RunTriggered` | `runId, suiteName, passingThreshold, triggeredAt` |

## View row

`EvalRunRow` is structurally identical to `EvalRun` — the `results` list is bounded at `maxCasesPerRun` (default 20) so the row stays small enough to push down the SSE stream. The view's `TableUpdater` consumes every `EvalRunEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRuns` returning the full list. Callers filter by `status` client-side.
