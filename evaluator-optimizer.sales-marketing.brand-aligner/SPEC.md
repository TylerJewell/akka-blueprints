# SPEC — brand-aligner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Brand Aligner.
**One-line pitch:** Submit a marketing brief; a copywriter agent generates a variant; a brand-compliance reviewer scores it against tone, vocabulary, claim-accuracy, and visual-reference standards; the two iterate until the reviewer approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`CopywriterAgent`) and a reviewer agent (`BrandReviewerAgent`), feeding each review back into the next variant until convergence or a halt. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's verdict for downstream brand-quality measurement, a **compliance check** that gates each variant against a deterministic word-count rule before the reviewer runs, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the material in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a brief (a campaign topic, target audience, and a word-count ceiling).

1. The system creates a `Material` record in `DRAFTING` and starts an `AlignmentWorkflow`.
2. The Copywriter generates variant #1: marketing copy on the brief, in the configured brand voice.
3. The compliance check vets the variant against the word-count ceiling. Over-length variants are short-circuited back to the Copywriter with a deterministic feedback note; they never reach the Reviewer.
4. The Reviewer scores the variant against the brand rubric (tone, vocabulary, claim accuracy, visual-reference compliance) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `ReviewNotes` payload (up to three bullets).
5. On `APPROVE`, the workflow transitions the material to `APPROVED` with the winning variant's text and the reviewer's rationale.
6. On `REVISE`, the workflow records the attempt, the compliance-check verdict, the review, and the reviewer's verdict on the entity, then calls the Copywriter again with the review attached. The Copywriter produces variant #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring variant is preserved on the entity along with every review for audit, and a `BrandEvalRecorded` event captures the rejection.

A `CampaignSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CopywriterAgent` | `AutonomousAgent` | Generates brand-voice marketing copy on a brief; accepts prior review notes on revisions. | `AlignmentWorkflow` | returns `CopyVariant` to workflow |
| `BrandReviewerAgent` | `AutonomousAgent` | Scores a variant against the brand rubric; returns `APPROVE` or `REVISE` with notes. | `AlignmentWorkflow` | returns `BrandReview` to workflow |
| `AlignmentWorkflow` | `Workflow` | Runs the generate → compliance-check → review → revise loop; halts at the ceiling. | `BrandEndpoint`, `BriefConsumer` | `MaterialEntity` |
| `MaterialEntity` | `EventSourcedEntity` | Holds the material lifecycle, every variant, every review, and the final outcome. | `AlignmentWorkflow` | `MaterialsView` |
| `BriefQueue` | `EventSourcedEntity` | Logs each submitted brief for replay and audit. | `BrandEndpoint`, `CampaignSimulator` | `BriefConsumer` |
| `MaterialsView` | `View` | List-of-materials read model. | `MaterialEntity` events | `BrandEndpoint` |
| `BriefConsumer` | `Consumer` | Subscribes to `BriefQueue` events; starts a workflow per submission. | `BriefQueue` events | `AlignmentWorkflow` |
| `CampaignSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/campaign-briefs.jsonl`. | scheduler | `BriefQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `MaterialsView`, records a `BrandEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `MaterialEntity` |
| `BrandEndpoint` | `HttpEndpoint` | `/api/materials/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `MaterialsView`, `BriefQueue`, `MaterialEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record CampaignBrief(String topic, String targetAudience, int wordCeiling, String requestedBy) {}

record CopyVariant(String text, int wordCount, Instant generatedAt) {}

record ComplianceVerdict(boolean passed, String reasonCode, String detail) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record BrandReview(ReviewerVerdict verdict, ReviewNotes notes, int score, Instant reviewedAt) {}

record VariantAttempt(
    int attemptNumber,
    CopyVariant variant,
    ComplianceVerdict compliance,
    Optional<BrandReview> review
) {}

record Material(
    String materialId,
    String topic,
    String targetAudience,
    int wordCeiling,
    int maxAttempts,
    MaterialStatus status,
    List<VariantAttempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<String> approvedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MaterialStatus { DRAFTING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewerVerdict { APPROVE, REVISE }
```

### Events (on `MaterialEntity`)

`MaterialCreated`, `VariantGenerated`, `ComplianceVerdictRecorded`, `VariantReviewed`, `MaterialApproved`, `MaterialRejectedFinal`, `BrandEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/materials` — body `{ topic, targetAudience?, wordCeiling?, requestedBy? }` → `{ materialId }`. Starts a workflow.
- `GET /api/materials` — list all materials. Optional `?status=DRAFTING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/materials/{id}` — one material (including every attempt and every review).
- `GET /api/materials/sse` — server-sent events stream of every material change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Brand Aligner"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, compliance-check = red, halt = red).
- **App UI** — form to submit a brief, live list of materials with status pills, click-to-expand per-attempt timeline showing each variant, the compliance verdict, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: Brand Aligner</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **CC1 — compliance check** (`before-agent-response` on `CopywriterAgent`): a deterministic check that the variant's word count is at or below the per-material ceiling. Over-length variants short-circuit back to the Copywriter with a structured feedback note (`reasonCode = OVER_WORD_CEILING`); they never reach the Reviewer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's review is recorded as a `BrandEvalRecorded` event with `{ attemptNumber, verdict, score, wordCeilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/materials/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `APPROVE`, the workflow ends with `MaterialRejectedFinal`. The entity preserves every variant, every review, the best-scoring variant's text, and a structured rejection reason. The system never deletes drafts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `CopywriterAgent` → `prompts/copywriter.md`. Generates brand-voice marketing copy on the brief; on a revision call, takes the prior `ReviewNotes` as input and produces a revised variant.
- `BrandReviewerAgent` → `prompts/brand-reviewer.md`. Scores a variant against the brand rubric; returns `APPROVE` with a one-line rationale or `REVISE` with up to three specific bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a brief; material progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's variant and review.
2. **J2 — halt at ceiling** — Submit a brief whose rubric is impossible (test mode forces the Reviewer to `REVISE` every attempt); material progresses through every attempt and lands in `REJECTED_FINAL` with the best variant preserved and a structured rejection reason.
3. **J3 — compliance check block** — Submit a brief with `wordCeiling = 20`; the Copywriter's first variant exceeds the ceiling; the compliance check short-circuits with `reasonCode = OVER_WORD_CEILING`, the Copywriter re-drafts under the ceiling, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any material shows one `BrandEvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named brand-aligner demonstrating the evaluator-optimizer ×
sales-marketing cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-sales-marketing-brand-aligner.
Java package io.akka.samples.brandaligner. Akka 3.6.0. HTTP port 9276.

Components to wire (exactly):
- 2 AutonomousAgents:
  * CopywriterAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_VARIANT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/copywriter.md. Returns CopyVariant{text,
    wordCount, generatedAt} for both GENERATE and REVISE_VARIANT. The
    REVISE_VARIANT task takes (originalBrief, priorVariant, ReviewNotes) as
    inputs.
  * BrandReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/brand-reviewer.md. Returns BrandReview{verdict, notes,
    score, reviewedAt} where verdict is the ReviewerVerdict enum (APPROVE |
    REVISE) and score is a 1–5 integer rubric.

- 1 Workflow AlignmentWorkflow with steps:
    startStep -> generateStep -> complianceStep ->
    [compliance FAIL? generateStep again with structured feedback :
    reviewStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       generateStep with review attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(CopywriterAgent.class, materialId)
    .runSingleTask(GENERATE or REVISE_VARIANT) then forTask(taskId)
    .result(GENERATE or REVISE_VARIANT). reviewStep calls
    forAutonomousAgent(BrandReviewerAgent.class, materialId)
    .runSingleTask(REVIEW). approveStep emits MaterialApproved. rejectStep
    emits MaterialRejectedFinal with the highest-scoring variant's text as
    best-of and a structured rejectionReason. Override settings() with
    stepTimeout(60s) on generateStep and reviewStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  complianceStep is a pure-function step (no LLM call): checks
    variant.wordCount() <= ceiling. On FAIL, emits
    ComplianceVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_WORD_CEILING", then transitions back to generateStep
    with a structured feedback ReviewNotes("Variant exceeds the configured
    word ceiling; shorten and resubmit.").

- 1 EventSourcedEntity MaterialEntity holding state Material{materialId,
  topic, targetAudience, wordCeiling, maxAttempts, MaterialStatus status,
  List<VariantAttempt> attempts, Optional<Integer> approvedAttemptNumber,
  Optional<String> approvedText, Optional<String> rejectionReason,
  Instant createdAt, Optional<Instant> finishedAt}. MaterialStatus enum:
  DRAFTING, REVIEWING, APPROVED, REJECTED_FINAL. Events: MaterialCreated,
  VariantGenerated, ComplianceVerdictRecorded, VariantReviewed,
  MaterialApproved, MaterialRejectedFinal, BrandEvalRecorded. Commands:
  createMaterial, recordVariant, recordCompliance, recordReview, approve,
  rejectFinal, recordEval, getMaterial. emptyState() returns
  Material.initial("", "", "", 150, 4) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity BriefQueue with command enqueueBrief(topic,
  targetAudience, wordCeiling, requestedBy) emitting
  BriefSubmitted{materialId, topic, targetAudience, wordCeiling,
  requestedBy, submittedAt}.

- 1 View MaterialsView with row type MaterialRow (mirrors Material; the
  attempts list is preserved as-is — the list is bounded at maxAttempts so
  size stays reasonable). Table updater consumes MaterialEntity events. ONE
  query getAllMaterials SELECT * AS materials FROM materials_view. No WHERE
  status filter — caller filters client-side because Akka cannot auto-index
  enum columns (Lesson 2).

- 1 Consumer BriefConsumer subscribed to BriefQueue events; on
  BriefSubmitted starts an AlignmentWorkflow with the materialId as the
  workflow id.

- 2 TimedActions:
  * CampaignSimulator — every 60s, reads next line from
    src/main/resources/sample-events/campaign-briefs.jsonl and calls
    BriefQueue.enqueueBrief.
  * EvalSampler — every 30s, queries MaterialsView.getAllMaterials, finds
    materials with a reviewed attempt that has not yet been recorded as a
    BrandEvalRecorded event, and calls MaterialEntity.recordEval(attemptNumber,
    verdict, score, wordCeilingExceeded). Idempotent per (materialId,
    attemptNumber).

- 2 HttpEndpoints:
  * BrandEndpoint at /api with POST /materials, GET /materials,
    GET /materials/{id}, GET /materials/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /materials body is
    {topic, targetAudience?, wordCeiling?, requestedBy?}; missing
    wordCeiling defaults to 150, missing targetAudience defaults to
    "general", missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- BrandTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  CopyVariant), REVISE_VARIANT (CopyVariant), REVIEW (BrandReview).
- Domain records CopyVariant, ComplianceVerdict, ReviewNotes, BrandReview,
  VariantAttempt, Material; enums MaterialStatus, ReviewerVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9276 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  brand-aligner.alignment.max-attempts = 4 and
  brand-aligner.alignment.default-word-ceiling = 150, overridable by
  env var.
- src/main/resources/sample-events/campaign-briefs.jsonl with 8 canned
  brief lines, each shaped {"topic":"...", "targetAudience":"...",
  "wordCeiling":150}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (CC1 compliance-check
  before-agent-response, E1 eval-event on-decision-eval) and a HT1 halt
  control, each with a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = marketing-copy-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/copywriter.md, prompts/brand-reviewer.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Brand Aligner",
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
  <title>Akka Sample: Brand Aligner</title>.

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
  named in Section 9: copywriter.json, brand-reviewer.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    copywriter.json — 6 CopyVariant entries. Three are first-pass copy
      variants between 120 and 148 words in a clear brand voice on the
      topics in campaign-briefs.jsonl. Two are revised variants that
      address prior review notes (tighter claims, adjusted tone). One is
      an intentionally over-ceiling variant (>180 words) used to exercise
      the compliance check in J3.
    brand-reviewer.json — 6 BrandReview entries. Three return
      verdict=APPROVE with score=4 or 5 and a one-sentence rationale.
      Three return verdict=REVISE with score=2 or 3 and a ReviewNotes
      payload of up to three bullets ("tone is too aggressive for the
      target segment", "product claim is unsubstantiated — remove or cite",
      "competitor name appears in copy — remove").
- A MockModelProvider.seedFor(materialId, attemptNumber) helper makes the
  selection deterministic per (materialId, attemptNumber) so the same
  material in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. CopywriterAgent
  and BrandReviewerAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a BrandTasks
  companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Material row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: BrandTasks.java is mandatory; generating CopywriterAgent or
  BrandReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9276, declared in application.conf
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
