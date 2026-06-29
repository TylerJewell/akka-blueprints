# SPEC — sre-incident-responder

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SRE Incident Response Agent.
**One-line pitch:** Submit an alert; an IncidentCommander triages the signal, dispatches telemetry and runbook probes on an investigation ledger, proposes a remediation action, waits for SRE approval, executes the approved action, then produces a post-incident report.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to incident response. The IncidentCommander owns two ledgers — an **investigation ledger** (known facts, open hypotheses, probe plan, current probe dispatch) and a **remediation ledger** (proposed actions, their approval status, execution outcomes, lessons learned). Each loop iteration the commander reads both ledgers, picks the next probe or action, and decides to `CONTINUE_INVESTIGATION`, `PROPOSE_REMEDIATION`, `REPLAN_INVESTIGATION`, `COMPOSE_REPORT`, or `FAIL`.

The blueprint demonstrates four governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each remediation action against an allow-list and an impact policy before the `RemediationAgent` executes it,
- a **human-in-the-loop approval gate** that pauses the workflow and requires an SRE to accept or reject every proposed remediation,
- an **automatic safety halt** that trips when a post-execution evaluator detects evidence of an uncontrolled blast radius or a failed safety precondition,
- a **post-incident eval-event** that scores the commander's investigation quality and remediation accuracy after the incident closes.

## 3. User-facing flows

The user opens the App UI tab and submits an alert description via the form.

1. The system creates an `Incident` record in `TRIAGING` and starts an `IncidentWorkflow`.
2. The IncidentCommander drafts an `InvestigationLedger { facts, hypotheses, probePlan, currentProbe }` and emits `IncidentTriaged`.
3. The workflow enters the probe loop. Each iteration:
   - IncidentCommander reads both ledgers and proposes a `ProbeDecision { probeKind, target, rationale }`.
   - The **before-tool-call guardrail** vets investigation probes; on rejection the workflow records a `ProbeBlocked` entry and asks the commander to revise.
   - The chosen probe agent (`TelemetryAgent` or `RunbookAgent`) runs and returns a typed `ProbeResult`.
   - The workflow appends a `ProbeEntry { probeKind, target, attempt, verdict, result }` to the investigation ledger.
   - The IncidentCommander decides: `CONTINUE_INVESTIGATION`, `REPLAN_INVESTIGATION`, `PROPOSE_REMEDIATION`, or `FAIL`.
4. When the commander emits `PROPOSE_REMEDIATION`, the workflow transitions to the approval step:
   - `ApprovalEntity` is created with the proposed `RemediationAction`.
   - The **HITL gate** pauses the workflow; the SRE sees the proposal in the App UI and clicks Accept or Reject.
   - On Accept: the **before-tool-call guardrail** re-vets the action; on pass, `RemediationAgent` executes it.
   - The **auto safety halt** evaluator inspects the execution outcome; on unsafe signal it emits `IncidentHaltedAutomatic`.
   - On success, the workflow records a `RemediationOutcome` and the commander decides whether to `COMPOSE_REPORT` or continue investigating.
   - On Reject: the commander sees the rejection reason and either replans or fails.
5. When the commander emits `COMPOSE_REPORT`, the workflow calls the IncidentCommander in `COMPOSE_REPORT` mode and emits `IncidentMitigated`.
6. The **post-incident eval-event** fires after the incident reaches a terminal state, scoring investigation coverage, hypothesis accuracy, and time-to-mitigate.
7. The operator can click **Halt new probes** at any time. The in-flight probe finishes; the loop exits with `IncidentHaltedOperator`.

An `AlertSimulator` (TimedAction) drips a sample alert every 120 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `IncidentCommanderAgent` | `AutonomousAgent` | Triages, plans investigation, decides next probe, proposes remediation, composes post-incident report. | `IncidentWorkflow` | returns typed result to workflow |
| `TelemetryAgent` | `AutonomousAgent` | Retrieves metric time-series, log excerpts, and trace summaries from seeded fixtures. | `IncidentWorkflow` | — |
| `RunbookAgent` | `AutonomousAgent` | Looks up runbook procedures and config baselines from fixture files. | `IncidentWorkflow` | — |
| `RemediationAgent` | `AutonomousAgent` | Executes an approved remediation action against the simulated control plane. Returns an `ActionOutcome`. | `IncidentWorkflow` | — |
| `IncidentWorkflow` | `Workflow` | Drives triage → probe-guarded-loop → approve → remediate → auto-halt-eval → decide → compose-report chain, plus operator halt and timeout branches. | `IncidentEndpoint`, `AlertRequestConsumer` | `IncidentEntity`, `ApprovalEntity` |
| `IncidentEntity` | `EventSourcedEntity` | Holds the incident lifecycle, investigation ledger, remediation ledger, and post-incident report. | `IncidentWorkflow` | `IncidentView` |
| `ApprovalEntity` | `EventSourcedEntity` | Holds the pending approval request and the SRE's decision. Keyed by `incidentId`. | `IncidentWorkflow`, `IncidentEndpoint` | `IncidentWorkflow` (polls) |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `IncidentEndpoint` (operator action) | `IncidentWorkflow` (polls) |
| `AlertQueue` | `EventSourcedEntity` | Audit log of submitted alerts. | `IncidentEndpoint`, `AlertSimulator` | `AlertRequestConsumer` |
| `IncidentView` | `View` | List-of-incidents read model for the UI. | `IncidentEntity` events | `IncidentEndpoint` |
| `AlertRequestConsumer` | `Consumer` | Subscribes to `AlertQueue` events; starts an `IncidentWorkflow` per submission. | `AlertQueue` events | `IncidentWorkflow` |
| `AlertSimulator` | `TimedAction` | Every 120 s, reads a line from `sample-events/alert-prompts.jsonl` and enqueues it. | scheduler | `AlertQueue` |
| `StaleIncidentMonitor` | `TimedAction` | Every 60 s, marks any incident stuck in `INVESTIGATING` past 10 minutes as `TIMED_OUT`. | scheduler | `IncidentEntity` |
| `IncidentEndpoint` | `HttpEndpoint` | `/api/incidents/*` — submit, get, list, SSE, approval decision, operator halt. | — | `IncidentView`, `AlertQueue`, `IncidentEntity`, `ApprovalEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record AlertRequest(String description, String severity, String reportedBy) {}

record InvestigationLedger(
    List<String> facts,
    List<String> hypotheses,
    List<String> probePlan,
    Optional<ProbeDecision> currentProbe
) {}

record ProbeDecision(
    ProbeKind probeKind,
    String target,
    String rationale
) {}

record ProbeResult(
    ProbeKind probeKind,
    String target,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record ProbeEntry(
    int attempt,
    ProbeKind probeKind,
    String target,
    ProbeVerdict verdict,
    String result,
    Optional<String> blocker,
    Instant recordedAt
) {}

record RemediationAction(
    ActionKind actionKind,
    String target,
    String parameters,
    String rationale,
    ImpactLevel estimatedImpact
) {}

record ApprovalRequest(
    String incidentId,
    RemediationAction action,
    Instant requestedAt,
    Optional<String> sreNote
) {}

record ApprovalDecision(
    boolean approved,
    String decidedBy,
    String reason,
    Instant decidedAt
) {}

record ActionOutcome(
    ActionKind actionKind,
    String target,
    boolean succeeded,
    String observedEffect,
    Optional<String> errorDetail,
    Instant executedAt
) {}

record RemediationLedger(
    List<RemediationAction> proposedActions,
    Optional<ApprovalDecision> lastDecision,
    List<ActionOutcome> outcomes
) {}

record PostIncidentReport(
    String summary,
    String rootCauseDiagnosis,
    List<String> timeline,
    List<String> lessonsLearned,
    Optional<String> followUpActions,
    Instant producedAt
) {}

record EvalScore(
    String incidentId,
    int investigationCoverageScore,
    int hypothesisAccuracyScore,
    int timeToMitigateMinutes,
    List<String> findings,
    Instant scoredAt
) {}

record Incident(
    String incidentId,
    String description,
    String severity,
    IncidentStatus status,
    Optional<InvestigationLedger> investigationLedger,
    Optional<RemediationLedger> remediationLedger,
    Optional<PostIncidentReport> report,
    Optional<EvalScore> evalScore,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ProbeKind { METRICS, LOGS, TRACES, RUNBOOK }
enum ProbeVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum ActionKind { RESTART_SERVICE, ROLLBACK_DEPLOYMENT, SHIFT_TRAFFIC, SCALE_UP, DISABLE_FEATURE_FLAG }
enum ImpactLevel { LOW, MEDIUM, HIGH, CRITICAL }
enum IncidentStatus { TRIAGING, INVESTIGATING, AWAITING_APPROVAL, REMEDIATING, MITIGATED, UNRESOLVED, HALTED, TIMED_OUT }
```

### Events (`IncidentEntity`)

`IncidentCreated`, `IncidentTriaged`, `ProbeDispatched`, `ProbeBlocked`, `ProbeRecorded`, `InvestigationReplanned`, `RemediationProposed`, `ApprovalGranted`, `ApprovalRejected`, `RemediationExecuted`, `IncidentMitigated`, `IncidentUnresolved`, `IncidentHaltedAutomatic`, `IncidentHaltedOperator`, `IncidentTimedOut`, `EvalScoreRecorded`.

### Events (`ApprovalEntity`)

`ApprovalRequested { incidentId, action, requestedAt }`, `ApprovalDecided { incidentId, decision, decidedAt }`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`AlertQueue`)

`AlertSubmitted { incidentId, description, severity, reportedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/incidents` — body `{ description, severity, reportedBy? }` → `202 { incidentId }`. Starts a workflow.
- `GET /api/incidents` — list all incidents. Optional `?status=...`.
- `GET /api/incidents/{id}` — one incident (full ledgers + report + eval).
- `GET /api/incidents/sse` — server-sent events stream of every incident change.
- `POST /api/incidents/{id}/approve` — body `{ approved, decidedBy, reason }` → `200`. Records the SRE decision on `ApprovalEntity`.
- `GET /api/incidents/{id}/approval` — `{ pending, action?, requestedAt?, decision? }`.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "SRE Incident Response Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an alert with severity selector, SRE approval pane, operator halt/resume control, live list of incidents with status pills, expand-row to see both ledgers, pending approval actions, execution outcomes, the post-incident report, and the eval score.

Browser title: `<title>Akka Sample: SRE Incident Response Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `IncidentCommanderAgent` and `RemediationAgent`): every `ProbeDecision` is checked against (a) the probe-kind allow-list, (b) a probe-scope policy that prevents reads outside `sample-data/`; every `RemediationAction` is checked against (a) the action-kind allow-list (`RESTART_SERVICE`, `ROLLBACK_DEPLOYMENT`, `SHIFT_TRAFFIC`, `SCALE_UP`, `DISABLE_FEATURE_FLAG`), (b) an impact policy that blocks `CRITICAL`-rated actions without explicit override, (c) a target-scope check that rejects actions on out-of-scope service names. Blocking. Failure → `ProbeBlocked` entry + replan.
- **HT1 — human-in-the-loop approval** (`hitl`, flavor `application`): after the commander emits `PROPOSE_REMEDIATION`, the workflow records the proposed action on `ApprovalEntity` and pauses in `AWAITING_APPROVAL`. The SRE sees the pending action in the App UI approval pane and either approves or rejects via `POST /api/incidents/{id}/approve`. The workflow resumes when `ApprovalEntity` records an `ApprovalDecided` event. Rejection routes back to the investigation loop; acceptance routes to the remediation step. The wait timeout is 30 minutes; expiry transitions the incident to `UNRESOLVED`.
- **HT2 — automatic safety halt** (`halt`, flavor `automatic-safety-halt`): after each `ActionOutcome` is recorded, a deterministic `SafetyEvaluator.evaluate(outcome)` runs. On an unsafe signal (e.g., `ROLLBACK_DEPLOYMENT` reports an unexpected cascade failure, `SHIFT_TRAFFIC` reports 100% traffic shift without a canary), the workflow emits `IncidentHaltedAutomatic` and ends.
- **E1 — post-incident eval-event** (`eval-event`, flavor `on-incident-reporter`): after `IncidentMitigated` or `IncidentUnresolved`, the workflow invokes `IncidentCommanderAgent` in `SCORE_INCIDENT` mode. The agent produces an `EvalScore { investigationCoverageScore, hypothesisAccuracyScore, timeToMitigateMinutes, findings }`. The workflow emits `EvalScoreRecorded` and stores the score on `IncidentEntity`.

## 9. Agent prompts

- `IncidentCommanderAgent` → `prompts/incident-commander.md`. Triages, plans, decides next probe, proposes remediation, composes report, and scores the incident.
- `TelemetryAgent` → `prompts/telemetry.md`. Returns metric, log, and trace excerpts.
- `RunbookAgent` → `prompts/runbook.md`. Returns runbook procedures and config baselines.
- `RemediationAgent` → `prompts/remediation.md`. Executes approved actions and returns outcome.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "CPU spike on payment-service: p99 latency 4.2 s, error rate 12%." Incident progresses `TRIAGING → INVESTIGATING → AWAITING_APPROVAL → REMEDIATING → MITIGATED` within ~5 minutes. Both ledgers populate; the post-incident report includes a root-cause diagnosis and at least two lessons-learned bullets; an `EvalScore` record is attached.
2. **J2** — Submit an incident whose proposed remediation would be an out-of-policy database operation. The guardrail rejects the `RemediationAction`; a `ProbeBlocked` entry appears; the commander revises the proposal to a `RESTART_SERVICE` action.
3. **J3** — Submit an incident, wait for the approval pane to appear, click Reject with a reason. The workflow replans; the commander proposes an alternate action or exhausts its budget and ends in `UNRESOLVED`.
4. **J4** — Submit an incident and click **Halt new probes** while the incident is `INVESTIGATING`. The in-flight telemetry probe finishes; no further probes dispatch; the incident ends in `HALTED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sre-incident-responder demonstrating the
planner-executor × ops-automation cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-ops-automation-sre-incident-responder.
Java package io.akka.samples.sreincidentresponseagent. Akka 3.6.0. HTTP port 9674.

Components to wire (exactly):
- 4 AutonomousAgents:
  * IncidentCommanderAgent — definition() with capabilities:
      capability(TaskAcceptance.of(TRIAGE).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(INVESTIGATE_DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(PROPOSE_REMEDIATION).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(SCORE_INCIDENT).maxIterationsPerTask(2)).
    System prompt from prompts/incident-commander.md.
    TRIAGE returns InvestigationLedger.
    INVESTIGATE_DECIDE returns a NextStep tagged union
      (ContinueInvestigation(ProbeDecision) | ReplanInvestigation(InvestigationLedger) |
       ProposeRemediation(RemediationAction) | FailInvestigation(String)).
    PROPOSE_REMEDIATION returns RemediationAction.
    COMPOSE_REPORT returns PostIncidentReport.
    SCORE_INCIDENT returns EvalScore.
  * TelemetryAgent — capabilities for FETCH_METRICS, FETCH_LOGS, FETCH_TRACES
    (each maxIterationsPerTask(2)). Prompt from prompts/telemetry.md. Returns ProbeResult.
  * RunbookAgent — capability(TaskAcceptance.of(LOOKUP_RUNBOOK).maxIterationsPerTask(2)).
    Prompt from prompts/runbook.md. Returns ProbeResult.
  * RemediationAgent — capability(TaskAcceptance.of(EXECUTE_ACTION).maxIterationsPerTask(2)).
    Prompt from prompts/remediation.md. Returns ActionOutcome.

- 1 Workflow IncidentWorkflow with steps:
  triageStep -> [probe loop entry] checkHaltStep -> proposeProbeStep ->
  probeGuardrailStep -> dispatchProbeStep -> recordProbeStep ->
  investigateDecideStep -> [back to checkHaltStep | to proposeRemediationStep]
  -> proposeRemediationStep -> remediationGuardrailStep ->
  awaitApprovalStep -> approvalRoutingStep
    -> [on approve] executeRemediationStep -> safetyEvalStep -> postRemediateDecideStep
    -> [compose | replan | fail]
    -> [on reject] replanOrFailStep
  -> composeReportStep -> scoreIncidentStep -> [end MITIGATED]
  Terminal branches: haltedStep (HALTED), unresolvedStep (UNRESOLVED), timedOutStep (TIMED_OUT).
  Step timeouts (override settings() per Lesson 4):
    triageStep ofSeconds(60), proposeProbeStep ofSeconds(45),
    dispatchProbeStep ofSeconds(90), investigateDecideStep ofSeconds(45),
    proposeRemediationStep ofSeconds(60), awaitApprovalStep ofMinutes(30),
    executeRemediationStep ofSeconds(120), composeReportStep ofSeconds(90),
    scoreIncidentStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(IncidentWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits IncidentHaltedOperator on IncidentEntity).
  probeGuardrailStep runs ProbeGuardrail.vet(ProbeDecision); on reject records
  ProbeBlocked entry via IncidentEntity.recordProbeBlock and loops back to proposeProbeStep.
  remediationGuardrailStep runs RemediationGuardrail.vet(RemediationAction); on reject
  records a RemediationBlocked entry and routes to replanOrFailStep.
  awaitApprovalStep polls ApprovalEntity.getDecision until decided or timeout.
  executeRemediationStep calls RemediationAgent via
    forAutonomousAgent(...).runSingleTask(EXECUTE_ACTION).
  safetyEvalStep runs SafetyEvaluator.evaluate(ActionOutcome); on unsafe transitions
  to haltedStep (emits IncidentHaltedAutomatic).
  scoreIncidentStep calls IncidentCommanderAgent(SCORE_INCIDENT); emits EvalScoreRecorded.

- 1 EventSourcedEntity IncidentEntity holding Incident state. emptyState() returns
  Incident.initial("", null) with no commandContext() reference. Commands:
  createIncident, triageIncident, recordProbeDispatch, recordProbeBlock,
  recordProbeEntry, replanInvestigation, proposeRemediation, grantApproval,
  rejectApproval, recordRemediationOutcome, mitigateIncident, failIncident,
  haltAutomatic, haltOperator, timeoutIncident, recordEvalScore, getIncident.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity ApprovalEntity keyed by incidentId. State
  ApprovalState { Optional<ApprovalRequest> pendingRequest,
  Optional<ApprovalDecision> decision }. Commands:
  requestApproval(incidentId, action), recordDecision(approved, decidedBy, reason),
  getDecision. Events: ApprovalRequested, ApprovalDecided.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant> haltedAt}.
  Commands: requestHalt(reason), clearHalt, get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity AlertQueue with command enqueueAlert(incidentId, description,
  severity, reportedBy) emitting AlertSubmitted.

- 1 View IncidentView with row type IncidentRow (mirror of Incident minus heavy ledger
  payloads — truncate investigationLedger probe entries to last 3 plus counts; the UI
  fetches the full incident by id on click). Table updater consumes IncidentEntity events.
  ONE query getAllIncidents SELECT * AS incidents FROM incident_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer AlertRequestConsumer subscribed to AlertQueue events; on AlertSubmitted
  starts an IncidentWorkflow with incidentId as the workflow id.

- 2 TimedActions:
  * AlertSimulator — every 120s, reads next line from
    src/main/resources/sample-events/alert-prompts.jsonl and calls
    AlertQueue.enqueueAlert.
  * StaleIncidentMonitor — every 60s, queries IncidentView.getAllIncidents, filters
    INVESTIGATING incidents whose createdAt is older than 10 minutes, calls
    IncidentEntity.timeoutIncident; IncidentWorkflow's investigateDecideStep detects
    TIMED_OUT status and exits via timedOutStep.

- 2 HttpEndpoints:
  * IncidentEndpoint at /api with POST /incidents, GET /incidents (filters client-side
    from getAllIncidents), GET /incidents/{id}, GET /incidents/sse,
    POST /incidents/{id}/approve, GET /incidents/{id}/approval,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- CommanderTasks.java declaring five Task<R> constants: TRIAGE (resultConformsTo
  InvestigationLedger), INVESTIGATE_DECIDE (InvestigationNextStep), PROPOSE_REMEDIATION
  (RemediationAction), COMPOSE_REPORT (PostIncidentReport), SCORE_INCIDENT (EvalScore).
- ProbeTasks.java declaring four Task<R> constants: FETCH_METRICS, FETCH_LOGS,
  FETCH_TRACES, LOOKUP_RUNBOOK (all resultConformsTo ProbeResult).
- RemediationTasks.java declaring one Task<R> constant: EXECUTE_ACTION (resultConformsTo ActionOutcome).
- Domain records as listed in SPEC §5, plus InvestigationNextStep sealed interface with
  permits ContinueInvestigation(ProbeDecision), ReplanInvestigation(InvestigationLedger),
  ProposeRemediation(RemediationAction), FailInvestigation(String reason).
- application/ProbeGuardrail.java — deterministic vetter for probe decisions.
  Reject if probeKind is not in the allow-list (METRICS, LOGS, TRACES, RUNBOOK), if
  a METRICS target does not match `^[a-z0-9_\-]+$`, if a RUNBOOK target path escapes
  `sample-data/runbooks/`.
- application/RemediationGuardrail.java — deterministic vetter for remediation actions.
  Reject if actionKind is not in the allow-list, if estimatedImpact is CRITICAL, if
  the target service name contains characters outside `^[a-z0-9_\-]+$`.
- application/SafetyEvaluator.java — post-execution check on ActionOutcome.
  Flag UNSAFE on ROLLBACK_DEPLOYMENT outcomes whose observedEffect contains
  "cascade" or "unrecoverable", on SHIFT_TRAFFIC outcomes reporting 100% shift
  without "canary" in the parameters, on any outcome whose errorDetail contains
  "timeout" and actionKind is RESTART_SERVICE (restart storm risk).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9674 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/alert-prompts.jsonl with 8 canned alert descriptions
  spanning CPU spikes, memory pressure, HTTP 5xx surges, disk-full, deployment regressions,
  network partition events, dependency timeouts, and certificate expiry.
- src/main/resources/sample-data/metrics/*.jsonl — 6 metric fixture files covering
  CPU, memory, request rate, error rate, latency percentiles, and disk usage.
- src/main/resources/sample-data/logs/*.jsonl — 4 log fixture files: application errors,
  infra errors, access logs, and audit logs.
- src/main/resources/sample-data/traces/*.jsonl — 3 trace fixture files covering
  payment flow, auth flow, and database query spans.
- src/main/resources/sample-data/runbooks/*.md — 5 runbook files: service-restart,
  rollback-deployment, traffic-shift, scale-up, disable-feature-flag. Include one
  runbook whose content references a deprecated API endpoint pattern for the guardrail test.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the project-root files for the metadata endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls (G1, HT1, HT2, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions, failure,
  oversight, operations.external_tool_calls, and compliance.capabilities; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/incident-commander.md, prompts/telemetry.md, prompts/runbook.md,
  prompts/remediation.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: SRE Incident Response Agent",
  one-line pitch, prerequisites (integration form: None), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — single self-contained HTML file.
  Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Browser title
  exactly: <title>Akka Sample: SRE Incident Response Agent</title>. No subtitle on
  the Overview tab. App UI includes: alert submit form (description textarea + severity
  selector + Submit button), SRE approval pane (shows pending action when
  AWAITING_APPROVAL; Accept/Reject buttons with reason text field), operator halt pane,
  live incident list with expand-on-click for both ledgers, approval state,
  execution outcomes, post-incident report, and eval score badge.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider returning shape-correct
        outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Per-agent mock-response shapes:
    incident-commander.json — four lists keyed by task id:
      "TRIAGE" → 4–6 InvestigationLedger entries (hypotheses spanning CPU contention,
      memory leak, noisy neighbour, bad deployment).
      "INVESTIGATE_DECIDE" → 4–6 InvestigationNextStep entries covering
      ContinueInvestigation (spanning all four probe kinds), ReplanInvestigation,
      ProposeRemediation, FailInvestigation.
      "COMPOSE_REPORT" → 4–6 PostIncidentReport entries with 80–140 word summaries,
      root-cause diagnoses, 3–5 timeline entries, and 2–4 lessons-learned bullets.
      "SCORE_INCIDENT" → 4–6 EvalScore entries with realistic scores and findings.
    telemetry.json — 8 ProbeResult entries: METRICS (CPU, memory, latency spikes),
      LOGS (error bursts, OOM events), TRACES (high-latency spans).
    runbook.json — 6 ProbeResult entries with runbook procedure excerpts; one entry's
      content includes a reference to "deprecated endpoint /v1/restart" so the
      guardrail can be exercised in tests.
    remediation.json — 6 ActionOutcome entries covering RESTART_SERVICE (success),
      ROLLBACK_DEPLOYMENT (success, one with "cascade" in observedEffect to trigger
      the safety evaluator), SHIFT_TRAFFIC (success, one with "100% shift" without
      "canary"), SCALE_UP (success), DISABLE_FEATURE_FLAG (success).

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent base class required for all four agents.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on IncidentRow and on the Incident
  entity state.
- Lesson 7: Companion CommanderTasks.java, ProbeTasks.java, RemediationTasks.java
  declaring every Task<R> constant.
- Lesson 8: Conservative model defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9674 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata files.
- Lesson 24: CSS overrides + themeVariables in index.html.
- Lesson 25: API-key sourcing follows the five-option flow; no key value written to disk.
- Lesson 26: Tab switching by data-tab / data-panel attribute; no zombie panels.
- Overview tab Try-it card shows just "/akka:build".
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
