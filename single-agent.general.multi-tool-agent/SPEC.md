# SPEC — multi-tool-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MultiToolAgent.
**One-line pitch:** A user sends a natural-language request; one AI agent selects among weather look-up, currency conversion, and unit transformation tools, calls them in whatever order the request requires, and returns a single combined answer — all external API calls validated before dispatch.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ToolCallingAgent` (AutonomousAgent) carries the entire decision about which tools to call and how to combine their outputs; the surrounding components only prepare its context, enforce call safety, and record the history. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** runs before every individual tool invocation: it validates that inputs are in range and well-typed (e.g., currency codes are ISO-4217, temperature values are numeric and within physical bounds, unit labels are in the registered vocabulary). An invalid tool call is rejected; the agent loop receives a structured error and reformulates the call within its iteration budget.

The blueprint shows that a multi-tool agent calling several external endpoints needs safety at the call boundary, not just at the response boundary. The guardrail fires once per tool invocation, not once per agent turn.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a natural-language request into the **Request** textarea (e.g., "What is 72°F in Celsius, and how many euros is $85?") or clicks one of three seeded examples to auto-fill.
2. The user clicks **Submit**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, the workflow's validate step confirms the request is non-empty and transitions to `DISPATCHING`.
4. The agent begins tool calls. Each tool call triggers the `before-tool-call` guardrail. On success, the tool result is recorded as a `ToolCallRecord`. The card shows a running log of tool calls as they land.
5. Within ~30 s, the agent finishes its final synthesis. The card transitions to `ANSWERED`. The right pane shows the combined answer paragraph plus the full tool-call log (tool name, input, result, timestamp).
6. The user can submit another request; the live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: submitted → dispatching → answered / failed. Source of truth. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `validateStep` → `dispatchStep` → `summarizeStep`. | started by `SessionEndpoint` after submit | `ToolCallingAgent`, `SessionEntity` |
| `ToolCallingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user request as its task instruction; uses registered tools to gather data; returns `ToolResponse`. | invoked by `SessionWorkflow` | returns combined answer |
| `ToolCallGuardrail` | supporting class | `before-tool-call` hook wired on `ToolCallingAgent`. Validates each tool's inputs before dispatch to the external (simulated) adapter. | bound to `ToolCallingAgent` | pass / reject per call |
| `WeatherTool` | supporting class | `@tool` — looks up current weather conditions for a city name. Simulated in-process. | invoked by `ToolCallingAgent` | returns `WeatherResult` |
| `CurrencyTool` | supporting class | `@tool` — converts an amount between two ISO-4217 currency codes. Simulated in-process. | invoked by `ToolCallingAgent` | returns `CurrencyResult` |
| `UnitTool` | supporting class | `@tool` — converts a numeric value between two unit labels (temperature, length, weight). Simulated in-process. | invoked by `ToolCallingAgent` | returns `UnitResult` |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolCallRecord(
    String callId,
    String toolName,
    Map<String, String> inputArgs,
    String result,
    Optional<String> guardRejectionReason,
    Instant calledAt
) {}

record SessionRequest(
    String sessionId,
    String requestText,
    String submittedBy,
    Instant submittedAt
) {}

record ToolResponse(
    String answer,
    List<ToolCallRecord> toolCalls,
    Instant answeredAt
) {}

record Session(
    String sessionId,
    Optional<SessionRequest> request,
    List<ToolCallRecord> toolCalls,
    Optional<ToolResponse> response,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    SUBMITTED, DISPATCHING, ANSWERED, FAILED
}
```

Events on `SessionEntity`: `SessionSubmitted`, `DispatchStarted`, `ToolCallRecorded`, `SessionAnswered`, `SessionFailed`.

Every nullable lifecycle field on the `Session` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ requestText, submittedBy }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MultiToolAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted sessions (status pill + age + request preview) and a right pane with the selected session's detail — request text, running tool-call log (tool name, input args, result, timestamp), final answer paragraph.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every individual tool invocation made by `ToolCallingAgent`. For `WeatherTool`, asserts the city name is non-empty and under 100 characters. For `CurrencyTool`, asserts the source and target currency codes are valid ISO-4217 codes and the amount is a positive finite number. For `UnitTool`, asserts the from- and to-unit labels are in the registered vocabulary and the numeric value is finite. On rejection, returns a structured `invalid-tool-call` error to the agent loop; the agent uses one of its 5 iterations to reformulate. Passing calls are dispatched to the tool adapter.

## 9. Agent prompts

- `ToolCallingAgent` → `prompts/tool-calling-agent.md`. The single decision-making LLM. System prompt instructs it to read the user request, decide which tools are needed, call them in whatever order yields the answer, and return one `ToolResponse` with the combined answer and the list of `ToolCallRecord` entries for every call made.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded multi-part request ("What is 72°F in Celsius, and how many euros is $85?"); within 30 s the answer appears with two `ToolCallRecord` entries (UnitTool + CurrencyTool) and a correct combined answer.
2. **J2** — A request containing an invalid currency code (`XYZ`) is submitted; the guardrail rejects the CurrencyTool call; the agent reformulates with a valid code and produces a valid answer; the UI shows one `ToolCallRecord` with a `guardRejectionReason` and one successful one.
3. **J3** — A deliberately unanswerable request exhausts all 5 iterations; the session lands in `FAILED` with the partial tool-call history visible.
4. **J4** — Every tool-call log entry shows non-null `toolName`, `inputArgs`, `result`, and `calledAt`; the UI displays them in the right pane.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-tool-agent demonstrating the single-agent × general cell. Runs
out of the box (no external services — all tool adapters are simulated in-process). Maven
group io.akka.samples. Maven artifact single-agent-general-multi-tool-agent. Java package
io.akka.samples.multipletoolsagent. Akka 3.6.0. HTTP port 9172.

Components to wire (exactly):

- 1 AutonomousAgent ToolCallingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/tool-calling-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_REQUEST).maxIterationsPerTask(5)). The task receives
  the user request as its instruction text. The agent calls WeatherTool, CurrencyTool, and
  UnitTool via the registered @tool mechanism. Output: ToolResponse{answer: String,
  toolCalls: List<ToolCallRecord>, answeredAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection, the agent loop retries the tool
  call within its 5-iteration budget.

- 3 @tool adapters — WeatherTool, CurrencyTool, UnitTool — each a class with a single
  @tool-annotated method. All are simulated in-process (no real HTTP calls):
    * WeatherTool.lookup(cityName: String) → WeatherResult{city, condition, tempCelsius,
      humidity}. Seeded map of 10 city names to fixed weather snapshots.
    * CurrencyTool.convert(amount: double, fromCode: String, toCode: String) →
      CurrencyResult{fromCode, toCode, amount, converted, rate}. Seeded exchange-rate
      table covering USD, EUR, GBP, JPY, CAD, AUD, CHF, CNY.
    * UnitTool.convert(value: double, fromUnit: String, toUnit: String) →
      UnitResult{fromUnit, toUnit, inputValue, outputValue}. Supports temperature
      (celsius/fahrenheit/kelvin), length (m/km/ft/mi), weight (kg/lb/oz/g).

- 1 Workflow SessionWorkflow per sessionId with three steps:
  * validateStep — verifies request.requestText is non-empty and under 2000 characters;
    on success calls SessionEntity.startDispatch; WorkflowSettings.stepTimeout 5s.
  * dispatchStep — calls componentClient.forAutonomousAgent(ToolCallingAgent.class,
    "agent-" + sessionId).runSingleTask(TaskDef.instructions(request.requestText))
    — returns a taskId, then forTask(taskId).result(ANSWER_REQUEST) to fetch ToolResponse.
    On success calls SessionEntity.recordAnswer(response). WorkflowSettings.stepTimeout 120s
    with defaultStepRecovery maxRetries(1).failoverTo(SessionWorkflow::error).
  * summarizeStep — no-op in baseline (the agent already returns a combined answer); reserved
    for deployer extension. WorkflowSettings.stepTimeout 5s.
    error step calls SessionEntity.fail(reason) and transitions to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  request: Optional<SessionRequest>, toolCalls: List<ToolCallRecord>,
  response: Optional<ToolResponse>, status: SessionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. SessionStatus enum: SUBMITTED, DISPATCHING, ANSWERED,
  FAILED. Events: SessionSubmitted{request}, DispatchStarted{}, ToolCallRecorded{record},
  SessionAnswered{response}, SessionFailed{reason}. Commands: submit, startDispatch,
  recordToolCall, recordAnswer, fail, getSession. emptyState() returns
  Session.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.
  The toolCalls field on the entity state uses List.of() (empty immutable list) initially
  and is appended via List.copyOf in the ToolCallRecorded applier.

- 1 View SessionView with row type SessionRow (mirrors Session). Table updater consumes
  SessionEntity events. ONE query getAllSessions: SELECT * AS sessions FROM session_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {requestText, submittedBy}; mints
    sessionId; calls SessionEntity.submit; starts SessionWorkflow; returns {sessionId}),
    GET /sessions (list from getAllSessions, sorted newest-first), GET /sessions/{id}
    (one row), GET /sessions/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SessionTasks.java declaring one Task<R> constant: ANSWER_REQUEST = Task.name("Answer
  request").description("Use available tools to answer the user's question and return a
  combined ToolResponse").resultConformsTo(ToolResponse.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records SessionRequest, ToolCallRecord, WeatherResult, CurrencyResult, UnitResult,
  ToolResponse, Session, SessionStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the tool name and
  input arguments from the pending tool invocation, runs the per-tool validation rules
  described in eval-matrix.yaml G1, and either passes the call through or returns
  Guardrail.reject(<structured-error>) to force the agent loop to reformulate.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9172 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ToolCallingAgent.definition()
  binds the configured provider via the per-agent override pattern from the akka-context
  docs.

- src/main/resources/sample-events/seed-requests.jsonl with 5 seeded natural-language
  requests:
    1. "What is 72°F in Celsius, and how many euros is $85?"
    2. "Convert 10 miles to kilometers and tell me the weather in Tokyo."
    3. "What is the weather in London? Also convert 500 GBP to USD."
    4. "How many kilograms is 150 pounds, and what is the current weather in Paris?"
    5. "Convert 1000 JPY to AUD and 100 kg to pounds."

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root pre-filled for the general domain; deployer-specific
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/tool-calling-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MultiToolAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session detail with request text, tool-call
  log table, and final answer). Browser title exactly:
  <title>Akka Sample: MultiToolAgent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-request.json — 8 ToolResponse entries covering all three tool combinations.
      Each entry has an answer paragraph, and a toolCalls array listing every tool
      invoked. Each ToolCallRecord has a non-null toolName, inputArgs map, result string,
      and calledAt. Include 2 entries whose ToolCallRecord list contains a record with a
      non-null guardRejectionReason (simulating a guardrail reject followed by a
      successful retry on the next call in the same response). The mock should select a
      rejection-containing entry on the FIRST iteration of every 3rd session
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ToolCallingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SessionTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (validateStep
  5s, dispatchStep 120s, summarizeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Session row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: SessionTasks.java with ANSWER_REQUEST = Task.name(...).description(...)
  .resultConformsTo(ToolResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9172 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ToolCallingAgent).
  The three tool adapters (WeatherTool, CurrencyTool, UnitTool) are @tool-annotated helper
  classes, NOT AutonomousAgents.
- The before-tool-call guardrail fires per tool invocation, not per agent turn. Verify the
  generated ToolCallGuardrail is wired via the agent's guardrail-configuration block with
  the before-tool-call hook, not as a post-hoc validator.
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
