# SPEC — reddit-search

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RedditSearch.
**One-line pitch:** A user submits a research topic; one AI agent drives a headless browser to search Reddit, reads post threads, and returns a structured `ResearchReport` — ranked post summaries, extracted themes, and sentiment signals — without any manual scraping code.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `BrowserResearchAgent` (AutonomousAgent) carries the entire extraction; the surrounding components only schedule its task and persist its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every browser navigation request the agent issues. Before the headless browser executes a `navigate(url)` call, `NavigationGuardrail` validates the destination against an allow-list (Reddit domains and HTTPS-only). A blocked URL causes the agent loop to record a `navigation.blocked` event and try an alternative rather than crashing the session.
- A **budget-based automatic halt** tracks the number of pages visited per agent session. Once the visit count hits the configured ceiling (default 20), `BudgetEnforcer` terminates the agent loop and records `BudgetExhausted`. The partial report collected so far is preserved; the job transitions to `BUDGET_EXHAUSTED` rather than `FAILED`.

The blueprint shows that a browser-driving agent is not intrinsically ungoverned: the before-tool-call hook puts policy at the navigation boundary, and the budget halt prevents runaway cost.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a research topic into the **Topic** input (e.g., "Akka actor model use cases") or picks one of three seeded topics.
2. The user optionally sets a **subreddit scope** (default: search across all of Reddit; or restrict to a comma-separated list like `r/scala,r/java`).
3. The user sets a **max pages** budget (1–20, default 10) and clicks **Start research**. The UI POSTs to `/api/jobs` and receives a `jobId`.
4. A job card appears in the live list with status `QUEUED`. Within ~1 s it transitions to `BROWSING` — the agent has started visiting pages. The card shows a live page-visit counter.
5. Within 30–120 s (depending on depth and LLM latency), the job reaches `REPORT_READY`. The report appears in the right pane: a ranked list of post summaries (title, subreddit, upvotes, summary sentence, URL), a theme list (extracted noun-phrases common across posts), and a sentiment bar (positive / neutral / negative count).
6. If the page budget is exhausted before the agent finishes, the card transitions to `BUDGET_EXHAUSTED` with a partial report and a note that the budget limit was reached.
7. The user can start another research job; the live list keeps history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ResearchJobEntity`, `ResearchView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ResearchJobEntity` | `EventSourcedEntity` | Per-job lifecycle: queued → browsing → report ready / budget exhausted / failed. Source of truth. | `ResearchEndpoint`, `ResearchWorkflow` | `ResearchView` |
| `ResearchWorkflow` | `Workflow` | One workflow per jobId. Steps: `browseStep` → `scoreStep`. | started by `ResearchEndpoint` on submit | `BrowserResearchAgent`, `ResearchJobEntity` |
| `BrowserResearchAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the topic and constraints as task instructions; uses a registered browser tool to navigate and extract. Returns `ResearchReport`. | invoked by `ResearchWorkflow` | returns report |
| `NavigationGuardrail` | before-tool-call hook | Validates every `navigate(url)` call before execution. Blocks off-domain and HTTP URLs. | wired on `BrowserResearchAgent` | — |
| `BudgetEnforcer` | supporting class | Tracks page-visit count per session; signals halt when budget exceeded. | invoked by `ResearchWorkflow.browseStep` | `ResearchJobEntity` |
| `ResearchView` | `View` | Read model: one row per job for the UI. | `ResearchJobEntity` events | `ResearchEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ResearchTopic(
    String topic,
    List<String> subredditScope,   // empty = all Reddit
    int maxPages                   // 1..20
) {}

record PostSummary(
    String postId,
    String title,
    String subreddit,
    int upvotes,
    String summaryLine,
    String url
) {}

record Theme(
    String phrase,
    int occurrences
) {}

record SentimentCounts(
    int positive,
    int neutral,
    int negative
) {}

record ResearchReport(
    List<PostSummary> posts,       // ranked by relevance descending
    List<Theme> themes,            // top extracted themes
    SentimentCounts sentiment,
    int pagesVisited,
    String noResultsReason,        // non-null only when posts is empty
    Instant completedAt
) {}

record ResearchJob(
    String jobId,
    Optional<ResearchTopic> topic,
    Optional<ResearchReport> report,
    JobStatus status,
    int pagesVisited,              // live counter
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus {
    QUEUED, BROWSING, REPORT_READY, BUDGET_EXHAUSTED, FAILED
}
```

Events on `ResearchJobEntity`: `JobQueued`, `BrowsingStarted`, `PageVisited`, `ReportRecorded`, `BudgetExhausted`, `JobFailed`.

Every nullable lifecycle field on the `ResearchJob` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ topic, subredditScope: [String], maxPages }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: RedditSearch</title>`.

The App UI tab is a two-column layout: a left rail with the live list of research jobs (status pill + page-visit counter + age) and a right pane with the selected job's detail — topic configuration, live page-visit count, ranked post list, theme tags, sentiment bar, and budget status.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `navigate(url)` tool call the agent issues before the browser executes it. `NavigationGuardrail` checks (1) the scheme is `https`, (2) the host is in the Reddit allow-list (`reddit.com`, `www.reddit.com`, `old.reddit.com`), and (3) the path does not match any blocked patterns (login pages, API token endpoints, direct-message paths). A blocked URL returns a structured `navigation.blocked` rejection to the agent loop, which records a warning and tries an alternative search strategy within its iteration budget.
- **H1 — automatic budget halt**: `BudgetEnforcer` counts `PageVisited` events accumulated by `ResearchJobEntity`. When `pagesVisited >= maxPages`, `browseStep` signals the workflow to stop the agent session immediately, records `BudgetExhausted`, and transitions the entity to `BUDGET_EXHAUSTED`. The partial `ResearchReport` built so far is preserved. This prevents runaway cost from loops, redirects, or unexpectedly deep pagination.

## 9. Agent prompts

- `BrowserResearchAgent` → `prompts/browser-research-agent.md`. The single decision-making LLM. System prompt instructs it to navigate Reddit search and subreddit pages, extract post metadata, and return a `ResearchReport` with ranked summaries and themes. It must honour the navigation guardrail: if a URL is blocked it should try an alternative search path rather than retrying the blocked URL.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "distributed systems tradeoffs" with default settings; within 120 s a report appears with at least 3 post summaries and a non-empty theme list.
2. **J2** — Agent attempts to navigate `http://reddit.com` (non-HTTPS) — the guardrail blocks it; the agent retries with `https://reddit.com` and the session continues normally.
3. **J3** — User sets maxPages=2; agent browses exactly 2 pages, `BudgetExhausted` fires, the job card shows `BUDGET_EXHAUSTED` with `pagesVisited = 2` and a partial report.
4. **J4** — User submits a topic with no Reddit results; the report lands with an empty `posts` list and a non-empty `noResultsReason` string; the UI shows the reason rather than a blank panel.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named reddit-search demonstrating the single-agent × research-intel cell.
Maven group io.akka.samples. Maven artifact single-agent-research-intel-browser-research-agent.
Java package io.akka.samples.redditsearch. Akka 3.6.0. HTTP port 9988.

Components to wire (exactly):

- 1 AutonomousAgent BrowserResearchAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/browser-research-agent.md>) and
  .capability(TaskAcceptance.of(RESEARCH_TOPIC).maxIterationsPerTask(5)). The task receives
  the topic, subreddit scope, and page budget as its instruction text. A registered
  BrowserTool wraps the headless-browser integration; the agent issues navigate(url) calls
  via this tool. The agent is configured with a before-tool-call guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop records a navigation.blocked event and continues with its
  remaining iterations.

- 1 Workflow ResearchWorkflow per jobId with two steps:
  * browseStep — emits BrowsingStarted, then calls componentClient.forAutonomousAgent(
    BrowserResearchAgent.class, "researcher-" + jobId).runSingleTask(
      TaskDef.instructions(formatTopic(job.topic))
    ) — returns a taskId, then forTask(taskId).result(RESEARCH_TOPIC) to fetch the report.
    During the task, BrowserResearchAgent calls BudgetEnforcer.checkBudget(jobId) before
    each page visit; BudgetEnforcer reads the entity's pagesVisited counter and if
    pagesVisited >= maxPages returns an EXCEEDED signal that causes browseStep to
    stop the task early and transition to BUDGET_EXHAUSTED rather than failing.
    WorkflowSettings.stepTimeout 180s with defaultStepRecovery maxRetries(1)
    .failoverTo(ResearchWorkflow::error).
  * scoreStep — runs a deterministic RelevanceScorer (NOT an LLM call) over the returned
    ResearchReport: re-ranks posts by a composite of upvotes + keyword-density + recency,
    caps the list at 20 entries, and records the final report via
    ResearchJobEntity.recordReport(report). WorkflowSettings.stepTimeout 10s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ResearchJobEntity (one per jobId). State ResearchJob{jobId: String,
  topic: Optional<ResearchTopic>, report: Optional<ResearchReport>, status: JobStatus,
  pagesVisited: int, createdAt: Instant, finishedAt: Optional<Instant>}.
  JobStatus enum: QUEUED, BROWSING, REPORT_READY, BUDGET_EXHAUSTED, FAILED.
  Events: JobQueued{topic}, BrowsingStarted{}, PageVisited{url: String},
  ReportRecorded{report}, BudgetExhausted{pagesVisited: int}, JobFailed{reason: String}.
  Commands: enqueue, markBrowsing, recordPageVisit, recordReport, recordBudgetExhausted,
  fail, getJob. emptyState() returns ResearchJob.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 View ResearchView with row type ResearchJobRow (mirrors ResearchJob). Table updater
  consumes ResearchJobEntity events. ONE query getAllJobs: SELECT * AS jobs FROM
  research_job_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /jobs (body {topic, subredditScope: [String],
    maxPages}; mints jobId; calls ResearchJobEntity.enqueue; starts ResearchWorkflow;
    returns {jobId}), GET /jobs (list from getAllJobs, sorted newest-first), GET /jobs/{id}
    (one row), GET /jobs/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ResearchTasks.java declaring one Task<R> constant: RESEARCH_TOPIC = Task.name("Research
  topic").description("Browse Reddit and extract structured insights for the given topic")
  .resultConformsTo(ResearchReport.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ResearchTopic, PostSummary, Theme, SentimentCounts, ResearchReport,
  ResearchJob, JobStatus.

- NavigationGuardrail.java implementing the before-tool-call hook. Reads the destination
  URL from the tool call payload. Checks (1) scheme == https, (2) host is in
  {reddit.com, www.reddit.com, old.reddit.com}, (3) path does not match blocked patterns
  (^/login, ^/register, ^/api/v1, ^/message). Returns Guardrail.allow() on pass or
  Guardrail.reject("navigation.blocked: <reason>") on failure.

- BudgetEnforcer.java — pure deterministic logic (no LLM). Reads pagesVisited from
  ResearchJobEntity and compares to maxPages from the submitted ResearchTopic. Returns
  BudgetSignal.OK or BudgetSignal.EXCEEDED. The browseStep polls this after each
  PageVisited event.

- RelevanceScorer.java — pure deterministic logic (no LLM). Re-ranks ResearchReport.posts
  by a composite of upvotes weight (0.5) + keyword-density weight (0.3) + recency
  weight (0.2). Caps at 20 entries. Documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9988 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-topics.jsonl with 3 seeded research topics:
  "Akka actor model use cases", "Rust async runtime comparison", and
  "PostgreSQL vs CockroachDB at scale". Each topic carries a subredditScope list
  and maxPages=5.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root with appropriate research-intel pre-fills and
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/browser-research-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RedditSearch", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of research job cards; right = selected-job detail with topic config, live
  page-visit counter, ranked post list, theme tags, sentiment bar, and budget note).
  Browser title exactly: <title>Akka Sample: RedditSearch</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns seeded
        ResearchReport entries from src/main/resources/mock-responses/research-topic.json.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java. Per-task mock-response shapes for THIS blueprint:
    research-topic.json — 6 ResearchReport entries, each with 3–8 PostSummary items,
    2–5 Theme items, realistic SentimentCounts, and pagesVisited values between 2 and 10.
    Include 1 entry with an empty posts list and a non-empty noResultsReason for J4.
    Include 1 entry where pagesVisited equals maxPages to exercise the budget path.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BrowserResearchAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion ResearchTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (browseStep
  180s, scoreStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on ResearchJob is Optional<T>.
- Lesson 7: ResearchTasks.java with RESEARCH_TOPIC = Task.name(...).description(...)
  .resultConformsTo(ResearchReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9988 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label reflects browser runtime choice (e.g., "Requires Playwright"
  or "Runs out of the box" with mock) — never T1/T2/T3/T4 in any user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BrowserResearchAgent).
  RelevanceScorer and BudgetEnforcer are deterministic supporting classes — they do NOT
  make LLM calls.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, unrecoverable compile error, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
