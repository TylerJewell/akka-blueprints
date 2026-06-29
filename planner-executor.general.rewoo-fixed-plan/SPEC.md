# SPEC — rewoo-fixed-plan

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** ReWOO (Reasoning Without Observation).
**One-line pitch:** Submit a query; a Planner writes the full reasoning plan with `#E<n>` variable placeholders up front; a Worker executes each tool call in order and fills the variables; a Solver reads the completed plan and composes the final answer — all in one deterministic pass with no mid-plan re-reasoning.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern in its fixed-plan variant. Unlike loop-driven orchestrators that observe intermediate results and continuously revise, ReWOO separates reasoning from observation entirely:

1. **Planner** emits a `QueryPlan` — a flat list of `PlanStep` records. Each step declares: a `ToolKind`, an `inputExpression` (free text that may embed `#E1`, `#E2`, … references), and an empty `result` slot.
2. **Worker** runs the steps one by one in the listed order. For each step it resolves any `#En` references by substituting the results already filled in for earlier steps, then calls the appropriate tool simulation and fills the step's `result` slot.
3. **Solver** receives the fully-filled `QueryPlan` and writes a `SolvedAnswer`.

The one governance control embedded in this flow is a **before-tool-call guardrail** on the Worker. Even though the plan is fixed, each individual tool call is an external side-effect; the guardrail vetoes tool calls that reference disallowed targets, destructive operations, or input expressions that would exfiltrate data outside the permitted scope.

## 3. User-facing flows

The user opens the App UI tab and submits a query via the form.

1. The system creates a `Query` record in `PLANNING` and starts a `QueryWorkflow`.
2. The PlannerAgent drafts a `QueryPlan { steps: List<PlanStep> }` and emits `QueryPlanned`. The task moves to `EXECUTING`.
3. The workflow iterates through the plan steps in index order:
   - `WorkerAgent` resolves `#En` references in `step.inputExpression` using already-filled results, then calls the designated tool simulation and returns a `ToolResult`.
   - The **before-tool-call guardrail** (`ToolCallGuardrail`) vets the resolved input before the tool call fires. On rejection: `StepBlocked` is emitted; the workflow transitions the query to `FAILED` with a clear block reason.
   - On success: the result is scrubbed by `SecretScrubber`, written into `step.result`, and `StepCompleted` is emitted.
4. Once all steps are completed, `SolverAgent` reads the fully-filled plan and returns a `SolvedAnswer { summary, citations }`. The query moves to `COMPLETED`.
5. The operator can press **Halt new steps** at any time. The workflow finishes the in-flight tool call, then ends with `QueryHaltedOperator`.

A `QuerySimulator` (TimedAction) drips a sample query every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Writes the full `QueryPlan` in one shot given the user's query. | `QueryWorkflow` | returns `QueryPlan` to workflow |
| `WorkerAgent` | `AutonomousAgent` | Executes one `PlanStep` at a time; resolves `#En` references; returns a `ToolResult`. | `QueryWorkflow` | returns `ToolResult` to workflow |
| `SolverAgent` | `AutonomousAgent` | Reads the fully-filled `QueryPlan`; composes the `SolvedAnswer`. | `QueryWorkflow` | returns `SolvedAnswer` to workflow |
| `QueryWorkflow` | `Workflow` | Drives the plan → [step loop: guardrail → execute → sanitize → record] → solve sequence, plus halt and failure branches. | `QueryEndpoint`, `QueryRequestConsumer` | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Holds the query lifecycle, the `QueryPlan` (with results filled in progressively), and the `SolvedAnswer`. | `QueryWorkflow` | `QueryView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `QueryEndpoint` (operator action) | `QueryWorkflow` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted queries. | `QueryEndpoint`, `QuerySimulator` | `QueryRequestConsumer` |
| `QueryView` | `View` | List-of-queries read model for the UI. | `QueryEntity` events | `QueryEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `QueryWorkflow` per submission. | `RequestQueue` events | `QueryWorkflow` |
| `QuerySimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/query-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckQueryMonitor` | `TimedAction` | Every 30 s, marks any query stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `QueryEntity` |
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE, operator halt. | — | `QueryView`, `RequestQueue`, `QueryEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record PlanStep(
    int stepIndex,
    ToolKind tool,
    String inputExpression,
    Optional<String> result
) {}

record QueryPlan(List<PlanStep> steps) {}

record ToolResult(
    ToolKind tool,
    String resolvedInput,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record StepRecord(
    int stepIndex,
    ToolKind tool,
    String resolvedInput,
    StepVerdict verdict,
    String scrubbedResult,
    Optional<String> blockReason,
    Instant recordedAt
) {}

record SolvedAnswer(
    String summary,
    List<String> citations,
    Instant producedAt
) {}

record Query(
    String queryId,
    String query,
    QueryStatus status,
    Optional<QueryPlan> plan,
    List<StepRecord> stepRecords,
    Optional<SolvedAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ToolKind { WEB_SEARCH, FILE_READ, CALCULATE, CODE_EVAL }
enum StepVerdict { COMPLETED, BLOCKED_BY_GUARDRAIL, FAILED }
enum QueryStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`QueryEntity`)

`QueryCreated`, `QueryPlanned`, `StepStarted`, `StepBlocked`, `StepCompleted`, `QuerySolved`, `QueryFailed`, `QueryHaltedOperator`, `QueryFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`QuerySubmitted { queryId, query, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ query, requestedBy? }` → `202 { queryId }`. Starts a workflow.
- `GET /api/queries` — list all queries. Optional `?status=...`.
- `GET /api/queries/{id}` — one query (full plan with results + answer).
- `GET /api/queries/sse` — server-sent events stream of every query change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "ReWOO (Reasoning Without <span class='accent'>Observation</span>)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a query, operator halt/resume control, live list of queries with status pills, expand-row to see the plan steps (with filled variables and scrubbedResult), and the final solved answer.

Browser title: `<title>Akka Sample: ReWOO (Reasoning Without Observation)</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `WorkerAgent`): before each tool call, `ToolCallGuardrail.vet(PlanStep, resolvedInput)` checks (a) whether the `ToolKind` is in the permitted set, (b) whether the resolved input references disallowed hosts (for `WEB_SEARCH`), forbidden file paths (for `FILE_READ`), or destructive expressions (for `CODE_EVAL` and `CALCULATE`). On rejection the workflow emits `StepBlocked` and transitions the query to `FAILED` with the block reason. Blocking.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Writes the full `QueryPlan` in one pass.
- `WorkerAgent` → `prompts/worker.md`. Executes one step; resolves variable references.
- `SolverAgent` → `prompts/solver.md`. Reads the completed plan; produces the `SolvedAnswer`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What is the current Akka version and how does it compare to the release six months ago?" Query progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a `QueryPlan` with all `result` slots filled, every `#En` reference resolved correctly in the inputs of later steps, and a `SolvedAnswer` with a 60–120 word summary.
2. **J2** — Submit a query whose plan would invoke `CODE_EVAL` with an expression that touches a path outside `/sandbox/`. The guardrail blocks the step; the query fails with a clear `failureReason` that names the blocked expression. The tool call never runs.
3. **J3** — Submit a query and click **Halt new steps** while it is `EXECUTING`. The in-flight tool call finishes and its result is recorded; no further steps execute; the query ends in `HALTED`.
4. **J4** — Submit a query whose plan exercises `FILE_READ`; one fixture response contains an `AKIA...` key shape. The `StepRecord.scrubbedResult` shows the key replaced by `[REDACTED:aws-access-key]`; the Solver's input never contains the literal key; the final `SolvedAnswer` is also clean.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rewoo-fixed-plan demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-rewoo-fixed-plan.
Java package io.akka.samples.rewooreasoningwithoutobservation.
Akka 3.6.0. HTTP port 9735.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PlannerAgent — definition() with capability:
      capability(TaskAcceptance.of(WRITE_PLAN).maxIterationsPerTask(3)).
    System prompt from prompts/planner.md. WRITE_PLAN returns QueryPlan.
  * WorkerAgent — capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)).
    Prompt from prompts/worker.md. Returns ToolResult.
  * SolverAgent — capability(TaskAcceptance.of(COMPOSE_ANSWER).maxIterationsPerTask(2)).
    Prompt from prompts/solver.md. Returns SolvedAnswer.

- 1 Workflow QueryWorkflow with steps:
  planStep -> [loop over plan steps] checkHaltStep -> resolveStep ->
  guardrailStep -> executeStep -> sanitizeStep -> recordStep -> nextStepOrSolveStep
  -> solveStep -> completeStep | failStep | haltedStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), executeStep ofSeconds(90) (covers tool simulation + agent),
    solveStep ofSeconds(60), resolveStep ofSeconds(15), guardrailStep ofSeconds(10).
  defaultStepRecovery(maxRetries(2).failoverTo(QueryWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits QueryHaltedOperator on QueryEntity).
  resolveStep substitutes #En references in step.inputExpression using the
  scrubbedResult values from prior StepRecord entries.
  guardrailStep runs ToolCallGuardrail.vet(step, resolvedInput); on reject
  emits StepBlocked and transitions to failStep.
  executeStep dispatches to WorkerAgent via forAutonomousAgent(...).runSingleTask(...)
  with the resolved input and tool kind; returns ToolResult.
  sanitizeStep applies SecretScrubber.scrub to ToolResult.content.
  recordStep calls QueryEntity.recordStep(stepRecord) emitting StepCompleted.
  nextStepOrSolveStep checks whether all plan steps are filled; if so transitions
  to solveStep; otherwise loops back to checkHaltStep with stepIndex+1.
  solveStep calls forAutonomousAgent(SolverAgent.class, COMPOSE_ANSWER) with the
  fully-filled QueryPlan; returns SolvedAnswer.
  completeStep emits QuerySolved on QueryEntity.

- 1 EventSourcedEntity QueryEntity holding Query state. emptyState() returns
  Query.initial("", null). Commands: createQuery, recordPlan, recordStepStart,
  recordStepBlocked, recordStepComplete, solveQuery, failQuery, haltOperator,
  timeoutFail, getQuery. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueQuery(queryId, query,
  requestedBy) emitting QuerySubmitted.

- 1 View QueryView with row type QueryRow (mirror of Query minus heavy plan
  payload — truncate to last 3 StepRecord entries plus counts; the UI fetches
  the full query by id on click). Table updater consumes QueryEntity events.
  ONE query getAllQueries SELECT * AS queries FROM query_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer QueryRequestConsumer subscribed to RequestQueue events; on
  QuerySubmitted starts a QueryWorkflow with queryId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 90s, reads next line from
    src/main/resources/sample-events/query-prompts.jsonl and calls
    RequestQueue.enqueueQuery.
  * StuckQueryMonitor — every 30s, queries QueryView.getAllQueries, filters
    EXECUTING queries whose createdAt is older than 5 minutes, calls
    QueryEntity.timeoutFail.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries, GET /queries (filters client-side
    from getAllQueries), GET /queries/{id}, GET /queries/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring Task<R> constant: WRITE_PLAN
  (resultConformsTo QueryPlan).
- WorkerTasks.java declaring Task<R> constant: EXECUTE_STEP
  (resultConformsTo ToolResult).
- SolverTasks.java declaring Task<R> constant: COMPOSE_ANSWER
  (resultConformsTo SolvedAnswer).
- Domain records as listed in SPEC §5.
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/ToolCallGuardrail.java — deterministic vetter. Reject if the
  ToolKind is not one of WEB_SEARCH/FILE_READ/CALCULATE/CODE_EVAL, if a
  WEB_SEARCH resolved input names a host not in the allow-list
  (akka.io, doc.akka.io, github.com), if a FILE_READ resolved input
  references a path outside sample-data/files/, if a CODE_EVAL resolved
  input contains shell metacharacters (;, &&, ||, |, >, >>, $(), `),
  if a CODE_EVAL resolved input references any path outside /sandbox/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9735 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/query-prompts.jsonl with 8 canned query
  prompts exercising web search, file reading, calculation, and code
  evaluation needs.
- src/main/resources/sample-data/web-fixtures.jsonl — 12 canned web
  fixtures (host, path, title, excerpt). Used by WorkerAgent WEB_SEARCH.
- src/main/resources/sample-data/files/* — 6 short text files used by
  WorkerAgent FILE_READ. Include one file whose content contains an AKIA-shaped
  key fragment for the J4 acceptance test.
- src/main/resources/sample-data/calculations.jsonl — expression-to-result
  pairs for CALCULATE; WorkerAgent matches against these.
- src/main/resources/sample-data/code-snippets.jsonl — input→output pairs
  for CODE_EVAL (results already evaluated; WorkerAgent returns the canned
  output for allow-listed expressions).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint).
- eval-matrix.yaml at project root with 1 control (G1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root, pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance.capabilities;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/worker.md, prompts/solver.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: ReWOO (Reasoning
  Without Observation)", one-line pitch, prerequisites (integration form
  host-software: None), generate-the-system, what-you-get, customise-
  before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for plan steps and solved answer).
  Browser title exactly: <title>Akka Sample: ReWOO (Reasoning Without
  Observation)</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  and the Task<R> id.
- Per-agent mock-response shapes for THIS blueprint:
    planner.json — a list keyed "WRITE_PLAN": 6 QueryPlan entries, each with
      3–5 PlanStep records whose inputExpression fields use #E1, #E2 references
      progressively. Steps span WEB_SEARCH, FILE_READ, CALCULATE, CODE_EVAL.
    worker.json — a list keyed "EXECUTE_STEP": 12 ToolResult entries (3 per
      ToolKind). One FILE_READ entry's content must include the literal substring
      "AKIAIOSFODNN7EXAMPLE" for the J4 sanitizer test.
    solver.json — a list keyed "COMPOSE_ANSWER": 6 SolvedAnswer entries with
      60–120 word summaries and 3–5 citation bullets referencing step indices
      and tool kinds.
- A MockModelProvider.seedFor(queryId) helper makes selection deterministic
  per query id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  executeStep, solveStep, resolveStep, guardrailStep.
- Lesson 6: Optional<T> for every nullable field on QueryRow and on the
  Query entity state.
- Lesson 7: PlannerTasks.java, WorkerTasks.java, SolverTasks.java each
  declare their Task<R> constants.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9735 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in any file.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides
  AND themeVariables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching by data-tab / data-panel attribute only.
  Exactly five <section class="tab-panel"> elements in the DOM.
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
