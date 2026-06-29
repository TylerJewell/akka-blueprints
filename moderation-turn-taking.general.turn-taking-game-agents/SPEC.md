# Sample Specification — `turn-taking-game-agents`

This document is the **natural-language input for `/akka:specify`**. The user runs `/akka:specify @SPEC.md` from inside this folder; the `@SPEC.md` token inlines the whole file as the skill's argument. Sections 1–11 together are the spec; Section 12 tells Claude how to continue after scaffolding.

---

## 1. System name + one-line pitch

**AgentChess.** A user starts a chess game in which an AI player faces an opponent — a human at the keyboard or a random mover. A referee gives each side one turn at a time, checks every proposed move for legality against the current board before applying it, and ends the game on checkmate, stalemate, or draw.

## 2. What this blueprint demonstrates

The **moderation-turn-taking** coordination pattern: a referee (the `GameWorkflow`) holds the turn order between two opposed sides (White and Black), gives exactly one side the move at a time, and after a move is applied decides whether the position is terminal or play continues with the turn passing to the other side. The **governance pattern** is a before-tool-call guardrail: the proposed move — whether it came from the AI player, the random opponent, or a human — is validated against the current board by the move validator before the apply-move tool is allowed to mutate the board state, so no illegal move can ever reach the board. Every component is a first-party Akka primitive in one buildable folder; the board, the chess rules, the opponent, the request stream, and the operator halt switch are all modeled inside the same service.

## 3. User-facing flows

1. **Start a game.** The user picks the AI side (White or Black) and the opponent type (human or random) in the App UI tab and clicks Start. The service returns a `gameId` and the row appears in `IN_PROGRESS` state with the opening position on the board.
2. **Watch the moves.** The live list streams each applied move as it lands — ply number, side, the move in standard notation, and the AI player's one-line rationale — so the user sees the game progress move by move on a rendered board.
3. **Take a turn (human opponent).** When it is the human side's move, the row shows `AWAITING_HUMAN`; the user submits a move (from-square and to-square). A legal move is applied; an illegal one is rejected with a reason and the user submits again.
4. **See the result.** When a side is checkmated, or the position is a stalemate or a draw, the row moves to `CONCLUDED` with a result (`WHITE_WINS`, `BLACK_WINS`, or `DRAW`) and a reason.
5. **Let it run on its own.** Without any input, the request simulator drips a canned game (AI versus random) every thirty seconds, so games keep arriving and concluding.
6. **Halt the system.** An operator can flip a halt switch; queued requests stop starting new games until the operator resumes.

These flows are the acceptance journeys in [`reference/user-journeys.md`](reference/user-journeys.md).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChessPlayerAgent` | AutonomousAgent | Proposes the AI side's move given the board position; returns a typed `ProposedMove` | `GameWorkflow` | `GameEntity` (via workflow) |
| `ChessTasks` | task definitions | Declares the `Task<ProposedMove>` constant the agent accepts | — | `ChessPlayerAgent` |
| `MoveValidator` | domain service | Validates a proposed move against the current board, returns the resulting position and terminal status, and enumerates legal moves for the random opponent and the mock provider — the before-tool-call guardrail | `GameWorkflow` | `GameEntity` (via workflow) |
| `GameWorkflow` | Workflow | The referee's turn-taking loop: `openStep` → `turnStep` → `validateAndApplyStep` → `checkEndStep`, alternating sides until a terminal position, then `concludeStep` | `GameRequestConsumer` | `ChessPlayerAgent`, `MoveValidator`, `GameEntity` |
| `GameEntity` | EventSourcedEntity | Durable per-game state: every applied move, the board position, the status, the result | `GameWorkflow`, `StalledGameMonitor` | `GamesView` |
| `InboundRequestQueue` | EventSourcedEntity | Records each incoming new-game request as an event | `RequestSimulator`, `ChessEndpoint` | `GameRequestConsumer` |
| `SystemControl` | KeyValueEntity | Holds the operator halt flag | `ChessEndpoint` | `GameRequestConsumer` |
| `GamesView` | View | Row type `Game`; one query returning all games for the UI list and SSE stream | `GameEntity` events | `ChessEndpoint`, `StalledGameMonitor` |
| `GameRequestConsumer` | Consumer | On each queued request, checks the halt flag and starts a `GameWorkflow` with a fresh id | `InboundRequestQueue` events | `GameWorkflow` |
| `RequestSimulator` | TimedAction | Every thirty seconds, drips the next canned game from a JSONL file into the queue | scheduled tick | `InboundRequestQueue` |
| `StalledGameMonitor` | TimedAction | Every thirty seconds, marks games still awaiting a human move after two minutes as `ESCALATED` | scheduled tick, `GamesView` | `GameEntity` |
| `ChessEndpoint` | HttpEndpoint | The `/api/*` surface: start, list, single, SSE, submit-human-move, halt/resume, and the three metadata endpoints | browser, simulator | `InboundRequestQueue`, `GamesView`, `GameEntity`, `SystemControl` |
| `AppEndpoint` | HttpEndpoint | Serves `/` (redirect) and `/app/*` (static UI) | browser | static-resources |
| `Bootstrap` | service-setup | On startup, schedules the two TimedActions | runtime | the TimedActions |

Names are authoritative — `/akka:implement` uses them verbatim.

## 5. Data model

Full field-by-field detail is in [`reference/data-model.md`](reference/data-model.md). Authoritative summary:

`Game` is the event-sourced state **and** the View row. Every field that is null until a later event fires is `Optional<T>` (Lesson 6).

```java
public record Game(
  String id,
  Optional<String>  aiSide,        // "WHITE" | "BLACK"
  Optional<String>  opponentType,  // "HUMAN" | "RANDOM"
  GameStatus        status,
  Optional<String>  boardFen,      // current position in FEN
  Optional<String>  sideToMove,    // "WHITE" | "BLACK"
  int               ply,           // half-moves applied, 0 at start
  List<MoveLine>    moves,         // append-only history, never null (empty list initially)
  Optional<String>  pendingHumanError, // last rejected human-move reason, cleared on a legal move
  Optional<String>  result,        // "WHITE_WINS" | "BLACK_WINS" | "DRAW" once concluded
  Optional<String>  resultReason,  // "checkmate" | "stalemate" | "draw" | "forfeit"
  Optional<Instant> startedAt,
  Optional<Instant> concludedAt,
  Optional<Instant> escalatedAt,
  Optional<Instant> lastMoveAt
) {
  public static Game initial(String id) { /* status CREATED, ply 0, empty moves, all Optional.empty() */ }
  public Game applyEvent(GameEvent e) { /* per-variant switch */ }
}
```

`MoveLine(int ply, String side, String from, String to, String promotion, String san, String rationale, Instant at)` — `side` is `"WHITE"` or `"BLACK"`; `from`/`to` are squares like `"e2"`/`"e4"`; `promotion` is `""` unless a pawn promotes.

`GameStatus` enum: `CREATED · IN_PROGRESS · AWAITING_HUMAN · CONCLUDED · ESCALATED`. The win-versus-draw split is carried by the `result` string, not the enum, so no view query filters on the enum (Lesson 2).

`GameEvent` (sealed, six variants): `GameStarted`, `MoveApplied`, `HumanMoveRequested`, `IllegalMoveRejected`, `GameConcluded`, `GameEscalated`.

Agent result record: `ProposedMove(String from, String to, String promotion, String rationale)`.

`MoveValidator` returns `Validation(boolean legal, String reason, String resultingFen, String terminal)` where `terminal` is `"NONE" | "CHECKMATE" | "STALEMATE" | "DRAW"`, and a `List<String> legalMoves(String fen)` for the random opponent and mock provider.

## 6. API contract

Every endpoint, payload schema, and the SSE event format are in [`reference/api-contract.md`](reference/api-contract.md). Top-level surface (all JSON, ACL open for local-dev):

```
POST /api/games                        -> { gameId }
GET  /api/games ?status=...            -> { games: [Game, ...] }   (status filtered client-side)
GET  /api/games/{id}                   -> Game
GET  /api/games/sse                    -> Server-Sent Events of Game
POST /api/games/{id}/move              -> 200 | 404 | 409 (illegal -> rejected, reason returned)
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

The UI is a single self-contained `src/main/resources/static-resources/index.html` (Lesson 17): inline CSS and JS, runtime CDN imports for markdown and YAML rendering are acceptable, no `ui/` folder and no build step. Browser title: `<title>Akka Sample: AgentChess</title>`. Full description in [`reference/ui-mockup.md`](reference/ui-mockup.md). Five tabs:

1. **Overview** — eyebrow "Overview" + headline (sample type), no subtitle, then the four cards Try it / How it works / Components / API contract.
2. **Architecture** — the four mermaid diagrams (component graph, sequence, state machine, entity model) with the Lesson 24 CSS overrides, plus a click-to-expand component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style; values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted and italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style; the label column carries the control id plus a colored mechanism pill.
5. **App UI** — the live interaction: a Start form (AI side, opponent type), the SSE-streamed list of games with their move history expanding inline beside a rendered board, a move-submit control when the game awaits a human move, the concluded result and reason, and the operator halt / resume control.

Tab switching matches by `data-tab` → `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Mermaid state-label color and edge-label overflow follow Lesson 24.

## 8. Governance

The controls the generated system must wire are in [`eval-matrix.yaml`](eval-matrix.yaml); the deployer risk posture is in [`risk-survey.yaml`](risk-survey.yaml). One sentence per mechanism:

- **G1 — move-legality guardrail (`guardrail` · before-tool-call).** Before the apply-move tool mutates the board, `MoveValidator` checks the proposed move against the current position; an illegal move is rejected and never applied — an illegal AI move fails the turn step and the agent is re-prompted, an illegal human move returns a reason for the user to correct.
- **A1 — test gate (`ci-gate` · test-gate).** The turn-taking loop and the move-legality rules must pass an integration test before any deploy.
- **HT1 — operator halt (`halt` · operator-regulator-stop).** `SystemControl` holds a halt flag the endpoint and the request consumer check before starting new games.

G1 is the pattern-specific control drawn from the corpus entry; A1 and HT1 are the standard operational controls every deployable governed service carries.

## 9. Agent prompts

One file per agent under `prompts/`, pasted verbatim into the agent's instructions:

- [`prompts/chess-player-agent.md`](prompts/chess-player-agent.md) — the AI player's stance: given the board position and the side to move, propose one legal move and a one-line rationale, never narrate the whole game.

## 10. Acceptance

The full journeys are in [`reference/user-journeys.md`](reference/user-journeys.md). The system generated correctly when these pass:

1. **Start and observe.** `POST /api/games` with `{ aiSide, opponentType }` returns a `gameId`; within seconds the game is `IN_PROGRESS` and the first applied move is present in `moves`, legal for the side that moved.
2. **Legality always holds.** Across a full game, every `MoveLine` is a legal move for its side; no illegal move is ever applied, and any `IllegalMoveRejected` event carries a reason and leaves the board unchanged.
3. **Games terminate.** A game reaches `CONCLUDED` with a `result` of `WHITE_WINS`, `BLACK_WINS`, or `DRAW` and a `resultReason` of `checkmate`, `stalemate`, or `draw`.
4. **Human move validated.** With a human opponent, submitting an illegal move via `POST /api/games/{id}/move` returns a rejection with a reason and does not change the board; submitting a legal move applies it and passes the turn to the AI side.

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named turn-taking-game-agents demonstrating the
moderation-turn-taking x general cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
turn-taking-game-agents. Java package io.akka.samples.turntakinggameagents.
Akka 3.6.0. HTTP port 9348.

Components to wire (exactly):
- 1 AutonomousAgent ChessPlayerAgent: given the board FEN, the side to move,
  and the list of legal moves, returns a typed ProposedMove{from, to,
  promotion, rationale}. definition() declares
  capability(TaskAcceptance.of(PROPOSE_MOVE).maxIterationsPerTask(3)). The
  class extends akka.javasdk.agent.autonomous.AutonomousAgent with a single
  definition() method — never silently downgraded to Agent (Lesson 1).
- ChessTasks.java declaring one Task constant: PROPOSE_MOVE (resultConformsTo
  ProposedMove.class), Task.name(...).description(...).resultConformsTo(...)
  (Lesson 7).
- MoveValidator: a plain domain service (no Akka annotation) holding the chess
  rules in-process — the "chess library" modeled inside the service. Methods:
  Validation validate(String fen, String from, String to, String promotion)
  returning {legal, reason, resultingFen, terminal in NONE/CHECKMATE/STALEMATE/
  DRAW}; List<String> legalMoves(String fen) enumerating legal from-to(-promo)
  moves for the side to move; String initialFen() returning the standard start
  position. Used by GameWorkflow as the before-tool-call guardrail and by the
  random opponent and mock provider to pick legal moves.
- 1 Workflow GameWorkflow with steps openStep -> turnStep ->
  validateAndApplyStep -> checkEndStep -> (loop or concludeStep). openStep
  writes GameStarted with the initial FEN (MoveValidator.initialFen()), aiSide,
  opponentType, sideToMove = WHITE; goes to turnStep. turnStep branches on the
  side to move and its player type:
    * AI side: calls forAutonomousAgent(ChessPlayerAgent.class, "ai-"+id)
      .runSingleTask(PROPOSE_MOVE.instructions(fen, sideToMove,
      legalMoves)) then forTask(taskId).result(PROPOSE_MOVE); goes to
      validateAndApplyStep with the proposed move.
    * RANDOM side: picks a uniformly random move from
      MoveValidator.legalMoves(fen); goes to validateAndApplyStep.
    * HUMAN side: calls GameEntity.requestHumanMove (status AWAITING_HUMAN),
      self-schedules a 3-second resume timer, and on resume reads
      GameEntity.getGame; if a pending human move is present goes to
      validateAndApplyStep with it, else re-schedules (Lesson 4 await-poll
      pattern).
  validateAndApplyStep calls MoveValidator.validate(fen, from, to, promotion)
  — the before-tool-call guardrail. If legal: GameEntity.recordMove(MoveLine +
  resultingFen + terminal) then checkEndStep. If illegal: for an AI/random move
  GameEntity.rejectMove(reason) and loop back to turnStep to re-propose (up to
  maxRetries, then concludeStep with forfeit against the side that could not
  move); for a human move GameEntity.rejectMove(reason) clears the pending move
  and loops back to turnStep (status returns to AWAITING_HUMAN). checkEndStep
  inspects the terminal flag from the last validation: CHECKMATE ->
  concludeStep(winner = side that just moved); STALEMATE or DRAW ->
  concludeStep(DRAW); NONE -> flip sideToMove via GameEntity and loop to
  turnStep. concludeStep writes GameConcluded with result and resultReason and
  ends. Override settings() with stepTimeout(60s) on turnStep and
  validateAndApplyStep, and a defaultStepRecovery(maxRetries(2)
  .failoverTo(error)); the error step concludes the game DRAW so a stuck game
  always reaches a terminal state.
- 1 EventSourcedEntity GameEntity holding the Game record (id, Optional aiSide,
  Optional opponentType, GameStatus enum, Optional boardFen, Optional
  sideToMove, int ply, List<MoveLine> moves, Optional pendingHumanError,
  Optional result, Optional resultReason, Optional startedAt, Optional
  concludedAt, Optional escalatedAt, Optional lastMoveAt). Events: GameStarted,
  MoveApplied, HumanMoveRequested, IllegalMoveRejected, GameConcluded,
  GameEscalated. Commands: start, recordMove, requestHumanMove, submitHumanMove,
  rejectMove, flipSide, conclude, markEscalated, getGame. submitHumanMove stores
  the pending move for the workflow to validate (it does NOT apply it directly —
  the guardrail runs in the workflow). emptyState() returns Game.initial("")
  with no commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue, single instance "default", one
  command enqueueRequest(aiSide, opponentType) emitting GameRequestQueued.
- 1 KeyValueEntity SystemControl, single instance "default", holding a boolean
  halted; commands halt, resume, isHalted.
- 1 View GamesView with row type Game, table updater consuming GameEntity
  events. ONE query: getAllGames SELECT * AS games FROM games_view. NO WHERE
  status filter (Akka cannot auto-index enum columns, Lesson 2) — callers
  filter client-side. Provide a streamAllGames variant for the SSE endpoint.
- 1 Consumer GameRequestConsumer subscribed to InboundRequestQueue events; on
  each event, reads SystemControl.isHalted; if not halted, starts a GameWorkflow
  with a fresh UUID.
- 2 TimedActions: RequestSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/game-requests.jsonl and calls
  InboundRequestQueue.enqueueRequest); StalledGameMonitor (every 30s, queries
  GamesView.getAllGames, filters status AWAITING_HUMAN with lastMoveAt older
  than 2 minutes, calls GameEntity.markEscalated).
- 2 HttpEndpoints: ChessEndpoint at /api with games (POST start, GET list
  filtered client-side from getAllGames, GET single, GET sse, POST
  {id}/move calling GameEntity.submitHumanMove), system halt/resume/status
  backed by SystemControl, and three /api/metadata/* endpoints serving the
  YAML/MD files from src/main/resources/metadata/. AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.
- 1 service-setup Bootstrap scheduling RequestSimulator and StalledGameMonitor
  on startup; fails fast (Lesson 25 step 4) with a clear message naming the
  configured key reference if the model provider key does not resolve, never
  echoing key material.

Companion files:
- ProposedMove(String from, String to, String promotion, String rationale),
  MoveLine(int ply, String side, String from, String to, String promotion,
  String san, String rationale, Instant at), Validation(boolean legal, String
  reason, String resultingFen, String terminal).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9348 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking
  (Lesson 8).
- src/main/resources/sample-events/game-requests.jsonl with 8 canned new-game
  requests; mix aiSide WHITE/BLACK and opponentType RANDOM (the unattended
  simulator path needs no human, so canned games use RANDOM opponents).
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the project-root files so the endpoint serves them from classpath).
- eval-matrix.yaml at the project root with controls G1, A1, HT1 and a matching
  simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling sector, decisions, data
  types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the five tabs, an ASCII architecture
  diagram, project layout, API contract, license. NO governance-mechanisms
  section, NO configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — one self-contained HTML file
  (Lesson 17), inline CSS + JS, runtime CDN imports for markdown and YAML
  acceptable. Five tabs (Overview, Architecture, Risk Survey, Eval Matrix, App
  UI). Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid). Include the Lesson 24 mermaid CSS overrides and
  theme variables; switch tabs by data-tab / data-panel attribute (Lesson 26).
  The App UI renders the board from the current FEN with a simple unicode-piece
  grid.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider for ChessPlayerAgent whose
  output is a ProposedMove chosen at random from MoveValidator.legalMoves(fen)
  for the current position, guaranteeing a legal, game-advancing move with no
  key; see src/main/resources/mock-responses/chess-player-agent.json with 4-6
  fallback entries for positions where dynamic generation is not wired. Sets
  model-provider = mock. Under mock, AI play is legal and games still terminate
  so the journeys pass deterministically.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.

Mock LLM provider per-agent output schema (option (a)):
- chess-player-agent.json: entries shaped ProposedMove{from, to, promotion "",
  rationale}. At run time the mock prefers a move drawn uniformly from
  MoveValidator.legalMoves(fen) so the move is always legal for the live board;
  the static entries are a fallback only.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent is never silently downgraded to Agent; the extends
  clause matches this spec verbatim.
- Lesson 4: every workflow step that calls an agent overrides stepTimeout to
  60s; the 5s default would time out on LLM calls; the human-await step uses
  the self-scheduled resume-timer poll.
- Lesson 6: every nullable lifecycle field on the Game row record is
  Optional<T>; the moves list is never null (empty list initially).
- Lesson 7: AutonomousAgent requires the companion ChessTasks.java; the agent
  class references its Task constant.
- Lesson 8: verify model names against the provider's current lineup before
  writing model-name.
- Lesson 9: the run command is "/akka:build" everywhere user-facing, never
  "mvn akka:run".
- Lesson 10: port 9348 is declared explicitly in application.conf.
- Lesson 11: no source.platform or any corpus-internal provenance appears in
  any user-facing surface.
- Lesson 12: the UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration is described as "Runs out of the box", never a tier
  code, never "deferred".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: the index.html includes the mermaid state-label color overrides,
  edge-label foreignObject overflow:visible, and transitionLabelColor #cccccc.
- Lesson 25: the five-option key sourcing above; never a key value on disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by
  NodeList index; removed tabs are deleted from the DOM, not display:none.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6 and [`PLAN.md`](PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9348`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step runs longer than thirty seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key reference (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
