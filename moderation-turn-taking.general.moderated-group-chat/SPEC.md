# Sample Specification — `moderated-group-chat`

This document is the **natural-language input for `/akka:specify`**. The user runs `/akka:specify @SPEC.md` from inside this folder; the `@SPEC.md` token inlines the whole file as the skill's argument. Sections 1–11 together are the spec; Section 12 tells Claude how to continue after scaffolding.

---

## 1. System name + one-line pitch

**Moderated Group Chat.** A user starts a group chat session with a topic and a set of assistant personas. An Orchestrator manages the turn order between a Researcher and a Critic, checks each reply before it enters the shared transcript, and ends the session when the group reaches consensus or the turn cap is hit.

## 2. What this blueprint demonstrates

The **moderation-turn-taking** coordination pattern: an orchestrator holds the turn order across multiple assistant participants, gives each participant exactly one turn per round, and after every turn decides whether the session should continue, conclude, or be escalated. Up to twenty turns run before the orchestrator must conclude. The **governance pattern** pairs an output guardrail on every assistant response — so no assistant can inject off-topic content or violate persona constraints before the turn is committed to the shared transcript — with an automated evaluator that fires on the stop decision and scores the concluded session. Every component is a first-party Akka primitive in one buildable folder; the assistant personas, the chat history, and the operator halt switch are all modeled inside the same service.

## 3. User-facing flows

1. **Start a chat session.** The user fills in a topic and optionally the assistant persona names in the App UI tab and clicks Start. The service returns a `sessionId` and the row appears in `CHATTING` state.
2. **Watch the turns.** The live list streams each turn as it lands — turn number, assistant name, message text, and a relevance flag — so the user sees the conversation unfold.
3. **See the conclusion.** When the Orchestrator decides, the row moves to `CONCLUDED` with a `terminationReason` of `CONSENSUS` (a summary of agreed points) or `MAX_TURNS_REACHED` or `ESCALATED`, and a quality score appears beside it.
4. **Let it run on its own.** Without any input, the request simulator drips a canned topic every thirty seconds, so sessions keep arriving and concluding.
5. **Halt the system.** An operator can flip a halt switch; queued requests stop starting new sessions until the operator resumes.

These flows are the acceptance journeys in [`reference/user-journeys.md`](reference/user-journeys.md).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearcherAssistant` | AutonomousAgent | Produces the researcher's contribution for the current turn; returns a typed `ChatTurn` | `ChatSessionWorkflow` | `ChatSessionEntity` (via workflow) |
| `CriticAssistant` | AutonomousAgent | Produces the critic's evaluation or counterpoint for the current turn; returns a typed `ChatTurn` | `ChatSessionWorkflow` | `ChatSessionEntity` (via workflow) |
| `OrchestratorAgent` | AutonomousAgent | Reads the full transcript each round and returns a typed `OrchestratorDecision` of continue / conclude / escalate, with a summary on conclusion | `ChatSessionWorkflow` | `ChatSessionEntity` (via workflow) |
| `ChatTasks` | task definitions | Declares the `Task<ChatTurn>` and `Task<OrchestratorDecision>` constants the three agents accept | — | the three agents |
| `ChatSessionWorkflow` | Workflow | The Orchestrator's turn-taking loop: `openStep` → `researcherTurnStep` → `criticTurnStep` → `orchestrateStep`, repeating up to twenty turns, then `concludeStep` | `SessionRequestConsumer` | the three agents, `ChatSessionEntity` |
| `ChatSessionEntity` | EventSourcedEntity | Durable per-session state: every turn, the status, the concluded reason and summary, the quality score | `ChatSessionWorkflow`, `SessionQualityEvaluator`, `StalledSessionMonitor` | `SessionsView` |
| `SessionRequestQueue` | EventSourcedEntity | Records each incoming chat session request as an event | `RequestSimulator`, `ChatEndpoint` | `SessionRequestConsumer` |
| `SystemControl` | KeyValueEntity | Holds the operator halt flag | `ChatEndpoint` | `SessionRequestConsumer` |
| `SessionsView` | View | Row type `ChatSession`; one query returning all sessions for the UI list and SSE stream | `ChatSessionEntity` events | `ChatEndpoint`, `StalledSessionMonitor` |
| `SessionRequestConsumer` | Consumer | On each queued request, checks the halt flag and starts a `ChatSessionWorkflow` with a fresh id | `SessionRequestQueue` events | `ChatSessionWorkflow` |
| `SessionQualityEvaluator` | Consumer | On `SessionConcluded`, scores the session and records it on the entity | `ChatSessionEntity` events | `ChatSessionEntity` |
| `RequestSimulator` | TimedAction | Every thirty seconds, drips the next canned topic from a JSONL file into the queue | scheduled tick | `SessionRequestQueue` |
| `StalledSessionMonitor` | TimedAction | Every thirty seconds, marks sessions still running after three minutes as `ESCALATED` | scheduled tick, `SessionsView` | `ChatSessionEntity` |
| `ChatEndpoint` | HttpEndpoint | The `/api/*` surface: start, list, single, SSE, halt/resume, and the three metadata endpoints | browser, simulator | `SessionRequestQueue`, `SessionsView`, `SystemControl` |
| `AppEndpoint` | HttpEndpoint | Serves `/` (redirect) and `/app/*` (static UI) | browser | static-resources |
| `Bootstrap` | service-setup | On startup, schedules the two TimedActions | runtime | the TimedActions |

Names are authoritative — `/akka:implement` uses them verbatim.

## 5. Data model

Full field-by-field detail is in [`reference/data-model.md`](reference/data-model.md). Authoritative summary:

`ChatSession` is the event-sourced state **and** the View row. Every field that is null until a later event fires is `Optional<T>` (Lesson 6).

```java
public record ChatSession(
  String id,
  Optional<String>    topic,
  ChatSessionStatus   status,
  int                 currentTurn,
  List<TurnLine>      turns,              // append-only history, never null (empty list initially)
  Optional<String>    terminationReason,  // "CONSENSUS" | "MAX_TURNS_REACHED" | "ESCALATED" once concluded
  Optional<String>    conclusionSummary,
  Optional<Instant>   startedAt,
  Optional<Instant>   concludedAt,
  Optional<Instant>   escalatedAt,
  Optional<Double>    qualityScore,       // set by SessionQualityEvaluator
  Optional<String>    qualityNotes
) {
  public static ChatSession initial(String id) { /* status CREATED, turn 0, empty turns, all Optional.empty() */ }
  public ChatSession applyEvent(SessionEvent e) { /* per-variant switch */ }
}
```

`TurnLine(int turn, String assistant, String message, boolean flagged, Instant at)` — `assistant` is `"RESEARCHER"` or `"CRITIC"`.

`ChatSessionStatus` enum: `CREATED · CHATTING · CONCLUDED · ESCALATED`. The termination-reason distinction is carried by the `terminationReason` string, not the enum, so no view query filters on the enum.

`SessionEvent` (sealed, six variants): `SessionStarted`, `TurnRecorded`, `RoundAdvanced`, `SessionConcluded`, `SessionEscalated`, `QualityEvaluated`.

Agent result records: `ChatTurn(String message, String rationale, boolean flagged)` and `OrchestratorDecision(String verdict, String summary, String reasoning)` where `verdict` is `CONTINUE` / `CONCLUDE` / `ESCALATE`.

## 6. API contract

Every endpoint, payload schema, and the SSE event format are in [`reference/api-contract.md`](reference/api-contract.md). Top-level surface (all JSON, ACL open for local-dev):

```
POST /api/sessions                     -> { sessionId }
GET  /api/sessions ?status=...         -> { sessions: [ChatSession, ...] }   (status filtered client-side)
GET  /api/sessions/{id}                -> ChatSession
GET  /api/sessions/sse                 -> Server-Sent Events of ChatSession
POST /api/system/halt                  -> { halted: true }
POST /api/system/resume                -> { halted: false }
GET  /api/system/status                -> { halted: bool }

GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown

GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

The UI is a single self-contained `src/main/resources/static-resources/index.html` (Lesson 17): inline CSS and JS, runtime CDN imports for markdown and YAML rendering are acceptable, no `ui/` folder and no build step. Browser title: `<title>Akka Sample: Moderated Group Chat</title>`. Full description in [`reference/ui-mockup.md`](reference/ui-mockup.md). Five tabs:

1. **Overview** — eyebrow "Overview" + headline (sample type), no subtitle, then the four cards Try it / How it works / Components / API contract.
2. **Architecture** — the four mermaid diagrams (component graph, sequence, state machine, entity model) with the Lesson 24 CSS overrides, plus a click-to-expand component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style; values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted and italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style; the label column carries the control id plus a colored mechanism pill.
5. **App UI** — the live interaction: a Start form (topic), the SSE-streamed list of sessions with their per-turn messages expanding inline, the concluded outcome and quality score, and the operator halt / resume control.

Tab switching matches by `data-tab` → `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Mermaid state-label color and edge-label overflow follow Lesson 24.

## 8. Governance

The controls the generated system must wire are in [`eval-matrix.yaml`](eval-matrix.yaml); the deployer risk posture is in [`risk-survey.yaml`](risk-survey.yaml). One sentence per mechanism:

- **G1 — output guardrail (`guardrail` · before-agent-response).** A before-agent-response guardrail on `ResearcherAssistant`, `CriticAssistant`, and `OrchestratorAgent` rejects any turn that is flagged as off-topic or violates persona constraints before it is committed to the shared transcript, and rejects any `OrchestratorDecision` whose verdict is `CONCLUDE` without a non-empty summary.
- **HT1 — automatic safety halt (`halt` · automatic-safety-halt).** The twenty-turn cap enforced in `ChatSessionWorkflow` bounds runaway sessions; the operator halt switch on `SystemControl` allows an operator to stop new sessions from starting at any time.

G1 and HT1 are the pattern-specific controls drawn from the corpus entry. A1 and HT1 also include the standard operational controls every deployable governed service carries.

## 9. Agent prompts

One file per agent under `prompts/`, pasted verbatim into the agent's instructions:

- [`prompts/researcher-assistant.md`](prompts/researcher-assistant.md) — the researcher's stance: contribute factual content, cite reasoning, stay on topic.
- [`prompts/critic-assistant.md`](prompts/critic-assistant.md) — the critic's stance: challenge claims, surface gaps, never be dismissive.
- [`prompts/orchestrator-agent.md`](prompts/orchestrator-agent.md) — the moderator's adjudication rule: call consensus when the assistants converge, escalate on harmful content, conclude at the turn cap.

## 10. Acceptance

The full journeys are in [`reference/user-journeys.md`](reference/user-journeys.md). The system generated correctly when these pass:

1. **Start and observe.** `POST /api/sessions` with `{ topic }` returns a `sessionId`; within seconds the session is `CHATTING` and the Researcher's turn-1 message is present in `turns`.
2. **Consensus conclusion.** When the topic is well-scoped, the session reaches `CONCLUDED` with `terminationReason = CONSENSUS` and a non-empty `conclusionSummary`, at or before turn twenty.
3. **Max-turns conclusion.** When the topic is deliberately irresolvable, the session reaches `CONCLUDED` with `terminationReason = MAX_TURNS_REACHED` no later than turn twenty.
4. **Session scored.** Every `CONCLUDED` session gets a non-empty `qualityScore` from the evaluator within seconds of concluding.

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named moderated-group-chat demonstrating the
moderation-turn-taking x general cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
moderation-turn-taking-general-moderated-group-chat. Java package
io.akka.samples.skassistantsgroupchat. Akka 3.6.0. HTTP port 9693.

Components to wire (exactly):
- 3 AutonomousAgents:
  - ResearcherAssistant: given topic, turn number, and the previous turn's
    message, returns a typed ChatTurn{message, rationale, flagged}.
    definition() declares
    capability(TaskAcceptance.of(RESEARCHER_TURN).maxIterationsPerTask(3)).
  - CriticAssistant: given topic, turn number, and the researcher's latest
    message, returns a typed ChatTurn{message, rationale, flagged}. Same
    capability pattern with CRITIC_TURN.
  - OrchestratorAgent: given the full turn history, the turn number, and the
    max-turns cap, returns a typed OrchestratorDecision{verdict, summary,
    reasoning}. Capability over ORCHESTRATE.
  Each agent class extends akka.javasdk.agent.autonomous.AutonomousAgent and
  has a single definition() method — never silently downgraded to Agent.
- ChatTasks.java declaring three Task constants: RESEARCHER_TURN
  (resultConformsTo ChatTurn.class), CRITIC_TURN (ChatTurn.class), ORCHESTRATE
  (OrchestratorDecision.class), each Task.name(...).description(...)
  .resultConformsTo(...).
- 1 Workflow ChatSessionWorkflow with steps openStep -> researcherTurnStep ->
  criticTurnStep -> orchestrateStep -> (loop or concludeStep). openStep writes
  SessionStarted and goes to researcherTurnStep. researcherTurnStep calls
  forAutonomousAgent(ResearcherAssistant.class, "researcher-"+id)
  .runSingleTask(RESEARCHER_TURN.instructions(...)) then
  forTask(taskId).result(RESEARCHER_TURN); records the turn via
  ChatSessionEntity.recordTurn; transitions to criticTurnStep. criticTurnStep
  is symmetric with CriticAssistant, then transitions to orchestrateStep.
  orchestrateStep calls OrchestratorAgent; on verdict CONCLUDE transitions to
  concludeStep(CONSENSUS); on ESCALATE transitions to
  concludeStep(ESCALATED); on CONTINUE increments the round via
  ChatSessionEntity.advanceRound and, if the new turn exceeds 20, transitions
  to concludeStep(MAX_TURNS_REACHED), else back to researcherTurnStep.
  concludeStep writes SessionConcluded with terminationReason, summary,
  turnsUsed and ends. Override settings() with stepTimeout(60s) on
  researcherTurnStep, criticTurnStep, orchestrateStep, and concludeStep, and a
  defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity ChatSessionEntity holding the ChatSession record (id,
  Optional topic, ChatSessionStatus enum, int currentTurn, List<TurnLine>
  turns, Optional terminationReason, Optional conclusionSummary, Optional
  startedAt, Optional concludedAt, Optional escalatedAt, Optional qualityScore,
  Optional qualityNotes). Events: SessionStarted, TurnRecorded, RoundAdvanced,
  SessionConcluded, SessionEscalated, QualityEvaluated. Commands: start,
  recordTurn, advanceRound, conclude, markEscalated, recordQuality,
  getSession. emptyState() returns ChatSession.initial("") with no
  commandContext() reference (Lesson 3).
- 1 EventSourcedEntity SessionRequestQueue, single instance "default", one
  command enqueueRequest(topic) emitting SessionRequestQueued.
- 1 KeyValueEntity SystemControl, single instance "default", holding a
  boolean halted; commands halt, resume, isHalted.
- 1 View SessionsView with row type ChatSession, table updater consuming
  ChatSessionEntity events. ONE query: getAllSessions SELECT * AS sessions FROM
  sessions_view. NO WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — callers filter client-side. Provide a streamAllSessions variant
  for the SSE endpoint.
- 1 Consumer SessionRequestConsumer subscribed to SessionRequestQueue events;
  on each event, reads SystemControl.isHalted; if not halted, starts a
  ChatSessionWorkflow with a fresh UUID.
- 1 Consumer SessionQualityEvaluator subscribed to ChatSessionEntity events;
  on SessionConcluded, computes a quality score (topic coherence, turn depth,
  termination reason weight) and calls ChatSessionEntity.recordQuality. Ignore
  other event variants.
- 2 TimedActions: RequestSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/session-requests.jsonl and calls
  SessionRequestQueue.enqueueRequest); StalledSessionMonitor (every 30s,
  queries SessionsView.getAllSessions, filters status CHATTING with startedAt
  older than 3 minutes, calls ChatSessionEntity.markEscalated).
- 2 HttpEndpoints: ChatEndpoint at /api with sessions (POST start, GET list
  filtered client-side from getAllSessions, GET single, GET sse), system
  halt/resume/status backed by SystemControl, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.
- 1 service-setup Bootstrap scheduling RequestSimulator and StalledSessionMonitor
  on startup; fails fast (Lesson 25 step 4) with a clear message naming the
  configured key reference if the model provider key does not resolve, never
  echoing key material.

Companion files:
- ChatTurn(String message, String rationale, boolean flagged),
  OrchestratorDecision(String verdict, String summary, String reasoning),
  TurnLine(int turn, String assistant, String message, boolean flagged,
  Instant at).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9693 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking
  (Lesson 8).
- src/main/resources/sample-events/session-requests.jsonl with 8 canned
  topics; mix topics that converge (consensus) and topics that exhaust turns
  (max-turns-reached).
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the project-root files so the endpoint serves them from classpath).
- eval-matrix.yaml at the project root with controls G1, HT1, A1 and a
  matching simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling sector, decisions, data
  types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the five tabs, an ASCII architecture
  diagram, project layout, API contract, license. NO governance-mechanisms
  section, NO configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — one self-contained HTML
  file (Lesson 17), inline CSS + JS, runtime CDN imports for markdown and YAML
  acceptable. Five tabs (Overview, Architecture, Risk Survey, Eval Matrix, App
  UI). Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid). Include the Lesson 24 mermaid CSS overrides and
  theme variables; switch tabs by data-tab / data-panel attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent schema-valid
  outputs (ResearcherAssistant -> ChatTurn, CriticAssistant -> ChatTurn,
  OrchestratorAgent -> OrchestratorDecision); see
  src/main/resources/mock-responses/{researcher-assistant,
  critic-assistant,orchestrator-agent}.json each with 4-6 entries. Sets
  model-provider = mock. The mock Researcher and Critic must converge within
  twenty turns on the consensus topics and exhaust turns on the max-turns
  topics so the journeys still pass deterministically.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.

Mock LLM provider per-agent output schemas (option (a)):
- researcher-assistant.json: entries shaped ChatTurn{message with factual
  content advancing the topic, rationale, flagged false}.
- critic-assistant.json: entries shaped ChatTurn{message challenging the
  researcher's prior point, rationale, flagged false}. Later entries may set
  flagged false and signal convergence through the message text.
- orchestrator-agent.json: entries shaped OrchestratorDecision{verdict CONTINUE
  for early turns, CONCLUDE with a non-empty summary once both assistants have
  converged, MAX_TURNS_REACHED when turn exceeds 20, reasoning}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent is never silently downgraded to Agent; the
  extends clause matches this spec verbatim.
- Lesson 4: every workflow step that calls an agent overrides stepTimeout to
  60s; the 5s default would time out on LLM calls.
- Lesson 6: every nullable lifecycle field on the ChatSession row record is
  Optional<T>; the turns list is never null (empty list initially).
- Lesson 7: AutonomousAgent requires the companion ChatTasks.java; the
  agent classes reference its Task constants.
- Lesson 8: verify model names against the provider's current lineup before
  writing model-name.
- Lesson 9: the run command is "/akka:build" everywhere user-facing, never
  "mvn akka:run".
- Lesson 10: port 9693 is declared explicitly in application.conf.
- Lesson 11: no source.platform or any corpus-internal provenance appears in
  any user-facing surface.
- Lesson 12: the UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration is described as "Runs out of the box", never a tier
  code, never "deferred".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: the index.html includes the mermaid state-label color overrides,
  edge-label foreignObject overflow:visible, and transitionLabelColor #cccccc.
- Lesson 25: the five-option key sourcing above; never a key value on disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  by NodeList index; removed tabs are deleted from the DOM, not display:none.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6 and [`PLAN.md`](PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9693`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step runs longer than thirty seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key reference (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
