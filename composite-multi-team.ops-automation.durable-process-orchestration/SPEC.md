# SPEC — sk-process

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SK Process.
**One-line pitch:** Submit an automation job; a process-coordinator agent plans the ordered steps, a pool of executor agents claims steps off a shared board, a quality-checker panel validates each completed step, and a passing verdict finalises the job — with runtime monitoring and graceful pause/stop at every stage.

## 2. What this blueprint demonstrates

The **composite-multi-team** coordination pattern applied to long-running operational workflows, wired with Akka's first-party primitives: a top-level process-orchestration pipeline delegates three stages — planning, execution, validation — to three capability desks, each running a *different* internal coordination capability over one shared job workspace. The planning desk uses **delegation**: a step planner decomposes the job into ordered steps and the pipeline fans out one executor per step. The execution desk uses a **team** over a shared list: a roster of executor loops claim open steps atomically from the shared step board and run them. The validation desk uses **moderation**: a panel of quality-checker instances each evaluate the completed step batch on one criterion, and a deterministic rule turns the panel's verdicts into a pass-or-retry outcome.

The blueprint also demonstrates four governance mechanisms a durable automation pipeline needs: a **before-agent-response guardrail** on the job summary assembled at completion, a **before-tool-call guardrail** on every write into the shared step workspace, **runtime monitoring** of long-running processes via an operator dashboard, and **graceful degradation** — safe pause and controlled stop that preserve state without data loss.

## 3. User-facing flows

The user opens the App UI tab and submits a job via the form (a process template name, an optional payload, and an optional submitter label).

1. The system logs the submission on `JobQueue` and a `Consumer` starts a `ProcessOrchestrationWorkflow` for the job.
2. The `ProcessCoordinator` agent plans the job — it produces a `StepPlan` (an ordered list of step definitions with their expected outputs). The job moves to `PLANNED`.
3. **Execution desk (team over a shared list).** The `StepPlanner` agent refines the plan into a `DetailedPlan`. The workflow writes one `StepEntity` per step onto the board (status `OPEN`) and the job moves to `EXECUTING`. The executor roster (`executor-1`, `executor-2`, `executor-3`) is already running — each executor is an `ExecutorWorkflow` that polls the shared `StepBoardView` for an `OPEN` step, claims it atomically, runs the `StepExecutor` agent to produce the step output through `StepTools`, and marks the step `DONE`. If two executors race for the same step, one wins and the other returns to polling.
4. When every step for the job is `DONE`, the workflow assembles a `CompletedBatch` and the job moves to `VALIDATING`.
5. **Validation desk (moderation).** The workflow runs the quality panel (`format`, `completeness`, `policy`) — one `QualityChecker` instance per criterion, each writing a `CheckNote` (a per-criterion `PASS`/`RETRY` verdict and explanation). A deterministic `QualityRule` aggregates the notes into a `QualityVerdict`: `PASS` only when every criterion passes, otherwise `RETRY` with the list of steps to redo. The job moves through `VALIDATING` then to `APPROVED` or back to `EXECUTING` for one retry round.
6. **Finalise.** On a passing verdict the workflow runs `ProcessCoordinator` once more to assemble the `JobSummary` from the approved batch. A before-agent-response guardrail vets the summary before it is persisted. The job moves to `COMPLETED` with a result reference.
7. A `StepEvalConsumer` fires a non-blocking quality signal on each step completion and on the validation verdict and records a `StepSignal` on the job, surfaced beside the job in the UI.
8. After a job is `COMPLETED`, an operator can post a post-completion `AuditReview` (approved or flagged, with notes). It is recorded against the finished job and shown in the UI; it does not change the completed state.
9. An operator may send a pause signal at any time during execution. The `ProcessOrchestrationWorkflow` enters a `PAUSED` checkpoint and the operator dashboard reflects this; a resume signal returns the workflow to the same execution step without data loss.

A `JobSimulator` (TimedAction) drips a sample process template every 90 seconds so the App UI is not empty when first loaded. A `StuckStepMonitor` (TimedAction) releases any step that has been claimed but idle for more than three minutes, returning it to `OPEN` so another executor can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ProcessCoordinator` | `AutonomousAgent` | Plans the job (produces `StepPlan`) and, at finalise, assembles the `JobSummary` under an output guardrail. | `ProcessOrchestrationWorkflow` | returns typed result to workflow; writes via `JobEntity` |
| `StepPlanner` | `AutonomousAgent` | Refines the plan into a `DetailedPlan` with step-level context and expected outputs. | `ProcessOrchestrationWorkflow` | returns `DetailedPlan` |
| `StepExecutor` | `AutonomousAgent` | Produces one step's output for one step; writes it via `StepTools`. Run as several instances (`executor-1`, `executor-2`, `executor-3`). | `ExecutorWorkflow` | `StepTools` → `StepEntity` |
| `QualityChecker` | `AutonomousAgent` | Evaluates the completed batch on one criterion; returns a `CheckNote`; writes it via `StepTools`. Run as several instances (`format`, `completeness`, `policy`). | `ProcessOrchestrationWorkflow` | `StepTools` → `JobEntity` |
| `ProcessOrchestrationWorkflow` | `Workflow` | The coordinator pipeline: plan → execute → validate → finalise, with a retry loop and pause/resume support. | `JobRequestConsumer` | all agents, `JobEntity`, `StepEntity`, `StepBoardView` |
| `ExecutorWorkflow` | `Workflow` | Per-executor durable loop: poll the board → claim a step atomically → run executor → mark done. | `Bootstrap` (one instance per executor id) | `StepEntity`, `StepExecutor`, `StepBoardView` |
| `JobEntity` | `EventSourcedEntity` | The shared job workspace: one job's lifecycle, plan, batch, verdict, summary, step signals, audit review. | `ProcessOrchestrationWorkflow`, `StepTools`, `StepEvalConsumer`, `JobQueueEndpoint` | `JobBoardView` |
| `StepEntity` | `EventSourcedEntity` | One per step. Atomic `claim`; step lifecycle. | `ProcessOrchestrationWorkflow`, `ExecutorWorkflow`, `StepTools`, `StuckStepMonitor` | `StepBoardView` |
| `JobQueue` | `EventSourcedEntity` | Single instance; logs each submitted job for replay and audit. | `JobQueueEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `JobBoardView` | `View` | The list-of-jobs read model the UI streams. | `JobEntity` events | `JobQueueEndpoint` |
| `StepBoardView` | `View` | The shared step board the executors poll and the UI shows. | `StepEntity` events | `ExecutorWorkflow`, `ProcessOrchestrationWorkflow`, `JobQueueEndpoint` |
| `JobRequestConsumer` | `Consumer` | Listens to `JobQueue` events; starts a `ProcessOrchestrationWorkflow` per submission. | `JobQueue` events | `ProcessOrchestrationWorkflow`, `JobEntity` |
| `StepEvalConsumer` | `Consumer` | Listens to `JobEntity` step events; runs a non-blocking signal eval and records a `StepSignal`. | `JobEntity` events | `JobEntity` |
| `JobSimulator` | `TimedAction` | Drips a sample process template every 90 s. | scheduler | `JobQueue` |
| `StuckStepMonitor` | `TimedAction` | Every 45 s, releases steps claimed but idle > 3 min back to `OPEN`. | scheduler | `StepEntity` |
| `JobQueueEndpoint` | `HttpEndpoint` | `/api/*` — submit, get job, list jobs, list steps, pause/resume, audit review, SSE, metadata. | — | `JobQueue`, `JobBoardView`, `StepBoardView`, `JobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `ExecutorWorkflow` per executor id. | — | `ExecutorWorkflow`, scheduler |

Companion classes that are not Akka components: `ProcessTasks` (the `Task<R>` constants), `StepTools` (the function tool the worker agents call to write into the workspace; the before-tool-call guardrail vets it), and `QualityRule` (the deterministic pure function that aggregates the checker panel's notes into a verdict).

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record JobRequest(String jobId, String template, String payload, String submittedBy) {}

record StepDefinition(String name, String description, String expectedOutput) {}
record StepPlan(String objective, List<StepDefinition> steps) {}

record DetailedStepSpec(String stepId, String name, String description, String expectedOutput, int sequence) {}
record DetailedPlan(String approach, List<DetailedStepSpec> steps) {}

record StepOutput(String stepId, String name, String result, int outputTokens) {}
record CompletedBatch(String jobId, List<StepOutput> outputs, int totalOutputTokens) {}

record CheckNote(String criterion, CheckOutcome outcome, String explanation) {}
record QualityVerdict(CheckOutcome outcome, List<CheckNote> notes, List<String> mustRetry) {}

record JobSummary(String jobId, String overview, List<String> keyOutputs, String resultRef, Instant completedAt) {}

record StepSignal(String stage, int score, List<String> flags, Instant evaluatedAt) {}
record AuditReview(String reviewedBy, AuditOutcome outcome, String notes, Instant reviewedAt) {}
```

### Entity state — `Job` (state of `JobEntity`, basis of the board row)

```java
record Job(
    String jobId,
    String template,
    String payload,
    String submittedBy,
    JobStatus status,
    Optional<StepPlan>       stepPlan,
    Optional<DetailedPlan>   detailedPlan,
    List<String>             stepIds,
    Optional<CompletedBatch> completedBatch,
    Optional<QualityVerdict> qualityVerdict,
    Optional<JobSummary>     jobSummary,
    Optional<String>         resultRef,
    Optional<Instant>        completedAt,
    List<StepSignal>         stepSignals,
    Optional<AuditReview>    auditReview,
    int                      retryCount,
    Instant                  createdAt
) {}

enum JobStatus { SUBMITTED, PLANNED, EXECUTING, VALIDATING, APPROVED, COMPLETED, PAUSED, FAILED }
```

### Entity state — `Step` (state of `StepEntity`, basis of the step-board row)

```java
record Step(
    String stepId,
    String jobId,
    String name,
    String description,
    String expectedOutput,
    int sequence,
    StepStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  result,
    Optional<Integer> outputTokens,
    Optional<Instant> doneAt,
    Instant createdAt
) {}

enum StepStatus  { OPEN, CLAIMED, DONE }
enum CheckOutcome { PASS, RETRY }
enum AuditOutcome { APPROVED, FLAGGED }
```

### Events

`JobEntity`: `JobCreated`, `StepPlanned`, `DetailedPlanRecorded`, `StepsInitialised`, `BatchCompleted`, `ValidationCompleted`, `RetryRequested`, `JobFinalised`, `JobPaused`, `JobResumed`, `StepSignalRecorded`, `AuditReviewRecorded`.
`StepEntity`: `StepCreated`, `StepClaimed`, `StepDone`, `StepReleased`.
`JobQueue`: `JobSubmitted`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ template, payload?, submittedBy? }` → `202 { jobId }`. Logs the submission and starts the pipeline.
- `GET /api/jobs` — list all jobs. Optional `?status=...`, filtered client-side.
- `GET /api/jobs/{id}` — one job with its plan, batch, verdict, summary, step signals, and audit review.
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `GET /api/jobs/{id}/steps` — the steps on the board for one job.
- `GET /api/steps/sse` — server-sent events stream of every step change.
- `POST /api/jobs/{id}/pause` — pause a running job (accepted when status is `EXECUTING`). Returns `200 Job` or `409`.
- `POST /api/jobs/{id}/resume` — resume a paused job (accepted when status is `PAUSED`). Returns `200 Job` or `409`.
- `POST /api/jobs/{id}/audit-review` — body `{ reviewedBy, outcome, notes }`. Records a post-completion audit review; allowed only when the job is `COMPLETED`. Returns `200 Job` or `409`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SK Process</title>`.

- **Overview** — eyebrow "Overview" + headline "SK Process"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, job state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a job submission form; a live job board grouped by status with per-job cards showing the plan objective, the step progress, the quality verdict, the step-signal scores, and (when completed) the summary and an audit-review box; plus a step-board panel grouped by step status (Open / Claimed / Done) showing the claiming executor.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **HO1 — runtime monitoring** (`hotl`, `deployer-runtime-monitoring` flavor, on the running job pipeline): the operator dashboard polls `JobBoardView` and the step board via SSE to show live job status, step progress, execution times, and signal scores for every running and completed job. An operator can act on a running job (pause, resume) through the dashboard without stopping the service. Non-blocking; system-level.
- **H1 — graceful degradation** (`halt`, `graceful-degradation` flavor, on `ProcessOrchestrationWorkflow`): when a pause signal arrives, the workflow saves a checkpoint (`JobPaused` event) and idles; executors finish their current step and stop polling. A resume signal reloads the checkpoint and the workflow continues from the same step. A controlled stop drains in-progress steps and records the last safe state, so no output is lost. Blocking on the specific job; non-blocking on the service.
- **G1 — before-agent-response guardrail** (`llm-content-evaluation` flavor, on `ProcessCoordinator`): when the coordinator assembles the `JobSummary` at finalise, a guardrail vets the response before it is persisted — it checks minimum length, requires at least one key output entry, and refuses a summary whose `resultRef` is empty. A failing summary blocks the finalise step and the job stays `APPROVED` with the guardrail reason recorded. Blocking.
- **G2 — before-tool-call guardrail** (`tool-permission` flavor, on `StepExecutor` and `QualityChecker`): every write into the shared step workspace through `StepTools` passes a guardrail that refuses a write whose target `jobId` or `stepId` is not the one the agent was assigned, a write to a job already in `COMPLETED` or `FAILED` status, or an oversized payload. A refused write records the step with the guardrail reason and the executor workflow releases the claim. Blocking.

## 9. Agent prompts

- `ProcessCoordinator` → `prompts/process-coordinator.md`. Plans the step breakdown and assembles the final summary.
- `StepPlanner` → `prompts/step-planner.md`. Refines the plan into executor-ready step specs.
- `StepExecutor` → `prompts/step-executor.md`. Runs one step and produces its output.
- `QualityChecker` → `prompts/quality-checker.md`. Evaluates the completed batch on one criterion.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a job; the coordinator plans steps; executors claim and run steps off a shared board; the quality panel validates the batch; a passing verdict assembles and finalises the job summary. The boards update live via SSE.
2. **J2** — Two executors poll at the same instant; each step ends up claimed by exactly one executor (no double-claim).
3. **J3** — The quality panel returns a `RETRY` verdict; the affected steps reset to `OPEN` and an executor rewrites them; the pipeline terminates after one bounded retry round.
4. **J4** — An executor attempts a write outside its assigned step; the before-tool-call guardrail refuses it and the step is recorded with the guardrail reason.
5. **J5** — A running job receives a pause signal; the workflow enters `PAUSED` state, all executors idle, and the operator dashboard reflects the pause; resume returns the workflow to the same step without data loss.
6. **J6** — After a job completes, an operator posts an audit review; it is recorded against the completed job and shown in the UI without changing the completed state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sk-process demonstrating the
composite-multi-team × ops-automation cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
composite-multi-team-ops-automation-durable-process-orchestration. Java
package io.akka.samples.skprocess. Akka 3.6.0. HTTP port 9440.

Components to wire (exactly):
- 4 AutonomousAgents (each extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgrade to Agent):
  * ProcessCoordinator — definition() with capability(TaskAcceptance.of(PLAN_JOB)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(FINALISE_JOB)
    .maxIterationsPerTask(2)). System prompt loaded from
    prompts/process-coordinator.md. PLAN_JOB returns StepPlan{objective, steps}.
    FINALISE_JOB returns JobSummary{jobId, overview, keyOutputs, resultRef,
    completedAt}. Register a before-agent-response guardrail on this agent that
    validates the FINALISE_JOB output (control G1).
  * StepPlanner — capability(TaskAcceptance.of(DETAIL_PLAN)
    .maxIterationsPerTask(2)). System prompt from prompts/step-planner.md.
    DETAIL_PLAN returns DetailedPlan{approach, steps: List<DetailedStepSpec>}.
  * StepExecutor — capability(TaskAcceptance.of(EXECUTE_STEP)
    .maxIterationsPerTask(3)). System prompt from prompts/step-executor.md.
    Returns StepOutput{stepId, name, result, outputTokens}. Run as several
    runtime instances (executor-1, executor-2, executor-3) — ONE agent class,
    several instance ids. Equipped with the StepTools function tool; the G2
    before-tool-call guardrail is registered on this agent.
  * QualityChecker — capability(TaskAcceptance.of(CHECK_QUALITY)
    .maxIterationsPerTask(2)). System prompt from prompts/quality-checker.md.
    Returns CheckNote{criterion, outcome, explanation}. Run as several runtime
    instances (format, completeness, policy) — ONE agent class, several instance
    ids; the criterion is passed as the instruction. Equipped with StepTools;
    the G2 before-tool-call guardrail is registered on this agent.

- 1 Workflow ProcessOrchestrationWorkflow, one instance per jobId. State carries
  jobId, template, stepPlan (Optional), detailedPlan (Optional), stepIds
  (List), completedBatch (Optional), verdict (Optional), retryCount, paused
  (boolean), and stage. Steps:
  planStep -> detailStep -> executionStep -> validationStep -> finaliseStep.
  Also supports pause/resume signals between steps.
  * planStep (stepTimeout 60s): call forAutonomousAgent(ProcessCoordinator.class,
    jobId).runSingleTask(PLAN_JOB.instructions(template, payload)) then
    forTask(taskId).result(PLAN_JOB); call JobEntity.recordPlan(stepPlan).
    Go to detailStep.
  * detailStep (stepTimeout 60s): call StepPlanner DETAIL_PLAN -> DetailedPlan;
    call JobEntity.recordDetailedPlan(detailedPlan). Write one StepEntity per
    DetailedStepSpec with deterministic stepId = jobId + "-p" + index
    (status OPEN) and call JobEntity.initialiseSteps(stepIds). Go to
    executionStep.
  * executionStep (team over a shared list): WAIT — query
    StepBoardView.getAllSteps, filter to this job; if every step is DONE,
    assemble a CompletedBatch from the step outputs and call
    JobEntity.recordBatch(batch) and go to validationStep; if paused=true,
    save checkpoint and pause workflow; otherwise schedule a 5s resume timer.
    The executor roster fills steps independently (see ExecutorWorkflow).
  * validationStep (stepTimeout 120s): for each criterion in [format,
    completeness, policy] call forAutonomousAgent(QualityChecker.class,
    jobId + "-" + criterion).runSingleTask(CHECK_QUALITY.instructions(
    criterion, completedBatch)) then result(CHECK_QUALITY); each checker writes
    its CheckNote via StepTools. Pass the notes through QualityRule ->
    QualityVerdict. Call JobEntity.recordVerdict(verdict). On PASS go to
    finaliseStep. On RETRY, if retryCount < 1, call
    JobEntity.requestRetry(mustRetry), reset named steps to OPEN on their
    StepEntity, increment retryCount, and go back to executionStep; if
    retryCount already 1, accept the batch and go to finaliseStep.
  * finaliseStep (stepTimeout 60s): call ProcessCoordinator FINALISE_JOB ->
    JobSummary (guarded by G1). If the guardrail refuses, call
    JobEntity.recordFinaliseBlock(reason) and end with the job left APPROVED.
    Otherwise call JobEntity.finalise(summary, ref) with ref =
    "https://ops.example/results/" + jobId. End the workflow.
  Handle pause signal: when a PAUSE command is received while in executionStep,
  emit JobPaused, set paused=true, and self-suspend. On RESUME command, emit
  JobResumed, set paused=false, schedule immediate resume, and re-enter
  executionStep from the same checkpoint.

- 1 Workflow ExecutorWorkflow, one durable instance per executor id
  (executor-1, executor-2, executor-3). State carries executorId and the
  currently-claimed stepId (Optional). Steps: pollStep -> claimStep ->
  executeStep -> finishStep.
  * pollStep: query StepBoardView.getAllSteps; pick the lowest-sequence OPEN
    step; if none, schedule a 5s resume timer and pause; else go to claimStep.
    Also check if the owning job is PAUSED — if so, schedule a 10s resume timer
    and wait.
  * claimStep: call StepEntity.claim(executorId); if the entity reports the step
    is no longer OPEN (lost the race), go back to pollStep; else go to
    executeStep.
  * executeStep (stepTimeout 120s): call forAutonomousAgent(StepExecutor.class,
    executorId).runSingleTask(EXECUTE_STEP.instructions(step)); the executor
    writes the result via StepTools and the workflow calls
    StepEntity.recordDone(result, outputTokens). If a guardrail refusal, call
    StepEntity.release (back to OPEN) and go to pollStep. Go to finishStep.
  * finishStep: clear the claimed step and go to pollStep.

- 3 EventSourcedEntities:
  * JobEntity holding Job state, one per jobId. Commands: createJob, recordPlan,
    recordDetailedPlan, initialiseSteps, recordBatch, recordVerdict,
    requestRetry, finalise, recordFinaliseBlock, recordStepSignal,
    recordAuditReview, pauseJob, resumeJob, appendCheckNote, getJob.
    JobStatus enum: SUBMITTED, PLANNED, EXECUTING, VALIDATING, APPROVED,
    COMPLETED, PAUSED, FAILED. Events: JobCreated, StepPlanned,
    DetailedPlanRecorded, StepsInitialised, BatchCompleted, ValidationCompleted,
    RetryRequested, JobFinalised, JobPaused, JobResumed, StepSignalRecorded,
    AuditReviewRecorded. emptyState() returns Job.initial("") with NO
    commandContext() reference. recordAuditReview is accepted only when
    status==COMPLETED. pauseJob accepted only when status==EXECUTING.
    resumeJob accepted only when status==PAUSED.
  * StepEntity holding Step state, one per stepId. Commands: createStep, claim,
    recordDone, release, getStep. claim emits StepClaimed ONLY when
    status==OPEN; otherwise it is a no-op returning the current Step.
    StepStatus enum: OPEN, CLAIMED, DONE. Events: StepCreated, StepClaimed,
    StepDone, StepReleased. emptyState() returns Step.initial("", "") with NO
    commandContext() reference.
  * JobQueue, single instance "default". Command submitJob(JobRequest) emitting
    JobSubmitted{jobId, template, payload, submittedBy, submittedAt}.

- 2 Views:
  * JobBoardView with row type JobRow (mirrors Job but drops the heavy
    CompletedBatch outputs and CheckNote text — keeps plan objective,
    detailedPlan approach, stepIds, completedBatch headline + token count,
    verdict outcome, stepSignals, resultRef, auditReview).
    ONE query getAllJobs SELECT * AS jobs FROM job_board. No WHERE status
    filter (Lesson 2); callers filter client-side. Add streamAllJobs for SSE.
  * StepBoardView with row type StepRow (mirrors Step minus the heavy result
    — keeps outputTokens and a resultPresent boolean). ONE query getAllSteps
    SELECT * AS steps FROM step_board. No WHERE filter; callers filter
    client-side. Add streamAllSteps for SSE.

- 2 Consumers:
  * JobRequestConsumer subscribed to JobQueue events; on JobSubmitted calls
    JobEntity.createJob then starts a ProcessOrchestrationWorkflow with the
    jobId as the workflow id.
  * StepEvalConsumer subscribed to JobEntity events; on BatchCompleted and
    ValidationCompleted it runs a deterministic StepEvaluator (pure function,
    NOT an LLM call) producing a StepSignal{stage, score, flags, evaluatedAt}
    and calls JobEntity.recordStepSignal(signal). Ignore all other JobEntity
    events. This is the informational monitoring signal.

- 2 TimedActions:
  * JobSimulator — every 90s, reads the next line from
    src/main/resources/sample-events/process-templates.jsonl (wraps when
    exhausted) and calls JobQueue.submitJob.
  * StuckStepMonitor — every 45s, queries StepBoardView.getAllSteps, finds steps
    in CLAIMED whose claimedAt is older than 3 minutes, and calls
    StepEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * JobQueueEndpoint at /api with POST /jobs, GET /jobs (client-side filter by
    status), GET /jobs/{id}, GET /jobs/sse, GET /jobs/{id}/steps,
    GET /steps/sse, POST /jobs/{id}/pause (accepted only when EXECUTING; returns
    409 otherwise), POST /jobs/{id}/resume (accepted only when PAUSED; returns
    409 otherwise), POST /jobs/{id}/audit-review (accepted only when COMPLETED;
    returns 409 otherwise), and three /api/metadata/* endpoints serving YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule JobSimulator and StuckStepMonitor; start
  one ExecutorWorkflow instance per executor id
  (executor-1, executor-2, executor-3).

Companion files:
- ProcessTasks.java declaring the Task<R> constants: PLAN_JOB (resultConformsTo
  StepPlan), FINALISE_JOB (JobSummary), DETAIL_PLAN (DetailedPlan),
  EXECUTE_STEP (StepOutput), CHECK_QUALITY (CheckNote).
- StepTools.java — the function tool the worker agents (StepExecutor,
  QualityChecker) call to write into the shared workspace:
  writeStepResult(stepId, result), appendCheckNote(jobId, note). Each method
  routes through the StepEntity / JobEntity. The G2 before-tool-call guardrail
  vets every call: refuse when the jobId/stepId is not the one the agent was
  assigned, when the target job is COMPLETED or FAILED, or when the payload
  exceeds a size cap.
- QualityRule.java (deterministic, in application/) — a pure function over a
  List<CheckNote> returning a QualityVerdict: outcome is PASS only when every
  note is PASS; otherwise RETRY with mustRetry = the distinct step names named
  in the RETRY notes' explanations (or all steps when none are named). NOT an
  LLM call.
- StepEvaluator.java (deterministic, in application/) — a pure function used by
  StepEvalConsumer: scores a stage result 0-100 and lists flags (e.g., batch
  with fewer than two outputs, a step result under a token floor, a validation
  with any RETRY criterion). NOT an LLM call.
- Domain records JobRequest, StepDefinition, StepPlan, DetailedStepSpec,
  DetailedPlan, StepOutput, CompletedBatch, CheckNote, QualityVerdict,
  JobSummary, StepSignal, AuditReview, and the enums JobStatus, StepStatus,
  CheckOutcome, AuditOutcome in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9440 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also sk-process.executors =
  ["executor-1","executor-2","executor-3"] and sk-process.quality-criteria =
  ["format","completeness","policy"] read by Bootstrap and
  ProcessOrchestrationWorkflow.
- src/main/resources/sample-events/process-templates.jsonl with 6 canned
  process templates (e.g., "data-pipeline-validation", "config-drift-check",
  "dependency-audit", "api-contract-verification", "log-anomaly-scan",
  "deployment-readiness-check").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (HO1
  deployer-runtime-monitoring, H1 graceful-degradation, G1
  before-agent-response guardrail, G2 before-tool-call guardrail) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/process-coordinator.md, prompts/step-planner.md,
  prompts/step-executor.md, prompts/quality-checker.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: SK Process", one-line
  pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sections from governance.html with answers from risk-survey.yaml; unanswered
  .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows and a colored mechanism
  pill in the ID column), App UI (job form + a job board grouped by JobStatus
  showing plan/step-progress/verdict/signals/summary/audit box, and a step-
  board panel grouped by StepStatus showing the claiming executor). Browser
  title exactly: <title>Akka Sample: SK Process</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate MockModelProvider with per-agent
        dispatch. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file. Only the REFERENCE lives on disk.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class name
  (or Task<R> id). Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    process-coordinator.json — 4-6 entries. PLAN_JOB variants are StepPlan with
      an objective and 3-5 StepDefinition items. FINALISE_JOB variants are
      JobSummary with an overview, 2-4 keyOutputs, and a non-empty resultRef.
      Include 1 FINALISE_JOB entry whose keyOutputs list is empty so the G1
      guardrail blocks it for coverage on the finalise path.
    step-planner.json — 4-6 DetailedPlan entries, each an approach sentence and
      3-5 DetailedStepSpec items with a name, description, expectedOutput, and
      sequence.
    step-executor.json — 4-6 StepOutput entries, each a stepId, name, result
      paragraph, and outputTokens. Include 1 entry whose result is empty to
      exercise a release-and-retry.
    quality-checker.json — 6-8 CheckNote entries spanning the three criteria.
      Most are PASS; include at least one RETRY note per the J3 retry journey,
      naming a step in its explanation so QualityRule can populate mustRetry.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26.
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1).
- Views have no WHERE filter on enum status column (Lesson 2).
- Workflow steps calling agents set explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port 9440 in application.conf (Lessons 10, 13).
- static-resources/index.html includes mermaid CSS overrides AND theme
  variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching MUST match by data-tab / data-panel attribute (Lesson 26).
- No forbidden words in user-facing text (Lessons 21, 22, 23).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
