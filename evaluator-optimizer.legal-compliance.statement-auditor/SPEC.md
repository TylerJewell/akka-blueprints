# SPEC — statement-auditor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Statement Auditor.
**One-line pitch:** Submit a financial statement section; an auditor agent identifies consistency gaps and traces each finding to a working paper; a reviewer agent scores each finding against a materiality rubric and validates citations; the two iterate until the reviewer approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between an analysis agent (`AuditorAgent`) and a reviewer agent (`ReviewerAgent`), feeding each review back into the next analysis pass until convergence or a halt. The blueprint also demonstrates two governance mechanisms — a **guardrail** that blocks finding reports whose working-paper citations are unresolvable before the reviewer runs (audit findings are evidence-of-record; they must reference traceable working papers), and an **eval-event** that records every cycle's verdict with materiality score and citation-traceability status so operators have a continuous quality stream over the audit pipeline.

## 3. User-facing flows

The user opens the App UI tab and submits a statement section (a section identifier, raw text, and a reference to the available working-paper registry).

1. The system creates an `Engagement` record in `ANALYZING` and starts an `AuditWorkflow`.
2. The Auditor analyzes attempt #1: it reads the section text, identifies internal inconsistencies and unsupported figures, and returns a `FindingReport` (list of `Finding` records, each with a citation to a working-paper reference).
3. The citation guardrail verifies that every cited working-paper reference exists in the working-paper registry. Reports with any unresolvable citation are short-circuited back to the Auditor with a deterministic feedback note; they never reach the Reviewer.
4. The Reviewer scores each finding against a four-dimension rubric (materiality, source traceability, factual accuracy, clarity) and returns either `APPROVED` with a one-line sign-off rationale, or `REVISE` with a typed `ReviewNotes` payload (up to four bullets).
5. On `APPROVED`, the workflow transitions the engagement to `SIGNED_OFF` with the winning finding report and the reviewer's rationale.
6. On `REVISE`, the workflow records the iteration, the guardrail verdict, the review, and the reviewer's verdict on the entity, then calls the Auditor again with the review attached. The Auditor produces attempt #2.
7. If the loop reaches `maxAttempts` (default 3) without an `APPROVED`, the halt mechanism activates: the workflow ends with `UNRESOLVED`, the best-scored finding report is preserved on the entity along with every review for audit, and an `EvalRecorded` event captures the outcome.

A `StatementSimulator` (TimedAction) drips a canned statement section every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AuditorAgent` | `AutonomousAgent` | Analyzes a financial statement section; identifies consistency gaps and traces each finding to a working-paper citation. Accepts prior review feedback on revision calls. | `AuditWorkflow` | returns `FindingReport` to workflow |
| `ReviewerAgent` | `AutonomousAgent` | Scores each finding against the materiality rubric; returns `APPROVED` or `REVISE` with notes. | `AuditWorkflow` | returns `Review` to workflow |
| `AuditWorkflow` | `Workflow` | Runs the analyze → citation-guardrail → review → revise loop; halts at the ceiling. | `AuditEndpoint`, `StatementConsumer` | `EngagementEntity` |
| `EngagementEntity` | `EventSourcedEntity` | Holds the engagement lifecycle, every finding iteration, every review, and the final outcome. | `AuditWorkflow` | `EngagementsView` |
| `StatementQueue` | `EventSourcedEntity` | Logs each submitted statement section for replay and audit. | `AuditEndpoint`, `StatementSimulator` | `StatementConsumer` |
| `EngagementsView` | `View` | List-of-engagements read model. | `EngagementEntity` events | `AuditEndpoint` |
| `StatementConsumer` | `Consumer` | Subscribes to `StatementQueue` events; starts a workflow per submission. | `StatementQueue` events | `AuditWorkflow` |
| `StatementSimulator` | `TimedAction` | Drips a sample statement section every 60 s from `sample-events/statement-sections.jsonl`. | scheduler | `StatementQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `EngagementsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `EngagementEntity` |
| `AuditEndpoint` | `HttpEndpoint` | `/api/engagements/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `EngagementsView`, `StatementQueue`, `EngagementEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record StatementSection(String sectionId, String sectionTitle, String rawText, List<String> workingPaperRefs) {}

record Finding(String findingId, String description, FindingCategory category, MaterialityLevel materiality, String citedWorkingPaper) {}

record FindingReport(List<Finding> findings, String auditorRationale, Instant analyzedAt) {}

record CitationGuardrailVerdict(boolean passed, String reasonCode, List<String> unresolvedRefs) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record Review(ReviewVerdict verdict, ReviewNotes notes, int materialityScore, Instant reviewedAt) {}

record AuditIteration(
    int iterationNumber,
    FindingReport report,
    CitationGuardrailVerdict guardrail,
    Optional<Review> review
) {}

record Engagement(
    String engagementId,
    String sectionId,
    String sectionTitle,
    int maxAttempts,
    EngagementStatus status,
    List<AuditIteration> iterations,
    Optional<Integer> approvedIterationNumber,
    Optional<FindingReport> approvedReport,
    Optional<String> unresolvedReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EngagementStatus { ANALYZING, REVIEWING, SIGNED_OFF, UNRESOLVED }

enum ReviewVerdict { APPROVED, REVISE }

enum FindingCategory { CONSISTENCY_GAP, UNSUPPORTED_FIGURE, DISCLOSURE_OMISSION, CLASSIFICATION_ERROR }

enum MaterialityLevel { LOW, MEDIUM, HIGH, CRITICAL }
```

### Events (on `EngagementEntity`)

`EngagementCreated`, `IterationAnalyzed`, `IterationCitationVerdictRecorded`, `IterationReviewed`, `EngagementSignedOff`, `EngagementUnresolved`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/engagements` — body `{ sectionId, sectionTitle, rawText, workingPaperRefs?, submittedBy? }` → `{ engagementId }`. Starts a workflow.
- `GET /api/engagements` — list all engagements. Optional `?status=ANALYZING|REVIEWING|SIGNED_OFF|UNRESOLVED`.
- `GET /api/engagements/{id}` — one engagement (including every iteration and every review).
- `GET /api/engagements/sse` — server-sent events stream of every engagement change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Statement Auditor"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (guardrail = red, eval-event = blue).
- **App UI** — form to submit a statement section, live list of engagements with status pills, click-to-expand per-iteration timeline showing each finding report, the citation guardrail verdict, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: Statement Auditor</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — citation guardrail** (`before-agent-response` on `AuditorAgent`): a deterministic check that every working-paper citation in the `FindingReport` resolves against the submitted `workingPaperRefs` list. Reports with any unresolvable citation are blocked with `reasonCode = UNRESOLVED_CITATION` and returned to the Auditor with a structured feedback note listing the unresolvable references. The reviewer never sees unsupported findings. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's review verdict is recorded as an `EvalRecorded` event with `{ iterationNumber, verdict, materialityScore, citationsPassed }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits one event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-iteration timeline and in `/api/engagements/{id}`.

## 9. Agent prompts

- `AuditorAgent` → `prompts/auditor.md`. Analyzes a financial statement section; on a revision call, takes prior `ReviewNotes` and re-examines the section with the reviewer's corrections incorporated.
- `ReviewerAgent` → `prompts/reviewer.md`. Scores each finding against the materiality rubric; returns `APPROVED` with a sign-off rationale, or `REVISE` with up to four bullets naming specific corrections.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a statement section; engagement progresses `ANALYZING` → `REVIEWING` → `SIGNED_OFF` within the retry ceiling; the App UI shows every iteration's finding report and review.
2. **J2 — halt at ceiling** — Submit a section whose rubric is impossible (test mode forces the Reviewer to `REVISE` every iteration); engagement progresses through every iteration and lands in `UNRESOLVED` with the best-scored finding report preserved and a structured unresolved reason.
3. **J3 — citation guardrail block** — Submit a section where the Auditor's first report cites a non-existent working paper; the guardrail short-circuits with `reasonCode = UNRESOLVED_CITATION`; the Auditor re-drafts the report with corrected citations.
4. **J4 — eval-event timeline** — The expanded view of any engagement shows one `EvalRecorded` event per reviewed iteration and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named statement-auditor demonstrating the evaluator-optimizer ×
legal-compliance cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-legal-compliance-statement-auditor.
Java package io.akka.samples.statementauditor. Akka 3.6.0. HTTP port 9320.

Components to wire (exactly):
- 2 AutonomousAgents:
  * AuditorAgent — definition() with
    capability(TaskAcceptance.of(ANALYZE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ANALYSIS).maxIterationsPerTask(3)).
    System prompt loaded from prompts/auditor.md. Returns FindingReport{findings,
    auditorRationale, analyzedAt} for both ANALYZE and REVISE_ANALYSIS. The
    REVISE_ANALYSIS task takes (originalSection, priorReport, ReviewNotes) as
    inputs.
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/reviewer.md. Returns Review{verdict, notes, materialityScore,
    reviewedAt} where verdict is the ReviewVerdict enum (APPROVED | REVISE)
    and materialityScore is a 1–5 integer rubric.

- 1 Workflow AuditWorkflow with steps:
    startStep -> analyzeStep -> citationGuardrailStep -> [guardrail FAIL?
    analyzeStep again with structured feedback : reviewStep] ->
    [verdict APPROVED? signOffStep : (iterationCount < maxAttempts ?
       analyzeStep with review attached : unresolvedStep)] -> END.
  analyzeStep calls forAutonomousAgent(AuditorAgent.class, engagementId)
    .runSingleTask(ANALYZE or REVISE_ANALYSIS) then forTask(taskId)
    .result(ANALYZE or REVISE_ANALYSIS). reviewStep calls
    forAutonomousAgent(ReviewerAgent.class, engagementId).runSingleTask(REVIEW).
    signOffStep emits EngagementSignedOff. unresolvedStep emits
    EngagementUnresolved with the highest-scoring iteration's report as best-of
    and a structured unresolvedReason. Override settings() with stepTimeout(60s)
    on analyzeStep and reviewStep, and defaultStepRecovery(maxRetries(2)
    .failoverTo(unresolvedStep)).
  citationGuardrailStep is a pure-function step (no LLM call): checks that every
    Finding.citedWorkingPaper in the report exists in the engagement's
    workingPaperRefs list. On FAIL, emits IterationCitationVerdictRecorded with
    verdict.passed = false and reasonCode = "UNRESOLVED_CITATION" listing the
    unresolvable refs, then transitions back to analyzeStep with a structured
    feedback ReviewNotes("Citations reference working papers not in the
    engagement registry; correct and resubmit: [refs]").

- 1 EventSourcedEntity EngagementEntity holding state Engagement{engagementId,
  sectionId, sectionTitle, maxAttempts, EngagementStatus status,
  List<AuditIteration> iterations, Optional<Integer> approvedIterationNumber,
  Optional<FindingReport> approvedReport, Optional<String> unresolvedReason,
  Instant createdAt, Optional<Instant> finishedAt}. EngagementStatus enum:
  ANALYZING, REVIEWING, SIGNED_OFF, UNRESOLVED. Events: EngagementCreated,
  IterationAnalyzed, IterationCitationVerdictRecorded, IterationReviewed,
  EngagementSignedOff, EngagementUnresolved, EvalRecorded. Commands:
  createEngagement, recordIteration, recordCitationVerdict, recordReview,
  signOff, markUnresolved, recordEval, getEngagement. emptyState() returns
  Engagement.initial("", "", 3) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity StatementQueue with command enqueueSection(sectionId,
  sectionTitle, rawText, workingPaperRefs, submittedBy) emitting
  SectionSubmitted{engagementId, sectionId, sectionTitle, rawText,
  workingPaperRefs, submittedBy, submittedAt}.

- 1 View EngagementsView with row type EngagementRow (mirrors Engagement; the
  iterations list is preserved as-is — the list is bounded at maxAttempts so
  size stays reasonable). Table updater consumes EngagementEntity events. ONE
  query getAllEngagements SELECT * AS engagements FROM engagements_view.
  No WHERE status filter — caller filters client-side because Akka cannot
  auto-index enum columns (Lesson 2).

- 1 Consumer StatementConsumer subscribed to StatementQueue events; on
  SectionSubmitted starts an AuditWorkflow with the engagementId as the
  workflow id.

- 2 TimedActions:
  * StatementSimulator — every 60s, reads next line from
    src/main/resources/sample-events/statement-sections.jsonl and calls
    StatementQueue.enqueueSection.
  * EvalSampler — every 30s, queries EngagementsView.getAllEngagements, finds
    engagements with a reviewed iteration that has not yet been recorded as an
    EvalRecorded event, and calls EngagementEntity.recordEval(iterationNumber,
    verdict, materialityScore, citationsPassed). Idempotent per
    (engagementId, iterationNumber).

- 2 HttpEndpoints:
  * AuditEndpoint at /api with POST /engagements, GET /engagements,
    GET /engagements/{id}, GET /engagements/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /engagements body is
    {sectionId, sectionTitle, rawText, workingPaperRefs?, submittedBy?};
    missing workingPaperRefs defaults to [], missing submittedBy defaults to
    "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AuditTasks.java declaring three Task<R> constants: ANALYZE (resultConformsTo
  FindingReport), REVISE_ANALYSIS (FindingReport), REVIEW (Review).
- Domain records FindingReport, CitationGuardrailVerdict, ReviewNotes, Review,
  AuditIteration, Engagement; enums EngagementStatus, ReviewVerdict,
  FindingCategory, MaterialityLevel.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9320 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  statement-auditor.audit.max-attempts = 3 overridable by env var.
- src/main/resources/sample-events/statement-sections.jsonl with 8 canned
  statement section lines, each shaped {"sectionId":"...","sectionTitle":"...",
  "rawText":"...","workingPaperRefs":["WP-001","WP-002"]}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 citation guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = financial-statement-audit,
  decisions.authority_level = advisory-findings-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/auditor.md, prompts/reviewer.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Statement Auditor",
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
  per-iteration timeline). Browser title exactly:
  <title>Akka Sample: Statement Auditor</title>.

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
  named in Section 9: auditor.json, reviewer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    auditor.json — 6 FindingReport entries. Three are "first-pass" reports
      with two or three findings each, citing working papers WP-001 through
      WP-004 on typical income-statement sections. Two are "revision" reports
      that address prior reviewer bullets (fewer findings, tighter citations).
      One is an intentionally bad-citation report (references WP-999, which
      is not in the registry) used to exercise the citation guardrail in J3.
    reviewer.json — 6 Review entries. Three return verdict=APPROVED with
      materialityScore=4 or 5 and a one-sentence sign-off rationale. Three
      return verdict=REVISE with materialityScore=2 or 3 and a ReviewNotes
      payload of three bullets ("finding F-002 lacks materiality quantification",
      "cited working paper WP-003 does not support the stated figure",
      "classification error in note 4 described as high but evidence is medium").
- A MockModelProvider.seedFor(engagementId, iterationNumber) helper makes the
  selection deterministic per (engagementId, iterationNumber) so the same
  engagement in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. AuditorAgent
  and ReviewerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an AuditTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Engagement row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: AuditTasks.java is mandatory; generating AuditorAgent or
  ReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9320, declared in application.conf
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
