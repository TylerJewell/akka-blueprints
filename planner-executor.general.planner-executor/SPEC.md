# SPEC — planner-executor-general-planner-executor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** PlanAgents.
**One-line pitch:** Submit a goal; a Planner agent breaks it into a dynamic ordered plan, an Executor agent works through each step, and the Planner revises the plan whenever a step fails or produces unexpected output.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern at its most direct: two agents paired in a tight feedback loop. The Planner owns a **plan ledger** (goal text, list of open steps, list of completed steps, list of blocked steps, and the current active step). The Executor owns no persistent state — it receives one step at a time and returns a `StepResult`. After each `StepResult` is recorded the Planner reads the full ledger and decides: `CONTINUE` (the next step), `REPLAN` (produce a revised step list), `COMPLETE`, or `FAIL`.

The blueprint demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each step before the Executor receives it, checking the step text against a content policy (no destructive commands, no writes outside `/workspace/`, no outbound calls to non-allow-listed hosts),
- an **automatic safety halt** that trips when the replan-cycle budget is exhausted or when the Executor returns an unsafe-action signal, stopping the workflow before further harm can occur.

## 3. User-facing flows

The user opens the App UI tab and submits a goal via the form.

1. The system creates a `Plan` record in `PLANNING` and starts a `PlanWorkflow`.
2. The Planner produces a `PlanLedger { goal, openSteps, completedSteps, blockedSteps, activeStep }` and emits `PlanCreated`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads the full ledger and proposes a `StepDecision { step, rationale }`.
   - The **before-tool-call guardrail** vets the decision; on rejection the workflow records a `StepBlocked` entry and asks the Planner to revise.
   - The Executor runs the step and returns a typed `StepResult`.
   - The workflow appends a `StepEntry { step, attempt, verdict, result, recordedAt }` to the step log.
   - The **automatic safety halt** evaluator inspects the new entry; on an unsafe-action signal it emits `PlanHaltedAutomatic` and the workflow ends.
4. The Planner decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After two consecutive `REPLAN` outputs without a `CONTINUE` in between, the Planner is forced to `FAIL`.
5. On `COMPLETE`, the Planner produces a `PlanOutcome { summary, completedSteps }` and emits `PlanCompleted`. The Plan moves to `COMPLETED`.
6. The operator can press **Halt new steps** at any time. The workflow finishes the in-flight step, then ends with `PlanHaltedOperator`. The Plan moves to `HALTED`.

A `GoalSimulator` (TimedAction) drips a sample goal every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Produces a plan ledger on first call; on subsequent calls reads the step log and decides the next action. Emits `PlanOutcome` on completion. | `PlanWorkflow` | returns typed result to workflow |
| `ExecutorAgent` | `AutonomousAgent` | Receives one step text, executes it against seeded fixtures, returns a typed `StepResult`. | `PlanWorkflow` | — |
| `PlanWorkflow` | `Workflow` | Drives the plan → check-halt → propose → guardrail → execute → record → auto-halt-eval → decide loop, plus replan and halt branches. | `PlanEndpoint`, `GoalRequestConsumer` | `PlanEntity` |
| `PlanEntity` | `EventSourcedEntity` | Holds the plan lifecycle, plan ledger, step log, and final outcome. | `PlanWorkflow` | `PlanView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `PlanEndpoint` (operator action) | `PlanWorkflow` (polls) |
| `GoalQueue` | `EventSourcedEntity` | Audit log of submitted goals. | `PlanEndpoint`, `GoalSimulator` | `GoalRequestConsumer` |
| `PlanView` | `View` | List-of-plans read model for the UI. | `PlanEntity` events | `PlanEndpoint` |
| `GoalRequestConsumer` | `Consumer` | Subscribes to `GoalQueue` events; starts a `PlanWorkflow` per submission. | `GoalQueue` events | `PlanWorkflow` |
| `GoalSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/goal-prompts.jsonl` and enqueues it. | scheduler | `GoalQueue` |
| `StuckPlanMonitor` | `TimedAction` | Every 30 s, marks any plan stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `PlanEntity` |
| `PlanEndpoint` | `HttpEndpoint` | `/api/plans/*` — submit, get, list, SSE, operator halt. | — | `PlanView`, `GoalQueue`, `PlanEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record GoalRequest(String goal, String requestedBy) {}

record PlanLedger(
    String goal,
    List<String> openSteps,
    List<String> completedSteps,
    List<String> blockedSteps,
    Optional<String> activeStep
) {}

record StepDecision(
    String step,
    String rationale
) {}

record StepResult(
    String step,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record StepEntry(
    int attempt,
    String step,
    StepVerdict verdict,
    String result,
    Optional<String> blocker,
    Instant recordedAt
) {}

record StepLog(List<StepEntry> entries) {}

record PlanOutcome(
    String summary,
    List<String> completedSteps,
    Instant producedAt
) {}

record Plan(
    String planId,
    String goal,
    PlanStatus status,
    Optional<PlanLedger> ledger,
    Optional<StepLog> stepLog,
    Optional<PlanOutcome> outcome,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum StepVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum PlanStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`PlanEntity`)

`PlanCreated`, `PlanStarted`, `StepProposed`, `StepBlocked`, `StepRecorded`, `PlanRevised`, `PlanCompleted`, `PlanFailed`, `PlanHaltedAutomatic`, `PlanHaltedOperator`, `PlanFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`GoalQueue`)

`GoalSubmitted { planId, goal, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/plans` — body `{ goal, requestedBy? }` → `202 { planId }`. Starts a workflow.
- `GET /api/plans` — list all plans. Optional `?status=...`.
- `GET /api/plans/{id}` — one plan (full ledger + step log + outcome).
- `GET /api/plans/sse` — server-sent events stream of every plan change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "PlanAgents"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a goal, operator halt/resume control, live list of plans with status pills, expand-row to see the plan ledger, the step log entries, and the final outcome.

Browser title: `<title>Akka Sample: PlanAgents</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `PlannerAgent`): every `StepDecision` is checked against (a) a content policy that forbids destructive shell commands, file writes outside `/workspace/`, and outbound calls to non-allow-listed hosts, and (b) a step-length limit to prevent runaway subtasks. Blocking. Failure → `StepBlocked` entry + replan request.
- **HT1 — automatic safety halt** (`halt`, flavor `automatic-safety-halt`): after each `StepEntry` is appended, the workflow runs a deterministic evaluator. On an unsafe-action signal (executor returns evidence of a destructive action attempt or the replan budget is exhausted), the workflow emits `PlanHaltedAutomatic` and ends.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Owns the plan ledger; decides next step.
- `ExecutorAgent` → `prompts/executor.md`. Runs one step against fixtures; returns result.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Draft and send a weekly status report to the team." Plan progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a plan ledger with a non-empty step list, a step log with 3–8 entries, and a non-empty `PlanOutcome`.
2. **J2** — Submit a goal whose first step would violate the content policy (e.g., "Delete all temporary files with `rm -rf /tmp`"). The guardrail blocks the step; the planner replans; the plan either completes via a different path or fails after the replan budget is exhausted.
3. **J3** — Submit a goal and click **Halt new steps** while it is `EXECUTING`. The in-flight step finishes; no further steps are dispatched; the plan ends in `HALTED`.
4. **J4** — Force two consecutive `REPLAN` decisions (mock LLM fixture or injected test goal). On the third consecutive replan the workflow emits `PlanFailed`; plan status moves to `FAILED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named planner-executor-general-planner-executor demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-planner-executor.
Java package io.akka.samples.planagents. Akka 3.6.0. HTTP port 9712.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(CREATE_PLAN).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(DECIDE_STEP).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(COMPOSE_OUTCOME).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. CREATE_PLAN returns PlanLedger.
    DECIDE_STEP returns a NextAction tagged union (Continue(StepDecision) |
    Replan(revisedLedger) | Complete(PlanOutcome stub) | Fail(reason)).
    COMPOSE_OUTCOME returns PlanOutcome.
  * ExecutorAgent — capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)).
    Prompt from prompts/executor.md. Returns StepResult.

- 1 Workflow PlanWorkflow with steps:
  createPlanStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  executeStep -> recordStep -> autoHaltEvalStep -> decideStep
  -> [back to checkHaltStep or to composeOutcomeStep / completeStep / failStep /
  haltedStep].
  Step timeouts (override settings() per Lesson 4):
    createPlanStep ofSeconds(60), proposeStep ofSeconds(45), executeStep
    ofSeconds(120), decideStep ofSeconds(45), composeOutcomeStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(PlanWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits PlanHaltedOperator on PlanEntity).
  guardrailStep runs StepGuardrail.vet(StepDecision); on reject records a
  StepBlocked entry via PlanEntity.recordBlock(step, reason) and loops back to
  proposeStep.
  executeStep calls ExecutorAgent via forAutonomousAgent(...).runSingleTask(...)
  then forPlan(planId).result(...).
  recordStep calls PlanEntity.recordStep(entry).
  autoHaltEvalStep runs StepSafetyEvaluator.evaluate(entry) and, on unsafe,
  transitions to haltedStep (emits PlanHaltedAutomatic).
  decideStep calls forAutonomousAgent(PlannerAgent.class, DECIDE_STEP);
  on Continue loops; on Replan loops (increment replan counter; on third
  consecutive replan transition to failStep); on Complete transitions to
  composeOutcomeStep -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity PlanEntity holding Plan state. emptyState() returns
  Plan.initial("", null) with no commandContext() reference. Commands:
  createPlan, recordLedger, recordBlock, recordStep, reviseLedger,
  completePlan, failPlan, haltAutomatic, haltOperator, timeoutFail, getPlan.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity GoalQueue with command enqueueGoal(planId, goal,
  requestedBy) emitting GoalSubmitted.

- 1 View PlanView with row type PlanRow (mirror of Plan minus heavy step log —
  truncate to last 3 step entries plus counts; UI fetches the full plan by id
  on click). Table updater consumes PlanEntity events.
  ONE query getAllPlans SELECT * AS plans FROM plan_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer GoalRequestConsumer subscribed to GoalQueue events; on
  GoalSubmitted starts a PlanWorkflow with planId as the workflow id.

- 2 TimedActions:
  * GoalSimulator — every 90s, reads next line from
    src/main/resources/sample-events/goal-prompts.jsonl and calls
    GoalQueue.enqueueGoal.
  * StuckPlanMonitor — every 30s, queries PlanView.getAllPlans, filters
    EXECUTING plans whose createdAt is older than 5 minutes, calls
    PlanEntity.timeoutFail; PlanWorkflow polls PlanEntity.getPlan in its
    decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * PlanEndpoint at /api with POST /plans, GET /plans (filters client-side
    from getAllPlans), GET /plans/{id}, GET /plans/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: CREATE_PLAN
  (resultConformsTo PlanLedger), DECIDE_STEP (NextAction), COMPOSE_OUTCOME
  (PlanOutcome).
- ExecutorTasks.java declaring one Task<R> constant: EXECUTE_STEP
  (resultConformsTo StepResult).
- Domain records as listed in SPEC §5, plus a NextAction sealed interface with
  permits Continue, Replan, Complete, Fail (each carrying its own payload —
  Continue with StepDecision, Replan with revisedLedger, Complete with
  PlanOutcome stub, Fail with failureReason).
- application/StepGuardrail.java — deterministic vetter. Reject if the
  step text matches /^(rm|sudo|mkfs|dd|chmod 777|>>?\s*\/(etc|proc|sys|dev))/,
  if it references file writes outside /workspace/, or if it names a host not
  on the allow-list (akka.io, doc.akka.io, github.com). Also reject if
  step text exceeds 500 characters (runaway subtask guard).
- application/StepSafetyEvaluator.java — deterministic post-execution check.
  Flag UNSAFE on step results containing destructive-shell evidence (e.g.,
  "permission denied" plus "sudo"), on results whose content references writes
  to /etc, /root, or ~/.ssh/, on results containing explicit prompt-injection
  markers ("ignore previous instructions", "<system>").
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9712 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/goal-prompts.jsonl with 8 canned goals
  spanning planning, research, writing, and multi-step data tasks.
- src/main/resources/sample-data/fixtures.jsonl — 12 canned fixture entries
  (topic, content excerpt) used by ExecutorAgent to simulate step execution.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, HT1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance.capabilities; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md and prompts/executor.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: PlanAgents", one-line
  pitch, prerequisites (integration form: None), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs: Overview,
  Architecture (4 mermaid diagrams + click-to-expand component table with
  syntax-highlighted Java snippets), Risk Survey (7 sub-tabs with answers
  populated from risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix
  (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form + operator halt/resume control + live
  list with status pills and expand-on-click for ledger and step log and
  outcome). Browser title exactly:
  <title>Akka Sample: PlanAgents</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  ModelProvider with per-agent dispatch on agent class name and Task<R> id.
  Reads from src/main/resources/mock-responses/<agent-name>.json.
  Per-agent mock-response shapes:
    planner.json — three lists keyed by task id:
      "CREATE_PLAN" → 4 PlanLedger entries (goal, openSteps 3–6 items,
        completedSteps [], blockedSteps [], activeStep null).
      "DECIDE_STEP" → 6 NextAction entries: 4 Continue (advancing through a
        plausible step narrative), 1 Replan (with revised ledger), 1 Complete.
      "COMPOSE_OUTCOME" → 3 PlanOutcome entries with 60–100 word summaries
        and 3–4 completedSteps bullets.
    executor.json — 8 StepResult entries, ok=true, content fields are 4–6
      line simulated step outputs (research notes, drafted paragraphs,
      data summaries).

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: AutonomousAgent — both agents extend AutonomousAgent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent.
- Lesson 6: Optional<T> for every nullable field on PlanRow and Plan entity
  state (ledger, stepLog, outcome, failureReason, haltReason, finishedAt,
  activeStep on PlanLedger).
- Lesson 7: PlannerTasks.java and ExecutorTasks.java declaring every Task<R>.
- Lesson 8: conservative model defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: HTTP port 9712 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: tab switching by data-tab / data-panel; no zombie panels.
- The Overview tab's Try-it card shows just "/akka:build".
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
