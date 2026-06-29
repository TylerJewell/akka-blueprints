# SPEC — peerreview

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Peer Review Panel.
**One-line pitch:** Submit a document; a moderator briefs a Technical, a Style, and a Compliance reviewer, runs them in parallel against the same text, and reconciles their findings into one overall verdict with per-axis detail.

## 2. What this blueprint demonstrates

The **debate-multi-perspective** coordination pattern wired with Akka's first-party primitives: a Workflow asks one moderator agent to brief three specialist reviewer agents, runs those reviewers in parallel over the same document, then asks the moderator to reconcile three independent perspectives into a single verdict. The blueprint also demonstrates three governance mechanisms — a **PII sanitizer** that redacts personal data from the document before any reviewer reads it, an **output guardrail** that vets the reconciled verdict before it is returned, and an **eval-event** sampler that scores how mutually consistent the three axis findings are.

## 3. User-facing flows

The user opens the App UI tab and submits a document (title + body) via the form.

1. The system creates a `Review` record in `INTAKE` and starts a `ReviewWorkflow`.
2. A deterministic PII sanitizer redacts emails, phone numbers, government identifiers, and obvious names from the body. Only the redacted body is persisted and passed downstream; the raw body is never stored. The review moves to `REVIEWING`.
3. The Moderator decomposes the document into three axis briefs: a technical focus, a style focus, and a compliance focus.
4. The workflow forks: `TechnicalReviewer`, `StyleReviewer`, and `ComplianceReviewer` run concurrently over the redacted body. Each returns a typed `AxisReview` with a per-axis verdict, a score, and a list of findings.
5. The Moderator reconciles the three axis reviews into an `OverallVerdict { verdict, summary, axisReviews }`.
6. An output guardrail vets the reconciled verdict; if it fails, the review moves to `BLOCKED`. Otherwise, the review moves to `SYNTHESISED`.
7. If any reviewer times out after 60 seconds, the workflow short-circuits: the Moderator reconciles from whichever axes returned, and the review enters `DEGRADED`.
8. Every five minutes, an eval sampler picks one synthesised review and asks a judge whether the three axis findings agree with the overall verdict, attaching a 1–5 consistency score.

A `SubmissionSimulator` (TimedAction) drips a sample document every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewModerator` | `AutonomousAgent` | Decomposes the document into three axis briefs; later reconciles the three axis reviews into one overall verdict. | `ReviewWorkflow` | returns typed result to workflow |
| `TechnicalReviewer` | `AutonomousAgent` | Assesses factual and technical accuracy of the redacted document. | `ReviewWorkflow` | — |
| `StyleReviewer` | `AutonomousAgent` | Assesses clarity, tone, and structure. | `ReviewWorkflow` | — |
| `ComplianceReviewer` | `AutonomousAgent` | Assesses policy and regulatory fit. | `ReviewWorkflow` | — |
| `EvalJudge` | `AutonomousAgent` | Scores how mutually consistent the three axis findings are against the overall verdict. | `EvalSampler` | — |
| `ReviewWorkflow` | `Workflow` | Coordinates redaction, the parallel reviewer fan-out, the reconciliation, and the guardrail. | `ReviewEndpoint`, `SubmissionConsumer` | `ReviewEntity` |
| `ReviewEntity` | `EventSourcedEntity` | Holds the review's lifecycle (intake → reviewing → synthesised / degraded / blocked). | `ReviewWorkflow`, `EvalSampler` | `ReviewView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted document for replay/audit. | `ReviewEndpoint`, `SubmissionSimulator` | `SubmissionConsumer` |
| `ReviewView` | `View` | List-of-reviews read model. | `ReviewEntity` events | `ReviewEndpoint` |
| `SubmissionConsumer` | `Consumer` | Listens to `SubmissionQueue` events and starts a workflow per submission. | `SubmissionQueue` events | `ReviewWorkflow` |
| `SubmissionSimulator` | `TimedAction` | Drips a sample document every 60 s. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised review every 5 minutes for consistency scoring. | scheduler | `EvalJudge`, `ReviewEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/reviews/*` — submit, get, list, SSE. | — | `ReviewView`, `SubmissionQueue`, `ReviewEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record DocumentSubmission(String title, String body, String submittedBy) {}

record ReviewPlan(String technicalFocus, String styleFocus, String complianceFocus) {}

record ReviewFinding(String severity, String comment) {}            // severity: INFO | MINOR | MAJOR | BLOCKER
record AxisReview(String axis, String verdict, int score,
                  List<ReviewFinding> findings, Instant reviewedAt) {}  // axis: TECHNICAL | STYLE | COMPLIANCE; verdict: PASS | REVISE | FAIL

record OverallVerdict(String verdict, String summary,
                      List<AxisReview> axisReviews,
                      String guardrailVerdict, Instant synthesisedAt) {}  // verdict: APPROVE | REVISE | REJECT

record ConsistencyVerdict(int score, String rationale) {}            // score 1–5

record Review(
    String reviewId,
    String title,
    ReviewStatus status,
    Optional<String> redactedBody,
    Optional<Integer> redactionCount,
    Optional<AxisReview> technical,
    Optional<AxisReview> style,
    Optional<AxisReview> compliance,
    Optional<OverallVerdict> verdict,
    Optional<String> failureReason,
    Optional<Integer> consistencyScore,
    Optional<String> consistencyRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReviewStatus { INTAKE, REVIEWING, SYNTHESISED, DEGRADED, BLOCKED }
```

The raw submission `body` is never stored on `ReviewEntity`; only `redactedBody` is persisted, after the sanitizer runs.

### Events (on `ReviewEntity`)

`ReviewCreated`, `DocumentSanitized`, `TechnicalReviewAttached`, `StyleReviewAttached`, `ComplianceReviewAttached`, `VerdictSynthesised`, `ReviewDegraded`, `ReviewBlocked`, `ConsistencyScored`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reviews` — body `{ title, body, submittedBy? }` → `{ reviewId }`. Starts a workflow.
- `GET /api/reviews` — list all reviews. Optional `?status=INTAKE|REVIEWING|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/reviews/{id}` — one review.
- `GET /api/reviews/sse` — server-sent events stream of every review change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Peer Review Panel"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a document, live list of reviews with status pills, expand-row to see the three axis reviews, the overall verdict, the redaction count, and the consistency score.

Browser title: `<title>Akka Sample: Peer Review Panel</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid state-diagram label colours and edge-label `overflow:visible` per Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`sanitizer`, flavor `pii`): the `ReviewWorkflow` sanitize step runs a deterministic redactor over the submitted body before any reviewer reads it. Emails, phone numbers, government identifiers, and obvious personal names are replaced with typed placeholders. Only the redacted body is persisted. The redaction count is recorded on the entity. System-level.
- **G1 — output guardrail** (`before-agent-response` on `ReviewModerator`): vets the reconciled `OverallVerdict` for structural violations (missing summary, empty axis reviews, a verdict value outside the allowed set) and prohibited content. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesised review every 5 minutes and asks `EvalJudge` whether the three axis findings agree with the overall verdict, emitting a `ConsistencyScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `ReviewModerator` → `prompts/moderator.md`. Decomposes the document into three axis briefs; later reconciles the three axis reviews into one overall verdict.
- `TechnicalReviewer` → `prompts/technical-reviewer.md`. Returns an `AxisReview` on the technical axis.
- `StyleReviewer` → `prompts/style-reviewer.md`. Returns an `AxisReview` on the style axis.
- `ComplianceReviewer` → `prompts/compliance-reviewer.md`. Returns an `AxisReview` on the compliance axis.
- `EvalJudge` → `prompts/eval-judge.md`. Returns a `ConsistencyVerdict` scoring cross-axis agreement.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a document; review progresses INTAKE → REVIEWING → SYNTHESISED within 60 s; the expanded view shows three axis reviews and one overall verdict; UI reflects each transition via SSE.
2. **J2** — Submit a document whose body contains an email and a phone number; the persisted review has those redacted, the raw values appear nowhere in `/api/reviews/{id}`, and `redactionCount` is at least 2.
3. **J3** — Inject a reviewer timeout (set one reviewer's step timeout to 1 s); review enters DEGRADED with the overall verdict reconciled from the remaining axes and `failureReason` naming the missing reviewer.
4. **J4** — Inject a guardrail failure (Moderator returns a verdict with an empty summary); review enters BLOCKED with the axis reviews still visible for audit.
5. **J5** — Wait one eval interval after a successful synthesis; the review row shows a consistency score (1–5) and rationale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named peerreview demonstrating the
debate-multi-perspective × content-editorial cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact peerreview.
Java package io.akka.samples.peerreview. Akka 3.6.0. HTTP port 9607.

Components to wire (exactly):
- 5 AutonomousAgents:
  * ReviewModerator — definition() with capability(TaskAcceptance.of(PLAN)
    .maxIterationsPerTask(2)) AND capability(TaskAcceptance.of(SYNTHESISE)
    .maxIterationsPerTask(3)). System prompt loaded from prompts/moderator.md.
    Returns ReviewPlan{technicalFocus, styleFocus, complianceFocus} for PLAN
    and OverallVerdict{verdict, summary, axisReviews, guardrailVerdict,
    synthesisedAt} for SYNTHESISE.
  * TechnicalReviewer — capability(TaskAcceptance.of(REVIEW_TECHNICAL)
    .maxIterationsPerTask(2)). System prompt from prompts/technical-reviewer.md.
    Returns AxisReview{axis="TECHNICAL", verdict, score, findings: List<
    ReviewFinding{severity, comment}>, reviewedAt}.
  * StyleReviewer — capability(TaskAcceptance.of(REVIEW_STYLE)
    .maxIterationsPerTask(2)). System prompt from prompts/style-reviewer.md.
    Returns AxisReview{axis="STYLE", ...}.
  * ComplianceReviewer — capability(TaskAcceptance.of(REVIEW_COMPLIANCE)
    .maxIterationsPerTask(2)). System prompt from prompts/compliance-reviewer.md.
    Returns AxisReview{axis="COMPLIANCE", ...}.
  * EvalJudge — capability(TaskAcceptance.of(SCORE_CONSISTENCY)
    .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md.
    Returns ConsistencyVerdict{score, rationale}.

- 1 deterministic helper PiiSanitizer (plain class in application/, NOT an
  agent): method redact(String body) -> RedactionResult{redactedBody,
  redactionCount}. Replaces emails, phone numbers, government identifiers
  (SSN-like, passport-like), and capitalised two-token name patterns with
  typed placeholders ([EMAIL], [PHONE], [ID], [NAME]). Pure, no LLM call.

- 1 Workflow ReviewWorkflow with steps:
  createStep -> sanitizeStep -> planStep -> [parallel] technicalStep,
  styleStep, complianceStep -> joinStep -> synthesiseStep -> guardrailStep
  -> emitStep.
  createStep calls ReviewEntity.createReview emitting ReviewCreated (INTAKE).
  sanitizeStep calls PiiSanitizer.redact(rawBody), then
  ReviewEntity.attachSanitized(redactedBody, redactionCount) emitting
  DocumentSanitized (status -> REVIEWING). The raw body is held only in the
  workflow's transient state and never persisted.
  planStep calls forAutonomousAgent(ReviewModerator.class, PLAN) -> ReviewPlan.
  technicalStep, styleStep, complianceStep run in parallel (CompletionStage
  zip of three calls); each wrapped with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(60)). Each attaches its AxisReview via
  TechnicalReviewAttached / StyleReviewAttached / ComplianceReviewAttached.
  On any reviewer timeout, transition to degradeStep that calls synthesiseStep
  with whichever axes returned, then ends with ReviewDegraded (DEGRADED) and
  failureReason naming the missing reviewer.
  synthesiseStep calls forAutonomousAgent(ReviewModerator.class, SYNTHESISE)
  with the available axis reviews. guardrailStep runs the deterministic
  verdict vetter (structural checks: non-empty summary, >=1 axis review,
  verdict in {APPROVE, REVISE, REJECT}) plus an LLM judge with a 5-second
  timeout; on failure, ends with ReviewBlocked (BLOCKED). emitStep emits
  VerdictSynthesised (SYNTHESISED).
  Override settings() with stepTimeout(60s) on the three reviewer steps and
  the synthesiseStep, and defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity ReviewEntity holding state Review{reviewId, title,
  ReviewStatus, Optional<String> redactedBody, Optional<Integer>
  redactionCount, Optional<AxisReview> technical, Optional<AxisReview> style,
  Optional<AxisReview> compliance, Optional<OverallVerdict> verdict,
  Optional<String> failureReason, Optional<Integer> consistencyScore,
  Optional<String> consistencyRationale, Instant createdAt, Optional<Instant>
  finishedAt}. ReviewStatus enum: INTAKE, REVIEWING, SYNTHESISED, DEGRADED,
  BLOCKED. Events: ReviewCreated, DocumentSanitized, TechnicalReviewAttached,
  StyleReviewAttached, ComplianceReviewAttached, VerdictSynthesised,
  ReviewDegraded, ReviewBlocked, ConsistencyScored. Commands: createReview,
  attachSanitized, attachTechnical, attachStyle, attachCompliance, synthesise,
  degrade, block, recordConsistency, getReview. emptyState() returns
  Review.initial("", "") with no commandContext() reference.

- 1 EventSourcedEntity SubmissionQueue with command enqueueSubmission(title,
  body, submittedBy) emitting SubmissionReceived{reviewId, title, body,
  submittedBy, submittedAt}.

- 1 View ReviewView with row type ReviewRow (mirrors Review minus the heavy
  redactedBody text; keep axis verdicts/scores and the overall verdict but
  truncate findings to counts for the list view). Table updater consumes
  ReviewEntity events. ONE query getAllReviews SELECT * AS reviews FROM
  review_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  caller filters client-side.

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  SubmissionReceived starts a ReviewWorkflow with the reviewId as the
  workflow id, passing the raw body in the workflow start command (not the
  entity).

- 2 TimedActions:
  * SubmissionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/review-submissions.jsonl and calls
    SubmissionQueue.enqueueSubmission.
  * EvalSampler — every 5 minutes, queries ReviewView.getAllReviews, picks the
    oldest SYNTHESISED review without a consistencyScore, calls
    forAutonomousAgent(EvalJudge.class, SCORE_CONSISTENCY) with the three axis
    reviews and the overall verdict, then calls
    ReviewEntity.recordConsistency(score, rationale).

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /reviews, GET /reviews, GET /reviews/{id},
    GET /reviews/sse, and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/. The posts list filters by status
    client-side from getAllReviews.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 Bootstrap (service-setup) that schedules the two TimedActions on startup
  and fails fast with a clear message if the configured model-provider key
  reference does not resolve (never echoing key material).

Companion files:
- ReviewTasks.java declaring six Task<R> constants: PLAN (ReviewPlan),
  REVIEW_TECHNICAL (AxisReview), REVIEW_STYLE (AxisReview), REVIEW_COMPLIANCE
  (AxisReview), SYNTHESISE (OverallVerdict), SCORE_CONSISTENCY
  (ConsistencyVerdict).
- Domain records ReviewPlan, ReviewFinding, AxisReview, OverallVerdict,
  ConsistencyVerdict, DocumentSubmission, RedactionResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9607 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/review-submissions.jsonl with 8 canned
  document lines, at least 3 of which contain PII (email/phone/id) so the
  sanitizer is visibly exercised.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (S1 sanitizer pii,
  G1 guardrail before-agent-response, E1 eval-event on-decision-eval) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types (note: PII may transit the system but is redacted before review),
  capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/moderator.md, technical-reviewer.md, style-reviewer.md,
  compliance-reviewer.md, eval-judge.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Peer Review Panel",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live
  list with status pills; expand shows three axis reviews, overall verdict,
  redaction count, consistency score). Browser title exactly:
  <title>Akka Sample: Peer Review Panel</title>. No subtitle on Overview.

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
  named in Section 9: moderator.json, technical-reviewer.json,
  style-reviewer.json, compliance-reviewer.json, eval-judge.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    moderator.json — list of either ReviewPlan or OverallVerdict objects. 4–6
      ReviewPlan entries (technicalFocus / styleFocus / complianceFocus triples
      across plausible editorial documents) and 4–6 OverallVerdict entries
      (each with a verdict in {APPROVE, REVISE, REJECT}, an 60–120 word
      summary, three nested AxisReview objects, guardrailVerdict = "ok").
    technical-reviewer.json — 4–6 AxisReview entries with axis="TECHNICAL", a
      verdict, a 1–5 score, and 2–5 ReviewFinding objects whose severity is
      drawn from {INFO, MINOR, MAJOR, BLOCKER}.
    style-reviewer.json — 4–6 AxisReview entries with axis="STYLE".
    compliance-reviewer.json — 4–6 AxisReview entries with axis="COMPLIANCE",
      at least one finding referencing a policy or disclosure obligation.
    eval-judge.json — 4–6 ConsistencyVerdict entries (score 1–5 + one-sentence
      rationale on whether the axes agree with the overall verdict).
- A MockModelProvider.seedFor(reviewId) helper makes the selection
  deterministic per review id so the same review in dev produces the same
  output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent never silently downgraded to Agent; the extends
  clause matches the spec verbatim.
- (Lesson 4) Workflow step timeouts set explicitly on every agent-calling step
  (60s on the three reviewer steps and the synthesise step).
- (Lesson 6) Optional<T> for every nullable lifecycle field on the Review
  record and the View row record.
- (Lesson 7) AutonomousAgent requires the companion ReviewTasks.java declaring
  every Task<R> constant.
- (Lesson 8) Verify model names against the provider's current lineup before
  locking them in application.conf.
- (Lesson 9) Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- (Lesson 10) Port 9607 declared explicitly in application.conf.
- (Lesson 11) Source-platform metadata is corpus-internal — never user-facing.
- (Lesson 12) UI fits the 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier labels are descriptive ("Runs out of the box"),
  never T1/T2/T3/T4 and never the word "deferred".
- (Lesson 23) No competitor brand names anywhere in user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip.
- (Lesson 25) Never write the model-provider key value to disk; record only
  the reference.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. Exactly five .tab-panel sections; no display:none zombie
  panels — delete removed tabs from the DOM.
- Parallel reviewer steps use CompletionStage zip, NOT sequential calls.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, use, use, deferred.
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
