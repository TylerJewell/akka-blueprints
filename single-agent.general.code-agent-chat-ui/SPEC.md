# SPEC — code-agent-chat-ui

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GradioCodeAgent.
**One-line pitch:** A user types a natural-language question into a streaming chat UI; one AI agent executes web searches and code to answer it, replanning every 3 steps, with guardrails on every tool call and every outbound message.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `CodeAssistantAgent` (AutonomousAgent) carries all reasoning; the surrounding components govern its inputs and outputs. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before every tool invocation. Code execution tool calls require a simulated sandbox boundary: the guardrail checks that the requested code snippet does not attempt file-system escape, network exfiltration, or process spawning outside the allowed list. Web search calls are checked for query length and allowed-domain constraints. A blocked call returns a structured rejection so the agent can reason around the constraint.
- A **before-agent-response guardrail** runs on every chat turn before the response streams to the user. It checks that the response does not contain raw code-execution output mixed with unsanitized error tracebacks, does not echo tool-call secrets from the request context, and meets a minimum-coherence heuristic (response is not empty, not a raw JSON dump, not a bare stack trace).

The blueprint shows that a general-purpose code agent is not inherently ungoverned — two independent guardrails bracket the one decision-making LLM.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a natural-language question into the **Chat input** field (e.g., "What is the current Python version and can you show me a hello-world?").
2. The user clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/messages` and immediately opens an SSE stream to receive the response.
3. The agent's thinking steps appear incrementally: a web-search tool call, a code-execution tool call, intermediate planning. Each step is shown in a collapsible **Steps** panel in the chat bubble.
4. Every 3 steps, the agent emits a `PlanRevision` event; the UI shows a small "Replanned" badge in the steps panel.
5. The agent's final message appears fully rendered (markdown + code blocks) once the `before-agent-response` guardrail passes it.
6. The user can continue the conversation. The session retains history; subsequent turns include prior context.
7. A side panel lists all active and past sessions. Clicking a past session replays its message history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create, send message, list, get, SSE; `/api/metadata/*`. | — | `ChatSessionEntity`, `ChatView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ChatSessionEntity` | `EventSourcedEntity` | Per-session lifecycle and history. Source of truth. | `ChatEndpoint`, `ChatSessionWorkflow` | `ChatView` |
| `ChatSessionWorkflow` | `Workflow` | One workflow per session. Steps: `initStep` → `activeStep` → `idleStep`. | started by `ChatEndpoint` on session create | `CodeAssistantAgent`, `ChatSessionEntity` |
| `CodeAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the conversation history as task instructions; uses `WebSearchTool` and `CodeExecutionTool`; replans every 3 steps. Returns `ChatResponse`. | invoked by `ChatSessionWorkflow` | returns response |
| `ToolCallValidator` | `Consumer` | Subscribes to `ToolCallRequested` events; runs the `before-tool-call` policy; emits `ToolCallApproved` or `ToolCallBlocked`. | `ChatSessionEntity` events | `ChatSessionEntity` |
| `ResponseGuardrail` | supporting class | Registered on `CodeAssistantAgent`'s `before-agent-response` hook. Validates outbound chat text. | invoked by agent loop | pass / reject |
| `ChatView` | `View` | Read model: one row per session for the UI. | `ChatSessionEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ChatMessage(
    String messageId,
    MessageRole role,      // USER | ASSISTANT | SYSTEM
    String content,
    Instant sentAt,
    Optional<List<ToolCallRecord>> toolCalls   // only on ASSISTANT turns
) {}
enum MessageRole { USER, ASSISTANT, SYSTEM }

record ToolCallRecord(
    String toolCallId,
    String toolName,       // "web-search" | "code-execution"
    String inputSummary,   // truncated for display
    ToolCallStatus status, // PENDING | APPROVED | BLOCKED | COMPLETED | FAILED
    Optional<String>  blockReason,
    Optional<String>  outputSummary,
    Instant requestedAt,
    Optional<Instant> resolvedAt
) {}
enum ToolCallStatus { PENDING, APPROVED, BLOCKED, COMPLETED, FAILED }

record PlanRevision(
    int stepNumberAtRevision,
    String revisedGoal,
    Instant revisedAt
) {}

record ChatResponse(
    String responseText,
    List<ToolCallRecord> toolCallLog,
    List<PlanRevision> planRevisions,
    Instant generatedAt
) {}

record ChatSession(
    String sessionId,
    String title,          // auto-derived from the first user message
    List<ChatMessage> messages,
    List<PlanRevision> planRevisions,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> lastActiveAt
) {}

enum SessionStatus {
    INITIALIZING, ACTIVE, IDLE, FAILED
}
```

Events on `ChatSessionEntity`: `SessionCreated`, `UserMessageReceived`, `AssistantMessageRecorded`, `ToolCallRequested`, `ToolCallResolved`, `PlanRevised`, `SessionFailed`.

Every nullable lifecycle field on the `ChatSession` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ title? }` → `{ sessionId }`.
- `POST /api/sessions/{id}/messages` — body `{ content }` → `{ messageId }`, then streams response via SSE.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with full message history.
- `GET /api/sessions/{id}/sse` — Server-Sent Events; one event per session state change or new message.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Gradio UI Code Agent</title>`.

The App UI tab is a two-column layout: a left rail listing active and past sessions (status pill + title + age) and a right pane rendering the selected session's chat thread — messages with collapsible tool-call step panels, plan-revision badges, and an input box at the bottom.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`, wired inside `CodeAssistantAgent`'s guardrail configuration): runs before every tool invocation. For `code-execution` calls, checks that the requested snippet does not use disallowed syscalls or attempt file-system paths outside `/sandbox`. For `web-search` calls, checks query length (≤ 512 chars) and a blocked-domain list. On rejection returns a structured `tool-blocked` message to the agent loop; the agent may rephrase and retry within its iteration budget.
- **G2 — before-agent-response guardrail** (`guardrail`, `before-agent-response`, implemented by `ResponseGuardrail`): runs on every outbound chat turn. Checks that the response is not empty, not a bare JSON object, not a raw stack trace, and does not contain `[CONTEXT_SECRET]` tokens that indicate the agent accidentally echoed request-context material. On failure, returns a structured `invalid-response` error; the agent loop retries.

## 9. Agent prompts

- `CodeAssistantAgent` → `prompts/code-assistant.md`. The single decision-making LLM. System prompt instructs it to answer questions by searching and writing code, re-evaluate its plan every 3 steps, and produce a final response that is human-readable markdown.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends "What Python version is current and show a hello-world?"; within 30 s the agent responds with a code block and an eval tile.
2. **J2** — Mock agent attempts a `code-execution` call with a disallowed syscall; the `before-tool-call` guardrail blocks it; the log records `ToolCallBlocked`; the agent continues and produces a response that acknowledges the constraint.
3. **J3** — Mock agent produces a bare JSON dump as its first response candidate; the `before-agent-response` guardrail rejects it; the second iteration produces a human-readable answer; the UI never displays the raw JSON.
4. **J4** — A session with 9 turns includes 3 `PlanRevision` events (at steps 3, 6, 9); each revision badge is visible in the UI steps panel.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named gradiouicodeagent demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-code-agent-chat-ui. Java package io.akka.samples.gradiouicodeagent.
Akka 3.6.0. HTTP port 9809.

Components to wire (exactly):

- 1 AutonomousAgent CodeAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-assistant.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(12)). The task receives
  the full conversation history as its instruction text. The agent is configured with two
  guardrail hooks:
    (a) before-tool-call: bound to ToolCallPolicy.java, which validates each tool invocation
        before it executes.
    (b) before-agent-response: bound to ResponseGuardrail.java, which validates the outbound
        text before it streams to the user.
  On guardrail rejection the agent loop retries within its 12-iteration budget.
  The agent's definition includes two tools: WebSearchTool (simulated in-process) and
  CodeExecutionTool (simulated sandbox). The agent replans every 3 steps by checking
  step count modulo 3 and emitting a PlanRevision when the condition triggers.

- 1 Workflow ChatSessionWorkflow per sessionId with three steps:
  * initStep — calls ChatSessionEntity.markActive; transitions to activeStep immediately.
    WorkflowSettings.stepTimeout 5s.
  * activeStep — waits for a UserMessageReceived event on the entity (polls every 1s up to
    timeout); on detection calls componentClient.forAutonomousAgent(
    CodeAssistantAgent.class, "agent-" + sessionId).runSingleTask(
      TaskDef.instructions(formatHistory(session.messages))
    ); records the ChatResponse via ChatSessionEntity.recordResponse.
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(ChatSessionWorkflow::error).
  * idleStep — marks entity IDLE. Re-activates when a new UserMessageReceived event arrives.
    WorkflowSettings.stepTimeout 30s.
  error step transitions entity to FAILED. WorkflowSettings is Workflow.WorkflowSettings —
  DO NOT import a top-level WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ChatSessionEntity (one per sessionId). State ChatSession{sessionId:
  String, title: String, messages: List<ChatMessage>, planRevisions: List<PlanRevision>,
  status: SessionStatus, createdAt: Instant, lastActiveAt: Optional<Instant>}.
  SessionStatus enum: INITIALIZING, ACTIVE, IDLE, FAILED.
  Events: SessionCreated{title}, UserMessageReceived{message}, AssistantMessageRecorded{message},
  ToolCallRequested{record}, ToolCallResolved{toolCallId, status, outputSummary, blockReason},
  PlanRevised{revision}, SessionFailed{reason}.
  Commands: create, receiveUserMessage, recordResponse, resolveToolCall, recordPlanRevision,
  markActive, markIdle, fail, getSession.
  emptyState() returns ChatSession.initial("") with all list fields as empty lists and
  status = INITIALIZING (Lesson 3). Every Optional<T> field uses Optional.empty() in initial
  state and Optional.of(...) inside the event-applier.

- 1 Consumer ToolCallValidator subscribed to ChatSessionEntity events; on ToolCallRequested
  runs the tool-call policy: for code-execution calls checks the inputSummary for disallowed
  patterns (file-system escapes, process spawning, network sockets outside the simulated
  sandbox); for web-search calls checks query length and a blocked-domain list. Calls
  ChatSessionEntity.resolveToolCall with APPROVED or BLOCKED + reason.

- 1 View ChatView with row type ChatSessionRow (mirrors ChatSession minus per-message
  rawContent — the full history is available via GET /api/sessions/{id}). Table updater
  consumes ChatSessionEntity events. ONE query getAllSessions: SELECT * AS sessions FROM
  chat_session_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with POST /sessions (body {title?}; mints sessionId; calls
    ChatSessionEntity.create; starts ChatSessionWorkflow; returns {sessionId}),
    POST /sessions/{id}/messages (body {content}; calls ChatSessionEntity.receiveUserMessage;
    returns {messageId}), GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one session), GET /sessions/{id}/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AgentTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Use available tools to answer the user's question and return a
  ChatResponse").resultConformsTo(ChatResponse.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records ChatMessage, MessageRole, ToolCallRecord, ToolCallStatus, PlanRevision,
  ChatResponse, ChatSession, SessionStatus.

- ToolCallPolicy.java implementing the before-tool-call hook. Reads the pending tool name
  and input, runs the two-policy check (code-execution policy + web-search policy), and
  either passes the call through or returns Guardrail.reject(<structured-error>) with the
  policy name and the failing pattern.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  response text, runs the four checks listed in eval-matrix.yaml G2, and either passes
  the response through or returns Guardrail.reject(<structured-error>) naming the check
  that failed.

- WebSearchTool.java and CodeExecutionTool.java — simulated in-process tools. WebSearchTool
  returns a fixed set of plausible search results keyed to known queries in the seed prompts.
  CodeExecutionTool runs the code string through a whitelist-only evaluator (only arithmetic
  and string operations) and returns a simulated output. Both tools record a ToolCallRecord
  on the entity before and after execution.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9809 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-prompts.jsonl with 5 seeded conversation starters:
  "What is the current Python version?", "Calculate the 20th Fibonacci number",
  "Search for recent news about Akka and summarize it", "Write a function to check if
  a number is prime", "What are the top 3 results for 'event sourcing patterns'?".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with the pre-filled fields from Section 8 and
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-assistant.md loaded as the agent system prompt.

- README.md at the project root as specified.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = session list; right = chat thread with tool-step panels, plan-revision badges,
  and message input). Browser title exactly:
  <title>Akka Sample: Gradio UI Code Agent</title>. No subtitle on the Overview tab.

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
    answer-question.json — 6 ChatResponse entries covering varied question types.
      Each entry has a responseText (markdown with at least one code block), a toolCallLog
      of 2-4 records (mix of web-search and code-execution, all COMPLETED), and
      1-2 PlanRevision entries. Plus 2 deliberately MALFORMED entries — one with an
      empty responseText string (blocked by G2), one with a raw JSON dump as responseText
      (blocked by G2) — exercising the retry path.
  A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (initStep 5s, activeStep 90s,
  idleStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on the ChatSession row record is Optional<T>.
- Lesson 7: AgentTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(ChatResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9809 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeAssistantAgent).
  ToolCallValidator is a Consumer (not an agent); ResponseGuardrail and ToolCallPolicy
  are supporting classes (not agents).
- The conversation history is passed as TaskDef.instructions(...); tool calls are declared
  inside the agent definition and executed by the Akka agent runtime, NOT by the
  ChatSessionWorkflow directly.
- Both guardrail hooks (before-tool-call, before-agent-response) are wired via the agent's
  guardrail-configuration block, not as external checks after the agent returns.
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
