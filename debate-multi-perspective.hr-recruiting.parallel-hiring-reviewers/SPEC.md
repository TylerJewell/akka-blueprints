# SPEC — parallel-hiring-reviewers

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Parallel Multi-Reviewer Workflow.
**One-line pitch:** Submit a candidate profile; a panel coordinator briefs an HR reviewer, a hiring manager reviewer, and a team reviewer, runs them in parallel over the same profile, and aggregates their perspectives into one hiring recommendation with per-axis detail.

## 2. What this blueprint demonstrates

The **debate-multi-perspective** coordination pattern wired with Akka's first-party primitives: a Workflow asks one coordinator agent to brief three specialist reviewer agents, runs those reviewers in parallel over the same candidate profile, then asks the coordinator to aggregate three independent perspectives into a single hiring recommendation. The blueprint also demonstrates two governance mechanisms — a **special-category sanitizer** that redacts protected attributes (race, gender, age, marital status, national origin) from the candidate profile before any reviewer reads it, and an **eval-periodic fairness watch** that samples completed evaluations to detect whether one reviewer role consistently drives harsher or more lenient outcomes than the others.

## 3. User-facing flows

The user opens the App UI tab and submits a candidate profile (name, role applied for, résumé text) via the form.

1. The system creates a `CandidateEvaluation` record in `INTAKE` and starts a `ReviewWorkflow`.
2. A deterministic special-category sanitizer redacts protected attributes — inferred gender markers, age indicators, marital status phrases, national-origin cues, and race/ethnicity markers — from the résumé text. Only the redacted profile is persisted and passed downstream; the raw text is never stored. The evaluation moves to `EVALUATING`.
3. The PanelCoordinator decomposes the profile into three axis briefs: an HR compliance focus, a manager role-fit focus, and a team compatibility focus.
4. The workflow forks: `HrReviewer`, `ManagerReviewer`, and `TeamReviewer` run concurrently over the redacted profile. Each returns a typed `PerspectiveReview` with a per-axis recommendation, a score, and a list of findings.
5. The PanelCoordinator aggregates the three perspective reviews into a `HiringRecommendation { decision, rationale, perspectiveReviews }`.
6. An output guardrail vets the aggregated recommendation; if it fails, the evaluation moves to `BLOCKED`. Otherwise, the evaluation moves to `DECIDED`.
7. If any reviewer times out after 60 seconds, the workflow short-circuits: the PanelCoordinator aggregates from whichever perspectives returned, and the evaluation enters `DEGRADED`.
8. Every ten minutes, a fairness sampler picks one decided evaluation and asks a judge whether the three reviewer perspectives drove the recommendation proportionally, attaching a 1–5 fairness score.

A `CandidateSimulator` (TimedAction) drips a sample candidate profile every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PanelCoordinator` | `AutonomousAgent` | Decomposes the candidate profile into three axis briefs; later aggregates the three perspective reviews into one hiring recommendation. | `ReviewWorkflow` | returns typed result to workflow |
| `HrReviewer` | `AutonomousAgent` | Assesses compliance posture, compensation band fit, and protected-attribute neutrality of the evaluation. | `ReviewWorkflow` | — |
| `ManagerReviewer` | `AutonomousAgent` | Assesses role fit, skill alignment, and team needs against the job description. | `ReviewWorkflow` | — |
| `TeamReviewer` | `AutonomousAgent` | Assesses cultural fit and peer working-style compatibility. | `ReviewWorkflow` | — |
| `FairnessJudge` | `AutonomousAgent` | Scores how proportionately the three reviewer perspectives drove the overall recommendation. | `FairnessSampler` | — |
| `ReviewWorkflow` | `Workflow` | Coordinates sanitization, the parallel reviewer fan-out, the aggregation, and the guardrail. | `ReviewEndpoint`, `CandidateConsumer` | `CandidateEntity` |
| `CandidateEntity` | `EventSourcedEntity` | Holds the evaluation's lifecycle (intake → evaluating → decided / degraded / blocked). | `ReviewWorkflow`, `FairnessSampler` | `EvaluationView` |
| `CandidateQueue` | `EventSourcedEntity` | Logs each submitted candidate profile for replay/audit. | `ReviewEndpoint`, `CandidateSimulator` | `CandidateConsumer` |
| `EvaluationView` | `View` | List-of-evaluations read model. | `CandidateEntity` events | `ReviewEndpoint` |
| `CandidateConsumer` | `Consumer` | Listens to `CandidateQueue` events and starts a workflow per submission. | `CandidateQueue` events | `ReviewWorkflow` |
| `CandidateSimulator` | `TimedAction` | Drips a sample candidate profile every 90 s. | scheduler | `CandidateQueue` |
| `FairnessSampler` | `TimedAction` | Samples one decided evaluation every 10 minutes for fairness scoring. | scheduler | `FairnessJudge`, `CandidateEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/evaluations/*` — submit, get, list, SSE. | — | `EvaluationView`, `CandidateQueue`, `CandidateEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record CandidateProfile(String candidateName, String roleApplied, String resumeText, String submittedBy) {}

record EvaluationPlan(String hrFocus, String managerFocus, String teamFocus) {}

record EvaluationFinding(String severity, String comment) {}   // severity: INFO | MINOR | CONCERN | DISQUALIFYING
record PerspectiveReview(String perspective, String decision, int score,
                         List<EvaluationFinding> findings, Instant reviewedAt) {}
    // perspective: HR | MANAGER | TEAM; decision: ADVANCE | HOLD | DECLINE

record HiringRecommendation(String decision, String rationale,
                             List<PerspectiveReview> perspectiveReviews,
                             String guardrailVerdict, Instant decidedAt) {}
    // decision: HIRE | FURTHER_REVIEW | REJECT

record FairnessVerdict(int score, String rationale) {}         // score 1–5

record CandidateEvaluation(
    String evaluationId,
    String candidateName,
    String roleApplied,
    EvaluationStatus status,
    Optional<String> redactedResumeText,
    Optional<Integer> redactionCount,
    Optional<PerspectiveReview> hrReview,
    Optional<PerspectiveReview> managerReview,
    Optional<PerspectiveReview> teamReview,
    Optional<HiringRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> fairnessScore,
    Optional<String> fairnessRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EvaluationStatus { INTAKE, EVALUATING, DECIDED, DEGRADED, BLOCKED }
```

The raw résumé text is never stored on `CandidateEntity`; only `redactedResumeText` is persisted, after the sanitizer runs.

### Events (on `CandidateEntity`)

`EvaluationCreated`, `ProfileSanitized`, `HrReviewAttached`, `ManagerReviewAttached`, `TeamReviewAttached`, `RecommendationDecided`, `EvaluationDegraded`, `EvaluationBlocked`, `FairnessScored`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/evaluations` — body `{ candidateName, roleApplied, resumeText, submittedBy? }` → `{ evaluationId }`. Starts a workflow.
- `GET /api/evaluations` — list all evaluations. Optional `?status=INTAKE|EVALUATING|DECIDED|DEGRADED|BLOCKED`.
- `GET /api/evaluations/{id}` — one evaluation.
- `GET /api/evaluations/sse` — server-sent events stream of every evaluation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Parallel Multi-Reviewer Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a candidate profile, live list of evaluations with status pills, expand-row to see the three perspective reviews, the overall recommendation, the redaction count, and the fairness score.

Browser title: `<title>Akka Sample: Parallel Multi-Reviewer Workflow</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid state-diagram label colours and edge-label `overflow:visible` per Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — special-category sanitizer** (`sanitizer`, flavor `special-category`): the `ReviewWorkflow` sanitize step runs a deterministic redactor over the submitted résumé text before any reviewer reads it. Protected attributes — inferred gender markers, age indicators, marital status phrases, national-origin cues, race/ethnicity markers — are replaced with typed placeholders. Only the redacted text is persisted. The redaction count is recorded on the entity. System-level.
- **E1 — fairness drift watch** (`eval-periodic`, flavor `drift-fairness-watch`): `FairnessSampler` (TimedAction) picks one decided evaluation every 10 minutes and asks `FairnessJudge` whether the three reviewer perspectives drove the recommendation proportionally, emitting a `FairnessScored` event with a 1–5 score and a short rationale. The aggregated scores surface a trend that operators use to detect systematic reviewer drift.

## 9. Agent prompts

- `PanelCoordinator` → `prompts/panel-coordinator.md`. Decomposes the profile into three axis briefs; later aggregates the three perspective reviews into one hiring recommendation.
- `HrReviewer` → `prompts/hr-reviewer.md`. Returns a `PerspectiveReview` on the HR axis.
- `ManagerReviewer` → `prompts/manager-reviewer.md`. Returns a `PerspectiveReview` on the manager axis.
- `TeamReviewer` → `prompts/team-reviewer.md`. Returns a `PerspectiveReview` on the team axis.
- `FairnessJudge` → `prompts/fairness-judge.md`. Returns a `FairnessVerdict` scoring cross-perspective proportionality.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a candidate profile; evaluation progresses INTAKE → EVALUATING → DECIDED within 60 s; the expanded view shows three perspective reviews and one overall recommendation; UI reflects each transition via SSE.
2. **J2** — Submit a profile whose résumé text contains protected-attribute markers; the persisted evaluation has those redacted, the raw values appear nowhere in `/api/evaluations/{id}`, and `redactionCount` is at least 2.
3. **J3** — Inject a reviewer timeout (set one reviewer's step timeout to 1 s); evaluation enters DEGRADED with the recommendation aggregated from the remaining perspectives and `failureReason` naming the missing reviewer.
4. **J4** — Inject a guardrail failure (coordinator returns a recommendation with an empty rationale); evaluation enters BLOCKED with the perspective reviews still visible for audit.
5. **J5** — Wait one fairness interval after a successful evaluation; the evaluation row shows a fairness score (1–5) and rationale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named parallel-hiring-reviewers demonstrating the
debate-multi-perspective × hr-recruiting cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
debate-multi-perspective-hr-recruiting-parallel-hiring-reviewers.
Java package io.akka.samples.parallelmultireviewerworkflow. Akka 3.6.0.
HTTP port 9853.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PanelCoordinator — definition() with capability(TaskAcceptance.of(PLAN)
    .maxIterationsPerTask(2)) AND capability(TaskAcceptance.of(AGGREGATE)
    .maxIterationsPerTask(3)). System prompt loaded from
    prompts/panel-coordinator.md.
    Returns EvaluationPlan{hrFocus, managerFocus, teamFocus} for PLAN
    and HiringRecommendation{decision, rationale, perspectiveReviews,
    guardrailVerdict, decidedAt} for AGGREGATE.
  * HrReviewer — capability(TaskAcceptance.of(REVIEW_HR)
    .maxIterationsPerTask(2)). System prompt from prompts/hr-reviewer.md.
    Returns PerspectiveReview{perspective="HR", decision, score,
    findings: List<EvaluationFinding{severity, comment}>, reviewedAt}.
  * ManagerReviewer — capability(TaskAcceptance.of(REVIEW_MANAGER)
    .maxIterationsPerTask(2)). System prompt from prompts/manager-reviewer.md.
    Returns PerspectiveReview{perspective="MANAGER", ...}.
  * TeamReviewer — capability(TaskAcceptance.of(REVIEW_TEAM)
    .maxIterationsPerTask(2)). System prompt from prompts/team-reviewer.md.
    Returns PerspectiveReview{perspective="TEAM", ...}.
  * FairnessJudge — capability(TaskAcceptance.of(SCORE_FAIRNESS)
    .maxIterationsPerTask(2)). System prompt from prompts/fairness-judge.md.
    Returns FairnessVerdict{score, rationale}.

- 1 deterministic helper SpecialCategorySanitizer (plain class in
  application/, NOT an agent): method redact(String resumeText) ->
  RedactionResult{redactedText, redactionCount}. Replaces inferred gender
  markers, age indicators, marital status phrases, national-origin cues,
  and race/ethnicity markers with typed placeholders ([GENDER], [AGE],
  [MARITAL_STATUS], [ORIGIN], [RACE]). Pure, no LLM call.

- 1 Workflow ReviewWorkflow with steps:
  createStep -> sanitizeStep -> planStep -> [parallel] hrStep,
  managerStep, teamStep -> joinStep -> aggregateStep -> guardrailStep
  -> emitStep.
  createStep calls CandidateEntity.createEvaluation emitting
  EvaluationCreated (INTAKE).
  sanitizeStep calls SpecialCategorySanitizer.redact(rawResumeText),
  then CandidateEntity.attachSanitized(redactedText, redactionCount)
  emitting ProfileSanitized (status -> EVALUATING). The raw résumé text
  is held only in the workflow's transient state and never persisted.
  planStep calls forAutonomousAgent(PanelCoordinator.class, PLAN)
  -> EvaluationPlan.
  hrStep, managerStep, teamStep run in parallel (CompletionStage zip of
  three calls); each wrapped with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(60)). Each attaches its PerspectiveReview
  via HrReviewAttached / ManagerReviewAttached / TeamReviewAttached.
  On any reviewer timeout, transition to degradeStep that calls
  aggregateStep with whichever perspectives returned, then ends with
  EvaluationDegraded (DEGRADED) and failureReason naming the missing
  reviewer.
  aggregateStep calls forAutonomousAgent(PanelCoordinator.class, AGGREGATE)
  with the available perspective reviews. guardrailStep runs the
  deterministic recommendation vetter (structural checks: non-empty
  rationale, >=1 perspective review, decision in {HIRE, FURTHER_REVIEW,
  REJECT}) plus an LLM judge with a 5-second timeout; on failure, ends
  with EvaluationBlocked (BLOCKED). emitStep emits RecommendationDecided
  (DECIDED).
  Override settings() with stepTimeout(60s) on the three reviewer steps
  and the aggregateStep, and defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity CandidateEntity holding state CandidateEvaluation
  {evaluationId, candidateName, roleApplied, EvaluationStatus,
  Optional<String> redactedResumeText, Optional<Integer> redactionCount,
  Optional<PerspectiveReview> hrReview, Optional<PerspectiveReview>
  managerReview, Optional<PerspectiveReview> teamReview,
  Optional<HiringRecommendation> recommendation, Optional<String>
  failureReason, Optional<Integer> fairnessScore, Optional<String>
  fairnessRationale, Instant createdAt, Optional<Instant> finishedAt}.
  EvaluationStatus enum: INTAKE, EVALUATING, DECIDED, DEGRADED, BLOCKED.
  Events: EvaluationCreated, ProfileSanitized, HrReviewAttached,
  ManagerReviewAttached, TeamReviewAttached, RecommendationDecided,
  EvaluationDegraded, EvaluationBlocked, FairnessScored.
  Commands: createEvaluation, attachSanitized, attachHrReview,
  attachManagerReview, attachTeamReview, decide, degrade, block,
  recordFairness, getEvaluation. emptyState() returns
  CandidateEvaluation.initial("", "", "") with no commandContext()
  reference.

- 1 EventSourcedEntity CandidateQueue with command enqueueProfile
  (candidateName, roleApplied, resumeText, submittedBy) emitting
  ProfileReceived{evaluationId, candidateName, roleApplied, resumeText,
  submittedBy, submittedAt}.

- 1 View EvaluationView with row type EvaluationRow (mirrors
  CandidateEvaluation minus the heavy redactedResumeText; keep perspective
  decisions/scores and the overall recommendation but truncate findings to
  counts for the list view). Table updater consumes CandidateEntity events.
  ONE query getAllEvaluations SELECT * AS evaluations FROM evaluation_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller
  filters client-side.

- 1 Consumer CandidateConsumer subscribed to CandidateQueue events; on
  ProfileReceived starts a ReviewWorkflow with the evaluationId as the
  workflow id, passing the raw résumé text in the workflow start command
  (not the entity).

- 2 TimedActions:
  * CandidateSimulator — every 90s, reads next line from
    src/main/resources/sample-events/candidate-profiles.jsonl and calls
    CandidateQueue.enqueueProfile.
  * FairnessSampler — every 10 minutes, queries
    EvaluationView.getAllEvaluations, picks the oldest DECIDED evaluation
    without a fairnessScore, calls
    forAutonomousAgent(FairnessJudge.class, SCORE_FAIRNESS) with the three
    perspective reviews and the overall recommendation, then calls
    CandidateEntity.recordFairness(score, rationale).

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /evaluations, GET /evaluations,
    GET /evaluations/{id}, GET /evaluations/sse, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The list filters by status
    client-side from getAllEvaluations.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 Bootstrap (service-setup) that schedules the two TimedActions on
  startup and fails fast with a clear message if the configured
  model-provider key reference does not resolve (never echoing key
  material).

Companion files:
- ReviewTasks.java declaring six Task<R> constants: PLAN (EvaluationPlan),
  REVIEW_HR (PerspectiveReview), REVIEW_MANAGER (PerspectiveReview),
  REVIEW_TEAM (PerspectiveReview), AGGREGATE (HiringRecommendation),
  SCORE_FAIRNESS (FairnessVerdict).
- Domain records EvaluationPlan, EvaluationFinding, PerspectiveReview,
  HiringRecommendation, FairnessVerdict, CandidateProfile, RedactionResult.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9853 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/candidate-profiles.jsonl with 8 canned
  candidate entries, at least 3 of which contain protected-attribute
  markers in the résumé text so the sanitizer is visibly exercised.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the root-level files for the endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 sanitizer
  special-category, E1 eval-periodic drift-fairness-watch) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types (note: protected attributes may appear in résumé text but are
  redacted before review), capability.*, model.*, decisions.employment_decision;
  marking jurisdictions, declared_frameworks, organization fields,
  incidents, and data.residency as TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/panel-coordinator.md, hr-reviewer.md, manager-reviewer.md,
  team-reviewer.md, fairness-judge.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Parallel
  Multi-Reviewer Workflow", one-line pitch, prerequisites (host software:
  none), generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live
  list with status pills; expand shows three perspective reviews, overall
  recommendation, redaction count, fairness score). Browser title exactly:
  <title>Akka Sample: Parallel Multi-Reviewer Workflow</title>. No subtitle
  on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: panel-coordinator.json, hr-reviewer.json,
  manager-reviewer.json, team-reviewer.json, fairness-judge.json), picks
  one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    panel-coordinator.json — list of either EvaluationPlan or
      HiringRecommendation objects. 4–6 EvaluationPlan entries (hrFocus /
      managerFocus / teamFocus triples across plausible candidate profiles)
      and 4–6 HiringRecommendation entries (each with a decision in
      {HIRE, FURTHER_REVIEW, REJECT}, a 60–120 word rationale, three nested
      PerspectiveReview objects, guardrailVerdict = "ok").
    hr-reviewer.json — 4–6 PerspectiveReview entries with
      perspective="HR", a decision, a 1–5 score, and 2–5
      EvaluationFinding objects whose severity is drawn from {INFO, MINOR,
      CONCERN, DISQUALIFYING}.
    manager-reviewer.json — 4–6 PerspectiveReview entries with
      perspective="MANAGER".
    team-reviewer.json — 4–6 PerspectiveReview entries with
      perspective="TEAM", at least one finding referencing working-style
      or collaboration.
    fairness-judge.json — 4–6 FairnessVerdict entries (score 1–5 +
      one-sentence rationale on whether the perspectives drove the
      recommendation proportionally).
- A MockModelProvider.seedFor(evaluationId) helper makes the selection
  deterministic per evaluation id so the same candidate in dev produces
  the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent never silently downgraded to Agent; the extends
  clause matches the spec verbatim.
- (Lesson 4) Workflow step timeouts set explicitly on every agent-calling
  step (60s on the three reviewer steps and the aggregate step).
- (Lesson 6) Optional<T> for every nullable lifecycle field on the
  CandidateEvaluation record and the View row record.
- (Lesson 7) AutonomousAgent requires the companion ReviewTasks.java
  declaring every Task<R> constant.
- (Lesson 8) Verify model names against the provider's current lineup
  before locking them in application.conf.
- (Lesson 9) Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- (Lesson 10) Port 9853 declared explicitly in application.conf.
- (Lesson 11) Source-platform metadata is corpus-internal — never
  user-facing.
- (Lesson 12) UI fits the 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier labels are descriptive ("Runs out of the
  box"), never T1/T2/T3/T4 and never the word "deferred".
- (Lesson 23) No competitor brand names anywhere in user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS
  overrides AND theme variables (state-diagram label colour white,
  edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render
  black-on-black and arrow labels clip.
- (Lesson 25) Never write the model-provider key value to disk; record
  only the reference.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. Exactly five .tab-panel sections; no
  display:none zombie panels — delete removed tabs from the DOM.
- Parallel reviewer steps use CompletionStage zip, NOT sequential calls.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller,
  complex, Akka SDK in narrative, use, use, deferred.
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
