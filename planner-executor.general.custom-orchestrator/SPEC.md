# SPEC — custom-orchestration-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Custom Orchestration Agent.
**One-line pitch:** Submit a task; a pluggable orchestration strategy decides at each step which tool the orchestrator calls next, whether to revisit prior results, and when the task is complete — all without rebuilding the service.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with a **swappable orchestration strategy**. The standard planner-executor loop has a fixed routing function: the Orchestrator plans once and then iterates with a static decide step. This blueprint separates the routing logic into a named `OrchestrationStrategy` object stored in `StrategyRegistry` (an EventSourcedEntity). At the start of a task the workflow loads the active strategy; at each loop tick a `RoutingDecision` is computed by passing the current `ExecutionContext` through the strategy's rule set. The result may be `CallTool(toolName, args)`, `Revisit(traceIndex)`, `Conclude(answer)`, or `Abort(reason)`.

The blueprint also demonstrates three governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that inspects every `CallTool` decision against the tool allow-list and a per-tool content policy before the call fires,
- a **deployer runtime-monitoring** surface (operator dashboard with halt/resume) that lets a human-on-the-loop stop new tool dispatches without killing the in-flight call,
- a **secret sanitizer** that scrubs API-key-shaped strings, JWTs, and high-entropy tokens from every tool result before it lands on the execution trace.

## 3. User-facing flows

The user opens the App UI tab and submits a task via the form.

1. The system creates a `Task` record in `INITIALIZING` and starts a `TaskWorkflow`.
2. The workflow loads the active `OrchestrationStrategy` from `StrategyRegistry` and emits `TaskInitialized`.
3. The `OrchestratorAgent` initializes an `ExecutionContext { goal, strategy, traceEntries: [] }` and emits `ContextInitialized`. Task moves to `EXECUTING`.
4. The workflow enters the routing loop. Each iteration:
   - The **OrchestratorAgent** reads the current `ExecutionContext` and the active `OrchestrationStrategy` and produces a `RoutingDecision`.
   - If `CallTool`: the **before-tool-call guardrail** vets the decision; on rejection the workflow appends a `TraceEntry { verdict: BLOCKED }` and loops back.
   - The **ToolDispatcherAgent** receives the `ToolCall` and routes it to the matching simulated handler. Returns a `ToolResult`.
   - The **secret sanitizer** scrubs the result content.
   - The workflow appends a `TraceEntry { toolName, args, scrubbedResult, verdict }` and loops.
   - If `Revisit`: the orchestrator re-examines a prior trace entry; no tool fires; the workflow appends a `RevisitEntry` and loops.
   - If `Conclude`: the orchestrator produces a `TaskAnswer` and the workflow emits `TaskCompleted`.
   - If `Abort`: the workflow emits `TaskFailed`.
5. The operator can click **Halt new dispatches** at any time. The in-flight tool call finishes; the workflow exits with `TaskHaltedOperator`. The Task moves to `HALTED`.
6. An operator can also swap the active strategy via `POST /api/strategies/activate/{name}` without restarting. Subsequent tasks load the new strategy; in-flight tasks run to completion on the strategy they started with.

A `RequestSimulator` (TimedAction) drips a sample task every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OrchestratorAgent` | `AutonomousAgent` | Holds `ExecutionContext`; produces `RoutingDecision` on each loop tick; produces `TaskAnswer` on conclude. | `TaskWorkflow` | returns typed result to workflow |
| `ToolDispatcherAgent` | `AutonomousAgent` | Receives a `ToolCall` and routes it to the matching fixture-backed handler (search, file, code, command). Returns `ToolResult`. | `TaskWorkflow` | — |
| `TaskWorkflow` | `Workflow` | Drives initialize → [loop: route → guardrail → dispatch → sanitize → record → evaluate] → conclude / fail / halt branches. | `TaskEndpoint`, `TaskRequestConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds task lifecycle, `ExecutionContext`, execution trace, and final answer. | `TaskWorkflow` | `TaskView` |
| `StrategyRegistry` | `EventSourcedEntity` | Holds named `OrchestrationStrategy` configs; single instance keyed by `"registry"`. Supports register, activate, list, get. | `TaskEndpoint` (operator action) | `TaskWorkflow` (read on task start) |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by `"global"`. | `TaskEndpoint` (operator action) | `TaskWorkflow` (polls per tick) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted tasks. | `TaskEndpoint`, `RequestSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model for the UI. | `TaskEntity` events | `TaskEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `TaskWorkflow` per submission. | `RequestQueue` events | `TaskWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads the next line from `sample-events/task-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, marks any task stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `TaskEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*`, `/api/strategies/*`, `/api/control/*`, `/api/metadata/*`. | — | `TaskView`, `RequestQueue`, `TaskEntity`, `StrategyRegistry`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and redirects `/` → `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record TaskRequest(String prompt, String requestedBy) {}

record OrchestrationStrategy(
    String name,
    String description,
    int maxRouteIterations,
    int maxRevisits,
    List<String> allowedTools
) {}

record ToolCall(
    String toolName,
    String args,
    String rationale
) {}

record ToolResult(
    String toolName,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record RoutingDecision(
    RoutingAction action,
    Optional<ToolCall> toolCall,
    Optional<Integer> revisitIndex,
    Optional<String> concludeAnswer,
    Optional<String> abortReason
) {}

record TraceEntry(
    int sequence,
    TraceKind kind,
    Optional<ToolCall> toolCall,
    Optional<String> scrubbedResult,
    TraceVerdict verdict,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ExecutionContext(
    String goal,
    OrchestrationStrategy strategy,
    List<TraceEntry> traceEntries
) {}

record TaskAnswer(
    String summary,
    List<String> citations,
    Instant producedAt
) {}

record Task(
    String taskId,
    String prompt,
    TaskStatus status,
    Optional<OrchestrationStrategy> activeStrategy,
    Optional<ExecutionContext> context,
    Optional<TaskAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RoutingAction { CALL_TOOL, REVISIT, CONCLUDE, ABORT }
enum TraceKind { TOOL_CALL, REVISIT, BLOCKED }
enum TraceVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED }
enum TaskStatus { INITIALIZING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`TaskEntity`)

`TaskCreated`, `TaskInitialized`, `ContextInitialized`, `ToolDispatched`, `ToolBlocked`, `ToolRecorded`, `ContextRevisited`, `TaskCompleted`, `TaskFailed`, `TaskHaltedOperator`, `TaskFailedTimeout`.

### Events (`StrategyRegistry`)

`StrategyRegistered { name, strategy, registeredAt }`, `StrategyActivated { name, activatedAt }`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`TaskSubmitted { taskId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ prompt, requestedBy? }` → `202 { taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=...`.
- `GET /api/tasks/{id}` — one task (full context + answer).
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `POST /api/strategies` — body `OrchestrationStrategy` → `201`. Registers a named strategy.
- `POST /api/strategies/activate/{name}` — `200`. Makes a strategy active for new tasks.
- `GET /api/strategies` — list registered strategies with active flag.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets operator halt flag.
- `POST /api/control/resume` — `200`. Clears operator halt flag.
- `GET /api/control` — `{ halted, reason? }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Custom Orchestration Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, strategy selector (dropdown populated from `GET /api/strategies`), operator halt/resume control, live list of tasks with status pills, expand-row to see the execution context, trace timeline, and final answer.

Browser title: `<title>Akka Sample: Custom Orchestration Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `OrchestratorAgent`): every `CallTool` routing decision is checked against (a) the strategy's `allowedTools` list, (b) a per-tool content policy (no destructive commands, no file writes outside `/workspace/`, no fetches to non-allow-listed hosts). Blocking. Failure → `TraceEntry { verdict: BLOCKED_BY_GUARDRAIL }` + loop back to route step.
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows all in-flight tasks and the active strategy. Halt / Resume buttons drive `SystemControlEntity`. Every `TaskWorkflow` polls the halt flag before each route tick. On `halted=true`, the workflow finishes the in-flight tool call, then emits `TaskHaltedOperator`.
- **S1 — secret sanitizer** (`sanitizer`, flavor `secret`): every `ToolResult.content` is scrubbed by a deterministic redactor before the `TraceEntry` is written. The scrubbed text is what the orchestrator sees on the next routing tick.

## 9. Agent prompts

- `OrchestratorAgent` → `prompts/orchestrator.md`. Maintains `ExecutionContext`; produces `RoutingDecision` per tick and `TaskAnswer` on conclude.
- `ToolDispatcherAgent` → `prompts/tool-dispatcher.md`. Routes `ToolCall` to the correct fixture-backed handler.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Research the latest Akka SDK release and produce a short changelog summary." Task progresses `INITIALIZING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. Expanded view shows a non-empty execution trace (3–6 entries) and a `TaskAnswer` with citations.
2. **J2** — Activate a different strategy via `POST /api/strategies/activate/{name}`. Submit a new task. Expanded view shows the new strategy name in the `activeStrategy` field and routing decisions that differ from J1.
3. **J3** — Submit any task and click **Halt new dispatches** while status is `EXECUTING`. The in-flight tool call finishes; the next route tick reads the halt flag and exits with `TaskHaltedOperator`.
4. **J4** — Submit a task that exercises a file-read tool. A fixture file contains an `AKIA...` key shape. The trace entry shows the key replaced by `[REDACTED:aws-access-key]`; the orchestrator's next routing prompt never contains the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named custom-orchestration-agent demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-custom-orchestrator.
Java package io.akka.samples.customorchestrationagent. Akka 3.6.0. HTTP port 9180.

Components to wire (exactly):
- 2 AutonomousAgents:
  * OrchestratorAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(INITIALIZE_CONTEXT).maxIterationsPerTask(2))
      capability(TaskAcceptance.of(ROUTE).maxIterationsPerTask(1))
      capability(TaskAcceptance.of(CONCLUDE).maxIterationsPerTask(2)).
    System prompt from prompts/orchestrator.md.
    INITIALIZE_CONTEXT returns ExecutionContext.
    ROUTE returns RoutingDecision.
    CONCLUDE returns TaskAnswer.
  * ToolDispatcherAgent — capability(TaskAcceptance.of(DISPATCH_TOOL).maxIterationsPerTask(2)).
    Prompt from prompts/tool-dispatcher.md. Returns ToolResult.

- 1 Workflow TaskWorkflow with steps:
  initStep -> loadStrategyStep -> initContextStep -> [loop entry] checkHaltStep ->
  routeStep -> guardrailStep -> dispatchStep -> sanitizeStep -> recordStep ->
  evaluateStep -> [back to checkHaltStep, or to concludeStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    initStep ofSeconds(15), loadStrategyStep ofSeconds(10),
    initContextStep ofSeconds(45), routeStep ofSeconds(45),
    dispatchStep ofSeconds(90), concludeStep ofSeconds(60).
  defaultStepRecovery(maxRetries(2).failoverTo(TaskWorkflow::error)).
  loadStrategyStep reads StrategyRegistry.getActive; on absent uses built-in
  DefaultStrategy (maxRouteIterations=10, maxRevisits=2, allowedTools=all).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits TaskHaltedOperator on TaskEntity).
  guardrailStep runs ToolCallGuardrail.vet(RoutingDecision, activeStrategy);
  on reject records a TraceEntry{verdict=BLOCKED_BY_GUARDRAIL} via
  TaskEntity.recordBlock and loops back to routeStep.
  dispatchStep calls forAutonomousAgent(ToolDispatcherAgent.class, DISPATCH_TOOL)
  with the ToolCall; then calls TaskEntity.recordTool with taskId and entry.
  sanitizeStep applies SecretScrubber.scrub to the ToolResult.content before
  the entry is constructed.
  evaluateStep checks iteration budget: if routeCount >= strategy.maxRouteIterations,
  transitions to failStep(reason="iteration budget exhausted").
  concludeStep calls forAutonomousAgent(OrchestratorAgent.class, CONCLUDE) then
  emits TaskCompleted on TaskEntity.

- 1 EventSourcedEntity TaskEntity holding Task state. emptyState() returns
  Task.initial("", null). Commands: createTask, initializeTask, initializeContext,
  recordTool, recordBlock, recordRevisit, completeTask, failTask, haltOperator,
  timeoutFail, getTask. Events as listed in SPEC §5.

- 1 EventSourcedEntity StrategyRegistry keyed by literal "registry". State:
  StrategyRegistryState{Map<String,OrchestrationStrategy> strategies, String activeName}.
  Commands: registerStrategy(strategy), activateStrategy(name), getActive, listStrategies.
  Events: StrategyRegistered, StrategyActivated.
  Pre-seeded at startup with two built-in strategies:
    "default" — maxRouteIterations=10, maxRevisits=2, allowedTools=["search","file","code","command"].
    "conservative" — maxRouteIterations=5, maxRevisits=1, allowedTools=["search","file"].
  "default" is the active strategy at first boot.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State:
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant> haltedAt}.
  Commands: requestHalt(reason), clearHalt, get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueTask(taskId, prompt,
  requestedBy) emitting TaskSubmitted.

- 1 View TaskView with row type TaskRow (mirror of Task minus heavy trace
  payload — truncate to last 3 trace entries plus counts; UI fetches full
  task by id on click). Table updater consumes TaskEntity events.
  ONE query getAllTasks SELECT * AS tasks FROM task_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer TaskRequestConsumer subscribed to RequestQueue events; on
  TaskSubmitted starts a TaskWorkflow with taskId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    RequestQueue.enqueueTask.
  * StuckTaskMonitor — every 30s, queries TaskView.getAllTasks, filters
    EXECUTING tasks whose createdAt is older than 5 minutes, calls
    TaskEntity.timeoutFail; TaskWorkflow polls TaskEntity.getTask in its
    evaluateStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, POST /strategies, POST /strategies/activate/{name},
    GET /strategies, POST /control/halt, POST /control/resume, GET /control,
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- OrchestratorTasks.java declaring three Task<R> constants:
    INITIALIZE_CONTEXT (resultConformsTo ExecutionContext),
    ROUTE (resultConformsTo RoutingDecision),
    CONCLUDE (resultConformsTo TaskAnswer).
- DispatcherTasks.java declaring one Task<R> constant:
    DISPATCH_TOOL (resultConformsTo ToolResult).
- Domain records as listed in SPEC §5.
- application/ToolCallGuardrail.java — deterministic vetter. Reject if
  the toolName is not in strategy.allowedTools, if a "command" tool call
  matches /^(rm|sudo|mkfs|dd|chmod 777|>>?\s*\/(etc|proc|sys|dev))/, if
  a "code" tool call references writes outside /workspace/, if a "search"
  tool call names a host not on the allow-list (akka.io, doc.akka.io, github.com).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9180 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned task
  prompts covering search, file, code, and command tool needs.
- src/main/resources/sample-data/search-fixtures.jsonl — 10 canned search
  results (host, path, title, excerpt). Used by ToolDispatcherAgent for
  "search" tool calls.
- src/main/resources/sample-data/files/* — 6 short text files used by
  ToolDispatcherAgent for "file" tool calls. Include one file whose content
  contains an AKIA-shaped key fragment for the J4 acceptance test.
- src/main/resources/sample-data/commands.jsonl — allow-listed command set
  with canned output (ls, pwd, cat <fixture path>, echo …, head, wc -l).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of project-root files for the metadata endpoint).
- eval-matrix.yaml at the project root with 3 controls (G1, HO1, S1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/tool-dispatcher.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Custom Orchestration
  Agent", one-line pitch, prerequisites (host-software requirement: None),
  generate-the-system, what-you-get, customise-before-generating, what-gets-
  validated, license. NO Configuration section. NO governance-mechanisms
  section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form +
  strategy dropdown + operator halt/resume + live list with status pills and
  expand-on-click for context, trace, and answer). Browser title exactly:
  <title>Akka Sample: Custom Orchestration Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class name
  and Task<R> id. Each branch reads from src/main/resources/mock-responses/:
    orchestrator.json — three lists keyed by task id:
      "INITIALIZE_CONTEXT" → 4 ExecutionContext entries (goal filled, empty trace).
      "ROUTE" → 8 RoutingDecision entries spanning CallTool (search/file/code/command),
        Revisit, and Conclude, in a plausible narrative order.
      "CONCLUDE" → 4 TaskAnswer entries (50–100 word summaries, 2–4 citations).
    tool-dispatcher.json — 8 ToolResult entries, ok=true; ONE entry's content
      includes literal "AKIAIOSFODNN7EXAMPLE" for the J4 sanitizer test.
- MockModelProvider.seedFor(taskId) makes selection deterministic per task id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on View row records and entity state.
- Lesson 7: AutonomousAgent requires companion OrchestratorTasks.java and
  DispatcherTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current lineup.
  Defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9180 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  themeVariables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
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
