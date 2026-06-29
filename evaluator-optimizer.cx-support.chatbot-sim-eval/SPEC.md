# SPEC — chatbot-sim-eval

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Chatbot Simulation Evaluation.
**One-line pitch:** Submit a CX scenario; a simulated-user agent drives a multi-turn dialogue against the assistant chatbot; an evaluator agent scores the transcript and returns a pass/fail verdict with structured findings.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a simulated-user agent (`SimulatedUserAgent`) and an assistant agent (`ChatbotAgent`) for up to a configured number of turns, then hands the completed transcript to an evaluator agent (`EvaluatorAgent`). The blueprint also demonstrates three governance mechanisms — a **ci-gate** that prevents a simulation set from being marked green when any simulation in it fails evaluation, an **eval-periodic** event that records every simulation's verdict for continuous regression tracking, and an **output guardrail** that flags chatbot replies that violate a safe-content policy before the turn is added to the transcript.

## 3. User-facing flows

The user opens the App UI tab and submits a CX scenario (a persona key, an issue description, and an optional turn ceiling).

1. The system creates a `Simulation` record in `RUNNING` and starts a `SimulationWorkflow`.
2. `SimulatedUserAgent` opens the dialogue with an initial message consistent with the persona and the issue.
3. `ChatbotAgent` replies. Before the reply is committed to the transcript, the output guardrail checks it against the safe-content policy. A flagged reply is logged with `SafeContentFlag = true`; the turn is still included so the evaluator can score it.
4. The workflow loops: simulated-user turn → chatbot turn → guardrail check → append to transcript, until either the simulated user signals resolution (`RESOLVED`) or the turn ceiling is hit (`MAX_TURNS_REACHED`).
5. The workflow transitions the simulation to `EVALUATING` and hands the full transcript to `EvaluatorAgent`.
6. `EvaluatorAgent` scores the transcript against the rubric (resolution, tone, accuracy, policy compliance) and returns either `PASS` with a one-line summary, or `FAIL` with a typed `EvalFinding` payload (up to four dimension findings).
7. On `PASS`, the workflow transitions the simulation to `PASSED_EVALUATION` with the verdict and summary preserved on the entity.
8. On `FAIL`, the workflow transitions to `FAILED_EVALUATION`, preserving the full transcript, every finding, and the overall score.

A `ScenarioSimulator` (TimedAction) drips a canned scenario every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SimulatedUserAgent` | `AutonomousAgent` | Plays a CX customer persona through a configurable turn ceiling; signals resolution or exhaustion. | `SimulationWorkflow` | returns `UserTurn` to workflow |
| `ChatbotAgent` | `AutonomousAgent` | Plays the support assistant; answers each simulated-user turn using configured knowledge context. | `SimulationWorkflow` | returns `AssistantTurn` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a completed transcript against the rubric; returns `PASS` or `FAIL` with findings. | `SimulationWorkflow` | returns `EvalVerdict` to workflow |
| `SimulationWorkflow` | `Workflow` | Runs the turn loop, fires the guardrail check, calls EvaluatorAgent on completion, halts at ceiling. | `SimulationEndpoint`, `ScenarioConsumer` | `SimulationEntity` |
| `SimulationEntity` | `EventSourcedEntity` | Holds the simulation lifecycle: every turn, the transcript, and the evaluator's final verdict. | `SimulationWorkflow` | `SimulationsView` |
| `ScenarioQueue` | `EventSourcedEntity` | Logs each submitted scenario for replay and audit. | `SimulationEndpoint`, `ScenarioSimulator` | `ScenarioConsumer` |
| `SimulationsView` | `View` | List-of-simulations read model. | `SimulationEntity` events | `SimulationEndpoint` |
| `ScenarioConsumer` | `Consumer` | Subscribes to `ScenarioQueue` events; starts a workflow per submission. | `ScenarioQueue` events | `SimulationWorkflow` |
| `ScenarioSimulator` | `TimedAction` | Drips a sample scenario every 60 s from `sample-events/cx-scenarios.jsonl`. | scheduler | `ScenarioQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `SimulationsView`, records an `EvalRecorded` event for any simulation that completed since the last tick. | scheduler | `SimulationEntity` |
| `SimulationEndpoint` | `HttpEndpoint` | `/api/simulations/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SimulationsView`, `ScenarioQueue`, `SimulationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ScenarioBrief(String personaKey, String issueDescription, int maxTurns, String submittedBy) {}

record UserTurn(String text, boolean signaledResolution, Instant turnedAt) {}

record AssistantTurn(String text, boolean safeContentFlagged, String flagDetail, Instant turnedAt) {}

record DialogueTurn(
    int turnNumber,
    UserTurn userTurn,
    Optional<AssistantTurn> assistantTurn
) {}

record DimensionFinding(String dimension, int score, String observation) {}

record EvalFinding(List<DimensionFinding> findings, String overallSummary) {}

record EvalVerdict(
    EvalOutcome outcome,
    EvalFinding finding,
    int overallScore,
    Instant evaluatedAt
) {}

record Simulation(
    String simulationId,
    String personaKey,
    String issueDescription,
    int maxTurns,
    SimulationStatus status,
    List<DialogueTurn> turns,
    Optional<EvalVerdict> verdict,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SimulationStatus { RUNNING, EVALUATING, PASSED_EVALUATION, FAILED_EVALUATION }

enum EvalOutcome { PASS, FAIL }
```

### Events (on `SimulationEntity`)

`SimulationCreated`, `UserTurnRecorded`, `AssistantTurnRecorded`, `ConversationConcluded`, `EvalVerdictRecorded`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/simulations` — body `{ personaKey, issueDescription, maxTurns? }` → `{ simulationId }`. Starts a workflow.
- `GET /api/simulations` — list all simulations. Optional `?status=RUNNING|EVALUATING|PASSED_EVALUATION|FAILED_EVALUATION`.
- `GET /api/simulations/{id}` — one simulation (including every turn and the verdict).
- `GET /api/simulations/sse` — server-sent events stream of every simulation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Chatbot Simulation Evaluation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (ci-gate = red, eval-periodic = blue, guardrail = red).
- **App UI** — form to submit a scenario, live list of simulations with status pills, click-to-expand per-turn timeline showing each user turn, each assistant turn with the guardrail verdict, and the final evaluator findings.

Browser title: `<title>Akka Sample: Chatbot Simulation Evaluation</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ChatbotAgent`): a policy check on every assistant turn before it is appended to the transcript. A reply that matches the safe-content violation pattern sets `AssistantTurn.safeContentFlagged = true` with a structured `flagDetail`. The turn is still logged; the evaluator's rubric scores the Policy Compliance dimension accordingly. Enforcement: non-blocking (logged, not blocked), so the dialogue continues to completion.
- **CI1 — ci-gate** (`test-gate`): after a simulation set runs, a gate step checks whether every simulation in the set has `PASSED_EVALUATION`. Any `FAILED_EVALUATION` result causes the gate to return a non-zero exit so a CI pipeline can block a deployment. Enforcement: build-gate.
- **EP1 — eval-periodic** (`continuous-accuracy`): every completed simulation's verdict and score are recorded as an `EvalRecorded` event. The `EvalSampler` TimedAction is the canonical writer. Enforcement: non-blocking. Events surface in the App UI and in `/api/simulations/{id}`.

## 9. Agent prompts

- `SimulatedUserAgent` → `prompts/simulated-user.md`. Plays a CX customer persona; opens the conversation and responds to each assistant reply until the issue is resolved or the turn ceiling is hit.
- `ChatbotAgent` → `prompts/chatbot.md`. Answers each simulated-user turn using provided knowledge context; keeps replies concise and policy-compliant.
- `EvaluatorAgent` → `prompts/evaluator.md`. Scores a completed transcript against the rubric; returns `PASS` or `FAIL` with dimension-level findings.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a scenario; simulation progresses `RUNNING` → `EVALUATING` → `PASSED_EVALUATION`; the App UI shows every turn's exchange and the evaluator's summary.
2. **J2 — evaluation failure** — Submit a scenario whose issue is unresolvable (test mode forces the Evaluator to `FAIL`); simulation ends in `FAILED_EVALUATION` with every turn and the full `EvalFinding` preserved.
3. **J3 — guardrail flag** — Submit a scenario that provokes a policy-violating chatbot reply; the turn is logged with `safeContentFlagged = true`; the evaluator scores the Policy Compliance dimension low; the simulation ends without being blocked mid-dialogue.
4. **J4 — ci-gate** — After a set of simulations completes, calling `POST /api/simulations/ci-gate` returns `{ passed: false }` when any simulation in the set is `FAILED_EVALUATION` and `{ passed: true }` when all are `PASSED_EVALUATION`.
5. **J5 — eval-event timeline** — The expanded view of any simulation shows one `EvalRecorded` event per completed simulation and the terminal verdict.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named chatbot-sim-eval demonstrating the evaluator-optimizer ×
cx-support cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-cx-support-chatbot-sim-eval.
Java package io.akka.samples.chatbotsimulationevaluation. Akka 3.6.0.
HTTP port 9888.

Components to wire (exactly):
- 3 AutonomousAgents:
  * SimulatedUserAgent — definition() with
    capability(TaskAcceptance.of(OPEN_DIALOGUE).maxIterationsPerTask(1)) AND
    capability(TaskAcceptance.of(REPLY_AS_USER).maxIterationsPerTask(1)).
    System prompt loaded from prompts/simulated-user.md. Returns UserTurn{text,
    signaledResolution, turnedAt} for both OPEN_DIALOGUE and REPLY_AS_USER.
    The REPLY_AS_USER task takes (personaKey, issueDescription, priorAssistantTurn)
    as inputs.
  * ChatbotAgent — definition() with
    capability(TaskAcceptance.of(ANSWER_TURN).maxIterationsPerTask(1)).
    System prompt from prompts/chatbot.md. Returns AssistantTurn{text,
    safeContentFlagged, flagDetail, turnedAt}.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_TRANSCRIPT).maxIterationsPerTask(2)).
    System prompt from prompts/evaluator.md. Returns EvalVerdict{outcome,
    finding, overallScore, evaluatedAt} where outcome is the EvalOutcome enum
    (PASS | FAIL) and overallScore is a 1–10 integer.

- 1 Workflow SimulationWorkflow with steps:
    startStep -> openDialogueStep -> [loop: chatbotStep -> guardrailStep ->
    userReplyStep -> check(signaledResolution OR turnCount >= maxTurns)]
    -> concludeStep -> evaluateStep ->
    [verdict PASS? passStep : failStep] -> END.
  openDialogueStep calls forAutonomousAgent(SimulatedUserAgent.class, simulationId)
    .runSingleTask(OPEN_DIALOGUE).
  chatbotStep calls forAutonomousAgent(ChatbotAgent.class, simulationId)
    .runSingleTask(ANSWER_TURN) then emits AssistantTurnRecorded.
  guardrailStep is a pure-function step: checks the AssistantTurn text against
    a safe-content keyword list. On match, sets safeContentFlagged=true and
    flagDetail; emits AssistantTurnRecorded with the flag. Continues (non-blocking).
  userReplyStep calls forAutonomousAgent(SimulatedUserAgent.class, simulationId)
    .runSingleTask(REPLY_AS_USER) then emits UserTurnRecorded.
  concludeStep emits ConversationConcluded with haltReason (RESOLVED or MAX_TURNS_REACHED)
    and transitions status to EVALUATING.
  evaluateStep calls forAutonomousAgent(EvaluatorAgent.class, simulationId)
    .runSingleTask(EVALUATE_TRANSCRIPT) passing the full transcript.
  passStep emits EvalVerdictRecorded with outcome=PASS. failStep emits
    EvalVerdictRecorded with outcome=FAIL and the full EvalFinding payload.
  Override settings() with stepTimeout(90s) on chatbotStep, openDialogueStep,
    userReplyStep, and evaluateStep; stepTimeout(120s) on evaluateStep.
  defaultStepRecovery(maxRetries(2).failoverTo(failStep)).

- 1 EventSourcedEntity SimulationEntity holding state Simulation{simulationId,
  personaKey, issueDescription, maxTurns, SimulationStatus status,
  List<DialogueTurn> turns, Optional<EvalVerdict> verdict,
  Optional<String> haltReason, Instant createdAt, Optional<Instant> finishedAt}.
  SimulationStatus enum: RUNNING, EVALUATING, PASSED_EVALUATION, FAILED_EVALUATION.
  Events: SimulationCreated, UserTurnRecorded, AssistantTurnRecorded,
  ConversationConcluded, EvalVerdictRecorded, EvalRecorded.
  Commands: createSimulation, recordUserTurn, recordAssistantTurn,
  concludeConversation, recordVerdict, recordEval, getSimulation.
  emptyState() returns Simulation.initial("", "default", "none", 10) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity ScenarioQueue with command enqueueScenario(personaKey,
  issueDescription, maxTurns, submittedBy) emitting ScenarioSubmitted{simulationId,
  personaKey, issueDescription, maxTurns, submittedBy, submittedAt}.

- 1 View SimulationsView with row type SimulationRow (mirrors Simulation; the turns
  list is preserved as-is — bounded at maxTurns so size stays reasonable). Table
  updater consumes SimulationEntity events. ONE query getAllSimulations SELECT * AS
  simulations FROM simulations_view. No WHERE status filter — caller filters
  client-side (Lesson 2).

- 1 Consumer ScenarioConsumer subscribed to ScenarioQueue events; on ScenarioSubmitted
  starts a SimulationWorkflow with the simulationId as the workflow id.

- 2 TimedActions:
  * ScenarioSimulator — every 60s, reads next line from
    src/main/resources/sample-events/cx-scenarios.jsonl and calls
    ScenarioQueue.enqueueScenario.
  * EvalSampler — every 30s, queries SimulationsView.getAllSimulations, finds simulations
    with a verdict that have not yet been recorded as an EvalRecorded event, and calls
    SimulationEntity.recordEval(simulationId, outcome, overallScore, anyFlagged).
    Idempotent per simulationId.

- 2 HttpEndpoints:
  * SimulationEndpoint at /api with POST /simulations, GET /simulations,
    GET /simulations/{id}, GET /simulations/sse, GET /simulations/ci-gate,
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /simulations body is
    {personaKey, issueDescription, maxTurns?}; missing maxTurns defaults to 10,
    missing submittedBy defaults to "anonymous".
    GET /simulations/ci-gate accepts ?set=<comma-sep-ids> and returns
    { passed: boolean, failed: [simulationId...], pending: [simulationId...] }.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SimulationTasks.java declaring four Task<R> constants: OPEN_DIALOGUE (resultConformsTo
  UserTurn), REPLY_AS_USER (UserTurn), ANSWER_TURN (AssistantTurn),
  EVALUATE_TRANSCRIPT (EvalVerdict).
- Domain records UserTurn, AssistantTurn, DialogueTurn, DimensionFinding, EvalFinding,
  EvalVerdict, ScenarioBrief, Simulation; enums SimulationStatus, EvalOutcome.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9888
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from the
  canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  chatbot-sim-eval.simulation.max-turns = 10 and
  chatbot-sim-eval.simulation.default-persona-key = "frustrated-customer",
  overridable by env var.
- src/main/resources/sample-events/cx-scenarios.jsonl with 8 canned scenario
  lines, each shaped {"personaKey":"...", "issueDescription":"...", "maxTurns":10}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response non-blocking, CI1 ci-gate test-gate build-gate, EP1
  eval-periodic continuous-accuracy non-blocking) and a matching simplified_view
  list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = chatbot-quality-evaluation,
  decisions.authority_level = evaluation-only, data.data_classes.pii = false,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/simulated-user.md, prompts/chatbot.md, prompts/evaluator.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Chatbot Simulation Evaluation",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs matching the formal exemplar:
  Overview (eyebrow + headline + no subtitle + Try it / How it works / Components /
  API contract cards); Architecture (4 mermaid diagrams + click-to-expand component
  table); Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows);
  App UI (form + live list with status pills, click-to-expand per-turn timeline).
  Browser title exactly:
  <title>Akka Sample: Chatbot Simulation Evaluation</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via
        the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class name
  (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: simulated-user.json, chatbot.json, evaluator.json),
  picks one entry pseudo-randomly per call, and deserialises it into the
  agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    simulated-user.json — 6 UserTurn entries. Four are mid-dialogue user
      replies covering personas "frustrated-customer", "first-time-caller",
      "enterprise-admin", "returns-specialist" with signaledResolution=false.
      Two are resolution turns with signaledResolution=true.
    chatbot.json — 6 AssistantTurn entries. Four are valid concise replies
      with safeContentFlagged=false. One has safeContentFlagged=true with
      flagDetail="Contains unsupported promise about refund timeline." One is
      a fallback "I'm sorry, I don't have information on that" reply.
    evaluator.json — 6 EvalVerdict entries. Three return outcome=PASS with
      overallScore=8 or 9 and a one-sentence summary. Three return outcome=FAIL
      with overallScore=3 or 4 and an EvalFinding payload with four
      DimensionFinding entries for Resolution, Tone, Accuracy, PolicyCompliance.
- A MockModelProvider.seedFor(simulationId, turnNumber) helper makes the
  selection deterministic per (simulationId, turnNumber) so the same simulation
  in dev produces the same trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. SimulatedUserAgent,
  ChatbotAgent, and EvaluatorAgent all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a SimulationTasks
  companion declaring the four Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Simulation row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SimulationTasks.java is mandatory; generating any agent without it
  is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9888, declared in application.conf dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never surfaces
  a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never T1/T2/T3/T4,
  never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with display:none.
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
