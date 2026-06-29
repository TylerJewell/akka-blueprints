# SPEC — ui-demo

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AG-UI + CopilotKit Integration.
**One-line pitch:** A user types a prompt into a CopilotKit-powered chat interface; one AI agent produces a structured `CopilotResponse` — answer text with inline citations and a suggested follow-up — streamed token-by-token to the browser via the AG-UI Server-Sent Events protocol.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `CopilotAgent` (AutonomousAgent) handles the entire answer-generation step; surrounding components manage the session lifecycle, stream the output, and guard the response boundary.

One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates the agent's structured output on every turn: the response must parse as well-formed `CopilotResponse` JSON, every `citations[].source` must be a non-empty string, `answer` must not be blank, and total answer length must not exceed the configured `maxTokens` ceiling. A failing response triggers a retry inside the same task iteration budget.

The blueprint shows that a streaming copilot UI does not mean an unguarded one. The guardrail intercepts the structured envelope before the SSE stream commits the final payload to the session log.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Prompt** input or picks one of five seeded example prompts from a dropdown.
2. The user clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` and immediately opens an SSE stream to `/api/sessions/{sessionId}/stream`.
3. The answer streams token-by-token in the chat bubble. Inline citation markers (`[1]`, `[2]`) appear as the text arrives. A spinner shows `Thinking…` until the first token lands.
4. When the agent finishes, the final `CopilotResponse` is committed to `SessionEntity` and the stream closes. The session card transitions to `TURN_COMPLETE`.
5. A **Suggested follow-up** chip appears below the answer. Clicking it pre-fills the prompt input.
6. The user can continue the conversation; each turn is appended to the session history visible in the left rail.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CopilotEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create session, submit turn, stream SSE, get history; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView`, `SessionWorkflow` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: created → turn-submitted → streaming → turn-complete → (next turn). Holds full turn history. | `CopilotEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session turn. Steps: `agentCallStep` → `commitStep`. Started by `CopilotEndpoint` on each `/turns` POST. | `CopilotEndpoint` | `CopilotAgent`, `SessionEntity` |
| `CopilotAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the prompt and session history in the task definition; returns `CopilotResponse`. The `before-agent-response` guardrail (`ResponseGuardrail`) is wired here. | invoked by `SessionWorkflow` | returns response |
| `ResponseGuardrail` | guardrail class | `before-agent-response` hook. Validates `CopilotResponse` structure, citation non-emptiness, answer non-blankness, and token-length ceiling before the response leaves the agent loop. | registered on `CopilotAgent` | returns accept / reject |
| `SessionView` | `View` | Read model: one row per session for the UI list. | `SessionEntity` events | `CopilotEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Turn(
    String turnId,
    String prompt,
    Optional<CopilotResponse> response,
    TurnStatus status,
    Instant submittedAt,
    Optional<Instant> completedAt
) {}

record CopilotResponse(
    String answer,
    List<Citation> citations,
    String suggestedFollowUp,
    int tokenCount,
    Instant generatedAt
) {}

record Citation(
    int index,         // 1-based marker matching [n] in answer text
    String source,     // non-empty label, e.g. "Akka docs §3.2" or a URL
    String snippet     // verbatim passage supporting the citation
) {}

record Session(
    String sessionId,
    String title,
    List<Turn> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> lastActiveAt
) {}

enum TurnStatus { SUBMITTED, STREAMING, TURN_COMPLETE, TURN_FAILED }
enum SessionStatus { ACTIVE, IDLE, FAILED }
```

Events on `SessionEntity`: `SessionCreated`, `TurnSubmitted`, `StreamingStarted`, `TurnCompleted`, `TurnFailed`.

Every nullable lifecycle field on `Turn` and `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ title }` → `{ sessionId }`.
- `POST /api/sessions/{sessionId}/turns` — body `{ prompt }` → `{ turnId }`.
- `GET /api/sessions/{sessionId}/stream` — Server-Sent Events; AG-UI protocol envelopes streamed per token and on final commit.
- `GET /api/sessions/{sessionId}` — full session including all turns.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE envelope format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AG-UI + CopilotKit Integration</title>`.

The App UI tab is a two-column layout: a left rail with the session list (title + status + last active) and a right pane with the active session's chat history — each turn rendered as a user bubble (prompt) and an agent bubble (streaming answer with citation markers and suggested-follow-up chip).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `CopilotAgent`. Asserts the candidate response is well-formed `CopilotResponse` JSON, `answer` is non-blank, every `citations[].source` is a non-empty string, every citation `index` is a positive integer, and `tokenCount` does not exceed the deployment ceiling (default 2048). On failure, returns a structured `invalid-response` rejection to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `CopilotAgent` → `prompts/copilot-agent.md`. The single decision-making LLM. System prompt instructs it to answer the user's prompt using retrieved context, produce inline citation markers matching the `citations` list, and include one concise suggested follow-up question.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User types a seeded prompt; within 30 s the answer streams to completion, citations appear, and the suggested follow-up chip is active.
2. **J2** — The agent's first response is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed response; the chat bubble never flashes the malformed payload.
3. **J3** — Re-opening the same session shows the full turn history in order; no turns are missing or duplicated.
4. **J4** — A response exceeding the token-length ceiling is rejected by the guardrail; the agent retries with a shorter answer; the final response is within the allowed length.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ui-demo demonstrating the single-agent × general cell. Runs out of the
box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-ui-demo. Java package io.akka.samples.aguicopilotkitintegration.
Akka 3.6.0. HTTP port 9352.

Components to wire (exactly):

- 1 AutonomousAgent CopilotAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/copilot-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_PROMPT).maxIterationsPerTask(3)). The task receives
  the user prompt and the last 10 turns of history as its instruction text (formatted as a
  conversation transcript). Output: CopilotResponse{answer: String, citations: List<Citation>,
  suggestedFollowUp: String, tokenCount: int, generatedAt: Instant}. The agent is configured
  with a before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries the
  response within its 3-iteration budget.

- 1 Workflow SessionWorkflow per (sessionId, turnId) with two steps:
  * agentCallStep — emits StreamingStarted, then calls componentClient.forAutonomousAgent(
    CopilotAgent.class, "copilot-" + sessionId).runSingleTask(
      TaskDef.instructions(formatTurnContext(session, prompt))
    ) — returns a taskId, then forTask(taskId).result(ANSWER_PROMPT) to fetch the response.
    On success calls SessionEntity.completeTurn(turnId, response). WorkflowSettings
    .stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(SessionWorkflow::error).
  * commitStep — calls SessionEntity.markIdle(). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  title: String, turns: List<Turn>, status: SessionStatus, createdAt: Instant,
  lastActiveAt: Optional<Instant>}. SessionStatus enum: ACTIVE, IDLE, FAILED. Events:
  SessionCreated{title}, TurnSubmitted{turn}, StreamingStarted{turnId},
  TurnCompleted{turnId, response}, TurnFailed{turnId, reason}. Commands: createSession,
  submitTurn, markStreaming, completeTurn, failTurn, markIdle, getSession.
  emptyState() returns Session.initial("") with empty turns list and no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 View SessionView with row type SessionRow (mirrors Session: sessionId, title, status,
  turnCount, createdAt, lastActiveAt). Table updater consumes SessionEntity events. ONE
  query getAllSessions: SELECT * AS sessions FROM session_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * CopilotEndpoint at /api with:
    POST /sessions (body {title}; mints sessionId; calls SessionEntity.createSession; returns
      {sessionId}),
    POST /sessions/{sessionId}/turns (body {prompt}; mints turnId; calls
      SessionEntity.submitTurn; starts SessionWorkflow; returns {turnId}),
    GET /sessions/{sessionId}/stream (SSE stream forwarded from CopilotAgent task stream
      — AG-UI protocol envelopes; one envelope per token chunk and one RESPONSE_COMPLETE
      envelope on commit),
    GET /sessions/{sessionId} (full session including all turns),
    GET /sessions (list from getAllSessions, sorted newest-first),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- CopilotTasks.java declaring one Task<R> constant: ANSWER_PROMPT = Task.name("Answer prompt")
  .description("Read the conversation context and produce a CopilotResponse with answer,
  citations, and a suggested follow-up question").resultConformsTo(CopilotResponse.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Turn, CopilotResponse, Citation, Session, TurnStatus, SessionStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  CopilotResponse from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9352 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CopilotAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 5 seeded prompts covering common
  copilot use cases (e.g., "What are the key concepts in event-sourced systems?", "How does
  the AG-UI protocol differ from GraphQL subscriptions?", "What is a good pattern for
  handling idempotency in distributed workflows?", "Explain the trade-offs between polling
  and SSE for real-time UI updates.", "How do I pass context securely to an AI agent without
  leaking it to the model?"). Each entry includes a mock response payload with citations.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in Section
  8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (the copilot
  session store holds user prompts; PII presence depends on what the user types —
  TO_BE_COMPLETED_BY_DEPLOYER), decisions.authority_level = recommend-only (the agent's
  answer is informational, not enforced), oversight.human_in_loop = false (fully automated
  response path), failure.failure_modes including "hallucinated-citation",
  "blank-answer", "over-length-response", "citation-index-mismatch";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/copilot-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AG-UI + CopilotKit Integration",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session list rail; right = chat history with streaming agent bubble, citation chips, and
  suggested-follow-up button). Browser title exactly:
  <title>Akka Sample: AG-UI + CopilotKit Integration</title>. No subtitle on the
  Overview tab.

  The streaming chat bubble receives AG-UI SSE envelopes:
  - TOKEN_CHUNK: { text } — append to current bubble text
  - CITATION_ADDED: { index, source, snippet } — render citation marker [n]
  - RESPONSE_COMPLETE: { response } — finalise the bubble, show suggested follow-up chip

  Tab switching MUST be attribute-based (Lesson 26).

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
    answer-prompt.json — 6 CopilotResponse entries covering varied prompt topics.
      Each entry has a non-empty answer with citation markers ([1], [2]), a citations array
      with matching index values and non-empty source and snippet fields, a suggestedFollowUp
      question, a tokenCount within [100, 1800], and a generatedAt timestamp. Plus 2
      deliberately MALFORMED entries (one with an empty answer; one with a citations entry
      whose source is an empty string) — the guardrail blocks both, exercising the retry
      path. The mock should select a malformed entry on the FIRST iteration of every 4th
      turn (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CopilotAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CopilotTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (agentCallStep 60s, commitStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Turn and Session is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: CopilotTasks.java with ANSWER_PROMPT = Task.name(...).description(...)
  .resultConformsTo(CopilotResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9352 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CopilotAgent). The
  session history management and view projection are deterministic in-process logic — no
  second LLM call.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
