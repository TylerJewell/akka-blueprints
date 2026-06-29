# SPEC — financial-research-pipeline

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Financial Research Multi-Agent.
**One-line pitch:** Submit a research query; a pipeline of specialised agents plans, retrieves, analyses, writes, and verifies a financial research report — with a verifier quality-check loop before the report is released.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow drives five specialised agents in sequence — `PlannerAgent`, `SearchAgent`, `AnalystAgent`, `WriterAgent`, and `VerifierAgent` — where the verifier plays the critic role: it inspects the draft and either approves it or sends it back to the writer with structured revision notes. The blueprint also demonstrates four governance mechanisms: an **eval-event** that records every verification verdict for continuous quality measurement; a **sector sanitiser** that strips disallowed financial identifiers from analysis sections before they reach the writer; an **output guardrail** that gates each report draft against a word-count ceiling before the verifier runs; and a **human-on-the-loop** control that routes every approved report through a compliance review queue before release.

## 3. User-facing flows

The user opens the App UI tab and submits a research query (a topic plus an optional sector tag and word-count ceiling).

1. The system creates a `Report` record in `PLANNING` and starts a `ResearchWorkflow`.
2. `PlannerAgent` decomposes the query into 2–5 ordered `SubTask` records, each with a scope description and a sector tag.
3. For each sub-task, `SearchAgent` retrieves a `SourceBundle` (ranked excerpts with provenance).
4. `AnalystAgent` synthesises all `SourceBundle`s into a list of `AnalysisSection` records, each with claims and evidence citations.
5. Before the analyst's output reaches the writer, the sector sanitiser scrubs disallowed financial identifiers (unverified ticker symbols, un-sourced price targets) from each `AnalysisSection`.
6. `WriterAgent` assembles the sanitised sections into a `ReportDraft` observing the word-count ceiling.
7. The output guardrail checks the draft's word count against the ceiling. Over-length drafts are returned to the Writer with a structured feedback note; they never reach the Verifier.
8. `VerifierAgent` inspects the draft against a rubric (factual consistency, source coverage, sector-appropriateness, word count) and returns either `APPROVE` with a one-line rationale, or `REVISE` with structured `VerificationNotes` (up to four bullets).
9. On `APPROVE`, the workflow transitions the report to `AWAITING_COMPLIANCE` and enqueues a `ComplianceReviewRequest` (the HOTL control). Once compliance signs off, the report moves to `APPROVED`.
10. On `REVISE`, the workflow records the draft, the sanitiser log, and the verification verdict on the entity, then calls `WriterAgent` again with the notes attached. The writer produces the next draft.
11. If the loop reaches `maxVerifyAttempts` (default 3) without an `APPROVE`, the workflow transitions to `REJECTED_FINAL`, preserving the best-scoring draft on the entity.

A `QuerySimulator` (TimedAction) drips a canned query every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Decomposes a research query into ordered sub-tasks with sector tags. | `ResearchWorkflow` | returns `ResearchPlan` to workflow |
| `SearchAgent` | `AutonomousAgent` | Executes one sub-task's retrieval step; returns `SourceBundle`. | `ResearchWorkflow` | returns `SourceBundle` to workflow |
| `AnalystAgent` | `AutonomousAgent` | Synthesises source bundles into analysis sections with claims and citations. | `ResearchWorkflow` | returns `List<AnalysisSection>` to workflow |
| `WriterAgent` | `AutonomousAgent` | Assembles analysis sections into a prose report draft; accepts prior verifier notes on revisions. | `ResearchWorkflow` | returns `ReportDraft` to workflow |
| `VerifierAgent` | `AutonomousAgent` | Quality-checks a report draft; returns `APPROVE` or `REVISE` with notes. | `ResearchWorkflow` | returns `Verification` to workflow |
| `ResearchWorkflow` | `Workflow` | Runs the full pipeline; loops write → guardrail → verify → revise; halts at the retry ceiling. | `ResearchEndpoint`, `QueryConsumer` | `ResearchEntity` |
| `ResearchEntity` | `EventSourcedEntity` | Holds the full report lifecycle: sub-tasks, source bundles, analysis sections, drafts, verifications, final outcome. | `ResearchWorkflow` | `ReportsView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay and audit. | `ResearchEndpoint`, `QuerySimulator` | `QueryConsumer` |
| `ReportsView` | `View` | List-of-reports read model. | `ResearchEntity` events | `ResearchEndpoint` |
| `QueryConsumer` | `Consumer` | Subscribes to `QueryQueue` events; starts a workflow per submission. | `QueryQueue` events | `ResearchWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 90 s from `sample-events/research-queries.jsonl`. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Every 45 s, scans `ReportsView`, records an `EvalRecorded` event for any verification cycle that completed since the last tick. | scheduler | `ResearchEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/reports/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ReportsView`, `QueryQueue`, `ResearchEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ResearchQuery(String topic, String sectorTag, int wordCeiling, String requestedBy) {}

record SubTask(int taskNumber, String scope, String sectorTag) {}

record ResearchPlan(List<SubTask> subTasks, String planSummary, Instant plannedAt) {}

record SourceExcerpt(String text, String provenance, double relevanceScore) {}

record SourceBundle(int taskNumber, List<SourceExcerpt> excerpts, Instant retrievedAt) {}

record AnalysisSection(int sectionNumber, String heading, List<String> claims, List<String> citations) {}

record SanitizerLog(boolean changed, List<String> removedItems, String reasonCode) {}

record VerificationNotes(List<String> bullets, String overallRationale) {}

record Verification(
    VerifierVerdict verdict,
    VerificationNotes notes,
    int score,
    Instant verifiedAt
) {}

record ReportDraft(
    int draftNumber,
    String text,
    int wordCount,
    Instant draftedAt
) {}

record Report(
    String reportId,
    String topic,
    String sectorTag,
    int wordCeiling,
    int maxVerifyAttempts,
    ReportStatus status,
    Optional<ResearchPlan> plan,
    List<SourceBundle> sourceBundles,
    List<AnalysisSection> analysisSections,
    SanitizerLog sanitizerLog,
    List<ReportDraft> drafts,
    List<Verification> verifications,
    Optional<Integer> approvedDraftNumber,
    Optional<String> approvedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus {
    PLANNING, RESEARCHING, ANALYSING, WRITING, VERIFYING,
    AWAITING_COMPLIANCE, APPROVED, REJECTED_FINAL
}

enum VerifierVerdict { APPROVE, REVISE }
```

### Events (on `ResearchEntity`)

`ReportCreated`, `PlanRecorded`, `SourceBundleAdded`, `AnalysisSectionsRecorded`, `SanitizerLogRecorded`, `DraftWritten`, `DraftGuardrailVerdictRecorded`, `DraftVerified`, `ReportApprovedPending`, `ReportApproved`, `ReportRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reports` — body `{ topic, sectorTag?, wordCeiling?, requestedBy? }` → `{ reportId }`. Starts a workflow.
- `GET /api/reports` — list all reports. Optional `?status=PLANNING|RESEARCHING|ANALYSING|WRITING|VERIFYING|AWAITING_COMPLIANCE|APPROVED|REJECTED_FINAL`.
- `GET /api/reports/{id}` — one report (including every draft and every verification).
- `GET /api/reports/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Financial Research Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, sanitizer = orange, guardrail = red, hotl = purple).
- **App UI** — form to submit a query, live list of reports with status pills, click-to-expand per-draft timeline showing each draft, the guardrail verdict, the verifier's verdict, and the verifier's notes; compliance queue indicator.

Browser title: `<title>Akka Sample: Financial Research Multi-Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every verification cycle's verdict is recorded as an `EvalRecorded` event with `{ draftNumber, verdict, score, wordCeilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits one event on terminal transitions. Enforcement: non-blocking.
- **S1 — sector sanitiser** (`sector`): after `AnalystAgent` returns, a deterministic pass removes disallowed financial identifiers (unverified tickers, un-sourced price targets) from every `AnalysisSection`. The result is logged as a `SanitizerLog` on the entity. The sanitised sections — never the raw output — are passed to `WriterAgent`. Enforcement: blocking.
- **G1 — output guardrail** (`before-agent-response` on `WriterAgent`): checks that `draft.wordCount <= report.wordCeiling`. Over-length drafts short-circuit back to the Writer with `reasonCode = OVER_WORD_CEILING`; they never reach the Verifier. Enforcement: blocking.
- **H1 — HOTL** (`live-compliance-review`): every report that the Verifier marks `APPROVE` is enqueued as a `ComplianceReviewRequest` before the report is released. The workflow transitions the report to `AWAITING_COMPLIANCE` and waits for a compliance-reviewer `POST /api/reports/{id}/approve-compliance` call. Enforcement: non-blocking (the workflow parks; it does not block the runtime).

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`.
- `SearchAgent` → `prompts/search.md`.
- `AnalystAgent` → `prompts/analyst.md`.
- `WriterAgent` → `prompts/writer.md`.
- `VerifierAgent` → `prompts/verifier.md`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — full pipeline convergence** — Submit a query; report progresses through all pipeline states and reaches `AWAITING_COMPLIANCE` then `APPROVED` within the retry ceiling.
2. **J2 — halt at verify ceiling** — Submit a query in forced-revise mode; report hits `REJECTED_FINAL` after `maxVerifyAttempts` with the best draft preserved.
3. **J3 — guardrail short-circuit** — Submit a query with a tight word ceiling; the writer's first draft exceeds the ceiling; the guardrail returns `OVER_WORD_CEILING`; the writer re-drafts; the cycle continues.
4. **J4 — sanitiser active** — Submit a query in a sector that triggers identifier removal; the `SanitizerLog` shows `changed = true` and lists the removed items; the writer's draft does not contain the removed identifiers.
5. **J5 — HOTL queue** — Every `APPROVE` verdict routes to `AWAITING_COMPLIANCE`; a compliance call moves the report to `APPROVED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named financial-research-pipeline demonstrating the
evaluator-optimizer × finance-analysis cell. Runs out of the box (no
external services).
Maven group io.akka.samples. Artifact id
evaluator-optimizer-finance-analysis-akka-financial-research-pipeline.
Java package io.akka.samples.financialresearchmultiagent. Akka 3.6.0.
HTTP port 9767.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PlannerAgent — definition() with
    capability(TaskAcceptance.of(PLAN_QUERY).maxIterationsPerTask(3)).
    System prompt from prompts/planner.md. Returns ResearchPlan{subTasks,
    planSummary, plannedAt}.
  * SearchAgent — definition() with
    capability(TaskAcceptance.of(RETRIEVE_SOURCES).maxIterationsPerTask(3)).
    System prompt from prompts/search.md. Returns SourceBundle{taskNumber,
    excerpts, retrievedAt}.
  * AnalystAgent — definition() with
    capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt from prompts/analyst.md. Returns List<AnalysisSection>
    wrapped as AnalysisSectionsResult{sections}.
  * WriterAgent — definition() with
    capability(TaskAcceptance.of(WRITE_DRAFT).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DRAFT).maxIterationsPerTask(3)).
    System prompt from prompts/writer.md. Returns ReportDraft{draftNumber,
    text, wordCount, draftedAt}. The REVISE_DRAFT task takes
    (sanitisedSections, priorDraft, VerificationNotes) as inputs.
  * VerifierAgent — definition() with
    capability(TaskAcceptance.of(VERIFY_DRAFT).maxIterationsPerTask(2)).
    System prompt from prompts/verifier.md. Returns Verification{verdict,
    notes, score, verifiedAt} where verdict is VerifierVerdict enum
    (APPROVE | REVISE).

- 1 Workflow ResearchWorkflow with steps:
    startStep -> planStep -> [for each subTask: searchStep] ->
    analyseStep -> sanitiseStep (pure-function) -> writeStep ->
    guardrailStep (pure-function) -> [guardrail FAIL? writeStep again with
    structured feedback : verifyStep] ->
    [verdict APPROVE? complianceStep : (draftCount < maxVerifyAttempts ?
       writeStep with notes attached : rejectStep)] -> END.
  planStep calls forAutonomousAgent(PlannerAgent.class, reportId)
    .runSingleTask(PLAN_QUERY).
  searchStep loops over subTasks; each iteration calls
    forAutonomousAgent(SearchAgent.class, reportId + "-" + taskNumber)
    .runSingleTask(RETRIEVE_SOURCES). Steps run sequentially; a parallel
    fan-out is an optimisation left for the deployer.
  analyseStep calls forAutonomousAgent(AnalystAgent.class, reportId)
    .runSingleTask(SYNTHESISE) with all SourceBundles as input.
  sanitiseStep is a pure-function step: strips items matching
    financial-research-pipeline.sanitiser.disallowed-patterns (regex list,
    configurable) from each AnalysisSection's claims. Emits
    SanitizerLogRecorded with changed=true/false and the list of removed
    items.
  writeStep calls forAutonomousAgent(WriterAgent.class, reportId)
    .runSingleTask(WRITE_DRAFT or REVISE_DRAFT).
  guardrailStep is a pure-function step: checks
    draft.wordCount() <= wordCeiling. On FAIL, emits
    DraftGuardrailVerdictRecorded with reasonCode = "OVER_WORD_CEILING",
    then transitions back to writeStep with a structured feedback
    VerificationNotes("Draft exceeds the configured word ceiling; condense
    and resubmit.").
  verifyStep calls forAutonomousAgent(VerifierAgent.class, reportId)
    .runSingleTask(VERIFY_DRAFT).
  complianceStep emits ReportApprovedPending and parks; resumes on
    POST /api/reports/{id}/approve-compliance, which emits ReportApproved.
  rejectStep emits ReportRejectedFinal with the highest-scoring draft as
    best-of and a structured rejectionReason.
  Override settings() with stepTimeout(90s) on planStep, each searchStep,
    analyseStep, writeStep, verifyStep; stepTimeout(5s) on sanitiseStep
    and guardrailStep; defaultStepRecovery(maxRetries(2)
    .failoverTo(rejectStep)).

- 1 EventSourcedEntity ResearchEntity holding state Report{reportId, topic,
  sectorTag, wordCeiling, maxVerifyAttempts, ReportStatus status,
  Optional<ResearchPlan> plan, List<SourceBundle> sourceBundles,
  List<AnalysisSection> analysisSections, SanitizerLog sanitizerLog,
  List<ReportDraft> drafts, List<Verification> verifications,
  Optional<Integer> approvedDraftNumber, Optional<String> approvedText,
  Optional<String> rejectionReason, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: PLANNING, RESEARCHING,
  ANALYSING, WRITING, VERIFYING, AWAITING_COMPLIANCE, APPROVED,
  REJECTED_FINAL. Events: ReportCreated, PlanRecorded, SourceBundleAdded,
  AnalysisSectionsRecorded, SanitizerLogRecorded, DraftWritten,
  DraftGuardrailVerdictRecorded, DraftVerified, ReportApprovedPending,
  ReportApproved, ReportRejectedFinal, EvalRecorded. emptyState() returns
  Report.initial("", "", "general", 800, 3). Event-applier wraps lifecycle
  fields with Optional.of(...).

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(topic,
  sectorTag, wordCeiling, requestedBy) emitting
  QuerySubmitted{reportId, topic, sectorTag, wordCeiling, requestedBy,
  submittedAt}.

- 1 View ReportsView with row type ReportRow (mirrors Report; drafts and
  verifications lists bounded at maxVerifyAttempts). ONE query
  getAllReports SELECT * AS reports FROM reports_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer QueryConsumer subscribed to QueryQueue events; on
  QuerySubmitted starts a ResearchWorkflow with the reportId as the
  workflow id.

- 2 TimedActions:
  * QuerySimulator — every 90s, reads next line from
    src/main/resources/sample-events/research-queries.jsonl and calls
    QueryQueue.enqueueQuery.
  * EvalSampler — every 45s, queries ReportsView.getAllReports, finds
    reports with a verified draft that has not yet been recorded as an
    EvalRecorded event, and calls
    ResearchEntity.recordEval(draftNumber, verdict, score,
    wordCeilingExceeded). Idempotent per (reportId, draftNumber).

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /reports, GET /reports,
    GET /reports/{id}, GET /reports/sse,
    POST /reports/{id}/approve-compliance (compliance reviewer action),
    and three /api/metadata/* endpoints serving YAML/MD from
    src/main/resources/metadata/. POST /reports body is
    {topic, sectorTag?, wordCeiling?, requestedBy?}; missing sectorTag
    defaults to "general"; missing wordCeiling defaults to 800; missing
    requestedBy defaults to "anonymous". wordCeiling must be in
    [200, 5000]; otherwise 400.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ResearchTasks.java declaring five Task<R> constants: PLAN_QUERY
  (resultConformsTo ResearchPlan), RETRIEVE_SOURCES (SourceBundle),
  SYNTHESISE (AnalysisSectionsResult), WRITE_DRAFT (ReportDraft),
  REVISE_DRAFT (ReportDraft), VERIFY_DRAFT (Verification).
- Domain records ResearchQuery, SubTask, ResearchPlan, SourceExcerpt,
  SourceBundle, AnalysisSection, SanitizerLog, VerificationNotes,
  Verification, ReportDraft, Report; enums ReportStatus, VerifierVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9767 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Per-workflow config
  financial-research-pipeline.research.max-verify-attempts = 3 and
  financial-research-pipeline.research.default-word-ceiling = 800.
  financial-research-pipeline.sanitiser.disallowed-patterns = []
  (deployer adds regex strings; default is no-op).
- src/main/resources/sample-events/research-queries.jsonl with 8 canned
  query lines shaped
  {"topic":"...", "sectorTag":"equity|credit|macro|commodities",
   "wordCeiling":800}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of root-level files for the metadata endpoint).
- eval-matrix.yaml at the project root with 4 controls (E1 eval-event
  on-decision-eval, S1 sanitizer sector, G1 guardrail
  before-agent-response, H1 hotl live-compliance-review) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, decisions,
  data, capabilities for the finance-analysis domain; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/search.md, prompts/analyst.md,
  prompts/writer.md, prompts/verifier.md loaded at agent startup as
  system prompts.
- README.md at the project root: title
  "Akka Sample: Financial Research Multi-Agent", one-line pitch,
  prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single
  self-contained HTML file (no ui/ folder, no npm build). Inline CSS + JS.
  Runtime CDN imports for markdown and YAML libs are acceptable. Five tabs
  matching the formal exemplar: Overview; Architecture (4 mermaid
  diagrams + component table); Risk Survey (7 sub-tabs); Eval Matrix
  (5-column table with click-to-expand rows); App UI (query form + live
  report list with status pills + click-to-expand per-draft timeline +
  compliance queue indicator). Browser title exactly:
  <title>Akka Sample: Financial Research Multi-Agent</title>.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. All five
  agents extend akka.javasdk.agent.autonomous.AutonomousAgent and ship
  with a ResearchTasks companion declaring the six Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Report row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ResearchTasks.java is mandatory; generating any agent without
  it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: the run command is /akka:build.
- Lesson 10: HTTP port is 9767, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; no competitor brand name
  in any user-facing surface.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables.
- Lesson 25: NEVER write the key value to disk; application.conf records
  only ${?VAR_NAME} substitution.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. Exactly five <section class="tab-panel">
  elements in the DOM.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
