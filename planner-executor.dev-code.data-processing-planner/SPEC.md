# SPEC — data-processing-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Amazon DataProcessing Agent.
**One-line pitch:** Submit a pipeline request; a PlannerAgent decomposes the work into an ordered job plan, dispatches each step to a JobExecutorAgent (Glue crawler, EMR step, Spark submit, or S3 copy), records outcomes on a run ledger, enforces a cost guardrail before each submission, and halts on operator command.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with a single executor agent covering multiple job engine types. The PlannerAgent owns two ledgers — a **job ledger** (known inputs, missing parameters, ordered job plan, current dispatch decision) and a **run ledger** (each step's attempt count, verdict, cost estimate consumed, observed output). Each loop iteration the PlannerAgent reads both ledgers, picks the next step, and either continues, revises the plan, halts, or completes. On two consecutive failures of the same step, or two consecutive replans without progress, the PlannerAgent emits a terminal failure.

The blueprint also demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each job dispatch against a cost ceiling and a resource allow-list before the step runs, because job submissions consume real infrastructure and money,
- an **operator-regulator-stop** that lets a human (an operator or a regulator's representative) cancel an in-flight pipeline run without killing the step that is currently executing.

## 3. User-facing flows

The user opens the App UI tab and submits a pipeline request via the form.

1. The system creates a `PipelineRun` record in `PLANNING` and starts a `PipelineWorkflow`.
2. The PlannerAgent drafts a `JobLedger { knownInputs, missingParams, jobPlan, currentDispatch }` and emits `PipelineRunPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - PlannerAgent reads both ledgers and proposes a `JobDispatch { engine, stepSpec, estimatedCostUsd, rationale }`.
   - The **before-tool-call guardrail** checks the dispatch against the cost ceiling and the engine allow-list; on rejection the workflow records a `StepBlocked` entry on the run ledger and asks the PlannerAgent to revise.
   - `JobExecutorAgent` runs the step against seeded fixtures and returns a `JobStepResult`.
   - The **secret sanitizer** scrubs the result for credential-shaped strings.
   - The workflow appends a `RunEntry { engine, stepSpec, attempt, verdict, costConsumed, scrubbedOutput }` to the run ledger.
4. The PlannerAgent decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After two consecutive failures on the same step or two consecutive replans without progress, the PlannerAgent emits `FAIL`.
5. On `COMPLETE`, the PlannerAgent produces a `PipelineOutput { outputLocation, rowsProcessed, totalCostUsd, summary }` and emits `PipelineRunCompleted`. The run moves to `COMPLETED`.
6. The operator can press **Cancel pipeline** in the dashboard at any time. The workflow finishes the in-flight step, then ends with `PipelineRunHaltedOperator`. The run moves to `HALTED`.

A `PipelineSimulator` (TimedAction) drips a sample request every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains the job ledger; reads run ledger. Produces `PipelineOutput` on completion. | `PipelineWorkflow` | returns typed result to workflow |
| `JobExecutorAgent` | `AutonomousAgent` | Runs a single pipeline step (Glue crawler, EMR step, Spark submit, S3 copy) from seeded fixtures. Returns `JobStepResult`. | `PipelineWorkflow` | — |
| `PipelineWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → sanitize → record → decide loop, plus replan and halt branches. | `PipelineEndpoint`, `PipelineRequestConsumer` | `PipelineEntity` |
| `PipelineEntity` | `EventSourcedEntity` | Holds the pipeline run's lifecycle, job ledger, run ledger, and final output. | `PipelineWorkflow` | `PipelineView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `PipelineEndpoint` (operator action) | `PipelineWorkflow` (polls) |
| `PipelineQueue` | `EventSourcedEntity` | Audit log of submitted pipeline requests. | `PipelineEndpoint`, `PipelineSimulator` | `PipelineRequestConsumer` |
| `PipelineView` | `View` | List-of-runs read model for the UI. | `PipelineEntity` events | `PipelineEndpoint` |
| `PipelineRequestConsumer` | `Consumer` | Subscribes to `PipelineQueue` events; starts a `PipelineWorkflow` per submission. | `PipelineQueue` events | `PipelineWorkflow` |
| `PipelineSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/pipeline-requests.jsonl` and enqueues it. | scheduler | `PipelineQueue` |
| `StuckPipelineMonitor` | `TimedAction` | Every 30 s, marks any run stuck in `RUNNING` past 8 minutes as `STUCK`. The workflow polls this and ends with `PipelineRunFailedTimeout`. | scheduler | `PipelineEntity` |
| `PipelineEndpoint` | `HttpEndpoint` | `/api/pipelines/*` — submit, get, list, SSE, operator halt. | — | `PipelineView`, `PipelineQueue`, `PipelineEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record PipelineRequest(String description, String requestedBy, String targetDataset) {}

record JobLedger(
    List<String> knownInputs,
    List<String> missingParams,
    List<String> jobPlan,
    Optional<JobDispatch> currentDispatch
) {}

record JobDispatch(
    JobEngine engine,
    String stepSpec,
    double estimatedCostUsd,
    String rationale
) {}

record JobStepResult(
    JobEngine engine,
    String stepSpec,
    boolean ok,
    String output,
    double actualCostUsd,
    Optional<String> errorReason
) {}

record RunEntry(
    int attempt,
    JobEngine engine,
    String stepSpec,
    RunVerdict verdict,
    String scrubbedOutput,
    double costConsumed,
    Optional<String> blocker,
    Instant recordedAt
) {}

record RunLedger(List<RunEntry> entries) {}

record PipelineOutput(
    String outputLocation,
    long rowsProcessed,
    double totalCostUsd,
    String summary,
    Instant producedAt
) {}

record PipelineRun(
    String runId,
    String description,
    String targetDataset,
    PipelineStatus status,
    Optional<JobLedger> ledger,
    Optional<RunLedger> runLedger,
    Optional<PipelineOutput> output,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobEngine { GLUE_CRAWLER, EMR_STEP, SPARK_SUBMIT, S3_COPY }
enum RunVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, COST_EXCEEDED }
enum PipelineStatus { PLANNING, RUNNING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`PipelineEntity`)

`PipelineRunCreated`, `PipelineRunPlanned`, `JobStepDispatched`, `JobStepBlocked`, `JobStepRecorded`, `JobLedgerRevised`, `PipelineRunCompleted`, `PipelineRunFailed`, `PipelineRunHaltedOperator`, `PipelineRunFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`PipelineQueue`)

`PipelineSubmitted { runId, description, targetDataset, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/pipelines` — body `{ description, requestedBy?, targetDataset }` → `202 { runId }`. Starts a workflow.
- `GET /api/pipelines` — list all runs. Optional `?status=...`.
- `GET /api/pipelines/{id}` — one run (full ledgers + output).
- `GET /api/pipelines/sse` — server-sent events stream of every run state change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Amazon DataProcessing Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a pipeline request, operator halt/resume control, live list of runs with status pills, expand-row to see the job ledger, run ledger entries, and the final output.

Browser title: `<title>Akka Sample: Amazon DataProcessing Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `PlannerAgent`): every `JobDispatch` is checked against (a) the engine allow-list (`GLUE_CRAWLER`, `EMR_STEP`, `SPARK_SUBMIT`, `S3_COPY`), (b) a per-dispatch cost ceiling (`estimatedCostUsd ≤ 5.00` per step by default), and (c) a resource scope that forbids targeting production buckets and clusters listed in `guardrail-config.yaml`. Blocking. Failure → `StepBlocked` entry + replan request.
- **H1 — operator-regulator-stop** (`halt`, flavor `operator-regulator-stop`): the operator or a regulator's representative presses **Cancel pipeline** in the dashboard. `SystemControlEntity` stores the halt flag. The workflow polls the flag at the top of each loop iteration. On `halted=true`, the workflow finishes the in-flight step, then emits `PipelineRunHaltedOperator` and ends with the run in `HALTED`.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Maintains both ledgers; decides next step.
- `JobExecutorAgent` → `prompts/job-executor.md`. Runs a single pipeline step from fixtures.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Crawl the raw-events S3 prefix, run an EMR transform to de-duplicate records, and write the output to the processed/ prefix." Run progresses `PLANNING → RUNNING → COMPLETED` within ~4 minutes. UI reflects each transition via SSE. The expanded view shows a job ledger with a non-empty plan, a run ledger with 3–6 entries spanning at least two engines, and a non-empty `PipelineOutput`.
2. **J2** — Submit a pipeline whose plan includes a step with `estimatedCostUsd = 12.00`. The guardrail blocks the dispatch on the first attempt; the planner either finds a cheaper alternative or fails after the replan budget is exhausted.
3. **J3** — Submit a pipeline and click **Cancel pipeline** while it is `RUNNING`. The in-flight step finishes; no further dispatches occur; the run ends in `HALTED`.
4. **J4** — Submit a pipeline whose fixture step output contains an `AKIA...` key shape. The run ledger entry shows the key replaced by `[REDACTED:aws-access-key]`; the PlannerAgent's next prompt never contains the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named data-processing-planner demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-data-processing-planner.
Java package io.akka.samples.amazondataprocessingagent. Akka 3.6.0. HTTP port 9394.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_OUTPUT).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN returns JobLedger.
    DECIDE returns a NextStep tagged union (ContinueDispatch | RevisePlan |
    Complete | Fail). COMPOSE_OUTPUT returns PipelineOutput.
  * JobExecutorAgent — capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)).
    Prompt from prompts/job-executor.md. Returns JobStepResult.

- 1 Workflow PipelineWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  executeStep -> sanitizeStep -> recordStep -> decideStep
  -> [back to checkHaltStep or to composeOutputStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), executeStep
    ofSeconds(180) (covers job simulation), decideStep ofSeconds(45),
    composeOutputStep ofSeconds(60). defaultStepRecovery(maxRetries(2)
    .failoverTo(PipelineWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits PipelineRunHaltedOperator on PipelineEntity).
  guardrailStep runs CostGuardrail.vet(JobDispatch); on reject records a
  StepBlocked entry via PipelineEntity.recordBlock(stepSpec, reason) and
  loops back to proposeStep.
  executeStep calls forAutonomousAgent(JobExecutorAgent.class, EXECUTE_STEP)
  .runSingleTask(dispatch) then forTask(runId).result(...).
  sanitizeStep applies SecretScrubber.scrub to the JobStepResult.output.
  recordStep calls PipelineEntity.recordStep(entry).
  decideStep calls forAutonomousAgent(PlannerAgent.class, DECIDE);
  on ContinueDispatch or RevisePlan loops; on Complete transitions to
  composeOutputStep -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity PipelineEntity holding PipelineRun state.
  emptyState() returns PipelineRun.initial("", null) with no commandContext()
  reference. Commands: createRun, recordPlan, recordDispatch, recordBlock,
  recordStep, reviseLedger, completeRun, failRun, haltOperator, timeoutFail,
  getRun. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity PipelineQueue with command enqueuePipeline(runId,
  description, targetDataset, requestedBy) emitting PipelineSubmitted.

- 1 View PipelineView with row type PipelineRunRow (mirror of PipelineRun
  minus heavy ledger payloads — truncate to last 3 run entries plus counts;
  the UI fetches the full run by id on click). Table updater consumes
  PipelineEntity events. ONE query getAllRuns SELECT * AS runs FROM
  pipeline_run_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer PipelineRequestConsumer subscribed to PipelineQueue events; on
  PipelineSubmitted starts a PipelineWorkflow with runId as the workflow id.

- 2 TimedActions:
  * PipelineSimulator — every 90s, reads next line from
    src/main/resources/sample-events/pipeline-requests.jsonl and calls
    PipelineQueue.enqueuePipeline.
  * StuckPipelineMonitor — every 30s, queries PipelineView.getAllRuns, filters
    RUNNING runs whose createdAt is older than 8 minutes, calls
    PipelineEntity.timeoutFail; PipelineWorkflow polls PipelineEntity.getRun
    in its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * PipelineEndpoint at /api with POST /pipelines, GET /pipelines (filters
    client-side from getAllRuns), GET /pipelines/{id}, GET /pipelines/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo JobLedger), DECIDE (NextStep), COMPOSE_OUTPUT
  (resultConformsTo PipelineOutput).
- ExecutorTasks.java declaring one Task<R> constant: EXECUTE_STEP
  (resultConformsTo JobStepResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits ContinueDispatch, RevisePlan, Complete, Fail (each carrying its
  own payload — ContinueDispatch with JobDispatch, RevisePlan with revised
  JobLedger, Complete with PipelineOutput stub, Fail with failureReason).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token], [REDACTED:jwt],
  [REDACTED:openai-key], [REDACTED:bearer-token], [REDACTED:high-entropy].
- application/CostGuardrail.java — deterministic vetter. Reject if the
  engine is not in {GLUE_CRAWLER, EMR_STEP, SPARK_SUBMIT, S3_COPY}, if
  estimatedCostUsd > 5.00 (configurable via guardrail-config.yaml), or if
  stepSpec references a bucket or cluster name matching the production
  deny-list in guardrail-config.yaml (e.g., "prod-", "prd-", "-prod").
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9394 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/pipeline-requests.jsonl with 8 canned
  pipeline request descriptions spanning Glue crawler, EMR, Spark, and S3
  copy needs.
- src/main/resources/sample-data/job-fixtures.jsonl — 10 canned job step
  fixture results (one per engine type combination, with realistic output
  formats). One entry must contain an AKIA-shaped key fragment for the J4
  acceptance test.
- src/main/resources/sample-data/guardrail-config.yaml — cost ceiling
  (5.00 USD) and production bucket/cluster deny-list.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/job-executor.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Amazon DataProcessing
  Agent", one-line pitch, prerequisites (including the integration form's
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for ledgers and output). Browser title
  exactly: <title>Akka Sample: Amazon DataProcessing Agent</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java with
  per-agent dispatch on agent class name and Task<R> id. Each branch reads
  from src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes for THIS blueprint:
    planner.json — three lists keyed by task id:
      "PLAN" → 4–6 JobLedger entries (knownInputs, missingParams, jobPlan
      steps spanning all four engines).
      "DECIDE" → 4–6 NextStep entries covering ContinueDispatch (across
      all engines), RevisePlan, Complete, Fail.
      "COMPOSE_OUTPUT" → 4–6 PipelineOutput entries with 60–120 word
      summaries, realistic outputLocation and rowsProcessed values.
    job-executor.json — 8 JobStepResult entries covering all four engines;
      ONE entry's output must contain "AKIAIOSFODNN7EXAMPLE" so the J4
      sanitizer test fires.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, executeStep, decideStep,
  composeOutputStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the PipelineRun entity state.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  ExecutorTasks.java.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9394 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute.
  Exactly five <section class="tab-panel"> elements — no zombie panels.
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
