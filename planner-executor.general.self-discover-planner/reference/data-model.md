# Data model — self-discover-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SolveRequest` | `prompt` | `String` | no | User-submitted task prompt. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ReasoningModule` | `moduleId` | `String` | no | Unique identifier (e.g., `m-decompose-01`). |
| | `kind` | `ModuleKind` | no | One of the six module kinds. |
| | `name` | `String` | no | Short human-readable name. |
| | `description` | `String` | no | One-sentence description of what the module does. |
| `ReasoningPlan` | `selectedModuleIds` | `List<String>` | no | IDs of the chosen modules (3–7). |
| | `stepOrder` | `List<PlanStep>` | no | Ordered steps; each references a selected module. |
| | `rationale` | `String` | no | 2–3 sentences explaining the module selection and ordering. |
| | `revisionNote` | `Optional<String>` | yes | Populated on REVISE_PLAN; absent on the initial plan. |
| `PlanStep` | `index` | `int` | no | 0-based position in `stepOrder`. |
| | `moduleId` | `String` | no | The module this step runs. |
| | `objective` | `String` | no | One sentence naming what this step produces. |
| | `inputsFrom` | `List<Integer>` | no | Indices of prior steps whose outputs feed this step; empty for step 0. |
| `PlanEval` | `score` | `double` | no | Quality score in [0.0, 1.0]. |
| | `verdict` | `PlanVerdict` | no | PASS (score ≥ 0.65) or FAIL. |
| | `feedback` | `String` | no | Concrete evaluator feedback. |
| `ModuleResult` | `moduleId` | `String` | no | Module that ran. |
| | `kind` | `ModuleKind` | no | Kind of the module. |
| | `output` | `String` | no | Scrubbed textual output of the step. |
| | `ok` | `boolean` | no | True if the module completed successfully. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ExecutionStep` | `stepIndex` | `int` | no | Matches `PlanStep.index`. |
| | `moduleId` | `String` | no | Module that ran this step. |
| | `kind` | `ModuleKind` | no | Kind of the module. |
| | `objective` | `String` | no | Echo of `PlanStep.objective`. |
| | `result` | `ModuleResult` | no | The (scrubbed) result. |
| | `executedAt` | `Instant` | no | When the step was recorded. |
| `ExecutionLog` | `steps` | `List<ExecutionStep>` | no | Append-only; one entry per executed step. |
| `TaskAnswer` | `summary` | `String` | no | 60–120 word answer to the original prompt. |
| | `evidence` | `List<String>` | no | 3–5 bullets citing module kinds and step indices. |
| | `producedAt` | `Instant` | no | When the synthesiser produced the answer. |
| `Solve` (entity state) | `solveId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `SolveStatus` | no | See enum. |
| | `plan` | `Optional<ReasoningPlan>` | yes | Populated after `SolvePlanned`. Updated on revision. |
| | `planEval` | `Optional<PlanEval>` | yes | Populated after `PlanAccepted` or last `PlanRejected`. |
| | `executionLog` | `Optional<ExecutionLog>` | yes | Populated after first `ModuleExecuted`. |
| | `answer` | `Optional<TaskAnswer>` | yes | Populated after `SolveCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `SolveFailed` / `SolveFailedTimeout`. |
| | `planRevisionCount` | `int` | no | Number of plan revisions before acceptance (0, 1, or 2). |
| | `createdAt` | `Instant` | no | When `SolveCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the solve reached a terminal state. |
| `ModuleRegistryState` | `modules` | `List<ReasoningModule>` | no | The full module catalogue. |

## Enums

- `ModuleKind` → `DECOMPOSE`, `ANALYSE`, `COMPARE`, `GENERATE`, `VERIFY`, `REFLECT`.
- `PlanVerdict` → `PASS`, `FAIL`.
- `SolveStatus` → `PLANNING`, `EVALUATING`, `EXECUTING`, `COMPLETED`, `FAILED`, `STUCK`.

## Events (`SolveEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SolveCreated` | `solveId, prompt, createdAt` | → PLANNING |
| `SolvePlanned` | `plan: ReasoningPlan, revisionCount: int` | → EVALUATING; replaces `plan` field |
| `PlanAccepted` | `planEval: PlanEval` | → EXECUTING; sets `planEval` |
| `PlanRejected` | `planEval: PlanEval, revisionCount: int` | no status change; increments `planRevisionCount`; sets `planEval` |
| `ModuleExecuted` | `step: ExecutionStep` | no status change; appends to `executionLog.steps` |
| `SolveCompleted` | `answer: TaskAnswer` | → COMPLETED, `finishedAt = now` |
| `SolveFailed` | `failureReason: String` | → FAILED, `finishedAt = now` |
| `SolveFailedTimeout` | `failureReason: String` | → STUCK, `finishedAt = now` |

## Events (`ModuleRegistry`)

| Event | Payload |
|---|---|
| `RegistrySeeded` | `modules: List<ReasoningModule>, seededAt: Instant` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `SolveSubmitted` | `solveId, prompt, requestedBy, submittedAt` |

## View row

`SolveRow` mirrors `Solve` minus the heavy execution payload — `executionLog.steps` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each `result.output` is capped at 240 characters. The UI fetches the full solve by id on click via `GET /api/solves/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
