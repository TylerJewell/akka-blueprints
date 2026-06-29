# SPEC — akka-stateful-memory-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Hello World Stateful Agent.
**One-line pitch:** A user converses with one AI agent; the agent maintains two persistent memory blocks — `persona` (who the agent is) and `human` (what it has learned about the user) — and updates them after every turn so the conversation never loses context.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `MemoryAgent` (AutonomousAgent) handles every turn; the surrounding components prepare its context, sanitize what it writes, and periodically evaluate the accumulated human profile for fairness drift.

Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer each time the agent proposes updates to the human memory block. The sanitizer strips emails, phone numbers, and other identifiers before they are persisted, so the stored profile never contains raw personal data.
- A **periodic drift evaluator** re-scores the human memory block every ten turns — checking whether the accumulated profile has collapsed into demographic stereotypes. It emits a `DriftAnnotation` when the risk score exceeds the threshold.

The blueprint shows that a stateful "hello world" agent still needs meaningful governance: self-modifying memory is a persistent artefact, not an ephemeral conversation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a message into the chat input and presses **Send**. The UI POSTs to `/api/conversations/{id}/turns` and receives a `turnId`.
2. The agent response appears in the chat transcript within ~10–30 s.
3. The **Memory** pane on the right updates: the `persona` block and the `human` block both show their current text, with a diff-chip indicating which lines changed on the last turn.
4. On the tenth turn (and every ten turns thereafter), a `drift-risk` chip appears above the human block. Green = low risk, yellow = watch, red = flag. The chip links to the eval detail.
5. The user can start a new conversation (a fresh `conversationId`); prior conversations remain browsable in the left rail.
6. A service restart leaves all conversations and memory blocks intact — the read model replays from the entity log.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create, send turn, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle: created → active → active (repeated). Holds message history and live memory blocks. Source of truth. | `ConversationEndpoint`, `MemoryWriterConsumer`, `MemoryAgent` | `ConversationView` |
| `MemoryWriterConsumer` | `Consumer` | Subscribes to `AgentResponseReady` events; applies memory patches; sanitizes PII from the proposed human-block update; calls `ConversationEntity.applyMemoryPatch`. | `ConversationEntity` events | `ConversationEntity` |
| `ConversationWorkflow` | `Workflow` | One workflow per turn. Steps: `callAgentStep` → `applyMemoryStep`. Optionally `evalStep` on every tenth turn. | started by `ConversationEndpoint` after turn is submitted | `MemoryAgent`, `ConversationEntity`, `DriftEvaluator` |
| `MemoryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the current persona block, the human block, and the message history as task context; produces `AgentTurnResult` (reply text + proposed memory patches). | invoked by `ConversationWorkflow` | returns `AgentTurnResult` |
| `DriftEvaluator` | supporting class | Deterministic rule-based scorer (no LLM call). Checks the human memory block for demographic-stereotype patterns. Returns `DriftAnnotation{score, reason}`. | invoked by `ConversationWorkflow.evalStep` | returns annotation |
| `ConversationView` | `View` | Read model: one row per conversation for the UI. | `ConversationEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MemoryBlock(String blockId, String content, Instant lastUpdatedAt) {}
// blockId is "persona" or "human" (well-known names)

record MemoryPatch(String blockId, String newContent) {}

record TurnMessage(String role, String text, Instant sentAt) {}
// role: "user" | "agent"

record AgentTurnResult(
    String replyText,
    List<MemoryPatch> proposedPatches,
    Instant producedAt
) {}

record DriftAnnotation(
    double riskScore,    // 0.0..1.0
    String reason,
    int atTurnNumber,
    Instant evaluatedAt
) {}

record Conversation(
    String conversationId,
    MemoryBlock personaBlock,
    MemoryBlock humanBlock,
    List<TurnMessage> history,
    Optional<DriftAnnotation> latestDrift,
    ConversationStatus status,
    Instant createdAt,
    Instant lastActivityAt
) {}

enum ConversationStatus {
    ACTIVE, ARCHIVED
}
```

Events on `ConversationEntity`: `ConversationCreated`, `UserMessageReceived`, `AgentResponseReady`, `MemoryPatchApplied`, `DriftEvaluated`, `ConversationArchived`.

Every nullable lifecycle field on the `Conversation` state is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ initialPersona? }` → `{ conversationId }`.
- `POST /api/conversations/{id}/turns` — body `{ userMessage, submittedBy }` → `{ turnId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with full history and memory blocks.
- `GET /api/conversations/{id}/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Hello World Stateful Agent</title>`.

The App UI tab is a three-column layout: a left rail listing active conversations (age + last message preview), a centre chat pane with the message history and input field, and a right pane showing the live memory blocks (`persona` and `human`) with diff chips and the drift annotation.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MemoryWriterConsumer`): strips emails, phone numbers, government identifiers, and account-like tokens from proposed human-block content before it is persisted. The raw user message is preserved on the entity log for audit; only the sanitized form lands in the human memory block.
- **E1 — periodic drift evaluator** (`eval-periodic`, `drift-fairness-watch`): runs inside `ConversationWorkflow.evalStep` every ten turns. `DriftEvaluator` is deterministic and rule-based (no LLM call — keeping the single-agent invariant honest). It scans the human block for recurring demographic keywords (nationality, race, gender, age, religion) and checks whether the profile has collapsed into a single-attribute stereotype. Emits `DriftEvaluated` with a 0–1 risk score and a one-line reason. Score ≥ 0.7 is flagged red in the UI.

## 9. Agent prompts

- `MemoryAgent` → `prompts/memory-agent.md`. The single decision-making LLM. System prompt instructs it to read the persona and human blocks, respond to the user's message, and propose minimal memory patches that capture only new durable facts.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends three messages with distinct facts about themselves; after each turn the human block shows the merged content and the diff chip highlights what changed; a service restart leaves the blocks intact.
2. **J2** — A user message includes `my email is alice@example.com`; the human block stores `[REDACTED-EMAIL]`; the raw form never appears in the agent's context window on subsequent turns.
3. **J3** — Ten consecutive user messages each mention the same demographic attribute; the drift evaluator fires, emits `DriftEvaluated`, and the UI shows a yellow drift-risk chip on the human block.
4. **J4** — The persona block is seeded with initial text on conversation creation; after ten turns the persona block still contains the seed content merged with any agent-proposed additions; it has not been overwritten wholesale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-stateful-memory-agent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-stateful-memory-agent. Java package
io.akka.samples.helloworldstatefulagent. Akka 3.6.0. HTTP port 9941.

Components to wire (exactly):

- 1 AutonomousAgent MemoryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/memory-agent.md>) and
  .capability(TaskAcceptance.of(GENERATE_TURN).maxIterationsPerTask(3)). The task receives
  the current persona block, human block, and last N messages as the task instruction text;
  output is AgentTurnResult{replyText: String, proposedPatches: List<MemoryPatch>,
  producedAt: Instant}. There is no task attachment.

- 1 Workflow ConversationWorkflow per (conversationId + turnId) with three steps:
  * callAgentStep — calls componentClient.forAutonomousAgent(MemoryAgent.class,
    "agent-" + conversationId).runSingleTask(TaskDef.instructions(
      formatContext(conversation))) — fetches AgentTurnResult. Calls
    ConversationEntity.recordAgentResponse(result). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(ConversationWorkflow::error).
  * applyMemoryStep — picks up AgentResponseReady event; triggers MemoryWriterConsumer
    (Consumer fires asynchronously). WorkflowSettings.stepTimeout 15s.
  * evalStep — fires only when (turnNumber % 10 == 0). Calls DriftEvaluator.evaluate(
    conversation.humanBlock). Calls ConversationEntity.recordDrift(annotation).
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to a failed turn
    but does not archive the conversation.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, personaBlock: MemoryBlock, humanBlock: MemoryBlock,
  history: List<TurnMessage>, latestDrift: Optional<DriftAnnotation>,
  status: ConversationStatus, createdAt: Instant, lastActivityAt: Instant}.
  ConversationStatus enum: ACTIVE, ARCHIVED. Events: ConversationCreated{initialPersona},
  UserMessageReceived{userMessage, submittedBy}, AgentResponseReady{result},
  MemoryPatchApplied{patch}, DriftEvaluated{annotation}, ConversationArchived{}.
  Commands: createConversation, submitUserMessage, recordAgentResponse, applyMemoryPatch,
  recordDrift, archiveConversation, getConversation. emptyState() returns
  Conversation.initial("") with no commandContext() reference (Lesson 3). Optional<T> fields
  use Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MemoryWriterConsumer subscribed to ConversationEntity events; on
  AgentResponseReady: for each proposed MemoryPatch, runs PII sanitization pipeline (emails,
  phone numbers, SSN-like, account-id-like tokens — same regex stages as the sanitizer
  control); produces a sanitized MemoryPatch; calls ConversationEntity.applyMemoryPatch for
  each patch. The raw proposed content is not stored; only the sanitized form persists as
  the memory block content.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation; history is
  truncated to last 50 messages in the view row; full history on the entity). Table updater
  consumes ConversationEntity events. ONE query getAllConversations: SELECT * AS conversations
  FROM conversation_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with POST /conversations (body {initialPersona?: String};
    mints conversationId; calls ConversationEntity.createConversation; returns
    {conversationId}), POST /conversations/{id}/turns (body {userMessage, submittedBy};
    mints turnId; calls ConversationEntity.submitUserMessage; starts ConversationWorkflow;
    returns {turnId}), GET /conversations (list from getAllConversations, newest-first),
    GET /conversations/{id} (one conversation), GET /conversations/{id}/sse (SSE stream
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AgentTasks.java declaring one Task<R> constant: GENERATE_TURN = Task.name("Generate turn")
  .description("Read the persona and human memory blocks and the recent message history,
   reply to the user's message, and propose minimal memory patches for durable new facts")
  .resultConformsTo(AgentTurnResult.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records MemoryBlock, MemoryPatch, TurnMessage, AgentTurnResult, DriftAnnotation,
  Conversation, ConversationStatus.

- DriftEvaluator.java — pure deterministic logic (no LLM). Inputs: MemoryBlock humanBlock,
  int turnNumber. Outputs: DriftAnnotation. Scoring rubric: counts occurrences of demographic
  keyword clusters (nationality, race, gender, age range, religion) in the block content;
  if any single cluster accounts for > 60% of all attribute mentions, risk score =
  clusterCount / totalMentions; returns a reason naming the dominant cluster. Documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9941 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The MemoryAgent.definition() binds the
  configured provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded conversation
  starts: a user introducing themselves as a software engineer, a user discussing a travel
  plan, and a user asking for cooking advice. Each carries an initial persona block and 3
  opening messages so the UI has populated state from first launch.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous
  (the agent responds without human approval on each turn), oversight.human_in_loop = false,
  oversight.human_on_loop = true (a human can review the conversation at any time),
  failure.failure_modes including "pii-leakage-into-memory", "demographic-drift",
  "persona-overwrite", "context-window-overflow"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/memory-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Hello World Stateful Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = conversation list; centre = chat transcript + input; right = memory blocks pane
  with persona + human blocks, diff chips, and drift annotation chip). Browser title
  exactly: <title>Akka Sample: Hello World Stateful Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-turn.json — 8 AgentTurnResult entries. Each entry has a replyText paragraph
    and a proposedPatches list. Patches alternate between updating only the human block
    and updating both blocks. Two entries contain PII in their proposed human-block patch
    (e.g., an email address, a phone number) so S1 has real sanitization work to do.
    One entry proposes no patches at all (agent decided nothing new is worth remembering).
- A MockModelProvider.seedFor(conversationId) helper makes per-conversation selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MemoryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (callAgentStep 60s,
  applyMemoryStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation state is Optional<T>.
- Lesson 7: AgentTasks.java with GENERATE_TURN = Task.name(...).description(...)
  .resultConformsTo(AgentTurnResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9941 declared explicitly in application.conf's
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
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (MemoryAgent). The
  DriftEvaluator is rule-based and does NOT make an LLM call — keeping the pattern's "one
  agent" promise honest.
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
