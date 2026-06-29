# SPEC — flexibility-orchestrator

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Demand-Flexibility Orchestrator.
**One-line pitch:** A background worker monitors grid demand signals, evaluates curtailment needs with an AI agent, and dispatches flexibility programs — requiring operator approval above a configurable threshold and enforcing a hard budget cap per program window.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of a single AI primitive (`DispatchEvaluatorAgent`). Specifically:

- A **HITL approval gate** holds every dispatch request whose recommended curtailment exceeds a configured MW threshold. The workflow pauses; only an authenticated operator can advance it.
- A **halt/budget-cap** guard runs before any dispatch is forwarded to the HITL gate. If the requested curtailment would cause the program's cumulative window consumption to exceed the cap, the dispatch is immediately halted with `BUDGET_EXCEEDED` — no operator prompt, no AI retry.

Together they produce a system where high-impact grid interventions require human sign-off and where runaway AI recommendations cannot exceed pre-committed energy budgets.

## 3. User-facing flows

The operator opens the App UI tab.

1. The system shows the live dispatch board: every incoming demand signal, its evaluated recommendation, and the current program budget gauge.
2. `GridSignalPoller` (TimedAction) ticks every 60 s and inserts simulated demand readings into `DemandSignalEntity`. (A `RequestSimulator` style — drips canned readings.)
3. For each ELEVATED or CRITICAL signal, `DispatchWorkflow` is started. It calls `DispatchEvaluatorAgent` to produce a `DispatchRecommendation`.
4. Before presenting to the operator, the workflow checks the program's remaining budget via `FlexibilityProgramEntity`. If the recommended curtailment exceeds the remaining budget, the dispatch is halted (`BUDGET_EXCEEDED`).
5. If the recommended curtailment is above the operator-approval threshold, the dispatch transitions to `AWAITING_APPROVAL`. The operator clicks Approve or Cancel in the UI.
6. If below the threshold (or after Approve), the dispatch transitions to `EXECUTING` then `COMPLETED`.
7. `EvalRunner` (TimedAction) ticks every 30 minutes, picks N completed dispatches without an `evalScore`, calls an evaluator, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GridSignalPoller` | `TimedAction` | Drips simulated grid demand readings every 60 s. | scheduler | `DemandSignalEntity` |
| `DemandSignalEntity` | `EventSourcedEntity` | Append-only log of `DemandSignalReceived` events. | `GridSignalPoller`, `FlexibilityEndpoint` | `DispatchWorkflow` |
| `DispatchEvaluatorAgent` | `Agent` (typed, not autonomous) | Analyzes a demand reading and returns a `DispatchRecommendation`. | invoked by Workflow | returns DispatchRecommendation |
| `DispatchWorkflow` | `Workflow` | Per-signal orchestration: evaluate → budget check → (conditional) await approval → execute → complete. | `DemandSignalEntity` (on ELEVATED/CRITICAL) | `DispatchEntity`, `FlexibilityProgramEntity` |
| `DispatchEntity` | `EventSourcedEntity` | Lifecycle per dispatch request: requested → awaiting_approval → approved/cancelled → executing → completed/budget_exceeded. | `DispatchWorkflow` | `DispatchView` |
| `FlexibilityProgramEntity` | `EventSourcedEntity` | Per-program budget tracking and window status. | `DispatchWorkflow`, `FlexibilityEndpoint` | `DispatchView` |
| `PricingConsumer` | `Consumer` | Subscribes to `DispatchApproved` events; calculates and records pricing adjustments. | `DispatchEntity` events | `FlexibilityProgramEntity` |
| `DispatchView` | `View` | Read-model row per dispatch for the operator dashboard. | `DispatchEntity` events | `FlexibilityEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, samples completed dispatches without scores; calls evaluator; writes `EvalScored`. | scheduler | `DispatchEntity` |
| `FlexibilityEndpoint` | `HttpEndpoint` | `/api/dispatches/*` and `/api/programs/*` — list, get, approve, cancel, SSE. | — | `DispatchView`, `DispatchEntity`, `FlexibilityProgramEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record GridDemandSignal(
    String signalId,
    double demandMW,
    double forecastMW,
    double gridFrequencyHz,
    SignalSeverity severity,
    Instant timestampUtc
) {}

enum SignalSeverity { NORMAL, ELEVATED, CRITICAL }

record DispatchRecommendation(
    double curtailmentMW,
    double pricingDeltaPerMWh,
    String urgency,    // "routine" | "urgent" | "emergency"
    String rationale
) {}

record DispatchRequest(
    String requestId,
    String programId,
    String signalId,
    double curtailmentMW,
    double pricingDeltaPerMWh,
    Instant windowStart,
    Instant windowEnd
) {}

record OperatorDecision(
    boolean approved,
    String decidedBy,
    Optional<String> reason,
    Instant decidedAt
) {}

record PricingAdjustment(
    String requestId,
    String programId,
    double pricingDeltaPerMWh,
    Instant appliedAt
) {}

record FlexibilityProgram(
    String programId,
    String name,
    double maxCurtailmentMW,
    double budgetUsedMW,
    Instant windowStart,
    Instant windowEnd,
    ProgramStatus programStatus
) {}

enum ProgramStatus { ACTIVE, SUSPENDED, WINDOW_CLOSED }

record DispatchRecord(
    String requestId,
    String programId,
    String signalId,
    Optional<DispatchRecommendation> recommendation,
    Optional<OperatorDecision> decision,
    double curtailmentMW,
    double pricingDeltaPerMWh,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    DispatchStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DispatchStatus {
    REQUESTED, EVALUATING, AWAITING_APPROVAL, APPROVED, CANCELLED,
    EXECUTING, COMPLETED, BUDGET_EXCEEDED
}
```

Events on `DispatchEntity`: `DispatchRequested`, `DispatchEvaluated`, `DispatchAwaitingApproval`, `DispatchApproved`, `DispatchCancelled`, `DispatchExecuting`, `DispatchCompleted`, `BudgetCapExceeded`, `EvalScored`.

Events on `DemandSignalEntity`: `DemandSignalReceived`.

Events on `FlexibilityProgramEntity`: `ProgramCreated`, `BudgetDebited`, `BudgetReleased`, `PricingAdjustmentApplied`, `WindowClosed`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/dispatches` — list all dispatches. Optional `?status=…&programId=…`.
- `GET /api/dispatches/{id}` — one dispatch record.
- `POST /api/dispatches/{id}/approve` — body `{ decidedBy }` → transitions AWAITING_APPROVAL to APPROVED then EXECUTING.
- `POST /api/dispatches/{id}/cancel` — body `{ decidedBy, reason }` → transitions to CANCELLED.
- `GET /api/dispatches/sse` — Server-Sent Events for every dispatch state change.
- `GET /api/programs` — list all flexibility programs with budget status.
- `GET /api/programs/{id}` — one program record.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Demand-Flexibility Orchestrator</title>`.

App UI tab is the most distinctive: it shows the **live dispatch board** with status-column cards, a program budget gauge bar, and Approve/Cancel buttons on each card in `AWAITING_APPROVAL`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL approval gate** (`hitl`, `application`, wired in `DispatchWorkflow`): any dispatch whose `curtailmentMW` exceeds the configured threshold (default 50 MW) pauses in `AWAITING_APPROVAL`. Only `FlexibilityEndpoint`'s approve/cancel endpoints can advance it.
- **B1 — Halt budget-cap** (`halt`, `budget-cap`, wired in `DispatchWorkflow` before the HITL step): queries `FlexibilityProgramEntity` for `budgetUsedMW + curtailmentMW > maxCurtailmentMW`. If true, emits `BudgetCapExceeded` and ends the workflow without ever asking the operator.

## 9. Agent prompts

- `DispatchEvaluatorAgent` → `prompts/dispatch-evaluator.md`. Typed evaluator. Returns a `DispatchRecommendation` with `curtailmentMW`, `pricingDeltaPerMWh`, `urgency`, and a one-sentence `rationale`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an ELEVATED signal; within 60 s a dispatch enters `AWAITING_APPROVAL` with an AI recommendation visible.
2. **J2** — Operator clicks Approve; dispatch transitions to EXECUTING then COMPLETED.
3. **J3** — A signal that would exceed the program budget is halted at `BUDGET_EXCEEDED` before it reaches the operator.
4. **J4** — Operator cancels a pending dispatch; the program budget gauge does not change.
5. **J5** — EvalRunner scores at least one COMPLETED dispatch within 30 minutes; score and rationale appear in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named flexibility-orchestrator demonstrating the continuous-monitor × energy-utilities
cell. Runs out of the box (in-memory signal simulator; no real SCADA/ISO integration). Maven group
io.akka.samples. Artifact id continuous-monitor-energy-utilities-flexibility-orchestrator.
Java package io.akka.samples.demandflexibilityorchestrator. HTTP port 9793.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) DispatchEvaluatorAgent — evaluator. System prompt loaded from
  prompts/dispatch-evaluator.md. Input: GridDemandSignal{signalId, demandMW, forecastMW,
  gridFrequencyHz, severity, timestampUtc} + FlexibilityProgram{programId, name, maxCurtailmentMW,
  budgetUsedMW, windowStart, windowEnd, programStatus}. Output: DispatchRecommendation{curtailmentMW:
  double, pricingDeltaPerMWh: double, urgency: "routine"|"urgent"|"emergency", rationale: String}.
  Defaults to urgency="urgent" under uncertainty.

- 1 Workflow DispatchWorkflow per ELEVATED/CRITICAL signal with steps:
  evaluateStep -> budgetCheckStep -> conditional branch:
    budget_exceeded -> ends with BudgetCapExceeded (no operator prompt);
    below_threshold -> executeStep -> completeStep;
    above_threshold -> awaitApprovalStep -> conditional:
      approved -> executeStep -> completeStep;
      cancelled -> ends with DispatchCancelled.
  evaluateStep wraps DispatchEvaluatorAgent call with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(30)). budgetCheckStep reads FlexibilityProgramEntity
  synchronously (no timeout needed; in-memory). awaitApprovalStep polls DispatchEntity
  every 5 s; on decision.isPresent() advances. No auto-timeout on awaitApproval — dispatches
  wait indefinitely. executeStep emits DispatchExecuting; completeStep emits DispatchCompleted
  and debits FlexibilityProgramEntity budget.

- 3 EventSourcedEntities:
  * DemandSignalEntity — append-only log of DemandSignalReceived events. Command
    receive(GridDemandSignal) emits DemandSignalReceived{signal}. One entity (singleton key
    "grid"). Emits a workflow-start side effect for every ELEVATED/CRITICAL signal.
  * DispatchEntity (one per requestId) — full per-dispatch lifecycle. State
    DispatchRecord{requestId, programId, signalId, Optional<DispatchRecommendation>
    recommendation, Optional<OperatorDecision> decision, curtailmentMW, pricingDeltaPerMWh,
    Optional<Integer> evalScore, Optional<String> evalRationale, DispatchStatus status,
    Instant createdAt, Optional<Instant> finishedAt}. DispatchStatus enum: REQUESTED,
    EVALUATING, AWAITING_APPROVAL, APPROVED, CANCELLED, EXECUTING, COMPLETED,
    BUDGET_EXCEEDED. Events: DispatchRequested, DispatchEvaluated, DispatchAwaitingApproval,
    DispatchApproved, DispatchCancelled, DispatchExecuting, DispatchCompleted,
    BudgetCapExceeded, EvalScored. Commands: requestDispatch, attachRecommendation,
    markAwaitingApproval, recordApproval, recordCancellation, markExecuting, markCompleted,
    markBudgetExceeded, recordEval, getDispatch. emptyState() returns DispatchRecord.initial()
    without commandContext() reference.
  * FlexibilityProgramEntity (one per programId) — budget and window. State
    FlexibilityProgram{programId, name, maxCurtailmentMW, budgetUsedMW, windowStart,
    windowEnd, programStatus: ProgramStatus}. ProgramStatus enum: ACTIVE, SUSPENDED,
    WINDOW_CLOSED. Events: ProgramCreated, BudgetDebited, BudgetReleased, PricingAdjustmentApplied,
    WindowClosed. Commands: createProgram, debitBudget, releaseBudget, applyPricingAdjustment,
    closeWindow, getProgram.

- 1 Consumer PricingConsumer subscribed to DispatchEntity events; for each DispatchApproved,
  calculates the pricing adjustment (curtailmentMW × pricingDeltaPerMWh), builds
  PricingAdjustment, and calls FlexibilityProgramEntity.applyPricingAdjustment.

- 1 View DispatchView with row type DispatchRow (mirrors DispatchRecord). Table updater
  consumes DispatchEntity events. ONE query getAllDispatches SELECT * AS dispatches FROM
  dispatch_view. No WHERE filter — caller filters client-side.

- 2 TimedActions:
  * GridSignalPoller — every 60 s, reads next line from src/main/resources/sample-events/
    demand-signals.jsonl and calls DemandSignalEntity.receive.
  * EvalRunner — every 30 minutes, queries DispatchView.getAllDispatches, picks up to 5 COMPLETED
    dispatches without an evalScore (oldest-first), scores each (LLM or mock), then calls
    DispatchEntity.recordEval(score, rationale) per dispatch.

- 2 HttpEndpoints:
  * FlexibilityEndpoint at /api with GET /dispatches, GET /dispatches/{id},
    POST /dispatches/{id}/approve (body {decidedBy}),
    POST /dispatches/{id}/cancel (body {decidedBy, reason}),
    GET /dispatches/sse,
    GET /programs, GET /programs/{id},
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Approve writes DispatchApproved → workflow resumes;
    cancel writes DispatchCancelled → workflow ends.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DispatchTasks.java declaring two Task<R> constants: EVALUATE (DispatchRecommendation),
  SCORE (EvalResult).
- Domain records GridDemandSignal, DispatchRecommendation, DispatchRequest, OperatorDecision,
  PricingAdjustment, FlexibilityProgram, DispatchRecord.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9793 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/demand-signals.jsonl with 10 canned signal lines
  covering NORMAL (skipped by workflow), ELEVATED (dispatched with AI evaluation), and
  CRITICAL (dispatched at emergency urgency) severities across two simulated programIds.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: H1 hitl application, B1 halt budget-cap.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filled for energy-utilities sector with
  decisions.authority_level = dispatch-with-approval, oversight.human_in_loop = true,
  oversight.reviewer_must_approve_high_impact = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/dispatch-evaluator.md loaded as the agent system prompt.
- README.md at the project root: title "Akka Sample: Demand-Flexibility Orchestrator",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live dispatch board sorted by createdAt desc, with status pills, urgency chips, and
  MW values; right = selected dispatch detail with recommendation rationale, program budget
  gauge, and Approve/Cancel buttons for AWAITING_APPROVAL dispatches). Browser title
  exactly: <title>Akka Sample: Demand-Flexibility Orchestrator</title>. No subtitle on the
  Overview tab.

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
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    dispatch-evaluator.json — 10–12 DispatchRecommendation entries spanning:
      routine low-curtailment (5–15 MW, pricing delta 2–5 $/MWh, urgency "routine"),
      elevated mid-curtailment (20–45 MW, pricing delta 8–15 $/MWh, urgency "urgent"),
      critical high-curtailment (55–90 MW, pricing delta 20–40 $/MWh, urgency "emergency").
      Each entry has a one-sentence rationale citing frequency deviation or demand surplus.
      The mock should skew toward urgency="urgent" under ambiguous severity.
    eval-score.json — 6–8 EvalResult entries with score 1–5 and one-sentence rationales
      matching the rubric (curtailment accuracy, pricing appropriateness, urgency calibration).
- A `MockModelProvider.seedFor(signalId)` helper makes per-signal selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- DispatchEvaluatorAgent is typed (Agent), NOT AutonomousAgent.
- The budget-cap check runs INSIDE DispatchWorkflow before the HITL step — not inside the
  agent's prompt and not after the operator has already been asked.
- The operator NEVER executes a dispatch directly; only the workflow can emit DispatchExecuting.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
