# SPEC — llm-compiler-dag

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** LLMCompiler.
**One-line pitch:** Submit a query; a Planner compiles it into a dependency DAG of tool calls; a Task Fetching Unit dispatches independent calls in parallel; a Joiner synthesizes all results into one answer.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with explicit parallel fan-out. The Planner's output is a `CompilationPlan` — a list of `ToolCall` nodes each with a unique id, a tool type, its arguments, and a list of dependency ids. The Task Fetching Unit reads the DAG on each iteration, computes the current frontier (nodes whose dependencies are all resolved), and dispatches every frontier node concurrently as parallel workflow branches. It waits for all branches, records each `ToolResult`, and repeats until the DAG is exhausted or a blocking error occurs. The Joiner then receives the full `ResultSet` and produces a `QueryAnswer`.

The blueprint also demonstrates three governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each `ToolCall` node against an allow-list of tool types and argument constraints before the dispatch step fires,
- a **deployer runtime-monitoring** surface (operator dashboard + manual halt control) that pauses new frontier dispatches without killing in-flight parallel branches,
- a **secret sanitizer** that scrubs API-key-shaped and high-entropy strings from every `ToolResult` before they reach the Joiner's synthesis prompt and the SSE feed.

## 3. User-facing flows

The user opens the App UI tab and submits a query via the form.

1. The system creates a `Job` record in `COMPILING` and starts a `TaskFetchingUnit` workflow.
2. The Planner produces a `CompilationPlan { toolCalls: List<ToolCall>, description: String }` and emits `JobCompiled`.
3. The workflow enters the dispatch loop. Each iteration:
   - Compute frontier: `ToolCall` nodes whose `dependsOn` ids are all present in `resolvedIds`.
   - The **before-tool-call guardrail** vets every frontier node; rejected nodes emit `ToolCallBlocked` with a reason and are removed from the frontier (the DAG marks them `SKIPPED`).
   - Approved frontier nodes are dispatched in parallel. Each branch calls the appropriate simulated tool and returns a `ToolResult`.
   - The **secret sanitizer** scrubs every `ToolResult.output` in sequence.
   - Each scrubbed result is recorded via `JobEntity.recordResult`.
   - `resolvedIds` grows; the loop checks for a new frontier.
4. When the frontier is empty and no errors remain unresolved, the workflow calls the Joiner.
5. The Joiner reads the full `ResultSet` and produces a `QueryAnswer { summary, citations }`. The Job moves to `COMPLETED`.
6. If the guardrail or tool call fails exhausting the retry budget, the job moves to `FAILED`.
7. The operator can press **Halt new dispatches** at any time. The workflow finishes the current parallel batch, then ends with `JobHaltedOperator`. The Job moves to `HALTED`.

A `RequestSimulator` (TimedAction) drips a sample query every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Converts the user query into a `CompilationPlan` (DAG of `ToolCall` nodes). | `TaskFetchingUnit` | returns `CompilationPlan` to workflow |
| `JoinerAgent` | `AutonomousAgent` | Receives the `ResultSet` and synthesizes a `QueryAnswer`. | `TaskFetchingUnit` | returns `QueryAnswer` to workflow |
| `TaskFetchingUnit` | `Workflow` | Drives the compile → frontier-eval → parallel-dispatch → sanitize → record → frontier-re-eval loop; hands off to Joiner when DAG is exhausted. | `JobEndpoint`, `JobRequestConsumer` | `JobEntity` |
| `JobEntity` | `EventSourcedEntity` | Holds the job lifecycle, compiled DAG, accumulated results, and final answer. | `TaskFetchingUnit` | `JobView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `JobEndpoint` (operator action) | `TaskFetchingUnit` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted queries. | `JobEndpoint`, `RequestSimulator` | `JobRequestConsumer` |
| `JobView` | `View` | List-of-jobs read model for the UI. | `JobEntity` events | `JobEndpoint` |
| `JobRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `TaskFetchingUnit` per submission. | `RequestQueue` events | `TaskFetchingUnit` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/query-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StaleJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `RUNNING` past 5 minutes as `STALE`. | scheduler | `JobEntity` |
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE, operator halt/resume. | — | `JobView`, `RequestQueue`, `JobEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record ToolCall(
    String callId,
    ToolKind tool,
    Map<String, String> arguments,
    List<String> dependsOn
) {}

record CompilationPlan(
    String description,
    List<ToolCall> toolCalls
) {}

record ToolResult(
    String callId,
    ToolKind tool,
    boolean ok,
    String output,
    Optional<String> errorReason
) {}

record ResultSet(List<ToolResult> results) {}

record QueryAnswer(
    String summary,
    List<String> citations,
    Instant producedAt
) {}

record Job(
    String jobId,
    String query,
    JobStatus status,
    Optional<CompilationPlan> plan,
    Optional<ResultSet> results,
    Optional<QueryAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ToolKind { SEARCH, CALCULATOR, LOOKUP, CODE_EVAL }
enum CallStatus { PENDING, RUNNING, RESOLVED, SKIPPED, FAILED }
enum JobStatus { COMPILING, RUNNING, COMPLETED, FAILED, HALTED, STALE }
```

### Events (`JobEntity`)

`JobCreated`, `JobCompiled`, `ToolCallDispatched`, `ToolCallBlocked`, `ToolCallResolved`, `ToolCallFailed`, `JobCompleted`, `JobFailed`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`QuerySubmitted { jobId, query, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ query, requestedBy? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full plan + results + answer).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "LLMCompiler"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a query, operator halt/resume control, live list of jobs with status pills, expand-row to see the compilation plan (DAG nodes with dependency arrows), the resolved results per tool call, and the final answer.

Browser title: `<title>Akka Sample: LLMCompiler</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `TaskFetchingUnit`'s `guardStep`): every `ToolCall` node in the frontier is checked against (a) the tool allow-list (`SEARCH`, `CALCULATOR`, `LOOKUP`, `CODE_EVAL`), (b) per-tool argument constraints (SEARCH query length ≤ 500 chars; CALCULATOR expression must match safe math pattern; LOOKUP key must not reference system paths; CODE_EVAL code must not import `java.lang.Runtime` or `java.lang.ProcessBuilder`). Blocking. Rejected calls emit `ToolCallBlocked` and are marked `SKIPPED` in the DAG.
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows every in-flight job, its current frontier size, and the most recently resolved call. Two buttons — Halt new dispatches, Resume — drive `SystemControlEntity`. Every `TaskFetchingUnit` reads the flag before each new frontier dispatch; on `halted=true` it finishes the current parallel batch, then emits `JobHaltedOperator`.
- **S1 — secret sanitizer** (`sanitizer`, flavor `secret`): every `ToolResult.output` is scrubbed by a deterministic redactor matching AWS access keys, GitHub tokens, JWTs, common API-key patterns, and high-entropy strings of length ≥ 32 characters. The scrubbed output is what lands on the `JobEntity` and what the Joiner receives.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Parses the query; produces a `CompilationPlan`.
- `JoinerAgent` → `prompts/joiner.md`. Synthesizes a `QueryAnswer` from the `ResultSet`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What is the result of (42 * 17) + the current Akka version's minor number?" Job progresses `COMPILING → RUNNING → COMPLETED` within ~3 minutes. The compilation plan shows at least two `ToolCall` nodes with `CALCULATOR` and `SEARCH` types. The expanded view shows both resolved in parallel (overlapping `recordedAt` timestamps), and a non-empty `QueryAnswer`.
2. **J2** — Submit a query whose plan would emit a `CODE_EVAL` call importing `Runtime`. The guardrail blocks that call; the job either completes with remaining nodes or fails within budget.
3. **J3** — Submit any query and click **Halt new dispatches** while the job is `RUNNING`. The in-flight parallel batch finishes; no new frontier is dispatched; the job ends in `HALTED`.
4. **J4** — Submit a query that exercises the `LOOKUP` tool; a fixture response contains an `AKIA...` key shape. The `ToolResult.output` on the job record shows the key replaced by `[REDACTED:aws-access-key]`; the Joiner's prompt never contains the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named llm-compiler-dag demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-llm-compiler-dag.
Java package io.akka.samples.llmcompiler. Akka 3.6.0. HTTP port 9954.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PlannerAgent — definition() with capability:
      capability(TaskAcceptance.of(COMPILE).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. COMPILE returns CompilationPlan.
  * JoinerAgent — definition() with capability:
      capability(TaskAcceptance.of(JOIN).maxIterationsPerTask(2)).
    System prompt from prompts/joiner.md. JOIN returns QueryAnswer.

- 1 Workflow TaskFetchingUnit with steps:
  compileStep -> checkHaltStep -> [loop entry] guardStep ->
  parallelDispatchStep -> sanitizeStep -> recordStep ->
  frontierStep -> [back to checkHaltStep or to joinStep /
  failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    compileStep ofSeconds(60), guardStep ofSeconds(30),
    parallelDispatchStep ofSeconds(120) (covers all parallel tool calls),
    joinStep ofSeconds(60). defaultStepRecovery(maxRetries(2)
    .failoverTo(TaskFetchingUnit::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions
  to haltedStep (emits JobHaltedOperator on JobEntity).
  guardStep vets each frontier ToolCall via DispatchGuardrail.vet; rejected
  calls record ToolCallBlocked on JobEntity and are excluded from the
  frontier passed to parallelDispatchStep. If all frontier nodes are
  rejected, transitions to failStep.
  parallelDispatchStep fans out one concurrent Akka workflow branch per
  approved frontier node; each branch calls the matching simulated tool
  and returns a ToolResult. All branches must complete before sanitizeStep.
  sanitizeStep applies SecretScrubber.scrub to each ToolResult.output.
  recordStep calls JobEntity.recordResult(toolResult) for each result.
  frontierStep recomputes the frontier. If empty → joinStep; if error
  budget exhausted → failStep; otherwise → checkHaltStep.

- 1 EventSourcedEntity JobEntity holding Job state. emptyState() returns
  Job.initial("", null) with no commandContext() reference. Commands:
  createJob, recordPlan, recordToolCallBlocked, recordResult, recordFailure,
  completeJob, failJob, haltOperator, timeoutFail, getJob.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason,
  Optional<Instant> haltedAt}. Commands: requestHalt(reason), clearHalt,
  get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueQuery(jobId, query,
  requestedBy) emitting QuerySubmitted.

- 1 View JobView with row type JobRow (mirror of Job minus heavy result
  payloads — truncate to last 3 ToolResult entries plus counts; the UI
  fetches the full job by id on click). Table updater consumes JobEntity
  events. ONE query getAllJobs SELECT * AS jobs FROM job_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer JobRequestConsumer subscribed to RequestQueue events; on
  QuerySubmitted starts a TaskFetchingUnit with jobId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/query-prompts.jsonl and calls
    RequestQueue.enqueueQuery.
  * StaleJobMonitor — every 30s, queries JobView.getAllJobs, filters
    RUNNING jobs whose createdAt is older than 5 minutes, calls
    JobEntity.timeoutFail; TaskFetchingUnit polls JobEntity.getJob in
    its frontierStep and exits when status == STALE.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring Task<R> constant: COMPILE
  (resultConformsTo CompilationPlan).
- JoinerTasks.java declaring Task<R> constant: JOIN
  (resultConformsTo QueryAnswer).
- Domain records as listed in SPEC §5.
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36},
  JWT ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+,
  sk-[A-Za-z0-9]{32,}, Bearer [A-Za-z0-9._-]{20,}, and high-entropy
  fallback for tokens ≥ 32 chars whose Shannon entropy > 4.5 bits/char.
  Replacements: [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/DispatchGuardrail.java — deterministic vetter. Reject if
  the tool is not SEARCH/CALCULATOR/LOOKUP/CODE_EVAL; if SEARCH query
  length > 500 chars; if CALCULATOR expression fails the safe-math regex
  (/^[\d\s\+\-\*\/\(\)\.]+$/); if LOOKUP key starts with '/' or '~'; if
  CODE_EVAL code contains "Runtime" or "ProcessBuilder".
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9954 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/query-prompts.jsonl with 8 canned
  query prompts spanning calculation, search, lookup, and code-eval needs.
- src/main/resources/sample-data/search-fixtures.jsonl — 10 canned
  search results (query pattern, title, excerpt).
- src/main/resources/sample-data/lookup-fixtures.jsonl — 8 key/value
  lookup pairs; ONE entry's value must contain the literal substring
  "AKIAIOSFODNN7EXAMPLE" for the J4 acceptance test.
- src/main/resources/sample-data/calculator-fixtures.jsonl — 6 expression/
  result pairs used by the simulated CALCULATOR tool.
- src/main/resources/sample-data/code-eval-fixtures.jsonl — 4 code
  snippet / output pairs used by the simulated CODE_EVAL tool; none
  reference Runtime or ProcessBuilder.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1, HO1, S1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/joiner.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: LLMCompiler",
  one-line pitch, prerequisites (integration form: None), generate-the-
  system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
  NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for DAG view and answer).
  Browser title exactly: <title>Akka Sample: LLMCompiler</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider =
        mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java
  dispatching on agent class name and Task<R> id.
  Per-agent mock-response shapes for THIS blueprint:
    planner.json — "COMPILE" list with 5-8 CompilationPlan entries.
      Each plan has 3-6 ToolCall nodes spanning SEARCH, CALCULATOR, LOOKUP,
      CODE_EVAL. At least one plan has two independent sibling nodes
      (no dependsOn on each other) to exercise parallel dispatch. At least
      one has a linear chain to exercise sequential ordering.
    joiner.json — "JOIN" list with 5 QueryAnswer entries. Summaries
      60-120 words; citations list citing callId and tool kind.
      ONE entry's citations must reference the LOOKUP result that triggered
      the sanitizer in J4.
- MockModelProvider.seedFor(jobId) for deterministic selection per job.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent — both agents extend AutonomousAgent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on
  compileStep, guardStep, parallelDispatchStep, joinStep.
- Lesson 6: Optional<T> for every nullable field.
- Lesson 7: companion PlannerTasks.java and JoinerTasks.java.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9954 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: index.html includes mermaid CSS overrides AND themeVariables.
- Lesson 25: API-key sourcing five-option flow; no key value on disk.
- Lesson 26: Tab switching by data-tab / data-panel attribute; no zombie
  panels.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
