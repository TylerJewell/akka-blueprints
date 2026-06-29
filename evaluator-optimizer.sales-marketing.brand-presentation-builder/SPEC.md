# SPEC — brand-presentation-builder

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Brand-Aligned Presentations.
**One-line pitch:** Submit a topic and a target audience; a builder agent assembles a slide deck; a brand reviewer agent scores every slide against brand guidelines; the two iterate until the reviewer approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`BuilderAgent`) and a reviewer agent (`BrandReviewerAgent`), feeding each round of brand feedback back into the next draft until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's brand verdict for downstream quality measurement, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the presentation in a degenerate state. A brand guardrail (deterministic word-count check per slide) gates each draft before the reviewer runs, preventing non-compliant material from consuming a reviewer call.

## 3. User-facing flows

The user opens the App UI tab and submits a brief (a topic, a target audience, and a slide-count target).

1. The system creates a `Presentation` record in `BUILDING` and starts a `PresentationWorkflow`.
2. The Builder drafts attempt #1: a structured slide set covering title, agenda, content slides, and a closing slide, each under the per-slide word ceiling.
3. The brand guardrail vets each slide's word count against the per-slide ceiling. Any slide that exceeds the ceiling short-circuits back to the Builder with a deterministic feedback note; the draft never reaches the Reviewer.
4. The Brand Reviewer scores the slide set against a fixed rubric (tone alignment, colour-palette references, logo-usage language, messaging hierarchy, call-to-action clarity) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `BrandFeedback` payload (up to five bullets).
5. On `APPROVE`, the workflow transitions the presentation to `APPROVED` with the winning slide set and the reviewer's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the feedback, and the reviewer's verdict on the entity, then calls the Builder again with the feedback attached. The Builder produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every round of feedback for audit, and a `BrandEvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BuilderAgent` | `AutonomousAgent` | Drafts a slide deck on a brief; accepts prior brand feedback on revisions. | `PresentationWorkflow` | returns `SlideSet` to workflow |
| `BrandReviewerAgent` | `AutonomousAgent` | Scores a slide set against brand guidelines; returns `APPROVE` or `REVISE` with feedback. | `PresentationWorkflow` | returns `BrandReview` to workflow |
| `PresentationWorkflow` | `Workflow` | Runs the build → guardrail → brand-review → revise loop; halts at the ceiling. | `PresentationEndpoint`, `PresentationRequestConsumer` | `PresentationEntity` |
| `PresentationEntity` | `EventSourcedEntity` | Holds the presentation lifecycle, every attempt, every round of brand feedback, and the final outcome. | `PresentationWorkflow` | `PresentationsView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted brief for replay and audit. | `PresentationEndpoint`, `RequestSimulator` | `PresentationRequestConsumer` |
| `PresentationsView` | `View` | List-of-presentations read model. | `PresentationEntity` events | `PresentationEndpoint` |
| `PresentationRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `PresentationWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/presentation-briefs.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `PresentationsView`, records a `BrandEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `PresentationEntity` |
| `PresentationEndpoint` | `HttpEndpoint` | `/api/presentations/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `PresentationsView`, `RequestQueue`, `PresentationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record PresentationBrief(String topic, String targetAudience, int slideCount, int wordsPerSlide, String requestedBy) {}

record Slide(int slideNumber, String title, String body, int wordCount) {}

record SlideSet(List<Slide> slides, int totalWordCount, Instant builtAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record BrandFeedback(List<String> bullets, String overallRationale) {}

record BrandReview(ReviewVerdict verdict, BrandFeedback feedback, int score, Instant reviewedAt) {}

record Attempt(
    int attemptNumber,
    SlideSet slideSet,
    GuardrailVerdict guardrail,
    Optional<BrandReview> brandReview
) {}

record Presentation(
    String presentationId,
    String topic,
    String targetAudience,
    int slideCount,
    int wordsPerSlide,
    int maxAttempts,
    PresentationStatus status,
    List<Attempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<SlideSet> approvedSlideSet,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PresentationStatus { BUILDING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewVerdict { APPROVE, REVISE }
```

### Events (on `PresentationEntity`)

`PresentationCreated`, `AttemptBuilt`, `AttemptGuardrailVerdictRecorded`, `AttemptBrandReviewed`, `PresentationApproved`, `PresentationRejectedFinal`, `BrandEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/presentations` — body `{ topic, targetAudience?, slideCount?, wordsPerSlide?, requestedBy? }` → `{ presentationId }`. Starts a workflow.
- `GET /api/presentations` — list all presentations. Optional `?status=BUILDING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/presentations/{id}` — one presentation (including every attempt and every round of brand feedback).
- `GET /api/presentations/sse` — server-sent events stream of every presentation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Brand-Aligned Presentations"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit a brief, live list of presentations with status pills, click-to-expand per-attempt timeline showing each slide set, the guardrail verdict, the reviewer's verdict, and the brand feedback.

Browser title: `<title>Akka Sample: Brand-Aligned Presentations</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — brand guardrail** (`before-agent-response` on `BuilderAgent`): a deterministic check that no individual slide's word count exceeds the per-slide ceiling. Over-limit drafts short-circuit back to the Builder with a structured feedback note (`reasonCode = OVER_WORD_LIMIT`); they never reach the Reviewer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's brand review verdict is recorded as a `BrandEvalRecorded` event with `{ attemptNumber, verdict, score, guardedSlideNumbers }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/presentations/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `APPROVE`, the workflow ends with `PresentationRejectedFinal`. The entity preserves every attempt, every round of feedback, the best-scoring attempt's slide set, and a structured rejection reason. The system never deletes drafts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `BuilderAgent` → `prompts/builder.md`. Drafts a structured slide deck on the brief; on a revision call, takes the prior `BrandFeedback` as input and produces a new slide set.
- `BrandReviewerAgent` → `prompts/brand-reviewer.md`. Scores a slide set against brand guidelines; returns `APPROVE` with a one-line rationale or `REVISE` with up to five short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a brief; presentation progresses `BUILDING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's slide set and brand review.
2. **J2 — halt at ceiling** — Submit a brief whose rubric is impossible (test mode forces the Reviewer to `REVISE` every attempt); presentation progresses through every attempt and lands in `REJECTED_FINAL` with the best slide set preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a brief with `wordsPerSlide = 30`; the Builder's first draft has at least one slide that exceeds the limit; the guardrail short-circuits with `reasonCode = OVER_WORD_LIMIT`, the Builder re-drafts under the ceiling, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any presentation shows one `BrandEvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named brand-presentation-builder demonstrating the evaluator-optimizer ×
sales-marketing cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-sales-marketing-brand-presentation-builder.
Java package io.akka.samples.brandalignedpresentations. Akka 3.6.0. HTTP port 9366.

Components to wire (exactly):
- 2 AutonomousAgents:
  * BuilderAgent — definition() with
    capability(TaskAcceptance.of(BUILD).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_BUILD).maxIterationsPerTask(3)).
    System prompt loaded from prompts/builder.md. Returns SlideSet{slides,
    totalWordCount, builtAt} for both BUILD and REVISE_BUILD. The
    REVISE_BUILD task takes (originalBrief, priorSlideSet, BrandFeedback) as
    inputs.
  * BrandReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/brand-reviewer.md. Returns BrandReview{verdict, feedback,
    score, reviewedAt} where verdict is the ReviewVerdict enum (APPROVE | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow PresentationWorkflow with steps:
    startStep -> buildStep -> guardrailStep -> [guardrail FAIL? buildStep
    again with structured feedback : reviewStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       buildStep with feedback attached : rejectStep)] -> END.
  buildStep calls forAutonomousAgent(BuilderAgent.class, presentationId).runSingleTask(
    BUILD or REVISE_BUILD) then forTask(taskId).result(BUILD or
    REVISE_BUILD). reviewStep calls forAutonomousAgent(BrandReviewerAgent.class,
    presentationId).runSingleTask(REVIEW). approveStep emits PresentationApproved.
    rejectStep emits PresentationRejectedFinal with the highest-scoring attempt's
    slide set as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on buildStep and reviewStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks that every
    slide.wordCount() <= wordsPerSlide. On FAIL, emits
    AttemptGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_WORD_LIMIT" (detail names the offending slide numbers),
    then transitions back to buildStep with a structured feedback
    BrandFeedback("Slides N exceed the configured word limit; trim those slides.").

- 1 EventSourcedEntity PresentationEntity holding state Presentation{
  presentationId, topic, targetAudience, slideCount, wordsPerSlide, maxAttempts,
  PresentationStatus status, List<Attempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<SlideSet> approvedSlideSet,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. PresentationStatus enum: BUILDING, REVIEWING, APPROVED,
  REJECTED_FINAL. Events: PresentationCreated, AttemptBuilt,
  AttemptGuardrailVerdictRecorded, AttemptBrandReviewed, PresentationApproved,
  PresentationRejectedFinal, BrandEvalRecorded. Commands: createPresentation,
  recordBuild, recordGuardrail, recordBrandReview, approve, rejectFinal,
  recordBrandEval, getPresentation. emptyState() returns
  Presentation.initial("", "", "", 5, 80, 4) with no commandContext()
  reference. Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueBrief(topic,
  targetAudience, slideCount, wordsPerSlide, requestedBy) emitting
  BriefSubmitted{presentationId, topic, targetAudience, slideCount,
  wordsPerSlide, requestedBy, submittedAt}.

- 1 View PresentationsView with row type PresentationRow (mirrors
  Presentation; the attempts list is preserved as-is — the list is bounded
  at maxAttempts so size stays reasonable). Table updater consumes
  PresentationEntity events. ONE query getAllPresentations SELECT * AS
  presentations FROM presentations_view. No WHERE status filter — caller
  filters client-side because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer PresentationRequestConsumer subscribed to RequestQueue events;
  on BriefSubmitted starts a PresentationWorkflow with the presentationId as
  the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/presentation-briefs.jsonl and calls
    RequestQueue.enqueueBrief.
  * EvalSampler — every 30s, queries PresentationsView.getAllPresentations,
    finds presentations with a reviewed attempt that has not yet been
    recorded as a BrandEvalRecorded event, and calls
    PresentationEntity.recordBrandEval(attemptNumber, verdict, score,
    guardedSlideNumbers). Idempotent per (presentationId, attemptNumber).

- 2 HttpEndpoints:
  * PresentationEndpoint at /api with POST /presentations, GET /presentations,
    GET /presentations/{id}, GET /presentations/sse, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /presentations body is
    {topic, targetAudience?, slideCount?, wordsPerSlide?, requestedBy?};
    missing targetAudience defaults to "general", missing slideCount
    defaults to 5, missing wordsPerSlide defaults to 80, missing requestedBy
    defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PresentationTasks.java declaring three Task<R> constants: BUILD (resultConformsTo
  SlideSet), REVISE_BUILD (SlideSet), REVIEW (BrandReview).
- Domain records Slide, SlideSet, GuardrailVerdict, BrandFeedback, BrandReview,
  Attempt, Presentation; enums PresentationStatus, ReviewVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9366 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  brand-presentation.refinement.max-attempts = 4 and
  brand-presentation.refinement.default-words-per-slide = 80, overridable by
  env var.
- src/main/resources/sample-events/presentation-briefs.jsonl with 8 canned brief
  lines, each shaped {"topic":"...", "targetAudience":"...", "slideCount":5, "wordsPerSlide":80}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 brand guardrail
  before-agent-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = sales-presentation-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/builder.md, prompts/brand-reviewer.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Brand-Aligned Presentations",
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
  <title>Akka Sample: Brand-Aligned Presentations</title>.

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
  named in Section 9: builder.json, brand-reviewer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    builder.json — 6 SlideSet entries. Three are "first-pass" slide sets
      of 5 slides each, all under 80 words per slide, covering a product
      launch topic. Two are "revision" slide sets that visibly incorporate
      brand-feedback bullets (tightened tone on slide 2, revised CTA on
      slide 5). One is an intentionally over-limit slide set where slide 3
      exceeds 100 words, used to exercise the guardrail in J3.
    brand-reviewer.json — 6 BrandReview entries. Three return
      verdict=APPROVE with score=4 or 5 and a one-sentence rationale.
      Three return verdict=REVISE with score=2 or 3 and a BrandFeedback
      payload of up to five bullets ("slide 2 uses informal tone not
      aligned to brand voice", "slide 4 references competitor product by
      name", "CTA on slide 5 missing brand-standard call-to-action phrase",
      "colour palette references use wrong hex codes", "logo-usage language
      does not match brand guidelines §3").
- A MockModelProvider.seedFor(presentationId, attemptNumber) helper makes
  the selection deterministic per (presentationId, attemptNumber) so the
  same presentation in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. BuilderAgent
  and BrandReviewerAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a
  PresentationTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Presentation row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: PresentationTasks.java is mandatory; generating BuilderAgent or
  BrandReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9366, declared in application.conf
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
