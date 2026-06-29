# SPEC — multi-turn-simulator

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Multi-Turn Actor Simulator.
**One-line pitch:** Submit a dialog scenario; a simulator agent plays a user across multi-turn conversation; an evaluator agent scores each response turn by turn and produces a fairness/drift verdict when the session closes.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow drives a turn loop between a generator agent (`ActorAgent`) that impersonates a user persona and a reviewer agent (`EvaluatorAgent`) that scores the target agent's responses. The loop runs until the scenario is exhausted or a turn ceiling is hit. The blueprint also demonstrates three governance mechanisms — a **periodic eval event** that records every session's drift-fairness verdict for downstream quality measurement (`E1`), an **output guardrail** that checks each target-agent response against a deterministic policy rule before the evaluator runs (`G1`), and a **halt** that ends the loop gracefully when the turn ceiling is reached without leaving the session in a degenerate state (`HT1`).

## 3. User-facing flows

The user opens the App UI tab and submits a scenario (a persona label and a goal description).

1. The system creates a `Session` record in `RUNNING` and starts a `SimulationWorkflow`.
2. The ActorAgent generates the first user utterance for the persona+goal.
3. The target-agent proxy forwards the utterance to the configured target endpoint and returns a `TargetResponse`.
4. The output guardrail checks the response against the policy rule (no prompt-injection echoing, no sensitive-pattern disclosure). Violating responses are short-circuited; the evaluator receives a synthetic `POLICY_VIOLATION` turn verdict without calling the model.
5. The EvaluatorAgent scores the response on four dimensions (coherence, policy adherence, persona consistency, bias) and returns a `TurnVerdict` with an overall `score` and a `drift` boolean.
6. The workflow records the turn — the actor utterance, the response, the guardrail verdict, and the evaluator's verdict — on `SessionEntity`, then advances the scenario: if the goal is satisfied or the turn ceiling is reached, the workflow closes with a `SessionVerdict`; otherwise it calls the ActorAgent for the next turn.
7. On close, the EvaluatorAgent aggregates all turn verdicts and produces a `SessionVerdict` with a `driftFlagged` boolean and an overall `sessionScore`. If any turn was flagged for drift or policy violation, `driftFlagged = true` and the session transitions to `FLAGGED`; otherwise `COMPLETED`.

A `ScenarioSimulator` (TimedAction) drips a canned scenario every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ActorAgent` | `AutonomousAgent` | Generates the next user utterance for the active persona+goal; on subsequent turns, takes prior dialog history as input. | `SimulationWorkflow` | returns `ActorUtterance` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a target-agent response turn; on session close, aggregates turn verdicts into `SessionVerdict`. | `SimulationWorkflow` | returns `TurnVerdict` or `SessionVerdict` to workflow |
| `SimulationWorkflow` | `Workflow` | Drives the turn loop; halts at the turn ceiling or on goal completion; routes guardrail failures as synthetic turn verdicts. | `SimulationEndpoint`, `ScenarioConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, every turn record, every verdict, and the final session verdict. | `SimulationWorkflow` | `SessionsView` |
| `ScenarioQueue` | `EventSourcedEntity` | Logs each submitted scenario for replay and audit. | `SimulationEndpoint`, `ScenarioSimulator` | `ScenarioConsumer` |
| `SessionsView` | `View` | List-of-sessions read model. | `SessionEntity` events | `SimulationEndpoint` |
| `ScenarioConsumer` | `Consumer` | Subscribes to `ScenarioQueue` events; starts a `SimulationWorkflow` per submission. | `ScenarioQueue` events | `SimulationWorkflow` |
| `ScenarioSimulator` | `TimedAction` | Drips a canned scenario every 90 s from `sample-events/dialog-scenarios.jsonl`. | scheduler | `ScenarioQueue` |
| `DriftSampler` | `TimedAction` | Every 45 s, scans `SessionsView`, records a `DriftEvalRecorded` event for any completed turn that has not yet been sampled. | scheduler | `SessionEntity` |
| `SimulationEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SessionsView`, `ScenarioQueue`, `SessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Scenario(String scenarioId, String persona, String goal, String submittedBy) {}

record ActorUtterance(String text, int turnNumber, Instant utteredAt) {}

record TargetResponse(String text, int turnNumber, Instant respondedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record TurnScore(int coherence, int policyAdherence, int personaConsistency, int bias) {}

record TurnVerdict(
    TurnOutcome outcome,
    TurnScore scores,
    int overallScore,
    boolean driftFlagged,
    String rationale,
    Instant evaluatedAt
) {}

record TurnRecord(
    int turnNumber,
    ActorUtterance utterance,
    TargetResponse response,
    GuardrailVerdict guardrail,
    Optional<TurnVerdict> verdict
) {}

record SessionVerdict(
    SessionOutcome outcome,
    int sessionScore,
    boolean driftFlagged,
    String summaryRationale,
    Instant closedAt
) {}

record Session(
    String sessionId,
    String persona,
    String goal,
    int maxTurns,
    SessionStatus status,
    List<TurnRecord> turns,
    Optional<SessionVerdict> sessionVerdict,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { RUNNING, COMPLETED, FLAGGED, HALTED }

enum TurnOutcome { PASS, REVISE, POLICY_VIOLATION }

enum SessionOutcome { CLEAN, FLAGGED, HALTED }
```

### Events (on `SessionEntity`)

`SessionCreated`, `TurnUtteranceRecorded`, `TurnResponseRecorded`, `TurnGuardrailVerdictRecorded`, `TurnVerdictRecorded`, `SessionClosed`, `DriftEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ persona, goal, maxTurns?, submittedBy? }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=RUNNING|COMPLETED|FLAGGED|HALTED`.
- `GET /api/sessions/{id}` — one session (including every turn record and every verdict).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Multi-Turn Actor Simulator"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, guardrail = red, halt = red).
- **App UI** — form to submit a scenario, live list of sessions with status pills, click-to-expand per-turn timeline showing each utterance, the guardrail verdict, the evaluator's verdict, and the evaluator's notes.

Browser title: `<title>Akka Sample: Multi-Turn Actor Simulator</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `EvaluatorAgent`): a deterministic check applied to each `TargetResponse` before the evaluator model runs. The check scans for sensitive-pattern disclosure (e.g., system-prompt echoing keywords: `[SYSTEM]`, `<|im_start|>`, `<|endoftext|>`) and prompt-injection artifacts. Violating responses are short-circuited: the guardrail emits `TurnGuardrailVerdictRecorded` with `verdict.passed = false` and `reasonCode = POLICY_VIOLATION`, synthesizes a `TurnVerdict` with `outcome = POLICY_VIOLATION` and `overallScore = 0`, and skips the EvaluatorAgent call for that turn. Enforcement: blocking.
- **E1 — eval-periodic** (`drift-fairness-watch`): each session's turn-level scores and the session-level `driftFlagged` flag are persisted as a `DriftEvalRecorded` event. The `DriftSampler` TimedAction is the canonical writer. The events enable operators to track score distribution over time and detect behavioral drift across sessions without any per-request instrumentation. Enforcement: non-blocking.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxTurns` without a goal-completion signal from the ActorAgent, the workflow ends with `SessionClosed` carrying `outcome = HALTED`. The entity preserves every turn record, every verdict, the best-scoring turn, and a structured halt reason. Enforcement: system-level.

## 9. Agent prompts

- `ActorAgent` → `prompts/actor.md`. Generates the next user utterance for the persona+goal; on subsequent turns, receives the full prior dialog history and produces only the next user message.
- `EvaluatorAgent` → `prompts/evaluator.md`. Scores a single target-agent response turn; on the session-close call, aggregates all `TurnVerdict` objects into a `SessionVerdict`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — clean session** — Submit a scenario; session progresses `RUNNING` → `COMPLETED`; every turn is scored; `driftFlagged = false`; the App UI shows the full dialog with per-turn verdicts.
2. **J2 — halt at turn ceiling** — Submit a scenario with a goal the ActorAgent never signals complete; session exhausts `maxTurns`, transitions to `HALTED`, and preserves every turn for audit.
3. **J3 — guardrail policy violation** — Submit a scenario where the target endpoint echoes `[SYSTEM]`; the guardrail fires on that turn; the evaluator is not called; the turn shows `POLICY_VIOLATION`; the session transitions to `FLAGGED`.
4. **J4 — drift eval timeline** — The expanded view of any session shows one `DriftEvalRecorded` event per sampled turn and a session-level event on close.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-turn-simulator demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-multi-turn-simulator.
Java package io.akka.samples.multiturnactorsimulator. Akka 3.6.0. HTTP port 9769.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ActorAgent — definition() with
    capability(TaskAcceptance.of(INITIATE_TURN).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(CONTINUE_TURN).maxIterationsPerTask(3)).
    System prompt loaded from prompts/actor.md. Returns ActorUtterance{text,
    turnNumber, utteredAt} for both INITIATE_TURN and CONTINUE_TURN. The
    CONTINUE_TURN task takes (persona, goal, dialogHistory: List<TurnRecord>)
    as inputs.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(SCORE_TURN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CLOSE_SESSION).maxIterationsPerTask(2)).
    System prompt from prompts/evaluator.md. SCORE_TURN returns TurnVerdict.
    CLOSE_SESSION takes (turns: List<TurnRecord>) and returns SessionVerdict.

- 1 Workflow SimulationWorkflow with steps:
    startStep -> actorStep -> targetStep -> guardrailStep ->
    [guardrail FAIL? syntheticVerdictStep : scoreStep] ->
    [goalComplete? closeStep : (turnCount < maxTurns ? actorStep again
    with updated history : haltStep)] -> END.
  actorStep calls forAutonomousAgent(ActorAgent.class, sessionId).runSingleTask(
    INITIATE_TURN or CONTINUE_TURN). targetStep forwards the utterance to
    the TargetAgentProxy (a simple HttpClient call to the configured target
    URL; defaults to a stub that returns a canned response). guardrailStep
    is a pure-function step checking for sensitive-pattern keywords.
    scoreStep calls forAutonomousAgent(EvaluatorAgent.class, sessionId)
    .runSingleTask(SCORE_TURN). closeStep calls EvaluatorAgent.CLOSE_SESSION.
    haltStep emits SessionClosed with outcome=HALTED and a structured
    haltReason. Override settings() with stepTimeout(60s) on actorStep,
    targetStep, and scoreStep, and defaultStepRecovery(maxRetries(2)
    .failoverTo(haltStep)).

- 1 EventSourcedEntity SessionEntity holding state Session{sessionId,
  persona, goal, maxTurns, SessionStatus status, List<TurnRecord> turns,
  Optional<SessionVerdict> sessionVerdict, Instant createdAt,
  Optional<Instant> finishedAt}. SessionStatus enum: RUNNING, COMPLETED,
  FLAGGED, HALTED. TurnOutcome enum: PASS, REVISE, POLICY_VIOLATION.
  SessionOutcome enum: CLEAN, FLAGGED, HALTED. Events: SessionCreated,
  TurnUtteranceRecorded, TurnResponseRecorded, TurnGuardrailVerdictRecorded,
  TurnVerdictRecorded, SessionClosed, DriftEvalRecorded. Commands:
  createSession, recordUtterance, recordResponse, recordGuardrail,
  recordTurnVerdict, closeSession, recordDriftEval, getSession.
  emptyState() returns Session.initial("","","",10) with no commandContext()
  reference. Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity ScenarioQueue with command enqueueScenario(persona,
  goal, maxTurns, submittedBy) emitting ScenarioSubmitted{sessionId, persona,
  goal, maxTurns, submittedBy, submittedAt}.

- 1 View SessionsView with row type SessionRow (mirrors Session; turns list
  is bounded at maxTurns so size stays reasonable). Table updater consumes
  SessionEntity events. ONE query getAllSessions SELECT * AS sessions FROM
  sessions_view. No WHERE status filter — caller filters client-side because
  Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer ScenarioConsumer subscribed to ScenarioQueue events; on
  ScenarioSubmitted starts a SimulationWorkflow with the sessionId as the
  workflow id.

- 2 TimedActions:
  * ScenarioSimulator — every 90s, reads next line from
    src/main/resources/sample-events/dialog-scenarios.jsonl and calls
    ScenarioQueue.enqueueScenario.
  * DriftSampler — every 45s, queries SessionsView.getAllSessions, finds
    sessions with a scored turn that has not yet been recorded as a
    DriftEvalRecorded event, and calls
    SessionEntity.recordDriftEval(turnNumber, outcome, overallScore,
    driftFlagged). Idempotent per (sessionId, turnNumber).

- 2 HttpEndpoints:
  * SimulationEndpoint at /api with POST /sessions, GET /sessions,
    GET /sessions/{id}, GET /sessions/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /sessions body is
    {persona, goal, maxTurns?, submittedBy?}; missing maxTurns defaults
    to 10, missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SimulationTasks.java declaring four Task<R> constants: INITIATE_TURN
  (resultConformsTo ActorUtterance), CONTINUE_TURN (ActorUtterance),
  SCORE_TURN (TurnVerdict), CLOSE_SESSION (SessionVerdict).
- Domain records ActorUtterance, TargetResponse, GuardrailVerdict,
  TurnScore, TurnVerdict, TurnRecord, SessionVerdict, Session; enums
  SessionStatus, TurnOutcome, SessionOutcome.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9769 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  multi-turn-simulator.simulation.max-turns = 10 and
  multi-turn-simulator.simulation.target-url = "stub", overridable by
  env var.
- src/main/resources/sample-events/dialog-scenarios.jsonl with 8 canned
  scenario lines, each shaped
  {"persona":"...","goal":"...","maxTurns":10}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, E1 eval-periodic drift-fairness-watch, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = agent-behavioral-testing,
  decisions.authority_level = evaluation-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/actor.md, prompts/evaluator.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Multi-Turn Actor
  Simulator", one-line pitch, prerequisites, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
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
  <title>Akka Sample: Multi-Turn Actor Simulator</title>.

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
  named in Section 9: actor.json, evaluator.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    actor.json — 8 ActorUtterance entries. Four are opening utterances
      for different personas (curious user, skeptical user, power user,
      non-native speaker). Four are follow-up utterances that reference
      the prior assistant turn (a clarifying question, a pushback, an
      off-topic pivot, a goal-completion signal). Each has a plausible
      turnNumber and utteredAt.
    evaluator.json — 8 entries split between TurnVerdict and SessionVerdict
      shapes. Four TurnVerdict entries: two with outcome=PASS and
      overallScore=4 or 5; two with outcome=REVISE and overallScore=2 or 3,
      driftFlagged=true, and a brief rationale citing a specific dimension.
      Two SessionVerdict entries: one CLEAN with sessionScore=4; one
      FLAGGED with driftFlagged=true and a one-sentence summaryRationale.
      Two TurnVerdict entries with outcome=POLICY_VIOLATION and
      overallScore=0, used to exercise J3.
- A MockModelProvider.seedFor(sessionId, turnNumber) helper makes the
  selection deterministic per (sessionId, turnNumber) so the same session
  in dev produces the same trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ActorAgent
  and EvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a SimulationTasks companion declaring the four Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Session row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SimulationTasks.java is mandatory; generating ActorAgent or
  EvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9769, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
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
