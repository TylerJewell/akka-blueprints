# SPEC — graph-pattern

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Graph Pattern.
**One-line pitch:** A user submits a task description; one `GraphAgent` walks it through four task phases — **PARSE** the request into a structured form, **PLAN** an explicit DAG of processing nodes, **EXECUTE** each node in topological order gated on predecessor outputs, and **MERGE** the node outputs into a typed result — with each phase gated on the prior phase's recorded output and each phase's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern applied to a DAG-structured workload. One `GraphAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency across four phases**: the PARSE task's typed output becomes the PLAN task's instruction context; the PLAN task's typed output becomes the EXECUTE task's context; the EXECUTE task's typed output becomes the MERGE task's context. The agent never holds all four phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

The blueprint specifically demonstrates the **graph primitive**: within the EXECUTE phase, the agent processes `GraphNode` entries in topological order, where each node declares its predecessors, and the `DependencyGuardrail` enforces that a node's execution tool is only callable after all predecessor nodes have recorded their output to the `GraphRunEntity`.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`PARSE` / `PLAN` / `EXECUTE` / `MERGE`) and the current `GraphRunEntity` status. An EXECUTE-phase tool called while the entity has not yet recorded `GraphPlanned` is rejected before the tool body runs. Within the EXECUTE phase, the guardrail additionally checks that all predecessor nodes of the target node have recorded `NodeExecuted` events on the entity before allowing the node-execution tool to proceed. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget.
- An **`on-decision-eval`** runs immediately after `OutputsMerged` lands, as `evalStep` inside the workflow. A deterministic, rule-based `CoverageScorer` (no LLM call — the eval is rule-based on purpose) checks that every planned `GraphNode` has a recorded `NodeExecuted` event (node coverage), that every node output referenced by the merge result exists in the entity's recorded outputs (output traceability), that no node output in the merge result references a node absent from the `GraphPlan` (no phantom nodes), and that the topological ordering of node execution events matches the declared predecessor edges (ordering proof).

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **task description** into the input (or picks one of three seeded tasks — `Summarize recent Akka release notes`, `Classify incoming support tickets by severity`, `Generate a changelog from merged pull requests`).
2. The user clicks **Run graph**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
4. Within ~10–20 s the card reaches `PLANNED` — the typed `GraphPlan` is visible in the card detail (a DAG table showing nodes, labels, and declared predecessors). The agent's PARSE task returned; the workflow recorded `RequestParsed` and ran the PLAN task.
5. Within ~10–20 s more the card reaches `EXECUTING`. The agent processes nodes one by one in topological order; the right pane shows a live node-execution list with status chips per node.
6. Within ~10–20 s more the card reaches `MERGED`, then immediately `EVALUATED`. The right pane now shows the full typed `TaskResult` — title, summary, per-node `Output` entries with body and source references — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GraphEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `GraphRunEntity`, `GraphRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `GraphRunEntity` | `EventSourcedEntity` | Per-run lifecycle: created → parsing → parsed → planning → planned → executing → executed → merging → merged → evaluated. Source of truth. | `GraphEndpoint`, `GraphExecutionWorkflow` | `GraphRunView` |
| `GraphExecutionWorkflow` | `Workflow` | One workflow per run. Steps: `parseStep` → `planStep` → `executeStep` → `mergeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `GraphEndpoint` after `CREATED` | `GraphAgent`, `GraphRunEntity` |
| `GraphAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `GraphTasks.java`: `PARSE_REQUEST` → `ParsedRequest`, `PLAN_GRAPH` → `GraphPlan`, `EXECUTE_NODES` → `ExecutionResult`, `MERGE_OUTPUTS` → `TaskResult`. Each task is registered with the phase-appropriate function tools. | invoked by `GraphExecutionWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `extractIntent(description)` and `identifyConstraints(description)`. Reads from `src/main/resources/sample-data/tasks/*.json` for deterministic offline output. | called from PARSE task | returns `ParsedRequest` |
| `PlanTools` | function-tools class | Implements `buildNodes(parsedRequest)` and `defineEdges(nodes)`. Pure in-memory DAG construction. | called from PLAN task | returns `List<GraphNode>` / `List<Edge>` |
| `ExecuteTools` | function-tools class | Implements `runNode(nodeId, inputs)` and `readPredecessorOutput(nodeId)`. | called from EXECUTE task | returns `NodeOutput` |
| `MergeTools` | function-tools class | Implements `aggregateOutputs(outputs)` and `formatResult(aggregated)`. | called from MERGE task | returns `TaskResult` |
| `DependencyGuardrail` | `before-tool-call` guardrail (registered on `GraphAgent`) | Reads the in-flight task's declared phase and the current `GraphRunEntity` status. Within EXECUTE phase, additionally checks that all predecessor nodes of the target node have `NodeExecuted` events on the entity. Rejects any tool call whose phase or node-dependency precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `CoverageScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `TaskResult`, `ExecutionResult`, `GraphPlan`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `GraphRunView` | `View` | Read model: one row per run for the UI. | `GraphRunEntity` events | `GraphEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ParsedRequest(
    String intent,
    List<String> constraints,
    Instant parsedAt
) {}

record GraphNode(
    String nodeId,
    String label,
    List<String> predecessorIds   // empty for root nodes
) {}

record Edge(String fromNodeId, String toNodeId) {}

record GraphPlan(
    List<GraphNode> nodes,
    List<Edge> edges,
    Instant plannedAt
) {}

record NodeInput(String nodeId, List<String> predecessorOutputIds) {}

record NodeOutput(
    String nodeId,
    String outputId,
    String body,
    List<String> sourceNodeIds    // must equal predecessor nodeIds that fed this node
) {}

record ExecutionResult(
    List<NodeOutput> outputs,
    List<String> executionOrder,  // nodeIds in the order they were executed
    Instant executedAt
) {}

record OutputRef(String nodeId, String outputId) {}

record TaskResult(
    String title,
    String summary,
    List<OutputRef> outputRefs,
    Instant mergedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record RunRecord(
    String runId,
    Optional<String> description,
    Optional<ParsedRequest> parsedRequest,
    Optional<GraphPlan> plan,
    Optional<ExecutionResult> executionResult,
    Optional<TaskResult> taskResult,
    Optional<EvalResult> eval,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    CREATED, PARSING, PARSED, PLANNING, PLANNED, EXECUTING, EXECUTED,
    MERGING, MERGED, EVALUATED, FAILED
}
```

Events on `GraphRunEntity`: `RunCreated`, `ParseStarted`, `RequestParsed`, `PlanStarted`, `GraphPlanned`, `ExecuteStarted`, `NodeExecuted`, `AllNodesExecuted`, `MergeStarted`, `OutputsMerged`, `EvaluationScored`, `DependencyViolated`, `RunFailed`.

Every nullable lifecycle field on the `RunRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ description }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Graph Pattern</title>`.

The App UI tab is a two-column layout: a left rail with the live list of runs (status pill + description + age) and a right pane with the selected run's detail — description, parsed intent and constraints, DAG plan with node/edge table, execution log with per-node status chips, merged result sections, eval score chip, and a dependency-violation log strip if any guardrail rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (dependency-gate)**: `DependencyGuardrail` is registered on `GraphAgent` and runs before every tool call. For phase-level gating, it reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.PARSE`, `Phase.PLAN`, `Phase.EXECUTE`, `Phase.MERGE`) and the current `GraphRunEntity.status` for the run the task is bound to. The phase accept rule: `PARSE` tools require `status ∈ {CREATED, PARSING}`; `PLAN` tools require `status ∈ {PARSED, PLANNING}` AND `parsedRequest.isPresent()`; `EXECUTE` tools require `status ∈ {PLANNED, EXECUTING}` AND `plan.isPresent()`; `MERGE` tools require `status ∈ {EXECUTED, MERGING}` AND `executionResult.isPresent()`. Within the EXECUTE phase, `DependencyGuardrail` additionally checks that every `predecessorId` declared on the target node appears in the list of `NodeExecuted` events already recorded on the entity — if any predecessor has not yet been executed, the call is rejected with `dependency-violation: node <id> requires predecessors <ids> but <missing> not yet executed`. On reject, the guardrail returns a structured error to the agent loop and the workflow records a `DependencyViolated{phase, nodeId, missingPredecessors, reason}` event. The agent loop retries within its 5-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `OutputsMerged` lands, as `evalStep` inside the workflow. `CoverageScorer` is a deterministic rule-based scorer (no LLM call): every planned `GraphNode` must have a `NodeExecuted` event (node coverage), every `OutputRef` in the `TaskResult` must reference a `NodeOutput` present in the `ExecutionResult` (output traceability), every referenced `nodeId` in `OutputRef` must appear in `GraphPlan.nodes` (no phantom nodes), and the execution order recorded in `ExecutionResult.executionOrder` must be consistent with the declared edges — no node precedes a declared predecessor (ordering proof). Emits `EvaluationScored{score: 1–5, rationale}` on a one-point-per-rule basis.

## 9. Agent prompts

- `GraphAgent` → `prompts/graph-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded task `Summarize recent Akka release notes`; within 60 s the run reaches `EVALUATED` with a non-empty `ParsedRequest`, a `GraphPlan` with ≥ 2 nodes, all nodes executed, and an eval score chip.
2. **J2** — The agent's first iteration on an EXECUTE task calls `runNode` on a node before its declared predecessor's `NodeExecuted` event has been recorded (mock LLM path). `DependencyGuardrail` rejects the call; a `DependencyViolated` event lands on the entity; the agent retries in topological order; the run eventually completes correctly.
3. **J3** — A run whose mock-LLM trajectory produces a `TaskResult` citing a node absent from the recorded `GraphPlan` is scored 1 with a rationale naming the phantom node; the UI flags the card.
4. **J4** — The execution order recorded in `ExecutionResult.executionOrder` is verified against the declared edges: no node precedes any of its declared predecessors. The presence of a score of 5 confirms the ordering proof passed.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named graph-pattern demonstrating the sequential-pipeline x general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-graph-pattern. Java package io.akka.samples.graphpattern.
Akka 3.6.0. HTTP port 9145.

Components to wire (exactly):

- 1 AutonomousAgent GraphAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/graph-agent.md>) and four .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  5)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PARSE, PLAN, EXECUTE, and MERGE tool sets are ALL registered on the agent; phase and
  dependency gating is the job of DependencyGuardrail, NOT of conditional .tools(...) wiring.
  The before-tool-call guardrail (DependencyGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 5-iteration budget.

- 1 Workflow GraphExecutionWorkflow per runId with five steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(GraphAgent.class, "agent-" + runId).runSingleTask(
      TaskDef.instructions("Task: " + description + "\nPhase: PARSE\nUse the parse tools
      to extract the intent and constraints.")
        .metadata("runId", runId)
        .metadata("phase", "PARSE")
        .taskType(GraphTasks.PARSE_REQUEST)
    ). Reads result(PARSE_REQUEST) to get ParsedRequest. Writes
    GraphRunEntity.recordParsedRequest(parsedRequest). WorkflowSettings.stepTimeout 60s.
  * planStep — emits PlanStarted, then runSingleTask with TaskDef.instructions
    (formatPlanContext(parsedRequest, description)) and metadata.phase = "PLAN", taskType
    PLAN_GRAPH. Writes GraphRunEntity.recordPlan(plan). stepTimeout 60s.
  * executeStep — emits ExecuteStarted, then runSingleTask with TaskDef.instructions
    (formatExecuteContext(plan, parsedRequest)) and metadata.phase = "EXECUTE", taskType
    EXECUTE_NODES. Writes GraphRunEntity.recordExecutionResult(executionResult). stepTimeout
    120s (node execution may be larger than a single agent call).
  * mergeStep — emits MergeStarted, then runSingleTask with TaskDef.instructions
    (formatMergeContext(executionResult, plan)) and metadata.phase = "MERGE", taskType
    MERGE_OUTPUTS. Writes GraphRunEntity.recordTaskResult(taskResult). stepTimeout 60s.
  * evalStep — runs the deterministic CoverageScorer over (taskResult, executionResult, plan)
    and writes GraphRunEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(GraphExecutionWorkflow::error). The error step writes
  RunFailed and ends.

- 1 EventSourcedEntity GraphRunEntity (one per runId). State RunRecord{runId,
  description: Optional<String>, parsedRequest: Optional<ParsedRequest>,
  plan: Optional<GraphPlan>, executionResult: Optional<ExecutionResult>,
  taskResult: Optional<TaskResult>, eval: Optional<EvalResult>, status: RunStatus,
  createdAt: Instant, finishedAt: Optional<Instant>, dependencyViolations:
  List<DependencyViolation>}. RunStatus enum: CREATED, PARSING, PARSED, PLANNING, PLANNED,
  EXECUTING, EXECUTED, MERGING, MERGED, EVALUATED, FAILED. Events: RunCreated{description},
  ParseStarted, RequestParsed{parsedRequest}, PlanStarted, GraphPlanned{plan},
  ExecuteStarted, NodeExecuted{nodeId, output}, AllNodesExecuted{executionResult},
  MergeStarted, OutputsMerged{taskResult}, EvaluationScored{eval},
  DependencyViolated{phase, nodeId, missingPredecessors, reason}, RunFailed{reason}.
  Commands: create, startParse, recordParsedRequest, startPlan, recordPlan, startExecute,
  recordNodeExecuted, recordAllNodesExecuted, startMerge, recordTaskResult,
  recordEvaluation, recordDependencyViolation, fail, getRun.
  emptyState() returns RunRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View GraphRunView with row type RunRow that mirrors RunRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes GraphRunEntity events. ONE query
  getAllRuns: SELECT * AS runs FROM graph_run_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * GraphEndpoint at /api with POST /runs (body {description}; mints runId; calls
    GraphRunEntity.create(description); then starts GraphExecutionWorkflow with id
    "graph-" + runId; returns {runId}), GET /runs (list from getAllRuns, sorted newest-first),
    GET /runs/{id} (one row), GET /runs/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- GraphTasks.java declaring four Task<R> constants:
    PARSE_REQUEST = Task.name("Parse request").description("Extract intent and constraints
      from the task description").resultConformsTo(ParsedRequest.class);
    PLAN_GRAPH = Task.name("Plan graph").description("Build a DAG of processing nodes with
      explicit predecessor edges").resultConformsTo(GraphPlan.class);
    EXECUTE_NODES = Task.name("Execute nodes").description("Run each node in topological
      order, gated on predecessor outputs").resultConformsTo(ExecutionResult.class);
    MERGE_OUTPUTS = Task.name("Merge outputs").description("Aggregate node outputs into a
      unified TaskResult").resultConformsTo(TaskResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {PARSE, PLAN, EXECUTE, MERGE}. Each function-tool method is annotated
  with the constant phase, e.g. @FunctionTool(name = "extractIntent", phase = Phase.PARSE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- ParseTools.java — @FunctionTool extractIntent(String description) -> String reading from
  src/main/resources/sample-data/tasks/*.json keyed by description slug; @FunctionTool
  identifyConstraints(String description) -> List<String> reading from the matching task
  entry's constraints list.

- PlanTools.java — @FunctionTool buildNodes(String intent, List<String> constraints) ->
  List<GraphNode> (deterministic: 2–4 nodes per task); @FunctionTool
  defineEdges(List<GraphNode> nodes) -> List<Edge> (deterministic: linear chain for
  simple tasks, fan-out for tasks with parallelisable subtasks; predecessor lists derived
  from the chain).

- ExecuteTools.java — @FunctionTool runNode(String nodeId, List<String> inputNodeIds) ->
  NodeOutput (reads the sample-data file; body composed from predecessor node outputs and
  the node label); @FunctionTool readPredecessorOutput(String nodeId) -> String (reads
  the NodeOutput for a node already recorded on the entity).

- MergeTools.java — @FunctionTool aggregateOutputs(List<String> nodeIds) ->
  List<OutputRef> (one OutputRef per executed node); @FunctionTool
  formatResult(List<OutputRef> refs, String intent) -> TaskResult (title from intent,
  summary composed from node bodies, outputRefs from the aggregated list).

- DependencyGuardrail.java — implements the before-tool-call hook. Performs two checks:
  (1) Phase check — reads the candidate tool's @FunctionTool.phase attribute, looks up the
  GraphRunEntity status by runId (carried in TaskDef metadata), applies the phase accept
  matrix from Section 8, rejects on mismatch. (2) Within EXECUTE phase — for runNode and
  readPredecessorOutput calls, reads the target nodeId from the tool arguments, fetches the
  node's predecessorIds from the GraphPlan recorded on the entity, checks that each
  predecessorId appears in the set of NodeExecuted events already on the entity, and rejects
  with dependency-violation if any predecessor is missing. On reject ALSO calls
  GraphRunEntity.recordDependencyViolation(phase, nodeId, missingPredecessors, reason).

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: TaskResult, 
  ExecutionResult, GraphPlan. Outputs: EvalResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1:
  (1) node coverage — every GraphPlan.nodes[i].nodeId appears in
      ExecutionResult.executionOrder,
  (2) output traceability — every TaskResult.outputRefs[j].outputId matches a
      NodeOutput.outputId in ExecutionResult.outputs,
  (3) no phantom nodes — every TaskResult.outputRefs[j].nodeId appears in
      GraphPlan.nodes[].nodeId,
  (4) ordering proof — for every pair (A, B) where A is a declared predecessor of B,
      A appears before B in ExecutionResult.executionOrder.
  Score range 1–5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9145 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/tasks.jsonl with 5 seeded task description lines
  covering the three surfaces named in J1–J5 plus two extras.

- src/main/resources/sample-data/tasks/*.json — three files keyed by seeded task slug, each
  carrying a ParsedRequest, a deterministic 2–4 node GraphPlan with edges, and per-node
  NodeOutput entries so ExecuteTools.runNode returns the same output across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 0 controls (this is a baseline with
  controls: []) but the schema_version and simplified_view blocks still present.
  Wait — the corpus entry declares controls: [], so generate an eval-matrix.yaml with an
  empty controls list and an empty simplified_view. The workflow still wires G1 and E1
  as documented in Section 8; eval-matrix.yaml reflects the deployer-declared controls
  for compliance reporting purposes and is separate from the runtime wiring.

- risk-survey.yaml at the project root with data.data_classes.pii = false (task descriptions
  are content-level, not person-level), decisions.authority_level = recommend-only (the
  TaskResult is advisory), oversight.human_in_loop = true (a human reviews the result
  before acting), operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "phantom-node-reference", "missing-node-coverage",
  "dependency-violation", "out-of-order-execution"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/graph-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Graph Pattern", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with description header, parsed intent
  and constraints, DAG plan table with node/edge rows, execution log with per-node status
  chips, merged result, eval-score chip, dependency-violation log strip). Browser title
  exactly: <title>Akka Sample: Graph Pattern</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    parse-request.json — 5 ParsedRequest entries with deterministic intent and 2–3
      constraints per entry. Each entry's tool_calls array contains 2 calls:
      1 extractIntent(description) + 1 identifyConstraints(description). Plus 1
      deliberately PHASE-VIOLATING entry whose tool_calls array starts with a
      runNode(...) call (an EXECUTE-phase tool called during PARSE) — the guardrail
      rejects it, the mock then falls through to a normal parse sequence. The mock
      should select the violating entry on the FIRST iteration of every 3rd run.
    plan-graph.json — 5 GraphPlan entries paired with the parse entries, each with 2–4
      GraphNode items and matching Edge items. tool_calls contain buildNodes + defineEdges.
    execute-nodes.json — 5 ExecutionResult entries. Each carries per-node NodeOutput
      entries in topological order, with tool_calls containing readPredecessorOutput +
      runNode pairs in order. Plus 1 deliberately OUT-OF-ORDER entry whose tool_calls
      array attempts runNode on a non-root node before its predecessor's runNode —
      DependencyGuardrail rejects it; J2 verifies this.
    merge-outputs.json — 5 TaskResult entries paired with the execute entries. Each
      carries OutputRef lists matching the paired ExecutionResult nodeIds. tool_calls
      contain aggregateOutputs + formatResult. Plus 1 deliberately PHANTOM-NODE entry
      whose OutputRefs reference a nodeId absent from the GraphPlan — evalStep scores
      it 1; J3 verifies this.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. GraphAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion GraphTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, planStep 60s, executeStep 120s, mergeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the RunRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: GraphTasks.java with PARSE_REQUEST, PLAN_GRAPH, EXECUTE_NODES, MERGE_OUTPUTS
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9145 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (GraphAgent). The
  on-decision eval is rule-based (CoverageScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (DependencyGuardrail) is the runtime mechanism that
  enforces both phase order and within-EXECUTE node-dependency order. Do NOT conditionally
  register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: parseStep writes ParsedRequest onto the
  entity, planStep reads it and builds the PLAN task's instruction context from it,
  executeStep reads both, mergeStep reads all three. The agent itself is stateless across
  phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
- Within executeStep, each NodeExecuted event is written to the entity INDIVIDUALLY after
  the matching runNode tool call returns, so that DependencyGuardrail can read the
  partial execution state and enforce predecessor checks for subsequent nodes. The
  AllNodesExecuted event is written once all NodeExecuted events are committed.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
