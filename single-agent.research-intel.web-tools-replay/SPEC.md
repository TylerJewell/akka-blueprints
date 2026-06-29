# SPEC — web-tools-replay

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WebToolsReplay.
**One-line pitch:** A user submits a research query; one AI agent issues web-search tool calls and records each call as a structured trace; a replay harness re-issues the same queries on a schedule and scores result-set drift against the original baseline — making provider-side behavioral changes observable.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `SearchReplayAgent` (AutonomousAgent) issues every web-search tool call; the surrounding components record its behavior and replay it deterministically. One governance mechanism is wired around the agent:

- A **periodic replay evaluator** fires after each `TraceRecorded` event. On each replay cycle it re-issues the recorded queries against the same provider stub, compares result sets by a deterministic scoring function, and emits a `DriftReport` with a 0–100 drift score and a per-query breakdown. This is a rule-based check — no LLM makes the drift decision — so the same trace always produces the same score.

The blueprint shows that a single-agent search pattern can carry meaningful behavioral governance: every provider-side change is caught by comparing a deterministic replay against the recorded baseline.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **research query** into the query input (or picks one of four seeded queries — a technology trend, a regulatory update, a competitive landscape search, a scientific literature query).
2. The user clicks **Run search**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SEARCHING` — the agent has begun issuing tool calls.
4. Within ~10–30 s the agent completes. The card transitions to `TRACE_RECORDED`. The right pane shows the recorded trace: a numbered list of each `WebSearchCall` with its query string, result count, and top-3 result titles.
5. The user clicks **Replay now** (or waits for the next scheduled replay). The card transitions to `REPLAYING`. Within ~5 s it reaches `DRIFT_CLEAR` or `DRIFT_DETECTED`. The right pane adds a drift score chip (0–100) and a per-query comparison table showing which result sets changed.
6. The user can submit another query; the live list keeps all runs visible with their drift status.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SearchRunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE, trigger replay; serves `/api/metadata/*`. | — | `SearchRunEntity`, `SearchRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SearchRunEntity` | `EventSourcedEntity` | Per-run lifecycle: submitted → searching → trace_recorded → replaying → drift_clear / drift_detected. Source of truth. | `SearchRunEndpoint`, `TraceRecordingConsumer`, `ReplayWorkflow` | `SearchRunView` |
| `TraceRecordingConsumer` | `Consumer` | Subscribes to `SearchCompleted` events; writes the full trace back via `SearchRunEntity.recordTrace`; starts a `ReplayWorkflow`. | `SearchRunEntity` events | `SearchRunEntity` |
| `ReplayWorkflow` | `Workflow` | One workflow per run. Steps: `awaitTraceStep` → `replayStep` → `scoreStep`. | started by `TraceRecordingConsumer` | `SearchReplayAgent`, `SearchRunEntity` |
| `SearchReplayAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the research query as task instructions; issues web-search tool calls; returns a `SearchTrace` of every call made. | invoked by `ReplayWorkflow.replayStep` | returns trace |
| `ReplayDriftScorer` | supporting class | Deterministic scorer (no LLM). Compares two `SearchTrace` objects field-by-field and emits a `DriftReport`. | invoked by `ReplayWorkflow.scoreStep` | returns report |
| `SearchRunView` | `View` | Read model: one row per run for the UI. | `SearchRunEntity` events | `SearchRunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SearchQuery(
    String queryId,
    String text,
    String topic,
    Instant submittedAt
) {}

record WebSearchCall(
    String callId,
    String queryString,
    int resultCount,
    List<SearchResult> results,
    Instant calledAt
) {}

record SearchResult(
    String title,
    String url,
    String snippet
) {}

record SearchTrace(
    String traceId,
    String runId,
    List<WebSearchCall> calls,
    Instant recordedAt
) {}

record DriftEntry(
    String callId,
    String queryString,
    int originalResultCount,
    int replayResultCount,
    int titleOverlapPct,    // 0..100
    boolean changed
) {}

record DriftReport(
    int driftScore,         // 0..100
    String rationale,
    List<DriftEntry> entries,
    Instant evaluatedAt
) {}

record SearchRun(
    String runId,
    Optional<SearchQuery> query,
    Optional<SearchTrace> originalTrace,
    Optional<SearchTrace> replayTrace,
    Optional<DriftReport> driftReport,
    SearchRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SearchRunStatus {
    SUBMITTED, SEARCHING, TRACE_RECORDED, REPLAY_QUEUED, REPLAYING, DRIFT_CLEAR, DRIFT_DETECTED, FAILED
}
```

Events on `SearchRunEntity`: `RunSubmitted`, `SearchStarted`, `SearchCompleted`, `TraceRecorded`, `ReplayStarted`, `DriftScored`, `RunFailed`.

Every nullable lifecycle field on the `SearchRun` record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ queryText, topic, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/{id}/replay` — trigger a manual replay of a `TRACE_RECORDED` run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Server-Tool Web-Search Replay</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + drift score chip + age) and a right pane with the selected run's detail — query text, recorded trace call list, replay trace call list (when available), and drift report with per-query breakdown.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — periodic replay eval** (`eval-periodic`, `replay-eval`): fires after each `TraceRecorded` event as `replayStep` + `scoreStep` inside the workflow. The agent re-issues the same queries from the original trace. `ReplayDriftScorer` (deterministic, no LLM call) compares original and replay result sets: title overlap percentage, result-count delta, URL churn. Emits `DriftScored` with a 0–100 drift score and a per-query `DriftEntry` list. Runs with drift score ≥ 30 are flagged `DRIFT_DETECTED`; below that threshold they close as `DRIFT_CLEAR`.

## 9. Agent prompts

- `SearchReplayAgent` → `prompts/search-replay-agent.md`. The single decision-making LLM. System prompt instructs it to issue web-search tool calls for the given research query and return a complete `SearchTrace` of every call made, including raw result titles, URLs, and snippets.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the "AI regulatory update" seed query; within 30 s the trace appears with ≥ 1 `WebSearchCall` and the run reaches `TRACE_RECORDED`.
2. **J2** — A replay is triggered on a `TRACE_RECORDED` run; `ReplayDriftScorer` compares result sets; the run reaches `DRIFT_CLEAR` or `DRIFT_DETECTED` within 5 s of replay completion.
3. **J3** — A replay against a modified stub (simulating provider result-set change) produces a drift score ≥ 30 and the run flags `DRIFT_DETECTED` with per-query entries visible in the UI.
4. **J4** — The same query submitted twice produces two independent runs; each trace is recorded separately and each can be replayed independently.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named web-tools-replay demonstrating the single-agent × research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-web-tools-replay. Java package
io.akka.samples.servertoolwebsearchreplay. Akka 3.6.0. HTTP port 9566.

Components to wire (exactly):

- 1 AutonomousAgent SearchReplayAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/search-replay-agent.md>) and
  .capability(TaskAcceptance.of(SEARCH_AND_TRACE).maxIterationsPerTask(3)). The task receives
  the research query as its instruction text. Output: SearchTrace{traceId: String, runId: String,
  calls: List<WebSearchCall>, recordedAt: Instant}. The agent uses a web-search tool stub
  (WebSearchStub.java) that returns seeded result sets deterministically per query string.
  The agent has no guardrail — structural validity of SearchTrace is enforced by the task's
  resultConformsTo declaration.

- 1 Workflow ReplayWorkflow per runId with three steps:
  * awaitTraceStep — polls SearchRunEntity.getRun every 1s; on run.originalTrace().isPresent()
    advances to replayStep. WorkflowSettings.stepTimeout 20s.
  * replayStep — emits ReplayStarted, then calls componentClient.forAutonomousAgent(
    SearchReplayAgent.class, "replay-" + runId).runSingleTask(
      TaskDef.instructions(run.query().get().text())
    ) — returns a taskId, then forTask(taskId).result(SEARCH_AND_TRACE) to fetch the replay
    trace. On success calls SearchRunEntity.recordReplayTrace(replayTrace).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ReplayWorkflow::error).
  * scoreStep — runs ReplayDriftScorer (NOT an LLM call) over the original and replay traces:
    computes per-call title overlap, result-count delta, URL churn; produces a 0–100 drift
    score. Drift score >= 30 → DRIFT_DETECTED; < 30 → DRIFT_CLEAR. Emits DriftScored{report}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SearchRunEntity (one per runId). State SearchRun{runId: String,
  query: Optional<SearchQuery>, originalTrace: Optional<SearchTrace>,
  replayTrace: Optional<SearchTrace>, driftReport: Optional<DriftReport>,
  status: SearchRunStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  SearchRunStatus enum: SUBMITTED, SEARCHING, TRACE_RECORDED, REPLAY_QUEUED, REPLAYING,
  DRIFT_CLEAR, DRIFT_DETECTED, FAILED. Events: RunSubmitted{query}, SearchStarted{},
  SearchCompleted{}, TraceRecorded{originalTrace}, ReplayStarted{},
  DriftScored{driftReport}, RunFailed{reason}. Commands: submit, markSearching,
  markSearchCompleted, recordTrace, markReplaying, recordReplayTrace, recordDrift, fail,
  getRun. emptyState() returns SearchRun.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer TraceRecordingConsumer subscribed to SearchRunEntity events; on SearchCompleted
  fetches the completed trace from the agent task result (passed as event payload or via the
  entity query), writes it back via SearchRunEntity.recordTrace(trace), then starts a
  ReplayWorkflow with id = "replay-" + runId.

- 1 View SearchRunView with row type SearchRunRow (mirrors SearchRun minus full call result
  bodies — the view holds summary counts for the UI; full call bodies are fetched via
  GET /api/runs/{id}). Table updater consumes SearchRunEntity events. ONE query getAllRuns:
  SELECT * AS runs FROM search_run_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SearchRunEndpoint at /api with POST /runs (body {queryText, topic, submittedBy};
    mints runId; calls SearchRunEntity.submit; returns {runId}), GET /runs (list from
    getAllRuns, sorted newest-first), GET /runs/{id} (one row), GET /runs/{id}/replay
    (trigger manual replay — only valid in TRACE_RECORDED state; starts new ReplayWorkflow
    instance with a versioned id), GET /runs/sse (Server-Sent Events forwarded from
    the view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SearchTasks.java declaring one Task<R> constant: SEARCH_AND_TRACE = Task.name("Search and
  trace").description("Issue web-search tool calls for the given query and return a
  SearchTrace of every call made").resultConformsTo(SearchTrace.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records SearchQuery, WebSearchCall, SearchResult, SearchTrace, DriftEntry,
  DriftReport, SearchRun, SearchRunStatus.

- WebSearchStub.java — a bundled web-search tool stub that implements the tool-call interface.
  Returns seeded result sets deterministically by query string. Four seed buckets map to the
  four seeded queries in src/main/resources/sample-events/seed-queries.jsonl. A fifth bucket
  returns a modified result set (fewer results, different titles) used by J3 to exercise
  the DRIFT_DETECTED path.

- ReplayDriftScorer.java — pure deterministic logic (no LLM). Inputs: two SearchTrace objects
  (original and replay). Outputs: DriftReport. Scoring: for each matching call pair (by
  callId), compute titleOverlapPct as the fraction of original titles present in replay
  results (0..100). Drift score = 100 − mean(titleOverlapPct across all calls), clamped 0..100.
  A run is DRIFT_DETECTED if drift score >= 30. Scoring rubric documented in Javadoc on
  the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9566 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-queries.jsonl with 4 seeded queries:
  "Latest AI regulatory updates EU 2026", "Transformer architecture benchmarks 2025",
  "Open source LLM licensing changes", "Agentic AI safety frameworks comparison".

- src/main/resources/mock-responses/ with stub response files keyed by query bucket.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root as specified in Section 5 of the SPEC.

- prompts/search-replay-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Server-Tool Web-Search Replay",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with query text, trace call list,
  replay trace call list, and drift report with per-query breakdown table).
  Browser title exactly: <title>Akka Sample: Server-Tool Web-Search Replay</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        SearchTrace outputs per query bucket.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. For SEARCH_AND_TRACE: reads
  src/main/resources/mock-responses/<bucket>.json by hashing the query string to one of four
  normal buckets (returning seeded traces) or the drift bucket (returning fewer results per
  call). The drift bucket is selected on every 4th run (modulo seed) so J3 is reproducible.
- A MockModelProvider.bucketFor(queryText) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SearchReplayAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SearchTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitTraceStep 20s, replayStep
  60s, scoreStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the SearchRun record is Optional<T>.
- Lesson 7: SearchTasks.java with SEARCH_AND_TRACE is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9566 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SearchReplayAgent).
  ReplayDriftScorer is deterministic and does NOT make an LLM call.
- The replay step re-issues the same queries via the SAME agent definition to ensure the
  scoring reflects provider-side changes, not agent-side changes.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
