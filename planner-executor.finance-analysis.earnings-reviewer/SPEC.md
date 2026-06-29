# SPEC — earnings-reviewer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Earnings Reviewer.
**One-line pitch:** Submit a `(ticker, period)` pair; a ReviewPlanner reads the earnings transcript and the matching filing, dispatches passages to specialist readers, proposes typed updates to an internal financial model, and emits thesis-relevant flags scored against evidence quality.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to finance analysis. The ReviewPlanner owns a single ledger — a **review ledger** carrying (a) extracted passages by source, (b) proposed `ModelUpdate` rows, (c) candidate `ThesisFlag` records, and (d) the current dispatch decision. Each loop iteration the planner picks one of four specialists — TRANSCRIPT, FILING, MODEL, FLAG — runs it, appends the typed result to the ledger, and decides whether to continue, replan, complete, or fail. A review terminates `COMPLETED` once at least one model update and at least one flag are on the ledger, or `FAILED` if the planner exhausts its replan budget.

The blueprint also demonstrates two governance mechanisms wired into the output side of the loop:

- a **before-agent-response guardrail** that vets every `ThesisFlag` before it lands on the review answer (confidence must clear a floor; evidence list must cite at least two passages from the ledger; category must come from the closed enum),
- an **on-decision eval-event** that scores each accepted flag against an evidence-quality rubric (passage count, source diversity, recency, contradiction flags) and writes the score onto the answer.

Material-change flags drive downstream trading decisions; both controls live on the output of the planner-executor loop, not the input.

## 3. User-facing flows

The user opens the App UI tab and submits a review request by ticker and period.

1. The system creates a `Review` record in `PENDING` and starts a `ReviewWorkflow`.
2. The ReviewPlanner drafts a `ReviewLedger { passages, modelUpdates, flags, currentDispatch }` and emits `ReviewPlanned`. Status → `READING`.
3. The workflow enters the executor loop. Each iteration:
   - ReviewPlanner reads the ledger and proposes a `DispatchDecision { specialist, focus, rationale }`.
   - The chosen specialist runs the focus area and returns a typed `SpecialistResult` (passages, model updates, or candidate flags).
   - The workflow appends the result to the ledger via `ReviewEntity`.
4. When the planner emits `Complete`, the workflow runs the **finalize** branch:
   - For every candidate `ThesisFlag`, the **before-agent-response guardrail** vets the flag. Rejected flags are dropped and a `FlagRejected` entry is appended to the ledger with the rejection reason; the planner is given one opportunity to revise.
   - For every accepted flag, the **on-decision eval-event** scores the evidence quality and writes the score onto the flag.
   - The workflow emits `ReviewCompleted` carrying the `ReviewAnswer { acceptedFlags, modelDiff, rejectedFlags, narrative }`. Status → `COMPLETED`.
5. On `COMPLETED`, the workflow applies every accepted `ModelUpdate` to `ModelEntity` for that ticker.
6. If the planner exhausts the replan budget (three consecutive `Replan` outputs) or the same `(specialist, focus)` pair fails three times, the workflow emits `ReviewFailed`. Status → `FAILED`.

A `EarningsSimulator` (TimedAction) drips a sample review request every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewPlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains the review ledger. Produces `ReviewAnswer` on completion. | `ReviewWorkflow` | returns typed result to workflow |
| `TranscriptReaderAgent` | `AutonomousAgent` | Extracts management-commentary and analyst-Q&A passages from a transcript fixture. | `ReviewWorkflow` | — |
| `FilingReaderAgent` | `AutonomousAgent` | Extracts line-item deltas (revenue, margin, guidance) from a filing fixture. | `ReviewWorkflow` | — |
| `ModelUpdaterAgent` | `AutonomousAgent` | Converts passages into typed `ModelUpdate` rows against the internal model vocabulary. | `ReviewWorkflow` | — |
| `ThesisFlagAgent` | `AutonomousAgent` | Classifies candidate flags into `THESIS_CONFIRMING`, `THESIS_CHALLENGING`, `NOT_MATERIAL`; assigns a confidence and an evidence list. | `ReviewWorkflow` | — |
| `ReviewWorkflow` | `Workflow` | Drives the plan → dispatch → record → decide loop, plus the finalize branch (guardrail + eval). | `ReviewEndpoint`, `ReviewSubmissionConsumer` | `ReviewEntity`, `ModelEntity` |
| `ReviewEntity` | `EventSourcedEntity` | Holds one review's lifecycle, the review ledger, and the final answer. | `ReviewWorkflow` | `ReviewView` |
| `ModelEntity` | `EventSourcedEntity` | Holds the internal financial model per ticker. Keyed by ticker. | `ReviewWorkflow` | `ReviewView` (on join) |
| `SubmissionQueue` | `EventSourcedEntity` | Audit log of submitted review requests. | `ReviewEndpoint`, `EarningsSimulator` | `ReviewSubmissionConsumer` |
| `ReviewView` | `View` | List-of-reviews read model for the UI. | `ReviewEntity` events | `ReviewEndpoint` |
| `ReviewSubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a `ReviewWorkflow` per submission. | `SubmissionQueue` events | `ReviewWorkflow` |
| `EarningsSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/review-requests.jsonl` and enqueues it. | scheduler | `SubmissionQueue` |
| `StaleReviewMonitor` | `TimedAction` | Every 60 s, marks any review stuck in `READING` past 5 minutes as `STALE`. The workflow polls this and ends with `ReviewFailedStale`. | scheduler | `ReviewEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/reviews/*` and `/api/models/*` — submit, get, list, SSE, current model. | — | `ReviewView`, `SubmissionQueue`, `ReviewEntity`, `ModelEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record ReviewRequest(String ticker, String period, String submittedBy) {}

record Passage(
    PassageSource source,
    String section,
    String text,
    Instant extractedAt
) {}

record ModelUpdate(
    String metric,
    String priorValue,
    String proposedValue,
    String sourceSection,
    Optional<String> rationale
) {}

record ThesisFlag(
    FlagCategory category,
    String summary,
    double confidence,
    List<String> evidence,
    Optional<Double> evidenceQualityScore
) {}

record DispatchDecision(
    SpecialistKind specialist,
    String focus,
    String rationale
) {}

record SpecialistResult(
    SpecialistKind specialist,
    String focus,
    boolean ok,
    List<Passage> passages,
    List<ModelUpdate> modelUpdates,
    List<ThesisFlag> candidateFlags,
    Optional<String> errorReason
) {}

record ReviewLedger(
    List<Passage> passages,
    List<ModelUpdate> modelUpdates,
    List<ThesisFlag> candidateFlags,
    Optional<DispatchDecision> currentDispatch
) {}

record ReviewAnswer(
    List<ThesisFlag> acceptedFlags,
    List<ModelUpdate> modelDiff,
    List<RejectedFlag> rejectedFlags,
    String narrative,
    Instant producedAt
) {}

record RejectedFlag(ThesisFlag flag, String reason) {}

record Review(
    String reviewId,
    String ticker,
    String period,
    ReviewStatus status,
    Optional<ReviewLedger> ledger,
    Optional<ReviewAnswer> answer,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record InternalModel(
    String ticker,
    Map<String, String> metrics,
    Instant updatedAt
) {}

enum SpecialistKind { TRANSCRIPT, FILING, MODEL, FLAG }
enum PassageSource { TRANSCRIPT, FILING }
enum FlagCategory { THESIS_CONFIRMING, THESIS_CHALLENGING, NOT_MATERIAL }
enum ReviewStatus { PENDING, READING, MODELING, FLAGGING, FINALIZING, COMPLETED, FAILED, STALE }
```

### Events (`ReviewEntity`)

`ReviewCreated`, `ReviewPlanned`, `DispatchProposed`, `PassagesAppended`, `ModelUpdatesAppended`, `FlagsAppended`, `LedgerRevised`, `FlagAccepted`, `FlagRejected`, `FlagScored`, `ReviewCompleted`, `ReviewFailed`, `ReviewFailedStale`.

### Events (`ModelEntity`)

`ModelInitialized`, `MetricUpdated`.

### Events (`SubmissionQueue`)

`ReviewSubmitted { reviewId, ticker, period, submittedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reviews` — body `{ ticker, period, submittedBy? }` → `202 { reviewId }`. Starts a workflow.
- `GET /api/reviews` — list all reviews. Optional `?status=...` filter is applied client-side per Lesson 2.
- `GET /api/reviews/{id}` — one review (full ledger + answer).
- `GET /api/reviews/sse` — server-sent events stream of every review change.
- `GET /api/models/{ticker}` — current internal model for that ticker.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Earnings Reviewer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit `(ticker, period)`, live list of reviews with status pills, expand-row to see the review ledger, the accepted/rejected flag splits, the model diff applied, and the evidence-quality scores.

Browser title: `<title>Akka Sample: Earnings Reviewer</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`before-agent-response` on `ThesisFlagAgent`): every candidate `ThesisFlag` is vetted before it lands on `ReviewAnswer.acceptedFlags`. Three checks: (a) `confidence >= 0.55`, (b) `evidence.size() >= 2` and every cited evidence string matches an actual passage on the ledger, (c) `category` is one of the closed enum values. Blocking. Failure → `FlagRejected` event with the rejection reason; planner is given one opportunity to revise.
- **E1 — on-decision eval-event** (`eval-event`, flavor `on-decision-eval`): each accepted `ThesisFlag` is scored by `EvidenceQualityEvaluator.score(flag, ledger)`. The deterministic rubric awards points for passage count (up to 0.30), source diversity across `TRANSCRIPT` and `FILING` (up to 0.30), recency relative to `period` (up to 0.20), and absence of contradicting passages on the ledger (up to 0.20). The score is written onto the flag and surfaced in the UI; flags scoring below `0.40` are highlighted but not auto-rejected.

## 9. Agent prompts

- `ReviewPlannerAgent` → `prompts/review-planner.md`. Maintains the ledger; decides next step; composes the final answer.
- `TranscriptReaderAgent` → `prompts/transcript-reader.md`. Returns passages.
- `FilingReaderAgent` → `prompts/filing-reader.md`. Returns passages.
- `ModelUpdaterAgent` → `prompts/model-updater.md`. Returns `ModelUpdate` rows.
- `ThesisFlagAgent` → `prompts/thesis-flag.md`. Returns candidate `ThesisFlag` records.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit `(ticker="ACME", period="2026-Q2")`. Review progresses `PENDING → READING → MODELING → FLAGGING → FINALIZING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a ledger with passages from both TRANSCRIPT and FILING sources, at least one `ModelUpdate`, and at least one accepted `ThesisFlag`. `ModelEntity` for `ACME` reflects the applied diff.
2. **J2** — Submit `(ticker="LOWCONF", period="2026-Q2")` against a seeded fixture that produces a candidate flag with `confidence=0.41`. The guardrail rejects the flag; a `FlagRejected` entry appears on the ledger with reason `"confidence 0.41 below floor 0.55"`. The planner is given one revise opportunity. If the revised flag passes, the review completes; otherwise it completes with an empty `acceptedFlags` list and a populated `rejectedFlags` list.
3. **J3** — Submit `(ticker="THIN", period="2026-Q2")` against a seeded fixture whose passages all come from a single source (TRANSCRIPT only, no FILING). The flag is accepted but the eval-event scores it at `0.35`. The UI highlights the flag with a low-confidence-score badge.
4. **J4** — The `EarningsSimulator` drips a request every 90 s; after two minutes the App UI shows at least two reviews even with no manual submissions.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named earnings-reviewer demonstrating the
planner-executor × finance-analysis cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
planner-executor-finance-analysis-earnings-reviewer. Java package
io.akka.samples.earningsreviewer. Akka 3.6.0. HTTP port 9722.

Components to wire (exactly):
- 5 AutonomousAgents:
  * ReviewPlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_ANSWER).maxIterationsPerTask(2)).
    System prompt from prompts/review-planner.md. PLAN returns ReviewLedger.
    DECIDE returns a NextStep tagged union (DispatchDecision | Replan |
    Complete | Fail). COMPOSE_ANSWER returns ReviewAnswer.
  * TranscriptReaderAgent — capability(TaskAcceptance.of(READ_TRANSCRIPT)
    .maxIterationsPerTask(2)). Prompt from prompts/transcript-reader.md.
    Returns SpecialistResult populating passages.
  * FilingReaderAgent — capability(TaskAcceptance.of(READ_FILING)
    .maxIterationsPerTask(2)). Prompt from prompts/filing-reader.md.
    Returns SpecialistResult populating passages.
  * ModelUpdaterAgent — capability(TaskAcceptance.of(UPDATE_MODEL)
    .maxIterationsPerTask(2)). Prompt from prompts/model-updater.md.
    Returns SpecialistResult populating modelUpdates.
  * ThesisFlagAgent — capability(TaskAcceptance.of(RAISE_FLAGS)
    .maxIterationsPerTask(2)). Prompt from prompts/thesis-flag.md.
    Returns SpecialistResult populating candidateFlags.

- 1 Workflow ReviewWorkflow with steps:
  planStep -> [loop entry] proposeStep -> dispatchStep -> recordStep ->
  decideStep -> [back to proposeStep, or to finalizeStep / failStep /
  staleStep]. finalizeStep -> guardrailStep -> evalStep -> applyModelStep
  -> completeStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any specialist call), decideStep ofSeconds(45),
    finalizeStep ofSeconds(60), completeStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(ReviewWorkflow::error)).
  dispatchStep uses switch on DispatchDecision.specialist to call the
  matching specialist agent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(reviewId).result(...).
  recordStep appends the typed payload to the ledger via the matching
  ReviewEntity command (appendPassages, appendModelUpdates, appendFlags).
  decideStep calls forAutonomousAgent(ReviewPlannerAgent.class, DECIDE);
  on Continue or Replan loops; on Complete transitions to finalizeStep;
  on Fail transitions to failStep.
  finalizeStep calls COMPOSE_ANSWER then hands the candidate flags to
  guardrailStep one by one.
  guardrailStep applies FlagGuardrail.vet(flag, ledger). Rejected flags
  emit FlagRejected; accepted flags emit FlagAccepted.
  evalStep applies EvidenceQualityEvaluator.score(flag, ledger) to each
  accepted flag and emits FlagScored.
  applyModelStep applies every accepted ModelUpdate to ModelEntity for
  the review's ticker.
  staleStep is reached when decideStep reads status == STALE; emits
  ReviewFailedStale.

- 1 EventSourcedEntity ReviewEntity holding Review state. emptyState()
  returns Review.initial("", "", "") with no commandContext() reference.
  Commands: createReview, recordPlan, recordDispatch, appendPassages,
  appendModelUpdates, appendFlags, reviseLedger, acceptFlag, rejectFlag,
  scoreFlag, completeReview, failReview, staleFail, getReview. Events as
  listed in SPEC §5.

- 1 EventSourcedEntity ModelEntity keyed by ticker. State InternalModel.
  Commands: initialize(ticker), updateMetric(metric, value, source), get.
  Events: ModelInitialized, MetricUpdated.

- 1 EventSourcedEntity SubmissionQueue with command enqueueReview(reviewId,
  ticker, period, submittedBy) emitting ReviewSubmitted.

- 1 View ReviewView with row type ReviewRow (mirror of Review minus heavy
  ledger payloads — truncate to last 5 passages, last 5 flags; the UI
  fetches the full review by id on click). Table updater consumes
  ReviewEntity events. ONE query getAllReviews SELECT * AS reviews FROM
  review_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer ReviewSubmissionConsumer subscribed to SubmissionQueue events;
  on ReviewSubmitted starts a ReviewWorkflow with reviewId as the workflow
  id.

- 2 TimedActions:
  * EarningsSimulator — every 90s, reads next line from
    src/main/resources/sample-events/review-requests.jsonl and calls
    SubmissionQueue.enqueueReview.
  * StaleReviewMonitor — every 60s, queries ReviewView.getAllReviews,
    filters READING or MODELING or FLAGGING reviews whose createdAt is
    older than 5 minutes, calls ReviewEntity.staleFail; ReviewWorkflow
    polls ReviewEntity.getReview in its decideStep and exits when
    status == STALE.

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /reviews, GET /reviews, GET
    /reviews/{id}, GET /reviews/sse, GET /models/{ticker}, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo ReviewLedger), DECIDE (NextStep), COMPOSE_ANSWER
  (ReviewAnswer).
- SpecialistTasks.java declaring four Task<R> constants: READ_TRANSCRIPT,
  READ_FILING, UPDATE_MODEL, RAISE_FLAGS (all resultConformsTo
  SpecialistResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface
  with permits Continue, Replan, Complete, Fail (each carrying its own
  payload — Continue with DispatchDecision, Replan with revisedLedger,
  Complete with ReviewAnswer stub, Fail with failureReason).
- application/FlagGuardrail.java — deterministic vetter for ThesisFlag.
  Rejects if confidence < 0.55, if evidence.size() < 2, if any evidence
  string does not appear as a substring of any passage on the ledger, or
  if category is not one of the closed enum values. Rejection reasons
  carried back in the FlagRejected event.
- application/EvidenceQualityEvaluator.java — deterministic scorer.
  Score formula:
    0.30 * min(1.0, passageCount/3.0)
    + 0.30 * (sources spanning both TRANSCRIPT and FILING ? 1.0 : 0.5)
    + 0.20 * recencyComponent(period, passages)
    + 0.20 * (no contradicting passages ? 1.0 : 0.0).
  Result in [0, 1].
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9722 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/review-requests.jsonl with 8 canned
  review requests covering ACME, LOWCONF, THIN, BIGCO across two periods.
- src/main/resources/sample-data/transcripts/<ticker>-<period>.txt — six
  short transcript fixtures.
- src/main/resources/sample-data/filings/<ticker>-<period>.txt — six
  short filing fixtures.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/review-planner.md, prompts/transcript-reader.md,
  prompts/filing-reader.md, prompts/model-updater.md,
  prompts/thesis-flag.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Earnings Reviewer",
  one-line pitch, prerequisites (including the integration form's host-
  software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-
  contained HTML file (no ui/ folder, no npm build). Inline CSS + JS.
  Runtime CDN imports for markdown and YAML libs are acceptable. Five
  tabs matching the formal exemplar: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted
  Java snippets), Risk Survey (7 sub-tabs from governance.html with
  answers populated from risk-survey.yaml; unanswered .qb opacity 0.45),
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows), App UI (submit form + live review list
  with status pills, expand-on-click for ledger, accepted/rejected
  flags, model diff, and evidence-quality score badges). Browser title
  exactly: <title>Akka Sample: Earnings Reviewer</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
  for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that
        returns random-but-shape-correct outputs per agent (see Mock LLM
        provider block below for per-agent shapes). Sets model-provider
        = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the
        Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-
        local .akka-build.yaml; /akka:build sources the file before
        spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://,
        vault://; recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session
        memory; passed to the JVM via the MCP tool's environment
        parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry
  in application.conf, no secrets.yaml, no .akka/ file with key material.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java
  implementing the ModelProvider interface with per-agent dispatch on
  the agent class name and the Task<R> id. Each branch reads a JSON file
  from src/main/resources/mock-responses/<agent-name>.json (one file per
  agent: review-planner.json, transcript-reader.json, filing-reader.json,
  model-updater.json, thesis-flag.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    review-planner.json — three sections keyed by task id:
      "PLAN" → 4 ReviewLedger entries (passages empty, modelUpdates
        empty, flags empty, currentDispatch null).
      "DECIDE" → 6 NextStep entries covering Continue (with
        DispatchDecision across all four specialists), Replan, Complete,
        Fail. The Continue entries advance through a plausible review
        narrative when iterated in order.
      "COMPOSE_ANSWER" → 4 ReviewAnswer stubs with 1–3 candidate flags,
        2–4 ModelUpdate rows, and a 60–120 word narrative.
    transcript-reader.json — 6 SpecialistResult entries, ok=true,
      passages populated with 2–4 short excerpts from the seeded
      transcript fixtures.
    filing-reader.json — 6 SpecialistResult entries with 2–4 passages
      from the seeded filing fixtures.
    model-updater.json — 6 SpecialistResult entries with 2–4
      ModelUpdate rows each (metric ∈ {revenue, gross_margin, guidance,
      headcount, capex}).
    thesis-flag.json — 6 SpecialistResult entries. ONE entry must
      surface a flag with confidence=0.41 so the J2 guardrail test
      fires. ONE entry must surface flags whose passages all come from
      TRANSCRIPT so the J3 eval-event test fires (score around 0.35).
- A MockModelProvider.seedFor(reviewId) helper makes the selection
  deterministic per review id so the same request in dev produces the
  same output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent — the
  base class clause must read `extends AutonomousAgent` for every agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep,
  decideStep, finalizeStep, completeStep).
- Lesson 6: Optional<T> for every nullable field on a View row record
  and on the Review entity state (ledger, answer, failureReason,
  finishedAt).
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  SpecialistTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9722 in application.conf — picked from the
  available range.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string
  "runs-out-of-the-box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no
  key value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM —
  delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
