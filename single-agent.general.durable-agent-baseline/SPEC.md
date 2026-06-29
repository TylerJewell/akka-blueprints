# SPEC — durable-agent-baseline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DurableAgentBaseline.
**One-line pitch:** A user submits a multi-step work order; one AI agent processes every step in sequence, recording progress as it goes, so that a JVM restart leaves the work order running from its last completed step rather than from the beginning.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain, with the focus on durable execution. One `WorkOrderAgent` (AutonomousAgent) carries all decision-making; the surrounding Workflow, Entity, and Consumer components manage durability, monitoring, and evaluation.

Two governance mechanisms are wired around the agent:

- A **hotl runtime monitor** runs inside a Consumer that subscribes to entity events. It tracks the elapsed time between step-started and step-completed events, detects stalls when that window exceeds the configured threshold, and records resource-consumption signals (iteration count, token estimates). A stall or resource alert is written back to the entity so the UI and on-call tooling see it without polling the agent loop.
- A **periodic performance evaluator** runs inside `PeriodicEvalStep` within the workflow, firing after every work order completes. It is deterministic and rule-based (no LLM call) — scoring latency-to-completion, per-step retry count, and stall-event density. The score feeds an aggregate view that the Eval Matrix tab renders.

The blueprint shows that durable execution does not require an external orchestrator. The Akka Workflow primitive journals its own step position; the EventSourcedEntity journals every step transition. Together, a restart leaves the system in a fully recoverable state.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **work-order template** from a dropdown (Data Pipeline, Code Analysis, Report Assembly) or enters a custom set of step descriptions.
2. The user fills in a **work-order title** and clicks **Submit**. The UI POSTs to `/api/work-orders` and receives a `workOrderId`.
3. The new card appears in the live list with status `INITIATED`. Within ~1 s it transitions to `RUNNING` as the workflow starts the agent.
4. The right-pane detail shows the steps list. As the agent completes each step, the step row updates: a green check, elapsed time, and the agent's output for that step. The UI receives these transitions over SSE.
5. When all steps finish, the card reaches `COMPLETED`. The result panel shows the final `WorkOrderResult` — a summary paragraph and a per-step outcome table (step id, status, output, elapsed).
6. Within ~1 s the `PeriodicEvalStep` fires. The card shows a **performance score** chip (1–5) and a one-line rationale summarising the run's latency and retry behaviour.
7. If the user restarts the service mid-run, the in-flight work order resumes from its last journaled step. The UI reconnects via SSE and continues updating.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WorkOrderEndpoint` | `HttpEndpoint` | `/api/work-orders/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `WorkOrderEntity`, `WorkOrderView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `WorkOrderEntity` | `EventSourcedEntity` | Per-work-order audit trail: initiated → running → each step transition → completed or failed. | `WorkOrderEndpoint`, `RuntimeMonitor`, `WorkOrderWorkflow` | `WorkOrderView` |
| `RuntimeMonitor` | `Consumer` | Subscribes to `WorkOrderEntity` events; detects stalls and resource over-runs; calls `WorkOrderEntity.recordAlert`. | `WorkOrderEntity` events | `WorkOrderEntity` |
| `WorkOrderWorkflow` | `Workflow` | One workflow per work order. Steps: `initStep` → `agentStep` → `evalStep`. Journals position between JVM restarts. | started by `WorkOrderEndpoint` on submit | `WorkOrderAgent`, `WorkOrderEntity` |
| `WorkOrderAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the work order's steps as task instructions and processes them one by one, returning a `WorkOrderResult`. | invoked by `WorkOrderWorkflow` | returns result |
| `PerformanceEvaluator` | (supporting class) | Deterministic rule-based scorer called inside `evalStep`. No LLM call. Inputs: `WorkOrderResult` + entity event history. Outputs: `EvalScore`. | `WorkOrderWorkflow.evalStep` | `WorkOrderEntity` |
| `WorkOrderView` | `View` | Read model: one row per work order for the UI. | `WorkOrderEntity` events | `WorkOrderEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WorkStep(
    String stepId,
    String description,
    int maxRetries
) {}

record WorkOrder(
    String workOrderId,
    String title,
    List<WorkStep> steps,
    String submittedBy,
    Instant submittedAt
) {}

record StepOutcome(
    String stepId,
    StepStatus status,
    String output,
    int retryCount,
    Instant startedAt,
    Instant finishedAt
) {}
enum StepStatus { PENDING, RUNNING, COMPLETED, FAILED }

record WorkOrderResult(
    WorkOrderResolution resolution,
    String summary,
    List<StepOutcome> outcomes,
    Instant decidedAt
) {}
enum WorkOrderResolution { COMPLETED, PARTIAL, FAILED }

record EvalScore(
    int score,              // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record AlertEvent(
    AlertKind kind,
    String stepId,
    String detail,
    Instant detectedAt
) {}
enum AlertKind { STALL, RESOURCE_OVER_RUN }

record WorkOrderRun(
    String workOrderId,
    Optional<WorkOrder> order,
    Optional<WorkOrderResult> result,
    Optional<EvalScore> eval,
    List<StepOutcome> stepOutcomes,
    List<AlertEvent> alerts,
    WorkOrderStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum WorkOrderStatus {
    INITIATED, RUNNING, STEP_COMPLETED, COMPLETED, EVALUATED, STALLED, FAILED
}
```

Events on `WorkOrderEntity`: `WorkOrderInitiated`, `WorkOrderStarted`, `StepStarted`, `StepCompleted`, `StepFailed`, `WorkOrderCompleted`, `WorkOrderFailed`, `AlertRecorded`, `PerformanceEvaluated`.

Every nullable lifecycle field on the `WorkOrderRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/work-orders` — body `{ title, steps: [WorkStep], submittedBy }` → `{ workOrderId }`.
- `GET /api/work-orders` — list all work orders, newest-first.
- `GET /api/work-orders/{id}` — one work order.
- `GET /api/work-orders/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Durable Agent Baseline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of work orders (status pill + resolution badge + age) and a right pane showing the selected work order's step-by-step progress, alert badges, final result summary, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **M1 — hotl runtime monitor** (`hotl`, `deployer-runtime-monitoring`, applied inside `RuntimeMonitor` Consumer): subscribes to `StepStarted` events on the entity; starts a stall window; if the matching `StepCompleted` or `StepFailed` event does not arrive within the configured threshold (default 90 s per step), writes an `AlertRecorded` event with `kind = STALL`. Also counts total iterations from the entity event log and writes a `RESOURCE_OVER_RUN` alert when iteration count exceeds `maxIterations` for a step. The alert is non-blocking — the agent continues running; the UI surfaces the alert.
- **E1 — periodic performance evaluator** (`eval-periodic`, `performance-monitor`): runs as `PeriodicEvalStep` inside the workflow, fired immediately after every `WorkOrderCompleted` event. The evaluator is deterministic and rule-based (no LLM call — keeping the single-agent invariant honest). It scores: latency (time from `INITIATED` to `COMPLETED` vs. expected baseline per step count), retry density (steps that required more than zero retries), and stall-event count. Emits `PerformanceEvaluated` with a 1–5 score. Score ≤ 2 flags the run for human review.

## 9. Agent prompts

- `WorkOrderAgent` → `prompts/work-order-agent.md`. The single decision-making LLM. System prompt instructs it to process each step from the work order in order, produce a `StepOutcome` per step, and assemble a final `WorkOrderResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Data Pipeline seed order; within 60 s the work order reaches `EVALUATED` with one `StepOutcome` per submitted step and an eval score chip.
2. **J2** — Service is restarted while a work order is `RUNNING`; after restart the workflow resumes from its last journaled step and the work order reaches `EVALUATED` without re-running completed steps.
3. **J3** — A work order whose agent iteration exceeds the stall threshold receives an `AlertRecorded` event with `kind = STALL`; the alert badge appears on the UI card.
4. **J4** — A work order where every step requires a retry receives an eval score of ≤ 2 with a clear rationale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named durable-agent-baseline demonstrating the single-agent × general cell
with a focus on durable execution. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact single-agent-general-durable-agent-baseline. Java package
io.akka.samples.durableagents. Akka 3.6.0. HTTP port 9629.

Components to wire (exactly):

- 1 AutonomousAgent WorkOrderAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/work-order-agent.md>) and
  .capability(TaskAcceptance.of(PROCESS_WORK_ORDER).maxIterationsPerTask(5)). The task
  receives the work order's steps as its instruction text. Output: WorkOrderResult{
  resolution: WorkOrderResolution (COMPLETED/PARTIAL/FAILED), summary: String,
  outcomes: List<StepOutcome>, decidedAt: Instant}. Configured with maxIterationsPerTask(5)
  to allow step-level retries within a single task run.

- 1 Workflow WorkOrderWorkflow per workOrderId with three steps:
  * initStep — calls WorkOrderEntity.markRunning; transitions to agentStep immediately.
    WorkflowSettings.stepTimeout 10s.
  * agentStep — calls componentClient.forAutonomousAgent(WorkOrderAgent.class,
    "agent-" + workOrderId).runSingleTask(
      TaskDef.instructions(formatSteps(order.steps()))
    ) — returns a taskId, then forTask(taskId).result(PROCESS_WORK_ORDER) to fetch the
    result. On success calls WorkOrderEntity.recordResult(result). WorkflowSettings
    .stepTimeout 120s with defaultStepRecovery maxRetries(2).failoverTo(
    WorkOrderWorkflow::error).
  * evalStep — runs a deterministic PerformanceEvaluator (NOT an LLM call) over the
    completed WorkOrderRun: scores latency vs. expected, retry density, and stall-event
    count. Emits PerformanceEvaluated{score: 1-5, rationale: String}. WorkflowSettings
    .stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity WorkOrderEntity (one per workOrderId). State WorkOrderRun{
  workOrderId: String, order: Optional<WorkOrder>, result: Optional<WorkOrderResult>,
  eval: Optional<EvalScore>, stepOutcomes: List<StepOutcome>, alerts: List<AlertEvent>,
  status: WorkOrderStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  WorkOrderStatus enum: INITIATED, RUNNING, STEP_COMPLETED, COMPLETED, EVALUATED,
  STALLED, FAILED. Events: WorkOrderInitiated{order}, WorkOrderStarted{},
  StepStarted{stepId}, StepCompleted{stepId, outcome}, StepFailed{stepId, reason},
  WorkOrderCompleted{result}, WorkOrderFailed{reason}, AlertRecorded{alert},
  PerformanceEvaluated{eval}. Commands: initiate, markRunning, recordStepStart,
  recordStepComplete, recordStepFail, recordResult, recordAlert, recordEvaluation,
  fail, getWorkOrder. emptyState() returns WorkOrderRun.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer RuntimeMonitor subscribed to WorkOrderEntity events; on StepStarted
  records the start time keyed by (workOrderId, stepId); on StepCompleted or StepFailed
  clears the window; on a periodic tick (every 30 s, implemented as a self-scheduled
  check using the consumer's own timer capability) compares now() against open windows —
  any window older than 90 s emits a STALL alert via WorkOrderEntity.recordAlert.
  Also tracks total event count per workOrderId; when event count for a single work order
  exceeds 50, emits a RESOURCE_OVER_RUN alert. Because the Consumer is event-driven
  and does not hold durable per-window timers between restarts, on restart it re-scans
  the entity's open windows from the event log via WorkOrderEntity.getWorkOrder before
  committing to the stall threshold check.

- 1 View WorkOrderView with row type WorkOrderRow (mirrors WorkOrderRun). Table updater
  consumes WorkOrderEntity events. ONE query getAllWorkOrders: SELECT * AS workOrders FROM
  work_order_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * WorkOrderEndpoint at /api with POST /work-orders (body
    {title, steps: [{stepId, description, maxRetries}], submittedBy}; mints workOrderId;
    calls WorkOrderEntity.initiate; starts WorkOrderWorkflow; returns {workOrderId}), GET
    /work-orders (list from getAllWorkOrders, sorted newest-first), GET /work-orders/{id}
    (one row), GET /work-orders/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- WorkOrderTasks.java declaring one Task<R> constant: PROCESS_WORK_ORDER = Task
  .name("Process work order").description("Execute all steps in the work order and
  produce a WorkOrderResult").resultConformsTo(WorkOrderResult.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records WorkStep, WorkOrder, StepOutcome, StepStatus, WorkOrderResult,
  WorkOrderResolution, EvalScore, AlertEvent, AlertKind, WorkOrderRun, WorkOrderStatus.

- PerformanceEvaluator.java — pure deterministic logic (no LLM). Inputs: WorkOrderRun
  (including stepOutcomes, alerts, timestamps). Outputs: EvalScore. Scoring rubric:
  +2 if latency-per-step ≤ 15 s on average; +1 if no step required more than one retry;
  +1 if no stall alerts; +1 if resolution == COMPLETED. Minimum score 1.
  Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9629 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  WorkOrderAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern from
  the akka-context docs.

- src/main/resources/sample-events/work-orders.jsonl with 3 seeded work orders:
  a 4-step Data Pipeline order, a 5-step Code Analysis order, and a 3-step Report
  Assembly order. Steps are described in plain English so the mock LLM can produce
  shape-correct outputs without domain knowledge.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (M1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors —
  durable execution is the use case, not a regulated domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (work orders
  are operational data), decisions.authority_level = autonomous (the agent executes steps
  autonomously without per-step human approval), oversight.human_on_loop = true (a human
  monitors the run via the UI), failure.failure_modes including "stalled-step",
  "missed-step", "partial-completion", "resource-over-run"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/work-order-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Durable Agent Baseline",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of work-order cards; right = selected work order detail with step
  list, alert badges, result summary, and eval-score chip). Browser title exactly:
  <title>Akka Sample: Durable Agent Baseline</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with dispatch
  on the Task<R> id. Each branch reads src/main/resources/mock-responses/<task-id>.json,
  picks one entry pseudo-randomly per call (seedFor(workOrderId)), and deserialises into
  the task's typed return.
- Per-task mock-response shapes:
    process-work-order.json — 8 WorkOrderResult entries covering all three
      WorkOrderResolution values. Each entry has a summary paragraph and an outcomes
      list with one StepOutcome per step in the matched order (Data Pipeline / Code
      Analysis / Report Assembly). Each StepOutcome has a non-empty output string, an
      elapsed time, and a realistic retryCount (0–2). Plus 2 entries where resolution ==
      PARTIAL (some steps FAILED) to exercise the partial-completion path. The mock
      selects a PARTIAL result on every 4th call (modulo seed) so J4 is reproducible.
- A MockModelProvider.seedFor(workOrderId) helper makes per-work-order selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WorkOrderAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion WorkOrderTasks.java MUST
  exist.
- Lesson 4: every workflow step has an explicit stepTimeout (initStep 10s, agentStep 120s,
  evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on WorkOrderRun is Optional<T>.
- Lesson 7: WorkOrderTasks.java with PROCESS_WORK_ORDER is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9629 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (WorkOrderAgent). The
  PerformanceEvaluator is rule-based and does NOT make an LLM call.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
