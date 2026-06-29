# SPEC — nl2sql-data-science

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent NL2SQL Data Science.
**One-line pitch:** Ask a financial question in plain English; a coordinator generates SQL, delegates data retrieval and statistical modeling to parallel workers, then synthesises a data science report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified analysis report. The blueprint also demonstrates a **before-tool-call guardrail** that vets every generated SQL statement for safety before it executes, and an **eval-event** governance control that samples NL2SQL translation decisions for correctness scoring.

## 3. User-facing flows

The user opens the App UI tab and submits a financial question via the form.

1. The system creates an `AnalysisJob` record in `TRANSLATING` and starts an `AnalysisWorkflow`.
2. The Coordinator translates the natural-language question into an `SqlPlan { queryId, sql, targetTable, safetyVerdict }`.
3. The before-tool-call guardrail vets the SQL for destructive keywords (`DROP`, `DELETE`, `TRUNCATE`, `INSERT`, `UPDATE`). If the SQL fails the check, the job moves to `BLOCKED`.
4. The workflow forks: DataRetriever executes the SQL and returns a `DataSet`; StatisticsModeler runs BQML-style computations over the same plan and returns a `ModelResult`. Both run concurrently.
5. The Coordinator merges the two payloads into a `DataReport { summary, dataSet, modelResult, synthesisedAt }`.
6. The job moves to `COMPLETE`.
7. If either worker times out after 60 seconds, the workflow short-circuits: the Coordinator synthesises from whichever side returned, and the job enters `DEGRADED`.

A `QuestionSimulator` (TimedAction) drips a sample financial question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryCoordinator` | `AutonomousAgent` | Translates the NL question to SQL, synthesises the merged report. | `AnalysisWorkflow` | returns typed result to workflow |
| `DataRetriever` | `AutonomousAgent` | Executes the SQL plan against the modelled BigQuery schema and returns a structured dataset. | `AnalysisWorkflow` | — |
| `StatisticsModeler` | `AutonomousAgent` | Applies BQML-style statistical operations to the SQL plan; returns model metrics. | `AnalysisWorkflow` | — |
| `AnalysisWorkflow` | `Workflow` | Coordinates SQL generation, guardrail, parallel fan-out, synthesis. | `AnalysisEndpoint`, `QueryRequestConsumer` | `AnalysisJobEntity` |
| `AnalysisJobEntity` | `EventSourcedEntity` | Holds the job's lifecycle (translating → running → complete / degraded / blocked). | `AnalysisWorkflow` | `AnalysisView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted question for replay/audit. | `AnalysisEndpoint`, `QuestionSimulator` | `QueryRequestConsumer` |
| `AnalysisView` | `View` | List-of-jobs read model. | `AnalysisJobEntity` events | `AnalysisEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Listens to `QueryQueue` events and starts a workflow per submission. | `QueryQueue` events | `AnalysisWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample financial question every 60 s. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Samples one completed job every 5 minutes for NL2SQL correctness scoring; emits a `TranslationEvalScored` event. | scheduler | `AnalysisJobEntity` |
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analysis/*` — submit, get, list, SSE. | — | `AnalysisView`, `QueryQueue`, `AnalysisJobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record NlQuestion(String question, String requestedBy) {}

record SqlPlan(String queryId, String sql, String targetTable, String safetyVerdict) {}

record DataRow(List<String> columns, List<List<String>> rows, Instant queriedAt) {}
record DataSet(String queryId, DataRow data, int rowCount, Instant retrievedAt) {}

record ModelMetric(String metricName, double value, String interpretation) {}
record ModelResult(String modelType, List<ModelMetric> metrics, Instant modelledAt) {}

record DataReport(String summary, DataSet dataSet, ModelResult modelResult,
                  Instant synthesisedAt) {}

record AnalysisJob(
    String jobId,
    String question,
    JobStatus status,
    Optional<SqlPlan> sqlPlan,
    Optional<DataSet> dataSet,
    Optional<ModelResult> modelResult,
    Optional<DataReport> report,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { TRANSLATING, RUNNING, COMPLETE, DEGRADED, BLOCKED }
```

### Events (on `AnalysisJobEntity`)

`JobCreated`, `SqlPlanAttached`, `DataSetAttached`, `ModelResultAttached`, `JobCompleted`, `JobDegraded`, `JobBlocked`, `TranslationEvalScored`.

### Events (on `QueryQueue`)

`QuestionSubmitted { jobId, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/analysis` — body `{ question }` → `{ jobId }`. Starts a workflow.
- `GET /api/analysis` — list all jobs. Optional `?status=TRANSLATING|RUNNING|COMPLETE|DEGRADED|BLOCKED`.
- `GET /api/analysis/{id}` — one job.
- `GET /api/analysis/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent NL2SQL Data Science"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a financial question, live list of jobs with status pills, expand-row to see SQL plan, dataset, model result, synthesised summary, and eval score.

Browser title: `<title>Akka Sample: Multi-Agent NL2SQL Data Science</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call SQL guardrail** (`before-tool-call` on `QueryCoordinator`): vets the generated SQL for destructive statements before any execution attempt. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event NL2SQL correctness** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one completed job every 5 minutes and emits a `TranslationEvalScored` event with a 1–5 score and a short rationale assessing translation fidelity.

## 9. Agent prompts

- `QueryCoordinator` → `prompts/coordinator.md`. Translates NL questions to SQL; later synthesises data and model results into the final report.
- `DataRetriever` → `prompts/data-retriever.md`. Executes the SQL plan against the modelled schema; returns a `DataSet`.
- `StatisticsModeler` → `prompts/statistics-modeler.md`. Applies statistical operations to the SQL plan; returns a `ModelResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a financial question; job progresses TRANSLATING → RUNNING → COMPLETE within 90 s; UI reflects each transition via SSE; the expanded row shows a SQL plan, a DataSet, a ModelResult, and a synthesised summary.
2. **J2** — Inject a worker timeout (set `DataRetriever` timeout to 1 s); job enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a destructive SQL keyword in the NL question (test fixture triggers the guardrail); job enters BLOCKED with a `failureReason`.
4. **J4** — Wait after a successful completion; the job's row in the UI shows a NL2SQL correctness eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named nl2sql-data-science demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-nl2sql-data-science.
Java package io.akka.samples.datascience. Akka 3.6.0. HTTP port 9814.

Components to wire (exactly):
- 3 AutonomousAgents:
  * QueryCoordinator — definition() with capability(TaskAcceptance.of(TRANSLATE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns SqlPlan{queryId, sql, targetTable, safetyVerdict} for
    TRANSLATE and DataReport{summary, dataSet, modelResult, synthesisedAt} for SYNTHESISE.
  * DataRetriever — capability(TaskAcceptance.of(RETRIEVE).maxIterationsPerTask(3)). System prompt
    from prompts/data-retriever.md. Returns DataSet{queryId, data: DataRow{columns, rows,
    queriedAt}, rowCount, retrievedAt}.
  * StatisticsModeler — capability(TaskAcceptance.of(MODEL).maxIterationsPerTask(2)). System prompt
    from prompts/statistics-modeler.md. Returns ModelResult{modelType, metrics:
    List<ModelMetric{metricName, value, interpretation}>, modelledAt}.

- 1 Workflow AnalysisWorkflow with steps:
  translateStep -> guardrailStep -> [parallel] retrieveStep, modelStep -> joinStep -> synthesiseStep -> emitStep.
  translateStep calls forAutonomousAgent(QueryCoordinator.class, TRANSLATE).
  guardrailStep inspects SqlPlan.safetyVerdict and SqlPlan.sql for destructive keywords; on failure
  transitions to blockedStep which calls AnalysisJobEntity.block and ends with JobBlocked.
  retrieveStep and modelStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(AnalysisWorkflow::retrieveStep, ofSeconds(60)) and
  stepTimeout(AnalysisWorkflow::modelStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with JobDegraded.
  synthesiseStep calls forAutonomousAgent(QueryCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. WorkflowSettings is nested inside Workflow —
  no import.

- 1 EventSourcedEntity AnalysisJobEntity holding state AnalysisJob{jobId, question,
  JobStatus, Optional<SqlPlan> sqlPlan, Optional<DataSet> dataSet,
  Optional<ModelResult> modelResult, Optional<DataReport> report,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  JobStatus enum: TRANSLATING, RUNNING, COMPLETE, DEGRADED, BLOCKED. Events:
  JobCreated, SqlPlanAttached, DataSetAttached, ModelResultAttached, JobCompleted,
  JobDegraded, JobBlocked, TranslationEvalScored. Commands: createJob, attachSqlPlan,
  attachDataSet, attachModelResult, completeJob, degradeJob, blockJob,
  recordEval, getJob. emptyState() returns AnalysisJob.initial("", null) with no
  commandContext() reference.

- 1 EventSourcedEntity QueryQueue with command enqueueQuestion(question, requestedBy)
  emitting QuestionSubmitted{jobId, question, requestedBy, submittedAt}.

- 1 View AnalysisView with row type AnalysisJobRow (mirrors AnalysisJob minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  AnalysisJobEntity events. ONE query getAllJobs SELECT * AS jobs FROM analysis_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer QueryRequestConsumer subscribed to QueryQueue events; on QuestionSubmitted
  starts an AnalysisWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/financial-questions.jsonl and calls
    QueryQueue.enqueueQuestion.
  * EvalSampler — every 5 minutes, queries AnalysisView.getAllJobs, picks the oldest
    COMPLETE job without an evalScore, runs a 1–5 NL2SQL correctness rubric judge
    over the sqlPlan and the original question, then calls
    AnalysisJobEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analysis, GET /analysis, GET /analysis/{id},
    GET /analysis/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DataScienceTasks.java declaring four Task<R> constants: TRANSLATE (SqlPlan), RETRIEVE
  (DataSet), MODEL (ModelResult), SYNTHESISE (DataReport).
- Domain records SqlPlan, DataRow, DataSet, ModelMetric, ModelResult, DataReport, NlQuestion.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9814 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/financial-questions.jsonl with 8 canned financial
  question lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-tool-call SQL guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  financial-data-analysis, decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false (except content_generation);
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/data-retriever.md, prompts/statistics-modeler.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent NL2SQL Data Science",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Multi-Agent NL2SQL Data Science</title>. No subtitle on the
  Overview tab.

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
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  data-retriever.json, statistics-modeler.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either SqlPlan or DataReport objects.
      4–6 SqlPlan entries (sql: SELECT statements against financial tables,
      targetTable: one of revenue, expenses, accounts, trades, positions;
      safetyVerdict: "ok") and 4–6 DataReport entries (each with an 80–150 word
      summary of a financial analysis finding, a non-null dataSet, a non-null
      modelResult).
    data-retriever.json — 4–6 DataSet entries, each with columns matching
      financial schema (e.g., ["date","symbol","close_price","volume"]),
      3–10 data rows, rowCount matching row length.
    statistics-modeler.json — 4–6 ModelResult entries, each with a modelType
      from ["linear_regression","time_series_forecast","clustering","anomaly_detection"]
      and 2–4 ModelMetric entries with realistic finance metric names (r_squared,
      mae, silhouette_score, anomaly_count).
- A MockModelProvider.seedFor(jobId) helper makes the selection deterministic
  per job id so the same job produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion DataScienceTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9814 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
