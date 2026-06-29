# SPEC — plumber-data-engineering-assistant

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Plumber Data Engineering Assistant.
**One-line pitch:** Submit a pipeline request; a PipelinePlanner agent drafts a pipeline ledger, dispatches each stage to one of four specialist agents (Spark, Beam, dBT, Schema), records outcomes on a progress ledger, enforces a test gate before finalising, and replans when stages fail.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to data engineering. The PipelinePlanner owns two ledgers — a **pipeline ledger** (sources declared, transforms declared, sink declared, validation plan, current stage) and a **progress ledger** (each stage's attempt count, verdict, errors, observed output). Each loop iteration the PipelinePlanner reads both ledgers, picks the next specialist, and either continues, replans, finalises, or fails.

The blueprint wires one governance mechanism:

- A **ci-gate (test-gate)** that every generated stage definition must pass before it is recorded on the progress ledger. The gate runs a deterministic schema-and-syntax validator. On failure, the stage is blocked and the PipelinePlanner must revise before proceeding.

## 3. User-facing flows

The user opens the App UI tab and submits a pipeline request via the form.

1. The system creates a `Pipeline` record in `PLANNING` and starts a `PipelineWorkflow`.
2. The PipelinePlanner drafts a `PipelineLedger { sources, transforms, sinks, validationPlan, currentStage }` and emits `PipelinePlanned`.
3. The workflow enters the executor loop. Each iteration:
   - PipelinePlanner reads both ledgers and proposes a `StageDecision { engine, stageKind, spec, rationale }`.
   - The chosen specialist runs the stage and returns a typed `StageResult`.
   - The **test gate** validates `StageResult.artifact` against the schema registry. On failure, a `StageBlocked` entry is appended to the progress ledger and the planner is asked to revise.
   - On gate pass, the workflow appends a `ProgressEntry { engine, stageKind, attempt, verdict, artifact }` to the progress ledger.
4. The PipelinePlanner decides on each loop tick: `CONTINUE`, `REPLAN`, `FINALISE`, or `FAIL`. After two consecutive failures on the same stage or two consecutive replans without progress, the planner emits `FAIL`.
5. On `FINALISE`, the PipelinePlanner produces a `PipelineDefinition { title, description, stages, sinkSchema }` and emits `PipelineFinalised`. The Pipeline moves to `FINALISED`.
6. The operator can press **Halt new dispatches** at any time. The current stage finishes; the loop exits with `PipelineHaltedOperator`. The Pipeline moves to `HALTED`.

A `RequestSimulator` (TimedAction) drips a sample pipeline request every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PipelinePlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains pipeline ledger; reads progress ledger. Produces `PipelineDefinition` on finalisation. | `PipelineWorkflow` | returns typed result to workflow |
| `SparkAgent` | `AutonomousAgent` | Generates Spark job definitions (DataFrameReader/Writer, transformations) from seeded fixtures (`sample-data/spark-fixtures.jsonl`). | `PipelineWorkflow` | — |
| `BeamAgent` | `AutonomousAgent` | Generates Beam pipeline definitions (PCollections, transforms) from seeded fixtures (`sample-data/beam-fixtures.jsonl`). | `PipelineWorkflow` | — |
| `DbtAgent` | `AutonomousAgent` | Generates dBT model SQL and YAML (sources.yml, schema.yml) from seeded fixtures (`sample-data/dbt-fixtures.jsonl`). | `PipelineWorkflow` | — |
| `SchemaAgent` | `AutonomousAgent` | Validates source and sink schemas against the fixture schema registry (`sample-data/schema-registry.jsonl`). Returns a `StageResult` with a schema-compatibility report. | `PipelineWorkflow` | — |
| `PipelineWorkflow` | `Workflow` | Drives the plan → dispatch → test-gate → execute → record → decide loop, plus replan and halt branches. | `PipelineEndpoint`, `PipelineRequestConsumer` | `PipelineEntity` |
| `PipelineEntity` | `EventSourcedEntity` | Holds the pipeline lifecycle, pipeline ledger, progress ledger, and final definition. | `PipelineWorkflow` | `PipelineView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `PipelineEndpoint` (operator action) | `PipelineWorkflow` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted pipeline requests. | `PipelineEndpoint`, `RequestSimulator` | `PipelineRequestConsumer` |
| `PipelineView` | `View` | List-of-pipelines read model for the UI. | `PipelineEntity` events | `PipelineEndpoint` |
| `PipelineRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `PipelineWorkflow` per submission. | `RequestQueue` events | `PipelineWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/pipeline-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckPipelineMonitor` | `TimedAction` | Every 30 s, marks any pipeline stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `PipelineEntity` |
| `PipelineEndpoint` | `HttpEndpoint` | `/api/pipelines/*` — submit, get, list, SSE, operator halt. | — | `PipelineView`, `RequestQueue`, `PipelineEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record PipelineRequest(String prompt, String requestedBy) {}

record PipelineLedger(
    List<String> sources,
    List<String> transforms,
    List<String> sinks,
    List<String> validationPlan,
    Optional<StageDecision> currentStage
) {}

record StageDecision(
    EngineKind engine,
    StageKind stageKind,
    String spec,
    String rationale
) {}

record StageResult(
    EngineKind engine,
    StageKind stageKind,
    boolean ok,
    String artifact,
    Optional<String> errorReason
) {}

record ProgressEntry(
    int attempt,
    EngineKind engine,
    StageKind stageKind,
    ProgressVerdict verdict,
    String artifact,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ProgressLedger(List<ProgressEntry> entries) {}

record PipelineDefinition(
    String title,
    String description,
    List<String> stages,
    String sinkSchema,
    Instant producedAt
) {}

record Pipeline(
    String pipelineId,
    String prompt,
    PipelineStatus status,
    Optional<PipelineLedger> ledger,
    Optional<ProgressLedger> progress,
    Optional<PipelineDefinition> definition,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EngineKind { SPARK, BEAM, DBT, SCHEMA }
enum StageKind { SOURCE, TRANSFORM, SINK, VALIDATE }
enum ProgressVerdict { OK, BLOCKED_BY_TEST_GATE, FAILED, SCHEMA_ERROR }
enum PipelineStatus { PLANNING, EXECUTING, FINALISED, FAILED, HALTED, STUCK }
```

### Events (`PipelineEntity`)

`PipelineCreated`, `PipelinePlanned`, `StageDispatched`, `StageBlocked`, `StageRecorded`, `LedgerRevised`, `PipelineFinalised`, `PipelineFailed`, `PipelineHaltedOperator`, `PipelineFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`PipelineSubmitted { pipelineId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/pipelines` — body `{ prompt, requestedBy? }` → `202 { pipelineId }`. Starts a workflow.
- `GET /api/pipelines` — list all pipelines. Optional `?status=...`.
- `GET /api/pipelines/{id}` — one pipeline (full ledgers + definition).
- `GET /api/pipelines/sse` — server-sent events stream of every pipeline change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Plumber Data Engineering Assistant"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a pipeline request, operator halt/resume control, live list of pipelines with status pills, expand-row to see the pipeline ledger, progress ledger entries, and the final definition.

Browser title: `<title>Akka Sample: Plumber Data Engineering Assistant</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — ci-gate (test-gate)**: every `StageResult.artifact` is validated by `PipelineTestGate.validate(stageResult, schemaRegistry)` before the workflow records it. The validator checks (a) JSON/YAML well-formedness, (b) schema field-type compatibility against the registry entry for the declared source and sink, (c) that no transform writes to paths outside the declared sink. On failure, the workflow appends a `StageBlocked` entry with `verdict = BLOCKED_BY_TEST_GATE` and the planner must revise the stage spec.

## 9. Agent prompts

- `PipelinePlannerAgent` → `prompts/pipeline-planner.md`. Maintains both ledgers; decides next step.
- `SparkAgent` → `prompts/spark-agent.md`. Returns Spark job definitions.
- `BeamAgent` → `prompts/beam-agent.md`. Returns Beam pipeline definitions.
- `DbtAgent` → `prompts/dbt-agent.md`. Returns dBT model SQL and YAML.
- `SchemaAgent` → `prompts/schema-agent.md`. Returns schema-compatibility reports.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Design a Spark pipeline that reads Parquet from s3://raw-events, filters nulls, and writes Avro to s3://clean-events." Pipeline progresses `PLANNING → EXECUTING → FINALISED` within ~3 minutes. The expanded view shows a pipeline ledger with a non-empty validationPlan, a progress ledger with 3–6 entries spanning at least Spark and Schema engines, and a non-empty `PipelineDefinition`.
2. **J2** — Submit a request whose plan would produce a stage artifact that fails schema validation (e.g., a Spark read using a column type incompatible with the registry entry). The test gate blocks it; the planner replans; the pipeline either completes via a revised stage or fails after the replan budget is exhausted.
3. **J3** — Submit a pipeline and click **Halt new dispatches** while it is `EXECUTING`. The in-flight stage finishes; no further dispatches occur; the pipeline ends in `HALTED`.
4. **J4** — A dBT fixture response contains a secret-shaped connection string (`postgresql://user:PASS@host:5432/db`). The progress ledger entry shows the password replaced by `[REDACTED:url-password]`; the planner's next prompt never contains the literal password.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named plumber-data-engineering-assistant demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-plumber-pipelines.
Java package io.akka.samples.plumberdataengineeringassistant. Akka 3.6.0.
HTTP port 9498.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PipelinePlannerAgent — definition() with capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(COMPOSE_DEFINITION).maxIterationsPerTask(2)).
    System prompt from prompts/pipeline-planner.md. PLAN returns PipelineLedger.
    DECIDE returns a NextStep tagged union (Continue(StageDecision) | Replan |
    Finalise | Fail). COMPOSE_DEFINITION returns PipelineDefinition.
  * SparkAgent — capability(TaskAcceptance.of(GENERATE_SPARK).maxIterationsPerTask(2)).
    Prompt from prompts/spark-agent.md. Returns StageResult.
  * BeamAgent — capability(TaskAcceptance.of(GENERATE_BEAM).maxIterationsPerTask(2)).
    Prompt from prompts/beam-agent.md. Returns StageResult.
  * DbtAgent — capability(TaskAcceptance.of(GENERATE_DBT).maxIterationsPerTask(2)).
    Prompt from prompts/dbt-agent.md. Returns StageResult.
  * SchemaAgent — capability(TaskAcceptance.of(VALIDATE_SCHEMA).maxIterationsPerTask(2)).
    Prompt from prompts/schema-agent.md. Returns StageResult.

- 1 Workflow PipelineWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> dispatchStep ->
  testGateStep -> recordStep -> decideStep
  -> [back to checkHaltStep or to composeDefinitionStep / finaliseStep /
  failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120), testGateStep ofSeconds(30), decideStep ofSeconds(45),
    composeDefinitionStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(PipelineWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits PipelineHaltedOperator on PipelineEntity).
  dispatchStep uses switch on StageDecision.engine to call the matching
  specialist agent via forAutonomousAgent(...).runSingleTask(...).
  testGateStep calls PipelineTestGate.validate(stageResult, schemaRegistry);
  on failure records a StageBlocked entry via PipelineEntity.recordBlock(stage, reason)
  and loops back to proposeStep.
  recordStep calls PipelineEntity.recordProgress(entry).
  decideStep calls forAutonomousAgent(PipelinePlannerAgent.class, DECIDE);
  on Continue or Replan loops; on Finalise transitions to composeDefinitionStep
  -> finaliseStep; on Fail transitions to failStep.

- 1 EventSourcedEntity PipelineEntity holding Pipeline state. emptyState()
  returns Pipeline.initial("", null). Commands: createPipeline, recordPlan,
  recordDispatch, recordBlock, recordProgress, reviseLedger, finalisePipeline,
  failPipeline, haltOperator, timeoutFail, getPipeline. Events as listed in
  SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueuePipeline(pipelineId,
  prompt, requestedBy) emitting PipelineSubmitted.

- 1 View PipelineView with row type PipelineRow (mirror of Pipeline minus heavy
  ledger payloads — truncate to last 3 progress entries plus counts; the UI
  fetches the full pipeline by id on click). Table updater consumes
  PipelineEntity events. ONE query getAllPipelines SELECT * AS pipelines FROM
  pipeline_view. No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer PipelineRequestConsumer subscribed to RequestQueue events; on
  PipelineSubmitted starts a PipelineWorkflow with pipelineId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/pipeline-prompts.jsonl and calls
    RequestQueue.enqueuePipeline.
  * StuckPipelineMonitor — every 30s, queries PipelineView.getAllPipelines,
    filters EXECUTING pipelines older than 5 minutes, calls
    PipelineEntity.timeoutFail.

- 2 HttpEndpoints:
  * PipelineEndpoint at /api with POST /pipelines, GET /pipelines,
    GET /pipelines/{id}, GET /pipelines/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo PipelineLedger), DECIDE (NextStep), COMPOSE_DEFINITION
  (PipelineDefinition).
- EngineTasks.java declaring four Task<R> constants: GENERATE_SPARK,
  GENERATE_BEAM, GENERATE_DBT, VALIDATE_SCHEMA (all resultConformsTo StageResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue(StageDecision), Replan(revisedLedger), Finalise(PipelineDefinition),
  Fail(failureReason).
- application/PipelineTestGate.java — deterministic schema-and-syntax validator.
  Validates: (a) artifact is parseable JSON or YAML, (b) all declared source
  column names appear in the schema registry entry for the declared source id,
  (c) sink column types match the schema registry entry for the declared sink id,
  (d) no transform path references outside the declared sinks list.
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: postgresql://[^:]+:([^@]+)@ (url-password), AKIA[0-9A-Z]{16}
  (aws-access-key), gh[pousr]_[A-Za-z0-9]{36} (github-token),
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+ (jwt),
  and high-entropy fallback for tokens ≥ 32 chars whose Shannon entropy > 4.5
  bits/char. Replacements: [REDACTED:url-password], [REDACTED:aws-access-key],
  [REDACTED:github-token], [REDACTED:jwt], [REDACTED:high-entropy].
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9498 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/pipeline-prompts.jsonl with 8 canned
  pipeline request prompts spanning Spark, Beam, dBT, and schema-only needs.
- src/main/resources/sample-data/spark-fixtures.jsonl — 10 canned Spark job
  definitions (source path, transforms list, sink path, format).
- src/main/resources/sample-data/beam-fixtures.jsonl — 8 canned Beam pipeline
  definitions (input, transform sequence, output, runner hint).
- src/main/resources/sample-data/dbt-fixtures.jsonl — 8 canned dBT model
  definitions (model name, SQL template, sources.yml, schema.yml). ONE fixture
  entry must embed a PostgreSQL connection string with a cleartext password for
  the J4 acceptance test.
- src/main/resources/sample-data/schema-registry.jsonl — 12 schema entries
  (id, columns: [{name, type}], format).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of project-root files for the metadata endpoint).
- eval-matrix.yaml at the project root with 1 control (E1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling sector, decisions, data
  classes, capabilities, oversight; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/pipeline-planner.md, prompts/spark-agent.md, prompts/beam-agent.md,
  prompts/dbt-agent.md, prompts/schema-agent.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Plumber Data Engineering
  Assistant", one-line pitch, prerequisites (integration form host-software: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO "Visual" prefix on
  tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports
  for markdown and YAML libs are acceptable. Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs with answers from risk-survey.yaml; unanswered .qb opacity 0.45),
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form + operator halt/resume + live list with
  status pills and expand-on-click for ledgers and definition).
  Browser title exactly: <title>Akka Sample: Plumber Data Engineering
  Assistant</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every step that
  calls an agent.
- Lesson 6: Optional<T> for every nullable field.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  EngineTasks.java.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code).
- Lesson 10: HTTP port 9498 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND
  themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute; no
  zombie panels.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
