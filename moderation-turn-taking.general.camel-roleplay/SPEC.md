# Sample Specification — `camel-roleplay`

This document is the **natural-language input for `/akka:specify`**. The user runs `/akka:specify @SPEC.md` from inside this folder; the `@SPEC.md` token inlines the whole file as the skill's argument. Sections 1–11 together are the spec; Section 12 tells Claude how to continue after scaffolding.

---

## 1. System name + one-line pitch

**CAMEL Role-Play Collaboration.** A user submits a task and two role descriptions. A Coordinator assigns the roles — one agent plays the task-solving Assistant, the other plays the goal-setting User — then moderates up to twelve rounds of structured dialogue until the task is solved, an impasse is declared, or the round cap is reached.

## 2. What this blueprint demonstrates

The **moderation-turn-taking** coordination pattern: a moderator (the Coordinator) holds the turn order between two role-playing parties (the AI User and the AI Assistant), gives each party exactly one turn per round, and after both have moved decides whether the round produced a solution, an impasse, or another round. Up to twelve rounds run before the moderator must conclude. The **governance pattern** pairs an output guardrail on every agent response — so no agent can break role, emit harmful content, or fabricate a solution it has not derived — with an automated evaluator that fires on the stop decision and scores the concluded outcome. Every component is a first-party Akka primitive in one buildable folder; the task stream, the role assignments, and the operator halt switch are all modeled inside the same service.

## 3. User-facing flows

1. **Submit a task.** The user fills in a task description, an assistant role label, and a user role label in the App UI tab and clicks Start. The service returns a `collaborationId` and the row appears in `COLLABORATING` state.
2. **Watch the rounds.** The live list streams each dialogue turn as it lands — round number, agent (Assistant or User), message content, and one-line intent — so the user sees the problem-solving process unfold turn by turn.
3. **See the outcome.** When the Coordinator decides, the row moves to `CONCLUDED` with an outcome of `SOLVED` (a solution summary and final answer) or `IMPASSE`, and an evaluator score appears beside it.
4. **Let it run on its own.** Without any input, the request simulator drips a canned task scenario every thirty seconds, so collaborations keep arriving and concluding.
5. **Halt the system.** An operator can flip a halt switch; queued tasks stop starting new collaborations until the operator resumes.

These flows are the acceptance journeys in [`reference/user-journeys.md`](reference/user-journeys.md).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AssistantAgent` | AutonomousAgent | Produces the assistant's dialogue turn for the current round; returns a typed `DialogueTurn` | `CollaborationWorkflow` | `CollaborationEntity` (via workflow) |
| `UserAgent` | AutonomousAgent | Produces the user agent's instruction or question for the current round; returns a typed `DialogueTurn` | `CollaborationWorkflow` | `CollaborationEntity` (via workflow) |
| `CoordinatorAgent` | AutonomousAgent | Reads both latest turns each round and returns a typed `CoordinatorDecision` of continue / solved / impasse, with a solution summary on convergence | `CollaborationWorkflow` | `CollaborationEntity` (via workflow) |
| `CollaborationTasks` | task definitions | Declares the `Task<DialogueTurn>` and `Task<CoordinatorDecision>` constants the three agents accept | — | the three agents |
| `CollaborationWorkflow` | Workflow | The Coordinator's turn-taking loop: `openStep` → `userTurnStep` → `assistantTurnStep` → `coordinateStep`, repeating up to twelve rounds, then `concludeStep` | `TaskRequestConsumer` | the three agents, `CollaborationEntity` |
| `CollaborationEntity` | EventSourcedEntity | Durable per-collaboration state: every dialogue turn, the status, the concluded outcome and solution summary, the evaluator score | `CollaborationWorkflow`, `SolutionEvaluator`, `StalledCollaborationMonitor` | `CollaborationsView` |
| `InboundTaskQueue` | EventSourcedEntity | Records each incoming collaboration task as an event | `TaskSimulator`, `CollaborationEndpoint` | `TaskRequestConsumer` |
| `SystemControl` | KeyValueEntity | Holds the operator halt flag | `CollaborationEndpoint` | `TaskRequestConsumer` |
| `CollaborationsView` | View | Row type `Collaboration`; one query returning all collaborations for the UI list and SSE stream | `CollaborationEntity` events | `CollaborationEndpoint`, `StalledCollaborationMonitor` |
| `TaskRequestConsumer` | Consumer | On each queued task, checks the halt flag and starts a `CollaborationWorkflow` with a fresh id | `InboundTaskQueue` events | `CollaborationWorkflow` |
| `SolutionEvaluator` | Consumer | On `CollaborationConcluded`, scores the outcome and records it on the entity — this is the on-decision evaluator | `CollaborationEntity` events | `CollaborationEntity` |
| `TaskSimulator` | TimedAction | Every thirty seconds, drips the next canned task from a JSONL file into the queue | scheduled tick | `InboundTaskQueue` |
| `StalledCollaborationMonitor` | TimedAction | Every thirty seconds, marks collaborations still running after three minutes as `ESCALATED` | scheduled tick, `CollaborationsView` | `CollaborationEntity` |
| `CollaborationEndpoint` | HttpEndpoint | The `/api/*` surface: start, list, single, SSE, halt/resume, and the three metadata endpoints | browser, simulator | `InboundTaskQueue`, `CollaborationsView`, `SystemControl` |
| `AppEndpoint` | HttpEndpoint | Serves `/` (redirect) and `/app/*` (static UI) | browser | static-resources |
| `Bootstrap` | service-setup | On startup, schedules the two TimedActions | runtime | the TimedActions |

Names are authoritative — `/akka:implement` uses them verbatim.

## 5. Data model

Full field-by-field detail is in [`reference/data-model.md`](reference/data-model.md). Authoritative summary:

`Collaboration` is the event-sourced state **and** the View row. Every field that is null until a later event fires is `Optional<T>` (Lesson 6).

```java
public record Collaboration(
  String id,
  Optional<String>  taskDescription,
  Optional<String>  assistantRole,
  Optional<String>  userRole,
  CollaborationStatus status,
  int               currentRound,
  List<TurnLine>    turns,               // append-only history, never null (empty list initially)
  Optional<String>  latestAssistantMessage,
  Optional<String>  latestUserMessage,
  Optional<String>  outcome,             // "SOLVED" | "IMPASSE" once concluded
  Optional<String>  solutionSummary,
  Optional<String>  finalAnswer,
  Optional<Instant> startedAt,
  Optional<Instant> concludedAt,
  Optional<Instant> escalatedAt,
  Optional<Double>  outcomeScore,        // set by SolutionEvaluator
  Optional<String>  outcomeNotes
) {
  public static Collaboration initial(String id) { /* status CREATED, round 0, empty turns, all Optional.empty() */ }
  public Collaboration applyEvent(CollaborationEvent e) { /* per-variant switch */ }
}
```

`TurnLine(int round, String agent, String message, String intent, Instant at)` — `agent` is `"ASSISTANT"` or `"USER"`.

`CollaborationStatus` enum: `CREATED · COLLABORATING · CONCLUDED · ESCALATED`. The solved-versus-impasse split is carried by the `outcome` string, not the enum, so no view query filters on the enum.

`CollaborationEvent` (sealed, six variants): `CollaborationStarted`, `TurnRecorded`, `RoundAdvanced`, `CollaborationConcluded`, `CollaborationEscalated`, `OutcomeEvaluated`.

Agent result records: `DialogueTurn(String message, String intent, boolean taskComplete)` and `CoordinatorDecision(String verdict, String solutionSummary, String finalAnswer, String reasoning)` where `verdict` is `CONTINUE` / `SOLVED` / `IMPASSE`.

## 6. API contract

Every endpoint, payload schema, and the SSE event format are in [`reference/api-contract.md`](reference/api-contract.md). Top-level surface (all JSON, ACL open for local-dev):

```
POST /api/collaborations                 -> { collaborationId }
GET  /api/collaborations ?status=...     -> { collaborations: [Collaboration, ...] }   (status filtered client-side)
GET  /api/collaborations/{id}            -> Collaboration
GET  /api/collaborations/sse             -> Server-Sent Events of Collaboration
POST /api/system/halt                    -> { halted: true }
POST /api/system/resume                  -> { halted: false }
GET  /api/system/status                  -> { halted: bool }

GET  /api/metadata/eval-matrix           -> text/yaml
GET  /api/metadata/risk-survey           -> text/yaml
GET  /api/metadata/readme                -> text/markdown

GET  /                                   -> 302 /app/index.html
GET  /app/{*path}                        -> static-resources/{*path}
```

## 7. UI

The UI is a single self-contained `src/main/resources/static-resources/index.html` (Lesson 17): inline CSS and JS, runtime CDN imports for markdown and YAML rendering are acceptable, no `ui/` folder and no build step. Browser title: `<title>Akka Sample: CAMEL Role-Play Collaboration</title>`. Full description in [`reference/ui-mockup.md`](reference/ui-mockup.md). Five tabs:

1. **Overview** — eyebrow "Overview" + headline (sample type), no subtitle, then the four cards Try it / How it works / Components / API contract.
2. **Architecture** — the four mermaid diagrams (component graph, sequence, state machine, entity model) with the Lesson 24 CSS overrides, plus a click-to-expand component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style; values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted and italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style; the label column carries the control id plus a colored mechanism pill.
5. **App UI** — the live interaction: a Start form (task description, assistant role, user role), the SSE-streamed list of collaborations with their per-round turns expanding inline, the concluded outcome and evaluator score, and the operator halt / resume control.

Tab switching matches by `data-tab` → `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Mermaid state-label color and edge-label overflow follow Lesson 24.

## 8. Governance

The controls the generated system must wire are in [`eval-matrix.yaml`](eval-matrix.yaml); the deployer risk posture is in [`risk-survey.yaml`](risk-survey.yaml). One sentence per mechanism:

- **G1 — output guardrail (`guardrail` · before-agent-response).** A before-agent-response guardrail on `AssistantAgent`, `UserAgent`, and `CoordinatorAgent` rejects any turn that breaks the assigned role, contains disallowed content categories, or any `CoordinatorDecision` of `SOLVED` that lacks a non-empty `solutionSummary`, because a solution claim without a derivable answer misleads the operator about the collaboration outcome.
- **E1 — outcome evaluator (`eval-event` · on-decision-eval).** `SolutionEvaluator` subscribes to `CollaborationConcluded` and scores the stop decision (solution quality, rounds used, solved-or-impasse), recording the score on the entity for the UI.
- **A1 — test gate (`ci-gate` · test-gate).** The turn-taking loop and the convergence criteria must pass an integration test before any deploy.
- **HT1 — operator halt (`halt` · operator-regulator-stop).** `SystemControl` holds a halt flag the endpoint and the request consumer check before starting new collaborations.

G1 and E1 are the pattern-specific controls drawn from the corpus entry; A1 and HT1 are the standard operational controls every deployable governed service carries.

## 9. Agent prompts

One file per agent under `prompts/`, pasted verbatim into the agent's instructions:

- [`prompts/assistant-agent.md`](prompts/assistant-agent.md) — the assistant's task-solving stance: respond to the user's instructions, make steady progress toward the solution, signal completion only when the answer is fully derived.
- [`prompts/user-agent.md`](prompts/user-agent.md) — the user agent's instruction stance: provide focused sub-tasks and clarifying directions each turn, never answer the task itself.
- [`prompts/coordinator-agent.md`](prompts/coordinator-agent.md) — the coordinator's adjudication rule: declare solved when the assistant has produced a complete, correct answer and the user confirms it, impasse when the parties diverge or the round cap is reached, otherwise continue.

## 10. Acceptance

The full journeys are in [`reference/user-journeys.md`](reference/user-journeys.md). The system generated correctly when these pass:

1. **Start and observe.** `POST /api/collaborations` with `{ taskDescription, assistantRole, userRole }` returns a `collaborationId`; within seconds the collaboration is `COLLABORATING` and the User's round-1 turn is present in `turns`.
2. **Solved task.** When the task is well-posed and solvable in the domain, the collaboration reaches `CONCLUDED` with `outcome = SOLVED`, a non-empty `solutionSummary`, and a non-empty `finalAnswer`, at or before round twelve.
3. **Impasse.** When the task cannot converge (conflicting constraints or mutually exclusive goals), the collaboration reaches `CONCLUDED` with `outcome = IMPASSE` no later than round twelve.
4. **Outcome scored.** Every `CONCLUDED` collaboration gets a non-empty `outcomeScore` from the evaluator within seconds of concluding.

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named camel-roleplay demonstrating the
moderation-turn-taking x general cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
moderation-turn-taking-general-camel-roleplay. Java package
io.akka.samples.camel. Akka 3.6.0. HTTP port 9100.

Components to wire (exactly):
- 3 AutonomousAgents:
  - AssistantAgent: given taskDescription, assistantRole, userLatestMessage,
    and round, returns a typed DialogueTurn{message, intent, taskComplete}.
    definition() declares
    capability(TaskAcceptance.of(ASSISTANT_TURN).maxIterationsPerTask(3)).
  - UserAgent: given taskDescription, userRole, assistantLatestMessage, and
    round, returns a typed DialogueTurn{message, intent, taskComplete}. Same
    capability pattern with USER_TURN.
  - CoordinatorAgent: given both latest messages, the round number, and the
    task description, returns a typed CoordinatorDecision{verdict,
    solutionSummary, finalAnswer, reasoning}. Capability over COORDINATE.
  Each agent class extends akka.javasdk.agent.autonomous.AutonomousAgent and
  has a single definition() method — never silently downgraded to Agent.
- CollaborationTasks.java declaring three Task constants: ASSISTANT_TURN
  (resultConformsTo DialogueTurn.class), USER_TURN (DialogueTurn.class),
  COORDINATE (CoordinatorDecision.class), each Task.name(...)
  .description(...).resultConformsTo(...).
- 1 Workflow CollaborationWorkflow with steps openStep -> userTurnStep ->
  assistantTurnStep -> coordinateStep -> (loop or concludeStep). openStep
  writes CollaborationStarted and goes to userTurnStep. userTurnStep calls
  forAutonomousAgent(UserAgent.class, "user-"+id).runSingleTask(USER_TURN
  .instructions(...)) then forTask(taskId).result(USER_TURN); records the
  turn via CollaborationEntity.recordTurn; transitions to assistantTurnStep.
  assistantTurnStep is symmetric with AssistantAgent, then transitions to
  coordinateStep. coordinateStep calls CoordinatorAgent; on verdict SOLVED
  transitions to concludeStep(SOLVED); on IMPASSE transitions to
  concludeStep(IMPASSE); on CONTINUE increments the round via
  CollaborationEntity.advanceRound and, if the new round exceeds 12,
  transitions to concludeStep(IMPASSE), else back to userTurnStep.
  concludeStep writes CollaborationConcluded with outcome, solutionSummary,
  finalAnswer, roundsUsed and ends. Override settings() with
  stepTimeout(60s) on userTurnStep, assistantTurnStep, coordinateStep, and
  concludeStep, and a defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity CollaborationEntity holding the Collaboration record
  (id, Optional taskDescription, Optional assistantRole, Optional userRole,
  CollaborationStatus enum, int currentRound, List<TurnLine> turns,
  Optional latestAssistantMessage, Optional latestUserMessage, Optional
  outcome, Optional solutionSummary, Optional finalAnswer, Optional
  startedAt, Optional concludedAt, Optional escalatedAt, Optional
  outcomeScore, Optional outcomeNotes). Events: CollaborationStarted,
  TurnRecorded, RoundAdvanced, CollaborationConcluded, CollaborationEscalated,
  OutcomeEvaluated. Commands: start, recordTurn, advanceRound, conclude,
  markEscalated, recordEvaluation, getCollaboration. emptyState() returns
  Collaboration.initial("") with no commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundTaskQueue, single instance "default", one
  command enqueueTask(taskDescription, assistantRole, userRole) emitting
  TaskQueued.
- 1 KeyValueEntity SystemControl, single instance "default", holding a
  boolean halted; commands halt, resume, isHalted.
- 1 View CollaborationsView with row type Collaboration, table updater
  consuming CollaborationEntity events. ONE query: getAllCollaborations
  SELECT * AS collaborations FROM collaborations_view. NO WHERE status
  filter (Akka cannot auto-index enum columns, Lesson 2) — callers filter
  client-side. Provide a streamAllCollaborations variant for the SSE endpoint.
- 1 Consumer TaskRequestConsumer subscribed to InboundTaskQueue events; on
  each event, reads SystemControl.isHalted; if not halted, starts a
  CollaborationWorkflow with a fresh UUID.
- 1 Consumer SolutionEvaluator subscribed to CollaborationEntity events; on
  CollaborationConcluded, computes an outcome score (solution quality proxy,
  rounds used, solved flag) and calls CollaborationEntity.recordEvaluation.
  Ignore other event variants.
- 2 TimedActions: TaskSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/task-requests.jsonl and calls
  InboundTaskQueue.enqueueTask); StalledCollaborationMonitor (every 30s,
  queries CollaborationsView.getAllCollaborations, filters status COLLABORATING
  with startedAt older than 3 minutes, calls
  CollaborationEntity.markEscalated).
- 2 HttpEndpoints: CollaborationEndpoint at /api with collaborations (POST
  start, GET list filtered client-side from getAllCollaborations, GET single,
  GET sse), system halt/resume/status backed by SystemControl, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.
- 1 service-setup Bootstrap scheduling TaskSimulator and
  StalledCollaborationMonitor on startup; fails fast (Lesson 25 step 4)
  with a clear message naming the configured key reference if the model
  provider key does not resolve, never echoing key material.

Companion files:
- DialogueTurn(String message, String intent, boolean taskComplete),
  CoordinatorDecision(String verdict, String solutionSummary, String
  finalAnswer, String reasoning), TurnLine(int round, String agent, String
  message, String intent, Instant at).
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9100 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/task-requests.jsonl with 8 canned tasks;
  mix solvable (well-posed, convergeable) and unsolvable (conflicting
  constraints) cases.
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the project-root files so the endpoint serves them from
  classpath).
- eval-matrix.yaml at the project root with controls G1, E1, A1, HT1 and a
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
  file (Lesson 17), inline CSS + JS, runtime CDN imports for markdown and
  YAML acceptable. Five tabs (Overview, Architecture, Risk Survey, Eval
  Matrix, App UI). Match the governance.html visual style (dark / yellow
  accent / Instrument Sans / dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables; switch tabs by data-tab / data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent schema-valid
  outputs (AssistantAgent -> DialogueTurn, UserAgent -> DialogueTurn,
  CoordinatorAgent -> CoordinatorDecision); see
  src/main/resources/mock-responses/{assistant-agent,user-agent,
  coordinator-agent}.json each with 4-6 entries. Sets model-provider = mock.
  The mock User and Assistant must converge within twelve rounds on the
  solvable tasks and stay stuck on the unsolvable tasks so the journeys still
  pass deterministically.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in
  .akka-build.yaml; /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session
  end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.

Mock LLM provider per-agent output schemas (option (a)):
- assistant-agent.json: entries shaped DialogueTurn{message with incremental
  progress toward the solution, intent, taskComplete true only on the final
  round when the answer is fully derived}.
- user-agent.json: entries shaped DialogueTurn{message providing the next
  sub-task instruction or clarification, intent, taskComplete false until the
  assistant signals done}.
- coordinator-agent.json: entries shaped CoordinatorDecision{verdict CONTINUE
  while the assistant has not yet produced a complete answer and round <= 12,
  SOLVED with solutionSummary and finalAnswer when the assistant signals
  taskComplete and the user confirms, IMPASSE when round exceeds 12 or
  conflicting constraints are detected, reasoning}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent is never silently downgraded to Agent; the
  extends clause matches this spec verbatim.
- Lesson 4: every workflow step that calls an agent overrides stepTimeout to
  60s; the 5s default would time out on LLM calls.
- Lesson 6: every nullable lifecycle field on the Collaboration row record is
  Optional<T>; the turns list is never null (empty list initially).
- Lesson 7: AutonomousAgent requires the companion CollaborationTasks.java;
  the agent classes reference its Task constants.
- Lesson 8: verify model names against the provider's current lineup before
  writing model-name.
- Lesson 9: the run command is "/akka:build" everywhere user-facing, never
  "mvn akka:run".
- Lesson 10: port 9100 is declared explicitly in application.conf.
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

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9100`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step runs longer than thirty seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key reference (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
