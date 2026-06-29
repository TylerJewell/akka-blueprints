# SPEC — docreview

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DocReview.
**One-line pitch:** A user uploads a document and a list of review instructions; one AI agent reads the document (passed as a task attachment, never as inline prompt text) and returns a structured compliance verdict — PASS / FAIL / NEEDS_REVISION with a finding for each instruction.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the legal-compliance domain. One `DocumentReviewerAgent` (AutonomousAgent) carries the entire decision; the surrounding components only prepare its input and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw document submission and the agent call — so the model never sees identifiers.
- A **before-agent-response guardrail** validates the agent's verdict on every turn: well-formed JSON, every finding references a real instruction id, every severity is in the allowed set. A malformed verdict triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `VerdictRecorded` event, scoring the verdict on evidence quality (does every finding cite a section of the document? is the severity proportional? is the recommendation actionable?).

The blueprint shows that the single-agent pattern does not mean "ungoverned" — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a document into the **Document** textarea (or picks one of three seeded examples — a data-processing agreement, a vendor master service agreement, an open-source-licensed dependency notice).
2. The user picks an **instruction set** from a dropdown (DPA, MSA, OSS-notice) or pastes a custom list of review instructions.
3. The user clicks **Submit for review**. The UI POSTs to `/api/reviews` and receives a `reviewId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted document is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `reviewStep` completes. The card transitions to `REVIEWING` then `VERDICT_RECORDED`. The verdict appears: a top-level decision badge (PASS / FAIL / NEEDS_REVISION), a short summary paragraph, and a per-finding table (instruction id, severity, document section, recommendation).
6. Within ~1 s of the verdict, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the verdict's evidence is solid.
7. The user can submit another document; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewEndpoint` | `HttpEndpoint` | `/api/reviews/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ReviewEntity`, `ReviewView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ReviewEntity` | `EventSourcedEntity` | Per-review lifecycle: submitted → sanitized → reviewing → verdict → eval. Source of truth. | `ReviewEndpoint`, `DocumentSanitizer`, `ReviewWorkflow` | `ReviewView` |
| `DocumentSanitizer` | `Consumer` | Subscribes to `DocumentSubmitted` events; redacts PII; calls `ReviewEntity.attachSanitized`. | `ReviewEntity` events | `ReviewEntity` |
| `ReviewWorkflow` | `Workflow` | One workflow per review. Steps: `awaitSanitizedStep` → `reviewStep` → `evalStep`. | started by `DocumentSanitizer` once sanitized event lands | `DocumentReviewerAgent`, `ReviewEntity` |
| `DocumentReviewerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives review instructions in the task definition and the sanitized document as a task attachment; returns `ReviewVerdict`. | invoked by `ReviewWorkflow` | returns verdict |
| `ReviewView` | `View` | Read model: one row per review for the UI. | `ReviewEntity` events | `ReviewEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ReviewInstruction(String instructionId, String text, String severityFloor) {}

record ReviewRequest(
    String reviewId,
    String documentTitle,
    String rawDocument,
    List<ReviewInstruction> instructions,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedDocument(
    String redactedDocument,
    List<String> piiCategoriesFound
) {}

record Finding(
    String instructionId,
    Severity severity,
    String documentSection,
    String quote,
    String recommendation
) {}
enum Severity { LOW, MEDIUM, HIGH, CRITICAL }

record ReviewVerdict(
    Decision decision,
    String summary,
    List<Finding> findings,
    Instant decidedAt
) {}
enum Decision { PASS, FAIL, NEEDS_REVISION }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Review(
    String reviewId,
    Optional<ReviewRequest> request,
    Optional<SanitizedDocument> sanitized,
    Optional<ReviewVerdict> verdict,
    Optional<EvalResult> eval,
    ReviewStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReviewStatus {
    SUBMITTED, SANITIZED, REVIEWING, VERDICT_RECORDED, EVALUATED, FAILED
}
```

Events on `ReviewEntity`: `DocumentSubmitted`, `DocumentSanitized`, `ReviewStarted`, `VerdictRecorded`, `EvaluationScored`, `ReviewFailed`.

Every nullable lifecycle field on the `Review` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/reviews` — body `{ documentTitle, rawDocument, instructions: [ReviewInstruction], submittedBy }` → `{ reviewId }`.
- `GET /api/reviews` — list all reviews, newest-first.
- `GET /api/reviews/{id}` — one review.
- `GET /api/reviews/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: DocReview</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted reviews (status pill + decision badge + age) and a right pane with the selected review's detail — submitted instructions list, sanitized document preview, verdict summary, finding table, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `DocumentSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw document before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `DocumentReviewerAgent`. Asserts the candidate response is well-formed `ReviewVerdict` JSON, every `findings[].instructionId` matches a submitted instruction, every `severity` is in `{LOW, MEDIUM, HIGH, CRITICAL}`, and the verdict's `findings` list covers every instruction (no silent omissions). On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `VerdictRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call — the eval is rule-based on purpose, so the same decision always scores the same) checks that every finding cites a `documentSection` and a non-empty `quote`, that recommendations are actionable verbs, and that severity distributions are not collapsed (a 30-instruction review that returns all-LOW is suspect). Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `DocumentReviewerAgent` → `prompts/document-reviewer.md`. The single decision-making LLM. System prompt instructs it to read the attached document, walk every instruction, and return one `Finding` per instruction with a quoted document passage as evidence.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the DPA seed; within 30 s the verdict appears with one finding per submitted instruction and an eval score chip.
2. **J2** — The agent's first response on a review is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed verdict; the UI never displays the malformed response.
3. **J3** — A review whose findings all carry empty `quote` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A document containing `jane.doe@example.com` and `SSN 123-45-6789` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-SSN]`; the entity's `request.rawDocument` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named docreview demonstrating the single-agent × legal-compliance cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact docreview.
Java package io.akka.samples.docreview. Akka 3.6.0. HTTP port 9144.

Components to wire (exactly):

- 1 AutonomousAgent DocumentReviewerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/document-reviewer.md>) and
  .capability(TaskAcceptance.of(REVIEW_DOCUMENT).maxIterationsPerTask(3)). The task receives
  review instructions as its instruction text and the sanitized document as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: ReviewVerdict{decision: Decision (PASS/FAIL/NEEDS_REVISION), summary: String,
  findings: List<Finding>, decidedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow ReviewWorkflow per reviewId with three steps:
  * awaitSanitizedStep — polls ReviewEntity.getReview every 1s; on review.sanitized().isPresent()
    advances to reviewStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * reviewStep — emits ReviewStarted, then calls componentClient.forAutonomousAgent(
    DocumentReviewerAgent.class, "reviewer-" + reviewId).runSingleTask(
      TaskDef.instructions(formatInstructions(review.request.instructions))
        .attachment("document.txt", review.sanitized.redactedDocument.getBytes())
    ) — returns a taskId, then forTask(taskId).result(REVIEW_DOCUMENT) to fetch the verdict.
    On success calls ReviewEntity.recordVerdict(verdict). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(ReviewWorkflow::error).
  * evalStep — runs a deterministic rule-based EvaluationScorer (NOT an LLM call) over the
    recorded verdict: checks that every finding has non-empty quote and documentSection, that
    recommendations are actionable, and that severity distribution is not pathologically
    flattened. Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings
    .stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ReviewEntity (one per reviewId). State Review{reviewId: String,
  request: Optional<ReviewRequest>, sanitized: Optional<SanitizedDocument>,
  verdict: Optional<ReviewVerdict>, eval: Optional<EvalResult>, status: ReviewStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. ReviewStatus enum: SUBMITTED,
  SANITIZED, REVIEWING, VERDICT_RECORDED, EVALUATED, FAILED. Events: DocumentSubmitted{request},
  DocumentSanitized{sanitized}, ReviewStarted{}, VerdictRecorded{verdict},
  EvaluationScored{eval}, ReviewFailed{reason}. Commands: submit, attachSanitized,
  markReviewing, recordVerdict, recordEvaluation, fail, getReview. emptyState() returns
  Review.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer DocumentSanitizer subscribed to ReviewEntity events; on DocumentSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over rawDocument, computes
  the list of categories found, builds SanitizedDocument, then calls
  ReviewEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a ReviewWorkflow with id = "review-" + reviewId.

- 1 View ReviewView with row type ReviewRow (mirrors Review minus request.rawDocument — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes ReviewEntity events. ONE query getAllReviews: SELECT * AS reviews FROM review_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /reviews (body
    {documentTitle, rawDocument, instructions: [{instructionId, text, severityFloor}],
    submittedBy}; mints reviewId; calls ReviewEntity.submit; returns {reviewId}), GET /reviews
    (list from getAllReviews, sorted newest-first), GET /reviews/{id} (one row), GET
    /reviews/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ReviewTasks.java declaring one Task<R> constant: REVIEW_DOCUMENT = Task.name("Review
  document").description("Read the attached document and produce a ReviewVerdict per
  instruction").resultConformsTo(ReviewVerdict.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records ReviewInstruction, ReviewRequest, SanitizedDocument, Finding, Severity,
  ReviewVerdict, Decision, EvalResult, Review, ReviewStatus.

- VerdictGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ReviewVerdict from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- EvaluationScorer.java — pure deterministic logic (no LLM). Inputs: ReviewVerdict and the
  list of submitted ReviewInstruction. Outputs: EvalResult. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9144 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The DocumentReviewerAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/review-instructions.jsonl with 3 seeded instruction sets:
  a 5-clause DPA checklist, a 7-clause MSA checklist, and a 4-clause OSS-notice checklist.

- src/main/resources/sample-events/seed-documents.jsonl with 3 paired example documents:
  a synthetic DPA (1200 words), a synthetic MSA (1600 words), and a synthetic OSS-notice
  block (350 words). Each contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors —
  legal-compliance is the use case, not the anchor.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the agent's verdict is advisory, not enforced), oversight.human_in_loop = true (a human
  reads the verdict before acting on it), failure.failure_modes including
  "hallucinated-finding", "missed-instruction", "pii-leakage-via-llm",
  "severity-miscalibration"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/document-reviewer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: DocReview", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of review cards; right = selected-review detail with submitted instructions,
  sanitized document preview, verdict summary, finding table, and eval-score chip).
  Browser title exactly: <title>Akka Sample: DocReview</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(reviewId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    review-document.json — 8 ReviewVerdict entries covering the three Decision values.
      Each entry has a summary paragraph and a `findings` array with one Finding per
      instruction in the matched instruction set (DPA / MSA / OSS-notice). Each Finding has
      a non-empty documentSection (e.g., "§4.2"), a non-empty quote (a paraphrase referring
      to a passage of the seeded document), and an actionable recommendation. Severities
      vary realistically. Plus 2 deliberately MALFORMED entries (one with a finding whose
      instructionId is not in the submitted list; one with a severity value outside the
      enum) — the guardrail blocks both, exercising the retry path. The mock should select
      a malformed entry on the FIRST iteration of every 3rd review (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(reviewId) helper makes per-review selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DocumentReviewerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ReviewTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (reviewStep
  60s, awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Review row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ReviewTasks.java with REVIEW_DOCUMENT = Task.name(...).description(...)
  .resultConformsTo(ReviewVerdict.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9144 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (DocumentReviewerAgent). The on-decision eval is rule-based (EvaluationScorer.java) and
  does NOT make an LLM call — keeping the pattern's "one agent" promise honest.
- The document is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated reviewStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
