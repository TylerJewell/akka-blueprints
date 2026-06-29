# SPEC — dynamic-route-agent

The natural-language brief `/akka:specify` reads to generate this system. Sections 1-11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Dynamic Route Agent.
**One-line pitch:** The user picks a route (SUMMARIZE or TRANSLATE), pastes text, and submits; one generic agent — configured per request with route-specific instructions — returns the summarized or translated result.

## 2. What this blueprint demonstrates

The single-agent coordination pattern with per-request configuration: one `Agent` class carries no hard-coded instructions, and each call builds an `AgentSetup` selecting the route's instruction text and capability. The same code path serves a SUMMARIZE route and a TRANSLATE route. The governance pattern is two guardrails on one agent: a before-agent-invocation guardrail that validates the per-request configuration before the model runs, and a before-agent-response guardrail that checks the model output before it reaches the user.

## 3. User-facing flows

1. The user opens the App UI tab, selects route SUMMARIZE, pastes a paragraph, and submits. A request row appears in `SUBMITTED`, then flips to `COMPLETED` with a shorter output once the agent returns.
2. The user selects route TRANSLATE, enters text plus a target language, and submits. The row completes with translated text.
3. The user submits a request whose configuration is unsafe (empty text, unknown route, or text over the size limit). The before-agent-invocation guardrail stops it; the row lands in `BLOCKED` with a reason, and the agent never runs.
4. Without any interaction, the simulator drips a sample request every 45 seconds; the row appears over SSE and completes on its own.

These become the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| DynamicAgent | Agent | Configured per request via AgentSetup; runs SUMMARIZE or TRANSLATE instructions | RouteEndpoint | (returns AgentResult) |
| RequestEntity | EventSourcedEntity | Holds one request's lifecycle (submitted, completed, blocked) | RouteEndpoint | RequestsView |
| RequestsView | View | Read model the UI lists and streams | RequestEntity | RouteEndpoint (queries) |
| RouteEndpoint | HttpEndpoint | `/api` surface: submit, list, single, SSE, metadata | Browser, RequestSimulator | RequestEntity, DynamicAgent, RequestsView |
| AppEndpoint | HttpEndpoint | Serves the single-file UI | Browser | static-resources |
| RequestSimulator | TimedAction | Drips sample requests for background activity | (scheduled tick) | RouteEndpoint |

Names matter. `/akka:specify` uses them verbatim.

## 5. Data model

See `reference/data-model.md` for the authoritative field list. Summary:

- `Route` enum: `SUMMARIZE`, `TRANSLATE`.
- `RequestStatus` enum: `SUBMITTED`, `COMPLETED`, `BLOCKED`.
- Entity state and view row `RequestRecord` carries `id`, `route`, `inputText`, `status`, `submittedAt`, plus the nullable lifecycle fields `targetLanguage`, `output`, `completedAt`, `blockedReason`, `blockedAt`, each declared `Optional<T>` (Lesson 6).
- Input record `AgentRequest(Route route, String text, Optional<String> targetLanguage)`. Agent output `AgentResult(String output)`.
- Events: `RequestSubmitted`, `RequestCompleted`, `RequestBlocked`.

## 6. API contract

Top-level surface (schemas in `reference/api-contract.md`):

```
POST /api/requests              -> { requestId }
GET  /api/requests              -> { requests: [RequestRecord, ...] }
GET  /api/requests/{id}         -> RequestRecord
GET  /api/requests/sse          -> Server-Sent Events of RequestRecord
GET  /api/metadata/eval-matrix  -> text/yaml
GET  /api/metadata/risk-survey  -> text/yaml
GET  /api/metadata/readme       -> text/markdown
GET  /                          -> 302 /app/index.html
GET  /app/{*path}               -> static-resources/{*path}
```

## 7. UI

The five-tab structure inherited from the formal exemplar — Overview / Architecture / Risk Survey / Eval Matrix / App UI. Details in `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: Dynamic Route Agent</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label CSS overrides and theme variables from Lesson 24. The App UI tab has a route selector, a text area, an optional target-language field, a submit button, and an SSE-driven list of request cards colored by status.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 (guardrail · before-agent-invocation):** before the endpoint invokes `DynamicAgent`, a config guardrail validates the per-request `AgentSetup` — known route, non-empty text, text within the size limit, target language present when route is TRANSLATE. A failing config records `RequestBlocked` and the agent never runs.
- **G2 (guardrail · before-agent-response):** the agent output passes a response guardrail before the endpoint records `RequestCompleted` — empty or oversized output, or output that echoes the unsafe-config markers, records `RequestBlocked` instead.

## 9. Agent prompts

- `DynamicAgent` — `prompts/dynamic-agent.md`. The base operator instruction; the route-specific instruction is appended per request.

## 10. Acceptance

Passing means the blueprint generated correctly. Inlined from `reference/user-journeys.md`:

1. **Summarize.** POST `/api/requests` with `{ route: "SUMMARIZE", text }` returns a `requestId`; the row reaches `COMPLETED` with non-empty, shorter `output` within ~30 s.
2. **Translate.** POST with `{ route: "TRANSLATE", text, targetLanguage: "French" }` reaches `COMPLETED` with translated `output`.
3. **Blocked config.** POST with empty text (or unknown route) reaches `BLOCKED` with a `blockedReason`; no agent call is made.
4. **Background simulator.** With no interaction, a request appears over SSE within ~45 s and completes.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1-11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named dynamic-route-agent demonstrating the single-agent x
general cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact dynamic-route-agent. Java package
io.akka.samples.dynamicrouteagent. Akka 3.6.0. HTTP port 9133.

Components to wire (exactly):
- 1 Agent DynamicAgent. NO hard-coded instructions on the class. A single
  entry method run(AgentRequest) returning Effect<AgentResult>. Build the
  per-request configuration with AgentSetup / effects(): select the route's
  instruction text (SUMMARIZE vs TRANSLATE) as the systemMessage, pass the
  input text (and target language for TRANSLATE) as the userMessage, and
  responseAs(AgentResult.class). The SUMMARIZE instruction asks for a 2-3
  sentence summary; the TRANSLATE instruction asks for a faithful translation
  into the named target language. Do NOT split this into two agent classes —
  the point of the sample is one generic agent configured per request.
- 1 EventSourcedEntity RequestEntity holding a RequestRecord with id, route
  (Route enum), inputText, RequestStatus enum, submittedAt, and the Optional
  lifecycle fields targetLanguage, output, completedAt, blockedReason,
  blockedAt. Events: RequestSubmitted, RequestCompleted, RequestBlocked.
  Commands: submit, recordCompletion, recordBlock, getRequest. emptyState()
  returns RequestRecord.initial("") with placeholder values and NO
  commandContext() reference (Lesson 3).
- 1 View RequestsView with row type RequestRecord, table updater consuming
  RequestEntity events. ONE query: getAllRequests SELECT * AS requests FROM
  requests_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 1 HttpEndpoint RouteEndpoint at /api: POST /requests (validate config =
  guardrail G1; on pass call DynamicAgent then guardrail G2 then record
  completion or block; on fail record block), GET /requests (list from
  RequestsView.getAllRequests), GET /requests/{id}, GET /requests/sse (SSE
  from the view), and three /api/metadata/* endpoints serving the YAML/MD
  files from src/main/resources/metadata/.
- 1 HttpEndpoint AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.
- 1 TimedAction RequestSimulator (every 45s, reads the next line from
  src/main/resources/sample-events/requests.jsonl and POSTs it through the
  same submit path).

Guardrails (the two eval-matrix controls):
- G1 before-agent-invocation: a validateConfig step in RouteEndpoint that
  rejects unknown route, empty/blank text, text longer than 8000 chars, or
  TRANSLATE without a target language. A rejection records RequestBlocked and
  the agent is never invoked.
- G2 before-agent-response: a validateOutput step that rejects empty output,
  output longer than 16000 chars, or output still carrying the input verbatim
  for a SUMMARIZE route. A rejection records RequestBlocked.

Companion files:
- AgentRequest(Route route, String text, Optional<String> targetLanguage),
  AgentResult(String output), SubmitRequest(String route, String text,
  String targetLanguage) for the inbound JSON.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9133 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/requests.jsonl with 8 canned lines mixing
  SUMMARIZE and TRANSLATE routes.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files the endpoint serves from the classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1, G2) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling the inferable fields and
  marking deployer-specific fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored).
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown, YAML, and mermaid are acceptable. Five tabs: Overview,
  Architecture, Risk Survey, Eval Matrix, App UI. Match the governance.html
  visual style (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (DynamicAgent -> AgentResult; see
  src/main/resources/mock-responses/dynamic-agent.json with 4-6 entries
  covering both routes). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes:
- DynamicAgent: returns AgentResult{ output } — for SUMMARIZE a 2-3 sentence
  condensation of the input; for TRANSLATE a placeholder translation prefixed
  with the target language. The mock keys off the route field so both paths
  produce shape-correct, deterministic output.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: the AI primitive is Agent (request/response) — never silently
  swapped for AutonomousAgent. Since it is an Agent, NO Tasks.java companion.
- Lesson 4: any step that calls the agent gets an explicit timeout of 60s.
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row.
- Lesson 7: no AutonomousAgent here, so no Tasks.java is expected.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9133 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: index.html carries the mermaid state-label CSS overrides and
  theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  by NodeList index; delete removed tabs, never display:none zombie panels.
- emptyState() never calls commandContext().
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars or the mock-model path from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
