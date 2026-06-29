# Data model — sk-process

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobRequest` | `jobId` | `String` | no | Id assigned at submission. |
| | `template` | `String` | no | The process template name. |
| | `payload` | `String` | no | Optional structured parameters (may be empty string). |
| | `submittedBy` | `String` | no | Operator identifier. |
| `StepDefinition` | `name` | `String` | no | Short step name. |
| | `description` | `String` | no | What the step must do. |
| | `expectedOutput` | `String` | no | What a passing result looks like. |
| `StepPlan` | `objective` | `String` | no | One-sentence job objective from the coordinator. |
| | `steps` | `List<StepDefinition>` | no | Three to five step definitions. |
| `DetailedStepSpec` | `stepId` | `String` | no | Deterministic id `jobId + "-p" + index`. |
| | `name` | `String` | no | Step name from the plan. |
| | `description` | `String` | no | Expanded executor-ready description. |
| | `expectedOutput` | `String` | no | Narrowed acceptance bar. |
| | `sequence` | `int` | no | Display order (1-based). |
| `DetailedPlan` | `approach` | `String` | no | One sentence on how steps fit together. |
| | `steps` | `List<DetailedStepSpec>` | no | Three to five refined step specs. |
| `StepOutput` | `stepId` | `String` | no | The step this output fills. |
| | `name` | `String` | no | Step name. |
| | `result` | `String` | no | One to two paragraphs (non-empty). |
| | `outputTokens` | `int` | no | Approximate token count. |
| `CompletedBatch` | `jobId` | `String` | no | Owning job. |
| | `outputs` | `List<StepOutput>` | no | All completed step outputs. |
| | `totalOutputTokens` | `int` | no | Sum of output tokens. |
| `CheckNote` | `criterion` | `String` | no | The criterion reviewed (`format`/`completeness`/`policy`). |
| | `outcome` | `CheckOutcome` | no | `PASS` or `RETRY`. |
| | `explanation` | `String` | no | For `RETRY`, names the step needing a redo. |
| `QualityVerdict` | `outcome` | `CheckOutcome` | no | Aggregated `PASS` only when every note passes. |
| | `notes` | `List<CheckNote>` | no | The panel's notes. |
| | `mustRetry` | `List<String>` | no | Step names to redo (empty on `PASS`). |
| `JobSummary` | `jobId` | `String` | no | Owning job. |
| | `overview` | `String` | no | Two to three sentence synthesis. |
| | `keyOutputs` | `List<String>` | no | Two to four actionable output strings (non-empty; gated by G1). |
| | `resultRef` | `String` | no | Reference to stored results (non-empty; gated by G1). |
| | `completedAt` | `Instant` | no | When the coordinator assembled the summary. |
| `StepSignal` | `stage` | `String` | no | `batch` / `validation`. |
| | `score` | `int` | no | Quality score 0-100 from `StepEvaluator`. |
| | `flags` | `List<String>` | no | Issues found (may be empty). |
| | `evaluatedAt` | `Instant` | no | When the signal ran. |
| `AuditReview` | `reviewedBy` | `String` | no | Auditor id. |
| | `outcome` | `AuditOutcome` | no | `APPROVED` or `FLAGGED`. |
| | `notes` | `String` | no | Post-completion audit notes. |
| | `reviewedAt` | `Instant` | no | When recorded. |

## Entity state — `Job` (`JobEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | Unique id; also the `ProcessOrchestrationWorkflow` id. |
| `template` | `String` | no | Submitted template name. |
| `payload` | `String` | no | Optional parameters. |
| `submittedBy` | `String` | no | Submitter id. |
| `status` | `JobStatus` | no | See enum. |
| `stepPlan` | `Optional<StepPlan>` | yes | Populated on `StepPlanned`. |
| `detailedPlan` | `Optional<DetailedPlan>` | yes | Populated on `DetailedPlanRecorded`. |
| `stepIds` | `List<String>` | no | Step ids created from the plan (may be empty early). |
| `completedBatch` | `Optional<CompletedBatch>` | yes | Populated on `BatchCompleted`. |
| `qualityVerdict` | `Optional<QualityVerdict>` | yes | Populated on `ValidationCompleted`. |
| `jobSummary` | `Optional<JobSummary>` | yes | Populated on `JobFinalised`. |
| `resultRef` | `Optional<String>` | yes | Generated result reference on finalise. |
| `completedAt` | `Optional<Instant>` | yes | When finalised. |
| `stepSignals` | `List<StepSignal>` | no | Signals recorded by `StepEvalConsumer` (may be empty). |
| `auditReview` | `Optional<AuditReview>` | yes | Populated on `AuditReviewRecorded`. |
| `retryCount` | `int` | no | Number of retry rounds taken (0 or 1). |
| `createdAt` | `Instant` | no | When `JobCreated` emitted. |

## Entity state — `Step` (`StepEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `stepId` | `String` | no | Deterministic id `jobId + "-p" + index`. |
| `jobId` | `String` | no | Owning job. |
| `name` | `String` | no | From the `DetailedStepSpec`. |
| `description` | `String` | no | From the `DetailedStepSpec`. |
| `expectedOutput` | `String` | no | From the `DetailedStepSpec`. |
| `sequence` | `int` | no | Display order (1-based). |
| `status` | `StepStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Executor that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `result` | `Optional<String>` | yes | Written step result. |
| `outputTokens` | `Optional<Integer>` | yes | Token count of the result. |
| `doneAt` | `Optional<Instant>` | yes | When marked done. |
| `createdAt` | `Instant` | no | When `StepCreated` emitted. |

## Enums

`JobStatus`: `SUBMITTED`, `PLANNED`, `EXECUTING`, `VALIDATING`, `APPROVED`, `COMPLETED`, `PAUSED`, `FAILED`.
`StepStatus`: `OPEN`, `CLAIMED`, `DONE`.
`CheckOutcome`: `PASS`, `RETRY`.
`AuditOutcome`: `APPROVED`, `FLAGGED`.

## Events — `JobEntity`

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, template, payload, submittedBy, createdAt` | → SUBMITTED |
| `StepPlanned` | `stepPlan` | SUBMITTED → PLANNED |
| `DetailedPlanRecorded` | `detailedPlan` | PLANNED (no status change; plan is refined) |
| `StepsInitialised` | `stepIds` | PLANNED → EXECUTING |
| `BatchCompleted` | `completedBatch` | EXECUTING → VALIDATING |
| `ValidationCompleted` | `verdict` | VALIDATING → APPROVED (on PASS) |
| `RetryRequested` | `mustRetry, retryCount` | VALIDATING → EXECUTING (one bounded round) |
| `JobFinalised` | `jobSummary, resultRef, completedAt` | APPROVED → COMPLETED |
| `JobPaused` | `pausedAt` | EXECUTING → PAUSED |
| `JobResumed` | `resumedAt` | PAUSED → EXECUTING |
| `StepSignalRecorded` | `stepSignal` | (no status change; appends to `stepSignals`) |
| `AuditReviewRecorded` | `auditReview` | (no status change; COMPLETED only) |

A finalise blocked by the G1 guardrail records its reason via `recordFinaliseBlock` (no status change) and leaves the job `APPROVED`. `pauseJob` is accepted only when status is `EXECUTING`; `resumeJob` only when `PAUSED`; `recordAuditReview` only when `COMPLETED`.

## Events — `StepEntity`

| Event | Payload | Transition |
|---|---|---|
| `StepCreated` | `stepId, jobId, name, description, expectedOutput, sequence, createdAt` | → OPEN |
| `StepClaimed` | `executorId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `StepDone` | `result, outputTokens, doneAt` | CLAIMED → DONE |
| `StepReleased` | `releasedAt` | CLAIMED → OPEN, clears `claimedBy` |

## Events — `JobQueue`

| Event | Payload |
|---|---|
| `JobSubmitted` | `jobId, template, payload, submittedBy, submittedAt` |

## View rows

`JobRow` (row of `JobBoardView`) mirrors `Job` but drops the heavy `CompletedBatch.outputs` step results and the `CheckNote` explanation text — it keeps the plan objective, the detailedPlan approach, `stepIds`, the batch headline and total token count, the verdict outcome, the `stepSignals`, the `resultRef`, and the `auditReview`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`StepRow` (row of `StepBoardView`) mirrors `Step` but drops the heavy `result` — it keeps `outputTokens` and a `resultPresent` boolean so the executors' poll and the board stay small. Every nullable lifecycle field is `Optional<T>`.

Both views expose one unfiltered query (`getAllJobs` / `getAllSteps`) plus a streaming query for the SSE endpoints. Status filtering happens client-side because the views cannot auto-index the enum status column (Lesson 2).
