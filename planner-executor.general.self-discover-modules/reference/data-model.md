# Data model — self-discover-modules

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskRequest` | `prompt` | `String` | no | User-submitted task prompt. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ModuleStep` | `moduleName` | `String` | no | Name of the reasoning module from the library. |
| | `adaptedDescription` | `String` | no | Task-specific description written by the Structurer. |
| | `role` | `ModuleRole` | no | Structural role this step plays in the reasoning sequence. |
| | `stepIndex` | `int` | no | 0-based position in the ordered structure. |
| `ReasoningStructure` | `selectedModules` | `List<String>` | no | Module names chosen during SELECT_MODULES. |
| | `adaptedSteps` | `List<ModuleStep>` | no | Ordered adapted steps (2–8). |
| | `compositionRationale` | `String` | no | One sentence explaining the ordering. |
| | `revisionNumber` | `Optional<Integer>` | yes | Null on first composition; incremented on each revision. |
| `EvalResult` | `passed` | `boolean` | no | True when score ≥ 0.7. |
| | `score` | `double` | no | 0.0–1.0 from `StructureEvaluator`. |
| | `rejectionReason` | `Optional<String>` | yes | Comma-separated failing rules; null when passed. |
| `StepExecution` | `stepIndex` | `int` | no | Matches `ModuleStep.stepIndex`. |
| | `moduleName` | `String` | no | Module that was executed. |
| | `observation` | `String` | no | 3–6 sentence finding from the Solver. |
| | `ok` | `boolean` | no | True if the step produced a usable observation. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| | `executedAt` | `Instant` | no | When the step finished. |
| `TaskAnswer` | `summary` | `String` | no | 60–120 word integrated summary. |
| | `observations` | `List<String>` | no | 3–5 cited bullets, each prefixed by module name. |
| | `producedAt` | `Instant` | no | When the Solver produced the answer. |
| `Task` (entity state) | `taskId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `TaskStatus` | no | See enum. |
| | `structure` | `Optional<ReasoningStructure>` | yes | Populated after `StructureComposed`. |
| | `lastEval` | `Optional<EvalResult>` | yes | Populated after each eval run. |
| | `stepExecutions` | `List<StepExecution>` | no | Append-only; empty until SOLVING. |
| | `answer` | `Optional<TaskAnswer>` | yes | Populated after `TaskCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `TaskFailed` / `TaskFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `TaskHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `TaskCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `ModuleRole` → `DECOMPOSE`, `ANALYZE`, `VERIFY`, `SYNTHESIZE`, `REFLECT`.
- `TaskStatus` → `STRUCTURING`, `SOLVING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`TaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, prompt, createdAt` | → STRUCTURING |
| `StructureComposed` | `structure: ReasoningStructure` | no status change; sets `structure`. |
| `StructureRejected` | `rejectionReason, score, revisionNumber` | no status change; updates `lastEval`. |
| `StructureRevised` | `structure: ReasoningStructure` | no status change; replaces `structure`, updates `lastEval`. |
| `StructureApproved` | `score` | → SOLVING; sets `lastEval.passed = true`. |
| `StepExecuted` | `execution: StepExecution` | no status change; appends to `stepExecutions`. |
| `TaskCompleted` | `answer: TaskAnswer` | → COMPLETED, `finishedAt = now`. |
| `TaskFailed` | `failureReason` | → FAILED, `finishedAt = now`. |
| `TaskHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now`. |
| `TaskFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now`. |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, prompt, requestedBy, submittedAt` |

## View row

`TaskRow` mirrors `Task` minus the heavy step payload — `stepExecutions` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `observation` is capped at 240 characters. The UI fetches the full task by id on click via `GET /api/tasks/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
