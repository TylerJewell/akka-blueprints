# SPEC — weather-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WeatherAgent.
**One-line pitch:** A user asks a natural-language weather question; one AI agent resolves the location via a geocoding tool, fetches conditions or forecasts via a weather tool, and returns a typed `WeatherAnswer` — the canonical multi-tool ReAct loop.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `WeatherAgent` (AutonomousAgent) owns the full decision: which tools to call, in what order, how to combine results. One governance mechanism is wired around it:

- A **before-tool-call guardrail** runs on every tool invocation the agent attempts. It validates that the location string is non-empty and within a maximum character limit, that the requested date range (if any) does not exceed 16 days into the future, and that the unit system is one of the allowed values (`metric`, `imperial`, `standard`). A bad parameter set is rejected with a structured error so the agent can correct and retry within its iteration budget.

The blueprint shows that even a stateless question-answering agent benefits from parameter validation before any external call — malformed coordinates or out-of-range date requests would otherwise silently produce nonsense results from downstream APIs.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a weather question into the **Question** textarea (or picks one of four seeded examples: current conditions, 5-day forecast, comparison, UV-index query).
2. The user optionally selects a **unit system** (Metric / Imperial / Standard) and clicks **Ask**.
3. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `PROCESSING` as the workflow starts.
5. Within ~5–30 s the agent completes its tool-call loop. Each tool call is recorded as a `ToolCallRecorded` event visible in the card detail. The card transitions to `ANSWERED`. The answer appears: a short natural-language response paragraph plus a structured `WeatherAnswer` block (location, conditions, temperature, wind, humidity, forecast days if applicable).
6. If the agent cannot resolve the location or the weather service returns an error, the card transitions to `FAILED` with a plain-language reason.
7. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → processing → answered / failed. Source of truth. | `QueryEndpoint`, `QueryWorkflow` | `QueryView` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `processStep` → (terminal). | started by `QueryEndpoint` after entity creation | `WeatherAgent`, `QueryEntity` |
| `WeatherAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question as task instructions; calls geocoding and weather tools; returns `WeatherAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `ToolCallGuardrail` | (supporting class) | Validates every tool-call parameter set before the agent issues the call. Registered on `WeatherAgent` via before-tool-call hook. | `WeatherAgent` | pass-through or rejection |
| `WeatherToolProvider` | (supporting class) | Implements the two tool definitions: `geocode(location)` and `getWeather(lat, lon, units, forecastDays)`. Backed by a stub in dev mode; configurable to real endpoints. | `WeatherAgent` | external APIs (stubbed) |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WeatherQuestion(
    String questionId,
    String questionText,
    UnitSystem units,
    String submittedBy,
    Instant submittedAt
) {}
enum UnitSystem { METRIC, IMPERIAL, STANDARD }

record GeocodingResult(
    String displayName,
    double latitude,
    double longitude
) {}

record CurrentConditions(
    double temperatureDegrees,
    double feelsLikeDegrees,
    double windSpeedKmh,
    int humidityPercent,
    String description,
    String iconCode
) {}

record ForecastDay(
    String date,
    double highDegrees,
    double lowDegrees,
    double precipitationMm,
    String description
) {}

record WeatherAnswer(
    String questionId,
    String locationDisplay,
    double latitude,
    double longitude,
    UnitSystem units,
    Optional<CurrentConditions> current,
    Optional<List<ForecastDay>> forecast,
    String narrative,
    Instant answeredAt
) {}

record ToolCallRecord(
    String toolName,
    Map<String, String> parameters,
    String outcome,    // "accepted" | "rejected" | "success" | "error"
    Instant calledAt
) {}

record Query(
    String questionId,
    Optional<WeatherQuestion> question,
    Optional<GeocodingResult> geocoding,
    Optional<WeatherAnswer> answer,
    List<ToolCallRecord> toolCalls,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    Optional<String> failureReason
) {}

enum QueryStatus {
    SUBMITTED, PROCESSING, ANSWERED, FAILED
}
```

Events on `QueryEntity`: `QuestionSubmitted`, `ProcessingStarted`, `ToolCallRecorded`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ questionText, units, submittedBy }` → `{ questionId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: WeatherAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + age + question excerpt) and a right pane with the selected query's detail — question text, unit system, tool-call timeline, weather answer block with current conditions or forecast, and failure reason if applicable.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call `WeatherAgent` attempts. Validates that (1) `location` (for the geocoding tool) is non-empty and ≤ 200 characters, (2) `forecastDays` (for the weather tool) is between 1 and 16, (3) `units` is one of `metric`, `imperial`, `standard`, and (4) latitude/longitude (when passed directly) are within valid ranges (lat ∈ [-90, 90], lon ∈ [-180, 180]). On any failure, returns a structured `ToolCallRejection` to the agent loop naming the failed check, so the agent can correct its parameters and retry within its 4-iteration budget.

## 9. Agent prompts

- `WeatherAgent` → `prompts/weather-agent.md`. The single decision-making LLM. System prompt instructs it to resolve the location via the geocoding tool first, then call the weather tool with the returned coordinates, and return a well-formed `WeatherAnswer`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks "What is the weather in Paris right now?"; within 30 s a `WeatherAnswer` appears with current conditions and a narrative paragraph; two tool-call records are visible (geocode + getWeather).
2. **J2** — The agent first attempts the geocoding tool with an empty location string; the `before-tool-call` guardrail rejects it; the agent corrects the parameter on the next iteration and produces a valid answer.
3. **J3** — User asks for a 20-day forecast; the guardrail rejects `forecastDays=20`; the agent falls back to 16 days (maximum) and returns a complete answer.
4. **J4** — User asks about a non-existent location "Zyxwvut City"; geocoding returns no results; the entity transitions to `FAILED` with reason `"Geocoding returned no results for: Zyxwvut City"`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named weather-agent demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-weather-tool-agent. Java package io.akka.samples.weatheragent.
Akka 3.6.0. HTTP port 9223.

Components to wire (exactly):

- 1 AutonomousAgent WeatherAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_WEATHER_QUESTION).maxIterationsPerTask(4)).
  The agent has access to two tools registered via .tools(WeatherToolProvider.TOOLS):
    geocode(location: String) — resolves a place name to GeocodingResult{displayName,
      latitude, longitude}. Stub returns deterministic results for the seeded cities.
    getWeather(lat: double, lon: double, units: String, forecastDays: int) — returns
      CurrentConditions + optional List<ForecastDay>. Stub generates plausible data
      from a seeded table keyed by (lat, lon) rounded to 1 decimal place.
  Output: WeatherAnswer{questionId, locationDisplay, latitude, longitude, units,
    current: Optional<CurrentConditions>, forecast: Optional<List<ForecastDay>>,
    narrative, answeredAt}.
  The agent is configured with a before-tool-call guardrail registered via the agent's
  guardrail-configuration block as ToolCallGuardrail. On guardrail rejection, the
  structured rejection is returned to the agent loop as a tool-call error; the loop
  retries within its 4-iteration budget.

- 1 Workflow QueryWorkflow per questionId with two steps:
  * processStep — emits ProcessingStarted on QueryEntity, then calls
    componentClient.forAutonomousAgent(WeatherAgent.class, "weather-" + questionId)
    .runSingleTask(TaskDef.instructions(question.questionText + " Units: " + question.units))
    — returns a taskId, then forTask(taskId).result(ANSWER_WEATHER_QUESTION) to fetch
    the answer. On success calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * error step — calls QueryEntity.fail(reason) and transitions to thenEnd().
    WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per questionId). State Query{questionId,
  question: Optional<WeatherQuestion>, geocoding: Optional<GeocodingResult>,
  answer: Optional<WeatherAnswer>, toolCalls: List<ToolCallRecord>,
  status: QueryStatus, createdAt: Instant, finishedAt: Optional<Instant>,
  failureReason: Optional<String>}. QueryStatus enum: SUBMITTED, PROCESSING,
  ANSWERED, FAILED. Events: QuestionSubmitted{question}, ProcessingStarted{},
  ToolCallRecorded{toolCall}, AnswerRecorded{answer}, QueryFailed{reason}.
  Commands: submit, startProcessing, recordToolCall, recordAnswer, fail, getQuery.
  emptyState() returns Query.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes
  QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {questionText, units, submittedBy};
    mints questionId; calls QueryEntity.submit; starts QueryWorkflow; returns
    {questionId}), GET /queries (list from getAllQueries, sorted newest-first), GET
    /queries/{id} (one row), GET /queries/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- WeatherTasks.java declaring one Task<R> constant: ANSWER_WEATHER_QUESTION =
  Task.name("Answer weather question")
  .description("Resolve the location and fetch current conditions or a forecast, then return a WeatherAnswer")
  .resultConformsTo(WeatherAnswer.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records WeatherQuestion, UnitSystem, GeocodingResult, CurrentConditions,
  ForecastDay, WeatherAnswer, ToolCallRecord, Query, QueryStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. On each proposed
  tool call it reads the tool name and parameter map and runs the four checks listed
  in eval-matrix.yaml G1: (1) location non-empty and ≤ 200 chars, (2) forecastDays
  in [1, 16], (3) units in {metric, imperial, standard}, (4) lat ∈ [-90, 90] and
  lon ∈ [-180, 180] when present. Returns Guardrail.reject(<ToolCallRejection>) on
  any failure; passes through otherwise.

- WeatherToolProvider.java — defines the two tool stubs. Each tool method is annotated
  with @Tool and a description the agent reads. The stub data lives in
  src/main/resources/mock-responses/geocoding-cities.json (20 well-known cities) and
  weather-conditions.json (current + 7-day forecast for each city). Unknown locations
  return an empty result from geocode(); the workflow's error step handles the empty case.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9223 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  WeatherAgent.definition() binds the configured provider via .modelProvider() or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-questions.jsonl with 4 seeded questions:
  "What is the weather in Tokyo right now?",
  "Give me a 5-day forecast for London.",
  "How does the weather in New York compare to Los Angeles today?",
  "What is the UV index in Sydney this afternoon?"

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = informational
  (the agent's answer is for user information only, not a binding decision),
  oversight.human_in_loop = false (weather queries require no human approval loop),
  failure.failure_modes including "invalid-location", "tool-call-parameter-error",
  "stale-stub-data", "unit-system-mismatch"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/weather-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: WeatherAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of query cards; right = selected-query detail with question text,
  unit system, tool-call timeline, weather answer block, failure reason if applicable).
  Browser title exactly: <title>Akka Sample: WeatherAgent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to
        the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured
  key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(questionId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-weather-question.json — 8 WeatherAnswer entries covering a range of cities
      (Tokyo, London, New York, Los Angeles, Sydney, Paris, Berlin, Dubai). Each entry
      has a locationDisplay, current CurrentConditions with plausible temperature/wind/
      humidity, an optional 5-day forecast list, and a 1–2 sentence narrative. Units
      vary (some metric, some imperial). Plus 2 deliberately MALFORMED entries (one with
      forecastDays out of range in the tool call trace, one with units="kelvin") — the
      guardrail blocks both on first attempt, exercising the retry path. The mock should
      select a malformed entry on the FIRST iteration of every 4th query (modulo seed)
      so J2 is reproducible.
- A MockModelProvider.seedFor(questionId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion WeatherTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (processStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: WeatherTasks.java with ANSWER_WEATHER_QUESTION = Task.name(...)
  .description(...).resultConformsTo(WeatherAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9223 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WeatherAgent). Both
  tools (geocode + getWeather) are functions registered on the agent, not separate agents.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the tool returns.
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
