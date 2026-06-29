# SPEC — any-llm-tool-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AnyLLMToolAgent.
**One-line pitch:** A user submits a natural-language weather query; one AI agent resolves the location, calls the `get_weather` tool, and returns a structured `WeatherReport` — while the LLM backend (InferenceClient, Transformers, Ollama, LiteLLM, or OpenAI) is a pure configuration switch.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `WeatherAgent` (AutonomousAgent) carries the entire decision; the surrounding components only accept the query, queue the agent call, and persist the result. One governance mechanism is wired:

- A **before-tool-call guardrail** runs before each `get_weather` invocation — so a blank, nonsensical, or suspiciously long location string never reaches the tool (or any downstream HTTP call). The guardrail forces the agent to request clarification rather than pass garbage to an external service.

The blueprint's primary educational goal is the backend-agnostic wiring: the same Java `AutonomousAgent` class runs against any supported model provider with no code changes. A secondary goal is showing how a tool-calling agent is governed at the tool boundary, not just at the response boundary.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a weather query into the **Query** textarea (e.g., "What is the weather in Tokyo right now?") or picks one of three seeded examples: Tokyo / London / São Paulo.
2. The user picks a **Backend** from a dropdown (anthropic / openai / googleai-gemini / ollama / mock) to illustrate the backend-agnostic property. The dropdown maps to the `application.conf` model-provider block.
3. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s the workflow's `invokeStep` starts and the card transitions to `INVOKING`.
5. Within ~10–30 s the agent resolves the location, calls `get_weather`, and returns a `WeatherReport`. The card transitions to `RESULT_RECORDED`. The report appears: location header, condition badge (SUNNY / CLOUDY / RAINY / SNOWY / UNKNOWN), temperature (°C), humidity (%), wind speed (km/h), and a one-sentence narrative summary.
6. The user can submit more queries; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → invoking → result recorded → failed. Source of truth. | `QueryEndpoint`, `QueryWorkflow` | `QueryView` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `invokeStep` → `recordStep`. | started by `QueryEndpoint` after `QueryEntity.submit` | `WeatherAgent`, `QueryEntity` |
| `WeatherAgent` | `AutonomousAgent` | The one decision-making LLM. Calls `get_weather` tool. Returns `WeatherReport`. | invoked by `QueryWorkflow` | returns report |
| `WeatherToolGuardrail` | supporting class | `before-tool-call` hook on `WeatherAgent`. Validates the `location` argument. | `WeatherAgent` tool invocation | returns accept or reject |
| `WeatherTools` | supporting class | Plain Java class holding the `get_weather` tool method. Returns a stub `WeatherData`. | `WeatherAgent` | returns `WeatherData` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WeatherQuery(
    String queryId,
    String queryText,
    String requestedBackend,
    String submittedBy,
    Instant submittedAt
) {}

record WeatherData(
    String location,
    Condition condition,
    double temperatureCelsius,
    int humidityPercent,
    double windSpeedKmh
) {}
enum Condition { SUNNY, CLOUDY, RAINY, SNOWY, UNKNOWN }

record WeatherReport(
    String location,
    WeatherData data,
    String narrative,
    Instant reportedAt
) {}

record Query(
    String queryId,
    Optional<WeatherQuery> query,
    Optional<WeatherReport> report,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, INVOKING, RESULT_RECORDED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `InvocationStarted`, `ReportRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ queryText, requestedBackend, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agent From Any LLM</title>`.

The App UI tab is a two-column layout: a left column with the query submission panel and the live list of submitted queries (status pill + condition badge + age), and a right pane with the selected query's detail — query text, backend used, current status, and the full weather report (condition badge, temperature, humidity, wind speed, narrative summary).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `get_weather` tool invocation issued by `WeatherAgent`. Validates that the `location` argument is present, non-blank, has at most 200 characters, and does not match a blocklist of known injection-like patterns (e.g., SQL fragments, script tags, path traversal strings). On failure, returns a structured rejection to the agent loop with an error code of `invalid-tool-input`; the agent is expected to respond with a clarification request to the user rather than retrying with the same bad input.

## 9. Agent prompts

- `WeatherAgent` → `prompts/weather-agent.md`. The single decision-making LLM. System prompt instructs it to extract a location from the user's query, call `get_weather(location)` exactly once, and wrap the returned `WeatherData` in a `WeatherReport` with a one-sentence narrative.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "What is the weather in Tokyo?" → agent calls `get_weather("Tokyo")` → `WeatherReport` appears in the UI within 30 s with a valid condition badge.
2. **J2** — User submits a query with a blank location string (e.g., "What is the weather in ?") → the `before-tool-call` guardrail rejects the `get_weather` invocation → the agent returns a clarification request → the card shows status `RESULT_RECORDED` with a report noting that the location was ambiguous.
3. **J3** — Two identical queries are submitted against different `requestedBackend` values (e.g., `anthropic` and `openai`) → both produce well-formed `WeatherReport` results with no code changes.
4. **J4** — The SSE stream delivers status transitions in real time; no page refresh is required between `SUBMITTED` and `RESULT_RECORDED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named any-llm-tool-agent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-any-llm-tool-agent. Java package io.akka.samples.agentfromanyllm.
Akka 3.6.0. HTTP port 9599.

Components to wire (exactly):

- 1 AutonomousAgent WeatherAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-agent.md>),
  .tools(WeatherTools.class), and
  .capability(TaskAcceptance.of(QueryTasks.WEATHER_REPORT).maxIterationsPerTask(3)).
  A before-tool-call guardrail WeatherToolGuardrail is registered on the agent via the
  agent's guardrail-configuration block. Output: WeatherReport{location: String,
  data: WeatherData, narrative: String, reportedAt: Instant}.

- 1 Workflow QueryWorkflow per queryId with two steps:
  * invokeStep — emits InvocationStarted on QueryEntity, then calls
    componentClient.forAutonomousAgent(WeatherAgent.class, "weather-" + queryId)
    .runSingleTask(TaskDef.instructions(query.queryText)) and collects the result via
    forTask(taskId).result(QueryTasks.WEATHER_REPORT). On success calls
    QueryEntity.recordReport(report). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
  * recordStep — emits ReportRecorded on QueryEntity. WorkflowSettings.stepTimeout 10s.
    error step emits QueryFailed and transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  query: Optional<WeatherQuery>, report: Optional<WeatherReport>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED,
  INVOKING, RESULT_RECORDED, FAILED. Events: QuerySubmitted{query}, InvocationStarted{},
  ReportRecorded{report}, QueryFailed{reason}. Commands: submit, markInvoking,
  recordReport, fail, getQuery. emptyState() returns Query.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes
  QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {queryText, requestedBackend,
    submittedBy}; mints queryId; calls QueryEntity.submit; starts QueryWorkflow; returns
    {queryId}), GET /queries (list from getAllQueries, sorted newest-first), GET
    /queries/{id} (one row), GET /queries/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

- WeatherTools.java — a plain Java class (NOT an Akka component) annotated so the agent
  framework discovers its @Tool-annotated methods. One method:
    @Tool("get_weather") WeatherData get_weather(String location)
  Returns a deterministic stub: Condition rotates SUNNY/CLOUDY/RAINY by
  Math.abs(location.hashCode()) % 3, temperature = 15 + (location.length() % 20),
  humidity = 60, windSpeedKmh = 10. Javadoc explains this is a stub for illustration.

- WeatherToolGuardrail.java implementing the before-tool-call hook. Receives the tool
  name and the `location` argument string. Checks: (1) location is non-null and non-blank,
  (2) length <= 200, (3) does not match a SQL/script/path-traversal blocklist (simple
  regex). On any failure returns Guardrail.reject("invalid-tool-input: " + reason) so
  the agent loop surfaces a clarification request. Passing calls flow through to
  WeatherTools.get_weather.

- QueryTasks.java declaring one Task<R> constant:
  WEATHER_REPORT = Task.name("Weather report")
    .description("Look up the weather for the location in the user query and return a WeatherReport")
    .resultConformsTo(WeatherReport.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records WeatherQuery, WeatherData, Condition, WeatherReport, Query, QueryStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9599 and
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o), and
  googleai-gemini (gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also include
  a mock model-provider block (model-provider = mock) as the default when no key is
  detected. WeatherAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs; the requestedBackend field from the
  WeatherQuery is forwarded to the workflow and used to select the provider block
  dynamically (if the deployer has configured it), falling back to the default.

- src/main/resources/sample-events/seed-queries.jsonl with 3 seeded queries:
  {"queryText":"What is the weather in Tokyo right now?","requestedBackend":"mock","submittedBy":"demo-user"}
  {"queryText":"Give me the current weather for London, UK.","requestedBackend":"mock","submittedBy":"demo-user"}
  {"queryText":"Is it raining in São Paulo today?","requestedBackend":"mock","submittedBy":"demo-user"}

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with purpose.primary_function = weather-query,
  decisions.authority_level = informational (the report is advisory, not enforced),
  oversight.human_in_loop = false (automated informational response), failure.failure_modes
  including "hallucinated-location", "tool-call-with-invalid-input", "backend-unavailable",
  "stale-weather-data"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/weather-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Agent From Any LLM", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = query submission panel + live list of query cards; right = selected-query
  detail with query text, backend badge, status pill, condition badge, temperature,
  humidity, wind speed, and narrative). Browser title exactly:
  <title>Akka Sample: Agent From Any LLM</title>. No subtitle on the Overview tab.

Mock LLM provider — required when option (a) is selected (or when no key is detected):

- Generate MockModelProvider.java implementing the ModelProvider interface. One branch
  for WEATHER_REPORT: reads src/main/resources/mock-responses/weather-report.json,
  picks one entry pseudo-randomly per call via seedFor(queryId).
- Per-task mock-response shapes:
    weather-report.json — 6 WeatherReport entries covering all Condition values.
      Each entry has a non-null location, a valid WeatherData, and a one-sentence
      narrative. Plus 1 entry where location is an empty string (triggers guardrail
      rejection path — the guardrail blocks before the tool call, so the agent returns
      a clarification report instead). seedFor selects the guardrail-path entry on
      every 4th query.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; MockModelProvider returns deterministic outputs.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value in Claude session memory only.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (invokeStep 60s,
  recordStep 10s, error 10s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: QueryTasks.java with WEATHER_REPORT = Task.name(...).description(...)
  .resultConformsTo(WeatherReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9599 declared explicitly in application.conf's
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
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WeatherAgent).
  WeatherTools is a tool class, not an agent. WeatherToolGuardrail is a guardrail hook,
  not an agent.
- The tool call is governed by the before-tool-call guardrail hook, NOT by a
  post-response check. The guardrail runs before get_weather is called; the tool body
  is never executed with invalid input.
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
