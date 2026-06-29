# SPEC — itsm-change-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** ITSM Change Management Agent.
**One-line pitch:** Submit a change request; a PlannerAgent analyzes impact and similar history to produce implementation, test, and backout plans on a change ledger; a Change Advisory Board approval gate holds execution; a production-touch guardrail intercepts each step; a failed test step automatically triggers the pre-generated backout plan.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to IT service management change requests. The PlannerAgent owns a `ChangeLedger` — an impact assessment, a ranked list of similar historical changes, a sequenced implementation plan, a parallel test plan, and a pre-generated backout plan. The ExecutorAgent takes one implementation step at a time; the TestRunnerAgent validates each step; the BackoutAgent reverses work when tests fail.

The blueprint wires three governance mechanisms into that loop:

- a **HITL application approval gate** that parks the change in `AWAITING_CAB` until a Change Advisory Board reviewer calls the approval endpoint — no execution steps run until the gate opens,
- a **before-tool-call guardrail** that vets each implementation step against a production-touch policy before the executor runs it,
- an **automatic safety halt** that fires when a test step fails, pausing the executor and handing control to the BackoutAgent.

## 3. User-facing flows

The user opens the App UI tab and submits a change request via the form.

1. The system creates a `Change` record in `PLANNING` and starts a `ChangeWorkflow`.
2. The PlannerAgent receives the request text, queries historical-change fixtures by keyword, and produces a `ChangeLedger { impactAssessment, similarChanges, implementationPlan, testPlan, backoutPlan }`. The workflow emits `ChangePlanned`.
3. The change moves to `AWAITING_CAB`. The workflow parks in a `cabApprovalStep` that polls `ChangeEntity.getChange` until `status == APPROVED` or `status == REJECTED`.
4. A CAB reviewer calls `POST /api/changes/{id}/approve` or `POST /api/changes/{id}/reject`.
   - On rejection: the workflow emits `ChangeRejected` and ends with status `REJECTED`.
   - On approval: the workflow emits `ChangeApproved` and enters the executor loop.
5. The executor loop processes implementation steps in order. Each iteration:
   - The workflow reads the next un-executed step from `ChangeLedger.implementationPlan`.
   - The **production-touch guardrail** vets the step against the CI allow-list and forbidden-path policy. On rejection, the workflow records a `StepBlocked` entry and asks the PlannerAgent to revise the step.
   - The ExecutorAgent runs the step and returns a `StepResult`.
   - The TestRunnerAgent runs the matching test step and returns a `TestResult`.
   - On test failure: the **automatic safety halt** fires, the workflow transitions to the backout branch.
   - On test pass: the workflow appends a `StepRecord` to the `ExecutionLog` and advances to the next step.
6. When all implementation steps pass their tests, the workflow emits `ChangeImplemented`. Status → `IMPLEMENTED`.
7. In the backout branch: the BackoutAgent executes each backout step, returning a `BackoutStepResult`. On completion, `ChangeRolledBack` is emitted. Status → `ROLLED_BACK`.
8. The operator can press **Halt execution** at any time. The workflow finishes the in-flight step pair, then emits `ChangeHaltedOperator` and ends with status `HALTED`.

A `ChangeSimulator` (TimedAction) drips a sample change request every 90 seconds.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Reads a change request + historical fixtures; produces `ChangeLedger`. Also called to revise a blocked step. | `ChangeWorkflow` | returns typed result to workflow |
| `ExecutorAgent` | `AutonomousAgent` | Executes one implementation step from the `ChangeLedger`; returns `StepResult`. | `ChangeWorkflow` | — |
| `TestRunnerAgent` | `AutonomousAgent` | Runs the test step corresponding to the last executed implementation step; returns `TestResult`. | `ChangeWorkflow` | — |
| `BackoutAgent` | `AutonomousAgent` | Executes backout steps from `ChangeLedger.backoutPlan`; returns `BackoutStepResult`. | `ChangeWorkflow` | — |
| `ChangeWorkflow` | `Workflow` | Drives plan → cab-approval-poll → execute-step → guardrail → run-step → test-step → decide loop; backout branch; terminal states. | `ChangeEndpoint`, `ChangeRequestConsumer` | `ChangeEntity` |
| `ChangeEntity` | `EventSourcedEntity` | Holds the change's lifecycle, `ChangeLedger`, `ExecutionLog`, and CAB decision. | `ChangeWorkflow` | `ChangeView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `ChangeEndpoint` (operator action) | `ChangeWorkflow` (polls) |
| `ChangeQueue` | `EventSourcedEntity` | Audit log of submitted change requests. | `ChangeEndpoint`, `ChangeSimulator` | `ChangeRequestConsumer` |
| `ChangeView` | `View` | List-of-changes read model for the UI. Row type is `ChangeRow`. | `ChangeEntity` events | `ChangeEndpoint` |
| `ChangeRequestConsumer` | `Consumer` | Subscribes to `ChangeQueue` events; starts a `ChangeWorkflow` per submission. | `ChangeQueue` events | `ChangeWorkflow` |
| `ChangeSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/change-requests.jsonl` and enqueues it. | scheduler | `ChangeQueue` |
| `StuckChangeMonitor` | `TimedAction` | Every 60 s, marks any change stuck in `EXECUTING` or `ROLLING_BACK` past 10 minutes as `STUCK`. | scheduler | `ChangeEntity` |
| `ChangeEndpoint` | `HttpEndpoint` | `/api/changes/*` — submit, get, list, SSE, approve, reject, operator halt. | — | `ChangeView`, `ChangeQueue`, `ChangeEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record ChangeRequest(String summary, String ciName, String requestedBy, ChangeCategory category) {}

record HistoricalChange(String changeId, String summary, String outcome, String lessonsLearned) {}

record ImplementationStep(int sequence, String description, String targetCi, String expectedOutcome) {}

record TestStep(int sequence, String description, String successCriteria) {}

record BackoutStep(int sequence, String description, String targetCi) {}

record ChangeLedger(
    String impactAssessment,
    List<HistoricalChange> similarChanges,
    List<ImplementationStep> implementationPlan,
    List<TestStep> testPlan,
    List<BackoutStep> backoutPlan
) {}

record StepResult(
    int sequence,
    String description,
    boolean ok,
    String evidence,
    Optional<String> errorReason
) {}

record TestResult(
    int sequence,
    boolean passed,
    String observation,
    Optional<String> failureDetail
) {}

record BackoutStepResult(
    int sequence,
    boolean ok,
    String evidence
) {}

record StepRecord(
    int sequence,
    String description,
    StepResult stepResult,
    TestResult testResult,
    Instant recordedAt
) {}

record ExecutionLog(List<StepRecord> records) {}

record CabDecision(
    CabOutcome outcome,
    String reviewedBy,
    Optional<String> comments,
    Instant decidedAt
) {}

record Change(
    String changeId,
    String summary,
    String ciName,
    ChangeCategory category,
    String requestedBy,
    ChangeStatus status,
    Optional<ChangeLedger> ledger,
    Optional<ExecutionLog> executionLog,
    Optional<CabDecision> cabDecision,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ChangeCategory { STANDARD, NORMAL, EMERGENCY }
enum CabOutcome { APPROVED, REJECTED }
enum ChangeStatus {
    PLANNING, AWAITING_CAB, APPROVED, EXECUTING,
    ROLLING_BACK, IMPLEMENTED, REJECTED, ROLLED_BACK,
    FAILED, HALTED, STUCK
}
```

### Events (`ChangeEntity`)

`ChangeCreated`, `ChangePlanned`, `ChangeApproved`, `ChangeRejected`, `StepBlocked`, `StepExecuted`, `TestPassed`, `TestFailed`, `BackoutStepExecuted`, `ChangeImplemented`, `ChangeRolledBack`, `ChangeFailed`, `ChangeHaltedOperator`, `ChangeFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`ChangeQueue`)

`ChangeSubmitted { changeId, summary, ciName, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/changes` — body `{ summary, ciName, requestedBy?, category? }` → `202 { changeId }`. Starts a workflow.
- `GET /api/changes` — list all changes. Optional `?status=...`.
- `GET /api/changes/{id}` — one change (full ledger + execution log + CAB decision).
- `GET /api/changes/sse` — server-sent events stream of every change state transition.
- `POST /api/changes/{id}/approve` — body `{ reviewedBy, comments? }` → `200`. Sets CAB approval.
- `POST /api/changes/{id}/reject` — body `{ reviewedBy, comments? }` → `200`. Sets CAB rejection.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "ITSM Change Management Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a change request, CAB approval/rejection controls, operator halt/resume control, live list of changes with status pills, expand-row to see the change ledger (implementation, test, and backout plans), execution log, and CAB decision.

Browser title: `<title>Akka Sample: ITSM Change Management Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL CAB approval gate** (`hitl`, flavor `application`): after `ChangePlanned`, the workflow parks in `cabApprovalStep` and polls `ChangeEntity.getChange` until the status field is `APPROVED` or `REJECTED`. The CAB reviewer calls `POST /api/changes/{id}/approve` or `POST /api/changes/{id}/reject`. No implementation steps execute until the gate opens with `APPROVED`. On `REJECTED`, the workflow ends immediately.
- **G1 — production-touch guardrail** (`guardrail`, flavor `before-tool-call`): before the ExecutorAgent runs each implementation step, `ProductionTouchGuardrail.vet(ImplementationStep)` checks (a) the `targetCi` is on the allow-listed CI set, (b) the description does not reference forbidden paths (`/etc`, `/root`, kernel modules, boot records). On rejection the workflow records a `StepBlocked` entry and asks the PlannerAgent to revise the step.
- **HT1 — automatic safety halt on test failure** (`halt`, flavor `automatic-safety-halt`): after the TestRunnerAgent returns, if `TestResult.passed == false`, the workflow transitions to the backout branch — it emits `TestFailed`, sets status to `ROLLING_BACK`, and invokes the BackoutAgent sequentially through `ChangeLedger.backoutPlan`. On all backout steps completing, it emits `ChangeRolledBack` and ends with `ROLLED_BACK`.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Analyzes impact; queries history fixtures; produces the three-plan `ChangeLedger`.
- `ExecutorAgent` → `prompts/executor.md`. Executes one implementation step; returns evidence.
- `TestRunnerAgent` → `prompts/test-runner.md`. Runs the corresponding test step; returns pass/fail with observation.
- `BackoutAgent` → `prompts/backout.md`. Executes backout steps in reverse order; returns evidence.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a normal change request. Change progresses `PLANNING → AWAITING_CAB → APPROVED → EXECUTING → IMPLEMENTED` within ~5 minutes. The execution log shows one `StepRecord` per implementation step, each with `testResult.passed = true`.
2. **J2** — CAB reviewer calls the reject endpoint. Change moves to `REJECTED`. No `StepExecuted` or `TestPassed` events appear on the entity.
3. **J3** — Submit a change and configure a test step to fail (mock LLM returns `passed=false`). After the failed test, the workflow enters the backout branch. Change ends in `ROLLED_BACK`. The execution log shows a partial set of `StepRecord`s plus `BackoutStepExecuted` entries.
4. **J4** — Submit a change whose first implementation step targets a CI not on the allow-list. The guardrail blocks it; a `StepBlocked` entry appears; the planner revises the step. The original disallowed step never reaches the ExecutorAgent.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named itsm-change-planner demonstrating the
planner-executor × ops-automation cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-ops-automation-itsm-change-planner.
Java package io.akka.samples.itsmchangemanagementagent. Akka 3.6.0. HTTP port 9979.

Components to wire (exactly):
- 4 AutonomousAgents:
  * PlannerAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(PLAN_CHANGE).maxIterationsPerTask(4)) and
      capability(TaskAcceptance.of(REVISE_STEP).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_CHANGE returns ChangeLedger.
    REVISE_STEP returns a revised ImplementationStep.
  * ExecutorAgent — capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)).
    Prompt from prompts/executor.md. Returns StepResult.
  * TestRunnerAgent — capability(TaskAcceptance.of(RUN_TEST).maxIterationsPerTask(2)).
    Prompt from prompts/test-runner.md. Returns TestResult.
  * BackoutAgent — capability(TaskAcceptance.of(EXECUTE_BACKOUT).maxIterationsPerTask(2)).
    Prompt from prompts/backout.md. Returns BackoutStepResult.

- 1 Workflow ChangeWorkflow with steps:
  planStep -> cabApprovalPollStep -> [loop: checkHaltStep -> guardrailStep ->
  executeStepStep -> testStepStep -> recordStepStep -> advanceStep]
  -> [on all steps done: completeStep | on test fail: backoutBranchStep ->
  backoutStep (repeating) -> rolledBackStep | on reject: rejectedStep |
  on halt: haltedStep | on fail: failStep].
  Step timeouts:
    planStep ofSeconds(90), executeStepStep ofSeconds(120),
    testStepStep ofSeconds(120), backoutStep ofSeconds(120).
    cabApprovalPollStep polls every 10s; no agent call timeout needed.
    defaultStepRecovery(maxRetries(2).failoverTo(ChangeWorkflow::error)).
  cabApprovalPollStep reads ChangeEntity.getChange; loops until status is
  APPROVED or REJECTED; exits to executor loop on APPROVED; exits to
  rejectedStep on REJECTED.
  guardrailStep calls ProductionTouchGuardrail.vet(nextStep); on rejection
  calls ChangeEntity.recordBlock(step, reason) and loops back with the
  PlannerAgent's REVISE_STEP result.
  executeStepStep calls forAutonomousAgent(ExecutorAgent.class, EXECUTE_STEP).
  testStepStep calls forAutonomousAgent(TestRunnerAgent.class, RUN_TEST).
  On TestResult.passed==false, transitions to backoutBranchStep.
  recordStepStep calls ChangeEntity.recordStep(StepRecord).
  advanceStep increments the current step index; when all steps done,
  transitions to completeStep.
  backoutBranchStep emits TestFailed + ChangeRolledBack_start on ChangeEntity,
  then iterates backoutSteps via BackoutAgent EXECUTE_BACKOUT.

- 1 EventSourcedEntity ChangeEntity holding Change state. emptyState() returns
  Change.initial("", "", ChangeCategory.NORMAL, "") with no commandContext().
  Commands: createChange, recordPlan, recordCabDecision, recordBlock,
  recordStep, recordBackoutStep, completeChange, failChange, haltOperator,
  timeoutFail, getChange. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity ChangeQueue with command enqueueChange(changeId, summary,
  ciName, requestedBy, category) emitting ChangeSubmitted.

- 1 View ChangeView with row type ChangeRow (mirror of Change minus heavy
  ledger payloads — implementationPlan/testPlan/backoutPlan omitted;
  executionLog truncated to last 3 StepRecords plus counts; the UI fetches
  the full change by id on click). Table updater consumes ChangeEntity events.
  ONE query getAllChanges SELECT * AS changes FROM change_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer ChangeRequestConsumer subscribed to ChangeQueue events; on
  ChangeSubmitted starts a ChangeWorkflow with changeId as the workflow id.

- 2 TimedActions:
  * ChangeSimulator — every 90s, reads next line from
    src/main/resources/sample-events/change-requests.jsonl and calls
    ChangeQueue.enqueueChange.
  * StuckChangeMonitor — every 60s, queries ChangeView.getAllChanges, filters
    EXECUTING or ROLLING_BACK changes older than 10 minutes, calls
    ChangeEntity.timeoutFail.

- 2 HttpEndpoints:
  * ChangeEndpoint at /api with POST /changes, GET /changes, GET /changes/{id},
    GET /changes/sse, POST /changes/{id}/approve, POST /changes/{id}/reject,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring two Task<R> constants: PLAN_CHANGE
  (resultConformsTo ChangeLedger), REVISE_STEP (resultConformsTo
  ImplementationStep).
- ExecutorTasks.java declaring three Task<R> constants: EXECUTE_STEP
  (resultConformsTo StepResult), RUN_TEST (resultConformsTo TestResult),
  EXECUTE_BACKOUT (resultConformsTo BackoutStepResult).
- Domain records as listed in SPEC §5, plus enums ChangeCategory, CabOutcome,
  ChangeStatus.
- application/ProductionTouchGuardrail.java — deterministic vetter.
  Reject if targetCi is not in the allow-listed CI set (loaded from
  src/main/resources/sample-data/ci-allowlist.json); if description
  references /etc, /root, kernel module names (ko extension), /boot, MBR,
  or grub.cfg.
- application/BackoutEvaluator.java — deterministic post-backout check.
  Flag INCOMPLETE if fewer backout steps succeeded than the change's
  implementationPlan length before failure; used to set failureReason on
  ChangeRolledBack event.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9979 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/change-requests.jsonl with 8 canned
  change requests spanning STANDARD, NORMAL, and EMERGENCY categories
  targeting seeded CI names.
- src/main/resources/sample-data/historical-changes.jsonl — 10 historical
  change records with outcomes (succeeded / failed / rolled-back) and
  lessons learned. Used by PlannerAgent.
- src/main/resources/sample-data/ci-allowlist.json — 12 CI entries
  (server names, DB instance names, load-balancer identifiers) that
  the production-touch guardrail permits.
- src/main/resources/sample-data/cmdb-snapshot.jsonl — 12 CI records
  (name, type, owner, environment: prod/staging/dev). Used by PlannerAgent
  for impact assessment.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies for the metadata endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (H1, G1, HT1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance.
- prompts/planner.md, prompts/executor.md, prompts/test-runner.md,
  prompts/backout.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: ITSM Change
  Management Agent", one-line pitch, prerequisites (integration form
  host-software: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — single self-contained
  HTML file. Inline CSS + JS. Runtime CDN imports for markdown/YAML libs
  acceptable. Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs with answers from risk-survey.yaml; unanswered
  .qb opacity 0.45), Eval Matrix (5-column table with click-to-expand
  rows), App UI (form + CAB approval pane + operator halt/resume + live
  list with status pills and expand-on-click for ledger, execution log,
  and CAB decision). Browser title exactly: <title>Akka Sample: ITSM
  Change Management Agent</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider returning
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file. No .env, no entry in
  application.conf, no secrets.yaml. Akka records only the REFERENCE.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch. Each branch
  reads from src/main/resources/mock-responses/<agent-name>.json:
    planner.json — 4–6 ChangeLedger entries, each with an impactAssessment,
      2–3 similarChanges, 3–5 implementationPlan steps, matching testPlan
      steps, and 3–5 backoutPlan steps in reverse order.
    executor.json — 5 StepResult entries, ok=true, evidence is 3–5 lines
      of simulated change evidence.
    test-runner.json — 5 TestResult entries; at least one has passed=false
      with a failureDetail so J3 can fire.
    backout.json — 5 BackoutStepResult entries, ok=true.
  MockModelProvider.seedFor(changeId) makes selection deterministic per
  change id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: AutonomousAgent never silently downgraded — extends AutonomousAgent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  executeStepStep, testStepStep, backoutStep.
- Lesson 6: Optional<T> for every nullable field on ChangeRow and Change state.
- Lesson 7: PlannerTasks.java and ExecutorTasks.java declaring every Task<R>.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: HTTP port 9979.
- Lesson 11: source.platform never in user-facing surfaces.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: mermaid CSS overrides AND themeVariables present.
- Lesson 25: API-key sourcing follows the five-option flow; no key value to disk.
- Lesson 26: tab switching by data-tab / data-panel; no zombie panels.
- Overview tab's Try-it card shows just "/akka:build".
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
