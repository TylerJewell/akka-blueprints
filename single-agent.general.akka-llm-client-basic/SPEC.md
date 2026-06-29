# SPEC — akka-llm-client-basic

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ConversationAPI LLM Client.
**One-line pitch:** A user sends a text prompt; one AI agent forwards it to the configured model provider via Akka's Conversation API and returns a typed reply — establishing the provider-portable LLM interface that every subsequent durable-agent quickstart builds on.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain, stripped to its essential form. One `ConversationAgent` (AutonomousAgent) carries the entire model interaction; the surrounding components only manage session state and output safety. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** (`ReplyGuardrail`) validates the agent's reply on every turn: non-empty reply text, reply length within the configured bound, and no obviously unsolicited refusal signals (empty body paired with a refusal metadata flag). A failing candidate triggers a retry inside the same task.

The blueprint shows that even a minimal LLM call benefits from a structured output gate. It deliberately avoids tools, multi-step workflows, and domain-specific pipelines so readers can trace the Conversation API surface without noise.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of four seeded examples — a factual question, a code-generation task, a summarisation request, and a creative brief).
2. The user picks a **model provider** from a dropdown (Anthropic / OpenAI / Google AI / Mock) or accepts the default configured at startup.
3. The user clicks **Send**. The UI POSTs to `/api/conversations/{sessionId}/turns` and receives a `turnId`.
4. The card for the new turn appears in the session history panel in `PENDING` state.
5. Within ~5–20 s, the agent completes the model call and the guardrail passes the reply. The card transitions to `REPLIED`. The reply text appears in the right pane along with latency, token counts (if the provider returns them), and the guardrail outcome.
6. The user can type another prompt in the same session; the previous turns are included as conversation history in the next model call.
7. A **New session** button resets the session id so conversation history starts fresh.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create session, add turn, get session, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-session lifecycle: holds turn history; tracks status of each turn. | `ConversationEndpoint`, `ConversationAgent` | `ConversationView` |
| `ConversationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives a prompt plus prior turns as context; returns `ConversationReply`. | invoked by `ConversationEndpoint` after writing `TurnSubmitted` | returns reply |
| `ReplyGuardrail` | supporting class | `before-agent-response` hook on `ConversationAgent`. Validates reply text is non-empty, within length bound, and not a structured refusal. | — | pass-through or reject |
| `ConversationView` | `View` | Read model: one row per session for the UI. | `ConversationEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Turn(
    String turnId,
    String prompt,
    Optional<ConversationReply> reply,
    TurnStatus status,
    Instant submittedAt,
    Optional<Instant> repliedAt
) {}
enum TurnStatus { PENDING, REPLIED, FAILED }

record ConversationReply(
    String text,
    Optional<Integer> inputTokens,
    Optional<Integer> outputTokens,
    Optional<Long> latencyMs,
    Instant generatedAt
) {}

record ConversationSession(
    String sessionId,
    List<Turn> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> lastActivityAt
) {}
enum SessionStatus { ACTIVE, CLOSED }

record TurnRequest(
    String prompt,
    Optional<String> modelOverride
) {}
```

Events on `ConversationEntity`: `SessionCreated`, `TurnSubmitted`, `TurnReplied`, `TurnFailed`.

Every nullable lifecycle field on the `ConversationSession` row record and `Turn` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{}` → `{ sessionId }`. Creates a new session.
- `POST /api/conversations/{sessionId}/turns` — body `{ prompt, modelOverride? }` → `{ turnId }`.
- `GET /api/conversations/{sessionId}` — full session with all turns.
- `GET /api/conversations/{sessionId}/sse` — Server-Sent Events; one event per turn-status change.
- `GET /api/conversations` — list all sessions, newest-first.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Conversation API LLM Client</title>`.

The App UI tab is a two-column layout: a left rail with the session prompt input, a provider/model selector, and a scrollable history of turns (status pill + reply preview + latency chip); a right pane with the full reply text, token counts, guardrail outcome badge, and a timestamp.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ConversationAgent`. Asserts the candidate reply text is non-empty, the reply text length does not exceed 8 000 characters, and the reply does not carry a structured-refusal flag alongside an empty body. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `ConversationAgent` → `prompts/conversation-agent.md`. The single decision-making LLM. System prompt instructs the agent to respond helpfully and concisely, include its reasoning when useful, and always return non-empty reply text even for edge-case inputs.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a seeded factual prompt; within 20 s a non-empty `ConversationReply` appears with `status = REPLIED`.
2. **J2** — The agent's first response fails the guardrail (mock LLM path returns an empty body); the guardrail rejects it; the second iteration produces a valid reply; the turn never transitions through a state with empty reply text visible in the UI.
3. **J3** — User submits an empty string prompt; the endpoint returns `400 Bad Request` before any agent call is made; the session history is unchanged.
4. **J4** — User sends three prompts in the same session; each new turn call includes prior turns as context; the entity's turn list grows to three entries.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-llm-client-basic demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-llm-client-basic. Java package
io.akka.samples.conversationapillmclient. Akka 3.6.0. HTTP port 9533.

Components to wire (exactly):

- 1 AutonomousAgent ConversationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/conversation-agent.md>) and
  .capability(TaskAcceptance.of(GENERATE_REPLY).maxIterationsPerTask(3)). The task receives
  the user prompt plus formatted prior turns as its instruction text. Output:
  ConversationReply{text: String, inputTokens: Optional<Integer>,
  outputTokens: Optional<Integer>, latencyMs: Optional<Long>, generatedAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the response within its 3-iteration budget.

- 1 EventSourcedEntity ConversationEntity (one per sessionId). State
  ConversationSession{sessionId: String, turns: List<Turn>, status: SessionStatus,
  createdAt: Instant, lastActivityAt: Optional<Instant>}. SessionStatus enum: ACTIVE, CLOSED.
  Turn record: {turnId: String, prompt: String, reply: Optional<ConversationReply>,
  status: TurnStatus, submittedAt: Instant, repliedAt: Optional<Instant>}.
  TurnStatus enum: PENDING, REPLIED, FAILED.
  Events: SessionCreated{sessionId}, TurnSubmitted{turnId, prompt, submittedAt},
  TurnReplied{turnId, reply, repliedAt}, TurnFailed{turnId, reason}.
  Commands: createSession, submitTurn, recordReply, failTurn, closeSession, getSession.
  emptyState() returns ConversationSession.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 View ConversationView with row type ConversationRow (mirrors ConversationSession including
  all turns — the view exposes full turn history). Table updater consumes ConversationEntity
  events. ONE query getAllSessions: SELECT * AS sessions FROM conversation_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with:
      POST /conversations (body {}; mints sessionId; calls ConversationEntity.createSession;
        returns {sessionId}),
      POST /conversations/{sessionId}/turns (body {prompt, modelOverride?}; validates prompt
        is non-empty — return 400 if blank; mints turnId; calls ConversationEntity.submitTurn;
        then immediately invokes ConversationAgent via componentClient.forAutonomousAgent(
          ConversationAgent.class, "agent-" + sessionId)
          .runSingleTask(TaskDef.instructions(formatTurnContext(session, prompt))) and awaits
          result(GENERATE_REPLY); calls ConversationEntity.recordReply(reply) on success or
          failTurn(reason) on agent error; returns {turnId}),
      GET /conversations (list all sessions from getAllSessions, sorted newest-first),
      GET /conversations/{sessionId} (one row),
      GET /conversations/{sessionId}/sse (Server-Sent Events forwarded from the view's
        stream-updates),
      GET /api/metadata/* (readme, risk-survey, eval-matrix from classpath metadata/).
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ConversationTasks.java declaring one Task<R> constant: GENERATE_REPLY = Task
  .name("Generate reply").description("Respond to the user prompt given prior conversation
  context").resultConformsTo(ConversationReply.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records: Turn, TurnRequest, TurnStatus, ConversationReply, ConversationSession,
  SessionStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ConversationReply from the LLM response, runs the three checks listed in eval-matrix.yaml
  G1 (non-empty text, length <= 8000 chars, no refusal-flag + empty-body combination), and
  either passes the response through or returns Guardrail.reject(<structured-error>) to force
  the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9533 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. ConversationAgent.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 4 seeded prompts:
    a factual question ("What is the capital of Iceland, and what is its estimated population?"),
    a code-generation task ("Write a Java method that reverses a string without using StringBuilder."),
    a summarisation request ("Summarise the following passage in two sentences: [100-word passage about cloud computing]"),
    a creative brief ("Write the opening paragraph of a noir detective story set in a flooded city.").

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = informational
  (the agent's reply is content, not a governance decision), oversight.human_in_loop = false
  (conversational output is read directly by the user with no gating reviewer),
  failure.failure_modes including "hallucinated-content", "refusal-loop", "empty-reply",
  "over-length-reply"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/conversation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Conversation API LLM Client",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  prompt input + provider selector + turn history list; right = selected-turn detail with full
  reply text, token counts, latency, guardrail outcome badge).
  Browser title exactly: <title>Akka Sample: Conversation API LLM Client</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
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
    generate-reply.json — 6 ConversationReply entries, one per logical reply type:
      a normal factual answer (non-empty text, plausible token counts),
      a concise code block reply,
      a multi-sentence summarisation reply,
      a creative paragraph reply,
      plus 2 deliberately MALFORMED entries — one with an empty text field (guardrail check 1
      fails), one with a text field exceeding 8 000 chars (guardrail check 2 fails). The mock
      selects a malformed entry on the FIRST iteration of every 4th turn (modulo seed) so J2
      is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ConversationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ConversationTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout. For the
  inline agent call inside ConversationEndpoint, set a server-side 30 s timeout on the
  component-client call.
- Lesson 6: every nullable lifecycle field on the ConversationSession and Turn records is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: ConversationTasks.java with GENERATE_REPLY = Task.name(...).description(...)
  .resultConformsTo(ConversationReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9533 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ConversationAgent).
  No Workflow is introduced. The guardrail (ReplyGuardrail.java) is a supporting class
  registered on the agent's guardrail-configuration block, not a second agent.
- The agent call is made directly from ConversationEndpoint's POST /turns handler via
  componentClient.forAutonomousAgent(...).runSingleTask(...). There is no intermediate
  Workflow because this baseline deliberately minimises moving parts.
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
