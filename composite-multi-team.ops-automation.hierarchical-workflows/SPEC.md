# SPEC — hierarchical-workflow-automation

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Hierarchical Workflow Automation.
**One-line pitch:** Submit an operations request; an orchestrator agent plans a structured workflow, a discovery team delegates to system scanners and synthesises their findings, an execution team's workers claim tasks from a shared board, a validation panel scores the results, and a passing verdict assembles and delivers the final operations report.

## 2. What this blueprint demonstrates

The **composite-multi-team** coordination pattern wired with Akka's first-party primitives: a top-level orchestrator pipeline delegates three stages — discovery, execution, validation — to three specialist teams, each running a *different* internal coordination capability over one shared workflow workspace. The discovery team uses **delegation**: a discovery lead plans which systems to probe and what to look for, fans out one scanner per target, then synthesises their scan results into a discovery summary. The execution team uses a **team** over a shared list: an execution lead plans tasks onto a board, and a roster of executor loops claim open tasks atomically, carry them out through `SystemTools`, and mark them complete. The validation team uses **moderation**: a panel of validator instances each score the execution result on one axis, and a deterministic rule turns the panel's notes into a pass-or-retry verdict. The blueprint also demonstrates four governance mechanisms a cross-system automation pipeline needs: a **before-agent-response guardrail** on the final report, a **before-tool-call guardrail** on every cross-system action via `SystemTools`, a non-blocking **audit eval** on each stage result, and a non-blocking **post-execution compliance review** of completed workflows.

## 3. User-facing flows

The user opens the App UI tab and submits an operations request (a description plus an optional requester name).

1. The system logs the submission on `RequestQueue` and a `Consumer` starts a `WorkflowInstance` for the request.
2. The `Orchestrator` agent decomposes the request — it produces a `WorkflowPlan` (the objective, target systems, and the execution task list). The workflow moves to `PLANNED`.
3. **Discovery team (delegation).** The `DiscoveryLead` plans a `DiscoveryPlan` (a list of target systems to scan). The workflow runs one `SystemScanner` instance per target; each scanner writes a `ScanResult` into the shared workspace through `SystemTools`. The lead then synthesises the results into a `DiscoverySummary`. The workflow moves to `DISCOVERED`.
4. **Execution team (team over a shared list).** The `ExecutionLead` plans an `ExecutionPlan`. The workflow writes one `TaskEntity` per task onto the board (status `OPEN`) and the workflow moves to `EXECUTING`. The executor roster (`executor-1`, `executor-2`) is already running — each executor is an `ExecutorWorkflow` that polls `TaskBoardView` for an `OPEN` task, claims it atomically, runs the `TaskExecutor` agent to carry out the action through `SystemTools`, and marks the task `DONE`. If two executors race for the same task, one wins and the other returns to polling.
5. When every task for the request is `DONE`, the workflow assembles an `ExecutionSummary` and moves to `EXECUTED`.
6. **Validation team (moderation).** The workflow runs the validation panel (`correctness`, `safety`, `compliance`) — one `Validator` instance per axis, each writing a `ValidationNote` (a per-axis `PASS`/`RETRY` verdict and comments). A deterministic `AggregationRule` turns the notes into a `ValidationVerdict`: `PASS` only when every axis passes, otherwise `RETRY` with the list of tasks to redo. The workflow moves to `VALIDATING` then to `VALIDATED` or back to `EXECUTING` for one retry round.
7. **Report.** On a passing verdict the workflow runs `Orchestrator` once more to assemble the final `OpsReport` from the validated execution summary. A before-agent-response guardrail vets the report before it is persisted. The workflow moves to `COMPLETED` with a generated report URL.
8. An `AuditEvalConsumer` fires a non-blocking quality eval on each stage result (discovery summary, execution summary, validation verdict) and records an `AuditEval` on the workflow entity, surfaced beside the request in the UI.
9. After a workflow reaches `COMPLETED`, an operator can post a post-execution `ComplianceReview` (cleared or flagged, with comments). It is recorded against the completed workflow and shown in the UI; it does not change the completed state — the reviewer is on the loop, not in it.

A `RequestSimulator` (TimedAction) drips a sample operations request every 60 seconds so the App UI is not empty when first loaded. A `StuckTaskMonitor` (TimedAction) releases any task that has been claimed but idle for more than two minutes, returning it to `OPEN` so another executor can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `Orchestrator` | `AutonomousAgent` | Decomposes the operations request (produces `WorkflowPlan`) and, at completion, assembles the final `OpsReport` under an output guardrail. | `WorkflowInstance` | returns typed result to workflow; writes via `WorkflowEntity` |
| `DiscoveryLead` | `AutonomousAgent` | Plans discovery targets and synthesises the scan results into a `DiscoverySummary`. | `WorkflowInstance` | returns `DiscoveryPlan` / `DiscoverySummary` |
| `SystemScanner` | `AutonomousAgent` | Produces one `ScanResult` for one target system; writes it into the workspace via `SystemTools`. Run as several instances (one per target). | `WorkflowInstance` | `SystemTools` → `WorkflowEntity` |
| `ExecutionLead` | `AutonomousAgent` | Plans the execution tasks into an `ExecutionPlan`. | `WorkflowInstance` | returns `ExecutionPlan` |
| `TaskExecutor` | `AutonomousAgent` | Carries out one task via `SystemTools`. Run as several instances (`executor-1`, `executor-2`). | `ExecutorWorkflow` | `SystemTools` → `TaskEntity` |
| `Validator` | `AutonomousAgent` | Scores the execution result on one axis; returns a `ValidationNote`; writes it via `SystemTools`. Run as several instances (`correctness`, `safety`, `compliance`). | `WorkflowInstance` | `SystemTools` → `WorkflowEntity` |
| `WorkflowInstance` | `Workflow` | The orchestrator pipeline: plan → discovery → execution → validation → report, with a retry loop. | `RequestConsumer` | all agents, `WorkflowEntity`, `TaskEntity`, `TaskBoardView` |
| `ExecutorWorkflow` | `Workflow` | Per-executor durable loop: poll the board → claim a task atomically → execute → mark done. | `Bootstrap` (one instance per executor id) | `TaskEntity`, `TaskExecutor`, `TaskBoardView` |
| `WorkflowEntity` | `EventSourcedEntity` | The shared workflow workspace: one request's lifecycle, plan, discovery summary, execution summary, validation verdict, report, audit evals, compliance review. | `WorkflowInstance`, `SystemTools`, `AuditEvalConsumer`, `WorkflowEndpoint` | `WorkflowBoardView` |
| `TaskEntity` | `EventSourcedEntity` | One per execution task. Atomic `claim`; task lifecycle. | `WorkflowInstance`, `ExecutorWorkflow`, `SystemTools`, `StuckTaskMonitor` | `TaskBoardView` |
| `RequestQueue` | `EventSourcedEntity` | Single instance; logs each submitted operations request for replay/audit. | `WorkflowEndpoint`, `RequestSimulator` | `RequestConsumer` |
| `WorkflowBoardView` | `View` | The list-of-requests read model the UI streams. | `WorkflowEntity` events | `WorkflowEndpoint` |
| `TaskBoardView` | `View` | The shared task board the executors poll and the UI shows. | `TaskEntity` events | `ExecutorWorkflow`, `WorkflowInstance`, `WorkflowEndpoint` |
| `RequestConsumer` | `Consumer` | Listens to `RequestQueue` events; starts a `WorkflowInstance` per submission. | `RequestQueue` events | `WorkflowInstance`, `WorkflowEntity` |
| `AuditEvalConsumer` | `Consumer` | Listens to `WorkflowEntity` stage events; runs a non-blocking eval and records an `AuditEval`. | `WorkflowEntity` events | `WorkflowEntity` |
| `RequestSimulator` | `TimedAction` | Drips a sample operations request every 60 s. | scheduler | `RequestQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, releases tasks claimed but idle > 2 min back to `OPEN`. | scheduler | `TaskEntity` |
| `WorkflowEndpoint` | `HttpEndpoint` | `/api/*` — submit, get request, list requests, list tasks, compliance review, SSE, metadata. | — | `RequestQueue`, `WorkflowBoardView`, `TaskBoardView`, `WorkflowEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `ExecutorWorkflow` per executor id. | — | `ExecutorWorkflow`, scheduler |

Companion classes that are not Akka components: `WorkflowTasks` (the `Task<R>` constants), `SystemTools` (the function tool the worker agents call to act on or scan systems; the before-tool-call guardrail vets it), and `AggregationRule` (the deterministic pure function that aggregates the validation panel's notes into a verdict).

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record OpsRequest(String requestId, String description, String requestedBy) {}

record WorkflowPlan(String objective, List<String> targetSystems, List<String> taskDescriptions) {}

record DiscoveryPlan(List<String> scanTargets) {}
record ScanResult(String target, String findings, List<String> references) {}
record DiscoverySummary(String overview, List<String> keyFindings) {}

record TaskSpec(String title, String instruction) {}
record ExecutionPlan(String approach, List<TaskSpec> tasks) {}
record TaskOutcome(String taskId, String title, String result, int stepsCompleted) {}
record ExecutionSummary(String headline, List<TaskOutcome> outcomes, int tasksCompleted) {}

record ValidationNote(String axis, ValidationOutcome outcome, String comments) {}
record ValidationVerdict(ValidationOutcome outcome, List<ValidationNote> notes, List<String> mustRetry) {}

record OpsReport(String title, String body, String preparedBy, Instant assembledAt) {}

record AuditEval(String stage, int score, List<String> flags, Instant evaluatedAt) {}
record ComplianceReview(String reviewedBy, ComplianceOutcome outcome, String comments, Instant reviewedAt) {}
```

### Entity state — `WorkflowState` (state of `WorkflowEntity`, basis of the board row)

```java
record WorkflowState(
    String requestId,
    String description,
    String requestedBy,
    WorkflowStatus status,
    Optional<WorkflowPlan>       plan,
    Optional<DiscoverySummary>   discoverySummary,
    List<String>                 taskIds,
    Optional<ExecutionSummary>   executionSummary,
    Optional<ValidationVerdict>  validationVerdict,
    Optional<OpsReport>          report,
    Optional<String>             reportUrl,
    Optional<Instant>            completedAt,
    List<AuditEval>              auditEvals,
    Optional<ComplianceReview>   complianceReview,
    int                          retryCount,
    Instant                      createdAt
) {}

enum WorkflowStatus { SUBMITTED, PLANNED, DISCOVERING, DISCOVERED, EXECUTING, EXECUTED, VALIDATING, VALIDATED, COMPLETED }
```

### Entity state — `Task` (state of `TaskEntity`, basis of the task-board row)

```java
record Task(
    String taskId,
    String requestId,
    String title,
    String instruction,
    TaskStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  result,
    Optional<Integer> stepsCompleted,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum TaskStatus { OPEN, CLAIMED, DONE }
enum ValidationOutcome { PASS, RETRY }
enum ComplianceOutcome { CLEARED, FLAGGED }
```

### Events

`WorkflowEntity`: `WorkflowCreated`, `WorkflowPlanned`, `DiscoverySynthesized`, `TasksPlanned`, `ExecutionSummarized`, `ValidationCompleted`, `RetryRequested`, `ReportCompleted`, `AuditEvaluated`, `ComplianceReviewRecorded`.
`TaskEntity`: `TaskCreated`, `TaskClaimed`, `TaskCompleted`, `TaskReleased`.
`RequestQueue`: `RequestSubmitted`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/requests` — body `{ description, requestedBy? }` → `202 { requestId }`. Logs the submission and starts the pipeline.
- `GET /api/requests` — list all requests. Optional `?status=...`, filtered client-side.
- `GET /api/requests/{id}` — one request with its plan, discovery summary, execution summary, validation verdict, report, audit evals, and compliance review.
- `GET /api/requests/sse` — server-sent events stream of every workflow change.
- `GET /api/requests/{id}/tasks` — the tasks on the board for one request.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `POST /api/requests/{id}/compliance-review` — body `{ reviewedBy, outcome, comments }`. Records a post-execution compliance review; allowed only when the request is `COMPLETED`. Non-blocking.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Hierarchical Workflow Automation</title>`.

- **Overview** — eyebrow "Overview" + headline "Hierarchical Workflow Automation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, workflow state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a request submission form; a live request board grouped by status with per-request cards showing the plan objective, the discovery summary, the task progress, the validation verdict, the audit-eval scores, and (when completed) the report and a compliance-review box; plus a task-board panel grouped by task status (Open / Claimed / Done) showing the claiming executor.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`llm-content-evaluation` flavor, on `Orchestrator`): when the orchestrator assembles the final `OpsReport` at completion, a guardrail vets the response before it is persisted — it checks minimum length, refuses reports that omit a `preparedBy` attribution, and refuses a report that contains no documented outcomes. A failing report blocks the completion step and the workflow stays `VALIDATED` with the guardrail reason recorded. Blocking.
- **G2 — before-tool-call guardrail** (`tool-permission` flavor, on the worker agents that call `SystemTools`): every cross-system action — a scan write, a task execution, a validation note — passes through a guardrail that refuses an action addressed to a request or task the agent was not assigned, an action on a request that is already `COMPLETED`, or an oversized payload. A refused action records the stage with the guardrail reason and the agent returns without a partial side effect. Blocking.
- **E1 — audit eval** (`eval-event`, `on-decision-eval` flavor): `AuditEvalConsumer` fires on each stage result event (`DiscoverySynthesized`, `ExecutionSummarized`, `ValidationCompleted`) and runs a deterministic quality eval — a score and a list of flags — recorded as an `AuditEval` on the workflow. Informational; surfaces context beside the request. Non-blocking.
- **HO1 — post-execution compliance review** (`hotl`, `live-compliance-review` flavor): after a workflow completes, an operator can post a `ComplianceReview` through `POST /api/requests/{id}/compliance-review`. It is recorded against the completed workflow and shown in the UI but does not change the completed state — the reviewer is on the loop, not gating it. Non-blocking, system-level.

## 9. Agent prompts

- `Orchestrator` → `prompts/orchestrator.md`. Decomposes the operations request and assembles the final report.
- `DiscoveryLead` → `prompts/discovery-lead.md`. Plans discovery targets and synthesises the scan results.
- `SystemScanner` → `prompts/system-scanner.md`. Produces one scan result for one target system.
- `ExecutionLead` → `prompts/execution-lead.md`. Plans the execution tasks.
- `TaskExecutor` → `prompts/task-executor.md`. Carries out one task via SystemTools.
- `Validator` → `prompts/validator.md`. Scores the execution result on one axis.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an operations request; the orchestrator plans a workflow; the discovery team delegates and synthesises; the execution team fills tasks on a shared board; the validation panel scores the results; a passing verdict assembles and delivers the operations report. The board updates live via SSE.
2. **J2** — Two executors poll at the same instant; each task ends up claimed by exactly one executor (no double-claim).
3. **J3** — The validation panel returns a `RETRY` verdict; the failed tasks reset to `OPEN`, re-execute, and the pipeline terminates on the second pass.
4. **J4** — A worker agent attempts an action on a request or task it was not assigned; the before-tool-call guardrail refuses it before execution and the stage is recorded with the guardrail reason.
5. **J5** — After a workflow reaches `COMPLETED`, an operator posts a compliance review; it is recorded against the completed workflow and shown in the UI without changing the completed state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hierarchical-workflow-automation demonstrating the
composite-multi-team × ops-automation cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
composite-multi-team-ops-automation-hierarchical-workflows. Java package
io.akka.samples.hierarchicalworkflowautomation. Akka 3.6.0. HTTP port 9811.

Components to wire (exactly):
- 6 AutonomousAgents (each extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgrade to Agent):
  * Orchestrator — definition() with capability(TaskAcceptance.of(PLAN)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(REPORT)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/orchestrator.md.
    PLAN returns WorkflowPlan{objective, targetSystems, taskDescriptions}. REPORT
    returns OpsReport{title, body, preparedBy, assembledAt}. Register a
    before-agent-response guardrail on this agent that validates the REPORT output
    (control G1).
  * DiscoveryLead — capability(TaskAcceptance.of(PLAN_DISCOVERY)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(SYNTHESIZE)
    .maxIterationsPerTask(2)). System prompt from prompts/discovery-lead.md.
    PLAN_DISCOVERY returns DiscoveryPlan{scanTargets}. SYNTHESIZE returns
    DiscoverySummary{overview, keyFindings}.
  * SystemScanner — capability(TaskAcceptance.of(SCAN).maxIterationsPerTask(2)).
    System prompt from prompts/system-scanner.md. Returns ScanResult{target,
    findings, references}. Run as several runtime instances addressed by instanceId
    (requestId + "-sc" + targetIndex) — ONE agent class, several instance ids.
    Equipped with the SystemTools function tool; the G2 before-tool-call
    guardrail is registered on this agent.
  * ExecutionLead — capability(TaskAcceptance.of(PLAN_EXECUTION)
    .maxIterationsPerTask(2)). System prompt from prompts/execution-lead.md.
    Returns ExecutionPlan{approach, tasks: List<TaskSpec{title, instruction}>}.
  * TaskExecutor — capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(3)).
    System prompt from prompts/task-executor.md. Returns TaskOutcome{taskId, title,
    result, stepsCompleted}. Run as several runtime instances (executor-1,
    executor-2) — ONE agent class, several instance ids; never generate two executor
    classes. Equipped with SystemTools; the G2 before-tool-call guardrail is
    registered on this agent.
  * Validator — capability(TaskAcceptance.of(VALIDATE).maxIterationsPerTask(2)).
    System prompt from prompts/validator.md. Returns ValidationNote{axis, outcome,
    comments}. Run as several runtime instances (correctness, safety, compliance) —
    ONE agent class, several instance ids; the axis is passed as the instruction.
    Equipped with SystemTools; the G2 before-tool-call guardrail is registered on
    this agent.

- 1 Workflow WorkflowInstance, one instance per requestId. State carries requestId,
  description, plan (Optional), discoverySummary (Optional), taskIds (List),
  executionSummary (Optional), verdict (Optional), retryCount, and stage. Steps:
  planStep -> discoveryStep -> executionStep -> validationStep -> reportStep, with a
  self-scheduled resume timer for the execution wait.
  * planStep: call forAutonomousAgent(Orchestrator.class, requestId)
    .runSingleTask(PLAN.instructions(description)) then forTask(taskId).result(PLAN);
    call WorkflowEntity.recordPlan(plan). Go to discoveryStep.
  * discoveryStep (delegation): call DiscoveryLead PLAN_DISCOVERY -> DiscoveryPlan.
    For each target, call forAutonomousAgent(SystemScanner.class, requestId + "-sc" +
    i).runSingleTask(SCAN.instructions(target)) then result(SCAN); each scanner writes
    its ScanResult into WorkflowEntity via SystemTools. Then call DiscoveryLead
    SYNTHESIZE over the results -> DiscoverySummary; call
    WorkflowEntity.recordDiscovery(summary). Go to executionStep.
  * executionStep (team over a shared list): call ExecutionLead PLAN_EXECUTION ->
    ExecutionPlan. Write one TaskEntity per TaskSpec with a deterministic taskId =
    requestId + "-t" + index (status OPEN) and call WorkflowEntity.recordTasks(taskIds).
    Then WAIT: query TaskBoardView.getAllTasks, filter to this request; if every task
    is DONE, assemble an ExecutionSummary from the task outcomes and call
    WorkflowEntity.recordExecution(summary) and go to validationStep; otherwise
    schedule a 5s resume timer and pause. The executor roster fills the tasks
    independently (see ExecutorWorkflow).
  * validationStep (moderation): for each axis in [correctness, safety, compliance]
    call forAutonomousAgent(Validator.class, requestId + "-" + axis).runSingleTask(
    VALIDATE.instructions(axis, executionSummary)) then result(VALIDATE); each
    validator writes its ValidationNote via SystemTools. Pass the notes through
    AggregationRule -> ValidationVerdict. Call WorkflowEntity.recordValidation(verdict).
    On PASS go to reportStep. On RETRY, if retryCount < 1, call
    WorkflowEntity.requestRetry(mustRetry), reset the named tasks to OPEN on their
    TaskEntity, increment retryCount, and go back to executionStep; if retryCount
    already 1, accept the summary as-is and go to reportStep (one bounded retry round).
  * reportStep: call Orchestrator REPORT -> OpsReport (guarded by G1). If the
    guardrail refuses, call WorkflowEntity.recordReportBlock(reason) and end the
    workflow with the entity left VALIDATED. Otherwise call
    WorkflowEntity.complete(report, url) with url =
    "https://ops.example/" + requestId. End the workflow.
  Override settings() with stepTimeout(planStep, 60s),
  stepTimeout(discoveryStep, 120s), stepTimeout(validationStep, 90s),
  stepTimeout(reportStep, 60s). executionStep is a poll step with a short timeout
  and a resume timer.

- 1 Workflow ExecutorWorkflow, one durable instance per executor id (executor-1,
  executor-2). State carries executorId and the currently-claimed taskId (Optional).
  Steps: pollStep -> claimStep -> executeStep -> finishStep, with a self-scheduled
  resume timer for idling.
  * pollStep: query TaskBoardView.getAllTasks; pick the oldest OPEN task; if none,
    schedule a 5s resume timer and pause; else go to claimStep.
  * claimStep: call TaskEntity.claim(executorId); if the entity reports the task is
    no longer OPEN (lost the race), go back to pollStep; else go to executeStep.
  * executeStep (stepTimeout 90s): call forAutonomousAgent(TaskExecutor.class,
    executorId).runSingleTask(EXECUTE_TASK.instructions(task)); the executor acts via
    SystemTools and the workflow calls TaskEntity.recordDone(result, stepsCompleted).
    If a guardrail refusal comes back, call TaskEntity.release (back to OPEN) and go
    to pollStep. Go to finishStep.
  * finishStep: clear the claimed task and go to pollStep.

- 3 EventSourcedEntities:
  * WorkflowEntity holding WorkflowState, one per requestId. Commands:
    createWorkflow, recordPlan, recordDiscovery, recordTasks, recordExecution,
    recordValidation, requestRetry, complete, recordReportBlock, recordAuditEval,
    recordComplianceReview, appendScanResult, appendValidationNote, getWorkflow.
    WorkflowStatus enum: SUBMITTED, PLANNED, DISCOVERING, DISCOVERED, EXECUTING,
    EXECUTED, VALIDATING, VALIDATED, COMPLETED. Events: WorkflowCreated,
    WorkflowPlanned, DiscoverySynthesized, TasksPlanned, ExecutionSummarized,
    ValidationCompleted, RetryRequested, ReportCompleted, AuditEvaluated,
    ComplianceReviewRecorded. emptyState() returns WorkflowState.initial("") with NO
    commandContext() reference. recordComplianceReview is accepted only when
    status==COMPLETED.
  * TaskEntity holding Task state, one per taskId. Commands: createTask, claim,
    recordDone, release, getTask. claim emits TaskClaimed ONLY when status==OPEN;
    otherwise it is a no-op that returns the current Task so the caller can detect
    the lost race. TaskStatus enum: OPEN, CLAIMED, DONE. Events: TaskCreated,
    TaskClaimed, TaskCompleted, TaskReleased. emptyState() returns Task.initial("",
    "") with NO commandContext() reference.
  * RequestQueue, single instance "default". Command submitRequest(OpsRequest)
    emitting RequestSubmitted{requestId, description, requestedBy, submittedAt}.

- 2 Views:
  * WorkflowBoardView with row type WorkflowRow (mirrors WorkflowState but drops the
    heavy OpsReport body and ScanResult contents — keeps plan objective, discovery
    overview, taskIds, execution headline + tasksCompleted, verdict outcome,
    auditEvals, reportUrl, complianceReview). Table updater consumes WorkflowEntity
    events. ONE query getAllWorkflows SELECT * AS workflows FROM workflow_board. No
    WHERE status filter (Akka cannot auto-index enum columns — Lesson 2); callers
    filter client-side. Add a streamAllWorkflows query for the SSE endpoint.
  * TaskBoardView with row type TaskRow (mirrors Task minus the heavy result — keeps
    stepsCompleted only, plus a resultPresent boolean). Table updater consumes
    TaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM task_board. No
    WHERE filter; callers filter client-side. Add a streamAllTasks query for the SSE
    endpoint.

- 2 Consumers:
  * RequestConsumer subscribed to RequestQueue events; on RequestSubmitted calls
    WorkflowEntity.createWorkflow then starts a WorkflowInstance with the requestId
    as the workflow id.
  * AuditEvalConsumer subscribed to WorkflowEntity events; on DiscoverySynthesized,
    ExecutionSummarized, and ValidationCompleted it runs a deterministic
    ResultEvaluator (pure function, NOT an LLM call) producing an AuditEval{stage,
    score, flags, evaluatedAt} and calls WorkflowEntity.recordAuditEval(eval). Ignore
    all other WorkflowEntity events. This is control E1.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/ops-requests.jsonl (wraps when exhausted) and
    calls RequestQueue.submitRequest.
  * StuckTaskMonitor — every 30s, queries TaskBoardView.getAllTasks, finds tasks in
    CLAIMED whose claimedAt is older than 2 minutes, and calls TaskEntity.release
    (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * WorkflowEndpoint at /api with POST /requests, GET /requests (client-side filter
    by status), GET /requests/{id}, GET /requests/sse, GET /requests/{id}/tasks,
    GET /tasks/sse, POST /requests/{id}/compliance-review (accepted only when the
    request is COMPLETED; returns 409 otherwise), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

- 1 service-setup Bootstrap: schedule RequestSimulator and StuckTaskMonitor; start
  one ExecutorWorkflow instance per executor id (executor-1, executor-2).

Companion files:
- WorkflowTasks.java declaring the Task<R> constants: PLAN (resultConformsTo
  WorkflowPlan), REPORT (OpsReport), PLAN_DISCOVERY (DiscoveryPlan), SCAN
  (ScanResult), SYNTHESIZE (DiscoverySummary), PLAN_EXECUTION (ExecutionPlan),
  EXECUTE_TASK (TaskOutcome), VALIDATE (ValidationNote).
- SystemTools.java — the function tool the worker agents (SystemScanner, TaskExecutor,
  Validator) call to act on or scan systems: appendScanResult(requestId, result),
  executeTask(taskId, result, stepsCompleted), appendValidationNote(requestId, note).
  Each method routes through WorkflowEntity / TaskEntity. The G2 before-tool-call
  guardrail vets every call: refuse when the requestId/taskId is not the one the
  agent was assigned, when the target request is already COMPLETED, or when the
  payload exceeds a size cap.
- AggregationRule.java (deterministic, in application/) — a pure function over a
  List<ValidationNote> returning a ValidationVerdict: outcome is PASS only when every
  note is PASS; otherwise RETRY with mustRetry = the distinct task titles named in the
  RETRY notes' comments (or all tasks when none are named). This is the moderation
  aggregation; it is NOT an LLM call.
- ResultEvaluator.java (deterministic, in application/) — a pure function used by
  AuditEvalConsumer: scores a stage result 0-100 and lists flags (e.g., a discovery
  summary with fewer than three key findings, an execution summary with no completed
  tasks, a validation with any RETRY axis). NOT an LLM call.
- Domain records OpsRequest, WorkflowPlan, DiscoveryPlan, ScanResult, DiscoverySummary,
  TaskSpec, ExecutionPlan, TaskOutcome, ExecutionSummary, ValidationNote,
  ValidationVerdict, OpsReport, AuditEval, ComplianceReview, and the enums
  WorkflowStatus, TaskStatus, ValidationOutcome, ComplianceOutcome in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9811
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also
  hierarchical-workflow-automation.executors = ["executor-1","executor-2"] and
  hierarchical-workflow-automation.validation-axes = ["correctness","safety","compliance"]
  read by Bootstrap and WorkflowInstance.
- src/main/resources/sample-events/ops-requests.jsonl with 6 canned operations
  requests (each a self-contained description — e.g., "Rotate TLS certificates across
  the API gateway cluster", "Scale down idle compute nodes in the staging environment",
  "Audit IAM permissions for the data-pipeline service account",
  "Patch OS packages on the canary fleet", "Snapshot and verify database replicas",
  "Reconcile DNS records with the current load-balancer configuration").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (G1 before-agent-response
  guardrail, G2 before-tool-call guardrail, E1 eval-event on-decision-eval, HO1 hotl
  live-compliance-review) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data classes,
  capabilities, model family, and oversight; deployer-specific fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/discovery-lead.md, prompts/system-scanner.md,
  prompts/execution-lead.md, prompts/task-executor.md, prompts/validator.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Hierarchical Workflow Automation",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix on tab
  names.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs matching the formal exemplar: Overview,
  Architecture (4 mermaid diagrams + click-to-expand component table with
  syntax-highlighted Java snippets), Risk Survey (7 sections from governance.html
  with answers from risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix
  (5-column ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows and a colored mechanism pill in the ID column), App UI (request form + a
  request board grouped by WorkflowStatus showing plan/discovery/task progress/
  verdict/audit-eval scores/report/compliance box, and a task-board panel grouped by
  TaskStatus showing the claiming executor). Browser title exactly:
  <title>Akka Sample: Hierarchical Workflow Automation</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  ModelProvider. Per-agent dispatch on agent class name / Task<R> id. Each branch
  reads JSON from src/main/resources/mock-responses/<agent-name>.json, picks one
  entry pseudo-randomly per call, and deserialises into the agent's typed return.
  MockModelProvider.seedFor(id) makes selection deterministic per request/task id.
- Per-agent mock-response shapes:
    orchestrator.json — 4-6 entries with two variants. PLAN variants are
      WorkflowPlan with an objective, 2-3 targetSystems, and 3-5 taskDescriptions.
      REPORT variants are OpsReport with a title, a multi-paragraph body, and a
      non-empty preparedBy. Include 1 REPORT entry whose body is too short so G1
      blocks it for guardrail coverage.
    discovery-lead.json — 4-6 entries. PLAN_DISCOVERY variants are DiscoveryPlan
      with 2-3 scanTargets. SYNTHESIZE variants are DiscoverySummary with a
      one-paragraph overview and 3-5 keyFindings.
    system-scanner.json — 4-6 ScanResult entries, each a target, a short findings
      paragraph, and 1-3 references. Include 1 entry whose findings target a
      mismatched requestId to exercise the G2 guardrail refusal.
    execution-lead.json — 4-6 ExecutionPlan entries, each an approach sentence and
      3-5 TaskSpec items (title + instruction).
    task-executor.json — 4-6 TaskOutcome entries with a title, result, and
      stepsCompleted. Include 1 entry whose result is empty to exercise a
      release-and-retry.
    validator.json — 6-8 ValidationNote entries spanning the three axes. Most are
      PASS; include at least one RETRY note per the J-retry journey, naming a task
      title in its comments so AggregationRule can populate mustRetry.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion WorkflowTasks Task<R> constants (Lesson 7).
- Views have no WHERE filter on the enum status column; filter client-side (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9811 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24. Without these, state names render black-on-black
  and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie panels.
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export
  block. Per Lesson 25, /akka:specify handles the key during generation.
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
