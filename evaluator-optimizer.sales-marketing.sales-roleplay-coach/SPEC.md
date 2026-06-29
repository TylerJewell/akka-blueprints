# SPEC — sales-roleplay-coach

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Sales Roleplay Coach.
**One-line pitch:** Submit a deal context; a buyer-simulator agent plays the prospect; a coach agent scores the rep's pitch against a rubric; the two iterate until the coach approves or the session hits its turn ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`BuyerSimulatorAgent`) and a reviewer agent (`CoachAgent`), feeding each coaching note back into the rep's next turn until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every turn's verdict for downstream quality measurement, and an **output guardrail** that gates each rep turn against a deterministic policy check (no competitor brand claims, no fabricated pricing) before the buyer simulator responds.

## 3. User-facing flows

The user opens the App UI tab and submits a practice request (a buyer persona, a product being pitched, a deal stage, and an optional deal amount).

1. The system creates a `Session` record in `PITCHING` and starts a `SessionWorkflow`.
2. `BuyerSimulatorAgent` opens the conversation as the buyer: a first objection or question calibrated to the deal context.
3. The output guardrail vets the rep's first pitch turn for prohibited content (competitor brand fabrications, pricing not in the scenario's price book). Flagged turns are short-circuited back to the rep with a deterministic feedback note; they never reach the buyer simulator or coach.
4. `CoachAgent` scores the turn against a fixed rubric (discovery discipline, objection handling, value articulation, closing signal, prohibited content) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `CoachingNotes` payload (up to four bullets).
5. On `ACCEPT`, the workflow transitions the session to `PASSED` with the accepted turn's text and the coach's rationale.
6. On `REVISE`, the workflow records the turn, the guardrail verdict, the coaching verdict, and the notes on the entity, then calls `BuyerSimulatorAgent` again with the rep's revised pitch. The buyer escalates or softens its response based on how the pitch changes.
7. If the loop reaches `maxTurns` (default 5) without an `ACCEPT`, the halt mechanism activates: the workflow ends with `EXHAUSTED`, the highest-scoring turn is preserved along with every coaching verdict for review, and an `EvalRecorded` event captures the outcome.

A `ScenarioSimulator` (TimedAction) drips a canned scenario every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BuyerSimulatorAgent` | `AutonomousAgent` | Simulates the buyer persona for a deal context; escalates or softens based on the rep's revised pitches. | `SessionWorkflow` | returns `BuyerTurn` to workflow |
| `CoachAgent` | `AutonomousAgent` | Scores the rep's pitch turn against the rubric; returns `ACCEPT` or `REVISE` with notes. | `SessionWorkflow` | returns `CoachVerdict` to workflow |
| `SessionWorkflow` | `Workflow` | Runs the pitch → guardrail → buyer-response → coach-score → revise loop; halts at the ceiling. | `CoachEndpoint`, `ScenarioRequestConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, every rep turn, every buyer response, every coach verdict, and the final outcome. | `SessionWorkflow` | `SessionsView` |
| `ScenarioQueue` | `EventSourcedEntity` | Logs each practice request for replay and audit. | `CoachEndpoint`, `ScenarioSimulator` | `ScenarioRequestConsumer` |
| `SessionsView` | `View` | List-of-sessions read model. | `SessionEntity` events | `CoachEndpoint` |
| `ScenarioRequestConsumer` | `Consumer` | Subscribes to `ScenarioQueue` events; starts a workflow per submission. | `ScenarioQueue` events | `SessionWorkflow` |
| `ScenarioSimulator` | `TimedAction` | Drips a sample scenario every 90 s from `sample-events/sales-scenarios.jsonl`. | scheduler | `ScenarioQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `SessionsView`, records an `EvalRecorded` event for any turn that completed since the last tick. | scheduler | `SessionEntity` |
| `CoachEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SessionsView`, `ScenarioQueue`, `SessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record DealContext(
    String buyerPersona,
    String product,
    DealStage dealStage,
    Optional<Integer> dealAmountUsd,
    String requestedBy
) {}

record RepTurn(String text, Instant submittedAt) {}

record BuyerTurn(String text, BuyerSignal signal, Instant respondedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record CoachingNotes(List<String> bullets, String overallRationale) {}

record CoachVerdict(
    CoachDecision decision,
    CoachingNotes notes,
    int score,
    Instant evaluatedAt
) {}

record Turn(
    int turnNumber,
    RepTurn repTurn,
    GuardrailVerdict guardrail,
    Optional<BuyerTurn> buyerResponse,
    Optional<CoachVerdict> coachVerdict
) {}

record Session(
    String sessionId,
    DealContext dealContext,
    int maxTurns,
    SessionStatus status,
    List<Turn> turns,
    Optional<Integer> passedAtTurnNumber,
    Optional<String> passedTurnText,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { PITCHING, EVALUATING, PASSED, EXHAUSTED }

enum CoachDecision { ACCEPT, REVISE }

enum DealStage { DISCOVERY, DEMO, PROPOSAL, NEGOTIATION, CLOSING }

enum BuyerSignal { INTERESTED, SKEPTICAL, RESISTANT, READY_TO_BUY }
```

### Events (on `SessionEntity`)

`SessionCreated`, `RepTurnRecorded`, `TurnGuardrailVerdictRecorded`, `BuyerResponseRecorded`, `TurnCoachVerdicted`, `SessionPassed`, `SessionExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ buyerPersona, product, dealStage?, dealAmountUsd?, requestedBy? }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=PITCHING|EVALUATING|PASSED|EXHAUSTED`.
- `GET /api/sessions/{id}` — one session (including every turn and every coach verdict).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Sales Roleplay Coach"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a scenario, live list of sessions with status pills, click-to-expand per-turn timeline showing the rep's pitch, the guardrail verdict, the buyer's response, the coach's verdict, and the coaching notes.

Browser title: `<title>Akka Sample: Sales Roleplay Coach</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`after-llm-response` on `BuyerSimulatorAgent`): a deterministic check that the rep's pitch turn does not contain competitor brand fabrications or pricing that contradicts the scenario's price book. Flagged turns short-circuit back to the rep with a structured feedback note (`reasonCode = PROHIBITED_CONTENT`); they never reach the buyer simulator or coach. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every turn's coaching verdict is recorded as an `EvalRecorded` event with `{ turnNumber, decision, score, guardrailFlagged }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-turn timeline and in `/api/sessions/{id}`.

## 9. Agent prompts

- `BuyerSimulatorAgent` → `prompts/buyer-simulator.md`. Simulates a named buyer persona for the configured deal context; escalates or softens based on the rep's pitch quality.
- `CoachAgent` → `prompts/coach.md`. Scores a rep's pitch turn against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with up to four short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a scenario; session progresses `PITCHING` → `EVALUATING` → `PASSED` within the turn ceiling; the App UI shows every turn's pitch, buyer response, and coaching verdict.
2. **J2 — halt at ceiling** — Submit a scenario in test mode (buyer always resistant, coach always REVISE); session progresses through `maxTurns` and lands in `EXHAUSTED` with the highest-scoring turn preserved.
3. **J3 — guardrail block** — Submit a pitch turn containing a competitor brand claim; the guardrail short-circuits with `reasonCode = PROHIBITED_CONTENT`, the rep is prompted to revise, the loop continues.
4. **J4 — eval-event timeline** — The expanded view of any session shows one `EvalRecorded` event per coached turn and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sales-roleplay-coach demonstrating the evaluator-optimizer ×
sales-marketing cell. Runs out of the box (no external services required by default).
Maven group io.akka.samples. Artifact id evaluator-optimizer-sales-marketing-sales-roleplay-coach.
Java package io.akka.samples.salescoach. Akka 3.6.0. HTTP port 9300.

Components to wire (exactly):
- 2 AutonomousAgents:
  * BuyerSimulatorAgent — definition() with
    capability(TaskAcceptance.of(OPEN_SESSION).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(RESPOND_TO_PITCH).maxIterationsPerTask(2)).
    System prompt loaded from prompts/buyer-simulator.md. Returns BuyerTurn{text,
    signal, respondedAt} for both OPEN_SESSION and RESPOND_TO_PITCH. The
    RESPOND_TO_PITCH task takes (dealContext, repTurn, priorBuyerTurns) as inputs.
  * CoachAgent — definition() with
    capability(TaskAcceptance.of(SCORE_TURN).maxIterationsPerTask(2)). System
    prompt from prompts/coach.md. Returns CoachVerdict{decision, notes, score,
    evaluatedAt} where decision is the CoachDecision enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow SessionWorkflow with steps:
    startStep -> openBuyerStep -> repTurnStep -> guardrailStep ->
    [guardrail FAIL? repTurnStep again with structured feedback :
      buyerResponseStep -> coachStep ->
      [decision ACCEPT? passStep : (turnCount < maxTurns ?
         repTurnStep with coaching attached : exhaustStep)]] -> END.
  openBuyerStep calls forAutonomousAgent(BuyerSimulatorAgent.class, sessionId)
    .runSingleTask(OPEN_SESSION). repTurnStep waits for the rep to submit a
    pitch via the API (CoachEndpoint.submitRepTurn). guardrailStep is
    pure-function: checks text for prohibited patterns. buyerResponseStep calls
    forAutonomousAgent(BuyerSimulatorAgent.class, sessionId).runSingleTask(
    RESPOND_TO_PITCH). coachStep calls forAutonomousAgent(CoachAgent.class,
    sessionId).runSingleTask(SCORE_TURN). passStep emits SessionPassed.
    exhaustStep emits SessionExhausted with the highest-scoring turn's text as
    best-of and a structured exhaustionReason. Override settings() with
    stepTimeout(90s) on openBuyerStep, buyerResponseStep, and coachStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep)).
  guardrailStep is a pure-function step (no LLM call): checks repTurn.text()
    for prohibited patterns (competitor brand names from a configurable list,
    pricing tokens not matching the price book). On FAIL, emits
    TurnGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "PROHIBITED_CONTENT", then transitions back to repTurnStep
    with structured feedback CoachingNotes("Turn contains prohibited content;
    remove competitor brand claims and unverified pricing before resubmitting.").

- 1 EventSourcedEntity SessionEntity holding state Session{sessionId,
  dealContext: DealContext, maxTurns, SessionStatus status, List<Turn> turns,
  Optional<Integer> passedAtTurnNumber, Optional<String> passedTurnText,
  Optional<String> exhaustionReason, Instant createdAt, Optional<Instant>
  finishedAt}. SessionStatus enum: PITCHING, EVALUATING, PASSED, EXHAUSTED.
  DealStage enum: DISCOVERY, DEMO, PROPOSAL, NEGOTIATION, CLOSING.
  BuyerSignal enum: INTERESTED, SKEPTICAL, RESISTANT, READY_TO_BUY.
  CoachDecision enum: ACCEPT, REVISE.
  Events: SessionCreated, RepTurnRecorded, TurnGuardrailVerdictRecorded,
  BuyerResponseRecorded, TurnCoachVerdicted, SessionPassed, SessionExhausted,
  EvalRecorded. Commands: createSession, recordRepTurn, recordGuardrail,
  recordBuyerResponse, recordCoachVerdict, passSession, exhaustSession,
  recordEval, getSession. emptyState() returns Session.initial("", "", 5) with
  no commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity ScenarioQueue with command enqueueScenario(buyerPersona,
  product, dealStage, dealAmountUsd, requestedBy) emitting
  ScenarioSubmitted{sessionId, buyerPersona, product, dealStage, dealAmountUsd,
  requestedBy, submittedAt}.

- 1 View SessionsView with row type SessionRow (mirrors Session; the turns list
  is preserved as-is — the list is bounded at maxTurns so size stays
  reasonable). Table updater consumes SessionEntity events. ONE query
  getAllSessions SELECT * AS sessions FROM sessions_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer ScenarioRequestConsumer subscribed to ScenarioQueue events; on
  ScenarioSubmitted starts a SessionWorkflow with the sessionId as the
  workflow id.

- 2 TimedActions:
  * ScenarioSimulator — every 90s, reads next line from
    src/main/resources/sample-events/sales-scenarios.jsonl and calls
    ScenarioQueue.enqueueScenario.
  * EvalSampler — every 30s, queries SessionsView.getAllSessions, finds sessions
    with a coached turn that has not yet been recorded as an EvalRecorded event,
    and calls SessionEntity.recordEval(turnNumber, decision, score,
    guardrailFlagged). Idempotent per (sessionId, turnNumber).

- 2 HttpEndpoints:
  * CoachEndpoint at /api with POST /sessions, GET /sessions, GET /sessions/{id},
    GET /sessions/sse, POST /sessions/{id}/turns (rep submits a pitch turn),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /sessions body is
    {buyerPersona, product, dealStage?, dealAmountUsd?, requestedBy?};
    missing dealStage defaults to DISCOVERY, missing requestedBy defaults to
    "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SalesCoachTasks.java declaring three Task<R> constants: OPEN_SESSION (resultConformsTo
  BuyerTurn), RESPOND_TO_PITCH (BuyerTurn), SCORE_TURN (CoachVerdict).
- Domain records DealContext, RepTurn, BuyerTurn, GuardrailVerdict, CoachingNotes,
  CoachVerdict, Turn, Session; enums SessionStatus, CoachDecision, DealStage,
  BuyerSignal.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9300 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  sales-coach.session.max-turns = 5 and
  sales-coach.session.default-deal-stage = DISCOVERY, overridable by
  env var.
- src/main/resources/sample-events/sales-scenarios.jsonl with 8 canned scenario
  lines, each shaped {"buyerPersona":"...", "product":"...", "dealStage":"DEMO"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = sales-coaching,
  decisions.authority_level = advisory-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/buyer-simulator.md, prompts/coach.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Sales Roleplay Coach",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-turn timeline). Browser title exactly:
  <title>Akka Sample: Sales Roleplay Coach</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: buyer-simulator.json, coach.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    buyer-simulator.json — 6 BuyerTurn entries. Two are OPEN_SESSION openers
      with signal=SKEPTICAL (classic "I've heard this before" opener). Two
      are RESPOND_TO_PITCH entries with signal=RESISTANT (objection to price
      or timeline). One is signal=INTERESTED (rep pitch landed well). One is
      signal=READY_TO_BUY (rep nailed the close).
    coach.json — 6 CoachVerdict entries. Three return decision=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      decision=REVISE with score=2 or 3 and a CoachingNotes payload of up
      to four bullets ("discovery question was skipped", "objection
      acknowledged but not resolved", "value proposition too generic",
      "no closing signal attempted").
- A MockModelProvider.seedFor(sessionId, turnNumber) helper makes the
  selection deterministic per (sessionId, turnNumber) so the same session in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. BuyerSimulatorAgent
  and CoachAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a SalesCoachTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Session row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SalesCoachTasks.java is mandatory; generating BuyerSimulatorAgent
  or CoachAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9300, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box (CRM integration opt-in)"
  — never T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
