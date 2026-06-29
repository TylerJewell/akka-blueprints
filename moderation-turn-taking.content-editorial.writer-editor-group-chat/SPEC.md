# SPEC — writer-editor-group-chat

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 chains the rest of the workflow.

---

## 1. System name + pitch

**System name:** Writer-Editor Group Chat.
**One-line pitch:** The user types a topic; a Writer agent and an Editor agent take moderated round-robin turns in a group chat until the Editor approves the draft or a round limit is reached.

## 2. What this blueprint demonstrates

The moderation-turn-taking coordination pattern: a group chat manager grants the floor to one agent at a time using a RequestToSpeak round-robin protocol, alternating Writer and Editor until a terminal decision. The governance pattern wires a before-agent-response guardrail that quality- and safety-checks each agent's content before it is broadcast to the transcript, and an event-triggered eval that treats each Editor critique as an evaluator decision and records its score. Coordination uses first-party Akka primitives — a durable Workflow as the manager, an event-sourced conversation, and a streaming read model — with no external message bus.

## 3. User-facing flows

1. The user opens the App UI tab and types a topic, then submits. The response carries a `conversationId`. The conversation appears in the live transcript in `IN_PROGRESS`.
2. The manager grants the floor to the Writer. The Writer publishes a draft turn; it appears in the transcript tagged WRITER with a round number.
3. The manager grants the floor to the Editor. The Editor publishes a critique turn with an approve/revise decision and a score; it appears tagged EDITOR.
4. If the Editor revises and the round limit is not reached, the floor returns to the Writer with the critique attached, and the cycle repeats.
5. If the Editor approves, the conversation transitions to `APPROVED` and the final draft is highlighted. If the round limit is reached first, it transitions to `MAX_ROUNDS_REACHED`.
6. A topic feed drips fresh topics on its own, so conversations keep arriving without user input. A conversation with no new turn for two minutes is marked `ESCALATED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| WriterAgent | AutonomousAgent | Drafts and revises content given the topic and the latest critique | GroupChatWorkflow | GroupChatWorkflow |
| EditorAgent | AutonomousAgent | Critiques the latest draft and returns approve/revise + score | GroupChatWorkflow | GroupChatWorkflow |
| GroupChatWorkflow | Workflow | Group chat manager; RequestToSpeak round-robin moderation and guardrail checks | ChatEndpoint, ConversationConsumer | WriterAgent, EditorAgent, ConversationEntity |
| ConversationEntity | EventSourcedEntity | Holds the session: topic, ordered turns, current speaker, round, status | GroupChatWorkflow, ChatEndpoint | TurnView, TurnEvalConsumer |
| InboundTopicQueue | EventSourcedEntity | Records each incoming topic submission | ChatEndpoint, TopicSimulator | ConversationConsumer |
| TurnView | View | Read model of conversations and turns the UI streams | ConversationEntity | ChatEndpoint |
| ConversationConsumer | Consumer | Starts a GroupChatWorkflow per queued topic | InboundTopicQueue | GroupChatWorkflow |
| TurnEvalConsumer | Consumer | Event-triggered eval: records a score when an Editor decision turn is published | ConversationEntity | ConversationEntity |
| TopicSimulator | TimedAction | Drips a canned topic every 30 s | sample-events file | InboundTopicQueue |
| StallMonitor | TimedAction | Escalates conversations with no new turn for 2 min | TurnView | ConversationEntity |
| ChatEndpoint | HttpEndpoint | `/api` surface — submit, list, single, SSE, metadata | UI | InboundTopicQueue, TurnView |
| AppEndpoint | HttpEndpoint | Serves the UI | browser | static-resources |

## 5. Data model

Authoritative. `/akka:implement` writes records exactly as specified. Full field-level detail in `reference/data-model.md`.

- `Conversation` (entity state, also the `TurnView` row type): `String id`, `Optional<String> topic`, `ConversationStatus status`, `Optional<String> currentSpeaker`, `int round`, `List<Turn> turns`, `Optional<String> latestDraft`, `Optional<String> latestCritique`, `Optional<Instant> createdAt`, `Optional<Instant> approvedAt`, `Optional<Instant> escalatedAt`, `Optional<Instant> closedAt`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).
- `Turn`: `String speaker`, `String content`, `int round`, `Optional<Double> evalScore`, `boolean blocked`, `Optional<String> blockedReason`, `Instant at`.
- `ConversationStatus` enum: `IN_PROGRESS`, `APPROVED`, `MAX_ROUNDS_REACHED`, `ESCALATED`, `CLOSED`.
- Events: `ConversationOpened`, `SpeakerGranted`, `TurnPublished`, `TurnBlocked`, `ConversationApproved`, `MaxRoundsReached`, `ConversationEscalated`, `ConversationClosed`.
- Agent result records: `Draft(String content)`, `Critique(boolean approved, String critique, double score)`.

## 6. API contract

Inline surface below; payload schemas and SSE event format in `reference/api-contract.md`.

```
POST /api/conversations              -> { conversationId }
GET  /api/conversations              -> { conversations: [Conversation, ...] }
GET  /api/conversations/{id}         -> Conversation
GET  /api/conversations/sse          -> Server-Sent Events of Conversation
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm). Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI. Browser title `<title>Akka Sample: Writer-Editor Group Chat</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, with no hidden zombie panels (Lesson 26). Architecture-tab mermaid diagrams carry the state-label CSS overrides and theme variables from Lesson 24. Full description in `reference/ui-mockup.md`. The App UI tab submits a topic and streams the live conversation transcript over SSE, tagging each turn WRITER or EDITOR with its round and eval score.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`. Wired by the generated system:

- **G1 — guardrail · before-agent-response.** After each agent produces content and before the manager broadcasts it as a turn, a quality/safety check runs in `GroupChatWorkflow`. A failing check emits `TurnBlocked` and re-requests the speaker rather than publishing.
- **E1 — eval-event · on-decision-eval.** `TurnEvalConsumer` subscribes to `ConversationEntity` events; each Editor decision turn fires an eval that records a score surfaced beside the turn in the UI.
- **A1 — ci-gate · test-gate.** Tests covering the full round-robin loop must pass before deploy.
- **HT1 — halt · operator-regulator-stop.** An operator can halt acceptance of new conversations via a system-control flag the endpoint and consumer check.

## 9. Agent prompts

- `prompts/writer-agent.md` — the Writer drafts and revises a piece given the topic and any prior critique.
- `prompts/editor-agent.md` — the Editor critiques the latest draft and returns an approve/revise decision with a score.

## 10. Acceptance

Inline journeys below; full preconditions/steps/expected in `reference/user-journeys.md`.

1. **Submit a topic and watch one round.** POST `/api/conversations`; within ~30 s a WRITER turn then an EDITOR turn appear in the SSE transcript with round 1.
2. **Round-robin to approval.** The cycle repeats until the Editor approves; status becomes `APPROVED` and the final draft is highlighted.
3. **Round limit.** If the Editor never approves, status becomes `MAX_ROUNDS_REACHED` at the configured limit.
4. **Stall escalation.** A conversation with no new turn for 2 min becomes `ESCALATED`.

---

## 11. Implementation directives

```
Create a sample named writer-editor-group-chat demonstrating the
moderation-turn-taking x content-editorial cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
writer-editor-group-chat. Java package
io.akka.samples.writereditorgroupchat. Akka 3.6.0. HTTP port 9686.

Components to wire (exactly):
- 2 AutonomousAgents:
  - WriterAgent: returns a typed Draft{content}. definition() with
    capability(TaskAcceptance.of(WRITE).maxIterationsPerTask(3)).
  - EditorAgent: returns a typed Critique{approved,critique,score}.
    definition() with capability(TaskAcceptance.of(CRITIQUE)
    .maxIterationsPerTask(3)).
- 1 Workflow GroupChatWorkflow acting as the group chat manager. Steps:
  openStep -> writerTurnStep -> editorTurnStep -> decideStep. The manager
  grants the floor (SpeakerGranted) before each agent step (RequestToSpeak
  round-robin: Writer then Editor). writerTurnStep calls
  forAutonomousAgent(WriterAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), runs the before-agent-response guardrail on
  the returned content; on pass calls ConversationEntity.publishTurn, on fail
  calls ConversationEntity.blockTurn and re-requests the Writer (bounded
  retries). editorTurnStep is analogous and calls publishTurn for the
  decision turn. decideStep: if Critique.approved -> ConversationEntity.approve
  and end; else if round < maxRounds -> increment round, loop to writerTurnStep
  with the critique attached; else -> ConversationEntity.markMaxRounds and end.
  maxRounds defaults to 4. Override settings() with stepTimeout(60s) on
  writerTurnStep and editorTurnStep, and defaultStepRecovery(maxRetries(2)
  .failoverTo(error)).
- 1 EventSourcedEntity ConversationEntity holding a Conversation record with
  id, topic (Optional<String>), ConversationStatus enum, currentSpeaker
  (Optional<String>), round (int), turns (List<Turn>), latestDraft
  (Optional<String>), latestCritique (Optional<String>), and Optional<Instant>
  lifecycle fields createdAt, approvedAt, escalatedAt, closedAt. Events:
  ConversationOpened, SpeakerGranted, TurnPublished, TurnBlocked,
  ConversationApproved, MaxRoundsReached, ConversationEscalated,
  ConversationClosed. Commands: open, grantSpeaker, publishTurn, blockTurn,
  approve, markMaxRounds, escalate, close, getConversation. emptyState()
  returns Conversation.initial("") with no commandContext() reference.
- 1 EventSourcedEntity InboundTopicQueue with command enqueueTopic(topic)
  emitting TopicQueued.
- 1 View TurnView with row type Conversation, table updater consuming
  ConversationEntity events. ONE query: getAllConversations
  SELECT * AS conversations FROM turn_view. No WHERE status filter (Akka
  cannot auto-index enum columns) — filter client-side in callers.
- 1 Consumer ConversationConsumer subscribed to InboundTopicQueue events;
  on each event starts a GroupChatWorkflow with a fresh UUID.
- 1 Consumer TurnEvalConsumer subscribed to ConversationEntity events; on a
  TurnPublished where speaker is EDITOR, records the eval score (event-driven
  eval) back onto the entity or a side note surfaced in the view.
- 2 TimedActions: TopicSimulator (every 30s, reads next line from
  src/main/resources/sample-events/topics.jsonl and calls
  InboundTopicQueue.enqueueTopic); StallMonitor (every 30s, queries
  TurnView.getAllConversations, filters IN_PROGRESS with last turn older than
  2 min, calls ConversationEntity.escalate).
- 2 HttpEndpoints: ChatEndpoint at /api with conversations (POST create, GET
  list filtered client-side from getAllConversations, GET single, GET sse)
  and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- GroupChatTasks.java declaring two Task<R> constants: WRITE (resultConformsTo
  Draft), CRITIQUE (resultConformsTo Critique).
- Draft(String content); Critique(boolean approved, String critique, double
  score).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9686 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/topics.jsonl with 8 canned topic lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1, E1, A1, HT1 and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the tabs, ASCII architecture, project
  layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file (no ui/ folder, no npm). Inline CSS + JS; runtime CDN imports for
  markdown and YAML are acceptable. Five tabs: Overview, Architecture (four
  mermaid diagrams), Risk Survey (matrix-card/matrix-row, mutes deployer
  placeholders), Eval Matrix (matrix-card/matrix-row, colored mechanism pill
  per control), App UI (submit topic, SSE transcript with WRITER/EDITOR turn
  tags, round numbers, eval scores, terminal status banner). Match the
  governance.html visual style (dark/yellow/Instrument Sans/dot-grid).

Generation workflow - see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM - generate a MockModelProvider with per-agent canned/random
  outputs (WriterAgent -> Draft, EditorAgent -> Critique; see
  src/main/resources/mock-responses/{writer-agent,editor-agent}.json with 4-6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var - record the NAME in application.conf.
  (c) Point to an existing env file path - record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI - recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session - value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE - env-var name, file path, secrets URI - never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key
  material.

Mock LLM provider per-agent shapes:
- WriterAgent -> Draft{content}: a 3-5 paragraph piece on the topic; on a
  revision round, visibly addresses the prior critique.
- EditorAgent -> Critique{approved,critique,score}: a short critique string, a
  boolean approved (false on early rounds, true by the last canned entry), a
  score 0.0-1.0.

Constraints - see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: explicit stepTimeout on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row.
- Lesson 7: GroupChatTasks.java companion is required for the AutonomousAgents.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9686.
- Lesson 11: no source-platform or brand names in any user-facing surface.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: descriptive integration label, never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label CSS overrides + theme variables.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
