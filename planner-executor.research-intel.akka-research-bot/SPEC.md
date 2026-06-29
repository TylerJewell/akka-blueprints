# SPEC — akka-research-bot

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Research Bot (Planner/Searcher/Writer).
**One-line pitch:** Submit a research question; a PlannerAgent produces a targeted search plan, a SearcherAgent executes each query in parallel, and a WriterAgent synthesizes the collected results into a cited report — all orchestrated as a durable workflow.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to information research. The PlannerAgent owns a **research plan** (the question, a list of search queries, each tagged with a `SearchKind` and an expected knowledge contribution) and, on revision, produces an updated plan. Each search query is dispatched to the SearcherAgent as a separate subtask; results land on a **result ledger** (append-only list of `SearchResult` records). After the planner judges the ledger sufficient, the WriterAgent synthesizes everything into a `ResearchReport`.

The blueprint also demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each search query before the SearcherAgent runs — blocking queries that name disallowed topics or match restricted host patterns,
- a **before-agent-response guardrail** that inspects the WriterAgent's draft report before it is published — blocking reports that contain content-policy violations, secret-shaped strings, or disallowed citations.

## 3. User-facing flows

The user opens the App UI tab and submits a research question via the form.

1. The system creates a `ResearchJob` in `PLANNING` and starts a `ResearchWorkflow`.
2. The PlannerAgent receives the question and produces a `ResearchPlan { question, queries: List<SearchQuery>, notes }`. The job emits `JobPlanned`.
3. The workflow enters the search loop. For each `SearchQuery` in the plan:
   - The **before-tool-call guardrail** vets the query against the topic allow-list and the host allow-list; on rejection it records a `QueryBlocked` entry and the workflow asks the PlannerAgent to revise the plan.
   - The SearcherAgent runs the query and returns a `SearchResult`.
   - A secret sanitizer scrubs the result content.
   - The workflow appends a `ResultEntry` to the result ledger via the entity.
4. After all approved queries complete, the workflow calls the PlannerAgent in `ASSESS` mode. The planner returns `PlanAssessment`: `SUFFICIENT` (proceed to writing) or `NEEDS_MORE(revisedPlan)` (replan). Budget: at most two replans; a third triggers `FAIL`.
5. The WriterAgent receives the full result ledger and produces a draft `ResearchReport { title, summary, sections: List<ReportSection>, bibliography }`.
6. The **before-agent-response guardrail** inspects the draft report. On rejection the workflow records `ReportBlocked` and asks the WriterAgent to revise (at most two revision attempts; a third triggers `FAIL`).
7. On approval, the workflow publishes the report via `JobEntity.publishReport(report)` and emits `JobCompleted`.
8. The operator can press **Halt new searches** at any time. The workflow records the in-flight search result, then exits with `JobHaltedOperator`.

A `JobSimulator` (TimedAction) drips a sample research question every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Produces `ResearchPlan` (PLAN mode) and `PlanAssessment` (ASSESS mode). | `ResearchWorkflow` | returns typed result to workflow |
| `SearcherAgent` | `AutonomousAgent` | Executes one search query against seeded fixtures; returns `SearchResult`. | `ResearchWorkflow` | — |
| `WriterAgent` | `AutonomousAgent` | Synthesizes the result ledger into a `ResearchReport`; revises on REVISE task. | `ResearchWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Drives the plan → query-guarded → search → sanitize → record → assess → write → response-guarded → publish loop. | `JobEndpoint`, `JobRequestConsumer` | `ResearchJobEntity` |
| `ResearchJobEntity` | `EventSourcedEntity` | Holds the job lifecycle, research plan, result ledger, and final report. | `ResearchWorkflow` | `ResearchJobView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `JobEndpoint` (operator action) | `ResearchWorkflow` (polls) |
| `JobQueue` | `EventSourcedEntity` | Audit log of submitted jobs. | `JobEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `ResearchJobView` | `View` | List-of-jobs read model for the UI. | `ResearchJobEntity` events | `JobEndpoint` |
| `JobRequestConsumer` | `Consumer` | Subscribes to `JobQueue` events; starts a `ResearchWorkflow` per submission. | `JobQueue` events | `ResearchWorkflow` |
| `JobSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/job-prompts.jsonl` and enqueues it. | scheduler | `JobQueue` |
| `StaleJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `SEARCHING` past 5 minutes as `STALE`. | scheduler | `ResearchJobEntity` |
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE, operator halt. | — | `ResearchJobView`, `JobQueue`, `ResearchJobEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record JobRequest(String question, String requestedBy) {}

record SearchQuery(
    SearchKind kind,
    String query,
    String expectedContribution
) {}

record ResearchPlan(
    String question,
    List<SearchQuery> queries,
    Optional<String> notes
) {}

record SearchResult(
    String query,
    SearchKind kind,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record ResultEntry(
    int attempt,
    String query,
    SearchKind kind,
    ResultVerdict verdict,
    String scrubbedContent,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ResultLedger(List<ResultEntry> entries) {}

record ReportSection(String heading, String body, List<String> citations) {}

record ResearchReport(
    String title,
    String summary,
    List<ReportSection> sections,
    List<String> bibliography,
    Instant producedAt
) {}

record ResearchJob(
    String jobId,
    String question,
    JobStatus status,
    Optional<ResearchPlan> plan,
    Optional<ResultLedger> results,
    Optional<ResearchReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SearchKind { WEB, ACADEMIC, NEWS, PATENT }
enum ResultVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, REDACTED }
enum JobStatus { PLANNING, SEARCHING, WRITING, COMPLETED, FAILED, HALTED, STALE }
```

### Events (`ResearchJobEntity`)

`JobCreated`, `JobPlanned`, `QueryDispatched`, `QueryBlocked`, `ResultRecorded`, `PlanRevised`, `WritingStarted`, `ReportBlocked`, `JobCompleted`, `JobFailed`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`JobQueue`)

`JobSubmitted { jobId, question, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ question, requestedBy? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full plan + results + report).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Research Bot (Planner/Searcher/Writer)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a research question, operator halt/resume control, live list of jobs with status pills, expand-row to see the research plan, result ledger entries, and the final report.

Browser title: `<title>Akka Sample: Research Bot (Planner/Searcher/Writer)</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `SearcherAgent`): every `SearchQuery` is checked against (a) a topic allow-list that blocks queries touching restricted subjects, (b) a host allow-list restricting web searches to approved domains. Blocking. Failure → `QueryBlocked` entry + plan revision request.
- **G2 — before-agent-response guardrail** (`before-agent-response` on `WriterAgent`): the draft `ResearchReport` is inspected before it is accepted by the workflow. The checker verifies that every bibliography citation is drawn from the result ledger, that no secret-shaped string survived the sanitizer, and that the report passes a content-policy scan. On rejection the workflow records `ReportBlocked` and asks the WriterAgent to revise. After two rejections the job fails.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Produces research plans; assesses result ledger sufficiency.
- `SearcherAgent` → `prompts/searcher.md`. Executes one search query against seeded fixtures.
- `WriterAgent` → `prompts/writer.md`. Synthesizes results into a structured report.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What are the key provisions of the EU AI Act and how do they affect autonomous decision-making systems?" Task progresses `PLANNING → SEARCHING → WRITING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a research plan with 3–6 queries, a result ledger with entries spanning at least WEB and NEWS kinds, and a `ResearchReport` with a non-empty summary and bibliography.
2. **J2** — Submit a question whose plan would search a restricted topic. The before-tool-call guardrail blocks the offending query on the first attempt; the planner revises the plan; the job either completes via a different query path or fails after the replan budget is exhausted.
3. **J3** — Submit a job and click **Halt new searches** while the job is `SEARCHING`. The in-flight search result is recorded normally; no further queries dispatch; the job ends in `HALTED`.
4. **J4** — A search result fixture contains an `AKIA...` key shape. The sanitizer scrubs it before it reaches the writer's synthesis prompt; the final report does not contain the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-research-bot demonstrating the
planner-executor × research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-research-intel-akka-research-bot. Java
package io.akka.samples.researchbotplannersearcherwriter. Akka 3.6.0. HTTP port 9516.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PlannerAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(ASSESS).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN returns ResearchPlan.
    ASSESS returns PlanAssessment (sealed interface permitting Sufficient
    and NeedsMore(revisedPlan: ResearchPlan)).
  * SearcherAgent — capability(TaskAcceptance.of(SEARCH).maxIterationsPerTask(2)).
    Prompt from prompts/searcher.md. Returns SearchResult.
  * WriterAgent — capability(TaskAcceptance.of(DRAFT_REPORT).maxIterationsPerTask(2))
      and capability(TaskAcceptance.of(REVISE_REPORT).maxIterationsPerTask(2)).
    Prompt from prompts/writer.md. Returns ResearchReport.

- 1 Workflow ResearchWorkflow with steps:
  planStep -> [loop entry for each query] checkHaltStep -> queryGuardrailStep ->
  searchStep -> sanitizeStep -> recordStep -> [after all queries] assessStep ->
  [Sufficient] writeStep -> reportGuardrailStep -> publishStep / [NeedsMore] replanStep ->
  [back to loop] / [max replans exceeded] failStep. Terminal steps: completedStep /
  failStep / haltedStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), searchStep ofSeconds(90) (covers fixture lookup
    plus any warm-up), writeStep ofSeconds(120), assessStep ofSeconds(45),
    reportGuardrailStep ofSeconds(30). defaultStepRecovery(maxRetries(2).failoverTo
    (ResearchWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits JobHaltedOperator on ResearchJobEntity).
  queryGuardrailStep runs QueryGuardrail.vet(SearchQuery); on reject records a
  QueryBlocked entry via ResearchJobEntity.recordBlock(query, reason) and
  skips this query (continues to next query in plan).
  searchStep uses SearcherAgent via forAutonomousAgent(...).runSingleTask(...).
  sanitizeStep applies SecretScrubber.scrub to the SearchResult.content.
  recordStep calls ResearchJobEntity.recordResult(entry).
  assessStep calls PlannerAgent with ASSESS task; on NeedsMore emits PlanRevised
  and loops back to the query loop; replan budget = 2.
  writeStep calls WriterAgent with DRAFT_REPORT task over the full result ledger.
  reportGuardrailStep runs ReportGuardrail.inspect(ResearchReport); on reject
  records ReportBlocked and calls WriterAgent with REVISE_REPORT task;
  at most 2 revision attempts before failStep.
  publishStep calls ResearchJobEntity.publishReport(report); transitions to
  completedStep which emits JobCompleted.

- 1 EventSourcedEntity ResearchJobEntity holding ResearchJob state. emptyState()
  returns ResearchJob.initial("", null) with no commandContext() reference. Commands:
  createJob, recordPlan, recordQueryDispatch, recordBlock, recordResult, revisePlan,
  startWriting, recordReportBlock, publishReport, failJob, haltOperator, timeoutFail,
  getJob. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity JobQueue with command enqueueJob(jobId, question,
  requestedBy) emitting JobSubmitted.

- 1 View ResearchJobView with row type JobRow (mirror of ResearchJob minus
  heavy result payloads — truncate to last 3 result entries plus counts; the
  UI fetches the full job by id on click). Table updater consumes
  ResearchJobEntity events. ONE query getAllJobs SELECT * AS jobs FROM
  research_job_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer JobRequestConsumer subscribed to JobQueue events; on JobSubmitted
  starts a ResearchWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 90s, reads next line from
    src/main/resources/sample-events/job-prompts.jsonl and calls
    JobQueue.enqueueJob.
  * StaleJobMonitor — every 30s, queries ResearchJobView.getAllJobs, filters
    SEARCHING jobs whose createdAt is older than 5 minutes, calls
    ResearchJobEntity.timeoutFail; ResearchWorkflow polls ResearchJobEntity.getJob
    in its checkHaltStep and ends when status == STALE.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring two Task<R> constants: PLAN
  (resultConformsTo ResearchPlan), ASSESS (resultConformsTo PlanAssessment).
- WriterTasks.java declaring two Task<R> constants: DRAFT_REPORT
  (resultConformsTo ResearchReport), REVISE_REPORT (resultConformsTo ResearchReport).
- SearcherTasks.java declaring one Task<R> constant: SEARCH
  (resultConformsTo SearchResult).
- Domain records as listed in SPEC §5, plus a PlanAssessment sealed interface
  with permits Sufficient and NeedsMore(revisedPlan: ResearchPlan).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens >= 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/QueryGuardrail.java — deterministic vetter. Reject if the
  SearchKind is not in {WEB, ACADEMIC, NEWS, PATENT}, if a WEB query names
  a host not on the allow-list (akka.io, doc.akka.io, github.com,
  arxiv.org, scholar.google.com), if the query text matches any restricted
  topic pattern (/\b(weapon|exploit|zero.?day|bypass|jailbreak)\b/i).
- application/ReportGuardrail.java — pre-publish inspection. Reject the
  report if any bibliography citation is not traceable to a ResultEntry,
  if any secret-shaped pattern survives (re-apply SecretScrubber.scan),
  if any section body contains content-policy markers
  ("ignore previous instructions", "<system>", "\\n\\n---\\n").
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9516 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/job-prompts.jsonl with 8 canned research
  questions spanning legal, technical, news, and academic topics.
- src/main/resources/sample-data/web-fixtures.jsonl — 14 canned web search
  fixtures (host, path, title, excerpt). Include one fixture whose content
  contains an AKIA-shaped key fragment for the J4 acceptance test.
- src/main/resources/sample-data/academic-fixtures.jsonl — 8 canned
  academic/arxiv-style fixtures.
- src/main/resources/sample-data/news-fixtures.jsonl — 8 canned news
  fixtures.
- src/main/resources/sample-data/patent-fixtures.jsonl — 4 canned patent
  abstract fixtures.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, G2) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance for the
  research-intel domain; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/searcher.md, prompts/writer.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Research Bot
  (Planner/Searcher/Writer)", one-line pitch, prerequisites (integration
  form host-software requirement: None), generate-the-system, what-you-
  get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for plan, result ledger, and report).
  Browser title exactly: <title>Akka Sample: Research Bot
  (Planner/Searcher/Writer)</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
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
  the ModelProvider interface with per-agent dispatch on the agent class
  name and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent:
  planner.json, searcher.json, writer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    planner.json — a sectioned file with two lists keyed by task id:
      "PLAN" → 4–6 ResearchPlan entries (question, 3–5 SearchQuery items
      spanning WEB/ACADEMIC/NEWS kinds, notes about expected findings).
      "ASSESS" → 4–6 PlanAssessment entries — a mix of Sufficient and
      NeedsMore (each NeedsMore carries a trimmed revised plan). The
      Sufficient entries appear after at least 2 SearchQuery results have
      been iterated.
    searcher.json — 8 SearchResult entries (ok=true), content fields are
      mocked search excerpts (4–6 lines each with host + title citations).
      ONE entry's content must include the literal substring
      "AKIAIOSFODNN7EXAMPLE" so the J4 sanitizer test fires.
    writer.json — 4 ResearchReport entries with 60–120 word summaries,
      2–3 ReportSection entries each, and bibliography lists drawn from
      the search result fixture hosts.
- A MockModelProvider.seedFor(jobId) helper makes the selection
  deterministic per job id so the same job in dev produces the same
  output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent — the
  base class clause must read `extends AutonomousAgent` for every agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, searchStep, writeStep, assessStep,
  reportGuardrailStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the ResearchJob entity state (plan, results, report, failureReason,
  haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java,
  SearcherTasks.java, and WriterTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9516 in application.conf — picked from the
  available range.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM — delete
  removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
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
