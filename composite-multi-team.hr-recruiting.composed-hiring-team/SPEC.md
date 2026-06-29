# SPEC — composed-hiring-workflow

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Composed Hiring Workflow.
**One-line pitch:** Submit a job application; a hiring-manager agent opens a brief, a screening desk delegates to several screeners and synthesises their notes, a CV improvement loop runs a coach-critic cycle to strengthen the candidate's profile, an interview panel scores the candidate, and a passing verdict produces an offer that is approved and extended.

## 2. What this blueprint demonstrates

The **composite-multi-team** coordination pattern wired with Akka's first-party primitives: a top-level `HiringTeamWorkflow` delegates four stages — screening, CV improvement, interview, offer — and two of those stages are themselves **nested workflows** (`CandidateWorkflow` and `CvImprovementLoop`). The screening desk uses **delegation**: a screening lead assigns criteria and fans out one screener per dimension, then synthesises their notes. The CV improvement desk uses a **team feedback loop**: a `CvCoach` rewrites the candidate's CV and a `CvCritic` scores it; the loop repeats until the critic passes or the iteration cap is reached. The interview desk uses **moderation**: a panel of interviewer instances each score the candidate on one competency axis, and a deterministic rule turns the panel's scores into a `PanelVerdict`. The blueprint also demonstrates four governance mechanisms a hiring pipeline needs: a **before-agent-invocation guardrail** at every nested workflow boundary, a non-blocking **stage eval** on each stage result, a **before-agent-response guardrail** on the final offer letter, and a non-blocking **post-hire compliance review** of extended offers.

## 3. User-facing flows

The user opens the App UI tab and submits an application via the form (a candidate name, a job role, and a CV text).

1. The system logs the submission on `ApplicationQueue` and a `Consumer` starts a `HiringTeamWorkflow` for the application.
2. The `HiringManager` agent opens the requisition — it produces a `HiringBrief` (a role summary, required dimensions, and target competencies). The application moves to `OPENED`.
3. **Screening desk (delegation).** A `CandidateWorkflow` is started as a nested workflow. The `ScreeningLead` plans a `ScreeningPlan` (a list of dimensions). The top-level workflow runs one `Screener` instance per dimension; each screener writes a `ScreeningNote` into the shared workspace through `ApplicationTools`. The lead synthesises the notes into a `ScreeningReport`. The application moves to `SCREENED`. The `CandidateWorkflow` completes and returns the report to `HiringTeamWorkflow`.
4. **CV improvement loop (team feedback loop).** A `CvImprovementLoop` is started as a nested workflow. The `CvCoach` produces an improved `CvDraft` from the original CV text and the screening report. The `CvCritic` scores the draft on fit, clarity, and completeness, producing a `CritiqueNote`. If the critique passes or the iteration cap (3) is reached, the loop ends and returns the best `CvDraft` to `HiringTeamWorkflow`. Otherwise the coach revises and the critic re-scores. The application moves to `CV_IMPROVED`.
5. **Interview desk (moderation).** The workflow runs the interview panel (`technical`, `behavioural`, `cultural`) — one `Interviewer` instance per axis, each writing an `InterviewScore` (a per-axis `PROCEED`/`DECLINE` verdict and comments). A deterministic `PanelRule` aggregates the scores into a `PanelVerdict`: `PROCEED` only when every axis passes, otherwise `REASSESS` with the list of competencies to address. The application moves to `INTERVIEWING` then to `SHORTLISTED` or back to `CV_IMPROVING` for one re-assessment round.
6. **Offer.** On a `PROCEED` verdict the workflow runs `HiringManager` once more to draft the final `OfferLetter`. A before-agent-response guardrail vets the letter before it is persisted. The application moves to `OFFER_PENDING`, and on a successful guardrail check moves to `OFFER_EXTENDED` with a generated offer reference.
7. A `StageEvalConsumer` fires a non-blocking quality eval on each stage result (screening report, improved CV, panel verdict) and records a `StageEval` on the application.
8. After an application is `OFFER_EXTENDED`, a compliance officer can post a `PostHireReview` (cleared or flagged, with comments). It is recorded against the extended offer without changing the application state.

A `ApplicationSimulator` (TimedAction) drips a sample application every 90 seconds. A `StaleDimensionMonitor` (TimedAction) releases any screening dimension that has been claimed but idle for more than three minutes, returning it to `OPEN`.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `HiringManager` | `AutonomousAgent` | Opens the requisition (produces `HiringBrief`) and, at offer, drafts the final `OfferLetter` under an output guardrail. | `HiringTeamWorkflow` | returns typed result to workflow; writes via `ApplicationEntity` |
| `ScreeningLead` | `AutonomousAgent` | Plans screening dimensions and synthesises the screeners' notes into a `ScreeningReport`. | `CandidateWorkflow` | returns `ScreeningPlan` / `ScreeningReport` |
| `Screener` | `AutonomousAgent` | Evaluates one dimension; writes a `ScreeningNote` via `ApplicationTools`. Run as several instances (one per dimension). | `CandidateWorkflow` | `ApplicationTools` → `ApplicationEntity` |
| `CvCoach` | `AutonomousAgent` | Produces an improved `CvDraft` from the original CV and screening report. | `CvImprovementLoop` | returns `CvDraft`; writes via `CvEntity` |
| `CvCritic` | `AutonomousAgent` | Scores the current `CvDraft` on fit, clarity, and completeness; returns a `CritiqueNote`. | `CvImprovementLoop` | returns `CritiqueNote` |
| `Interviewer` | `AutonomousAgent` | Scores the candidate on one competency axis; returns an `InterviewScore`. Run as several instances (`technical`, `behavioural`, `cultural`). | `HiringTeamWorkflow` | `ApplicationTools` → `ApplicationEntity` |
| `HiringTeamWorkflow` | `Workflow` | Top-level pipeline: open → screen → improve CV → interview → offer, with a re-assessment loop. Starts and awaits nested workflows. | `ApplicationRequestConsumer` | all agents, `ApplicationEntity`, `CvEntity`, nested workflows |
| `CandidateWorkflow` | `Workflow` | Nested workflow for one candidate's screening: plan dimensions → fan out screeners → synthesise report. One instance per applicationId. | `HiringTeamWorkflow` | `ScreeningLead`, `Screener`, `ApplicationEntity`, `ApplicationTools` |
| `CvImprovementLoop` | `Workflow` | Nested workflow running the coach-critic feedback loop: coach → critic → repeat up to 3 iterations. One instance per applicationId. | `HiringTeamWorkflow` | `CvCoach`, `CvCritic`, `CvEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | The shared application workspace: one candidate's lifecycle, brief, screening notes, improved CV, interview scores, offer, stage evals, post-hire review. | `HiringTeamWorkflow`, `ApplicationTools`, `StageEvalConsumer`, `HiringEndpoint` | `ApplicationBoardView` |
| `CvEntity` | `EventSourcedEntity` | Tracks CV improvement iterations: original CV, each draft, the final accepted draft. | `CvImprovementLoop`, `ApplicationTools` | `CvBoardView` |
| `ApplicationQueue` | `EventSourcedEntity` | Single instance; logs each submitted application for replay/audit. | `HiringEndpoint`, `ApplicationSimulator` | `ApplicationRequestConsumer` |
| `ApplicationBoardView` | `View` | List-of-applications read model the UI streams. | `ApplicationEntity` events | `HiringEndpoint` |
| `CvBoardView` | `View` | CV iteration board the UI shows. | `CvEntity` events | `HiringTeamWorkflow`, `HiringEndpoint` |
| `ApplicationRequestConsumer` | `Consumer` | Listens to `ApplicationQueue` events; starts a `HiringTeamWorkflow` per submission. | `ApplicationQueue` events | `HiringTeamWorkflow`, `ApplicationEntity` |
| `StageEvalConsumer` | `Consumer` | Listens to `ApplicationEntity` stage events; runs a deterministic eval and records a `StageEval`. | `ApplicationEntity` events | `ApplicationEntity` |
| `ApplicationSimulator` | `TimedAction` | Drips a sample application every 90 s. | scheduler | `ApplicationQueue` |
| `StaleDimensionMonitor` | `TimedAction` | Every 60 s, releases screening dimensions claimed but idle > 3 min back to `OPEN`. | scheduler | `ApplicationEntity` |
| `HiringEndpoint` | `HttpEndpoint` | `/api/*` — submit, get application, list applications, list CV iterations, post-hire review, SSE, metadata. | — | `ApplicationQueue`, `ApplicationBoardView`, `CvBoardView`, `ApplicationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts `HiringTeamWorkflow` instances on restart as needed. | — | scheduler |

Companion classes (not Akka components): `HiringTasks` (the `Task<R>` constants), `ApplicationTools` (the function tool agents call to write into the workspace; the before-agent-invocation guardrail vets it), and `PanelRule` (deterministic pure function aggregating interview scores into a verdict).

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record ApplicationBrief(String applicationId, String candidateName, String jobRole, String cvText) {}

record HiringBrief(String roleSummary, List<String> requiredDimensions, List<String> targetCompetencies) {}

record ScreeningPlan(List<String> dimensions) {}
record ScreeningNote(String dimension, ScreeningOutcome outcome, String comments) {}
record ScreeningReport(String summary, List<ScreeningNote> notes, ScreeningOutcome overallOutcome) {}

record CvDraft(String candidateName, String jobRole, String revisedCvText, int iteration, List<String> changesApplied) {}
record CritiqueNote(int iteration, CritiqueOutcome outcome, int fitScore, int clarityScore, int completenessScore, List<String> suggestions) {}

record InterviewScore(String axis, InterviewOutcome outcome, int score, String comments) {}
record PanelVerdict(InterviewOutcome outcome, List<InterviewScore> scores, List<String> reassessCompetencies) {}

record OfferLetter(String candidateName, String jobRole, String offerText, String offerReference, Instant draftedAt) {}

record StageEval(String stage, int score, List<String> flags, Instant evaluatedAt) {}
record PostHireReview(String reviewedBy, PostHireOutcome outcome, String comments, Instant reviewedAt) {}
```

### Entity state — `Application` (state of `ApplicationEntity`, basis of the board row)

```java
record Application(
    String applicationId,
    String candidateName,
    String jobRole,
    String cvText,
    ApplicationStatus status,
    Optional<HiringBrief>     hiringBrief,
    Optional<ScreeningReport> screeningReport,
    Optional<CvDraft>         acceptedCvDraft,
    List<String>              dimensionIds,
    Optional<PanelVerdict>    panelVerdict,
    Optional<OfferLetter>     offerLetter,
    Optional<String>          offerReference,
    Optional<Instant>         offerExtendedAt,
    List<StageEval>           stageEvals,
    Optional<PostHireReview>  postHireReview,
    int                       reassessCount,
    Instant                   createdAt
) {}

enum ApplicationStatus {
  SUBMITTED, OPENED, SCREENING, SCREENED, CV_IMPROVING, CV_IMPROVED,
  INTERVIEWING, SHORTLISTED, OFFER_PENDING, OFFER_EXTENDED, DECLINED
}
```

### Entity state — `CvIteration` (state of `CvEntity`, basis of the CV board row)

```java
record CvIteration(
    String applicationId,
    String originalCvText,
    List<CvDraft>         drafts,
    Optional<CvDraft>     acceptedDraft,
    Optional<CritiqueNote> latestCritique,
    int                   iterationCount,
    Instant               createdAt
) {}

enum ScreeningOutcome  { PASS, FAIL, PENDING }
enum CritiqueOutcome   { PASS, REVISE }
enum InterviewOutcome  { PROCEED, DECLINE, REASSESS }
enum PostHireOutcome   { CLEARED, FLAGGED }
```

### Events

`ApplicationEntity`: `ApplicationCreated`, `BriefOpened`, `ScreeningCompleted`, `CvImproved`, `InterviewCompleted`, `ReassessRequested`, `OfferDrafted`, `OfferExtended`, `OfferBlocked`, `StageEvaluated`, `PostHireReviewRecorded`.
`CvEntity`: `CvCreated`, `DraftAdded`, `DraftAccepted`.
`ApplicationQueue`: `ApplicationSubmitted`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/applications` — body `{ candidateName, jobRole, cvText, requestedBy? }` → `202 { applicationId }`. Logs the submission and starts the pipeline.
- `GET /api/applications` — list all applications. Optional `?status=...`, filtered client-side.
- `GET /api/applications/{id}` — one application with its brief, report, CV draft, verdict, offer, stage evals, and post-hire review.
- `GET /api/applications/sse` — server-sent events stream of every application change.
- `GET /api/applications/{id}/cv-iterations` — the CV iteration history for one application.
- `GET /api/cv/sse` — server-sent events stream of every CV draft change.
- `POST /api/applications/{id}/post-hire-review` — body `{ reviewedBy, outcome, comments }`. Records a post-hire review; allowed only when the application is `OFFER_EXTENDED`. Non-blocking.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Composed Hiring Workflow</title>`.

- **Overview** — eyebrow "Overview" + headline "Composed Hiring Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, application state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — an application submission form; a live application board grouped by status with per-application cards showing the hiring brief, the screening report, the CV improvement progress, the interview verdict, the stage-eval scores, and (when extended) the offer letter and a post-hire review box; plus a CV board panel grouped by improvement iteration.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** (`before-agent-invocation` hook, on nested workflow boundaries): before each nested workflow (`CandidateWorkflow`, `CvImprovementLoop`) is started, a guardrail checks that the application is in the correct predecessor status, that the agent roster passed to the nested workflow is non-empty, and that the required upstream result (e.g., `HiringBrief` present before screening) is populated. A failing check blocks the nested workflow from starting and records the reason on the application. Blocking.
- **G2 — before-agent-response guardrail** (`before-agent-response` hook, on `HiringManager`): when the hiring manager drafts the final `OfferLetter`, a guardrail vets the response before it is persisted — it checks minimum letter length, requires the offer reference field to be non-empty, and refuses letters containing prohibited compensation language. A failing letter blocks the offer and the application stays `OFFER_PENDING` with the guardrail reason recorded. Blocking.
- **E1 — stage eval** (`eval-event`, `on-decision-eval` flavor): `StageEvalConsumer` fires on each stage result event (`ScreeningCompleted`, `CvImproved`, `InterviewCompleted`) and runs a deterministic quality eval — a score and a list of flags — recorded as a `StageEval` on the application. Non-blocking.
- **HO1 — post-hire compliance review** (`hotl`, `live-compliance-review` flavor): after an offer is extended, a compliance officer can post a `PostHireReview` through `POST /api/applications/{id}/post-hire-review`. It is recorded against the application without changing the `OFFER_EXTENDED` status. Non-blocking, system-level.

## 9. Agent prompts

- `HiringManager` → `prompts/hiring-manager.md`. Opens the requisition brief and drafts the final offer letter.
- `ScreeningLead` → `prompts/screening-lead.md`. Plans screening dimensions and synthesises the notes.
- `Screener` → `prompts/screener.md`. Evaluates one screening dimension.
- `CvCoach` → `prompts/cv-coach.md`. Produces an improved CV draft.
- `CvCritic` → `prompts/cv-critic.md`. Scores a CV draft and returns revision suggestions.
- `Interviewer` → `prompts/interviewer.md`. Scores the candidate on one competency axis.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an application; the hiring manager opens a brief; the screening desk delegates and synthesises; the CV improvement loop runs; the interview panel scores; a `PROCEED` verdict drafts and extends an offer. The board updates live via SSE.
2. **J2** — Two screeners claim the same dimension concurrently; each dimension ends up evaluated by exactly one screener.
3. **J3** — The panel returns a `REASSESS` verdict; the application loops back for one CV improvement round, then proceeds to offer on the second pass.
4. **J4** — A nested workflow is invoked when its pre-condition is not met; the before-agent-invocation guardrail blocks the call and records the reason.
5. **J5** — After an offer is extended, a compliance officer posts a post-hire review; it is recorded without changing the offer state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named composed-hiring-workflow demonstrating the
composite-multi-team × hr-recruiting cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
composite-multi-team-hr-recruiting-composed-hiring-team. Java package
io.akka.samples.composedhiringworkflow. Akka 3.6.0. HTTP port 9828.

Components to wire (exactly):
- 6 AutonomousAgents (each extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgrade to Agent):
  * HiringManager — definition() with capability(TaskAcceptance.of(OPEN_REQ)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(DRAFT_OFFER)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/hiring-manager.md.
    OPEN_REQ returns HiringBrief{roleSummary, requiredDimensions,
    targetCompetencies}. DRAFT_OFFER returns OfferLetter{candidateName, jobRole,
    offerText, offerReference, draftedAt}. Register a before-agent-response
    guardrail on this agent that validates the DRAFT_OFFER output (control G2).
  * ScreeningLead — capability(TaskAcceptance.of(PLAN_SCREENING)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(SYNTHESIZE_SCREENING)
    .maxIterationsPerTask(2)). System prompt from prompts/screening-lead.md.
    PLAN_SCREENING returns ScreeningPlan{dimensions}. SYNTHESIZE_SCREENING returns
    ScreeningReport{summary, notes, overallOutcome}.
  * Screener — capability(TaskAcceptance.of(SCREEN_DIMENSION)
    .maxIterationsPerTask(2)). System prompt from prompts/screener.md. Returns
    ScreeningNote{dimension, outcome, comments}. Run as several runtime instances
    addressed by instanceId (applicationId + "-d" + dimensionIndex) — ONE agent
    class, several instance ids. Equipped with ApplicationTools function tool; the
    G1 before-agent-invocation guardrail is registered on this agent.
  * CvCoach — capability(TaskAcceptance.of(IMPROVE_CV).maxIterationsPerTask(3)).
    System prompt from prompts/cv-coach.md. Returns CvDraft{candidateName, jobRole,
    revisedCvText, iteration, changesApplied}. ONE agent class per applicationId;
    called iteratively by CvImprovementLoop. Equipped with ApplicationTools; the
    G1 before-agent-invocation guardrail is registered on this agent.
  * CvCritic — capability(TaskAcceptance.of(CRITIQUE_CV).maxIterationsPerTask(2)).
    System prompt from prompts/cv-critic.md. Returns CritiqueNote{iteration,
    outcome, fitScore, clarityScore, completenessScore, suggestions}. ONE agent
    class per applicationId, called once per iteration by CvImprovementLoop.
  * Interviewer — capability(TaskAcceptance.of(INTERVIEW_AXIS)
    .maxIterationsPerTask(2)). System prompt from prompts/interviewer.md. Returns
    InterviewScore{axis, outcome, score, comments}. Run as several runtime instances
    (technical, behavioural, cultural) — ONE agent class, several instance ids; the
    axis is passed as the instruction. Equipped with ApplicationTools; the G1
    before-agent-invocation guardrail is registered on this agent.

- 3 Workflows:
  * HiringTeamWorkflow, one instance per applicationId. State carries applicationId,
    candidateName, jobRole, cvText, brief (Optional), screeningReport (Optional),
    acceptedCvDraft (Optional), panelVerdict (Optional), reassessCount, and stage.
    Steps: openStep -> screenStep -> improveStep -> interviewStep -> offerStep,
    with a re-assessment path when the panel returns REASSESS.
    * openStep (stepTimeout 60s): call forAutonomousAgent(HiringManager.class,
      applicationId).runSingleTask(OPEN_REQ.instructions(jobRole, cvText)) then
      result(OPEN_REQ); call ApplicationEntity.openBrief(brief). Go to screenStep.
    * screenStep (stepTimeout 180s): start CandidateWorkflow(applicationId);
      await its completion via the nested workflow pattern; receive ScreeningReport.
      Call ApplicationEntity.recordScreeningReport(report). Go to improveStep.
    * improveStep (stepTimeout 180s): start CvImprovementLoop(applicationId);
      await its completion; receive acceptedCvDraft. Call
      ApplicationEntity.recordImprovedCv(cvDraft). Go to interviewStep.
    * interviewStep (moderation, stepTimeout 120s): for each axis in [technical,
      behavioural, cultural] call forAutonomousAgent(Interviewer.class,
      applicationId + "-" + axis).runSingleTask(INTERVIEW_AXIS.instructions(axis,
      acceptedCvDraft)) then result(INTERVIEW_AXIS); each interviewer writes its
      InterviewScore via ApplicationTools. Pass scores through PanelRule ->
      PanelVerdict. Call ApplicationEntity.recordPanelVerdict(verdict). On PROCEED
      go to offerStep. On REASSESS, if reassessCount < 1, call
      ApplicationEntity.requestReassess(reassessCompetencies), increment
      reassessCount, and go back to improveStep. If reassessCount already 1, accept
      the current state and go to offerStep (one bounded round).
    * offerStep (stepTimeout 60s): call HiringManager DRAFT_OFFER -> OfferLetter
      (guarded by G2). If the guardrail refuses, call
      ApplicationEntity.recordOfferBlock(reason) and end the workflow with the
      application left OFFER_PENDING. Otherwise call
      ApplicationEntity.extendOffer(offerLetter) with
      offerReference = "OFFER-" + applicationId. End the workflow.
    Override settings() with stepTimeout(openStep, 60s),
    stepTimeout(screenStep, 180s), stepTimeout(improveStep, 180s),
    stepTimeout(interviewStep, 120s), stepTimeout(offerStep, 60s).

  * CandidateWorkflow, one instance per applicationId. State carries applicationId,
    dimensionIds, and screeningReport (Optional). Steps:
    planStep -> screenStep -> synthesiseStep.
    * planStep (stepTimeout 60s): call ScreeningLead PLAN_SCREENING ->
      ScreeningPlan. Write one dimension slot per dimension with deterministic id
      = applicationId + "-d" + index (status OPEN) on ApplicationEntity. Go to
      screenStep.
    * screenStep (stepTimeout 120s): for each dimension call
      forAutonomousAgent(Screener.class, applicationId + "-d" + i)
      .runSingleTask(SCREEN_DIMENSION.instructions(dimension, cvText)); each screener
      writes its ScreeningNote via ApplicationTools. Await all results.
    * synthesiseStep (stepTimeout 60s): call ScreeningLead SYNTHESIZE_SCREENING
      over notes -> ScreeningReport. Call ApplicationEntity.recordScreeningReport.
      End workflow, return ScreeningReport.

  * CvImprovementLoop, one instance per applicationId. State carries applicationId,
    originalCvText, currentDraft (Optional), latestCritique (Optional),
    iterationCount. Steps: coachStep -> criticStep -> decideStep.
    * coachStep (stepTimeout 90s): call forAutonomousAgent(CvCoach.class,
      applicationId).runSingleTask(IMPROVE_CV.instructions(originalCvText,
      latestCritique, iterationCount)). Returns CvDraft. Call
      CvEntity.addDraft(draft). Go to criticStep.
    * criticStep (stepTimeout 60s): call forAutonomousAgent(CvCritic.class,
      applicationId).runSingleTask(CRITIQUE_CV.instructions(currentDraft)) ->
      CritiqueNote. Go to decideStep.
    * decideStep: if critique.outcome == PASS or iterationCount >= 3, call
      CvEntity.acceptDraft(currentDraft) and end workflow returning currentDraft.
      Otherwise increment iterationCount and go back to coachStep.

- 2 EventSourcedEntities:
  * ApplicationEntity holding Application state, one per applicationId. Commands:
    createApplication, openBrief, recordScreeningReport, recordImprovedCv,
    recordPanelVerdict, requestReassess, extendOffer, recordOfferBlock,
    recordStageEval, recordPostHireReview, appendScreeningNote, appendInterviewScore,
    getApplication. ApplicationStatus enum: SUBMITTED, OPENED, SCREENING, SCREENED,
    CV_IMPROVING, CV_IMPROVED, INTERVIEWING, SHORTLISTED, OFFER_PENDING,
    OFFER_EXTENDED, DECLINED. Events: ApplicationCreated, BriefOpened,
    ScreeningCompleted, CvImproved, InterviewCompleted, ReassessRequested,
    OfferDrafted, OfferExtended, OfferBlocked, StageEvaluated,
    PostHireReviewRecorded. emptyState() returns Application.initial("", "", "") with
    NO commandContext() reference. recordPostHireReview is accepted only when
    status==OFFER_EXTENDED.
  * CvEntity holding CvIteration state, one per applicationId. Commands: createCv,
    addDraft, acceptDraft, getCvIteration. Events: CvCreated, DraftAdded,
    DraftAccepted. emptyState() returns CvIteration.initial("") with NO
    commandContext() reference.
  * ApplicationQueue, single instance "default". Command
    submitApplication(ApplicationBrief) emitting
    ApplicationSubmitted{applicationId, candidateName, jobRole, submittedAt}.

- 2 Views:
  * ApplicationBoardView with row type ApplicationRow (mirrors Application but drops
    heavy OfferLetter.offerText — keeps brief roleSummary, report summary,
    acceptedCvDraft iteration count, panelVerdict outcome, stageEvals,
    offerReference, postHireReview). Table updater consumes ApplicationEntity events.
    ONE query getAllApplications SELECT * AS applications FROM application_board. No
    WHERE status filter (Lesson 2); callers filter client-side. Add a
    streamAllApplications query for the SSE endpoint.
  * CvBoardView with row type CvRow (mirrors CvIteration minus the full draft text —
    keeps iterationCount and a draftPresent boolean). Table updater consumes
    CvEntity events. ONE query getAllCvIterations SELECT * AS cvIterations FROM
    cv_board. Add streamAllCvIterations for SSE.

- 2 Consumers:
  * ApplicationRequestConsumer subscribed to ApplicationQueue events; on
    ApplicationSubmitted calls ApplicationEntity.createApplication then starts a
    HiringTeamWorkflow with the applicationId as the workflow id.
  * StageEvalConsumer subscribed to ApplicationEntity events; on
    ScreeningCompleted, CvImproved, and InterviewCompleted it runs a deterministic
    StageEvaluator (pure function, NOT an LLM call) producing a
    StageEval{stage, score, flags, evaluatedAt} and calls
    ApplicationEntity.recordStageEval(eval). Ignore all other events. This is
    control E1.

- 2 TimedActions:
  * ApplicationSimulator — every 90s, reads the next line from
    src/main/resources/sample-events/sample-applications.jsonl (wraps when
    exhausted) and calls ApplicationQueue.submitApplication.
  * StaleDimensionMonitor — every 60s, queries ApplicationBoardView
    .getAllApplications, finds applications in SCREENING with any dimension claim
    idle > 3 minutes, and calls ApplicationEntity.releaseScreeningDimension to
    reset the dimension slot to OPEN.

- 2 HttpEndpoints:
  * HiringEndpoint at /api with POST /applications, GET /applications (client-side
    filter by status), GET /applications/{id}, GET /applications/sse,
    GET /applications/{id}/cv-iterations, GET /cv/sse,
    POST /applications/{id}/post-hire-review (accepted only when the application is
    OFFER_EXTENDED; returns 409 otherwise), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

- 1 service-setup Bootstrap: schedule ApplicationSimulator and
  StaleDimensionMonitor.

Companion files:
- HiringTasks.java declaring the Task<R> constants: OPEN_REQ (resultConformsTo
  HiringBrief), DRAFT_OFFER (OfferLetter), PLAN_SCREENING (ScreeningPlan),
  SCREEN_DIMENSION (ScreeningNote), SYNTHESIZE_SCREENING (ScreeningReport),
  IMPROVE_CV (CvDraft), CRITIQUE_CV (CritiqueNote), INTERVIEW_AXIS (InterviewScore).
- ApplicationTools.java — the function tool agents call to write into the shared
  workspace: appendScreeningNote(applicationId, note),
  appendInterviewScore(applicationId, score), recordCvDraft(applicationId, draft).
  Each method routes through ApplicationEntity / CvEntity. The G1 before-agent-
  invocation guardrail vets every call: refuse when the applicationId is not the
  one the agent was assigned, when the target application is already OFFER_EXTENDED
  or DECLINED, or when the payload exceeds a size cap.
- PanelRule.java (deterministic, in application/) — a pure function over a
  List<InterviewScore> returning a PanelVerdict: outcome is PROCEED only when every
  score has outcome PROCEED; outcome is REASSESS when at least one is REASSESS and
  none are DECLINE; otherwise DECLINE. reassessCompetencies = the axes with
  REASSESS outcome. NOT an LLM call.
- StageEvaluator.java (deterministic, in application/) — a pure function used by
  StageEvalConsumer: scores a stage result 0-100 and lists flags (e.g., screening
  report with overall FAIL, CV draft with no changesApplied, panel with any DECLINE
  axis). NOT an LLM call.
- Domain records ApplicationBrief, HiringBrief, ScreeningPlan, ScreeningNote,
  ScreeningReport, CvDraft, CritiqueNote, InterviewScore, PanelVerdict, OfferLetter,
  StageEval, PostHireReview, CvIteration, and the enums ApplicationStatus,
  ScreeningOutcome, CritiqueOutcome, InterviewOutcome, PostHireOutcome in domain/.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9828 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also
  composed-hiring-workflow.screening-dimensions = ["skills", "experience",
  "education"] and
  composed-hiring-workflow.interview-axes = ["technical","behavioural","cultural"]
  read by Bootstrap and HiringTeamWorkflow.
- src/main/resources/sample-events/sample-applications.jsonl with 6 canned
  applications (each a candidateName + jobRole + short cvText — e.g.,
  "Alice Chen / Senior Software Engineer / 5 years Java, distributed systems",
  "Bob Ramirez / Product Manager / 3 years B2B SaaS, roadmapping",
  "Clara White / Data Scientist / PhD statistics, Python ML pipelines",
  "David Osei / DevOps Engineer / Kubernetes, CI/CD, GCP",
  "Elena Kowalski / UX Designer / Figma, user research, design systems",
  "Frank Nguyen / Solutions Architect / cloud-native, customer-facing design").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (G1 before-agent-
  invocation guardrail, G2 before-agent-response guardrail, E1 eval-event
  on-decision-eval, HO1 hotl live-compliance-review) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/hiring-manager.md, prompts/screening-lead.md, prompts/screener.md,
  prompts/cv-coach.md, prompts/cv-critic.md, prompts/interviewer.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Composed Hiring Workflow",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sections with answers from risk-survey.yaml; unanswered .qb
  opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source
  table with click-to-expand rows and colored mechanism pill), App UI (application
  submission form + application board grouped by ApplicationStatus showing brief/
  screening report/CV iteration count/panel verdict/stage-eval scores/offer/
  post-hire review box, and a CV board panel grouped by iteration). Browser title
  exactly: <title>Akka Sample: Composed Hiring Workflow</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class name.
  Per-agent mock-response shapes:
    hiring-manager.json — 4-6 entries. OPEN_REQ variants are HiringBrief with a
      roleSummary, 3-4 requiredDimensions, and 3-4 targetCompetencies. DRAFT_OFFER
      variants are OfferLetter with a candidateName, jobRole, offerText, and a
      non-empty offerReference. Include 1 DRAFT_OFFER entry with an empty
      offerReference so the G2 guardrail blocks it.
    screening-lead.json — 4-6 entries. PLAN_SCREENING variants are ScreeningPlan
      with 3-4 dimensions. SYNTHESIZE_SCREENING variants are ScreeningReport with
      a summary, notes list, and overallOutcome of PASS.
    screener.json — 4-6 ScreeningNote entries, each with a dimension, a PASS
      outcome, and short comments. Include 1 entry targeting a mismatched
      applicationId to exercise the G1 before-agent-invocation guardrail.
    cv-coach.json — 4-6 CvDraft entries each with revised text, iteration 1,
      and 2-3 changesApplied items. Include 1 entry with an empty revisedCvText to
      trigger a no-op iteration.
    cv-critic.json — 6 CritiqueNote entries. Most have outcome PASS. Include at
      least 1 REVISE note with suggestions for the revision-loop journey.
    interviewer.json — 6-8 InterviewScore entries spanning the three axes. Most
      have outcome PROCEED. Include at least 1 REASSESS note per the re-assessment
      journey, naming a competency in its comments.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion HiringTasks Task<R> constants (Lesson 7).
- Views have no WHERE filter on the enum status column; filter client-side
  (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9828 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26).
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export
  block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text (Lessons 21, 22, 23).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
