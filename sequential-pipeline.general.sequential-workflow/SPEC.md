# SPEC — sequential-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Sequential Workflow.
**One-line pitch:** A user submits a job definition; one `WorkflowAgent` walks it through four task phases — **VALIDATE** inputs, **ENRICH** with context, **EXECUTE** the work steps, **SUMMARIZE** outcomes — with each phase gated on the prior phase's recorded output and each phase's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a general-purpose workflow domain. One `WorkflowAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the VALIDATE task's typed output becomes the ENRICH task's instruction context; the ENRICH task's typed output becomes the EXECUTE task's instruction context; and so on through SUMMARIZE. The agent never holds all four phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`VALIDATE` / `ENRICH` / `EXECUTE` / `SUMMARIZE`) and the current `JobEntity` status. An EXECUTE-phase tool called while the entity has not yet recorded `JobEnriched` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The same hook enforces the deeper pipeline property: a phase's tool set is unreachable until its preconditions hold.
- An **`on-completion-eval`** runs immediately after `JobSummarized` lands, as `evalStep` inside the workflow. A deterministic, rule-based `QualityScorer` (no LLM call — the eval is rule-based on purpose, so the same job always scores the same) checks that every execution step has a matching summary entry, that every summary entry references at least one output artifact, that no artifact referenced in a summary entry was absent from the `JobOutput`, and that the step count in the summary equals the step count in the execution output.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the per-phase tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **job name** and selects a **job type** (or picks one of three seeded types — `csv-transform`, `report-generation`, `data-validation-run`).
2. The user clicks **Run workflow**. The UI POSTs to `/api/jobs` and receives a `jobId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `VALIDATING` — the workflow has started `validateStep` and the agent has been handed the VALIDATE task.
4. Within ~10–20 s the card reaches `ENRICHING` — the typed `ValidationResult` is visible in the card detail (a small table of validated fields with status and notes). The agent's VALIDATE task returned; the workflow recorded `JobValidated` and ran the ENRICH task.
5. Within ~10–20 s more the card reaches `EXECUTING`. The `EnrichedJob` is visible (context map + resolved parameters).
6. Within ~10–20 s more the card reaches `SUMMARIZING`. The `JobOutput` is visible (step list + artifact list).
7. Within ~10–20 s more the card reaches `EVALUATED`. The right pane shows the full typed `JobSummary` — title, outcome statement, per-step `SummaryEntry` with body and artifact references — plus an eval score chip (1–5) and a one-line rationale.
8. The user can submit another job; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `JobEntity`, `JobView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `JobEntity` | `EventSourcedEntity` | Per-job lifecycle: created → validating → validated → enriching → enriched → executing → executed → summarizing → summarized → evaluated. Source of truth. | `JobEndpoint`, `JobPipelineWorkflow` | `JobView` |
| `JobPipelineWorkflow` | `Workflow` | One workflow per job. Steps: `validateStep` → `enrichStep` → `executeStep` → `summarizeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `JobEndpoint` after `CREATED` | `WorkflowAgent`, `JobEntity` |
| `WorkflowAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `WorkflowTasks.java`: `VALIDATE_JOB` → `ValidationResult`, `ENRICH_JOB` → `EnrichedJob`, `EXECUTE_JOB` → `JobOutput`, `SUMMARIZE_JOB` → `JobSummary`. Each task is registered with the phase-appropriate function tools. | invoked by `JobPipelineWorkflow` | returns typed results |
| `ValidateTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `checkFields(jobSpec)` and `verifyConstraints(fields)`. Reads from `src/main/resources/sample-data/job-types/*.json` for deterministic offline output. | called from VALIDATE task | returns `List<FieldResult>` |
| `EnrichTools` | function-tools class | Implements `resolveParameters(validatedFields)` and `attachContext(params)`. Pure in-memory transformations. | called from ENRICH task | returns `List<ResolvedParam>` / `ContextMap` |
| `ExecuteTools` | function-tools class | Implements `runStep(step, context)` and `collectArtifacts(stepResults)`. | called from EXECUTE task | returns `StepResult` / `List<Artifact>` |
| `SummarizeTools` | function-tools class | Implements `buildSummaryEntry(step, artifacts)` and `writeOutcomeStatement(entries)`. | called from SUMMARIZE task | returns `SummaryEntry` / `String` |
| `StepGuardrail` | `before-tool-call` guardrail (registered on `WorkflowAgent`) | Reads the in-flight task's declared phase and the current `JobEntity` status. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-completion evaluator. Inputs: `JobSummary`, `JobOutput`, `EnrichedJob`. Output: `QualityResult{score, rationale}`. | called from `evalStep` | returns score |
| `JobView` | `View` | Read model: one row per job for the UI. | `JobEntity` events | `JobEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record FieldResult(String fieldName, String status, String note) {}

record ValidationResult(List<FieldResult> fieldResults, boolean valid, Instant validatedAt) {}

record ResolvedParam(String key, String value, String source) {}

record ContextMap(List<ResolvedParam> params, Map<String, String> metadata, Instant resolvedAt) {}

record EnrichedJob(ValidationResult validation, ContextMap context, Instant enrichedAt) {}

record Artifact(String artifactId, String name, String kind, String ref) {}

record StepResult(String stepId, String stepName, String outcome, List<Artifact> artifacts) {}

record JobOutput(List<StepResult> steps, Instant executedAt) {}

record SummaryEntry(String stepId, String heading, String body, List<String> artifactIds) {}

record JobSummary(
    String title,
    String outcomeStatement,
    List<SummaryEntry> entries,
    Instant summarizedAt
) {}

record QualityResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record JobSpec(String jobName, String jobType, Map<String, String> parameters) {}

record JobRecord(
    String jobId,
    Optional<JobSpec> spec,
    Optional<ValidationResult> validation,
    Optional<EnrichedJob> enrichedJob,
    Optional<JobOutput> output,
    Optional<JobSummary> summary,
    Optional<QualityResult> quality,
    JobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus {
    CREATED, VALIDATING, VALIDATED, ENRICHING, ENRICHED,
    EXECUTING, EXECUTED, SUMMARIZING, SUMMARIZED, EVALUATED, FAILED
}
```

Events on `JobEntity`: `JobCreated`, `ValidateStarted`, `JobValidated`, `EnrichStarted`, `JobEnriched`, `ExecuteStarted`, `JobExecuted`, `SummarizeStarted`, `JobSummarized`, `QualityScored`, `GuardrailRejected`, `JobFailed`.

Every nullable lifecycle field on the `JobRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ jobName, jobType, parameters }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sequential Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of jobs (status pill + job name + age) and a right pane with the selected job's detail — spec fields, validation table, enriched context parameters, execution steps with artifacts, summary entries, eval score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate)**: `StepGuardrail` is registered on `WorkflowAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.VALIDATE`, `Phase.ENRICH`, `Phase.EXECUTE`, `Phase.SUMMARIZE`) and the current `JobEntity.status` for the job the task is bound to. The accept rule is precise: `VALIDATE` tools require `status ∈ {CREATED, VALIDATING}`; `ENRICH` tools require `status ∈ {VALIDATED, ENRICHING}` AND `validation.isPresent()`; `EXECUTE` tools require `status ∈ {ENRICHED, EXECUTING}` AND `enrichedJob.isPresent()`; `SUMMARIZE` tools require `status ∈ {EXECUTED, SUMMARIZING}` AND `output.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-completion-eval`**: runs immediately after `JobSummarized` lands, as `evalStep` inside the workflow. `QualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every execution step must have a matching summary entry (step coverage), every summary entry must reference at least one artifact (artifact attribution), every referenced artifact id must appear in the recorded `JobOutput.steps[].artifacts` (no invented references), and the entry count must equal the step count (no silent expansion or collapse). Emits `QualityScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `WorkflowAgent` → `prompts/workflow-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded job type `csv-transform`; within 80 s the job reaches `EVALUATED` with a non-empty validation result, ≥ 2 execution steps, ≥ 2 summary entries, and an eval score chip on the card.
2. **J2** — The agent's first iteration on a job calls an EXECUTE-phase tool (`runStep`) before `JobEnriched` has been recorded (mock LLM path). `StepGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the job eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A job whose mock-LLM trajectory produces a summary entry referencing an artifact id absent from the recorded `JobOutput` is scored 1 with a rationale naming the missing artifact; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-job trace (logged at `INFO`); the VALIDATE task's log shows only VALIDATE-tool calls, the ENRICH task's log shows only ENRICH-tool calls, the EXECUTE task's log shows only EXECUTE-tool calls, and the SUMMARIZE task's log shows only SUMMARIZE-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sequential-workflow demonstrating the sequential-pipeline x general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-sequential-workflow. Java package
io.akka.samples.workflowssequential. Akka 3.6.0. HTTP port 9509.

Components to wire (exactly):

- 1 AutonomousAgent WorkflowAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/workflow-agent.md>) and four .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  VALIDATE, ENRICH, EXECUTE, and SUMMARIZE tool sets are ALL registered on the agent; phase
  gating is the job of StepGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (StepGuardrail) is registered on the agent via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow JobPipelineWorkflow per jobId with five steps:
  * validateStep — emits ValidateStarted on the entity, then calls componentClient
    .forAutonomousAgent(WorkflowAgent.class, "agent-" + jobId).runSingleTask(
      TaskDef.instructions("Job: " + jobName + " (" + jobType + ")\nPhase: VALIDATE\nCheck
      the provided fields against the job type constraints.")
        .metadata("jobId", jobId)
        .metadata("phase", "VALIDATE")
        .taskType(WorkflowTasks.VALIDATE_JOB)
    ). Reads forTask(taskId).result(VALIDATE_JOB) to get ValidationResult. Writes
    JobEntity.recordValidation(validationResult). WorkflowSettings.stepTimeout 60s.
  * enrichStep — emits EnrichStarted, then runSingleTask with TaskDef.instructions
    (formatEnrichContext(validationResult, jobSpec)) and metadata.phase = "ENRICH", taskType
    ENRICH_JOB. Writes JobEntity.recordEnrichment(enrichedJob). stepTimeout 60s.
  * executeStep — emits ExecuteStarted, then runSingleTask with TaskDef.instructions
    (formatExecuteContext(enrichedJob, jobSpec)) and metadata.phase = "EXECUTE", taskType
    EXECUTE_JOB. Writes JobEntity.recordOutput(jobOutput). stepTimeout 60s.
  * summarizeStep — emits SummarizeStarted, then runSingleTask with TaskDef.instructions
    (formatSummarizeContext(jobOutput, enrichedJob, jobSpec)) and metadata.phase = "SUMMARIZE",
    taskType SUMMARIZE_JOB. Writes JobEntity.recordSummary(jobSummary). stepTimeout 60s.
  * evalStep — runs the deterministic QualityScorer over (jobSummary, jobOutput, enrichedJob)
    and writes JobEntity.recordQuality(qualityResult). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(JobPipelineWorkflow::error). The error step writes
  JobFailed and ends.

- 1 EventSourcedEntity JobEntity (one per jobId). State JobRecord{jobId,
  spec: Optional<JobSpec>, validation: Optional<ValidationResult>,
  enrichedJob: Optional<EnrichedJob>, output: Optional<JobOutput>,
  summary: Optional<JobSummary>, quality: Optional<QualityResult>,
  status: JobStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  JobStatus enum: CREATED, VALIDATING, VALIDATED, ENRICHING, ENRICHED, EXECUTING, EXECUTED,
  SUMMARIZING, SUMMARIZED, EVALUATED, FAILED. Events:
  JobCreated{jobSpec}, ValidateStarted, JobValidated{validation}, EnrichStarted,
  JobEnriched{enrichedJob}, ExecuteStarted, JobExecuted{output}, SummarizeStarted,
  JobSummarized{summary}, QualityScored{quality}, GuardrailRejected{phase, tool, reason},
  JobFailed{reason}.
  Commands: create, startValidate, recordValidation, startEnrich, recordEnrichment,
  startExecute, recordOutput, startSummarize, recordSummary, recordQuality,
  recordGuardrailRejection, fail, getJob. emptyState() returns JobRecord.initial("") with
  all Optional fields as Optional.empty() and no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View JobView with row type JobRow that mirrors JobRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes JobEntity events. ONE query
  getAllJobs: SELECT * AS jobs FROM job_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs (body {jobName, jobType, parameters}; mints jobId;
    calls JobEntity.create(jobSpec); then starts JobPipelineWorkflow with id
    "pipeline-" + jobId; returns {jobId}), GET /jobs (list from getAllJobs, sorted
    newest-first), GET /jobs/{id} (one row), GET /jobs/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- WorkflowTasks.java declaring four Task<R> constants:
    VALIDATE_JOB = Task.name("Validate job").description("Check all job fields against
      the job type's declared constraints and return a ValidationResult")
      .resultConformsTo(ValidationResult.class);
    ENRICH_JOB = Task.name("Enrich job").description("Resolve parameters and attach
      contextual metadata to produce an EnrichedJob")
      .resultConformsTo(EnrichedJob.class);
    EXECUTE_JOB = Task.name("Execute job").description("Run each declared step and collect
      output artifacts to produce a JobOutput")
      .resultConformsTo(JobOutput.class);
    SUMMARIZE_JOB = Task.name("Summarize job").description("Build a per-step SummaryEntry
      for every executed step and compose the outcome statement")
      .resultConformsTo(JobSummary.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {VALIDATE, ENRICH, EXECUTE, SUMMARIZE}. Each function-tool method is
  annotated with the constant phase, e.g. @FunctionTool(name = "checkFields",
  phase = Phase.VALIDATE) (use a custom annotation if the SDK's @FunctionTool does not carry
  a phase field — the guardrail reads it from a parallel registry built at startup if so).

- ValidateTools.java — @FunctionTool checkFields(JobSpec spec) -> List<FieldResult> reading
  from src/main/resources/sample-data/job-types/*.json keyed by jobType; @FunctionTool
  verifyConstraints(List<FieldResult> fields) -> ValidationResult assembling the result.

- EnrichTools.java — @FunctionTool resolveParameters(ValidationResult v) ->
  List<ResolvedParam> (one ResolvedParam per field, source = "validation"); @FunctionTool
  attachContext(List<ResolvedParam> params) -> ContextMap (deterministic metadata
  assignment by jobType; typical 2-3 metadata entries).

- ExecuteTools.java — @FunctionTool runStep(String stepId, String stepName,
  ContextMap context) -> StepResult (deterministic in-process step execution; artifacts
  minted as "a-" + sha1(stepId + stepName).substring(0,8)); @FunctionTool
  collectArtifacts(List<StepResult> steps) -> List<Artifact> (flattened artifact list
  across all steps).

- SummarizeTools.java — @FunctionTool buildSummaryEntry(StepResult step,
  List<Artifact> artifacts) -> SummaryEntry (heading from stepName, body composed from
  outcome and artifact names, artifactIds from artifacts[].artifactId); @FunctionTool
  writeOutcomeStatement(List<SummaryEntry> entries) -> String (one-sentence outcome
  statement across all steps).

- StepGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the JobEntity status by jobId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>,
  saw <status>"). On reject ALSO calls JobEntity.recordGuardrailRejection(phase, tool,
  reason) so the rejection is visible in the UI's rejection-log strip and in the audit log.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: JobSummary, JobOutput,
  EnrichedJob. Outputs: QualityResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1: step coverage (every StepResult has a
  SummaryEntry), artifact attribution (every SummaryEntry references ≥ 1 artifactId),
  artifact provenance (every referenced artifactId appears in JobOutput.steps[].artifacts[].
  artifactId), and entry parity (entries.size() == steps.size()). Score range 1-5. Rationale
  names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9509 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/job-types.jsonl with 5 seeded job type lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/job-types/*.json — three files keyed by seeded job type,
  each carrying 3-5 field definitions with deterministic constraint sets so ValidateTools
  returns the same result across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (job definitions
  are non-personal), decisions.authority_level = recommend-only (the job result is advisory),
  oversight.human_in_loop = true, operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "missing-artifact-reference", "missing-step-coverage", "phase-violation",
  "empty-execution-step"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/workflow-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Sequential Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of job cards; right = selected-job detail with spec fields, validation table,
  enriched context parameters, execution steps, summary entries, eval-score chip,
  rejection-log strip). Browser title exactly: <title>Akka Sample: Sequential Workflow</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(jobId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    validate-job.json — 5 ValidationResult entries, each for a different seeded job type.
      Each entry's tool_calls array contains checkFields(jobSpec) + verifyConstraints(fields)
      in order. Plus 1 deliberately PHASE-VIOLATING entry whose tool_calls array starts with
      a runStep(...) call (an EXECUTE-phase tool called during the VALIDATE phase) — the
      guardrail rejects it, the mock then falls through to a normal validate sequence. The
      mock should select the violating entry on the FIRST iteration of every 3rd job
      (modulo seed) so J2 is reproducible.
    enrich-job.json — 5 EnrichedJob entries paired one-to-one with the validate entries,
      each with 2-4 ResolvedParam items and a ContextMap with 2 metadata entries, with
      tool_calls containing resolveParameters + attachContext in order.
    execute-job.json — 5 JobOutput entries paired one-to-one. Each carries 2-4 StepResult
      items with artifact lists, tool_calls containing runStep (one per step) + collectArtifacts.
      Plus 1 deliberately MISSING-ARTIFACT-REFERENCE entry whose first SummaryEntry (in
      the paired summarize-job entry) references an artifactId absent from this
      JobOutput — the workflow's evalStep scores it 1; J3 verifies this.
    summarize-job.json — 5 JobSummary entries paired one-to-one with the execute entries.
      Each carries one SummaryEntry per StepResult, with tool_calls containing
      buildSummaryEntry (one per step) + writeOutcomeStatement.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WorkflowAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion WorkflowTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (validateStep
  60s, enrichStep 60s, executeStep 60s, summarizeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the JobRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: WorkflowTasks.java with VALIDATE_JOB, ENRICH_JOB, EXECUTE_JOB, SUMMARIZE_JOB
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9509 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WorkflowAgent). The
  on-completion eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (StepGuardrail) is the runtime mechanism that enforces
  the phase order. Do NOT conditionally register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: validateStep writes ValidationResult
  onto the entity, enrichStep reads it and builds the ENRICH task's instruction context,
  executeStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
- folder: sequential-pipeline.general.sequential-workflow
- Maven group: io.akka.samples, artifact: sequential-pipeline-general-sequential-workflow
- Java package: io.akka.samples.workflowssequential
- Akka version: 3.6.0
- HTTP port: 9509
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
