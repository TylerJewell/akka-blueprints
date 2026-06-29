# SPEC — structured-mcp-tool

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** StructuredMcpTool.
**One-line pitch:** A user submits a location; one AI agent calls an inline MCP FastMCP server's `get_weather_info` tool, receives a Pydantic-typed weather payload, and returns a structured `WeatherSummary` — current conditions, temperature, humidity, wind, and a one-sentence narrative.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `WeatherQueryAgent` (AutonomousAgent) makes every decision; the surrounding components only queue the request and record the result. One governance mechanism is wired around the agent:

- An **after-tool-call guardrail** runs each time `WeatherQueryAgent` receives a response from the `get_weather_info` MCP tool — before the agent processes the payload into its final answer. The guardrail validates that all required fields are present, that temperature is in a plausible physical range (−90 °C to 60 °C), that humidity is 0–100 %, and that the `condition` enum value is in the declared set. A schema-invalid payload triggers a rejection that forces the agent to retry the tool call within its iteration budget.

The blueprint shows that a tool-calling agent is not implicitly safe because the tool is local — the `after-tool-call` hook is the correct place to catch bad payloads before the agent reasons over them.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a location name into the **Location** text input (or picks one of three seeded examples — London, Tokyo, Sydney).
2. The user optionally sets a **Unit** toggle (Celsius / Fahrenheit).
3. The user clicks **Get weather**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `RUNNING` as the workflow starts the agent.
5. Within ~10–30 s, the agent returns. The card transitions to `COMPLETED`. The result pane shows: a conditions badge, temperature and humidity chips, a wind speed chip, and the one-sentence narrative paragraph.
6. If the MCP tool returns a schema-invalid payload on the first tool call, the guardrail rejects it, the agent retries, and the card eventually reaches `COMPLETED` with a valid result. The invalid intermediate payload never appears in the result pane.
7. The user can submit another location; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → running → completed / failed. Source of truth. | `QueryEndpoint`, `WeatherQueryWorkflow` | `QueryView` |
| `WeatherQueryWorkflow` | `Workflow` | One workflow per query. Steps: `runStep` → result recording. | started by `QueryEndpoint` after entity submit | `WeatherQueryAgent`, `QueryEntity` |
| `WeatherQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the location and unit preference in the task definition; calls `get_weather_info` on the inline MCP server; returns `WeatherSummary`. | invoked by `WeatherQueryWorkflow` | returns summary |
| `McpToolResultGuardrail` | supporting class | `after-tool-call` hook on `WeatherQueryAgent`. Validates `WeatherPayload` fields before the agent processes them. | wired into `WeatherQueryAgent` definition | pass-through or reject |
| `InlineMcpServer` | supporting class | A FastMCP-style server registered in-process. Exposes `get_weather_info(location: str, unit: str) -> WeatherPayload`. Returns seeded or mock weather data. | called by `WeatherQueryAgent` tool call | `WeatherQueryAgent` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WeatherPayload(
    String location,
    double temperatureCelsius,
    int humidityPercent,
    double windSpeedKph,
    WeatherCondition condition,
    String observedAt     // ISO-8601 string from the tool
) {}
enum WeatherCondition { CLEAR, PARTLY_CLOUDY, OVERCAST, RAIN, THUNDERSTORM, SNOW, FOG }

record WeatherSummary(
    String location,
    double temperatureCelsius,
    Optional<Double> temperatureFahrenheit,
    int humidityPercent,
    double windSpeedKph,
    WeatherCondition condition,
    String narrative,     // 1-sentence agent-generated description
    Instant returnedAt
) {}

record QueryRequest(
    String queryId,
    String location,
    UnitPreference unit,
    String submittedBy,
    Instant submittedAt
) {}
enum UnitPreference { CELSIUS, FAHRENHEIT }

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<WeatherSummary> summary,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, RUNNING, COMPLETED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `QueryStarted`, `SummaryRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ location, unit, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Structured Output Tool (MCP)</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + conditions badge + age) and a right pane with the selected query's detail — submitted location, unit preference, weather conditions tiles, and the narrative paragraph.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — after-tool-call guardrail**: runs each time `WeatherQueryAgent` receives a `get_weather_info` response from the MCP server. Asserts the payload is well-formed: all required fields present (`location`, `temperatureCelsius`, `humidityPercent`, `windSpeedKph`, `condition`, `observedAt`), `temperatureCelsius` is in [−90, 60], `humidityPercent` is in [0, 100], and `condition` is in the declared `WeatherCondition` enum. On failure, returns a structured rejection naming the failed field; the agent loop counts one iteration and retries the tool call. Passing payloads flow through to the agent's reasoning step.

## 9. Agent prompts

- `WeatherQueryAgent` → `prompts/weather-query-agent.md`. The single decision-making LLM. System prompt instructs it to call `get_weather_info`, validate that it received a response (the guardrail runs automatically), and return a `WeatherSummary` with a one-sentence narrative describing current conditions in plain language.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "London"; within 30 s the result pane shows a `WeatherSummary` with all fields populated and the conditions badge visible.
2. **J2** — Mock MCP server returns a payload missing `humidityPercent` on the first call; the `after-tool-call` guardrail rejects it; the second iteration succeeds; the UI shows only the final valid summary.
3. **J3** — Mock MCP server returns a `temperatureCelsius` of 9999 (implausible); the guardrail rejects with `temperature-out-of-range`; the agent retries; the log shows exactly one guardrail rejection line before the successful result.
4. **J4** — The SSE stream carries one event per status transition; a browser tab opened after `COMPLETED` still sees the current state without replay.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named structured-mcp-tool demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-structured-mcp-tool. Java package
io.akka.samples.structuredoutputtoolmcp. Akka 3.6.0. HTTP port 9215.

Components to wire (exactly):

- 1 AutonomousAgent WeatherQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-query-agent.md>) and
  .capability(TaskAcceptance.of(GET_WEATHER_SUMMARY).maxIterationsPerTask(3))
  and .mcpServer(InlineMcpServer.definition()). The task receives the location and
  unit preference in its instruction text. The agent calls the InlineMcpServer's
  get_weather_info tool and returns WeatherSummary{location, temperatureCelsius,
  temperatureFahrenheit: Optional<Double>, humidityPercent, windSpeedKph, condition,
  narrative: String, returnedAt: Instant}. The agent is configured with an
  after-tool-call guardrail (G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the
  tool call within its 3-iteration budget.

- 1 Workflow WeatherQueryWorkflow per queryId with two steps:
  * runStep — emits QueryStarted, then calls componentClient.forAutonomousAgent(
    WeatherQueryAgent.class, "query-agent-" + queryId).runSingleTask(
      TaskDef.instructions(formatQuery(request))
    ) — returns a taskId, then forTask(taskId).result(GET_WEATHER_SUMMARY) to fetch
    the summary. On success calls QueryEntity.recordSummary(summary).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(WeatherQueryWorkflow::error).
  * error step — transitions the entity to FAILED. WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, summary: Optional<WeatherSummary>, status:
  QueryStatus, createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum:
  SUBMITTED, RUNNING, COMPLETED, FAILED. Events: QuerySubmitted{request},
  QueryStarted{}, SummaryRecorded{summary}, QueryFailed{reason}. Commands: submit,
  markRunning, recordSummary, fail, getQuery. emptyState() returns Query.initial("")
  with no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes
  QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {location, unit, submittedBy};
    mints queryId; calls QueryEntity.submit; starts WeatherQueryWorkflow; returns
    {queryId}), GET /queries (list from getAllQueries, sorted newest-first), GET
    /queries/{id} (one row), GET /queries/sse (Server-Sent Events forwarded from
    the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- WeatherQueryTasks.java declaring one Task<R> constant: GET_WEATHER_SUMMARY =
  Task.name("Get weather summary").description("Call get_weather_info for the
  requested location and return a structured WeatherSummary").resultConformsTo(
  WeatherSummary.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- InlineMcpServer.java — a FastMCP-style server registered as an Akka component.
  Exposes get_weather_info(location: String, unit: String) -> WeatherPayload.
  Implementation: for known seeded locations (London, Tokyo, Sydney) returns
  static data; for all other locations returns a deterministic pseudo-random
  WeatherPayload seeded by location.hashCode(). Registered in-process via
  AgentDefinition.mcpServer(InlineMcpServer.definition()).

- McpToolResultGuardrail.java implementing the after-tool-call hook. Reads the
  WeatherPayload from the tool response, runs the four checks listed in
  eval-matrix.yaml G1, and either passes the payload through or returns
  Guardrail.reject(<structured-error>) naming the failed field to force a retry.

- Domain records QueryRequest, WeatherPayload, WeatherSummary, WeatherCondition,
  Query, QueryRequest, UnitPreference, QueryStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9215
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  WeatherQueryAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-queries.jsonl with 3 seeded queries:
  London (CELSIUS), Tokyo (CELSIUS), Sydney (FAHRENHEIT). Each maps to a known
  WeatherPayload in InlineMcpServer's static table.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with sector = general, decisions.authority_level
  = recommend-only (the summary is informational), oversight.human_in_loop = false
  (weather data does not require human sign-off), failure.failure_modes including
  "schema-invalid-tool-response", "temperature-out-of-range", "unknown-condition-enum",
  "missing-required-field"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/weather-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Structured Output Tool (MCP)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of query cards; right = selected-query detail with location,
  unit, conditions tiles, and narrative paragraph).
  Browser title exactly: <title>Akka Sample: Structured Output Tool (MCP)</title>.
  No subtitle on the Overview tab.

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
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
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
  per call (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    get-weather-summary.json — 8 WeatherSummary entries covering all WeatherCondition
    values. Each entry has a non-empty narrative sentence and plausible numeric values.
    Plus 2 deliberately schema-invalid entries (one missing humidityPercent; one with
    temperatureCelsius = 9999) — the guardrail blocks both. The mock should select a
    schema-invalid entry on the first tool-call iteration of every 3rd query (modulo
    seed) so J2 and J3 are reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherQueryAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  WeatherQueryTasks.java MUST exist.
- Lesson 4: the runStep has an explicit stepTimeout of 60s; the error step 5s.
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: WeatherQueryTasks.java with GET_WEATHER_SUMMARY = Task.name(...)
  .description(...).resultConformsTo(WeatherSummary.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9215 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides (state-label
  colour, edge-label foreignObject overflow:visible) AND mermaid.initialize
  themeVariables block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WeatherQueryAgent).
  No second agent is needed for validation — McpToolResultGuardrail is a hook class,
  not an agent.
- The after-tool-call guardrail is wired via the agent's guardrail-configuration block
  at definition time, NOT as a post-hoc wrapper around the workflow step.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
