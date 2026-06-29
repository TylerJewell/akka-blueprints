# SPEC — durable-workflow-backed-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Ops Remediation Agent.
**One-line pitch:** A user submits an ops incident; one `OpsAgent` walks it through three task phases — **DIAGNOSE** root cause from metrics and logs, **REMEDIATE** by applying corrective actions, **VERIFY** the fix held — with each phase gated on the prior phase's typed output, a budget-cap guardrail halting runaway tool loops, and a scheduled evaluator monitoring workflow durability metrics across all incidents.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an ops-automation domain, with the workflow engine as the durability primitive. One `OpsAgent` (AutonomousAgent) carries every LLM call. The pattern's defining property is the **typed task handoff**: the DIAGNOSE task's `DiagnosisReport` becomes the REMEDIATE task's instruction context; the REMEDIATE task's `RemediationPlan` (together with the prior `DiagnosisReport`) becomes the VERIFY task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` budget-cap guardrail** sits between the agent and every tool call. It counts tool calls per task against a per-phase cap (`DIAGNOSE` cap: 8, `REMEDIATE` cap: 6, `VERIFY` cap: 6). When the cap is reached, the guardrail returns a structured `budget-cap-exceeded` halt to the agent loop and records a `BudgetExhausted{phase, tool, callsUsed, cap}` event on `IncidentEntity`. The agent cannot call any further tools in this task; the workflow step either succeeds with what the agent has already accumulated or exhausts its step-recovery budget and transitions to `FAILED`.
- A **`eval-periodic` scheduled evaluator** runs on a TimerService-registered hourly timer. `DurabilityEvaluator` queries `IncidentView` for incidents whose lifecycle closed in the trailing one-hour window, computes four durability metrics (completion rate, budget-violation rate, MTTR, timeout rate), and writes a `DurabilityReport` to `DurabilityReportEntity`. The App UI Overview panel renders the latest report.

The blueprint shows that the workflow's durability properties — retries, timeouts, failover steps — are not just recovery plumbing; they are a governance primitive that the budget-cap guardrail and the scheduled evaluator can observe and surface.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks an incident from the **Pick a seeded incident** dropdown (or types a custom `alertId`) and clicks **Remediate**. The UI POSTs to `/api/incidents` and receives an `incidentId`.
2. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `DIAGNOSING` — the workflow has started `diagnoseStep` and the agent has been handed the DIAGNOSE task.
3. Within ~10–20 s the card reaches `DIAGNOSED` — the typed `DiagnosisReport` is visible in the card detail (affected service, root cause, evidence metrics table, log evidence table).
4. Within ~10–20 s more the card reaches `REMEDIATED`. The `RemediationPlan` is visible (actions applied, outcome).
5. Within ~10–20 s more the card reaches `VERIFIED`. The right pane shows the full typed `VerificationResult` — health checks, resolution confirmation — plus a budget-violation strip if any cap was hit.
6. The hourly durability report (or a manually triggered evaluation via the test endpoint) appears at the bottom of the Overview tab as a metrics panel.
7. The user can submit another incident; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RemediationEndpoint` | `HttpEndpoint` | `/api/incidents/*` — submit, list, get, SSE; `/api/durability/*` — latest report, trigger eval; `/api/metadata/*`. | — | `IncidentEntity`, `IncidentView`, `DurabilityReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `IncidentEntity` | `EventSourcedEntity` | Per-incident lifecycle: CREATED → DIAGNOSING → DIAGNOSED → REMEDIATING → REMEDIATED → VERIFYING → VERIFIED → FAILED. Source of truth. | `RemediationEndpoint`, `RemediationWorkflow` | `IncidentView` |
| `DurabilityReportEntity` | `EventSourcedEntity` | Holds the latest durability report and a rolling log of past reports. | `DurabilityEvaluator` | `IncidentView` (via separate query) |
| `RemediationWorkflow` | `Workflow` | One workflow per incidentId. Steps: `diagnoseStep` → `remediateStep` → `verifyStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `RemediationEndpoint` after `CREATED` | `OpsAgent`, `IncidentEntity` |
| `OpsAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `OpsTasks.java`: `DIAGNOSE_INCIDENT` → `DiagnosisReport`, `APPLY_REMEDIATION` → `RemediationPlan`, `VERIFY_FIX` → `VerificationResult`. Function tools are ALL registered on the agent; phase gating is the job of `BudgetGuardrail`. | invoked by `RemediationWorkflow` | returns typed results |
| `DiagnoseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `queryMetrics(serviceId, windowMinutes)` and `fetchLogs(serviceId, level)`. Reads from `src/main/resources/sample-data/incidents/*.json`. | called from DIAGNOSE task | returns `List<MetricSample>` / `List<LogEntry>` |
| `RemediateTools` | function-tools class | Implements `applyAction(action, target)` and `confirmAction(receiptId)`. Pure in-memory with deterministic mock receipts. | called from REMEDIATE task | returns `ActionReceipt` / `ActionStatus` |
| `VerifyTools` | function-tools class | Implements `checkHealth(serviceId)` and `confirmResolved(incidentId)`. | called from VERIFY task | returns `HealthCheck` / `ResolutionCheck` |
| `BudgetGuardrail` | `before-tool-call` guardrail (registered on `OpsAgent`) | Counts tool calls per task against the per-phase cap. Rejects the call and records `BudgetExhausted` when cap is reached. | every tool call | accept / halt-reject |
| `DurabilityEvaluator` | plain class + TimerService action | Runs on an hourly timer. Queries `IncidentView` for closed incidents in the trailing window, computes four metrics, writes `DurabilityReport` to `DurabilityReportEntity`. | `IncidentView` | `DurabilityReportEntity` |
| `IncidentView` | `View` | Read model: one row per incident for the UI and for `DurabilityEvaluator`. | `IncidentEntity` events | `RemediationEndpoint`, `DurabilityEvaluator` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MetricSample(String metricName, double value, String unit, Instant sampledAt) {}

record LogEntry(String level, String message, String serviceId, Instant timestamp) {}

record DiagnosisReport(
    String rootCause,
    String affectedService,
    String severity,           // "critical" | "high" | "medium"
    List<MetricSample> evidenceMetrics,
    List<LogEntry> evidenceLogs,
    Instant diagnosedAt
) {}

record ActionReceipt(String receiptId, String action, String target, Instant issuedAt) {}

record ActionStatus(String receiptId, String state, Instant confirmedAt) {}

record RemediationPlan(
    List<ActionReceipt> actionsApplied,
    String outcome,            // "applied" | "partial" | "rolled-back"
    Instant remediatedAt
) {}

record HealthCheck(String serviceId, boolean healthy, double latencyP99Ms, Instant checkedAt) {}

record ResolutionCheck(String incidentId, boolean resolved, String evidence, Instant checkedAt) {}

record VerificationResult(
    boolean resolved,
    List<HealthCheck> healthChecks,
    ResolutionCheck resolutionCheck,
    Instant verifiedAt
) {}

record BudgetViolation(
    String phase,
    String tool,
    int callsUsed,
    int cap,
    Instant violatedAt
) {}

record IncidentRecord(
    String incidentId,
    Optional<String> alertId,
    Optional<String> serviceId,
    Optional<DiagnosisReport> diagnosis,
    Optional<RemediationPlan> remediation,
    Optional<VerificationResult> verification,
    IncidentStatus status,
    Instant createdAt,
    Optional<Instant> closedAt,
    List<BudgetViolation> budgetViolations
) {}

enum IncidentStatus {
    CREATED, DIAGNOSING, DIAGNOSED, REMEDIATING, REMEDIATED, VERIFYING, VERIFIED, FAILED
}

record DurabilityReport(
    String reportId,
    Instant windowStart,
    Instant windowEnd,
    int incidentsEvaluated,
    double completionRatePercent,
    double budgetViolationRatePercent,
    double mttrMinutes,
    double timeoutRatePercent,
    Instant generatedAt
) {}
```

Events on `IncidentEntity`: `IncidentCreated`, `DiagnoseStarted`, `DiagnosisCompleted`, `RemediateStarted`, `RemediationCompleted`, `VerifyStarted`, `VerificationCompleted`, `BudgetExhausted`, `IncidentFailed`.

Every nullable lifecycle field on `IncidentRecord` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/incidents` — body `{ alertId, serviceId }` → `{ incidentId }`.
- `GET /api/incidents` — list all incidents, newest-first.
- `GET /api/incidents/{id}` — one incident.
- `GET /api/incidents/sse` — Server-Sent Events; one event per state transition.
- `GET /api/durability/latest` — latest `DurabilityReport`.
- `POST /api/durability/trigger` — manually trigger `DurabilityEvaluator` (for testing).
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Durable Workflow-Backed Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of incidents (status pill + alertId + age + red dot if any budget violation fired) and a right pane with the selected incident's detail — affected service, diagnosis root cause, evidence tables, remediation actions, verification health checks, and a budget-violation strip if any cap was hit.

The Overview tab's lower section renders the latest `DurabilityReport` as a four-metric tile row (completion rate, budget-violation rate, MTTR, timeout rate).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `before-tool-call` budget-cap guardrail**: `BudgetGuardrail` is registered on `OpsAgent` and runs before every tool call. It maintains a per-task counter keyed by `(incidentId, taskName)` in a lightweight in-process registry initialised from the `TaskDef.metadata` (`incidentId`, `phase`). Caps: `DIAGNOSE` phase allows 8 tool calls; `REMEDIATE` allows 6; `VERIFY` allows 6. When the counter reaches the cap, the guardrail returns a structured `budget-cap-exceeded: <tool> halted at call <n>/<cap> for phase <phase>` halt to the agent loop and calls `IncidentEntity.recordBudgetExhausted(phase, tool, callsUsed, cap)`. The agent cannot call any further tools in this task; the workflow step either succeeds with whatever the agent has accumulated or the step's default recovery (`maxRetries(2).failoverTo(RemediationWorkflow::error)`) exhausts its budget and the entity transitions to `FAILED`.
- **E1 — `eval-periodic` scheduled evaluator**: `DurabilityEvaluator` is triggered by a TimerService-registered hourly timer (`DurabilityTimerAction`). On each fire, the evaluator calls `RemediationEndpoint`'s internal query for incidents closed in the trailing 60-minute window, computes: (1) completion rate = `VERIFIED` incidents / total closed; (2) budget-violation rate = incidents with ≥ 1 `BudgetExhausted` event / total; (3) MTTR = mean of `(closedAt - createdAt)` for `VERIFIED` incidents, in minutes; (4) timeout rate = `FAILED` incidents attributable to step-timeout / total. The result is written to `DurabilityReportEntity` as a `DurabilityReportRecorded` event. A manual trigger endpoint (`POST /api/durability/trigger`) fires the same logic immediately, making the evaluation testable without waiting for the hourly window.

## 9. Agent prompts

- `OpsAgent` → `prompts/ops-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, call only tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded incident `high-latency-api-gateway`; within 60 s the incident reaches `VERIFIED` with a non-empty `DiagnosisReport`, a non-empty `RemediationPlan`, at least one `HealthCheck` showing `healthy = true`, and `resolutionCheck.resolved = true`.
2. **J2** — A DIAGNOSE task that is fed the mock LLM's budget-exceeding trajectory generates a `BudgetExhausted` event. `BudgetGuardrail` fires; no further tool calls are made; the workflow retries the step once. If the retry succeeds, the incident resolves. The App UI shows the budget-violation strip on the right pane for that incident.
3. **J3** — `POST /api/durability/trigger` fires `DurabilityEvaluator` manually. The App UI Overview panel refreshes to show a four-metric tile row with completion rate, budget-violation rate, MTTR, and timeout rate computed over all incidents evaluated so far.
4. **J4** — Each task's tool calls are visible in the per-incident trace (logged at `INFO`); the DIAGNOSE task's log shows only DIAGNOSE-tool calls (`queryMetrics`, `fetchLogs`), the REMEDIATE task's log shows only `applyAction` and `confirmAction`, the VERIFY task's log shows only `checkHealth` and `confirmResolved`. The trace is empirical evidence the sequential dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named durable-workflow-backed-agent demonstrating the sequential-pipeline x
ops-automation cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-ops-automation-durable-workflow-agent. Java package
io.akka.samples.durableworkflowbackedagent. Akka 3.6.0. HTTP port 9731.

Components to wire (exactly):

- 1 AutonomousAgent OpsAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/ops-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  DIAGNOSE, REMEDIATE, and VERIFY tool sets are ALL registered on the agent; budget gating is
  the job of BudgetGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (BudgetGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail halt the agent loop cannot issue further tool
  calls in this task; it must return with whatever typed output it has accumulated.

- 1 Workflow RemediationWorkflow per incidentId with three steps plus an error step:
  * diagnoseStep — emits DiagnoseStarted on the entity, then calls componentClient
    .forAutonomousAgent(OpsAgent.class, "agent-" + incidentId).runSingleTask(
      TaskDef.instructions("AlertId: " + alertId + "\nServiceId: " + serviceId +
      "\nPhase: DIAGNOSE\nUse queryMetrics and fetchLogs to identify the root cause.")
        .metadata("incidentId", incidentId)
        .metadata("phase", "DIAGNOSE")
        .taskType(OpsTasks.DIAGNOSE_INCIDENT)
    ). Reads forTask(taskId).result(DIAGNOSE_INCIDENT) to get DiagnosisReport. Writes
    IncidentEntity.recordDiagnosis(diagnosis). WorkflowSettings.stepTimeout 90s.
  * remediateStep — emits RemediateStarted, then runSingleTask with TaskDef.instructions
    (formatRemediateContext(diagnosis, alertId, serviceId)) and metadata.phase = "REMEDIATE",
    taskType APPLY_REMEDIATION. Writes IncidentEntity.recordRemediation(plan). stepTimeout 90s.
  * verifyStep — emits VerifyStarted, then runSingleTask with TaskDef.instructions
    (formatVerifyContext(diagnosis, plan, incidentId)) and metadata.phase = "VERIFY", taskType
    VERIFY_FIX. Writes IncidentEntity.recordVerification(result). stepTimeout 60s.
  * error step — writes IncidentFailed and ends. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() declares defaultStepRecovery
  maxRetries(2).failoverTo(RemediationWorkflow::error).

- 1 EventSourcedEntity IncidentEntity (one per incidentId). State IncidentRecord{incidentId,
  alertId: Optional<String>, serviceId: Optional<String>, diagnosis: Optional<DiagnosisReport>,
  remediation: Optional<RemediationPlan>, verification: Optional<VerificationResult>,
  status: IncidentStatus, createdAt: Instant, closedAt: Optional<Instant>,
  budgetViolations: List<BudgetViolation>}. IncidentStatus enum: CREATED, DIAGNOSING,
  DIAGNOSED, REMEDIATING, REMEDIATED, VERIFYING, VERIFIED, FAILED. Events:
  IncidentCreated{alertId, serviceId}, DiagnoseStarted, DiagnosisCompleted{diagnosis},
  RemediateStarted, RemediationCompleted{plan}, VerifyStarted, VerificationCompleted{result},
  BudgetExhausted{phase, tool, callsUsed, cap}, IncidentFailed{reason}.
  Commands: create, startDiagnose, recordDiagnosis, startRemediate, recordRemediation,
  startVerify, recordVerification, recordBudgetExhausted, fail, getIncident.
  emptyState() returns IncidentRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3). budgetViolations initial value is List.of().

- 1 EventSourcedEntity DurabilityReportEntity (singleton id "durability-report"). State holds
  the latest DurabilityReport and a capped rolling list of the last 24 reports. Events:
  DurabilityReportRecorded{report}. Commands: record, getLatest, getHistory.

- 1 View IncidentView with row type IncidentRow that mirrors IncidentRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes IncidentEntity events. ONE
  query getAllIncidents: SELECT * AS incidents FROM incident_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * RemediationEndpoint at /api with POST /incidents (body {alertId, serviceId}; mints
    incidentId; calls IncidentEntity.create; starts RemediationWorkflow with id
    "remediation-" + incidentId; returns {incidentId}), GET /incidents (list from
    getAllIncidents, newest-first), GET /incidents/{id} (one row), GET /incidents/sse
    (Server-Sent Events from the view's stream-updates), GET /durability/latest (latest
    DurabilityReport from DurabilityReportEntity), POST /durability/trigger (fires
    DurabilityEvaluator immediately for testing), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- OpsTasks.java declaring three Task<R> constants:
    DIAGNOSE_INCIDENT = Task.name("Diagnose incident").description("Query metrics and fetch
      logs to identify root cause and severity").resultConformsTo(DiagnosisReport.class);
    APPLY_REMEDIATION = Task.name("Apply remediation").description("Apply corrective actions
      and confirm each action's status").resultConformsTo(RemediationPlan.class);
    VERIFY_FIX = Task.name("Verify fix").description("Check service health and confirm the
      incident is resolved").resultConformsTo(VerificationResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- OpsPhase.java — enum {DIAGNOSE, REMEDIATE, VERIFY}. Each function-tool method is annotated
  with the constant phase, e.g. @FunctionTool(name = "queryMetrics", phase = OpsPhase.DIAGNOSE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- DiagnoseTools.java — @FunctionTool queryMetrics(String serviceId, int windowMinutes) ->
  List<MetricSample> reading from src/main/resources/sample-data/incidents/*.json keyed by
  serviceId; @FunctionTool fetchLogs(String serviceId, String level) -> List<LogEntry> reading
  from the matching incident entry's log evidence.

- RemediateTools.java — @FunctionTool applyAction(String action, String target) ->
  ActionReceipt (deterministic in-memory receipt with id "r-" + sha1(action+target).
  substring(0,8)); @FunctionTool confirmAction(String receiptId) -> ActionStatus (deterministic
  state = "confirmed").

- VerifyTools.java — @FunctionTool checkHealth(String serviceId) -> HealthCheck (reads from
  sample-data; healthy = true in paired happy-path entries); @FunctionTool confirmResolved(
  String incidentId) -> ResolutionCheck (resolved = true in paired happy-path entries).

- BudgetGuardrail.java — implements the before-tool-call hook. Maintains a ConcurrentHashMap
  keyed by (incidentId + ":" + phase) counting tool calls. Caps: DIAGNOSE=8, REMEDIATE=6,
  VERIFY=6. On every call, increments the counter. If counter > cap BEFORE incrementing, halts:
  returns Guardrail.reject("budget-cap-exceeded: <tool> halted at <callsUsed>/<cap> for phase
  <phase>"). On halt ALSO calls IncidentEntity.recordBudgetExhausted(phase, tool, callsUsed,
  cap). Counter is reset when a new incidentId+phase combination is first seen.

- DurabilityEvaluator.java — pure logic class. Inputs: List<IncidentRow> from the trailing
  window. Computes: completionRatePercent, budgetViolationRatePercent, mttrMinutes,
  timeoutRatePercent. Returns DurabilityReport with a generated reportId and generatedAt.

- DurabilityTimerAction.java — registers the hourly timer with TimerService via the Akka
  components infrastructure. On fire, queries IncidentView for incidents with closedAt in
  [now-60m, now], calls DurabilityEvaluator.evaluate(incidents), calls
  DurabilityReportEntity.record(report).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9731 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/incidents.jsonl with 5 seeded incident lines covering the
  three named in J1-J4 plus two extras.

- src/main/resources/sample-data/incidents/*.json — three files keyed by serviceId, each
  carrying 4-8 MetricSample entries and 4-8 LogEntry entries with deterministic content so
  DiagnoseTools returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8. Matching simplified_view list. No regulation_anchors — ops-automation domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (incidents are
  service-level, not person-level), decisions.authority_level = automated-action
  (remediation actions are applied by the agent), oversight.human_in_loop = false,
  oversight.human_on_loop = true (ops team monitors the durability dashboard),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "budget-cap-exceeded", "misdiagnosed-root-cause",
  "failed-action-confirmation", "false-positive-resolution"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/ops-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Durable Workflow-Backed Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of incident cards; right = selected-incident detail with alertId, serviceId,
  diagnosis root-cause + evidence tables, remediation plan actions, verification health checks,
  budget-violation strip if present). Overview tab lower section renders the latest
  DurabilityReport as a four-metric tile row. Browser title exactly:
  <title>Akka Sample: Durable Workflow-Backed Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing ModelProvider with per-task dispatch on
  Task<R> id. Each branch reads src/main/resources/mock-responses/<task-id>.json, picks one
  entry pseudo-randomly per call (seedFor(incidentId)), deserialises into the task's typed
  return. Each entry carries a "tool_calls" array the mock replays in order before returning
  the final typed result.
- Per-task mock shapes:
    diagnose-incident.json — 6 DiagnosisReport entries, each with 3-6 MetricSample items and
      3-5 LogEntry items, tool_calls: 1 queryMetrics + 1 fetchLogs each. Plus 1 entry whose
      tool_calls array has 9 calls (exceeding the DIAGNOSE cap of 8) — BudgetGuardrail fires
      on call 9; the mock then falls through to a normal diagnose result. This violating entry
      is selected on the FIRST iteration of every 4th incident (modulo seed) so J2 is
      reproducible.
    apply-remediation.json — 6 RemediationPlan entries with 1-3 ActionReceipt items, tool_calls:
      applyAction + confirmAction per action.
    verify-fix.json — 6 VerificationResult entries with 1-3 HealthCheck items (healthy = true)
      and one ResolutionCheck (resolved = true). Plus 1 entry with resolved = false and a
      deliberately ambiguous resolutionCheck, to validate that the workflow still completes with
      a VERIFIED status even when resolution is uncertain (the agent returns what the tool gave).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded. OpsAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion OpsTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (diagnoseStep 90s, remediateStep
  90s, verifyStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on IncidentRecord is Optional<T>.
- Lesson 7: OpsTasks.java with DIAGNOSE_INCIDENT, APPLY_REMEDIATION, VERIFY_FIX is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9731 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides (state-label
  colour, edge-label foreignObject overflow:visible) AND the mermaid.initialize themeVariables
  block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
  Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: exactly ONE AutonomousAgent (OpsAgent). DurabilityEvaluator is
  a plain class with no LLM call — it is deterministic logic.
- The sequential-pipeline invariant: all three tool sets registered on the agent; BudgetGuardrail
  is the runtime enforcement mechanism. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: diagnoseStep writes DiagnosisCompleted onto
  the entity, remediateStep reads the recorded DiagnosisReport to build its instruction context,
  verifyStep reads both. The agent is stateless across phases.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
