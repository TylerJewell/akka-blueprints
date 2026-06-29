# SPEC — durable-workflow-recovery

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DurableWorkflowRecovery.
**One-line pitch:** An operator registers a long-running workflow execution; one AI agent reads the execution's checkpoint history (passed as a task attachment, never as inline prompt text) and returns a structured recovery decision — RESUME / ABORT / ESCALATE with a per-checkpoint status and a recommended next action.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `WorkflowRecoveryAgent` (AutonomousAgent) carries the recovery analysis; the surrounding components detect stalls, prepare the checkpoint snapshot, and audit the decision. Two governance mechanisms are wired around the agent:

- A **graceful-degradation halt** runs inside the workflow's recovery step: when a stall is detected, the system resumes from the last durable checkpoint rather than restarting from scratch. The halt is cooperative — the agent inspects the partial state and decides whether resumption is safe before any restart command is issued.
- A **periodic health evaluator** runs on a time-based schedule (every 60 s by default) against all in-flight executions, scoring each one for latency drift, checkpoint progress rate, and retry saturation. Scores below a configurable threshold trigger an automatic escalation path.

The blueprint shows that a single decision-making agent can govern multi-phase operational recovery — two independent checks sit on either side of the one LLM call, and neither silently replaces the other.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in the **Workflow ID** and **Workflow type** fields (or picks one of three seeded examples — an ETL pipeline, a payment-batch processor, a data-migration job).
2. The user clicks **Register execution**. The UI POSTs to `/api/executions` and receives an `executionId`.
3. The card appears in the live list in `REGISTERED` state. Within ~1 s, as checkpoint events arrive (seeded automatically), the card transitions to `RUNNING` and the checkpoint timeline fills in.
4. When the system detects a stall — no checkpoint for longer than the configured stall-timeout — the `CheckpointConsumer` emits a `StallDetected` event and the card transitions to `STALLED`.
5. Within ~10–30 s, the `RecoveryWorkflow` completes its `analyzeStep`. The card transitions to `ANALYZING` then `DECISION_RECORDED`. The decision appears: a top-level verdict badge (RESUME / ABORT / ESCALATE), a short rationale paragraph, and a per-checkpoint status table (checkpoint id, phase, elapsed ms, outcome, recommended action).
6. Within ~5 s, the `healthEvalStep` finishes. The card shows a **health score** chip (1–5) plus a one-line diagnosis covering latency drift and retry saturation.
7. If the decision is RESUME, a `ResumeCommand` is dispatched and the execution returns to RUNNING state. The user can register another execution; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RecoveryEndpoint` | `HttpEndpoint` | `/api/executions/*` — register, list, get, SSE; serves `/api/metadata/*`. | — | `ExecutionEntity`, `ExecutionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ExecutionEntity` | `EventSourcedEntity` | Per-execution lifecycle: registered → running → stalled → analyzing → decision → health-scored. Source of truth. | `RecoveryEndpoint`, `CheckpointConsumer`, `RecoveryWorkflow` | `ExecutionView` |
| `CheckpointConsumer` | `Consumer` | Subscribes to `CheckpointRecorded` events; checks stall condition; calls `ExecutionEntity.markStalled` when threshold exceeded. | `ExecutionEntity` events | `ExecutionEntity` |
| `RecoveryWorkflow` | `Workflow` | One workflow per execution. Steps: `awaitStalledStep` → `analyzeStep` → `healthEvalStep`. | started by `CheckpointConsumer` once stall event lands | `WorkflowRecoveryAgent`, `ExecutionEntity` |
| `WorkflowRecoveryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives execution metadata in the task definition and the checkpoint snapshot as a task attachment; returns `RecoveryDecision`. | invoked by `RecoveryWorkflow` | returns decision |
| `ExecutionView` | `View` | Read model: one row per execution for the UI. | `ExecutionEntity` events | `RecoveryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CheckpointRecord(
    String checkpointId,
    String phase,
    long elapsedMs,
    CheckpointOutcome outcome,
    Instant recordedAt
) {}
enum CheckpointOutcome { COMPLETED, FAILED, SKIPPED, PENDING }

record ExecutionRegistration(
    String executionId,
    String workflowId,
    String workflowType,
    String ownerTeam,
    Instant registeredAt
) {}

record CheckpointSnapshot(
    List<CheckpointRecord> checkpoints,
    int totalExpected,
    int completedCount,
    int failedCount,
    Duration stalledFor
) {}

record CheckpointStatus(
    String checkpointId,
    String phase,
    CheckpointOutcome outcome,
    String recommendedAction
) {}

record RecoveryDecision(
    RecoveryVerdict verdict,
    String rationale,
    List<CheckpointStatus> checkpointStatuses,
    Instant decidedAt
) {}
enum RecoveryVerdict { RESUME, ABORT, ESCALATE }

record HealthScore(
    int score,           // 1..5
    String diagnosis,
    Instant evaluatedAt
) {}

record Execution(
    String executionId,
    Optional<ExecutionRegistration> registration,
    Optional<CheckpointSnapshot> snapshot,
    Optional<RecoveryDecision> decision,
    Optional<HealthScore> health,
    ExecutionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExecutionStatus {
    REGISTERED, RUNNING, STALLED, ANALYZING, DECISION_RECORDED, HEALTH_SCORED, ABORTED, FAILED
}
```

Events on `ExecutionEntity`: `ExecutionRegistered`, `CheckpointRecorded`, `StallDetected`, `AnalysisStarted`, `DecisionRecorded`, `HealthScored`, `ExecutionAborted`, `ExecutionFailed`.

Every nullable lifecycle field on the `Execution` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/executions` — body `{ workflowId, workflowType, ownerTeam }` → `{ executionId }`.
- `GET /api/executions` — list all executions, newest-first.
- `GET /api/executions/{id}` — one execution.
- `GET /api/executions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Durable Workflows with DBOS</title>`.

The App UI tab is a two-column layout: a left rail with the live list of registered executions (status pill + verdict badge + age) and a right pane with the selected execution's detail — checkpoint timeline, snapshot summary, decision rationale, checkpoint status table, and health score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — graceful-degradation halt** (`halt`, applied inside `RecoveryWorkflow.analyzeStep`): when a stall is detected, the workflow snapshots the last durable checkpoint and passes it to `WorkflowRecoveryAgent` as a task attachment. The agent inspects the partial execution state and returns a recovery verdict before any restart command is issued. This avoids the most common failure mode — losing progress by restarting from scratch — and makes the agent the gate on whether resumption is safe. An ABORT verdict suppresses the resume command and transitions the execution to ABORTED.
- **E1 — periodic health eval** (`eval-periodic`, `performance-monitor`): a timer-driven Consumer runs every 60 s and scores all executions in RUNNING or STALLED state. The scorer is deterministic (no LLM call — rule-based so the same snapshot always scores the same): checks latency drift (median checkpoint elapsed vs. baseline), checkpoint progress rate (completed vs. expected per unit time), and retry saturation (failedCount / totalExpected). Emits `HealthScored` with a 1–5 score and a one-line diagnosis. Executions with score ≤ 2 trigger an automatic ESCALATE flag viewable on the UI.

## 9. Agent prompts

- `WorkflowRecoveryAgent` → `prompts/workflow-recovery-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached checkpoint snapshot, assess the stall cause, and return one `CheckpointStatus` per checkpoint plus a top-level recovery verdict.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User registers the ETL-pipeline seed; within 30 s the RESUME decision appears with one checkpoint status per checkpoint and a health score chip.
2. **J2** — An execution that exhausts its retry budget (all checkpoints FAILED) receives an ABORT verdict; the execution transitions to ABORTED and the UI card border turns red.
3. **J3** — An execution with high latency drift receives a health score of 1 with a clear diagnosis; the UI flags the card for review.
4. **J4** — A crash mid-checkpoint is recovered from the last COMPLETED checkpoint; the decision payload's `checkpointStatuses` shows PENDING for the in-progress checkpoint, not FAILED.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named durable-workflow-recovery demonstrating the single-agent × ops-automation
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-durable-workflow-recovery. Java package
io.akka.samples.durableworkflowswithdbos. Akka 3.6.0. HTTP port 9719.

Components to wire (exactly):

- 1 AutonomousAgent WorkflowRecoveryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/workflow-recovery-agent.md>) and
  .capability(TaskAcceptance.of(ANALYZE_EXECUTION).maxIterationsPerTask(3)). The task receives
  execution metadata as its instruction text and the checkpoint snapshot as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: RecoveryDecision{verdict: RecoveryVerdict (RESUME/ABORT/ESCALATE), rationale:
  String, checkpointStatuses: List<CheckpointStatus>, decidedAt: Instant}. The agent does NOT
  have a before-agent-response guardrail for this baseline (structural validation happens
  inside analyzeStep via a lightweight ResponseValidator helper class before the decision is
  written to the entity).

- 1 Workflow RecoveryWorkflow per executionId with three steps:
  * awaitStalledStep — polls ExecutionEntity.getExecution every 1s; on
    execution.snapshot().isPresent() advances to analyzeStep.
    WorkflowSettings.stepTimeout 30s.
  * analyzeStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    WorkflowRecoveryAgent.class, "recovery-" + executionId).runSingleTask(
      TaskDef.instructions(formatMetadata(execution.registration))
        .attachment("checkpoint-snapshot.json",
          serializeSnapshot(execution.snapshot).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANALYZE_EXECUTION) to fetch the
    decision. Runs ResponseValidator.validate(decision, snapshot) before calling
    ExecutionEntity.recordDecision(decision). If validation fails, fails over to error step.
    On success, dispatches a ResumeCommand if verdict == RESUME.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(RecoveryWorkflow::error).
  * healthEvalStep — runs a deterministic rule-based HealthEvaluator (NOT an LLM call) over
    the recorded decision and snapshot: scores latency drift (median checkpoint elapsedMs vs.
    a configurable baseline), checkpoint progress rate (completedCount / totalExpected per
    elapsed wall-clock time), and retry saturation (failedCount / totalExpected). Emits
    HealthScored{score: 1-5, diagnosis: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ExecutionEntity (one per executionId). State Execution{executionId:
  String, registration: Optional<ExecutionRegistration>, snapshot: Optional<CheckpointSnapshot>,
  decision: Optional<RecoveryDecision>, health: Optional<HealthScore>, status: ExecutionStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. ExecutionStatus enum: REGISTERED,
  RUNNING, STALLED, ANALYZING, DECISION_RECORDED, HEALTH_SCORED, ABORTED, FAILED. Events:
  ExecutionRegistered{registration}, CheckpointRecorded{checkpoint}, StallDetected{stalledFor},
  AnalysisStarted{}, DecisionRecorded{decision}, HealthScored{health}, ExecutionAborted{},
  ExecutionFailed{reason}. Commands: register, recordCheckpoint, markStalled, startAnalysis,
  recordDecision, recordHealth, abort, fail, getExecution. emptyState() returns
  Execution.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer CheckpointConsumer subscribed to ExecutionEntity events; on CheckpointRecorded
  checks whether the elapsed time since the last checkpoint exceeds the configured stall
  timeout (default 30 s); if exceeded, builds a CheckpointSnapshot from all checkpoint
  records in the entity state, then calls ExecutionEntity.markStalled(snapshot). After
  markStalled lands, the same Consumer starts a RecoveryWorkflow with id = "recovery-" +
  executionId. Also implements a periodic health tick: on a 60 s timer, fetches all RUNNING
  and STALLED executions from ExecutionView and runs HealthEvaluator.score(snapshot) for
  each, emitting HealthScored if the score changed.

- 1 View ExecutionView with row type ExecutionRow (mirrors Execution minus any internal
  retry counters — the audit log keeps raw event history; the view holds the projected form
  for the UI). Table updater consumes ExecutionEntity events. ONE query getAllExecutions:
  SELECT * AS executions FROM execution_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * RecoveryEndpoint at /api with POST /executions (body
    {workflowId, workflowType, ownerTeam}; mints executionId; calls
    ExecutionEntity.register; returns {executionId}), GET /executions (list from
    getAllExecutions, sorted newest-first), GET /executions/{id} (one row), GET
    /executions/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RecoveryTasks.java declaring one Task<R> constant: ANALYZE_EXECUTION = Task.name("Analyze
  execution").description("Read the checkpoint snapshot and produce a RecoveryDecision per
  checkpoint").resultConformsTo(RecoveryDecision.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records CheckpointRecord, ExecutionRegistration, CheckpointSnapshot, CheckpointStatus,
  RecoveryDecision, RecoveryVerdict, CheckpointOutcome, HealthScore, Execution, ExecutionStatus.

- ResponseValidator.java — validates a candidate RecoveryDecision against the snapshot:
  (1) verdict is a valid RecoveryVerdict enum, (2) checkpointStatuses is non-empty,
  (3) every checkpointStatus.checkpointId references a checkpoint in the snapshot,
  (4) rationale is non-empty. Returns a ValidationResult{valid: boolean, reason: String}.
  Used inside analyzeStep before the decision is committed to the entity.

- HealthEvaluator.java — pure deterministic logic (no LLM). Inputs: CheckpointSnapshot.
  Outputs: HealthScore. Scoring rubric: latency drift subtracts 1 point if median elapsedMs
  is more than 2× the baseline; retry saturation subtracts 1 point per 20% failed ratio;
  zero-progress (completedCount == 0) subtracts 1 point. Documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9719 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The WorkflowRecoveryAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/workflow-definitions.jsonl with 3 seeded workflow types:
  an ETL pipeline (8 phases), a payment-batch processor (5 phases), and a data-migration job
  (10 phases). Each definition includes a stall-timeout, a baseline checkpoint latency,
  and a total expected checkpoint count.

- src/main/resources/sample-events/seed-checkpoints.jsonl with 3 paired checkpoint histories:
  ETL pipeline with checkpoints 1–6 completed and checkpoint 7 stalled, a payment batch with
  all checkpoints FAILED (ABORT path), and a data-migration with mid-stream crash (PENDING
  on last checkpoint, RESUME path). Each history has realistic elapsedMs values.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = enforce (the ABORT verdict stops the execution),
  oversight.human_in_loop = false (recovery is fully automated; human escalation is the
  ESCALATE path), failure.failure_modes including "false-resume-on-corrupted-state",
  "missed-stall-detection", "abort-on-recoverable-error", "latency-miscalibration";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/workflow-recovery-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Durable Workflows with DBOS",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of execution cards; right = selected-execution detail with checkpoint timeline,
  snapshot summary, decision rationale, checkpoint status table, and health score chip).
  Browser title exactly: <title>Akka Sample: Durable Workflows with DBOS</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(executionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyze-execution.json — 8 RecoveryDecision entries covering all three RecoveryVerdict
      values. Each entry has a rationale paragraph and a checkpointStatuses array with one
      CheckpointStatus per checkpoint in the matched workflow type (ETL/payment-batch/
      data-migration). Each status has a non-empty checkpointId, a phase, a CheckpointOutcome,
      and a recommendedAction. Plus 2 deliberately MALFORMED entries (one with a checkpointId
      that does not exist in the snapshot; one with an empty rationale string) — ResponseValidator
      catches both, exercising the failover path.
    The mock should select a malformed entry on the FIRST iteration of every 3rd execution
    (modulo seed) so J2's ABORT path is reproducible.
- A MockModelProvider.seedFor(executionId) helper makes per-execution selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WorkflowRecoveryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RecoveryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (analyzeStep
  60s, awaitStalledStep 30s, healthEvalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Execution row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: RecoveryTasks.java with ANALYZE_EXECUTION = Task.name(...).description(...)
  .resultConformsTo(RecoveryDecision.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9719 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WorkflowRecoveryAgent).
  The health evaluator is rule-based (HealthEvaluator.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The checkpoint snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated analyzeStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The graceful-degradation halt is wired as an explicit check inside analyzeStep, not as an
  external check that runs after the agent returns. The agent sees the partial checkpoint
  state and decides; only after a RESUME verdict is the restart command dispatched.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
