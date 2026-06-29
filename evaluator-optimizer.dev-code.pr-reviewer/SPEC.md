# SPEC — pr-reviewer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Pull Request Review Agent.
**One-line pitch:** Submit a PR diff; a reviewer agent produces structured code-review feedback; an alignment agent checks the feedback against documented contracts and conventions; the two iterate until the alignment agent approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`ReviewerAgent`) and a reviewer agent (`AlignmentAgent`), feeding each alignment note back into the next feedback draft until convergence or a halt. The blueprint also demonstrates three governance mechanisms — an **output guardrail** that gates each feedback draft against a deterministic personal-critique check before the alignment agent runs, an **eval-event** that records every cycle's verdict for downstream quality measurement, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the review in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a PR diff (a code diff plus an optional description).

1. The system creates a `Review` record in `REVIEWING` and starts a `ReviewWorkflow`.
2. The Reviewer drafts feedback attempt #1: a set of structured code-review comments on the diff.
3. The output guardrail vets the feedback against a personal-critique policy. Feedback containing personal language directed at the author is short-circuited back to the Reviewer with a deterministic note; it never reaches the Alignment agent.
4. The Alignment agent checks the feedback against the project's documented contracts and conventions and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `AlignmentNotes` payload (up to three bullets).
5. On `APPROVE`, the workflow transitions the review to `APPROVED` with the winning attempt's feedback and the alignment agent's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the alignment notes, and the alignment verdict on the entity, then calls the Reviewer again with the notes attached. The Reviewer produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every alignment check for audit, and an `EvalRecorded` event captures the rejection.

A `PrSimulator` (TimedAction) drips a canned PR diff every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewerAgent` | `AutonomousAgent` | Analyzes a PR diff and produces structured feedback comments; accepts prior alignment notes on revisions. | `ReviewWorkflow` | returns `FeedbackDraft` to workflow |
| `AlignmentAgent` | `AutonomousAgent` | Checks feedback against documented contracts and conventions; returns `APPROVE` or `REVISE` with notes. | `ReviewWorkflow` | returns `AlignmentCheck` to workflow |
| `ReviewWorkflow` | `Workflow` | Runs the review → guardrail → alignment-check → revise loop; halts at the ceiling. | `ReviewEndpoint`, `PrSubmissionConsumer` | `ReviewEntity` |
| `ReviewEntity` | `EventSourcedEntity` | Holds the review lifecycle, every feedback attempt, every alignment check, and the final outcome. | `ReviewWorkflow` | `ReviewsView` |
| `PrQueue` | `EventSourcedEntity` | Logs each submitted PR for replay and audit. | `ReviewEndpoint`, `PrSimulator` | `PrSubmissionConsumer` |
| `ReviewsView` | `View` | List-of-reviews read model. | `ReviewEntity` events | `ReviewEndpoint` |
| `PrSubmissionConsumer` | `Consumer` | Subscribes to `PrQueue` events; starts a workflow per submission. | `PrQueue` events | `ReviewWorkflow` |
| `PrSimulator` | `TimedAction` | Drips a sample PR diff every 60 s from `sample-events/pr-diffs.jsonl`. | scheduler | `PrQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `ReviewsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `ReviewEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/reviews/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ReviewsView`, `PrQueue`, `ReviewEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record PrSubmission(String diffText, String description, String submittedBy) {}

record FeedbackDraft(List<ReviewComment> comments, int commentCount, Instant draftedAt) {}

record ReviewComment(String filePath, int lineNumber, String severity, String body) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record AlignmentNotes(List<String> bullets, String overallRationale) {}

record AlignmentCheck(AlignmentVerdict verdict, AlignmentNotes notes, int score, Instant checkedAt) {}

record ReviewAttempt(
    int attemptNumber,
    FeedbackDraft draft,
    GuardrailVerdict guardrail,
    Optional<AlignmentCheck> alignmentCheck
) {}

record Review(
    String reviewId,
    String diffText,
    String description,
    int maxAttempts,
    ReviewStatus status,
    List<ReviewAttempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<List<ReviewComment>> approvedComments,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReviewStatus { REVIEWING, CHECKING_ALIGNMENT, APPROVED, REJECTED_FINAL }

enum AlignmentVerdict { APPROVE, REVISE }

enum CommentSeverity { BLOCKER, SUGGESTION, NITPICK }
```

### Events (on `ReviewEntity`)

`ReviewCreated`, `FeedbackDrafted`, `FeedbackGuardrailVerdictRecorded`, `FeedbackAlignmentChecked`, `ReviewApproved`, `ReviewRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reviews` — body `{ diffText, description?, submittedBy? }` → `{ reviewId }`. Starts a workflow.
- `GET /api/reviews` — list all reviews. Optional `?status=REVIEWING|CHECKING_ALIGNMENT|APPROVED|REJECTED_FINAL`.
- `GET /api/reviews/{id}` — one review (including every attempt and every alignment check).
- `GET /api/reviews/sse` — server-sent events stream of every review change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Pull Request Review Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a PR diff, live list of reviews with status pills, click-to-expand per-attempt timeline showing each feedback draft, the guardrail verdict, the alignment verdict, and the alignment notes.

Browser title: `<title>Akka Sample: Pull Request Review Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`after-llm-response` on `ReviewerAgent`): a deterministic check that the feedback draft contains no personal critique language directed at the author. Feedback with personal language is short-circuited back to the Reviewer with a structured feedback note (`reasonCode = PERSONAL_CRITIQUE`); it never reaches the Alignment agent. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's alignment check is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, personalCritiqueBlocked }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/reviews/{id}`.

## 9. Agent prompts

- `ReviewerAgent` → `prompts/reviewer.md`. Analyzes a PR diff and produces structured feedback comments; on a revision call, takes the prior `AlignmentNotes` as input and produces a revised feedback draft.
- `AlignmentAgent` → `prompts/alignment.md`. Checks feedback against documented contracts and conventions; returns `APPROVE` with a one-line rationale or `REVISE` with up to three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a PR diff; review progresses `REVIEWING` → `CHECKING_ALIGNMENT` → `APPROVED` within the retry ceiling; the App UI shows every attempt's feedback and alignment check.
2. **J2 — halt at ceiling** — Submit a diff whose alignment cannot be satisfied (test mode forces the Alignment agent to `REVISE` every attempt); review progresses through every attempt and lands in `REJECTED_FINAL` with the best feedback preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a diff where the Reviewer's first draft contains a personal critique comment; the guardrail short-circuits with `reasonCode = PERSONAL_CRITIQUE`, the Reviewer re-drafts without personal language, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any review shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named pr-reviewer demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-pr-reviewer.
Java package io.akka.samples.pullrequestreviewagent. Akka 3.6.0. HTTP port 9610.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_REVIEW).maxIterationsPerTask(3)).
    System prompt loaded from prompts/reviewer.md. Returns FeedbackDraft{comments,
    commentCount, draftedAt} for both REVIEW and REVISE_REVIEW. The
    REVISE_REVIEW task takes (diffText, description, priorDraft, AlignmentNotes)
    as inputs.
  * AlignmentAgent — definition() with
    capability(TaskAcceptance.of(CHECK_ALIGNMENT).maxIterationsPerTask(2)).
    System prompt from prompts/alignment.md. Returns AlignmentCheck{verdict,
    notes, score, checkedAt} where verdict is the AlignmentVerdict enum
    (APPROVE | REVISE) and score is a 1–5 integer rubric.

- 1 Workflow ReviewWorkflow with steps:
    startStep -> reviewStep -> guardrailStep -> [guardrail FAIL? reviewStep
    again with structured feedback : alignmentStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       reviewStep with alignment notes attached : rejectStep)] -> END.
  reviewStep calls forAutonomousAgent(ReviewerAgent.class, reviewId).runSingleTask(
    REVIEW or REVISE_REVIEW) then forTask(taskId).result(REVIEW or
    REVISE_REVIEW). alignmentStep calls forAutonomousAgent(AlignmentAgent.class,
    reviewId).runSingleTask(CHECK_ALIGNMENT). approveStep emits ReviewApproved.
    rejectStep emits ReviewRejectedFinal with the highest-scoring attempt's
    comments as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on reviewStep and alignmentStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): scans
    draft.comments() for personal language patterns. On FAIL, emits
    FeedbackGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "PERSONAL_CRITIQUE", then transitions back to reviewStep
    with a fixed AlignmentNotes("Feedback contains personal critique directed
    at the author; rewrite as impersonal observations about the code.").

- 1 EventSourcedEntity ReviewEntity holding state Review{reviewId, diffText,
  description, maxAttempts, ReviewStatus status, List<ReviewAttempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<List<ReviewComment>>
  approvedComments, Optional<String> rejectionReason, Instant createdAt,
  Optional<Instant> finishedAt}. ReviewStatus enum: REVIEWING,
  CHECKING_ALIGNMENT, APPROVED, REJECTED_FINAL. Events: ReviewCreated,
  FeedbackDrafted, FeedbackGuardrailVerdictRecorded, FeedbackAlignmentChecked,
  ReviewApproved, ReviewRejectedFinal, EvalRecorded. Commands: createReview,
  recordDraft, recordGuardrail, recordAlignmentCheck, approve, rejectFinal,
  recordEval, getReview. emptyState() returns Review.initial("", "", 4) with
  no commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity PrQueue with command submitPr(diffText, description,
  submittedBy) emitting PrSubmitted{reviewId, diffText, description,
  submittedBy, submittedAt}.

- 1 View ReviewsView with row type ReviewRow (mirrors Review; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes ReviewEntity events. ONE query
  getAllReviews SELECT * AS reviews FROM reviews_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer PrSubmissionConsumer subscribed to PrQueue events; on PrSubmitted
  starts a ReviewWorkflow with the reviewId as the workflow id.

- 2 TimedActions:
  * PrSimulator — every 60s, reads next line from
    src/main/resources/sample-events/pr-diffs.jsonl and calls
    PrQueue.submitPr.
  * EvalSampler — every 30s, queries ReviewsView.getAllReviews, finds reviews
    with a checked attempt that has not yet been recorded as an EvalRecorded
    event, and calls ReviewEntity.recordEval(attemptNumber, verdict, score,
    personalCritiqueBlocked). Idempotent per (reviewId, attemptNumber).

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /reviews, GET /reviews, GET /reviews/{id},
    GET /reviews/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /reviews body
    is {diffText, description?, submittedBy?}; missing description defaults to
    "", missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ReviewTasks.java declaring three Task<R> constants: REVIEW (resultConformsTo
  FeedbackDraft), REVISE_REVIEW (FeedbackDraft), CHECK_ALIGNMENT (AlignmentCheck).
- Domain records FeedbackDraft, ReviewComment, GuardrailVerdict, AlignmentNotes,
  AlignmentCheck, ReviewAttempt, Review; enums ReviewStatus, AlignmentVerdict,
  CommentSeverity.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9610 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  pr-reviewer.review.max-attempts = 4, overridable by env var.
- src/main/resources/sample-events/pr-diffs.jsonl with 8 canned PR diff
  lines, each shaped {"diffText":"...", "description":"...", "submittedBy":"ci-bot"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = code-review-automation,
  decisions.authority_level = advisory-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/reviewer.md, prompts/alignment.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Pull Request Review Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
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
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Pull Request Review Agent</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
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
  named in Section 9: reviewer.json, alignment.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    reviewer.json — 6 FeedbackDraft entries. Three are first-pass drafts
      with 3–5 SUGGESTION or BLOCKER comments pointing to specific lines
      in the sample diffs. Two are revision drafts that address prior
      alignment notes (removing overly personal language, adding doc
      references). One draft intentionally contains a personal critique
      comment ("you should know better than this") used to exercise the
      guardrail in J3.
    alignment.json — 6 AlignmentCheck entries. Three return verdict=APPROVE
      with score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and an AlignmentNotes payload of
      up to three bullets ("comment lacks reference to API contract §3",
      "BLOCKER severity applied to a style concern", "missing doc anchor").
- A MockModelProvider.seedFor(reviewId, attemptNumber) helper makes the
  selection deterministic per (reviewId, attemptNumber) so the same review
  in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReviewerAgent
  and AlignmentAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a ReviewTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Review row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ReviewTasks.java is mandatory; generating ReviewerAgent or
  AlignmentAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9610, declared in application.conf
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
