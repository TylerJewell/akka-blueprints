# SPEC — deep-research-supervisor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Deep Research Supervisor.
**One-line pitch:** Submit a research question; a supervisor decomposes it into subqueries, delegates each to parallel search/extract/summarise workers, then synthesises a cited deep-research report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in a per-subquery pipeline, gathers their results, and asks a supervisor AutonomousAgent to synthesise a unified deep-research report. The blueprint also demonstrates a **guardrail** (after-llm-response) that vets the final synthesis for hallucinated citations before it is surfaced, and an **eval-event** that samples the supervisor's synthesis decision for citation-grounding quality.

## 3. User-facing flows

The user opens the App UI tab and submits a research question via the form.

1. The system creates a `ResearchReport` record in `PLANNING` and starts a `DeepResearchWorkflow`.
2. The Supervisor decomposes the question into 2–4 `Subquery` items.
3. The workflow fans out: for each subquery, SearchWorker retrieves raw passages, ExtractionWorker distils key claims, and SummaryWorker produces a per-subquery summary. The per-subquery pipelines run in parallel across subqueries.
4. The Supervisor merges all per-subquery summaries into a `SynthesisedReport { executiveSummary, subquerySummaries, citationList, guardrailVerdict }`.
5. An after-llm-response guardrail vets the synthesised report for citation grounding; if it fails, the report moves to `BLOCKED`. Otherwise the report moves to `SYNTHESISED`.
6. If any subquery's SearchWorker times out after 60 seconds, the workflow short-circuits that subquery's pipeline and marks it `PARTIAL`. The Supervisor synthesises from whichever subqueries completed; the report enters `DEGRADED`.

A `QuestionSimulator` (TimedAction) drips a sample research question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchSupervisor` | `AutonomousAgent` | Decomposes the question into subqueries; later synthesises per-subquery summaries into the final report, running the citation guardrail. | `DeepResearchWorkflow` | returns typed result to workflow |
| `SearchWorker` | `AutonomousAgent` | Retrieves raw passages from the seeded corpus for a subquery. | `DeepResearchWorkflow` | — |
| `ExtractionWorker` | `AutonomousAgent` | Distils key claims and supporting quotes from raw passages. | `DeepResearchWorkflow` | — |
| `SummaryWorker` | `AutonomousAgent` | Produces a short cited summary for one subquery's extracted claims. | `DeepResearchWorkflow` | — |
| `DeepResearchWorkflow` | `Workflow` | Coordinates subquery fan-out, per-subquery pipelines, synthesis, and guardrail. | `ReportEndpoint`, `QuestionConsumer` | `ResearchReportEntity` |
| `ResearchReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (planning → in-progress → synthesised / degraded / blocked). | `DeepResearchWorkflow` | `ReportView` |
| `QuestionQueue` | `EventSourcedEntity` | Logs each submitted question for replay/audit. | `ReportEndpoint`, `QuestionSimulator` | `QuestionConsumer` |
| `ReportView` | `View` | List-of-reports read model. | `ResearchReportEntity` events | `ReportEndpoint` |
| `QuestionConsumer` | `Consumer` | Listens to `QuestionQueue` events and starts a workflow per submission. | `QuestionQueue` events | `DeepResearchWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample research question every 60 s. | scheduler | `QuestionQueue` |
| `CitationEvalSampler` | `TimedAction` | Samples one synthesised report every 5 minutes for citation-grounding eval; emits a `ReportEvalScored` event. | scheduler | `ResearchReportEntity` |
| `ReportEndpoint` | `HttpEndpoint` | `/api/reports/*` — submit, get, list, SSE. | — | `ReportView`, `QuestionQueue`, `ResearchReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QuestionRequest(String question, String requestedBy) {}

record Subquery(String subqueryId, String queryText) {}
record DecompositionPlan(List<Subquery> subqueries) {}

record RawPassage(String passageId, String source, String text) {}
record PassageBundle(List<RawPassage> passages, Instant retrievedAt) {}

record ExtractedClaim(String claim, String sourcePassageId, String quote) {}
record ClaimsBundle(List<ExtractedClaim> claims, Instant extractedAt) {}

record SubquerySummary(String subqueryId, String queryText, String summary,
                       List<ExtractedClaim> supportingClaims, Instant summarisedAt) {}

record Citation(String citationId, String source, String quote) {}

record SynthesisedReport(String executiveSummary, List<SubquerySummary> subquerySummaries,
                         List<Citation> citationList, String guardrailVerdict,
                         Instant synthesisedAt) {}

record ResearchReport(
    String reportId,
    String question,
    ReportStatus status,
    Optional<DecompositionPlan> plan,
    Optional<List<SubquerySummary>> subquerySummaries,
    Optional<SynthesisedReport> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

### Events (on `ResearchReportEntity`)

`ReportCreated`, `PlanReady`, `SubquerySummaryAttached`, `ReportSynthesised`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `QuestionQueue`)

`QuestionSubmitted { reportId, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reports` — body `{ question }` → `{ reportId }`. Starts a workflow.
- `GET /api/reports` — list all reports. Optional `?status=PLANNING|IN_PROGRESS|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/reports/{id}` — one report.
- `GET /api/reports/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Deep Research Supervisor"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a research question, live list of reports with status pills, expand-row to see per-subquery summaries + executive summary + citation list + eval score.

Browser title: `<title>Akka Sample: Deep Research Supervisor</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — after-llm-response guardrail** (`after-llm-response` on `ResearchSupervisor`): vets the synthesised report for hallucinated citations and unsupported claims before the report is returned. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `CitationEvalSampler` (TimedAction) picks one synthesised report every 5 minutes and emits a `ReportEvalScored` event with a 1–5 citation-grounding score and a short rationale.

## 9. Agent prompts

- `ResearchSupervisor` → `prompts/supervisor.md`. Decomposes the question into subqueries; later synthesises per-subquery summaries into the final report.
- `SearchWorker` → `prompts/search-worker.md`. Retrieves raw passages; returns `PassageBundle`.
- `ExtractionWorker` → `prompts/extraction-worker.md`. Distils key claims; returns `ClaimsBundle`.
- `SummaryWorker` → `prompts/summary-worker.md`. Produces a per-subquery cited summary; returns `SubquerySummary`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a research question; report progresses PLANNING → IN_PROGRESS → SYNTHESISED within 120 s; UI reflects each transition via SSE.
2. **J2** — Inject a SearchWorker timeout (set timeout to 1 s); affected subquery is marked partial; report enters DEGRADED with summaries from the remaining subqueries.
3. **J3** — Inject a guardrail failure (Supervisor returns report with fabricated citations); report enters BLOCKED.
4. **J4** — Wait after a successful synthesis; the report's row shows a citation-grounding eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named deep-research-supervisor demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-deep-research-supervisor.
Java package io.akka.samples.deepresearch. Akka 3.6.0. HTTP port 9390.

Components to wire (exactly):
- 4 AutonomousAgents:
  * ResearchSupervisor — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/supervisor.md. Returns
    DecompositionPlan{subqueries: List<Subquery{subqueryId, queryText}>} for DECOMPOSE
    and SynthesisedReport{executiveSummary, subquerySummaries, citationList,
    guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * SearchWorker — capability(TaskAcceptance.of(SEARCH).maxIterationsPerTask(2)).
    System prompt from prompts/search-worker.md. Returns
    PassageBundle{passages: List<RawPassage{passageId, source, text}>, retrievedAt}.
  * ExtractionWorker — capability(TaskAcceptance.of(EXTRACT).maxIterationsPerTask(2)).
    System prompt from prompts/extraction-worker.md. Returns
    ClaimsBundle{claims: List<ExtractedClaim{claim, sourcePassageId, quote}>, extractedAt}.
  * SummaryWorker — capability(TaskAcceptance.of(SUMMARISE).maxIterationsPerTask(2)).
    System prompt from prompts/summary-worker.md. Returns
    SubquerySummary{subqueryId, queryText, summary, supportingClaims, summarisedAt}.

- 1 Workflow DeepResearchWorkflow with steps:
  decomposeStep -> [parallel per subquery] searchStep/extractStep/summariseStep ->
  joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(ResearchSupervisor.class, DECOMPOSE).
  Per-subquery pipeline: searchStep calls SearchWorker (SEARCH), extractStep calls
  ExtractionWorker (EXTRACT) using the PassageBundle, summariseStep calls SummaryWorker
  (SUMMARISE) using the ClaimsBundle. The per-subquery pipelines run in parallel via
  CompletionStage allOf across all subqueries. Each searchStep gets a
  WorkflowSettings.builder().stepTimeout of 60s; extractStep and summariseStep each get
  45s. On any searchStep timeout, route that subquery to a partial path and continue with
  remaining subqueries; if any partial path is taken, end with ReportDegraded after
  synthesis. synthesiseStep calls forAutonomousAgent(ResearchSupervisor.class, SYNTHESISE)
  with merged SubquerySummaries; give synthesiseStep a 120s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the synthesised content;
  on failure, end with ReportBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity ResearchReportEntity holding state ResearchReport{reportId, question,
  ReportStatus, Optional<DecompositionPlan> plan,
  Optional<List<SubquerySummary>> subquerySummaries,
  Optional<SynthesisedReport> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: PLANNING, IN_PROGRESS, SYNTHESISED,
  DEGRADED, BLOCKED. Events: ReportCreated, PlanReady, SubquerySummaryAttached,
  ReportSynthesised, ReportDegraded, ReportBlocked, ReportEvalScored. Commands:
  createReport, recordPlan, attachSubquerySummary, synthesise, degrade, block, recordEval,
  getReport. emptyState() returns ResearchReport.initial("", null) with no commandContext()
  reference.

- 1 EventSourcedEntity QuestionQueue with command enqueueQuestion(question, requestedBy)
  emitting QuestionSubmitted{reportId, question, requestedBy, submittedAt}.

- 1 View ReportView with row type ResearchReportRow (mirrors ResearchReport minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ResearchReportEntity events. ONE query getAllReports SELECT * AS reports FROM report_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer QuestionConsumer subscribed to QuestionQueue events; on QuestionSubmitted
  starts a DeepResearchWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-questions.jsonl and calls
    QuestionQueue.enqueueQuestion.
  * CitationEvalSampler — every 5 minutes, queries ReportView.getAllReports, picks the oldest
    SYNTHESISED report without an evalScore, runs a 1–5 citation-grounding rubric judge over
    the synthesised content, then calls ResearchReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ReportEndpoint at /api with POST /reports, GET /reports, GET /reports/{id},
    GET /reports/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DeepResearchTasks.java declaring five Task<R> constants: DECOMPOSE (DecompositionPlan),
  SEARCH (PassageBundle), EXTRACT (ClaimsBundle), SUMMARISE (SubquerySummary),
  SYNTHESISE (SynthesisedReport).
- Domain records Subquery, DecompositionPlan, RawPassage, PassageBundle, ExtractedClaim,
  ClaimsBundle, SubquerySummary, Citation, SynthesisedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9390 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-questions.jsonl with 8 canned question lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 after-llm-response guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  deep-research-synthesis, decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/search-worker.md, prompts/extraction-worker.md,
  prompts/summary-worker.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Deep Research Supervisor", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Deep Research Supervisor</title>. No
  subtitle on the Overview tab.

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
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
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
  src/main/resources/mock-responses/<agent-name>.json (supervisor.json,
  search-worker.json, extraction-worker.json, summary-worker.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either DecompositionPlan or SynthesisedReport objects.
      4–6 DecompositionPlan entries (each with 2–3 Subquery items).
      4–6 SynthesisedReport entries (each with a 100–180 word executiveSummary,
      matching SubquerySummaries, 4–8 citations, guardrailVerdict = "ok").
    search-worker.json — 4–6 PassageBundle entries, each with 3–5 RawPassage items
      whose source values are believable (e.g., "EU AI Act Recital 47",
      "NIST AI RMF 1.0 §2.3", "unsourced — knowledge").
    extraction-worker.json — 4–6 ClaimsBundle entries, each with 3–5 ExtractedClaim
      items with a one-sentence claim, a sourcePassageId, and a short supporting quote.
    summary-worker.json — 4–6 SubquerySummary entries, each with a 40–80 word
      summary and 2–4 supportingClaims.
- A MockModelProvider.seedFor(reportId) helper makes the selection
  deterministic per report id so the same report produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s search,
  45s extract/summarise, 120s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion DeepResearchTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9390 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf for per-subquery fan-out, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
