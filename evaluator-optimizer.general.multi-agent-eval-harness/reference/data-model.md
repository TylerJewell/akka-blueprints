# Data model — multi-agent-eval-harness

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ScenarioSpec` | `scenarioId` | `String` | no | Unique identifier for the scenario within its suite. |
| | `suiteName` | `String` | no | The suite this scenario belongs to. |
| | `prompt` | `String` | no | The input fed to the specialist agent. |
| | `expectedBehavior` | `String` | no | Human-readable acceptance criterion for this scenario. |
| | `category` | `String` | no | Scenario category (e.g., `reasoning`, `retrieval`, `safety`). |
| `ScenarioResult` | `scenarioId` | `String` | no | Matches the `ScenarioSpec.scenarioId`. |
| | `agentResponse` | `String` | no | Raw text produced by the specialist for this scenario. |
| | `thresholdMet` | `boolean` | no | Whether the response satisfies the per-scenario acceptance criterion. |
| | `completedAt` | `Instant` | no | When the orchestrator recorded this result. |
| `AggregateResult` | `suiteName` | `String` | no | The suite that was evaluated. |
| | `scenarioResults` | `List<ScenarioResult>` | no | One entry per scenario dispatched. |
| | `aggregateScore` | `double` | no | Fraction of `thresholdMet = true` across all scenarios (0.0–1.0). |
| | `aggregatedAt` | `Instant` | no | When the orchestrator returned. |
| `JudgmentNotes` | `bullets` | `List<String>` | no | 0–4 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Judgment` | `verdict` | `JudgeVerdict` | no | `PASS` or `FAIL`. |
| | `notes` | `JudgmentNotes` | no | See above. |
| | `score` | `double` | no | 0.0–1.0 rubric score from the judge. |
| | `evaluatedAt` | `Instant` | no | When the judge returned. |
| `EvalRun` (entity state) | `runId` | `String` | no | Unique id. |
| | `suiteName` | `String` | no | The scenario suite submitted. |
| | `passThreshold` | `double` | no | Deployer-configured acceptance bar (default 0.75). |
| | `maxRetries` | `int` | no | Per-run retry ceiling (default 2). |
| | `status` | `RunStatus` | no | See enum. |
| | `scenarioResults` | `List<ScenarioResult>` | no | Accumulated across all orchestration attempts; starts empty. |
| | `judgments` | `List<Judgment>` | no | One per workflow judgment cycle; starts empty. |
| | `finalScore` | `Optional<Double>` | yes | Populated on `RunPassed` or `RunFailed`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `RunFailed`. |
| | `ciGateBlocked` | `boolean` | no | Set to `true` when `ciGateStep` emits `CIGateBlocked`. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |

## Enums

`RunStatus`: `RUNNING`, `JUDGING`, `PASSED`, `FAILED`.

`JudgeVerdict`: `PASS`, `FAIL`.

## Events (`EvalRunEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RunCreated` | `runId, suiteName, passThreshold, maxRetries, createdAt` | Workflow `startStep` | → `RUNNING` |
| `ScenariosLoaded` | `runId, suiteName, scenarioCount` | After `loadScenariosStep` returns from `ScenarioRegistry` | (no status change; records scenario count) |
| `OrchestratorResultRecorded` | `runId, aggregateResult: AggregateResult` | After `dispatchStep` or `rerunStep` returns | (no status change; replaces or appends `scenarioResults`; updates `aggregateScore`) |
| `JudgmentRecorded` | `runId, judgment: Judgment` | After `judgeStep` returns | → `JUDGING`; appends to `judgments[]` |
| `RunPassed` | `runId, finalScore, acceptedJudgment` | `Judgment.verdict = PASS` | → `PASSED`, `finishedAt = now` |
| `RunFailed` | `runId, failureReason, bestJudgment` | `retriesRemaining = 0` AND last judgment is `FAIL`; OR `defaultStepRecovery` failover | → `FAILED`, `finishedAt = now` |
| `CIGateBlocked` | `runId, finalScore, passThreshold, blockedAt` | `ciGateStep` when `finalScore < passThreshold` | (no status change; sets `ciGateBlocked = true`) |
| `ScenarioEvalRecorded` | `scenarioId, verdict, score, thresholdMet, recordedAt` | `EvalSampler` per judged scenario; workflow on terminal transition | (no status change; appended to internal eval log) |

## Events (`ScenarioRegistry`)

| Event | Payload |
|---|---|
| `SuiteRegistered` | `registryId, suiteName, scenarios: List<ScenarioSpec>, registeredAt` |

## View row

`EvalRunRow` is structurally identical to `EvalRun` — the `scenarioResults` list is bounded by the number of scenarios in the suite (typical suites have 5–20 scenarios) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `EvalRunEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRuns` returning the full list. Callers filter by `status` client-side.
