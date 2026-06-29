# SPEC — durable-weather-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DurableWeatherAgent.
**One-line pitch:** A caller submits a weather query through the Workflow API; one AI agent fetches weather data via typed tool calls (each call is a durable workflow activity) and returns a structured `WeatherReport` — current conditions, a short forecast narrative, and per-tool evidence — with the agent accessible only through two invocation modes: `trigger_agent` (fire-and-wait) and `call_agent` (child workflow).

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain, with the defining twist that the agent is **Workflow-backed**: the caller never calls the agent component directly. Instead it talks to `AgentWorkflow`, which runs the agent as a set of activities. Two entry points show different composition styles:

- **`trigger_agent`** — the caller POSTs a query and polls (or holds an SSE stream) until the job reaches a terminal state. The workflow runs to completion and the caller reads the final `WeatherReport`.
- **`call_agent`** — a parent workflow calls `AgentWorkflow` as a child. The parent's step suspends until the child completes, then incorporates the child's report. This shows agent-as-subroutine composition without actor coupling.

One governance mechanism is wired:

- A **before-tool-call guardrail** runs inside `WeatherToolActivity` before any outbound HTTP call fires. It inspects the tool's arguments (city name, coordinate bounding-box, time horizon) and rejects calls whose arguments match a block-list (e.g., coordinates outside the configured bounding box, a forecast horizon exceeding 14 days, or a city name that resolves to a restricted jurisdiction). On rejection the agent loop sees a structured error and may retry with corrected arguments.

The blueprint shows that durability (workflow activities) and governance (guardrail) compose naturally: the activity boundary is the natural place for the guardrail to sit — after the agent decides to call a tool but before the call leaves the process.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **Weather query** (e.g., "current conditions and 3-day forecast for Berlin") into the query textarea, or picks one of four seeded cities (Berlin, Tokyo, São Paulo, Nairobi).
2. The user picks an **invocation mode**: `trigger_agent` (default) or `call_agent` (demonstrates child-workflow composition).
3. The user clicks **Submit query**. The UI POSTs to `/api/jobs` and receives a `jobId`.
4. The job card appears in the live list in `QUEUED` state. Within ~1 s it transitions to `RUNNING`. The progress row shows which tool call is currently active.
5. Within ~10–30 s the workflow's tool calls complete and the agent assembles the report. The card transitions to `COMPLETED`. The right pane shows: a conditions summary, a forecast narrative, and a per-tool evidence table (tool name, arguments used, raw response snippet).
6. Any tool call that was blocked by the guardrail is visible in the card as a `BLOCKED` row in the evidence table, with the guardrail rejection reason alongside.
7. The user can submit another query; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `WeatherJobEntity`, `JobView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `WeatherJobEntity` | `EventSourcedEntity` | Per-job lifecycle: queued → running → completed / failed. Source of truth. | `JobEndpoint`, `AgentWorkflow` | `JobView` |
| `AgentWorkflow` | `Workflow` | One workflow per job. Supports `triggerAgent` and `callAgent` entry transitions. Steps: `startAgentStep` → `runAgentStep` → `finalizeStep`. | started by `JobEndpoint` or a parent workflow | `WeatherAgent`, `WeatherJobEntity` |
| `WeatherAgent` | `AutonomousAgent` | The one decision-making LLM. Receives a weather query as the task instruction; executes `get_current_weather`, `get_forecast`, `get_weather_alerts` tool calls via `WeatherToolActivity`; returns `WeatherReport`. | invoked by `AgentWorkflow` | `WeatherToolActivity` (tool calls), returns report |
| `WeatherToolActivity` | Workflow activity (inner class on `AgentWorkflow`) | Executes each tool call as a durable activity. The `before-tool-call` guardrail runs here before any HTTP call fires. | `WeatherAgent` tool calls | external weather API (simulated in dev) |
| `ToolCallGuardrail` | Supporting class | Inspects tool-call arguments; returns pass or structured rejection. | called by `WeatherToolActivity` | returns to `WeatherAgent` |
| `JobView` | `View` | Read model: one row per job for the UI. | `WeatherJobEntity` events | `JobEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WeatherQuery(
    String jobId,
    String queryText,
    String city,             // resolved by the agent; null until agent runs
    InvocationMode mode,
    String submittedBy,
    Instant submittedAt
) {}
enum InvocationMode { TRIGGER_AGENT, CALL_AGENT }

record ToolCallRecord(
    String toolName,
    String argumentsSummary,
    ToolCallStatus status,
    Optional<String> rawResponseSnippet,
    Optional<String> guardRejectionReason,
    Instant calledAt
) {}
enum ToolCallStatus { EXECUTED, BLOCKED, FAILED }

record WeatherConditions(
    String city,
    String country,
    double temperatureCelsius,
    String conditionLabel,   // e.g. "Partly cloudy"
    int humidityPercent,
    double windSpeedKph,
    Instant observedAt
) {}

record ForecastDay(
    String date,             // ISO-8601 date
    double highCelsius,
    double lowCelsius,
    String conditionLabel,
    int precipitationChancePercent
) {}

record WeatherReport(
    String jobId,
    WeatherConditions current,
    List<ForecastDay> forecast,
    List<String> activeAlerts,
    String narrativeSummary,
    List<ToolCallRecord> toolCallLog,
    Instant completedAt
) {}

record WeatherJob(
    String jobId,
    Optional<WeatherQuery> query,
    Optional<WeatherReport> report,
    JobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus {
    QUEUED, RUNNING, COMPLETED, FAILED
}
```

Events on `WeatherJobEntity`: `JobQueued`, `JobStarted`, `JobCompleted`, `JobFailed`.

Every nullable lifecycle field on the `WeatherJob` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ queryText, city, mode: "TRIGGER_AGENT" | "CALL_AGENT", submittedBy }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Durable Weather Agent</title>`.

The App UI tab is a two-column layout: a left column with the query submission panel and the live list of jobs (status pill, mode badge, age) and a right pane with the selected job's detail — invocation mode badge, query text, conditions summary card, forecast table, tool-call evidence table with BLOCKED rows highlighted, and narrative summary paragraph.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs inside `WeatherToolActivity` before any HTTP call fires. Inspects the three weather tool argument sets: for `get_current_weather` checks the `city` field is non-empty and not in the restricted-location list; for `get_forecast` additionally checks that `days` is in `[1, 14]`; for `get_weather_alerts` checks that the `bbox` (bounding box) stays within the configured max-area threshold. On failure returns a structured rejection `{ tool, argument, reason }` to the agent loop; the agent may rephrase and retry within its iteration budget. Passing calls proceed to the HTTP activity.

## 9. Agent prompts

- `WeatherAgent` → `prompts/weather-agent.md`. The single decision-making LLM. System prompt instructs it to resolve the user's query to a city, call the weather tools in order, and assemble a typed `WeatherReport`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a Berlin query via `trigger_agent`; within 30 s the job reaches `COMPLETED` with a conditions block, a 3-day forecast, and a narrative summary.
2. **J2** — User submits a Tokyo query via `call_agent`; the parent workflow spawns the child, waits for it, and the UI shows the child job completing before the parent's job card updates.
3. **J3** — User submits a query with an extreme forecast horizon (e.g., "30-day forecast"); the `before-tool-call` guardrail blocks the `get_forecast` call with `days=30`; the agent retries with `days=14`; the card shows a BLOCKED row in the evidence table alongside the successful EXECUTED row.
4. **J4** — Service restarts with a job in RUNNING state; the workflow resumes from the last completed activity without re-running earlier tool calls; the job reaches COMPLETED normally.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named durable-weather-agent demonstrating the single-agent × general cell with
Workflow-backed agent invocation. Runs out of the box (no external services — weather API calls
are simulated). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-durable-tool-agent. Java package
io.akka.samples.durableweatheragentviaworkflowapi. Akka 3.6.0. HTTP port 9227.

Components to wire (exactly):

- 1 AutonomousAgent WeatherAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-agent.md>) and
  .capability(TaskAcceptance.of(FETCH_WEATHER_REPORT).maxIterationsPerTask(4)). The task
  receives the weather query as its instruction text. The agent has three tool calls:
  get_current_weather(city: String), get_forecast(city: String, days: int),
  get_weather_alerts(city: String, bbox: String). Each tool call delegates to
  WeatherToolActivity inside AgentWorkflow. Output: WeatherReport{jobId, current:
  WeatherConditions, forecast: List<ForecastDay>, activeAlerts: List<String>,
  narrativeSummary: String, toolCallLog: List<ToolCallRecord>, completedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the
  agent loop sees a structured {tool, argument, reason} error and retries within its
  4-iteration budget.

- 1 Workflow AgentWorkflow per jobId. Supports two entry transitions:
  * triggerAgent — called by JobEndpoint. Transition sequence:
    startAgentStep → runAgentStep → finalizeStep → done.
  * callAgent — called by a parent workflow step. Same internal sequence; the parent
    suspends on asyncCall(AgentWorkflow::callAgent, childJobId) and resumes when the
    child workflow transitions to done.
  Steps:
    * startAgentStep — emits JobStarted on WeatherJobEntity; advances to runAgentStep.
      WorkflowSettings.stepTimeout 10s.
    * runAgentStep — calls componentClient.forAutonomousAgent(WeatherAgent.class,
      "weather-" + jobId).runSingleTask(TaskDef.instructions(query.queryText)) —
      returns a taskId, then forTask(taskId).result(FETCH_WEATHER_REPORT) to fetch the
      WeatherReport. On success advances to finalizeStep. WorkflowSettings.stepTimeout
      120s with defaultStepRecovery maxRetries(2).failoverTo(AgentWorkflow::error).
    * finalizeStep — calls WeatherJobEntity.complete(report). WorkflowSettings
      .stepTimeout 10s.
    * error step — calls WeatherJobEntity.fail(reason). WorkflowSettings.stepTimeout 5s.
  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- WeatherToolActivity — inner class of AgentWorkflow. One method per tool call:
  executeGetCurrentWeather(String city), executeGetForecast(String city, int days),
  executeGetWeatherAlerts(String city, String bbox). Before each HTTP call, instantiates
  ToolCallGuardrail and calls guardrail.check(toolName, arguments). On rejection records
  a ToolCallRecord with status=BLOCKED and guardRejectionReason set; returns the rejection
  to the agent loop. On pass proceeds with the simulated HTTP call (returns deterministic
  fixture data keyed by city name for local dev), records status=EXECUTED, and returns the
  raw response snippet.

- ToolCallGuardrail.java — pure logic class. check(String toolName, Map<String,Object>
  args) inspects:
    get_current_weather: city must be non-empty and not in RESTRICTED_CITIES set.
    get_forecast: same city check PLUS days in [1, 14].
    get_weather_alerts: same city check PLUS bbox area (lat delta × lon delta) ≤ 25 sq deg.
  Returns GuardrailResult{pass: boolean, rejectionCode: String, rejectionDetail: String}.

- 1 EventSourcedEntity WeatherJobEntity (one per jobId). State WeatherJob{jobId: String,
  query: Optional<WeatherQuery>, report: Optional<WeatherReport>, status: JobStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. JobStatus enum: QUEUED, RUNNING,
  COMPLETED, FAILED. Events: JobQueued{query}, JobStarted{}, JobCompleted{report},
  JobFailed{reason}. Commands: queue, start, complete, fail, getJob. emptyState() returns
  WeatherJob.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View JobView with row type JobRow (mirrors WeatherJob). Table updater consumes
  WeatherJobEntity events. ONE query getAllJobs: SELECT * AS jobs FROM job_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs (body {queryText, city, mode, submittedBy};
    mints jobId; calls WeatherJobEntity.queue; starts AgentWorkflow.triggerAgent or
    AgentWorkflow.callAgent based on mode; returns {jobId}), GET /jobs (list from
    getAllJobs, sorted newest-first), GET /jobs/{id} (one row), GET /jobs/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- WeatherTasks.java declaring one Task<R> constant: FETCH_WEATHER_REPORT =
  Task.name("Fetch weather report").description("Resolve the city, call the weather
  tools, and assemble a WeatherReport").resultConformsTo(WeatherReport.class). DO NOT
  skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records WeatherQuery, InvocationMode, ToolCallRecord, ToolCallStatus,
  WeatherConditions, ForecastDay, WeatherReport, WeatherJob, JobStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9227 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  WeatherAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/seed-queries.jsonl with 4 seeded city queries:
  Berlin (current + 3-day), Tokyo (current + 5-day), São Paulo (current + alerts),
  Nairobi (current only). Each maps to a fixture response in the simulated API.

- src/main/resources/weather-fixtures/cities.json — deterministic fixture data for
  the 4 seeded cities plus a generic fallback. Each entry has WeatherConditions and
  a 7-day ForecastDay list so any horizon ≤ 7 is satisfied without a network call.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with domain=general, decisions.authority_level=
  inform-only (the report informs the user; no action is taken on their behalf),
  oversight.human_in_loop=false (fully automated), failure.failure_modes including
  "hallucinated-conditions", "stale-data-presented-as-current",
  "guardrail-bypass-via-indirect-location", "tool-call-loop";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/weather-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Durable Weather Agent via
  Workflow API", prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = query form + live list of job cards; right = selected-job detail with
  invocation mode badge, query text, conditions summary card, forecast table, tool-call
  evidence table, narrative summary). Browser title exactly:
  <title>Akka Sample: Durable Weather Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using
        the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session
        ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(jobId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    fetch-weather-report.json — 6 WeatherReport entries covering all 4 seeded cities
    plus 2 generic entries. Each entry has a populated WeatherConditions block, a
    List<ForecastDay> of 3 days, a non-empty narrativeSummary, and a toolCallLog with
    EXECUTED entries for each tool called. Plus 1 entry whose toolCallLog contains a
    BLOCKED row (days=30 blocked by the guardrail) followed by an EXECUTED row
    (days=14 retry) — exercising J3 reproducibly.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion WeatherTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (startAgentStep 10s, runAgentStep 120s, finalizeStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the WeatherJob row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: WeatherTasks.java with FETCH_WEATHER_REPORT = Task.name(...).description(...)
  .resultConformsTo(WeatherReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9227 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements —
  Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WeatherAgent). The
  ToolCallGuardrail is a pure logic class, not an agent — it makes no LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, bound to the before-tool-call hook, NOT as a post-call check or an
  external filter.
- The callAgent entry point must demonstrate child-workflow composition: a parent
  workflow step calls asyncCall(AgentWorkflow::callAgent, childJobId) and the child
  workflow resolves independently. The UI must show the invocation mode as a badge on
  each job card.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
