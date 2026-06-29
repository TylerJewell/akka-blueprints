# SPEC — scored-loop-tailor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Scored CV Reviewer.
**One-line pitch:** Submit a job description; a tailor agent produces a targeted CV; a reviewer agent scores it against a structured hiring rubric; the two iterate until the score meets the acceptance threshold or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`CvTailorAgent`) and a reviewer agent (`CvReviewerAgent`), feeding each set of review notes back into the next draft until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's score and verdict for downstream quality measurement, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the application in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a job posting (a job title, a description, and an optional candidate profile summary).

1. The system creates an `Application` record in `TAILORING` and starts a `TailoringWorkflow`.
2. The CV Tailor produces draft #1: a structured CV targeted at the job description.
3. The output guardrail checks that the CV draft contains all mandatory sections (Summary, Experience, Skills). Missing-section drafts are short-circuited back to the Tailor with a deterministic feedback note; they never reach the Reviewer.
4. The Reviewer scores the draft against a fixed rubric (relevance, completeness, keyword alignment, clarity) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `ReviewNotes` payload (three bullets at most).
5. On `APPROVE`, the workflow transitions the application to `APPROVED` with the winning draft's text and the reviewer's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the review, and the reviewer's verdict on the entity, then calls the Tailor again with the review notes attached. The Tailor produces draft #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the highest-scoring draft is preserved on the entity along with every review for audit, and a `ReviewEvalRecorded` event captures the rejection.

A `SubmissionSimulator` (TimedAction) drips a canned job posting every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CvTailorAgent` | `AutonomousAgent` | Produces a targeted CV draft for a job description; accepts prior review notes when revising. | `TailoringWorkflow` | returns `CvDraft` to workflow |
| `CvReviewerAgent` | `AutonomousAgent` | Scores a CV draft against the rubric; returns `APPROVE` or `REVISE` with notes. | `TailoringWorkflow` | returns `CvReview` to workflow |
| `TailoringWorkflow` | `Workflow` | Runs the tailor → guardrail → review → revise loop; halts at the ceiling. | `RecruitingEndpoint`, `SubmissionConsumer` | `ApplicationEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | Holds the application lifecycle, every CV draft, every review, and the final outcome. | `TailoringWorkflow` | `ApplicationsView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted job posting for replay and audit. | `RecruitingEndpoint`, `SubmissionSimulator` | `SubmissionConsumer` |
| `ApplicationsView` | `View` | List-of-applications read model. | `ApplicationEntity` events | `RecruitingEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a workflow per submission. | `SubmissionQueue` events | `TailoringWorkflow` |
| `SubmissionSimulator` | `TimedAction` | Drips a sample job posting every 60 s from `sample-events/job-postings.jsonl`. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `ApplicationsView`, records a `ReviewEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `ApplicationEntity` |
| `RecruitingEndpoint` | `HttpEndpoint` | `/api/applications/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ApplicationsView`, `SubmissionQueue`, `ApplicationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record JobPosting(String jobTitle, String description, String candidateProfile) {}

record CvDraft(String text, List<String> sectionsPresent, Instant draftedAt) {}

record SectionGuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record CvReview(ReviewVerdict verdict, ReviewNotes notes, int score, Instant reviewedAt) {}

record CvAttempt(
    int attemptNumber,
    CvDraft draft,
    SectionGuardrailVerdict guardrail,
    Optional<CvReview> review
) {}

record Application(
    String applicationId,
    String jobTitle,
    String description,
    int maxAttempts,
    ApplicationStatus status,
    List<CvAttempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<String> approvedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ApplicationStatus { TAILORING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewVerdict { APPROVE, REVISE }
```

### Events (on `ApplicationEntity`)

`ApplicationCreated`, `CvDrafted`, `SectionGuardrailVerdictRecorded`, `CvReviewed`, `ApplicationApproved`, `ApplicationRejectedFinal`, `ReviewEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/applications` — body `{ jobTitle, description?, candidateProfile? }` → `{ applicationId }`. Starts a workflow.
- `GET /api/applications` — list all applications. Optional `?status=TAILORING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/applications/{id}` — one application (including every attempt and every review).
- `GET /api/applications/sse` — server-sent events stream of every application change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Scored CV Reviewer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, halt = red).
- **App UI** — form to submit a job posting, live list of applications with status pills, click-to-expand per-attempt timeline showing each CV draft, the guardrail verdict, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: Scored CV Reviewer</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `CvTailorAgent`): a deterministic check that the CV draft contains all three mandatory sections (Summary, Experience, Skills). Incomplete drafts short-circuit back to the Tailor with a structured feedback note (`reasonCode = MISSING_SECTIONS`); they never reach the Reviewer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's review is recorded as a `ReviewEvalRecorded` event with `{ attemptNumber, verdict, score, missingSections }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/applications/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `APPROVE`, the workflow ends with `ApplicationRejectedFinal`. The entity preserves every draft, every review, the highest-scoring draft's text, and a structured rejection reason. The system never deletes drafts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `CvTailorAgent` → `prompts/cv-tailor.md`. Produces a targeted CV for a job description; on a revision call, takes the prior `ReviewNotes` as input and produces an updated draft.
- `CvReviewerAgent` → `prompts/cv-reviewer.md`. Scores a CV draft against the fixed rubric; returns `APPROVE` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a job posting; application progresses `TAILORING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's draft and review.
2. **J2 — halt at ceiling** — Submit a job posting whose rubric is impossible to satisfy (test mode forces the Reviewer to `REVISE` every attempt); application progresses through every attempt and lands in `REJECTED_FINAL` with the best draft preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a job posting; the Tailor's first draft omits the mandatory Skills section; the guardrail short-circuits with `reasonCode = MISSING_SECTIONS`, the Tailor re-drafts with all sections, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any application shows one `ReviewEvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named scored-loop-tailor demonstrating the evaluator-optimizer ×
hr-recruiting cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-hr-recruiting-scored-loop-tailor.
Java package io.akka.samples.loopworkflowscoredcvreviewer. Akka 3.6.0. HTTP port 9587.

Components to wire (exactly):
- 2 AutonomousAgents:
  * CvTailorAgent — definition() with
    capability(TaskAcceptance.of(TAILOR_CV).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_CV).maxIterationsPerTask(3)).
    System prompt loaded from prompts/cv-tailor.md. Returns CvDraft{text,
    sectionsPresent, draftedAt} for both TAILOR_CV and REVISE_CV. The
    REVISE_CV task takes (jobPosting, priorDraft, ReviewNotes) as inputs.
  * CvReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW_CV).maxIterationsPerTask(2)). System
    prompt from prompts/cv-reviewer.md. Returns CvReview{verdict, notes,
    score, reviewedAt} where verdict is the ReviewVerdict enum (APPROVE |
    REVISE) and score is a 1–5 integer rubric.

- 1 Workflow TailoringWorkflow with steps:
    startStep -> tailorStep -> guardrailStep -> [guardrail FAIL? tailorStep
    again with structured feedback : reviewStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       tailorStep with review attached : rejectStep)] -> END.
  tailorStep calls forAutonomousAgent(CvTailorAgent.class, applicationId).runSingleTask(
    TAILOR_CV or REVISE_CV) then forTask(taskId).result(TAILOR_CV or
    REVISE_CV). reviewStep calls forAutonomousAgent(CvReviewerAgent.class,
    applicationId).runSingleTask(REVIEW_CV). approveStep emits ApplicationApproved.
    rejectStep emits ApplicationRejectedFinal with the highest-scoring draft's
    text as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on tailorStep and reviewStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    draft.sectionsPresent() contains "Summary", "Experience", "Skills". On
    FAIL, emits SectionGuardrailVerdictRecorded with verdict.passed = false
    and reasonCode = "MISSING_SECTIONS", then transitions back to tailorStep
    with a structured feedback ReviewNotes("CV is missing required sections;
    include Summary, Experience, and Skills before resubmitting.").

- 1 EventSourcedEntity ApplicationEntity holding state Application{applicationId,
  jobTitle, description, maxAttempts, ApplicationStatus status,
  List<CvAttempt> attempts, Optional<Integer> approvedAttemptNumber,
  Optional<String> approvedText, Optional<String> rejectionReason,
  Instant createdAt, Optional<Instant> finishedAt}. ApplicationStatus enum:
  TAILORING, REVIEWING, APPROVED, REJECTED_FINAL. Events: ApplicationCreated,
  CvDrafted, SectionGuardrailVerdictRecorded, CvReviewed, ApplicationApproved,
  ApplicationRejectedFinal, ReviewEvalRecorded. Commands: createApplication,
  recordDraft, recordGuardrail, recordReview, approve, rejectFinal,
  recordEval, getApplication. emptyState() returns Application.initial("",
  "", "", 4) with no commandContext() reference. Event-applier wraps
  lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity SubmissionQueue with command submitJobPosting(jobTitle,
  description, candidateProfile) emitting JobPostingSubmitted{applicationId,
  jobTitle, description, candidateProfile, submittedAt}.

- 1 View ApplicationsView with row type ApplicationRow (mirrors Application;
  the attempts list is preserved as-is — the list is bounded at maxAttempts
  so size stays reasonable). Table updater consumes ApplicationEntity events.
  ONE query getAllApplications SELECT * AS applications FROM
  applications_view. No WHERE status filter — caller filters client-side
  because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  JobPostingSubmitted starts a TailoringWorkflow with the applicationId as
  the workflow id.

- 2 TimedActions:
  * SubmissionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/job-postings.jsonl and calls
    SubmissionQueue.submitJobPosting.
  * EvalSampler — every 30s, queries ApplicationsView.getAllApplications,
    finds applications with a reviewed attempt that has not yet been
    recorded as a ReviewEvalRecorded event, and calls
    ApplicationEntity.recordEval(attemptNumber, verdict, score,
    missingSections). Idempotent per (applicationId, attemptNumber).

- 2 HttpEndpoints:
  * RecruitingEndpoint at /api with POST /applications, GET /applications,
    GET /applications/{id}, GET /applications/sse, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /applications body is
    {jobTitle, description?, candidateProfile?}; missing description
    defaults to "", missing candidateProfile defaults to "".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- RecruitingTasks.java declaring three Task<R> constants: TAILOR_CV
  (resultConformsTo CvDraft), REVISE_CV (CvDraft), REVIEW_CV (CvReview).
- Domain records CvDraft, SectionGuardrailVerdict, ReviewNotes, CvReview,
  CvAttempt, Application; enums ApplicationStatus, ReviewVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9587 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  scored-loop-tailor.tailoring.max-attempts = 4, overridable by env var.
- src/main/resources/sample-events/job-postings.jsonl with 8 canned job
  posting lines, each shaped {"jobTitle":"...", "description":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = cv-tailoring-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = true,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/cv-tailor.md, prompts/cv-reviewer.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Scored CV Reviewer",
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
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Scored CV Reviewer</title>.

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
  named in Section 9: cv-tailor.json, cv-reviewer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    cv-tailor.json — 6 CvDraft entries. Three are first-pass CVs with all
      three mandatory sections (Summary, Experience, Skills) present and
      targeted to different job descriptions. Two are revision CVs that
      directly address the review bullets. One is an intentionally
      incomplete CV missing the Skills section used to exercise the
      guardrail in J3.
    cv-reviewer.json — 6 CvReview entries. Three return verdict=APPROVE
      with score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a ReviewNotes payload of three
      bullets ("experience section lacks quantified achievements",
      "skills list does not align with job requirements",
      "summary does not reference the target role").
- A MockModelProvider.seedFor(applicationId, attemptNumber) helper makes
  the selection deterministic per (applicationId, attemptNumber) so the
  same application in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. CvTailorAgent
  and CvReviewerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a RecruitingTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Application row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: RecruitingTasks.java is mandatory; generating CvTailorAgent or
  CvReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9587, declared in application.conf
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
