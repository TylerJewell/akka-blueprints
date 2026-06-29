# SPEC — impact-study-runner

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Grid Impact Study Runner.
**One-line pitch:** An engineer submits an interconnection request; one `StudyAgent` walks it through three task phases — **LOAD_FLOW** bus-voltage and power-flow computation, **CONTINGENCY** N-1 violation analysis, **DRAFT** a structured study report — with each phase gated on the prior phase's recorded output and a human-in-the-loop approval gate before the report is issued.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an energy-utilities interconnection-study domain. One `StudyAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the LOAD_FLOW task's typed output becomes the CONTINGENCY task's instruction context; the CONTINGENCY task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **human-in-the-loop (HITL) approval gate** sits between `draftStep` and issuance. After `ReportDrafted` lands and the `SimulatorScorer` produces an evaluation score, the workflow pauses in `AWAITING_APPROVAL`. An engineer reviews the drafted report (and the eval score) in the UI and either calls `POST /api/studies/{id}/approve` or `POST /api/studies/{id}/reject`. Only an `APPROVED` study transitions to a retrievable final-issued document. A rejected study records the engineer's reason and transitions to `REJECTED` (terminal). This gate enforces the entry's rationale: no study report is issued without a qualified engineer sign-off.
- An **`on-decision-eval`** runs immediately after `ReportDrafted` lands, as `evalStep` inside the workflow, before the approval pause. A deterministic, rule-based `SimulatorScorer` (no LLM call — the eval is rule-based on purpose) checks that the drafted report accounts for every contingency violation recorded in `ContingencyResult`, that every bus voltage in the report falls within the thermal and voltage limits encoded in `src/main/resources/sample-data/grid-params.json`, that the report's generator output figures match the load-flow results, and that the study's scope (bus list) matches the request. Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce the dependency contract, and adding a synchronous eval before a human approval gate makes the reviewer's job tractable on every study.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types an **interconnection request ID** into the input (or picks one of three seeded requests — `REQ-2026-001 Lakeside Wind 150 MW`, `REQ-2026-002 Desert Solar 80 MW`, `REQ-2026-003 River Pumped Storage 200 MW`).
2. The user clicks **Run study**. The UI POSTs to `/api/studies` and receives a `studyId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RUNNING_LOAD_FLOW` — the workflow has started `loadFlowStep` and the agent has been handed the LOAD_FLOW task.
4. Within ~10–20 s the card reaches `LOAD_FLOW_DONE` — the typed `LoadFlowResult` is visible in the card detail (a bus-voltage table and a power-flow summary). The agent's LOAD_FLOW task returned; the workflow recorded `LoadFlowComputed` and ran the CONTINGENCY task.
5. Within ~10–20 s more the card reaches `CONTINGENCY_DONE`. The `ContingencyResult` is visible (violation list with contingency ID, element name, limit, and observed value).
6. Within ~10–20 s more the card reaches `AWAITING_APPROVAL`. The right pane now shows the full typed `StudyReport` — executive summary, per-finding `Section` with body and a `busRefs` list — plus an eval score chip (1–5) and a one-line rationale. An `Approve` and a `Reject` button appear for the engineer.
7. The engineer clicks **Approve** (or **Reject** with a reason). The card transitions to `APPROVED` (or `REJECTED`).
8. The user can submit another request; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StudyEndpoint` | `HttpEndpoint` | `/api/studies/*` — submit, list, get, SSE, approve, reject; serves `/api/metadata/*`. | — | `StudyEntity`, `StudyView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `StudyEntity` | `EventSourcedEntity` | Per-study lifecycle: created → running_load_flow → load_flow_done → running_contingency → contingency_done → drafting → report_drafted → awaiting_approval → approved / rejected / failed. Source of truth. | `StudyEndpoint`, `StudyPipelineWorkflow` | `StudyView` |
| `StudyPipelineWorkflow` | `Workflow` | One workflow per study. Steps: `loadFlowStep` → `contingencyStep` → `draftStep` → `evalStep` → `approvalStep`. Each of the first three steps runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `evalStep` is synchronous (no LLM). `approvalStep` pauses until the engineer acts. | started by `StudyEndpoint` after `CREATED` | `StudyAgent`, `StudyEntity` |
| `StudyAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `StudyTasks.java`: `RUN_LOAD_FLOW` → `LoadFlowResult`, `RUN_CONTINGENCY` → `ContingencyResult`, `DRAFT_REPORT` → `StudyReport`. Each task is registered with the phase-appropriate function tools. | invoked by `StudyPipelineWorkflow` | returns typed results |
| `LoadFlowTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `computeBusVoltages(requestId)` and `computePowerFlows(requestId)`. Reads from `src/main/resources/sample-data/load-flow/*.json` for deterministic offline output. | called from LOAD_FLOW task | returns `List<BusReading>` / `List<LineFlow>` |
| `ContingencyTools` | function-tools class | Implements `runN1Analysis(loadFlowResult)` and `checkVoltageThresholds(loadFlowResult)`. Pure in-memory checks against limits from `grid-params.json`. | called from CONTINGENCY task | returns `List<Violation>` / `List<VoltageCheck>` |
| `DraftTools` | function-tools class | Implements `formatFinding(violation, loadFlowResult)` and `assembleExecutiveSummary(violations)`. | called from DRAFT task | returns `Section` / `String` |
| `SimulatorScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `StudyReport`, `ContingencyResult`, `LoadFlowResult`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `StudyView` | `View` | Read model: one row per study for the UI. | `StudyEntity` events | `StudyEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BusReading(String busId, String busName, double voltageKv, double voltagePu, Instant measuredAt) {}

record LineFlow(String lineId, String fromBus, String toBus, double flowMw, double loadPercent, Instant measuredAt) {}

record LoadFlowResult(List<BusReading> busReadings, List<LineFlow> lineFlows, Instant computedAt) {}

record Violation(String contingencyId, String elementName, String limitType, double limit, double observed, boolean isBinding) {}

record VoltageCheck(String busId, double voltageKv, double lowerLimitKv, double upperLimitKv, boolean withinLimits) {}

record ContingencyResult(List<Violation> violations, List<VoltageCheck> voltageChecks, int n1ContingenciesTested, Instant analyzedAt) {}

record BusRef(String busId, String busName) {}

record Section(String findingId, String heading, String body, List<BusRef> busRefs) {}

record StudyReport(
    String requestId,
    String executiveSummary,
    List<Section> sections,
    boolean meetsN1Criterion,
    Instant draftedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record StudyRecord(
    String studyId,
    Optional<String> requestId,
    Optional<LoadFlowResult> loadFlow,
    Optional<ContingencyResult> contingency,
    Optional<StudyReport> report,
    Optional<EvalResult> eval,
    Optional<String> approvalNote,
    StudyStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum StudyStatus {
    CREATED, RUNNING_LOAD_FLOW, LOAD_FLOW_DONE, RUNNING_CONTINGENCY, CONTINGENCY_DONE,
    DRAFTING, REPORT_DRAFTED, AWAITING_APPROVAL, APPROVED, REJECTED, FAILED
}
```

Events on `StudyEntity`: `StudyCreated`, `LoadFlowStarted`, `LoadFlowComputed`, `ContingencyStarted`, `ContingencyAnalyzed`, `DraftStarted`, `ReportDrafted`, `EvaluationScored`, `ApprovalRequested`, `StudyApproved`, `StudyRejected`, `StudyFailed`.

Every nullable lifecycle field on the `StudyRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/studies` — body `{ requestId }` → `{ studyId }`.
- `GET /api/studies` — list all studies, newest-first.
- `GET /api/studies/{id}` — one study.
- `GET /api/studies/sse` — Server-Sent Events; one event per state transition.
- `POST /api/studies/{id}/approve` — body `{ note? }` → `200 { studyId, status: "APPROVED" }`.
- `POST /api/studies/{id}/reject` — body `{ reason }` → `200 { studyId, status: "REJECTED" }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Grid Impact Study Runner</title>`.

The App UI tab is a two-column layout: a left rail with the live list of studies (status pill + request ID + age) and a right pane with the selected study's detail — request ID, bus-voltage table, contingency violation list, report sections, eval score chip, and approve/reject controls when the study is in `AWAITING_APPROVAL`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL approval gate (application-level)**: `StudyPipelineWorkflow.approvalStep` pauses the workflow after `EvaluationScored` has been recorded. The workflow step exposes a durable pause backed by the Akka workflow timer. It only resumes on `StudyEntity.approve(note)` or `StudyEntity.reject(reason)`, both callable only via `POST /api/studies/{id}/approve` and `POST /api/studies/{id}/reject`. These commands are the sole paths out of `AWAITING_APPROVAL`. The workflow enforces the constraint structurally: the approval commands are the workflow's only external-signal transitions. A study that has not received an explicit engineer action stays in `AWAITING_APPROVAL` indefinitely and is never auto-approved. The eval score and rationale are surfaced on the UI approval card so the reviewing engineer has the automated scoring in view when deciding.
- **E1 — `on-decision-eval`**: runs immediately after `ReportDrafted` lands, as `evalStep` inside the workflow, before the approval pause. `SimulatorScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest). Four checks, one point per check satisfied, starting from a base of 1: (1) violation coverage — every `ContingencyResult.violations[i]` has a matching `StudyReport.sections[j]` (`violation.contingencyId == section.findingId`); (2) voltage bounds — every `BusReading.voltageKv` in the `LoadFlowResult` falls within the limits in `grid-params.json`; (3) generator output parity — the report's `meetsN1Criterion` flag matches whether `ContingencyResult.violations` contains any binding violation (`violation.isBinding == true`); (4) scope coverage — `StudyReport.sections.size()` equals the number of binding violations in `ContingencyResult` (no silent collapse). Emits `EvaluationScored{score:1..5, rationale}` with a rationale sentence naming the largest gap.

## 9. Agent prompts

- `StudyAgent` → `prompts/study-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools matching that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded request `REQ-2026-001 Lakeside Wind 150 MW`; within 60 s the study reaches `AWAITING_APPROVAL` with a non-empty `LoadFlowResult`, ≥ 1 contingency violation, ≥ 1 report section, and an eval score chip on the card.
2. **J2** — Engineer clicks **Approve** with an optional note; the study transitions to `APPROVED`. The issued report is retrievable via `GET /api/studies/{id}` with `status: APPROVED`.
3. **J3** — Engineer clicks **Reject** with a reason; the study transitions to `REJECTED`. The card border highlights orange. The rejection reason is visible in the card detail.
4. **J4** — A study whose mock-LLM trajectory produces a report citing a binding violation as non-binding is scored ≤ 2 by `SimulatorScorer` with a rationale naming the N-1 criterion mismatch; the UI flags the card before the approval gate so the engineer sees the score.
5. **J5** — Each task's tool calls are visible in the per-study trace (logged at `INFO`); the LOAD_FLOW task's log shows only LOAD_FLOW-tool calls, the CONTINGENCY task's log shows only CONTINGENCY-tool calls, the DRAFT task's log shows only DRAFT-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named impact-study-runner demonstrating the sequential-pipeline x energy-utilities
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-energy-utilities-impact-study-runner. Java package
io.akka.samples.gridimpactstudyrunner. Akka 3.6.0. HTTP port 9684.

Components to wire (exactly):

- 1 AutonomousAgent StudyAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/study-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  LOAD_FLOW, CONTINGENCY, and DRAFT tool sets are ALL registered on the agent; phase
  isolation is enforced by the workflow step's task scoping, NOT by conditional .tools(...)
  wiring. There is no before-tool-call guardrail on StudyAgent.

- 1 Workflow StudyPipelineWorkflow per studyId with five steps:
  * loadFlowStep — emits LoadFlowStarted on the entity, then calls componentClient
    .forAutonomousAgent(StudyAgent.class, "agent-" + studyId).runSingleTask(
      TaskDef.instructions("RequestId: " + requestId + "\nPhase: LOAD_FLOW\nCompute bus
      voltages and power flows for this interconnection request.")
        .metadata("studyId", studyId)
        .metadata("phase", "LOAD_FLOW")
        .taskType(StudyTasks.RUN_LOAD_FLOW)
    ). Reads forTask(taskId).result(RUN_LOAD_FLOW) to get LoadFlowResult. Writes
    StudyEntity.recordLoadFlow(loadFlowResult). WorkflowSettings.stepTimeout 60s.
  * contingencyStep — emits ContingencyStarted, then runSingleTask with TaskDef.instructions
    (formatContingencyContext(loadFlowResult, requestId)) and metadata.phase = "CONTINGENCY",
    taskType RUN_CONTINGENCY. Writes StudyEntity.recordContingency(contingencyResult).
    stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(contingencyResult, loadFlowResult, requestId)) and metadata.phase =
    "DRAFT", taskType DRAFT_REPORT. Writes StudyEntity.recordReport(studyReport).
    stepTimeout 60s.
  * evalStep — runs the deterministic SimulatorScorer over (studyReport, contingencyResult,
    loadFlowResult) and writes StudyEntity.recordEvaluation(eval). stepTimeout 5s.
  * approvalStep — emits ApprovalRequested on the entity, then pauses the workflow until
    an external signal arrives via StudyEntity.approve(note) or StudyEntity.reject(reason).
    The step has no timeout (null timeout) — it waits indefinitely. The approve command
    writes StudyApproved; the reject command writes StudyRejected. Both are terminal.
    stepTimeout null.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(StudyPipelineWorkflow::error). The error step writes
  StudyFailed and ends.

- 1 EventSourcedEntity StudyEntity (one per studyId). State StudyRecord{studyId,
  requestId: Optional<String>, loadFlow: Optional<LoadFlowResult>, contingency:
  Optional<ContingencyResult>, report: Optional<StudyReport>, eval: Optional<EvalResult>,
  approvalNote: Optional<String>, status: StudyStatus, createdAt: Instant, finishedAt:
  Optional<Instant>}. StudyStatus enum: CREATED, RUNNING_LOAD_FLOW, LOAD_FLOW_DONE,
  RUNNING_CONTINGENCY, CONTINGENCY_DONE, DRAFTING, REPORT_DRAFTED, AWAITING_APPROVAL,
  APPROVED, REJECTED, FAILED. Events: StudyCreated{requestId}, LoadFlowStarted,
  LoadFlowComputed{loadFlow}, ContingencyStarted, ContingencyAnalyzed{contingency},
  DraftStarted, ReportDrafted{report}, EvaluationScored{eval}, ApprovalRequested,
  StudyApproved{note}, StudyRejected{reason}, StudyFailed{reason}. Commands: create,
  startLoadFlow, recordLoadFlow, startContingency, recordContingency, startDraft,
  recordReport, recordEvaluation, requestApproval, approve, reject, fail, getStudy.
  emptyState() returns StudyRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier.

- 1 View StudyView with row type StudyRow that mirrors StudyRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes StudyEntity events.
  ONE query getAllStudies: SELECT * AS studies FROM study_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * StudyEndpoint at /api with POST /studies (body {requestId}; mints studyId; calls
    StudyEntity.create(requestId); then starts StudyPipelineWorkflow with id
    "pipeline-" + studyId; returns {studyId}), GET /studies (list from getAllStudies,
    sorted newest-first), GET /studies/{id} (one row), GET /studies/sse (Server-Sent
    Events forwarded from the view's stream-updates), POST /studies/{id}/approve (calls
    StudyEntity.approve(note); returns {studyId, status:"APPROVED"}), POST
    /studies/{id}/reject (calls StudyEntity.reject(reason); returns {studyId,
    status:"REJECTED"}), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- StudyTasks.java declaring three Task<R> constants:
    RUN_LOAD_FLOW = Task.name("Run load flow").description("Compute bus voltages and power
      flows for the interconnection request").resultConformsTo(LoadFlowResult.class);
    RUN_CONTINGENCY = Task.name("Run contingency").description("Identify N-1 violations and
      check voltage thresholds").resultConformsTo(ContingencyResult.class);
    DRAFT_REPORT = Task.name("Draft report").description("Compose a StudyReport whose
      Sections address each binding contingency violation").resultConformsTo(StudyReport.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- LoadFlowTools.java — @FunctionTool computeBusVoltages(String requestId) ->
  List<BusReading> reading from src/main/resources/sample-data/load-flow/*.json keyed by
  requestId; @FunctionTool computePowerFlows(String requestId) -> List<LineFlow> reading
  from the same file's lineFlows array.

- ContingencyTools.java — @FunctionTool runN1Analysis(LoadFlowResult loadFlowResult) ->
  List<Violation> (checking each line outage scenario against thermal limits from
  grid-params.json); @FunctionTool checkVoltageThresholds(LoadFlowResult loadFlowResult)
  -> List<VoltageCheck> (comparing each BusReading.voltageKv to bus-level limits).

- DraftTools.java — @FunctionTool formatFinding(Violation violation, LoadFlowResult
  loadFlowResult) -> Section (heading from violation.elementName, body composed from the
  violation fields and load-flow context); @FunctionTool assembleExecutiveSummary(
  List<Violation> violations) -> String (one-paragraph summary of binding violations and
  overall N-1 compliance status).

- SimulatorScorer.java — pure deterministic logic (no LLM). Inputs: StudyReport,
  ContingencyResult, LoadFlowResult. Outputs: EvalResult with score and rationale.
  Four checks, one point per check satisfied, starting from a base of 1: violation
  coverage (every binding ContingencyResult.violations[i] has a matching Section in the
  report), voltage bounds (every BusReading.voltageKv in LoadFlowResult falls within
  limits from grid-params.json), N-1 criterion parity (report.meetsN1Criterion ==
  !hasBindingViolations(contingencyResult)), and section count parity
  (report.sections.size() == countBindingViolations(contingencyResult)). Score range
  1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9684 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/requests.jsonl with 5 seeded request lines covering
  the three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/load-flow/*.json — three files keyed by seeded requestId,
  each carrying 6-10 BusReading entries and 4-8 LineFlow entries with deterministic content
  so LoadFlowTools returns the same list across restarts.

- src/main/resources/sample-data/grid-params.json — voltage limits (lowerKv, upperKv)
  per busId and thermal limits (maxMw) per lineId. Used by ContingencyTools and
  SimulatorScorer.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for the energy-utilities domain.

- prompts/study-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Grid Impact Study Runner",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of study cards; right = selected-study detail with request ID header, bus-voltage
  table, contingency violation list, report sections, eval-score chip, approve/reject
  controls). Browser title exactly: <title>Akka Sample: Grid Impact Study Runner</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on the Task<R> id. Each branch
  reads src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(studyId)), and deserialises into the task's typed return.
- Per-task mock-response shapes:
    run-load-flow.json — 5 LoadFlowResult entries, each with 6-10 BusReading items and
      4-8 LineFlow items matching the seeded requestIds.
    run-contingency.json — 5 ContingencyResult entries paired one-to-one with the load-flow
      entries, each with 1-3 Violation items (at least one binding) and 6-10 VoltageCheck
      items; one entry carries a deliberately non-binding violation set so J4 can target it.
    draft-report.json — 5 StudyReport entries paired one-to-one. Each carries sections
      matching the paired ContingencyResult binding violations. One entry deliberately
      misreports meetsN1Criterion as true when a binding violation exists — SimulatorScorer
      flags this with score ≤ 2. MockModelProvider.seedFor(studyId) makes per-study
      selection deterministic across restarts.

Constraints:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. StudyAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion StudyTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (loadFlowStep
  60s, contingencyStep 60s, draftStep 60s, evalStep 5s, approvalStep null, error 5s).
- Lesson 6: every nullable lifecycle field on the StudyRecord row record is Optional<T>.
- Lesson 7: StudyTasks.java with RUN_LOAD_FLOW, RUN_CONTINGENCY, DRAFT_REPORT constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9684 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index.
- The single-agent invariant: there is exactly ONE AutonomousAgent (StudyAgent). The
  on-decision eval is rule-based (SimulatorScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent;
  the workflow step's task scoping enforces phase isolation. Do NOT conditionally register
  tools per task.
- Task dependency is carried by typed task results: loadFlowStep writes LoadFlowResult
  onto the entity, contingencyStep reads it and builds the CONTINGENCY task's instruction
  context from it, draftStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
