# SPEC — version-upgrade-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Airflow Version Upgrade Agent.
**One-line pitch:** Submit an Airflow upgrade job; an UpgradePlannerAgent builds an ordered plan on a plan ledger, dispatches each phase to one of three executor agents (compatibility check, test run, migration apply), records outcomes on a progress ledger, and requires explicit human approval before any destructive migration phase runs.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to a multi-phase software upgrade pipeline, with two governance controls wired into the execution loop.

The UpgradePlannerAgent owns two ledgers — a **plan ledger** (source version, target version, ordered phases, current phase index, replanning rationale) and a **progress ledger** (each phase's attempt count, executor verdict, test report, migration outcome). Each loop iteration the Planner reads both ledgers, picks the next phase, and either continues, replans, halts, or completes.

Two governance controls gate the loop:

- A **human-in-the-loop approval gate** (`hitl`, flavor `application`) that pauses the workflow before any migration phase marked `requiresApproval=true` is dispatched. A reviewer must explicitly approve or reject. On rejection the planner replans.
- A **CI test gate** (`ci-gate`, flavor `test-gate`) that runs the test suite after every applied migration phase. If the test report contains failures, the gate blocks the next phase from dispatching; the planner replans and may roll back or abort.

## 3. User-facing flows

The user opens the App UI tab and submits an upgrade job via the form.

1. The system creates an `UpgradeJob` record in `PLANNING` and starts an `UpgradeWorkflow`.
2. The UpgradePlannerAgent drafts a `PlanLedger { sourceVersion, targetVersion, phases, currentPhaseIndex, replanReason }` and emits `JobPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes a `PhaseDecision { phase, executor, rationale, requiresApproval }`.
   - If `requiresApproval=true`, the workflow enters `approvalGateStep`: it writes a pending `ApprovalRequest` to `ApprovalEntity`, sets job status to `AWAITING_APPROVAL`, and suspends until a reviewer calls `POST /api/approvals/{approvalId}/approve` or `/reject`.
   - On approval, execution continues; on rejection, the workflow records a `PhaseBlocked` entry and loops back to the planner.
   - The executor runs the phase and returns a typed `PhaseResult`.
   - The workflow appends a `ProgressEntry { phase, executor, attempt, verdict, migrationSummary, testReport }` to the progress ledger.
   - If the phase applied a migration, the **CI gate** step runs: it calls `TestRunnerAgent` and checks for failures. On failure, it records a `CiGateFailed` entry and loops back to the planner.
4. The Planner decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After two consecutive replans without progress or three consecutive failures on the same phase, the Planner emits `FAIL`.
5. On `COMPLETE`, the Planner produces an `UpgradeReport { summary, appliedPhases, testsPassed }` and emits `JobCompleted`. The job moves to `COMPLETED`.
6. The operator can press **Halt new phases** in the dashboard at any time. The workflow finishes the in-flight phase, then ends with `JobHaltedOperator`. The job moves to `HALTED`.

A `UpgradeRequestSimulator` (TimedAction) drips a sample upgrade job every 120 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `UpgradePlannerAgent` | `AutonomousAgent` | Plans upgrade phases, decides next phase on each loop tick. Maintains plan ledger; reads progress ledger. Produces `UpgradeReport` on completion. | `UpgradeWorkflow` | returns typed result to workflow |
| `CompatibilityCheckerAgent` | `AutonomousAgent` | Checks DAG and provider compatibility between source and target Airflow versions using seeded fixture data (`sample-data/compat-fixtures.jsonl`). | `UpgradeWorkflow` | — |
| `TestRunnerAgent` | `AutonomousAgent` | Executes the Airflow test suite against the target version (simulated via `sample-data/test-reports.jsonl`) and returns a structured `TestReport`. | `UpgradeWorkflow` | — |
| `MigrationApplierAgent` | `AutonomousAgent` | Applies a single migration phase from seeded scripts (`sample-data/migrations/*.yaml`) and returns a `MigrationOutcome`. | `UpgradeWorkflow` | — |
| `UpgradeWorkflow` | `Workflow` | Drives the plan → approval-gate → execute → record → ci-gate → decide loop, plus replan and halt branches. | `UpgradeJobEndpoint`, `UpgradeRequestConsumer` | `UpgradeJobEntity`, `ApprovalEntity` |
| `UpgradeJobEntity` | `EventSourcedEntity` | Holds the upgrade job lifecycle, plan ledger, progress ledger, and final report. | `UpgradeWorkflow` | `UpgradeJobView` |
| `ApprovalEntity` | `EventSourcedEntity` | Holds the pending approval request and the reviewer's decision. Keyed by `approvalId`. | `UpgradeWorkflow` | `UpgradeWorkflow` (polled) |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `UpgradeJobEndpoint` (operator action) | `UpgradeWorkflow` (polls) |
| `UpgradeRequestQueue` | `EventSourcedEntity` | Audit log of submitted upgrade jobs. | `UpgradeJobEndpoint`, `UpgradeRequestSimulator` | `UpgradeRequestConsumer` |
| `UpgradeJobView` | `View` | List-of-jobs read model for the UI. | `UpgradeJobEntity` events | `UpgradeJobEndpoint` |
| `UpgradeRequestConsumer` | `Consumer` | Subscribes to `UpgradeRequestQueue` events; starts an `UpgradeWorkflow` per submission. | `UpgradeRequestQueue` events | `UpgradeWorkflow` |
| `UpgradeRequestSimulator` | `TimedAction` | Every 120 s, reads a line from `sample-events/upgrade-requests.jsonl` and enqueues it. | scheduler | `UpgradeRequestQueue` |
| `StuckJobMonitor` | `TimedAction` | Every 60 s, marks any job stuck in `EXECUTING` past 10 minutes as `STUCK`. | scheduler | `UpgradeJobEntity` |
| `UpgradeJobEndpoint` | `HttpEndpoint` | `/api/jobs/*` and `/api/approvals/*` — submit, get, list, SSE, approve/reject, operator halt. | — | `UpgradeJobView`, `UpgradeRequestQueue`, `UpgradeJobEntity`, `ApprovalEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record UpgradeRequest(String sourceVersion, String targetVersion, String requestedBy) {}

record Phase(
    String phaseId,
    String name,
    PhaseKind kind,
    boolean requiresApproval
) {}

record PlanLedger(
    String sourceVersion,
    String targetVersion,
    List<Phase> phases,
    int currentPhaseIndex,
    Optional<String> replanReason
) {}

record PhaseDecision(
    Phase phase,
    ExecutorKind executor,
    String rationale,
    boolean requiresApproval
) {}

record TestReport(
    int total,
    int passed,
    int failed,
    List<String> failingTests
) {}

record MigrationOutcome(
    String phaseId,
    boolean applied,
    String summary,
    Optional<String> rollbackScript
) {}

record PhaseResult(
    ExecutorKind executor,
    String phaseId,
    boolean ok,
    String summary,
    Optional<TestReport> testReport,
    Optional<MigrationOutcome> migrationOutcome,
    Optional<String> errorReason
) {}

record ProgressEntry(
    int attempt,
    ExecutorKind executor,
    String phaseId,
    String phaseName,
    PhaseVerdict verdict,
    String summary,
    Optional<TestReport> testReport,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ProgressLedger(List<ProgressEntry> entries) {}

record ApprovalRequest(
    String approvalId,
    String jobId,
    String phaseId,
    String phaseName,
    String rationale,
    Instant requestedAt
) {}

record ApprovalDecision(
    String approvalId,
    boolean approved,
    String reviewedBy,
    Optional<String> comment,
    Instant decidedAt
) {}

record UpgradeReport(
    String summary,
    List<String> appliedPhases,
    boolean testsPassed,
    Instant producedAt
) {}

record UpgradeJob(
    String jobId,
    String sourceVersion,
    String targetVersion,
    String requestedBy,
    JobStatus status,
    Optional<PlanLedger> ledger,
    Optional<ProgressLedger> progress,
    Optional<ApprovalRequest> pendingApproval,
    Optional<UpgradeReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PhaseKind { COMPATIBILITY_CHECK, MIGRATION, TEST_RUN }
enum ExecutorKind { COMPAT_CHECKER, TEST_RUNNER, MIGRATION_APPLIER }
enum PhaseVerdict { OK, BLOCKED_BY_APPROVAL, CI_GATE_FAILED, FAILED }
enum JobStatus { PLANNING, EXECUTING, AWAITING_APPROVAL, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`UpgradeJobEntity`)

`JobCreated`, `JobPlanned`, `PhaseDispatched`, `ApprovalRequested`, `ApprovalGranted`, `ApprovalRejected`, `PhaseBlocked`, `PhaseRecorded`, `CiGateFailed`, `LedgerRevised`, `JobCompleted`, `JobFailed`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`ApprovalEntity`)

`ApprovalCreated { approvalId, jobId, phaseId, phaseName, rationale, requestedAt }`.
`ApprovalResolved { approvalId, approved, reviewedBy, comment, decidedAt }`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`UpgradeRequestQueue`)

`UpgradeSubmitted { jobId, sourceVersion, targetVersion, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ sourceVersion, targetVersion, requestedBy? }` → `202 { jobId }`.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full ledgers + report).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `GET /api/approvals/{approvalId}` — one pending approval request.
- `POST /api/approvals/{approvalId}/approve` — body `{ reviewedBy, comment? }` → `200`.
- `POST /api/approvals/{approvalId}/reject` — body `{ reviewedBy, comment }` → `200`.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Airflow Version Upgrade Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an upgrade job (source version, target version), operator halt/resume control, live list of jobs with status pills, expand-row to see the plan ledger, the progress ledger entries, any pending approval request, and the final report.

Browser title: `<title>Akka Sample: Airflow Version Upgrade Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **HI1 — human-in-the-loop approval gate** (`hitl`, flavor `application`): before the workflow dispatches any `Phase` whose `requiresApproval=true`, it writes an `ApprovalRequest` to `ApprovalEntity`, sets `UpgradeJobEntity` to `AWAITING_APPROVAL`, and suspends. The workflow's `approvalGateStep` polls `ApprovalEntity` until resolved. On approval, execution continues; on rejection, the workflow records a `PhaseBlocked` entry and loops back to `plannerDecideStep` so the Planner can revise.
- **CI1 — CI test gate** (`ci-gate`, flavor `test-gate`): after every phase that applied a migration (i.e., `PhaseKind.MIGRATION`), the workflow's `ciGateStep` calls `TestRunnerAgent`. If `TestReport.failed > 0`, the gate records a `CiGateFailed` entry and returns the test report to the Planner. The Planner may then emit `Replan` (e.g., schedule a rollback phase) or `Fail`. This step is non-blocking — the gate does not abort the workflow immediately; it informs the Planner, which decides the next action.

## 9. Agent prompts

- `UpgradePlannerAgent` → `prompts/upgrade-planner.md`. Maintains both ledgers; decides next phase.
- `CompatibilityCheckerAgent` → `prompts/compatibility-checker.md`. Returns compatibility findings from fixtures.
- `TestRunnerAgent` → `prompts/test-runner.md`. Returns a structured test report.
- `MigrationApplierAgent` → `prompts/migration-applier.md`. Applies one migration phase; returns the outcome.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an upgrade from Airflow 2.7.3 to 2.10.0. Job progresses `PLANNING → EXECUTING → AWAITING_APPROVAL → EXECUTING → COMPLETED` within ~5 minutes. UI reflects each transition via SSE. The expanded view shows a plan ledger with at least 3 ordered phases, a progress ledger with entries from all three executors, and a non-empty `UpgradeReport`.
2. **J2** — Submit the same job; when the approval gate fires, click **Reject** in the UI with a comment. The workflow records `PhaseBlocked` with `verdict = BLOCKED_BY_APPROVAL`; the planner either replans or fails. The rejected phase does not execute.
3. **J3** — Submit a job and click **Halt new phases** while the job is `EXECUTING`. The in-flight phase finishes; no further phases dispatch; the job ends in `HALTED`.
4. **J4** — Configure the test-runner fixture to return a failing test report on the first migration. The `ciGateStep` records `CiGateFailed`; the planner replans (e.g., adds a rollback phase); the job either recovers and completes or fails gracefully after the replan budget is exhausted.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named version-upgrade-planner demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-version-upgrade-planner.
Java package io.akka.samples.airflowversionupgradeagent. Akka 3.6.0. HTTP port 9335.

Components to wire (exactly):
- 4 AutonomousAgents:
  * UpgradePlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_UPGRADE).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(DECIDE_PHASE).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/upgrade-planner.md. PLAN_UPGRADE returns PlanLedger.
    DECIDE_PHASE returns a PhaseNextStep tagged union (Continue(PhaseDecision) |
    Replan(revisedPlanLedger) | Complete(UpgradeReport stub) | Fail(reason)).
    COMPOSE_REPORT returns UpgradeReport.
  * CompatibilityCheckerAgent — capability(TaskAcceptance.of(CHECK_COMPAT).maxIterationsPerTask(2)).
    Prompt from prompts/compatibility-checker.md. Returns PhaseResult.
  * TestRunnerAgent — capability(TaskAcceptance.of(RUN_TESTS).maxIterationsPerTask(2)).
    Prompt from prompts/test-runner.md. Returns PhaseResult with testReport populated.
  * MigrationApplierAgent — capability(TaskAcceptance.of(APPLY_MIGRATION).maxIterationsPerTask(2)).
    Prompt from prompts/migration-applier.md. Returns PhaseResult with migrationOutcome populated.

- 1 Workflow UpgradeWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> plannerDecideStep ->
  approvalGateStep (conditional on requiresApproval) -> dispatchStep ->
  ciGateStep (conditional on phase.kind==MIGRATION) -> recordStep -> decideStep
  -> [back to checkHaltStep or to composeReportStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(90), plannerDecideStep ofSeconds(60),
    approvalGateStep ofSeconds(3600) (long poll for human response),
    dispatchStep ofSeconds(180), ciGateStep ofSeconds(120),
    composeReportStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(UpgradeWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits JobHaltedOperator on UpgradeJobEntity).
  approvalGateStep writes ApprovalRequest to ApprovalEntity, then polls
  ApprovalEntity until ApprovalResolved arrives. On approved=true continues;
  on approved=false records PhaseBlocked and loops back to plannerDecideStep.
  dispatchStep switches on PhaseDecision.executor to call the matching agent
  via forAutonomousAgent(...).runSingleTask(...).
  ciGateStep calls TestRunnerAgent.RUN_TESTS after any MIGRATION phase.
  On TestReport.failed > 0, records CiGateFailed entry on UpgradeJobEntity
  and loops back to plannerDecideStep with the test report in context.
  recordStep calls UpgradeJobEntity.recordProgress(entry).
  decideStep calls forAutonomousAgent(UpgradePlannerAgent.class, DECIDE_PHASE);
  on Continue or Replan loops; on Complete transitions to composeReportStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity UpgradeJobEntity holding UpgradeJob state.
  emptyState() returns UpgradeJob.initial. Commands: createJob, recordPlan,
  requestApproval, grantApproval, rejectApproval, recordBlock, recordProgress,
  recordCiFailure, reviseLedger, completeJob, failJob, haltOperator,
  timeoutFail, getJob. Events as listed in SPEC §5.

- 1 EventSourcedEntity ApprovalEntity keyed by approvalId. State:
  ApprovalState { Optional<ApprovalRequest> request, Optional<ApprovalDecision> decision }.
  Commands: createApproval(request), resolve(decision), get. Events:
  ApprovalCreated, ApprovalResolved.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl { boolean halted, Optional<String> reason, Optional<Instant> haltedAt }.
  Commands: requestHalt(reason), clearHalt, get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity UpgradeRequestQueue with command enqueueJob(jobId,
  sourceVersion, targetVersion, requestedBy) emitting UpgradeSubmitted.

- 1 View UpgradeJobView with row type UpgradeJobRow (mirror of UpgradeJob
  minus heavy ledger payloads — truncate to last 3 progress entries plus counts;
  UI fetches full job by id on click). ONE query getAllJobs SELECT * AS jobs
  FROM upgrade_job_view. No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer UpgradeRequestConsumer subscribed to UpgradeRequestQueue events;
  on UpgradeSubmitted starts an UpgradeWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * UpgradeRequestSimulator — every 120s, reads next line from
    src/main/resources/sample-events/upgrade-requests.jsonl and calls
    UpgradeRequestQueue.enqueueJob.
  * StuckJobMonitor — every 60s, queries UpgradeJobView.getAllJobs, filters
    EXECUTING and AWAITING_APPROVAL jobs whose createdAt is older than 10 minutes,
    calls UpgradeJobEntity.timeoutFail.

- 2 HttpEndpoints:
  * UpgradeJobEndpoint at /api with POST /jobs, GET /jobs (filters client-side),
    GET /jobs/{id}, GET /jobs/sse, GET /approvals/{approvalId},
    POST /approvals/{approvalId}/approve, POST /approvals/{approvalId}/reject,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_UPGRADE
  (resultConformsTo PlanLedger), DECIDE_PHASE (PhaseNextStep),
  COMPOSE_REPORT (UpgradeReport).
- ExecutorTasks.java declaring three Task<R> constants: CHECK_COMPAT,
  RUN_TESTS, APPLY_MIGRATION (all resultConformsTo PhaseResult).
- Domain records as listed in SPEC §5, plus a PhaseNextStep sealed interface
  with permits Continue, Replan, Complete, Fail.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9335 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/upgrade-requests.jsonl with 6 canned
  upgrade-job requests spanning common Airflow version pairs.
- src/main/resources/sample-data/compat-fixtures.jsonl — 10 canned
  compatibility findings (provider, dag_count, issues, safe_to_proceed).
- src/main/resources/sample-data/test-reports.jsonl — 8 canned test reports
  (total, passed, failed, failingTests). At least two reports have failed > 0
  for J4 acceptance test coverage.
- src/main/resources/sample-data/migrations/*.yaml — 5 short migration
  scripts (one per common Airflow migration phase: db_migrate, provider_update,
  dag_serialisation, config_update, webserver_restart).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.
- eval-matrix.yaml at the project root with 2 controls (HI1, CI1).
- risk-survey.yaml at the project root.
- prompts/upgrade-planner.md, prompts/compatibility-checker.md,
  prompts/test-runner.md, prompts/migration-applier.md.
- src/main/resources/static-resources/index.html — single self-contained HTML
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture, Risk Survey,
  Eval Matrix, App UI. Browser title exactly:
  <title>Akka Sample: Airflow Version Upgrade Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — per-agent mock-response shapes:
- upgrade-planner.json — sectioned by task id:
    "PLAN_UPGRADE" → 4 PlanLedger entries (2–3 phases each, spanning all
      three executor kinds, with one migration phase flagged requiresApproval=true).
    "DECIDE_PHASE" → 5 PhaseNextStep entries covering Continue (advancing
      through compat-check → approval-gated migration → test-run), Replan
      (adding a rollback phase), Complete, Fail.
    "COMPOSE_REPORT" → 3 UpgradeReport entries with 60–120 word summaries.
- compatibility-checker.json — 5 PhaseResult entries, ok=true, content
  describes which providers and DAGs are safe for the target version.
- test-runner.json — 5 PhaseResult entries; at least two have
  testReport.failed > 0 and a non-empty failingTests list (for J4 coverage).
- migration-applier.json — 5 PhaseResult entries with migrationOutcome
  populated; one entry has applied=false and an errorReason (for replan path).

Constraints:
- Lesson 1: AutonomousAgent base class — extends AutonomousAgent for all four agents.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  plannerDecideStep, approvalGateStep, dispatchStep, ciGateStep, composeReportStep.
- Lesson 6: Optional<T> for all nullable fields.
- Lesson 7: Companion PlannerTasks.java and ExecutorTasks.java declaring every Task<R>.
- Lesson 8: Conservative model defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9335 in application.conf.
- Lesson 11: source.platform never in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key written to disk.
- Lesson 26: Tab switching by data-tab / data-panel attribute only.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
