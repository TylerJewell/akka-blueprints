# SPEC — ambient-durable-agent-pubsub

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AmbientWeatherAgent.
**One-line pitch:** Messages arriving on a pub/sub topic (`weather.requests`) each spawn an independent durable workflow that validates the payload, scrubs PII, calls a single `WeatherAgent`, and records the structured forecast result — no human at a browser required.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain, triggered by pub/sub rather than a direct HTTP call. One `WeatherAgent` (AutonomousAgent) carries the entire decision; the surrounding components validate, sanitize, and durably record each message lifecycle. Two governance mechanisms are wired at the message boundary:

- A **before-agent-invocation guardrail** runs inside `PayloadGuardrail` after the topic message is received but before the agent task is started. It validates that every required field is present, the location string is non-empty, the requested forecast horizon is within bounds, and no unexpected fields arrived that could represent a prompt-injection attempt. A rejected payload transitions the entity to FAILED without ever touching the agent.
- A **PII sanitizer** runs inside a dedicated `sanitizeStep` in the workflow. Open pub/sub topics often carry identifiers — contact emails, device owner names, internal network hostnames, or location strings that could identify individuals. The sanitizer scrubs those tokens from the payload before the `WeatherAgent` receives its task attachment.

The blueprint shows that ambient (asynchronous, message-driven) agent invocation demands governance at the message ingress — there is no user session to catch mistakes before they propagate.

## 3. User-facing flows

The user opens the App UI tab.

1. The user either publishes a raw JSON message to the in-process topic broker (via the **Publish message** panel in the App UI) or picks one of four seeded payloads (tropical-storm-alert, cold-front-advisory, heatwave-forecast, standard-daily-request).
2. The user clicks **Publish**. The UI POSTs to `/api/requests/publish` and the broker delivers the message to `TopicConsumer`.
3. Within ~1 s, the card for that message appears in the live list in `RECEIVED` state, then transitions to `VALIDATED` as `PayloadGuardrail` passes it, then to `SANITIZED` as `RequestSanitizer` runs.
4. The card transitions to `FORECASTING` while the agent works. Within ~10–30 s it transitions to `COMPLETED`. The right pane shows the structured `WeatherReport`: conditions summary, temperature range, wind, precipitation probability, and a hazard flag.
5. If the payload fails validation, the card appears in `FAILED` state with a reason label. No agent call happened.
6. The user can publish additional messages; the live list shows all requests newest-first.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestEndpoint` | `HttpEndpoint` | `/api/requests/*` — publish, list, get, SSE; serves `/api/metadata/*`. | — | `RequestEntity`, `RequestView`, topic broker |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle: received → validated → sanitized → forecasting → completed / failed. Source of truth. | `TopicConsumer`, `WeatherWorkflow` | `RequestView` |
| `TopicConsumer` | `Consumer` | Subscribes to `weather.requests` topic; on each message starts a `WeatherWorkflow`. | topic broker | `WeatherWorkflow`, `RequestEntity` |
| `WeatherWorkflow` | `Workflow` | One workflow per request. Steps: `validateStep` → `sanitizeStep` → `forecastStep` → `recordStep`. Error step transitions entity to FAILED. | started by `TopicConsumer` | `PayloadGuardrail`, `RequestSanitizer`, `WeatherAgent`, `RequestEntity` |
| `WeatherAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized forecast request as a task attachment and returns a `WeatherReport`. | invoked by `WeatherWorkflow.forecastStep` | returns report |
| `PayloadGuardrail` | supporting class | Called from `WeatherWorkflow.validateStep`; implements the before-agent-invocation check. Not itself an Akka component — called by the workflow step. | `WeatherWorkflow` | returns validation result |
| `RequestSanitizer` | supporting class | Called from `WeatherWorkflow.sanitizeStep`; scrubs PII from the raw payload. Returns `SanitizedPayload`. | `WeatherWorkflow` | returns sanitized form |
| `RequestView` | `View` | Read model: one row per request for the UI. | `RequestEntity` events | `RequestEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record WeatherRequestPayload(
    String requestId,
    String location,
    String requesterContact,    // may contain PII — sanitized before agent
    ForecastHorizon horizon,
    List<String> alertCategories,
    Instant receivedAt
) {}
enum ForecastHorizon { HOURLY_6H, DAILY_3D, WEEKLY_7D }

record ValidationResult(
    boolean valid,
    List<String> violationCodes  // empty when valid
) {}

record SanitizedPayload(
    String location,             // location retained but PII substrings removed
    ForecastHorizon horizon,
    List<String> alertCategories,
    List<String> piiCategoriesFound
) {}

record WeatherReport(
    String conditionsSummary,
    TemperatureRange temperatureRange,
    int windSpeedKph,
    int precipitationPct,
    boolean hazardFlag,
    String hazardDetail,        // empty string when hazardFlag is false
    Instant forecastedAt
) {}

record TemperatureRange(int lowCelsius, int highCelsius) {}

record Request(
    String requestId,
    Optional<WeatherRequestPayload> rawPayload,
    Optional<ValidationResult> validation,
    Optional<SanitizedPayload> sanitized,
    Optional<WeatherReport> report,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED, VALIDATED, SANITIZED, FORECASTING, COMPLETED, FAILED
}
```

Events on `RequestEntity`: `MessageReceived`, `PayloadValidated`, `PayloadSanitized`, `ForecastingStarted`, `ReportRecorded`, `RequestFailed`.

Every nullable lifecycle field on the `Request` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/requests/publish` — body `WeatherRequestPayload` (or `{ raw: String }` for raw JSON injection) → `{ requestId }`. Publishes to the in-process broker.
- `GET /api/requests` — list all requests, newest-first.
- `GET /api/requests/{id}` — one request.
- `GET /api/requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Ambient Durable Agent on Pub/Sub</title>`.

The App UI tab is a two-column layout: a left rail with the Publish panel and the live list of requests (status pill + hazard badge + age), and a right pane with the selected request's detail — raw payload summary, validation result, sanitized payload, and the weather report with temperature range, wind, precipitation, and hazard flag.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail**: runs inside `WeatherWorkflow.validateStep` via `PayloadGuardrail`. Asserts that `location` is non-empty and at most 256 characters, `horizon` is a valid `ForecastHorizon` enum value, `alertCategories` is a non-null list (empty is fine), and `requesterContact` does not contain SQL/script injection patterns. On any failure, returns a structured `ValidationResult` with violation codes; the workflow transitions the entity to FAILED without starting the agent. On pass, transitions entity to VALIDATED and advances to `sanitizeStep`.
- **S1 — PII sanitizer**: runs inside `WeatherWorkflow.sanitizeStep` via `RequestSanitizer`. Scrubs emails, phone numbers, person names, internal network hostnames (RFC-1918 subnet patterns, `.internal` / `.local` / `.corp` suffixes), and account-id-like tokens from `location`, `requesterContact`, and any free-text `alertCategories` entries. Records which categories were found. Produces `SanitizedPayload` that `forecastStep` passes as the `WeatherAgent` task attachment; the raw payload remains on the entity for audit only.

## 9. Agent prompts

- `WeatherAgent` → `prompts/weather-agent.md`. The single decision-making LLM. System prompt instructs it to read the sanitized forecast request (passed as a task attachment), reason about the location, horizon, and alert categories, and return one structured `WeatherReport`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — A valid standard-daily-request message is published; within 30 s the UI shows a `COMPLETED` card with a full `WeatherReport` and all lifecycle pills populated.
2. **J2** — A malformed payload (missing `location`) is published; the card appears in `FAILED` state with violation code `location.required`; no agent task ran.
3. **J3** — A payload whose `requesterContact` contains `user@example.com` is published; the agent's task attachment shows `[REDACTED-EMAIL]`; the entity audit log retains the raw.
4. **J4** — Two valid messages are published within 1 s of each other; two independent workflow executions complete with separate `requestId` values; each card shows its own report.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ambient-durable-agent-pubsub demonstrating the single-agent ×
ops-automation cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact single-agent-ops-automation-akka-event-driven-agent. Java package
io.akka.samples.ambientdurableagentonpubsub. Akka 3.6.0. HTTP port 9390.

Components to wire (exactly):

- 1 AutonomousAgent WeatherAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-agent.md>) and
  .capability(TaskAcceptance.of(FORECAST_WEATHER).maxIterationsPerTask(2)). The task receives
  the forecast context as its instruction text and the sanitized payload as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: WeatherReport{conditionsSummary: String, temperatureRange: TemperatureRange,
  windSpeedKph: int, precipitationPct: int, hazardFlag: boolean, hazardDetail: String,
  forecastedAt: Instant}.

- 1 Workflow WeatherWorkflow per requestId with four steps:
  * validateStep — instantiates PayloadGuardrail and calls validate(rawPayload). On
    ValidationResult.valid == false, calls RequestEntity.fail(violationCodes.toString()) and
    transitions to the error step. On pass calls RequestEntity.markValidated() and advances to
    sanitizeStep. WorkflowSettings.stepTimeout 5s.
  * sanitizeStep — instantiates RequestSanitizer and calls sanitize(rawPayload). Calls
    RequestEntity.attachSanitized(sanitized). Advances to forecastStep.
    WorkflowSettings.stepTimeout 5s.
  * forecastStep — emits ForecastingStarted via RequestEntity.markForecasting(), then calls
    componentClient.forAutonomousAgent(WeatherAgent.class, "weather-" + requestId)
    .runSingleTask(
      TaskDef.instructions(formatRequest(sanitized))
        .attachment("request.json", toJson(sanitized).getBytes())
    ) — returns a taskId, then forTask(taskId).result(FORECAST_WEATHER) to fetch the report.
    On success calls RequestEntity.recordReport(report). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(1).failoverTo(WeatherWorkflow::error).
  * recordStep — calls RequestEntity.complete(). WorkflowSettings.stepTimeout 5s.
    error step calls RequestEntity.fail("workflow-error") and transitions to thenEnd().

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RequestEntity (one per requestId). State Request{requestId: String,
  rawPayload: Optional<WeatherRequestPayload>, validation: Optional<ValidationResult>,
  sanitized: Optional<SanitizedPayload>, report: Optional<WeatherReport>, status: RequestStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. RequestStatus enum: RECEIVED, VALIDATED,
  SANITIZED, FORECASTING, COMPLETED, FAILED. Events: MessageReceived{payload},
  PayloadValidated{validation}, PayloadSanitized{sanitized}, ForecastingStarted{},
  ReportRecorded{report}, RequestFailed{reason}. Commands: receive, markValidated,
  attachSanitized, markForecasting, recordReport, complete, fail, getRequest.
  emptyState() returns Request.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer TopicConsumer subscribed to the in-process topic "weather.requests". On each
  message, calls RequestEntity.receive(payload) to record MessageReceived, then starts a
  WeatherWorkflow with id = "forecast-" + requestId.

- 1 View RequestView with row type RequestRow (mirrors Request minus rawPayload.requesterContact
  — the audit log keeps the raw contact; the view holds only the sanitized form). Table updater
  consumes RequestEntity events. ONE query getAllRequests: SELECT * AS requests FROM request_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * RequestEndpoint at /api with POST /requests/publish (body WeatherRequestPayload; mints
    requestId; publishes to broker; returns {requestId}), GET /requests (list from
    getAllRequests, sorted newest-first), GET /requests/{id} (one row), GET /requests/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Supporting classes:

- ForecastTasks.java declaring one Task<R> constant: FORECAST_WEATHER = Task.name("Forecast
  weather").description("Analyse the sanitized request payload and produce a WeatherReport")
  .resultConformsTo(WeatherReport.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records WeatherRequestPayload, ForecastHorizon, ValidationResult, SanitizedPayload,
  WeatherReport, TemperatureRange, Request, RequestStatus.

- PayloadGuardrail.java — plain Java class (NOT an Akka component). Called from validateStep.
  validate(WeatherRequestPayload) runs four checks: (1) location is non-null and 1–256 chars,
  (2) horizon is a known ForecastHorizon enum value, (3) alertCategories is non-null,
  (4) requesterContact does not match a basic injection pattern (< > ; -- ' " -- characters
  that suggest script or SQL injection). Returns ValidationResult{valid, violationCodes}.

- RequestSanitizer.java — plain Java class. Called from sanitizeStep. sanitize(WeatherRequestPayload)
  runs regex redaction over location and requesterContact (emails, phone numbers, person names
  via heuristic, internal-network hostname patterns). Produces SanitizedPayload{location
  (redacted), horizon, alertCategories, piiCategoriesFound}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9390 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. The WeatherAgent.definition() binds the configured provider
  via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/weather-requests.jsonl with 4 seeded payloads: a
  tropical-storm-alert (Miami Beach, 6h horizon), a cold-front-advisory (Minneapolis, 3d
  horizon), a heatwave-forecast (Phoenix AZ, 7d horizon), and a standard-daily-request
  (Denver CO, 3d horizon). The tropical-storm and cold-front entries include a
  requesterContact email address so S1 has work to do. One additional entry is deliberately
  malformed (missing location field) for J2.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only,
  oversight.human_in_loop = false (ambient mode — no human in the per-request loop; results
  are observable but not approved per-message), failure.failure_modes including
  "payload-injection-via-topic", "hallucinated-forecast", "pii-leakage-via-llm",
  "missed-hazard-flag"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/weather-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Ambient Durable Agent on Pub/Sub",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  Publish panel + live list of request cards; right = selected-request detail with raw payload
  summary, validation result, sanitized payload, and weather report).
  Browser title exactly: <title>Akka Sample: Ambient Durable Agent on Pub/Sub</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct WeatherReport outputs per agent task. Sets model-provider = mock.
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
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. For FORECAST_WEATHER: reads
  src/main/resources/mock-responses/forecast-weather.json, picks one entry pseudo-randomly
  per call (seedFor(requestId)), and deserialises into WeatherReport.
- forecast-weather.json — 6 WeatherReport entries covering the four seeded locations, one
  with hazardFlag = true (the tropical-storm-alert), one with hazardFlag = false
  (standard-daily-request), a wide temperature range (heatwave), and a precipitation-heavy
  entry (cold-front). Each has a non-empty conditionsSummary and a valid forecastedAt in
  ISO-8601.
- MockModelProvider.seedFor(requestId) makes per-request selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ForecastTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (validateStep 5s, sanitizeStep 5s,
  forecastStep 60s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Request row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: ForecastTasks.java with FORECAST_WEATHER = Task.name(...).description(...)
  .resultConformsTo(WeatherReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9390 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WeatherAgent). PayloadGuardrail
  and RequestSanitizer are plain Java classes, not agents.
- The forecast request is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated forecastStep uses TaskDef.attachment(...).
- The before-agent-invocation check runs in validateStep (a workflow step) via PayloadGuardrail,
  before the agent task is ever started. If validation fails, the forecastStep is never reached.
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
