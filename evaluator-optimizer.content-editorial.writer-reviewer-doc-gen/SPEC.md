# SPEC — writer-reviewer-doc-gen

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SK Document Generator.
**One-line pitch:** Type a topic; a writer agent produces a structured document; a reviewer agent evaluates it against quality criteria; the two iterate until the reviewer approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`WriterAgent`) and a reviewer agent (`ReviewerAgent`), feeding each review back into the next draft until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement and an **output guardrail** that gates each draft against a deterministic word-count rule before the reviewer runs.

## 3. User-facing flows

The user opens the App UI tab and submits a topic (a subject plus a target word-count ceiling).

1. The system creates a `Document` record in `DRAFTING` and starts a `DocumentWorkflow`.
2. The Writer produces draft #1: a structured document on the topic.
3. The output guardrail checks the draft against the word-count ceiling. Over-length drafts are short-circuited back to the Writer with a deterministic feedback note; they never reach the Reviewer.
4. The Reviewer evaluates the draft against a fixed rubric (structure, accuracy signals, clarity, length compliance) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `ReviewNotes` payload (three bullets at most).
5. On `APPROVE`, the workflow transitions the document to `APPROVED` with the winning draft's text and the reviewer's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the review, and the reviewer's verdict on the entity, then calls the Writer again with the review notes attached. The Writer produces draft #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring draft is preserved on the entity along with every review for audit, and an `EvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned topic every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WriterAgent` | `AutonomousAgent` | Produces a structured document on a topic; accepts prior review notes on revisions. | `DocumentWorkflow` | returns `DocumentDraft` to workflow |
| `ReviewerAgent` | `AutonomousAgent` | Evaluates a draft against the rubric; returns `APPROVE` or `REVISE` with notes. | `DocumentWorkflow` | returns `Review` to workflow |
| `DocumentWorkflow` | `Workflow` | Runs the write → guardrail → review → revise loop; halts at the ceiling. | `DocumentEndpoint`, `DocRequestConsumer` | `DocumentEntity` |
| `DocumentEntity` | `EventSourcedEntity` | Holds the document lifecycle, every draft attempt, every review, and the final outcome. | `DocumentWorkflow` | `DocumentsView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted topic for replay and audit. | `DocumentEndpoint`, `RequestSimulator` | `DocRequestConsumer` |
| `DocumentsView` | `View` | List-of-documents read model. | `DocumentEntity` events | `DocumentEndpoint` |
| `DocRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `DocumentWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample topic every 60 s from `sample-events/doc-topics.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `DocumentsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `DocumentEntity` |
| `DocumentEndpoint` | `HttpEndpoint` | `/api/documents/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `DocumentsView`, `RequestQueue`, `DocumentEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TopicRequest(String topic, int wordCeiling, String requestedBy) {}

record DocumentDraft(String text, int wordCount, Instant draftedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record Review(ReviewVerdict verdict, ReviewNotes notes, int score, Instant evaluatedAt) {}

record DraftAttempt(
    int attemptNumber,
    DocumentDraft draft,
    GuardrailVerdict guardrail,
    Optional<Review> review
) {}

record Document(
    String documentId,
    String topic,
    int wordCeiling,
    int maxAttempts,
    DocumentStatus status,
    List<DraftAttempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<String> approvedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DocumentStatus { DRAFTING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewVerdict { APPROVE, REVISE }
```

### Events (on `DocumentEntity`)

`DocumentCreated`, `DraftProduced`, `DraftGuardrailVerdictRecorded`, `DraftReviewed`, `DocumentApproved`, `DocumentRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/documents` — body `{ topic, wordCeiling?, requestedBy? }` → `{ documentId }`. Starts a workflow.
- `GET /api/documents` — list all documents. Optional `?status=DRAFTING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/documents/{id}` — one document (including every draft attempt and every review).
- `GET /api/documents/sse` — server-sent events stream of every document change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "SK Document Generator"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a topic, live list of documents with status pills, click-to-expand per-attempt timeline showing each draft, the guardrail verdict, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: SK Document Generator</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `WriterAgent`): a deterministic check that the draft's word count is at or below the per-document ceiling. Over-length drafts short-circuit back to the Writer with a structured feedback note (`reasonCode = OVER_CEILING`); they never reach the Reviewer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's review is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, ceilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/documents/{id}`.

## 9. Agent prompts

- `WriterAgent` → `prompts/writer.md`. Produces a structured document on the topic; on a revision call, takes the prior `ReviewNotes` as input and produces a revised draft.
- `ReviewerAgent` → `prompts/reviewer.md`. Evaluates a draft against the fixed rubric; returns `APPROVE` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a topic; document progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's draft and review.
2. **J2 — halt at ceiling** — Submit a topic whose rubric is impossible (test mode forces the Reviewer to `REVISE` every attempt); document progresses through every attempt and lands in `REJECTED_FINAL` with the best draft preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a topic with `wordCeiling = 50`; the Writer's first draft exceeds the ceiling; the guardrail short-circuits with `reasonCode = OVER_CEILING`, the Writer re-drafts under the ceiling, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any document shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named writer-reviewer-doc-gen demonstrating the evaluator-optimizer ×
content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-content-editorial-writer-reviewer-doc-gen.
Java package io.akka.samples.skdocumentgenerator. Akka 3.6.0. HTTP port 9885.

Components to wire (exactly):
- 2 AutonomousAgents:
  * WriterAgent — definition() with
    capability(TaskAcceptance.of(WRITE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DRAFT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/writer.md. Returns DocumentDraft{text,
    wordCount, draftedAt} for both WRITE and REVISE_DRAFT. The
    REVISE_DRAFT task takes (originalTopic, priorDraft, ReviewNotes) as
    inputs.
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/reviewer.md. Returns Review{verdict, notes, score,
    evaluatedAt} where verdict is the ReviewVerdict enum (APPROVE | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow DocumentWorkflow with steps:
    startStep -> writeStep -> guardrailStep -> [guardrail FAIL? writeStep
    again with structured feedback : reviewStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       writeStep with review attached : rejectStep)] -> END.
  writeStep calls forAutonomousAgent(WriterAgent.class, documentId).runSingleTask(
    WRITE or REVISE_DRAFT) then forTask(taskId).result(WRITE or
    REVISE_DRAFT). reviewStep calls forAutonomousAgent(ReviewerAgent.class,
    documentId).runSingleTask(REVIEW). approveStep emits DocumentApproved.
    rejectStep emits DocumentRejectedFinal with the highest-scoring draft's
    text as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on writeStep and reviewStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    draft.wordCount() <= ceiling. On FAIL, emits
    DraftGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_CEILING", then transitions back to writeStep with a
    structured feedback ReviewNotes("Draft exceeds the configured word
    ceiling; shorten and resubmit.").

- 1 EventSourcedEntity DocumentEntity holding state Document{documentId, topic,
  wordCeiling, maxAttempts, DocumentStatus status, List<DraftAttempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<String> approvedText,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. DocumentStatus enum: DRAFTING, REVIEWING, APPROVED,
  REJECTED_FINAL. Events: DocumentCreated, DraftProduced,
  DraftGuardrailVerdictRecorded, DraftReviewed, DocumentApproved,
  DocumentRejectedFinal, EvalRecorded. Commands: createDocument, recordDraft,
  recordGuardrail, recordReview, approve, rejectFinal, recordEval,
  getDocument. emptyState() returns Document.initial("", "", 500, 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueRequest(topic,
  wordCeiling, requestedBy) emitting TopicSubmitted{documentId, topic,
  wordCeiling, requestedBy, submittedAt}.

- 1 View DocumentsView with row type DocumentRow (mirrors Document; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes DocumentEntity events. ONE query
  getAllDocuments SELECT * AS documents FROM documents_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer DocRequestConsumer subscribed to RequestQueue events; on
  TopicSubmitted starts a DocumentWorkflow with the documentId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/doc-topics.jsonl and calls
    RequestQueue.enqueueRequest.
  * EvalSampler — every 30s, queries DocumentsView.getAllDocuments, finds documents
    with a reviewed attempt that has not yet been recorded as an
    EvalRecorded event, and calls DocumentEntity.recordEval(attemptNumber,
    verdict, score, ceilingExceeded). Idempotent per (documentId, attemptNumber).

- 2 HttpEndpoints:
  * DocumentEndpoint at /api with POST /documents, GET /documents,
    GET /documents/{id}, GET /documents/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /documents body is
    {topic, wordCeiling?, requestedBy?}; missing wordCeiling defaults to
    500, missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- DocTasks.java declaring three Task<R> constants: WRITE (resultConformsTo
  DocumentDraft), REVISE_DRAFT (DocumentDraft), REVIEW (Review).
- Domain records DocumentDraft, GuardrailVerdict, ReviewNotes, Review,
  DraftAttempt, Document; enums DocumentStatus, ReviewVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9885 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  writer-reviewer-doc-gen.refinement.max-attempts = 4 and
  writer-reviewer-doc-gen.refinement.default-word-ceiling = 500, overridable by
  env var.
- src/main/resources/sample-events/doc-topics.jsonl with 8 canned topic
  lines, each shaped {"topic":"...", "wordCeiling":500}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = document-generation-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/writer.md, prompts/reviewer.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: SK Document Generator",
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
  <title>Akka Sample: SK Document Generator</title>.

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
  named in Section 9: writer.json, reviewer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    writer.json — 6 DocumentDraft entries. Three are first-pass documents
      between 350 and 480 words on the topics in doc-topics.jsonl. Two are
      revised drafts that address prior review notes (shorter, tighter
      structure). One is an intentionally over-ceiling draft (>520 words)
      used to exercise the guardrail in J3.
    reviewer.json — 6 Review entries. Three return verdict=APPROVE with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a ReviewNotes payload of
      three bullets ("introduction lacks a clear thesis", "section 2 uses
      undefined acronyms", "conclusion does not reference the main claim").
- A MockModelProvider.seedFor(documentId, attemptNumber) helper makes the
  selection deterministic per (documentId, attemptNumber) so the same
  document in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. WriterAgent
  and ReviewerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a DocTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Document row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: DocTasks.java is mandatory; generating WriterAgent or
  ReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9885, declared in application.conf
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
