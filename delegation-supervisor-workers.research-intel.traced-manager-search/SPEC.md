# SPEC — traced-manager-search

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Inspect Multi-Agent Run (Phoenix).
**One-line pitch:** Submit a query; a manager agent delegates web search to a search agent and page visits to retrieve results, while Phoenix instrumentation captures per-step token usage and timing so you can inspect the full run trace.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern where a Workflow fans search work out to a ToolCallingAgent (SearchAgent), collects an OpenInference/Phoenix trace of the run, and asks a second agent (TraceInspector) to summarise token usage and timing. The blueprint also demonstrates a **before-tool-call guardrail** that vets every URL before the SearchAgent is allowed to visit it, and an **on-decision-eval** that samples the manager's synthesis decision for quality scoring.

## 3. User-facing flows

The user opens the App UI tab and submits a query via the form.

1. The system creates an `AgentRun` record in `DISPATCHED` and starts a `RunWorkflow`.
2. The ManagerAgent decomposes the query into a `SearchPlan { searchQuery, maxPages }`.
3. The workflow dispatches the SearchAgent with the plan. Before each page visit the before-tool-call guardrail checks the target URL against the allow-list.
4. SearchAgent returns a `SearchResultBundle { results, tokenUsage, durationMs }`.
5. The workflow emits a `PhoenixSpan` for each tool call and assembles a `RunTrace { spans, totalTokens, totalDurationMs }`.
6. TraceInspector analyses the trace and returns a `TraceReport { summary, hotStep, tokenBreakdown }`.
7. ManagerAgent synthesises a `SynthesisedAnswer { answer, sources, guardrailVerdict }`.
8. The run enters `TRACED`. The App UI shows the answer, trace summary, and eval score side by side.

If the SearchAgent visits more than `maxPages` URLs, the workflow short-circuits: it synthesises from whatever results exist and the run enters `PARTIAL`.

A `QuerySimulator` (TimedAction) drips a sample query every 60 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ManagerAgent` | `AutonomousAgent` | Decomposes the query into a SearchPlan; later synthesises results. | `RunWorkflow` | returns typed result to workflow |
| `SearchAgent` | `AutonomousAgent` | Executes web search and page visits; subject to URL guardrail. | `RunWorkflow` | — |
| `TraceInspector` | `AutonomousAgent` | Analyses the assembled Phoenix trace; returns a TraceReport. | `RunWorkflow` | — |
| `RunWorkflow` | `Workflow` | Coordinates the plan step, search step, trace-assembly step, inspect step, and synthesis step. | `RunEndpoint`, `RunRequestConsumer` | `RunEntity` |
| `RunEntity` | `EventSourcedEntity` | Holds the run's lifecycle (dispatched → searching → traced / partial / blocked-tool). | `RunWorkflow` | `RunView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay/audit. | `RunEndpoint`, `QuerySimulator` | `RunRequestConsumer` |
| `RunView` | `View` | List-of-runs read model. | `RunEntity` events | `RunEndpoint` |
| `RunRequestConsumer` | `Consumer` | Listens to `QueryQueue` events and starts one `RunWorkflow` per query. | `QueryQueue` events | `RunWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 60 s. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Samples one traced run every 5 minutes for eval scoring; emits a `RunEvalScored` event. | scheduler | `RunEntity` |
| `RunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, get, list, SSE. | — | `RunView`, `QueryQueue`, `RunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record SearchPlan(String searchQuery, int maxPages) {}

record PageResult(String url, String title, String excerpt, int tokensConsumed) {}

record SearchResultBundle(List<PageResult> results, int totalTokens, long durationMs) {}

record PhoenixSpan(String spanId, String stepName, String toolName,
                   int inputTokens, int outputTokens, long durationMs, Instant startedAt) {}

record RunTrace(List<PhoenixSpan> spans, int totalTokens, long totalDurationMs) {}

record TraceReport(String summary, String hotStep, Map<String, Integer> tokenBreakdown) {}

record SynthesisedAnswer(String answer, List<String> sources,
                         String guardrailVerdict, Instant synthesisedAt) {}

record AgentRun(
    String runId,
    String query,
    RunStatus status,
    Optional<SearchPlan> plan,
    Optional<SearchResultBundle> searchResults,
    Optional<RunTrace> trace,
    Optional<TraceReport> traceReport,
    Optional<SynthesisedAnswer> answer,
    Optional<String> blockedUrl,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus { DISPATCHED, SEARCHING, TRACED, PARTIAL, BLOCKED_TOOL }
```

### Events (on `RunEntity`)

`RunCreated`, `PlanReady`, `SearchCompleted`, `TraceAssembled`, `TraceInspected`,
`RunSynthesised`, `RunPartial`, `ToolCallBlocked`, `RunEvalScored`.

### Events (on `QueryQueue`)

`QuerySubmitted { runId, query, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ query }` → `{ runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=DISPATCHED|SEARCHING|TRACED|PARTIAL|BLOCKED_TOOL`.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Inspect Multi-Agent Run (Phoenix)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a query, live list of runs with status pills, expand-row to see search results, trace summary, synthesised answer, and eval score.

Browser title: `<title>Akka Sample: Inspect Multi-Agent Run (Phoenix)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — before-tool-call guardrail** (`before-tool-call` on `SearchAgent`): checks every URL the agent proposes to visit against a configurable allow-list before the call is made. Blocking. Failure → `BLOCKED_TOOL`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one traced run every 5 minutes and emits a `RunEvalScored` event with a 1–5 score and a short rationale based on answer quality and trace efficiency.

## 9. Agent prompts

- `ManagerAgent` → `prompts/manager-agent.md`. Decomposes the query into a SearchPlan; later synthesises the search results and trace report into a final answer.
- `SearchAgent` → `prompts/search-agent.md`. Executes web search and page visits; returns a SearchResultBundle.
- `TraceInspector` → `prompts/trace-inspector.md`. Analyses the assembled RunTrace and returns a TraceReport identifying the hottest step and token breakdown.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a query; run progresses DISPATCHED → SEARCHING → TRACED within 90 s; UI reflects each transition via SSE; synthesised answer and trace summary are visible.
2. **J2** — SearchAgent proposes a blocked URL; guardrail fires before the call; run enters BLOCKED_TOOL with the offending URL recorded.
3. **J3** — SearchAgent exceeds `maxPages`; workflow short-circuits to PARTIAL; synthesised answer notes incomplete search.
4. **J4** — Wait for EvalSampler; traced run gains an eval score visible in the App UI row.
5. **J5** — QuerySimulator drips queries without user interaction; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named traced-manager-search demonstrating the
delegation-supervisor-workers × research-intel cell with Phoenix observability.
Runs out of the box (no external services). Maven group io.akka.samples. Maven
artifact delegation-supervisor-workers-research-intel-traced-manager-search.
Java package io.akka.samples.inspectmultiagentrunphoenix. Akka 3.6.0. HTTP port 9305.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ManagerAgent — definition() with capability(TaskAcceptance.of(PLAN_QUERY).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE_ANSWER).maxIterationsPerTask(3)). System prompt
    loaded from prompts/manager-agent.md. Returns SearchPlan{searchQuery, maxPages} for PLAN_QUERY
    and SynthesisedAnswer{answer, sources, guardrailVerdict, synthesisedAt} for SYNTHESISE_ANSWER.
  * SearchAgent — capability(TaskAcceptance.of(EXECUTE_SEARCH).maxIterationsPerTask(5)).
    Subject to before-tool-call guardrail checking every URL against an allow-list loaded from
    application.conf (akka.javasdk.agent.search-agent.url-allowlist). System prompt from
    prompts/search-agent.md. Returns SearchResultBundle{results: List<PageResult{url, title,
    excerpt, tokensConsumed}>, totalTokens, durationMs}.
  * TraceInspector — capability(TaskAcceptance.of(INSPECT_TRACE).maxIterationsPerTask(2)).
    System prompt from prompts/trace-inspector.md. Returns TraceReport{summary, hotStep,
    tokenBreakdown: Map<String,Integer>}.

- 1 Workflow RunWorkflow with steps:
  planStep -> searchStep -> assembleTraceStep -> inspectStep -> synthesiseStep -> emitStep.
  planStep calls forAutonomousAgent(ManagerAgent.class, PLAN_QUERY).
  searchStep calls forAutonomousAgent(SearchAgent.class, EXECUTE_SEARCH) with stepTimeout(90s);
    if SearchAgent exceeds maxPages, workflow routes to partialStep ending with RunPartial.
  assembleTraceStep collects PhoenixSpan records from each tool call recorded during searchStep
    and builds a RunTrace; this step is deterministic Java — no LLM call.
  inspectStep calls forAutonomousAgent(TraceInspector.class, INSPECT_TRACE) with stepTimeout(60s).
  synthesiseStep calls forAutonomousAgent(ManagerAgent.class, SYNTHESISE_ANSWER) with
    stepTimeout(90s). WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity RunEntity holding state AgentRun{runId, query, RunStatus,
  Optional<SearchPlan> plan, Optional<SearchResultBundle> searchResults,
  Optional<RunTrace> trace, Optional<TraceReport> traceReport,
  Optional<SynthesisedAnswer> answer, Optional<String> blockedUrl,
  Optional<String> failureReason, Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}.
  RunStatus enum: DISPATCHED, SEARCHING, TRACED, PARTIAL, BLOCKED_TOOL.
  Events: RunCreated, PlanReady, SearchCompleted, TraceAssembled, TraceInspected,
  RunSynthesised, RunPartial, ToolCallBlocked, RunEvalScored.
  Commands: createRun, recordPlan, recordSearch, recordTrace, recordTraceReport,
  synthesise, markPartial, blockTool, recordEval, getRun.
  emptyState() returns AgentRun.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(query, requestedBy) emitting
  QuerySubmitted{runId, query, requestedBy, submittedAt}.

- 1 View RunView with row type AgentRunRow (mirrors AgentRun minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes RunEntity events. ONE query
  getAllRuns SELECT * AS runs FROM run_view. No WHERE status filter — caller filters client-side.

- 1 Consumer RunRequestConsumer subscribed to QueryQueue events; on QuerySubmitted
  starts a RunWorkflow with the runId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/search-queries.jsonl and calls QueryQueue.enqueueQuery.
  * EvalSampler — every 5 minutes, queries RunView.getAllRuns, picks the oldest
    TRACED run without an evalScore, runs a 1–5 rubric judge over the synthesised answer
    and trace efficiency, then calls RunEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * RunEndpoint at /api with POST /runs, GET /runs, GET /runs/{id},
    GET /runs/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- RunTasks.java declaring four Task<R> constants: PLAN_QUERY (SearchPlan),
  EXECUTE_SEARCH (SearchResultBundle), INSPECT_TRACE (TraceReport),
  SYNTHESISE_ANSWER (SynthesisedAnswer).
- Domain records SearchPlan, PageResult, SearchResultBundle, PhoenixSpan, RunTrace,
  TraceReport, SynthesisedAnswer, QueryRequest.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9305 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also
  akka.javasdk.agent.search-agent.url-allowlist = ["https://en.wikipedia.org",
  "https://arxiv.org", "https://www.bbc.com", "https://news.ycombinator.com"].
- src/main/resources/sample-events/search-queries.jsonl with 8 canned query lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose as traced agent observability,
  decisions.authority_level = recommend-only, data.pii = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/manager-agent.md, prompts/search-agent.md, prompts/trace-inspector.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Inspect Multi-Agent Run (Phoenix)",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills; expand row shows search results, trace summary with per-step token bars, synthesised
  answer, eval score). Browser title exactly:
  <title>Akka Sample: Inspect Multi-Agent Run (Phoenix)</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing ModelProvider with per-agent dispatch
  on agent class name. Each branch reads from src/main/resources/mock-responses/<agent>.json.
- Per-agent mock-response shapes:
    manager-agent.json — 4–6 SearchPlan entries (searchQuery + maxPages pairs) and
      4–6 SynthesisedAnswer entries (each with a 60–120 word answer, 2–4 source URLs,
      guardrailVerdict = "ok").
    search-agent.json — 4–6 SearchResultBundle entries, each with 3–5 PageResult objects
      whose url values are within the allow-list.
    trace-inspector.json — 4–6 TraceReport entries, each with a one-sentence summary,
      a hotStep name matching a plausible tool call, and a tokenBreakdown map with
      3–5 named steps.
- MockModelProvider.seedFor(runId) makes selection deterministic per run id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (90s search,
  60s inspect, 90s synthesise); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion RunTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9305 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text.
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
