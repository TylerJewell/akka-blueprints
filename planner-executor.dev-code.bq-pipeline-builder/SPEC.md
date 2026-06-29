# SPEC — bq-pipeline-builder

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** BigQuery Pipeline Builder.
**One-line pitch:** Describe a data pipeline in plain language; a Planner decomposes the request into ordered build steps, dispatches each step to one of four specialist executors (schema analysis, SQL composition, Dataform modeling, validation), records the outcome on a step ledger, and replans when steps fail.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to data-engineering pipeline construction. The Planner owns two ledgers — a **pipeline ledger** (dataset facts, schema gaps, build plan, current dispatch) and a **step ledger** (each build step's attempt count, verdict, blockers, observed output). Each loop iteration the Planner reads both ledgers, picks the next specialist executor, and either continues, replans, halts, or completes. On three consecutive failures of the same step, or two consecutive replans without progress, the Planner emits a terminal failure.

The blueprint also demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each executor invocation against a DDL/DML safety policy before any SQL is drafted or any schema is mutated,
- a **CI test gate** that runs a deterministic validation suite against every generated artifact before it is committed to the build manifest.

## 3. User-facing flows

The user opens the App UI tab and submits a pipeline request via the form.

1. The system creates a `Pipeline` record in `PLANNING` and starts a `PipelineWorkflow`.
2. The Planner drafts a `PipelineLedger { datasetFacts, schemaGaps, buildPlan, dispatch }` and emits `PipelinePlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes a `BuildStepDecision { executor, step, rationale }`.
   - The **before-tool-call guardrail** vets the decision; on rejection the workflow records a `StepBlocked` entry on the step ledger and asks the Planner to revise.
   - The chosen executor runs the build step and returns a typed `StepOutput`.
   - The **CI test gate** runs `ValidationSuite.run(stepOutput)` against fixture schemas; on failure the gate records a `StepFailed` verdict and the planner sees the failure on its next loop tick.
   - The workflow appends a `StepEntry { executor, step, attempt, verdict, output }` to the step ledger.
4. The Planner decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After three consecutive failures on the same step or two consecutive replans without progress, the Planner emits `FAIL`.
5. On `COMPLETE`, the Planner produces a `BuildManifest { summary, artifacts, producedAt }` and emits `PipelineCompleted`. The Pipeline moves to `COMPLETED`.
6. The operator can press **Halt new dispatches** in the dashboard at any time. The workflow finishes the in-flight step, then ends with `PipelineHaltedOperator`. The Pipeline moves to `HALTED`.

A `PipelineSimulator` (TimedAction) drips a sample pipeline request every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains pipeline ledger; reads step ledger. Produces `BuildManifest` on completion. | `PipelineWorkflow` | returns typed result to workflow |
| `SchemaAnalystAgent` | `AutonomousAgent` | Reads seeded BigQuery schema fixtures (`sample-data/schemas/*.json`) and reports table/field layouts. | `PipelineWorkflow` | — |
| `SqlComposerAgent` | `AutonomousAgent` | Drafts or refines SQL transformation queries scoped to the fixture dataset namespaces. | `PipelineWorkflow` | — |
| `DataformModelerAgent` | `AutonomousAgent` | Produces Dataform SQLX model definitions and `includes/` JS helper macros. | `PipelineWorkflow` | — |
| `ValidatorAgent` | `AutonomousAgent` | Runs deterministic lint and dry-run checks against fixture schema, returning a `ValidationReport`. | `PipelineWorkflow` | — |
| `PipelineWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → ciGate → record → decide loop, plus replan and halt branches. | `PipelineEndpoint`, `PipelineRequestConsumer` | `PipelineEntity` |
| `PipelineEntity` | `EventSourcedEntity` | Holds the pipeline's lifecycle, pipeline ledger, step ledger, and final manifest. | `PipelineWorkflow` | `PipelineView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `PipelineEndpoint` (operator action) | `PipelineWorkflow` (polls) |
| `PipelineQueue` | `EventSourcedEntity` | Audit log of submitted pipeline requests. | `PipelineEndpoint`, `PipelineSimulator` | `PipelineRequestConsumer` |
| `PipelineView` | `View` | List-of-pipelines read model for the UI. | `PipelineEntity` events | `PipelineEndpoint` |
| `PipelineRequestConsumer` | `Consumer` | Subscribes to `PipelineQueue` events; starts a `PipelineWorkflow` per submission. | `PipelineQueue` events | `PipelineWorkflow` |
| `PipelineSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/pipeline-prompts.jsonl` and enqueues it. | scheduler | `PipelineQueue` |
| `StalePipelineMonitor` | `TimedAction` | Every 30 s, marks any pipeline stuck in `BUILDING` past 5 minutes as `STUCK`. | scheduler | `PipelineEntity` |
| `PipelineEndpoint` | `HttpEndpoint` | `/api/pipelines/*` — submit, get, list, SSE, operator halt. | — | `PipelineView`, `PipelineQueue`, `PipelineEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record PipelineRequest(String description, String requestedBy) {}

record PipelineLedger(
    List<String> datasetFacts,
    List<String> schemaGaps,
    List<String> buildPlan,
    Optional<BuildStepDecision> currentDispatch
) {}

record BuildStepDecision(
    ExecutorKind executor,
    String step,
    String rationale
) {}

record StepOutput(
    ExecutorKind executor,
    String step,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record StepEntry(
    int attempt,
    ExecutorKind executor,
    String step,
    StepVerdict verdict,
    String output,
    Optional<String> blocker,
    Instant recordedAt
) {}

record StepLedger(List<StepEntry> entries) {}

record ValidationReport(
    boolean passed,
    List<String> failures,
    List<String> warnings
) {}

record BuildManifest(
    String summary,
    List<String> artifacts,
    Instant producedAt
) {}

record Pipeline(
    String pipelineId,
    String description,
    PipelineStatus status,
    Optional<PipelineLedger> ledger,
    Optional<StepLedger> steps,
    Optional<BuildManifest> manifest,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExecutorKind { SCHEMA_ANALYST, SQL_COMPOSER, DATAFORM_MODELER, VALIDATOR }
enum StepVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, CI_GATE_FAILED }
enum PipelineStatus { PLANNING, BUILDING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`PipelineEntity`)

`PipelineCreated`, `PipelinePlanned`, `StepDispatched`, `StepBlocked`, `StepRecorded`, `LedgerRevised`, `PipelineCompleted`, `PipelineFailed`, `PipelineHaltedOperator`, `PipelineFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`PipelineQueue`)

`PipelineSubmitted { pipelineId, description, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/pipelines` — body `{ description, requestedBy? }` → `202 { pipelineId }`. Starts a workflow.
- `GET /api/pipelines` — list all pipelines. Optional `?status=...`.
- `GET /api/pipelines/{id}` — one pipeline (full ledgers + manifest).
- `GET /api/pipelines/sse` — server-sent events stream of every pipeline change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "BigQuery Pipeline Builder"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a pipeline request, operator halt/resume control, live list of pipelines with status pills, expand-row to see the pipeline ledger, the step ledger entries, and the final build manifest.

Browser title: `<title>Akka Sample: BigQuery Pipeline Builder</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `PlannerAgent`): every `BuildStepDecision` is checked against (a) the executor allow-list, (b) a DDL/DML safety policy that forbids `DROP TABLE`, `TRUNCATE TABLE`, `DELETE FROM` without a `WHERE` clause, and `ALTER TABLE … DROP COLUMN` — all operations that would irreversibly destroy data in a production dataset. Blocking. Failure → `StepBlocked` entry + replan request.
- **CI1 — CI test gate** (`ci-gate`, flavor `test-gate`): after every `StepOutput` is produced, `ValidationSuite.run(stepOutput)` checks generated SQL against fixture schemas (column references, data types, required alias conventions), checks SQLX definitions for mandatory `config` blocks and `ref()` calls, and checks JS macros for forbidden `console.log` and `process.env` references. On gate failure the workflow records `CI_GATE_FAILED` verdict and the Planner sees the failure on the next tick.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Maintains both ledgers; decides next step.
- `SchemaAnalystAgent` → `prompts/schema-analyst.md`. Returns schema reports from fixtures.
- `SqlComposerAgent` → `prompts/sql-composer.md`. Returns SQL transformation text.
- `DataformModelerAgent` → `prompts/dataform-modeler.md`. Returns SQLX and JS macro content.
- `ValidatorAgent` → `prompts/validator.md`. Returns a `ValidationReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Build a pipeline that loads raw GA4 events into a clean sessions table with session-level aggregates." Pipeline progresses `PLANNING → BUILDING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a pipeline ledger with a non-empty build plan, a step ledger with 3–8 entries spanning at least `SCHEMA_ANALYST` and `SQL_COMPOSER`, and a non-empty `BuildManifest`.
2. **J2** — Submit a pipeline whose plan would issue a `DROP TABLE` DDL. The guardrail blocks the dispatch on the first attempt; the planner replans; the pipeline either completes via a safer path or fails after the replan budget is exhausted.
3. **J3** — Submit a pipeline and click **Halt new dispatches** while it is `BUILDING`. The in-flight step finishes; no further dispatches occur; the pipeline ends in `HALTED`.
4. **J4** — Trigger three consecutive `CI_GATE_FAILED` verdicts on the same step; confirm the planner exhausts its failure budget and the pipeline ends in `FAILED` with a clear `failureReason`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bq-pipeline-builder demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-bq-pipeline-builder.
Java package io.akka.samples.dataengineering. Akka 3.6.0. HTTP port 9535.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_PIPELINE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE_STEP).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_MANIFEST).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_PIPELINE returns PipelineLedger.
    DECIDE_STEP returns a NextBuildStep tagged union (Continue(BuildStepDecision) |
    Replan(PipelineLedger) | Complete(BuildManifest stub) | Fail(String reason)).
    COMPOSE_MANIFEST returns BuildManifest.
  * SchemaAnalystAgent — capability(TaskAcceptance.of(ANALYZE_SCHEMA).maxIterationsPerTask(2)).
    Prompt from prompts/schema-analyst.md. Returns StepOutput.
  * SqlComposerAgent — capability(TaskAcceptance.of(COMPOSE_SQL).maxIterationsPerTask(2)).
    Prompt from prompts/sql-composer.md. Returns StepOutput with content holding
    a SQL string.
  * DataformModelerAgent — capability(TaskAcceptance.of(MODEL_DATAFORM).maxIterationsPerTask(2)).
    Prompt from prompts/dataform-modeler.md. Returns StepOutput with content
    holding SQLX or JS macro text.
  * ValidatorAgent — capability(TaskAcceptance.of(VALIDATE_OUTPUT).maxIterationsPerTask(2)).
    Prompt from prompts/validator.md. Returns StepOutput with content holding
    a serialized ValidationReport.

- 1 Workflow PipelineWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  dispatchStep -> ciGateStep -> recordStep -> decideStep
  -> [back to checkHaltStep or to completeStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any executor call), ciGateStep ofSeconds(30),
    decideStep ofSeconds(45), completeStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(PipelineWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits PipelineHaltedOperator on PipelineEntity).
  guardrailStep runs DdlGuardrail.vet(BuildStepDecision); on reject records a
  StepBlocked entry via PipelineEntity.recordBlock(step, reason) and loops
  back to proposeStep.
  dispatchStep uses switch on BuildStepDecision.executor to call the matching
  specialist agent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(pipelineId).result(...).
  ciGateStep calls ValidationSuite.run(stepOutput); on gate failure records
  StepEntry with verdict CI_GATE_FAILED and loops back to proposeStep.
  recordStep calls PipelineEntity.recordStep(entry).
  decideStep calls forAutonomousAgent(PlannerAgent.class, DECIDE_STEP);
  on Continue or Replan loops; on Complete transitions to composeManifestStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity PipelineEntity holding Pipeline state. emptyState()
  returns Pipeline.initial("", null) with no commandContext() reference.
  Commands: createPipeline, recordPlan, recordDispatch, recordBlock,
  recordStep, reviseLedger, completePipeline, failPipeline, haltOperator,
  timeoutFail, getPipeline. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity PipelineQueue with command enqueueRequest(pipelineId,
  description, requestedBy) emitting PipelineSubmitted.

- 1 View PipelineView with row type PipelineRow (mirror of Pipeline minus
  heavy ledger payloads — truncate to last 3 step entries plus counts; the
  UI fetches the full pipeline by id on click). Table updater consumes
  PipelineEntity events. ONE query getAllPipelines SELECT * AS pipelines FROM
  pipeline_view. No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer PipelineRequestConsumer subscribed to PipelineQueue events; on
  PipelineSubmitted starts a PipelineWorkflow with pipelineId as the workflow id.

- 2 TimedActions:
  * PipelineSimulator — every 90s, reads next line from
    src/main/resources/sample-events/pipeline-prompts.jsonl and calls
    PipelineQueue.enqueueRequest.
  * StalePipelineMonitor — every 30s, queries PipelineView.getAllPipelines,
    filters BUILDING pipelines whose createdAt is older than 5 minutes, calls
    PipelineEntity.timeoutFail; PipelineWorkflow polls PipelineEntity.getPipeline
    in its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * PipelineEndpoint at /api with POST /pipelines, GET /pipelines (filters
    client-side from getAllPipelines), GET /pipelines/{id}, GET /pipelines/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_PIPELINE
  (resultConformsTo PipelineLedger), DECIDE_STEP (NextBuildStep),
  COMPOSE_MANIFEST (BuildManifest).
- ExecutorTasks.java declaring four Task<R> constants: ANALYZE_SCHEMA,
  COMPOSE_SQL, MODEL_DATAFORM, VALIDATE_OUTPUT (all resultConformsTo
  StepOutput).
- Domain records as listed in SPEC §5, plus a NextBuildStep sealed interface
  with permits Continue, Replan, Complete, Fail (each carrying its own
  payload — Continue with BuildStepDecision, Replan with revisedLedger,
  Complete with BuildManifest stub, Fail with failureReason).
- application/DdlGuardrail.java — deterministic vetter. Reject if the
  executor is not SCHEMA_ANALYST/SQL_COMPOSER/DATAFORM_MODELER/VALIDATOR,
  if a SQL_COMPOSER step contains the tokens DROP TABLE, TRUNCATE TABLE,
  DELETE FROM without WHERE, or ALTER TABLE … DROP COLUMN (case-insensitive,
  whitespace-normalised regex), if a DATAFORM_MODELER step references a
  target dataset not in the allow-list (raw, staging, mart, sandbox).
- application/ValidationSuite.java — deterministic CI gate. For
  ANALYZE_SCHEMA outputs: verify JSON structure matches fixture schema
  envelope. For COMPOSE_SQL outputs: verify all column references exist in
  the fixture schema and aliases follow `snake_case`. For MODEL_DATAFORM
  outputs: verify SQLX config block present, every table ref uses `ref()`,
  no bare `process.env` in JS macros. For VALIDATE_OUTPUT outputs: parse
  ValidationReport JSON and propagate passed/failures. Returns
  ValidationReport{passed, failures, warnings}.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9535 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/pipeline-prompts.jsonl with 8 canned
  pipeline requests spanning schema inspection, SQL transformation, Dataform
  modeling, and full end-to-end build needs.
- src/main/resources/sample-data/schemas/*.json — 6 BigQuery schema
  fixture files (ga4_events, sessions, orders, products, user_dim,
  daily_revenue). Used by SchemaAnalystAgent.
- src/main/resources/sample-data/sql/*.sql — 6 SQL transformation
  fixture files used by SqlComposerAgent as starting templates.
- src/main/resources/sample-data/dataform/*.sqlx — 4 SQLX fixture
  definitions used by DataformModelerAgent. One definition contains
  a bare process.env reference so the J4 CI gate test fires.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, CI1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/schema-analyst.md, prompts/sql-composer.md,
  prompts/dataform-modeler.md, prompts/validator.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: BigQuery Pipeline
  Builder", one-line pitch, prerequisites (including the integration form's
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for ledgers and manifest). Browser title
  exactly: <title>Akka Sample: BigQuery Pipeline Builder</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape.
- Per-agent mock-response shapes for THIS blueprint:
    planner.json — sectioned by task id:
      "PLAN_PIPELINE" → 4–6 PipelineLedger entries (datasetFacts, schemaGaps,
      buildPlan steps that span all four executors).
      "DECIDE_STEP" → 4–6 NextBuildStep entries covering Continue
      (BuildStepDecision across all executors), Replan, Complete, Fail.
      "COMPOSE_MANIFEST" → 4–6 BuildManifest entries with 60–120 word
      summaries and 3–5 artifact bullets.
    schema-analyst.json — 6 StepOutput entries, ok=true, content fields are
      mocked schema-inspection summaries describing table columns.
    sql-composer.json — 6 StepOutput entries with SQL SELECT/JOIN content;
      one entry intentionally contains a bare DROP TABLE statement so the
      guardrail test (J2) fires.
    dataform-modeler.json — 5 StepOutput entries; one SQLX definition omits
      the config block so the CI gate (J4) fires.
    validator.json — 5 StepOutput entries containing serialised
      ValidationReport objects; two entries have passed=false with failures
      listing to exercise the CI gate path.
- A MockModelProvider.seedFor(pipelineId) helper makes the selection
  deterministic per pipeline id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep,
  ciGateStep, decideStep, completeStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Pipeline entity state (ledger, steps, manifest, failureReason,
  haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  ExecutorTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: HTTP port 9535 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
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
