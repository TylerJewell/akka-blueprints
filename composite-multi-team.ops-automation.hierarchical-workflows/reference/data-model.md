# Data model — hierarchical-workflow-automation

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `OpsRequest` | `requestId` | `String` | no | Id assigned at submission. |
| | `description` | `String` | no | The operations request description. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `WorkflowPlan` | `objective` | `String` | no | One-sentence goal from the orchestrator. |
| | `targetSystems` | `List<String>` | no | Systems the workflow will act on. |
| | `taskDescriptions` | `List<String>` | no | Task descriptions the execution team carries out. |
| `DiscoveryPlan` | `scanTargets` | `List<String>` | no | System targets, one per scanner instance. |
| `ScanResult` | `target` | `String` | no | The system target this result covers. |
| | `findings` | `String` | no | One short paragraph of findings. |
| | `references` | `List<String>` | no | One to three identifiers or references. |
| `DiscoverySummary` | `overview` | `String` | no | One-paragraph synthesis. |
| | `keyFindings` | `List<String>` | no | Three to five facts the execution team can act on. |
| `TaskSpec` | `title` | `String` | no | Short task heading. |
| | `instruction` | `String` | no | One to two sentences on what the executor must do. |
| `ExecutionPlan` | `approach` | `String` | no | One sentence on task ordering rationale. |
| | `tasks` | `List<TaskSpec>` | no | Three to five task specs. |
| `TaskOutcome` | `taskId` | `String` | no | The task this outcome fills. |
| | `title` | `String` | no | Task heading. |
| | `result` | `String` | no | Short paragraph documenting the action and outcome (non-empty). |
| | `stepsCompleted` | `int` | no | Number of discrete steps taken. |
| `ExecutionSummary` | `headline` | `String` | no | Working headline for the assembled execution summary. |
| | `outcomes` | `List<TaskOutcome>` | no | The completed outcomes, in order. |
| | `tasksCompleted` | `int` | no | Total tasks completed. |
| `ValidationNote` | `axis` | `String` | no | The axis validated (`correctness`/`safety`/`compliance`). |
| | `outcome` | `ValidationOutcome` | no | `PASS` or `RETRY`. |
| | `comments` | `String` | no | For a `RETRY`, names the task needing re-execution. |
| `ValidationVerdict` | `outcome` | `ValidationOutcome` | no | Aggregated `PASS` only if every note passes. |
| | `notes` | `List<ValidationNote>` | no | The panel's notes. |
| | `mustRetry` | `List<String>` | no | Task titles to redo (empty on `PASS`). |
| `OpsReport` | `title` | `String` | no | Delivered report title. |
| | `body` | `String` | no | The assembled report body. |
| | `preparedBy` | `String` | no | Attribution line (non-empty; gated by G1). |
| | `assembledAt` | `Instant` | no | When the orchestrator assembled it. |
| `AuditEval` | `stage` | `String` | no | `discovery` / `execution` / `validation`. |
| | `score` | `int` | no | Quality score 0-100 from `ResultEvaluator`. |
| | `flags` | `List<String>` | no | Issues found (may be empty). |
| | `evaluatedAt` | `Instant` | no | When the eval ran. |
| `ComplianceReview` | `reviewedBy` | `String` | no | Operator/auditor id. |
| | `outcome` | `ComplianceOutcome` | no | `CLEARED` or `FLAGGED`. |
| | `comments` | `String` | no | Post-execution review notes. |
| | `reviewedAt` | `Instant` | no | When recorded. |

## Entity state — `WorkflowState` (`WorkflowEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `requestId` | `String` | no | Unique id; also the `WorkflowInstance` id. |
| `description` | `String` | no | Submitted operations request. |
| `requestedBy` | `String` | no | Submitter id. |
| `status` | `WorkflowStatus` | no | See enum. |
| `plan` | `Optional<WorkflowPlan>` | yes | Populated on `WorkflowPlanned`. |
| `discoverySummary` | `Optional<DiscoverySummary>` | yes | Populated on `DiscoverySynthesized`. |
| `taskIds` | `List<String>` | no | Task ids created from the plan (may be empty early). |
| `executionSummary` | `Optional<ExecutionSummary>` | yes | Populated on `ExecutionSummarized`. |
| `validationVerdict` | `Optional<ValidationVerdict>` | yes | Populated on `ValidationCompleted`. |
| `report` | `Optional<OpsReport>` | yes | Populated on `ReportCompleted`. |
| `reportUrl` | `Optional<String>` | yes | Generated URL on completion. |
| `completedAt` | `Optional<Instant>` | yes | When completed. |
| `auditEvals` | `List<AuditEval>` | no | Eval results recorded by `AuditEvalConsumer` (may be empty). |
| `complianceReview` | `Optional<ComplianceReview>` | yes | Populated on `ComplianceReviewRecorded`. |
| `retryCount` | `int` | no | Number of retry rounds taken (0 or 1). |
| `createdAt` | `Instant` | no | When `WorkflowCreated` emitted. |

## Entity state — `Task` (`TaskEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | Deterministic id `requestId + "-t" + index`. |
| `requestId` | `String` | no | Owning request. |
| `title` | `String` | no | From the `TaskSpec`. |
| `instruction` | `String` | no | From the `TaskSpec`. |
| `status` | `TaskStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Executor that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `result` | `Optional<String>` | yes | Documented task result. |
| `stepsCompleted` | `Optional<Integer>` | yes | Steps taken to complete the task. |
| `completedAt` | `Optional<Instant>` | yes | When marked done. |
| `createdAt` | `Instant` | no | When `TaskCreated` emitted. |

## Enums

`WorkflowStatus`: `SUBMITTED`, `PLANNED`, `DISCOVERING`, `DISCOVERED`, `EXECUTING`, `EXECUTED`, `VALIDATING`, `VALIDATED`, `COMPLETED`.
`TaskStatus`: `OPEN`, `CLAIMED`, `DONE`.
`ValidationOutcome`: `PASS`, `RETRY`.
`ComplianceOutcome`: `CLEARED`, `FLAGGED`.

## Events — `WorkflowEntity`

| Event | Payload | Transition |
|---|---|---|
| `WorkflowCreated` | `requestId, description, requestedBy, createdAt` | → SUBMITTED |
| `WorkflowPlanned` | `plan` | SUBMITTED → PLANNED |
| `DiscoverySynthesized` | `discoverySummary` | DISCOVERING → DISCOVERED (also marks PLANNED → DISCOVERING when discovery begins) |
| `TasksPlanned` | `taskIds` | DISCOVERED → EXECUTING |
| `ExecutionSummarized` | `executionSummary` | EXECUTING → EXECUTED |
| `ValidationCompleted` | `verdict` | EXECUTED/VALIDATING → VALIDATED (on PASS) |
| `RetryRequested` | `mustRetry, retryCount` | VALIDATING → EXECUTING (one bounded round) |
| `ReportCompleted` | `report, reportUrl, completedAt` | VALIDATED → COMPLETED |
| `AuditEvaluated` | `auditEval` | (no status change; appends to `auditEvals`) |
| `ComplianceReviewRecorded` | `complianceReview` | (no status change; COMPLETED only) |

A report blocked by the G1 guardrail records its reason via a `recordReportBlock` command (no status change) and leaves the workflow `VALIDATED`.

## Events — `TaskEntity`

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, requestId, title, instruction, createdAt` | → OPEN |
| `TaskClaimed` | `executorId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `TaskCompleted` | `result, stepsCompleted, completedAt` | CLAIMED → DONE |
| `TaskReleased` | `releasedAt` | CLAIMED → OPEN, clears `claimedBy` |

## Events — `RequestQueue`

| Event | Payload |
|---|---|
| `RequestSubmitted` | `requestId, description, requestedBy, submittedAt` |

## View rows

`WorkflowRow` (row of `WorkflowBoardView`) mirrors `WorkflowState` but drops the heavy `OpsReport.body` and the `ScanResult` contents — it keeps the plan objective, discovery overview, `taskIds`, the execution headline and `tasksCompleted`, the verdict outcome, the `auditEvals`, the `reportUrl`, and the `complianceReview`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`TaskRow` (row of `TaskBoardView`) mirrors `Task` but drops the heavy `result` — it keeps `stepsCompleted` and a `resultPresent` boolean so the executors' poll and the board stay small. Every nullable lifecycle field is `Optional<T>`.

Both views expose one unfiltered query (`getAllWorkflows` / `getAllTasks`) plus a streaming query for the SSE endpoints. Status filtering happens client-side because the views cannot auto-index the enum status column (Lesson 2).
