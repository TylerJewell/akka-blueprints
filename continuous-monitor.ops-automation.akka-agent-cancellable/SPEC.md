# SPEC — activity-interrupt-cancellation

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Activity Interrupt / Cancellation.
**One-line pitch:** A background automation runner executes long-running agent tasks and allows operators to safely interrupt any task mid-flight, triggering cleanup and continuing with the next queued item.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two halt governance mechanisms:

- An **operator-regulator-stop** halt: the operator API can cancel any RUNNING task at any point. The signal propagates into the running workflow within one iteration boundary, preventing the agent from advancing further.
- A **graceful-degradation** halt: every cancellation path runs a `CleanupAgent` that produces a structured remediation plan — released locks, rolled-back partials, recorded reason — before the activity is marked terminal. The system never leaves a task in an ambiguous half-done state.

The result is a system where interruption is a first-class operation, not an exception case. Operators can stop any task and know it will land cleanly.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live activity board: every task, its current status, the most recent step log, and (if cancelled) the cleanup summary.
2. `TaskPoller` (TimedAction) ticks every 20 s and inserts new simulated tasks into `TaskQueue`. (A `RequestSimulator` style — drips canned tasks.)
3. For each new task, `ActivityWorkflow` starts, transitions the entity to RUNNING, and calls `TaskRunnerAgent` iteratively. Each iteration emits a `StepCompleted` event and checks for a cancellation signal on the entity.
4. If no cancellation arrives, the agent runs until it emits `TaskCompleted`, and the entity transitions to COMPLETED.
5. An operator clicks Cancel in the UI — the entity transitions to CANCELLING. On the next iteration boundary, the workflow detects the signal, stops advancing, and calls `CleanupAgent`.
6. `CleanupAgent` returns a `CleanupPlan`, which the workflow records as a `CleanupRecorded` event before finalising the entity as CANCELLED.
7. `StaleActivityReaper` (TimedAction) ticks every 5 minutes, finds any RUNNING task with `startedAt` older than the configured timeout, and emits a `CancellationRequested` event (equivalent to an operator cancel) to prevent runaway tasks.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskPoller` | `TimedAction` | Drips simulated tasks into `TaskQueue` every 20 s. | scheduler | `TaskQueue` |
| `TaskQueue` | `EventSourcedEntity` | Append-only log of `TaskSubmitted` events. | `TaskPoller`, `ActivityEndpoint` | `ActivityWorkflow` (via Consumer) |
| `TaskDispatcher` | `Consumer` | Subscribes to `TaskSubmitted` events on `TaskQueue`; starts one `ActivityWorkflow` per task. | `TaskQueue` events | `ActivityWorkflow`, `ActivityEntity` |
| `TaskRunnerAgent` | `AutonomousAgent` | Executes a task step-by-step. Each iteration produces one `StepResult`. Checks `shouldContinue` flag before each step. | invoked by Workflow | returns `StepResult` |
| `CleanupAgent` | `Agent` (typed, not autonomous) | Given a partially-executed task and its last step log, produces a `CleanupPlan`. | invoked by Workflow | returns `CleanupPlan` |
| `ActivityWorkflow` | `Workflow` | Per-task orchestration: startStep → executeStep (loop) → cancellationCheck → cleanupStep → finaliseStep. | `TaskDispatcher` | `ActivityEntity` |
| `ActivityEntity` | `EventSourcedEntity` | Lifecycle per task. All status transitions and step logs live here. | `ActivityWorkflow`, `ActivityEndpoint` | `ActivityView` |
| `ActivityView` | `View` | Read-model row per task for the UI. | `ActivityEntity` events | `ActivityEndpoint` |
| `StaleActivityReaper` | `TimedAction` | Every 5 min, queries `ActivityView` for RUNNING tasks beyond timeout; emits `CancellationRequested`. | scheduler | `ActivityEntity` |
| `ActivityEndpoint` | `HttpEndpoint` | `/api/activities/*` — list, get, submit, cancel, SSE. | — | `ActivityView`, `ActivityEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record ActivityTask(String taskId, String name, String description,
                    TaskKind kind, int estimatedSteps, Instant submittedAt) {}

enum TaskKind { INFRA_PROVISION, REPORT_GENERATE, DATA_PIPELINE, MAINTENANCE }

record StepResult(int stepIndex, String summary, boolean terminal,
                  Optional<String> partialArtifact, Instant completedAt) {}

record CleanupPlan(List<String> steps, String reason,
                   Optional<String> rollbackNote, Instant planCreatedAt) {}

record CancellationRequest(String requestedBy, String reason, Instant requestedAt) {}

record ActivityRecord(
    String taskId,
    ActivityTask task,
    List<StepResult> stepLog,
    Optional<CleanupPlan> cleanupPlan,
    Optional<CancellationRequest> cancellationRequest,
    ActivityStatus status,
    Instant createdAt,
    Optional<Instant> startedAt,
    Optional<Instant> finishedAt
) {}

enum ActivityStatus {
    QUEUED, RUNNING, CANCELLING, CANCELLED, COMPLETED, FAILED
}
```

Events on `ActivityEntity`: `TaskQueued`, `TaskStarted`, `StepCompleted`, `CancellationRequested`, `CleanupRecorded`, `TaskCancelled`, `TaskCompleted`, `TaskFailed`.

Events on `TaskQueue`: `TaskSubmitted` (the immutable audit record of every submission).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/activities` — list all activities. Optional `?status=…`.
- `GET /api/activities/{id}` — one activity.
- `POST /api/activities` — body `{ name, description, kind, estimatedSteps }` → submits a new task; returns the queued `ActivityRecord`.
- `POST /api/activities/{id}/cancel` — body `{ requestedBy, reason }` → transitions RUNNING/QUEUED to CANCELLING.
- `GET /api/activities/sse` — Server-Sent Events for every activity state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Activity Interrupt / Cancellation</title>`.

App UI tab is the most distinctive: it shows the **live activity board** with a status column and expandable step log per task. Cancel button is available on any RUNNING or QUEUED task. Cancelled tasks show the cleanup plan inline.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — operator-regulator-stop halt** (`ActivityEndpoint.cancel`): a single `POST .../cancel` transitions any running task to CANCELLING. The workflow detects this on the next iteration boundary and stops advancing the agent.
- **H2 — graceful-degradation halt** (`CleanupAgent`): every cancellation (operator-initiated or timeout-triggered) invokes `CleanupAgent` before marking the entity CANCELLED. The cleanup plan is persisted and visible in the UI.

## 9. Agent prompts

- `TaskRunnerAgent` → `prompts/task-runner.md`. AutonomousAgent. Each iteration returns one `StepResult`. Stops when `terminal=true` or when instructed to stop.
- `CleanupAgent` → `prompts/cleanup.md`. Typed Agent. Given an `ActivityTask` and a list of completed `StepResult` entries, returns a `CleanupPlan` with concrete rollback steps.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a task; it appears in the UI within 20 s; transitions QUEUED → RUNNING; step log grows.
2. **J2** — Operator clicks Cancel; task transitions RUNNING → CANCELLING → CANCELLED; cleanup plan is visible.
3. **J3** — Task that runs to natural completion reaches COMPLETED; no cleanup runs.
4. **J4** — `StaleActivityReaper` auto-cancels a RUNNING task that has exceeded the timeout.
5. **J5** — Cancelled task's step log is preserved; operator can inspect every step that did complete.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named activity-interrupt-cancellation demonstrating the
continuous-monitor × ops-automation cell. Runs out of the box (in-memory
task simulator; no real queue integration). Maven group io.akka.samples.
Artifact id continuous-monitor-ops-automation-akka-agent-cancellable.
Java package io.akka.samples.activityinterruptcancellation. HTTP port 9855.
Akka version 3.6.0

Components to wire (exactly):

- 1 AutonomousAgent TaskRunnerAgent — definition() with capability(
  TaskAcceptance.of(EXECUTE).maxIterationsPerTask(10)). System prompt from
  prompts/task-runner.md. Input: ActivityTask + boolean shouldContinue.
  Output: StepResult{stepIndex: int, summary: String, terminal: boolean,
  partialArtifact: Optional<String>, completedAt: Instant}. When
  shouldContinue=false, TaskRunnerAgent returns StepResult with
  terminal=true and summary="Cancelled at operator request."

- 1 Agent (typed, NOT autonomous) CleanupAgent — system prompt from
  prompts/cleanup.md. Input: ActivityTask + List<StepResult> completedSteps.
  Output: CleanupPlan{steps: List<String>, reason: String, rollbackNote:
  Optional<String>, planCreatedAt: Instant}.

- 1 Workflow ActivityWorkflow per task with steps:
  startStep -> executeStep (loop up to estimatedSteps+2 iterations) ->
  cancellationCheckStep -> cleanupStep (only if CANCELLING) -> finaliseStep.
  startStep calls ActivityEntity.markStarted; emits TaskStarted.
  executeStep calls TaskRunnerAgent with shouldContinue derived from entity
  state (CANCELLING → false, otherwise true). On StepResult.terminal=true,
  proceeds to finaliseStep directly. On each non-terminal step, emits
  StepCompleted and loops back to executeStep.
  cancellationCheckStep reads entity status; if CANCELLING → cleanupStep,
  else → finaliseStep.
  cleanupStep calls CleanupAgent with stepTimeout(Duration.ofSeconds(30));
  emits CleanupRecorded.
  finaliseStep emits TaskCancelled (if came from cleanupStep) or
  TaskCompleted (normal exit).
  WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(15)) for
  executeStep; .stepTimeout(Duration.ofSeconds(10)) for startStep.
  No auto-timeout on cleanupStep — cleanup must complete.

- 2 EventSourcedEntities:
  * TaskQueue — append-only audit log. Command submit(ActivityTask) emits
    TaskSubmitted{task}. emptyState() returns empty list; no commandContext().
  * ActivityEntity (one per taskId) — full lifecycle.
    State: ActivityRecord{taskId, task: ActivityTask{taskId, name,
    description, kind: TaskKind (enum INFRA_PROVISION/REPORT_GENERATE/
    DATA_PIPELINE/MAINTENANCE), estimatedSteps: int, submittedAt: Instant},
    stepLog: List<StepResult>, cleanupPlan: Optional<CleanupPlan>,
    cancellationRequest: Optional<CancellationRequest{requestedBy: String,
    reason: String, requestedAt: Instant}}, status: ActivityStatus (enum
    QUEUED/RUNNING/CANCELLING/CANCELLED/COMPLETED/FAILED),
    createdAt: Instant, startedAt: Optional<Instant>,
    finishedAt: Optional<Instant>}.
    Events: TaskQueued, TaskStarted, StepCompleted{stepResult},
    CancellationRequested{cancellationRequest}, CleanupRecorded{cleanupPlan},
    TaskCancelled, TaskCompleted, TaskFailed{reason}.
    Commands: queueTask, markStarted, recordStep, requestCancellation,
    recordCleanup, markCompleted, markCancelled, markFailed, getActivity.
    emptyState() returns ActivityRecord.initial("", null).

- 1 Consumer TaskDispatcher subscribed to TaskQueue events; for each
  TaskSubmitted, calls ActivityEntity.queueTask to register the task, then
  starts an ActivityWorkflow with taskId as the workflow id.

- 1 View ActivityView with row type ActivityRow (mirrors ActivityRecord
  minus the raw ActivityTask description — that remains in the entity for
  detail view). Table updater consumes ActivityEntity events. ONE query
  getAllActivities SELECT * AS activities FROM activity_view.

- 2 TimedActions:
  * TaskPoller — every 20s, reads next line from
    src/main/resources/sample-events/activity-tasks.jsonl and calls
    TaskQueue.submit.
  * StaleActivityReaper — every 5 minutes, queries ActivityView.
    getAllActivities, picks RUNNING tasks with startedAt older than
    STALE_TIMEOUT_SECONDS (default 300), calls
    ActivityEntity.requestCancellation with requestedBy="reaper" and
    reason="timeout".

- 2 HttpEndpoints:
  * ActivityEndpoint at /api with GET /activities, GET /activities/{id},
    POST /activities (body {name, description, kind, estimatedSteps}),
    POST /activities/{id}/cancel (body {requestedBy, reason}),
    GET /activities/sse, and /api/metadata/* endpoints serving YAML/MD
    files from src/main/resources/metadata/.
    Cancel writes CancellationRequested to ActivityEntity.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ActivityTasks.java declaring two Task<R> constants: EXECUTE (StepResult),
  CLEANUP (CleanupPlan).
- Domain records ActivityTask, StepResult, CleanupPlan, CancellationRequest,
  ActivityRecord.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9855 and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/activity-tasks.jsonl with 8 canned task
  lines covering INFRA_PROVISION (3), REPORT_GENERATE (2), DATA_PIPELINE (2),
  MAINTENANCE (1). estimatedSteps varies 3–8.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.
- eval-matrix.yaml at root with 2 controls: H1 halt operator-regulator-stop,
  H2 halt graceful-degradation. Matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at root.
- prompts/task-runner.md, prompts/cleanup.md.
- README.md at root.
- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs. App UI tab shows the live activity board:
  left column = task list with status pill and kind badge; right column =
  selected task detail with step log timeline, Cancel button (active when
  RUNNING/QUEUED), cleanup plan block (visible after CANCELLED), and
  staleReaper badge when cancelled by timeout.
  Browser title exactly:
  <title>Akka Sample: Activity Interrupt / Cancellation</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in
        .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives in session memory; passed
        via MCP tool env parameter; gone when session ends.
- NEVER write the key value to any file.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch.
- task-runner.json — 8–10 StepResult entries. First 1–3 entries for each
  task have terminal=false. One entry per "task type" has terminal=true.
  Summaries are plausible one-liners for each TaskKind (e.g., "Provisioned
  security group sg-0abc for VPC vpc-1234"). partialArtifact present on
  ~half of non-terminal steps.
- cleanup.json — 5–7 CleanupPlan entries. Each has 2–4 steps (imperative
  sentences), a reason sentence, and an optional rollbackNote. Steps are
  domain-plausible (release locks, revert partial writes, notify upstream).
- A MockModelProvider.seedFor(taskId) helper for deterministic selection.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- TaskRunnerAgent checks the entity's CANCELLING status on every step
  boundary — it does NOT rely on the LLM refusing; the workflow enforces it.
- CleanupAgent is always called on cancellation before the terminal event —
  never skipped.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching MUST match by data-tab / data-panel attribute, NEVER by
  NodeList index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block.
- No forbidden words in user-facing text.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
