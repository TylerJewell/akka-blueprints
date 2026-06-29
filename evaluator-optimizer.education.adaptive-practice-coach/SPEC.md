# SPEC — adaptive-practice-coach

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Adaptive Practice Coach.
**One-line pitch:** Start a practice session; a tutor agent generates items calibrated to a learner's level; a scorer agent evaluates each response and updates a mastery estimate; the loop adapts difficulty until mastery is confirmed or the session cap is reached.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`ItemGeneratorAgent`) and a scorer agent (`ScorerAgent`), feeding each score and diagnostic tag back into the next item's difficulty selection until mastery convergence or a cap. The blueprint also demonstrates two governance mechanisms — a **guardrail** that vets every generated item for curriculum alignment before the learner sees it, and an **eval-event** that records each scored item's diagnostic value for downstream quality measurement.

## 3. User-facing flows

The user opens the App UI tab and starts a practice session (a topic plus a learner identifier).

1. The system creates a `LearnerSession` in `ACTIVE` state and starts a `PracticeSessionWorkflow`.
2. The ItemGenerator produces item #1 at the initial difficulty level for the topic.
3. The curriculum-alignment guardrail vets the item. Off-topic or malformed items are short-circuited back to the generator with a deterministic feedback note; they never reach the learner.
4. The learner submits a response through the App UI.
5. The Scorer evaluates the response against the answer key and returns a `ScoreReport` with correctness, a partial-credit analysis, a diagnostic tag, and a mastery delta.
6. On `MASTERED` (mastery estimate ≥ threshold), the workflow transitions the session to `MASTERED` with the item count and final mastery score.
7. On `CONTINUE`, the workflow records the item, the guardrail verdict, and the score on the entity, then calls the generator again with the diagnostic tag and updated difficulty. The generator produces item #2.
8. If the loop reaches `maxItems` (default 10) without `MASTERED`, the halt activates: the workflow ends with `CAP_REACHED`, the best mastery estimate and every item are preserved on the entity for audit.

An `EnrollmentSimulator` (TimedAction) drips a canned enrollment every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ItemGeneratorAgent` | `AutonomousAgent` | Generates a practice item at a specified difficulty for a topic; accepts prior diagnostic feedback on revision calls. | `PracticeSessionWorkflow` | returns `PracticeItem` to workflow |
| `ScorerAgent` | `AutonomousAgent` | Evaluates a learner's response against the item's answer key; returns a `ScoreReport` with correctness, partial-credit analysis, diagnostic tag, and mastery delta. | `PracticeSessionWorkflow` | returns `ScoreReport` to workflow |
| `PracticeSessionWorkflow` | `Workflow` | Runs the generate → guardrail → await-response → score → adapt loop; halts at the item cap or on mastery. | `CoachEndpoint`, `SessionRequestConsumer` | `LearnerSessionEntity` |
| `LearnerSessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, every item, every response, every score report, the mastery estimate, and the terminal outcome. | `PracticeSessionWorkflow` | `SessionsView` |
| `EnrollmentQueue` | `EventSourcedEntity` | Logs each session-start request for replay and audit. | `CoachEndpoint`, `EnrollmentSimulator` | `SessionRequestConsumer` |
| `SessionsView` | `View` | List-of-sessions read model. | `LearnerSessionEntity` events | `CoachEndpoint` |
| `SessionRequestConsumer` | `Consumer` | Subscribes to `EnrollmentQueue` events; starts a workflow per enrollment. | `EnrollmentQueue` events | `PracticeSessionWorkflow` |
| `EnrollmentSimulator` | `TimedAction` | Drips a sample enrollment every 60 s from `sample-events/practice-enrollments.jsonl`. | scheduler | `EnrollmentQueue` |
| `DiagnosticSampler` | `TimedAction` | Every 30 s, scans `SessionsView`, records a `DiagnosticRecorded` event for any scored item that has not yet been sampled. | scheduler | `LearnerSessionEntity` |
| `CoachEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start, get, list, submit response, SSE; plus `/api/metadata/*`. | — | `SessionsView`, `EnrollmentQueue`, `LearnerSessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Enrollment(String topic, String learnerId, int initialDifficulty) {}

record PracticeItem(
    String itemId,
    String question,
    String answerKey,
    int difficulty,
    String topic,
    Instant generatedAt
) {}

record CurriculumVerdict(boolean passed, String reasonCode, String detail) {}

record PartialCreditNote(String dimension, int pointsAwarded, int pointsPossible, String explanation) {}

record ScoreReport(
    ScoreVerdict verdict,
    boolean correct,
    List<PartialCreditNote> partialCredit,
    String diagnosticTag,
    double masteryDelta,
    String rationale,
    Instant scoredAt
) {}

record SessionItem(
    int itemNumber,
    PracticeItem item,
    CurriculumVerdict curriculumVerdict,
    Optional<String> learnerResponse,
    Optional<ScoreReport> scoreReport
) {}

record LearnerSession(
    String sessionId,
    String learnerId,
    String topic,
    int maxItems,
    double masteryThreshold,
    SessionStatus status,
    int currentDifficulty,
    double masteryEstimate,
    List<SessionItem> items,
    Optional<Integer> masteredAtItemNumber,
    Optional<String> terminalReason,
    Instant startedAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { ACTIVE, AWAITING_RESPONSE, MASTERED, CAP_REACHED }

enum ScoreVerdict { MASTERED, CONTINUE }
```

### Events (on `LearnerSessionEntity`)

`SessionStarted`, `ItemGenerated`, `ItemCurriculumVerdictRecorded`, `ResponseSubmitted`, `ItemScored`, `SessionMastered`, `SessionCapReached`, `DiagnosticRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ topic, learnerId?, initialDifficulty? }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=ACTIVE|AWAITING_RESPONSE|MASTERED|CAP_REACHED`.
- `GET /api/sessions/{id}` — one session (including every item and every score report).
- `POST /api/sessions/{id}/response` — body `{ response: String }` → `202`. Delivers learner answer; workflow advances.
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Adaptive Practice Coach"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (guardrail = red, eval-event = blue).
- **App UI** — form to start a session, live list of sessions with status pills, click-to-expand per-item timeline showing each question, the curriculum verdict, the learner's response, the scorer's verdict, and the diagnostic tag.

Browser title: `<title>Akka Sample: Adaptive Practice Coach</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — curriculum-alignment guardrail** (`before-agent-response` on `ItemGeneratorAgent`): a deterministic check that each generated item's topic, vocabulary, and structure match the session's declared subject area. Off-topic or malformed items short-circuit back to the generator with a structured feedback note (`reasonCode = CURRICULUM_MISMATCH`); they never reach the learner. Enforcement: blocking.
- **E1 — diagnostic eval-event** (`on-decision-eval`): every scored item's `diagnosticTag` and `masteryDelta` are recorded as a `DiagnosticRecorded` event. `DiagnosticSampler` (TimedAction) is the canonical writer; the workflow also emits on terminal transitions. Events surface in the App UI's per-item timeline and in `/api/sessions/{id}`. Enforcement: non-blocking.

## 9. Agent prompts

- `ItemGeneratorAgent` → `prompts/item-generator.md`. Generates a practice question and answer key at the specified difficulty for the topic; on a revision call, takes prior curriculum feedback and regenerates.
- `ScorerAgent` → `prompts/scorer.md`. Evaluates a learner's response against the answer key; returns a `ScoreReport` with correctness, partial-credit breakdown, a diagnostic tag, and a mastery delta.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — mastery convergence** — Start a session; the learner answers correctly several times; the mastery estimate crosses the threshold; session ends in `MASTERED` before the item cap.
2. **J2 — cap reached** — Learner consistently answers incorrectly; session runs through all `maxItems` and lands in `CAP_REACHED` with every item and score preserved.
3. **J3 — curriculum guardrail** — Generator returns an off-topic item; guardrail records `CURRICULUM_MISMATCH`; learner never sees the item; generator produces a replacement.
4. **J4 — diagnostic timeline** — Expanded session view shows one `DiagnosticRecorded` event per scored item plus a terminal event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named adaptive-practice-coach demonstrating the
evaluator-optimizer × education cell. Runs out of the box (no external
services). Maven group io.akka.samples. Artifact id
evaluator-optimizer-education-adaptive-practice-coach. Java package
io.akka.samples.adaptivepracticecoach. Akka 3.6.0. HTTP port 9330.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ItemGeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE_ITEM).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ITEM).maxIterationsPerTask(3)).
    System prompt loaded from prompts/item-generator.md. Returns
    PracticeItem{itemId, question, answerKey, difficulty, topic,
    generatedAt} for both GENERATE_ITEM and REVISE_ITEM. The
    REVISE_ITEM task takes (topic, difficulty, curriculumFeedback)
    as inputs.
  * ScorerAgent — definition() with
    capability(TaskAcceptance.of(SCORE_RESPONSE).maxIterationsPerTask(2)).
    System prompt from prompts/scorer.md. Returns
    ScoreReport{verdict, correct, partialCredit, diagnosticTag,
    masteryDelta, rationale, scoredAt} where verdict is the
    ScoreVerdict enum (MASTERED | CONTINUE).

- 1 Workflow PracticeSessionWorkflow with steps:
    startStep -> generateStep -> curriculumStep ->
    [curriculum FAIL? generateStep again with structured feedback :
    awaitResponseStep] ->
    [response received? scoreStep : awaitResponseStep (timeout)] ->
    [verdict MASTERED? masterStep :
    (itemCount < maxItems ? generateStep with diagnosticTag : capStep)]
    -> END.
  generateStep calls
    forAutonomousAgent(ItemGeneratorAgent.class, sessionId)
    .runSingleTask(GENERATE_ITEM or REVISE_ITEM).
  scoreStep calls
    forAutonomousAgent(ScorerAgent.class, sessionId)
    .runSingleTask(SCORE_RESPONSE).
  masterStep emits SessionMastered.
  capStep emits SessionCapReached with the highest mastery estimate
    reached and a structured terminalReason.
  Override settings() with stepTimeout(60s) on generateStep and
    scoreStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(capStep)).
  curriculumStep is a pure-function step (no LLM call): checks
    whether item topic matches session topic and item has non-empty
    question and answerKey. On FAIL, emits
    ItemCurriculumVerdictRecorded with verdict.passed = false and
    reasonCode = "CURRICULUM_MISMATCH", then transitions back to
    generateStep with a structured feedback note.
  awaitResponseStep parks the workflow until
    CoachEndpoint.submitResponse signals it via a workflow step
    result; stepTimeout(300s) returns to awaitResponseStep on
    expiry (the learner has 5 minutes to respond).

- 1 EventSourcedEntity LearnerSessionEntity holding state
  LearnerSession{sessionId, learnerId, topic, maxItems,
  masteryThreshold, SessionStatus status, int currentDifficulty,
  double masteryEstimate, List<SessionItem> items,
  Optional<Integer> masteredAtItemNumber,
  Optional<String> terminalReason, Instant startedAt,
  Optional<Instant> finishedAt}.
  SessionStatus enum: ACTIVE, AWAITING_RESPONSE, MASTERED, CAP_REACHED.
  Events: SessionStarted, ItemGenerated, ItemCurriculumVerdictRecorded,
  ResponseSubmitted, ItemScored, SessionMastered, SessionCapReached,
  DiagnosticRecorded.
  Commands: startSession, recordItem, recordCurriculumVerdict,
  recordResponse, recordScore, masterSession, reachCap,
  recordDiagnostic, getSession.
  emptyState() returns LearnerSession.initial("","","",10,0.8)
  with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity EnrollmentQueue with command
  enqueueEnrollment(topic, learnerId, initialDifficulty) emitting
  EnrollmentRequested{sessionId, topic, learnerId, initialDifficulty,
  requestedAt}.

- 1 View SessionsView with row type SessionRow (mirrors
  LearnerSession; the items list is bounded at maxItems so size stays
  manageable). Table updater consumes LearnerSessionEntity events.
  ONE query getAllSessions SELECT * AS sessions FROM sessions_view.
  No WHERE status filter — caller filters client-side because Akka
  cannot auto-index enum columns (Lesson 2).

- 1 Consumer SessionRequestConsumer subscribed to EnrollmentQueue
  events; on EnrollmentRequested starts a PracticeSessionWorkflow
  with the sessionId as the workflow id.

- 2 TimedActions:
  * EnrollmentSimulator — every 60s, reads next line from
    src/main/resources/sample-events/practice-enrollments.jsonl and
    calls EnrollmentQueue.enqueueEnrollment.
  * DiagnosticSampler — every 30s, queries SessionsView.getAllSessions,
    finds items that have been scored but do not yet have a matching
    DiagnosticRecorded event, and calls
    LearnerSessionEntity.recordDiagnostic(itemNumber, diagnosticTag,
    masteryDelta, correct). Idempotent per (sessionId, itemNumber).

- 2 HttpEndpoints:
  * CoachEndpoint at /api with POST /sessions, GET /sessions,
    GET /sessions/{id}, POST /sessions/{id}/response,
    GET /sessions/sse, and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/. The POST
    /sessions body is {topic, learnerId?, initialDifficulty?};
    missing learnerId defaults to "anonymous", missing
    initialDifficulty defaults to 3 (mid-range on a 1–5 scale).
    POST /sessions/{id}/response body is {response: String}.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- CoachTasks.java declaring three Task<R> constants: GENERATE_ITEM
  (resultConformsTo PracticeItem), REVISE_ITEM (PracticeItem),
  SCORE_RESPONSE (ScoreReport).
- Domain records PracticeItem, CurriculumVerdict, PartialCreditNote,
  ScoreReport, SessionItem, LearnerSession; enums SessionStatus,
  ScoreVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9330 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-session config
  adaptive-practice-coach.session.max-items = 10,
  adaptive-practice-coach.session.mastery-threshold = 0.8,
  adaptive-practice-coach.session.default-initial-difficulty = 3,
  overridable by env var.
- src/main/resources/sample-events/practice-enrollments.jsonl with
  8 canned enrollment lines, each shaped
  {"topic":"...", "learnerId":"sim-learner", "initialDifficulty":3}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the root-level files for the endpoint to serve
  from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = adaptive-tutoring,
  decisions.authority_level = advisory-only,
  data.data_classes.pii = true (learnerId),
  capabilities.content-generation = true,
  capabilities.predictive-policing = false;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/item-generator.md, prompts/scorer.md loaded at agent
  startup as system prompts.
- README.md at the project root: title
  "Akka Sample: Adaptive Practice Coach", one-line pitch,
  prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single
  self-contained HTML file (no ui/ folder, no npm build). Inline
  CSS + JS. Runtime CDN imports for markdown and YAML libs are
  acceptable. Five tabs matching the formal exemplar: Overview
  (eyebrow + headline + no subtitle + Try it / How it works /
  Components / API contract cards); Architecture (4 mermaid diagrams
  + click-to-expand component table); Risk Survey (7 sub-tabs from
  governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows); App UI (form + live list with status pills,
  click-to-expand per-item timeline). Browser title exactly:
  <title>Akka Sample: Adaptive Practice Coach</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
  for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY.
  If exactly one is set, default application.conf's model-provider to
  match and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that
        returns random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider
  interface with per-agent dispatch on the agent class name (or
  Task<R> id). Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    item-generator.json — 6 PracticeItem entries. Four are
      well-formed items across the topics in
      practice-enrollments.jsonl at difficulties 2–4. One is an
      intentionally off-topic item (exercises the guardrail in J3).
      One is a revision item that references the prior curriculum
      feedback.
    scorer.json — 6 ScoreReport entries. Three return
      verdict=MASTERED with correct=true, masteryDelta=0.15, and
      a concise diagnosticTag. Three return verdict=CONTINUE with
      correct=false, masteryDelta=-0.05, and a diagnosticTag naming
      the gap (e.g., "misconception:fraction-denominator",
      "recall-gap:periodic-table-period-3").
- A MockModelProvider.seedFor(sessionId, itemNumber) helper makes
  selection deterministic per (sessionId, itemNumber).

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: ItemGeneratorAgent and ScorerAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a
  CoachTasks companion declaring the three Task<R> constants.
- Lesson 2: SessionsView exposes one getAllSessions query; callers
  filter client-side.
- Lesson 4: generateStep and scoreStep each carry
  stepTimeout(Duration.ofSeconds(60));
  awaitResponseStep carries stepTimeout(Duration.ofSeconds(300)).
  The default 5-second timeout never applies to agent-calling steps.
- Lesson 6: every nullable lifecycle field on LearnerSession is
  Optional<T>; event-appliers wrap values with Optional.of(...).
- Lesson 7: CoachTasks.java is mandatory; generating
  ItemGeneratorAgent or ScorerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: the run command is /akka:build.
- Lesson 10: HTTP port is 9330, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI
  never surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no
  horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" —
  never T1/T2/T3/T4.
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour,
  edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf
  records only ${?VAR_NAME} substitution.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
