# SPEC — research-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Research Agent.
**One-line pitch:** Submit a research query; a Research Planner designs a search strategy on a search ledger, dispatches each step to a Web Searcher or Document Searcher, evaluates citation accuracy on a findings ledger, and synthesizes a cited report.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with two specialist search executors. The Research Planner owns two ledgers — a **search ledger** (query context, known facts, missing facts, search plan, current dispatch) and a **findings ledger** (each step's retrieved content, citation status, confidence signal, verdict). On each loop iteration the Planner reads both ledgers, picks the next search specialist, and either continues, replans, or completes. On two consecutive replans without new findings, or three consecutive failed steps on the same query, the Planner emits a terminal failure.

The blueprint also demonstrates one governance control wired into that loop:

- an **eval-event on-decision-eval** that runs after each finding is recorded, evaluating whether the retrieved content contains at least one verifiable citation (author, date, or source URL). Results that pass receive verdict `CITED`; results that contain no traceable source receive verdict `UNCITED` and the Planner is asked to retry with a more specific query. This is a non-blocking control — the finding is recorded regardless; the verdict influences the next dispatch decision.

## 3. User-facing flows

The user opens the App UI tab and submits a research query via the form.

1. The system creates a `ResearchJob` record in `PLANNING` and starts a `ResearchWorkflow`.
2. The Planner drafts a `SearchLedger { context, knownFacts, missingFacts, plan, currentDispatch }` and emits `JobPlanned`.
3. The workflow enters the search loop. Each iteration:
   - Planner reads both ledgers and proposes a `SearchDecision { searcher, query, rationale }`.
   - The chosen specialist runs the query and returns a typed `SearchResult`.
   - The **citation evaluator** runs on the result; it appends a `FindingEntry { searcher, query, attempt, content, citationVerdict, confidence, recordedAt }` to the findings ledger.
   - The Planner decides on each tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`.
4. On `COMPLETE`, the Planner produces a `ResearchReport { title, summary, findings, citations, producedAt }` and emits `JobCompleted`. The job moves to `COMPLETED`.
5. The operator can press **Halt new dispatches** at any time. The workflow finishes the in-flight step, then ends with `JobHaltedOperator`. The job moves to `HALTED`.

A `QuerySimulator` (TimedAction) drips a sample research query every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchPlannerAgent` | `AutonomousAgent` | Plans the search strategy; proposes each dispatch; synthesizes the final report. Maintains search ledger; reads findings ledger. | `ResearchWorkflow` | returns typed result to workflow |
| `WebSearchAgent` | `AutonomousAgent` | Executes web-search queries from seeded fixtures (`sample-data/web-fixtures.jsonl`). Returns `SearchResult`. | `ResearchWorkflow` | — |
| `DocumentSearchAgent` | `AutonomousAgent` | Retrieves excerpts from document index fixtures (`sample-data/docs/*`). Returns `SearchResult`. | `ResearchWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Drives the plan → checkHalt → propose → dispatch → citationEval → record → decide loop, plus replan and halt branches. | `ResearchEndpoint`, `QueryRequestConsumer` | `ResearchJobEntity` |
| `ResearchJobEntity` | `EventSourcedEntity` | Holds the job's lifecycle, search ledger, findings ledger, citation evaluation record, and final report. | `ResearchWorkflow` | `ResearchJobView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `ResearchEndpoint` (operator action) | `ResearchWorkflow` (polls) |
| `QueryQueue` | `EventSourcedEntity` | Audit log of submitted queries. | `ResearchEndpoint`, `QuerySimulator` | `QueryRequestConsumer` |
| `ResearchJobView` | `View` | List-of-jobs read model for the UI. | `ResearchJobEntity` events | `ResearchEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Subscribes to `QueryQueue` events; starts a `ResearchWorkflow` per submission. | `QueryQueue` events | `ResearchWorkflow` |
| `QuerySimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/query-prompts.jsonl` and enqueues it. | scheduler | `QueryQueue` |
| `StaleJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `SEARCHING` past 5 minutes as `STUCK`. The workflow polls this and ends with `JobFailedTimeout`. | scheduler | `ResearchJobEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE, operator halt. | — | `ResearchJobView`, `QueryQueue`, `ResearchJobEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record SearchLedger(
    String context,
    List<String> knownFacts,
    List<String> missingFacts,
    List<String> plan,
    Optional<SearchDecision> currentDispatch
) {}

record SearchDecision(
    SearcherKind searcher,
    String query,
    String rationale
) {}

record SearchResult(
    SearcherKind searcher,
    String query,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record FindingEntry(
    int attempt,
    SearcherKind searcher,
    String query,
    String content,
    CitationVerdict citationVerdict,
    double confidence,
    Optional<String> noSourceReason,
    Instant recordedAt
) {}

record FindingsLedger(List<FindingEntry> entries) {}

record Citation(String label, String source, String searcher) {}

record ResearchReport(
    String title,
    String summary,
    List<String> findings,
    List<Citation> citations,
    Instant producedAt
) {}

record ResearchJob(
    String jobId,
    String query,
    JobStatus status,
    Optional<SearchLedger> searchLedger,
    Optional<FindingsLedger> findingsLedger,
    Optional<ResearchReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SearcherKind { WEB, DOCUMENT }
enum CitationVerdict { CITED, UNCITED }
enum JobStatus { PLANNING, SEARCHING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`ResearchJobEntity`)

`JobCreated`, `JobPlanned`, `StepDispatched`, `StepRetried`, `FindingRecorded`, `LedgerRevised`, `JobCompleted`, `JobFailed`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`QueryQueue`)

`QuerySubmitted { jobId, query, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ query, requestedBy? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full ledgers + report).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Research Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a query, operator halt/resume control, live list of jobs with status pills, expand-row to see the search ledger, the findings ledger entries (with citation verdict chips), and the final report.

Browser title: `<title>Akka Sample: Research Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — citation accuracy eval** (`eval-event`, flavor `on-decision-eval`): after each `FindingEntry` is appended to the findings ledger, the `CitationEvaluator` inspects the result content for traceable source signals — a URL pattern, an author–year reference, a document path, or an ISBN. Content that carries at least one such signal receives `CitationVerdict.CITED` with a confidence in `[0.0, 1.0]`. Content with no traceable source receives `CitationVerdict.UNCITED` and a `noSourceReason`. The verdict and confidence are stored on the `FindingEntry` itself; the Planner reads them on the next loop tick and may replan to retrieve a better-sourced result. Non-blocking: the finding is always recorded, verdict or not.

- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows every in-flight job, its current dispatch, and its last finding. Two buttons — Halt new dispatches, Resume — drive `SystemControlEntity`. The workflow polls `SystemControlEntity` before each dispatch and exits with `JobHaltedOperator` if the flag is set.

## 9. Agent prompts

- `ResearchPlannerAgent` → `prompts/research-planner.md`. Designs strategy, proposes each step, synthesizes final report.
- `WebSearchAgent` → `prompts/web-search.md`. Returns search-style findings from fixtures.
- `DocumentSearchAgent` → `prompts/document-search.md`. Returns document excerpts from indexed fixtures.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Summarise the current regulatory landscape for AI transparency in the EU." Job progresses `PLANNING → SEARCHING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a search ledger with a non-empty plan, a findings ledger with 3–6 entries spanning both WEB and DOCUMENT searchers, and a non-empty `ResearchReport` with at least 2 citations.
2. **J2** — A web-search step returns content with no URL, date, or author reference. The citation evaluator marks it `UNCITED`. The Planner replans to run a document search for the same fact. The final report only cites findings where `citationVerdict = CITED`.
3. **J3** — Submit any query and click **Halt new dispatches** while status is `SEARCHING`. The in-flight step finishes; the job ends in `HALTED`; the operator pane shows the halt reason live.
4. **J4** — A document fixture entry carries a clear source URL. The `FindingEntry` in the expanded UI shows `citationVerdict = CITED` with confidence ≥ 0.8 and the citation label appears in the final report's citations list.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named research-agent demonstrating the
planner-executor × research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-research-intel-research-agent.
Java package io.akka.samples.researchagent. Akka 3.6.0. HTTP port 9599.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ResearchPlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_SEARCH).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(DECIDE_NEXT).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2))
    System prompt from prompts/research-planner.md. PLAN_SEARCH returns SearchLedger.
    DECIDE_NEXT returns a NextStep sealed interface (Continue(SearchDecision) |
    Replan(SearchLedger) | Complete(ResearchReport stub) | Fail(reason)).
    COMPOSE_REPORT returns ResearchReport.
  * WebSearchAgent — capability(TaskAcceptance.of(WEB_SEARCH).maxIterationsPerTask(2)).
    Prompt from prompts/web-search.md. Returns SearchResult.
  * DocumentSearchAgent — capability(TaskAcceptance.of(DOC_SEARCH).maxIterationsPerTask(2)).
    Prompt from prompts/document-search.md. Returns SearchResult.

- 1 Workflow ResearchWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> dispatchStep ->
  citationEvalStep -> recordStep -> decideStep
  -> [back to checkHaltStep or to composeReportStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45),
    dispatchStep ofSeconds(120) (covers any searcher call),
    citationEvalStep ofSeconds(15), decideStep ofSeconds(45),
    composeReportStep ofSeconds(60).
  defaultStepRecovery(maxRetries(2).failoverTo(ResearchWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits JobHaltedOperator on ResearchJobEntity).
  dispatchStep uses switch on SearchDecision.searcher to call the matching
  agent via forAutonomousAgent(...).runSingleTask(...) then forTask(jobId).result(...).
  citationEvalStep runs CitationEvaluator.evaluate(SearchResult) synchronously,
  producing a CitationVerdict and confidence. The step annotates the result
  before recordStep.
  recordStep calls ResearchJobEntity.recordFinding(entry).
  decideStep calls forAutonomousAgent(ResearchPlannerAgent.class, DECIDE_NEXT);
  on Continue or Replan loops; on Complete transitions to composeReportStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity ResearchJobEntity holding ResearchJob state. emptyState()
  returns ResearchJob.initial("", null). Commands: createJob, recordPlan,
  recordDispatch, recordFinding, reviseLedger, completeJob, failJob, haltOperator,
  timeoutFail, getJob. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant> haltedAt}.
  Commands: requestHalt(reason), clearHalt, get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(jobId, query,
  requestedBy) emitting QuerySubmitted.

- 1 View ResearchJobView with row type JobRow (mirror of ResearchJob minus heavy
  ledger payloads — truncate to last 3 finding entries plus counts; UI fetches
  the full job by id on click). ONE query getAllJobs SELECT * AS jobs FROM
  research_job_view. No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer QueryRequestConsumer subscribed to QueryQueue events; on
  QuerySubmitted starts a ResearchWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 90s, reads next line from
    src/main/resources/sample-events/query-prompts.jsonl and calls
    QueryQueue.enqueueQuery.
  * StaleJobMonitor — every 30s, queries ResearchJobView.getAllJobs, filters
    SEARCHING jobs whose createdAt is older than 5 minutes, calls
    ResearchJobEntity.timeoutFail; ResearchWorkflow polls ResearchJobEntity.getJob
    in its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_SEARCH
  (resultConformsTo SearchLedger), DECIDE_NEXT (NextStep), COMPOSE_REPORT (ResearchReport).
- SearcherTasks.java declaring two Task<R> constants: WEB_SEARCH, DOC_SEARCH
  (both resultConformsTo SearchResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue, Replan, Complete, Fail (each carrying its own payload).
- application/CitationEvaluator.java — deterministic citation checker.
  Signals: URL pattern https?://\S+, author-year pattern [A-Z][a-z]+\s+\d{4},
  document path pattern sample-data/docs/\S+, ISBN pattern 978-\d{9}[\dX].
  Confidence: each matched signal adds 0.25 (capped at 1.0). CITED if ≥ 1 signal
  matched; UNCITED otherwise. noSourceReason states which signals were absent.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9599 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/query-prompts.jsonl with 8 canned research
  queries spanning regulatory research, technical literature, market analysis,
  and historical background topics.
- src/main/resources/sample-data/web-fixtures.jsonl — 10 canned web fixtures
  (host, path, title, excerpt, sourceUrl). At least 4 entries carry a clear
  sourceUrl so the citation evaluator marks them CITED.
- src/main/resources/sample-data/docs/ — 6 short document fixture files.
  At least one carries an ISBN and a source URL for J4 acceptance test.
  One file deliberately has no author, date, or URL to exercise the UNCITED path.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1, HO1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/research-planner.md, prompts/web-search.md, prompts/document-search.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Research Agent", one-line
  pitch, prerequisites (including integration form host-software requirement: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO "Visual" prefix on
  tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports
  for markdown and YAML libs are acceptable. Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs
  from governance.html with answers populated from risk-survey.yaml; unanswered
  .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source
  table with click-to-expand rows), App UI (form + operator halt/resume control +
  live list with status pills and expand-on-click for ledgers, citation verdicts,
  and report). Browser title exactly: <title>Akka Sample: Research Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value in Claude session memory only.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    research-planner.json — sectioned by task id:
      "PLAN_SEARCH" → 4–6 SearchLedger entries (context, knownFacts, missingFacts,
      plan steps spanning web and document searches).
      "DECIDE_NEXT" → 4–6 NextStep entries covering Continue (with SearchDecision
      across both searchers), Replan, Complete, Fail. Continue entries advance
      through a plausible research narrative when iterated in order.
      "COMPOSE_REPORT" → 4 ResearchReport entries with 60–120 word summaries,
      3–5 findings, and 2–4 citations each.
    web-search.json — 6 SearchResult entries, ok=true, content fields are
      mocked search results (4–6 lines), at least 4 include a source URL.
    document-search.json — 6 SearchResult entries; at least 2 include a document
      path and ISBN; one entry has no traceable source for the J2 UNCITED path.
- A MockModelProvider.seedFor(jobId) helper makes selection deterministic per
  job id so the same job in dev produces the same output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep, decideStep,
  composeReportStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and on
  the ResearchJob entity state (searchLedger, findingsLedger, report,
  failureReason, haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  SearcherTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current lineup.
  Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: HTTP port 9599 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  themeVariables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow; no key value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. No hidden zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
