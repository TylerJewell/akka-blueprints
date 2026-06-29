# SPEC — akka-bridge

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Functional API Plugin.
**One-line pitch:** A user submits a graph definition; one `GraphAgent` walks it through three task phases — **PLAN** an execution order, **EXECUTE** each node as a durable activity, **FINALIZE** a typed output — with each node's LLM response checked by an after-llm-response guardrail before it is forwarded downstream.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern applied to graph-node execution. One `GraphAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PLAN task's typed `ExecutionPlan` becomes the EXECUTE task's instruction context; the EXECUTE task's collected `NodeOutput` list becomes the FINALIZE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent's LLM response and the downstream node or workflow step. After each node execution, the guardrail inspects the raw LLM output for policy violations — prohibited content, hallucinated tool references, malformed output schema — before that output is written to `GraphRunEntity` and forwarded to the next node. A violated output is blocked; the rejection is recorded as a `GuardrailViolation` event for audit. The agent loop retries within its 4-iteration budget. The guardrail's accept rule is precise: output passes if and only if it conforms to the declared node schema AND contains none of the configured prohibited patterns. On block, the guardrail returns a structured `content-violation` error so the task loop can correct course.

The blueprint shows that wrapping third-party graph execution with Akka workflows is not just a durability upgrade — the task-boundary handoffs are the right cut to enforce both the output-content contract and the per-node isolation that prevents upstream mistakes from silently propagating through downstream nodes.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **graph definition** from the input (or picks one of three seeded graphs — `summarize-and-classify`, `translate-and-validate`, `extract-and-enrich`).
2. The user clicks **Run graph**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PLANNING` — the workflow has started `planStep` and the agent has been handed the PLAN task.
4. Within ~10–20 s the card reaches `EXECUTING` — the typed `ExecutionPlan` is visible in the card detail (a table of planned nodes with type and estimated cost). The agent's PLAN task returned; the workflow recorded `GraphPlanned` and started the EXECUTE task.
5. Within ~10–20 s more the card reaches `FINALIZING`. The node outputs are visible (a per-node table with node id, output summary, and guardrail status badge).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `FinalResult` — title, summary, per-node `ResultItem` with body and a `sources` list — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another graph; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GraphEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `GraphRunEntity`, `GraphRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `GraphRunEntity` | `EventSourcedEntity` | Per-run lifecycle: created → planning → planned → executing → executed → finalizing → finalized → evaluated. Source of truth. | `GraphEndpoint`, `GraphExecutionWorkflow` | `GraphRunView` |
| `GraphExecutionWorkflow` | `Workflow` | One workflow per run. Steps: `planStep` → `executeStep` → `finalizeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `GraphEndpoint` after `CREATED` | `GraphAgent`, `GraphRunEntity` |
| `GraphAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `GraphTasks.java`: `PLAN_GRAPH` → `ExecutionPlan`, `EXECUTE_NODE` → `NodeOutputSet`, `FINALIZE_OUTPUT` → `FinalResult`. Each task is registered with the phase-appropriate function tools. | invoked by `GraphExecutionWorkflow` | returns typed results |
| `PlanTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `parseGraphDefinition(input)` and `estimateNodeCost(nodeType)`. Reads from `src/main/resources/sample-data/graphs/*.json` for deterministic offline output. | called from PLAN task | returns `List<PlannedNode>` |
| `ExecuteTools` | function-tools class | Implements `invokeNode(nodeId, input)` and `checkNodeStatus(nodeId)`. Pure in-memory execution against sample fixtures. | called from EXECUTE task | returns `NodeOutput` / `NodeStatus` |
| `FinalizeTools` | function-tools class | Implements `aggregateOutputs(outputs)` and `formatResult(outputs)`. | called from FINALIZE task | returns `ResultItem` / `FinalResult` |
| `OutputGuardrail` | `after-llm-response` guardrail (registered on `GraphAgent`) | Inspects the agent's raw LLM response after each node execution for policy violations. Blocks any output that fails schema conformance or prohibited-content checks. | every LLM response on every task | accept / structured-block |
| `PolicyChecker` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `FinalResult`, `NodeOutputSet`, `ExecutionPlan`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `GraphRunView` | `View` | Read model: one row per run for the UI. | `GraphRunEntity` events | `GraphEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PlannedNode(String nodeId, String nodeType, String description, int estimatedCostTokens) {}

record ExecutionPlan(List<PlannedNode> nodes, String graphId, Instant plannedAt) {}

record NodeOutput(
    String nodeId,
    String rawOutput,
    String outputSummary,
    boolean guardrailPassed,
    Instant executedAt
) {}

record NodeOutputSet(List<NodeOutput> outputs, Instant collectedAt) {}

record NodeStatus(String nodeId, String status, String message) {}

record ResultItem(String nodeId, String heading, String body, List<String> sourceNodeIds) {}

record FinalResult(
    String title,
    String summary,
    List<ResultItem> items,
    Instant finalizedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record GuardrailViolation(
    String nodeId,
    String violationType,
    String reason,
    Instant blockedAt
) {}

record GraphRunRecord(
    String runId,
    Optional<String> graphId,
    Optional<ExecutionPlan> plan,
    Optional<NodeOutputSet> nodeOutputs,
    Optional<FinalResult> result,
    Optional<EvalResult> eval,
    GraphRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailViolation> violations
) {}

enum GraphRunStatus {
    CREATED, PLANNING, PLANNED, EXECUTING, EXECUTED,
    FINALIZING, FINALIZED, EVALUATED, FAILED
}
```

Events on `GraphRunEntity`: `GraphRunCreated`, `PlanStarted`, `GraphPlanned`, `ExecuteStarted`, `NodeExecuted`, `FinalizeStarted`, `OutputFinalized`, `EvaluationScored`, `GuardrailViolated`, `GraphRunFailed`.

Every nullable lifecycle field on the `GraphRunRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ graphId }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Functional API Plugin</title>`.

The App UI tab is a two-column layout: a left rail with the live list of runs (status pill + graph id + age) and a right pane with the selected run's detail — graph id, planned nodes table, per-node execution outputs, final result items, eval score chip, and a guardrail-violation log strip if any output violations occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `after-llm-response` guardrail (output policy gate)**: `OutputGuardrail` is registered on `GraphAgent` and runs after every node's LLM response, before that response is written to `GraphRunEntity` or forwarded to a downstream node. It inspects the raw output for: (1) schema conformance — the response must deserialise into the declared `NodeOutput` type; (2) prohibited-content patterns — a configurable list of regex patterns loaded from `src/main/resources/policy/prohibited-patterns.txt`; (3) hallucinated references — any `sourceNodeId` in a `ResultItem` must correspond to a `PlannedNode.nodeId` in the current `ExecutionPlan`. On block, the guardrail returns a structured `content-violation` error to the agent loop and the workflow records a `GuardrailViolated{nodeId, violationType, reason}` event for visibility. The agent loop retries within its 4-iteration budget. Accept rule: output passes all three checks simultaneously.

- **E1 — `on-decision-eval`**: runs immediately after `OutputFinalized` lands, as `evalStep` inside the workflow. `PolicyChecker` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every planned node must have a corresponding `NodeOutput` (node coverage), every output must have passed its guardrail check (output integrity), every `ResultItem.sourceNodeIds` entry must match a `PlannedNode.nodeId` (reference provenance), and the item count must equal the executed node count (output parity). Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis.

## 9. Agent prompts

- `GraphAgent` → `prompts/graph-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User selects the seeded graph `summarize-and-classify`; within 60 s the run reaches `EVALUATED` with a non-empty plan, ≥ 2 node outputs, and an eval score chip on the card.
2. **J2** — A node's LLM response contains a prohibited pattern (mock LLM path). `OutputGuardrail` blocks the output; a `GuardrailViolated` event lands on the entity; the agent retries and produces a clean response; the run eventually completes correctly. The UI's violation-log strip shows the one blocked output.
3. **J3** — A run whose mock-LLM trajectory produces a `ResultItem` referencing a `sourceNodeId` absent from the `ExecutionPlan` is scored 1 with a rationale naming the hallucinated reference; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-run trace (logged at `INFO`); the PLAN task's log shows only PLAN-tool calls, the EXECUTE task's log shows only EXECUTE-tool calls, the FINALIZE task's log shows only FINALIZE-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-bridge demonstrating the sequential-pipeline x general cell. Runs out
of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-akka-bridge. Java package io.akka.samples.functionalapiplugin.
Akka 3.6.0. HTTP port 9994.

Components to wire (exactly):

- 1 AutonomousAgent GraphAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/graph-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PLAN, EXECUTE, and FINALIZE tool sets are ALL registered on the agent; the after-llm-response
  guardrail (OutputGuardrail) intercepts responses, NOT tool registrations. The guardrail is
  registered on the agent via the agent's guardrail-configuration block. On guardrail block the
  agent loop retries within its 4-iteration budget.

- 1 Workflow GraphExecutionWorkflow per runId with four steps:
  * planStep — emits PlanStarted on the entity, then calls componentClient
    .forAutonomousAgent(GraphAgent.class, "agent-" + runId).runSingleTask(
      TaskDef.instructions("Graph: " + graphId + "\nPhase: PLAN\nUse the parsing and cost tools
      to produce an ordered ExecutionPlan for this graph.")
        .metadata("runId", runId)
        .metadata("phase", "PLAN")
        .taskType(GraphTasks.PLAN_GRAPH)
    ). Reads forTask(taskId).result(PLAN_GRAPH) to get ExecutionPlan. Writes
    GraphRunEntity.recordPlan(plan). WorkflowSettings.stepTimeout 60s.
  * executeStep — emits ExecuteStarted, then runSingleTask with TaskDef.instructions
    (formatExecuteContext(plan, graphId)) and metadata.phase = "EXECUTE", taskType
    EXECUTE_NODE. Writes GraphRunEntity.recordNodeOutputs(nodeOutputs). stepTimeout 60s.
  * finalizeStep — emits FinalizeStarted, then runSingleTask with TaskDef.instructions
    (formatFinalizeContext(nodeOutputs, plan, graphId)) and metadata.phase = "FINALIZE",
    taskType FINALIZE_OUTPUT. Writes GraphRunEntity.recordResult(result). stepTimeout 60s.
  * evalStep — runs the deterministic PolicyChecker over (result, nodeOutputs, plan)
    and writes GraphRunEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(GraphExecutionWorkflow::error). The error step writes
  GraphRunFailed and ends.

- 1 EventSourcedEntity GraphRunEntity (one per runId). State GraphRunRecord{runId,
  graphId: Optional<String>, plan: Optional<ExecutionPlan>, nodeOutputs:
  Optional<NodeOutputSet>, result: Optional<FinalResult>, eval: Optional<EvalResult>,
  status: GraphRunStatus, createdAt: Instant, finishedAt: Optional<Instant>,
  violations: List<GuardrailViolation>}. GraphRunStatus enum: CREATED, PLANNING, PLANNED,
  EXECUTING, EXECUTED, FINALIZING, FINALIZED, EVALUATED, FAILED. Events:
  GraphRunCreated{graphId}, PlanStarted, GraphPlanned{plan}, ExecuteStarted,
  NodeExecuted{nodeOutputs}, FinalizeStarted, OutputFinalized{result},
  EvaluationScored{eval}, GuardrailViolated{nodeId, violationType, reason, blockedAt},
  GraphRunFailed{reason}.
  Commands: create, startPlan, recordPlan, startExecute, recordNodeOutputs, startFinalize,
  recordResult, recordEvaluation, recordViolation, fail, getRun. emptyState()
  returns GraphRunRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View GraphRunView with row type GraphRunRow that mirrors GraphRunRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes GraphRunEntity events. ONE
  query getAllRuns: SELECT * AS runs FROM graph_run_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * GraphEndpoint at /api with POST /runs (body {graphId}; mints runId; calls
    GraphRunEntity.create(graphId); then starts GraphExecutionWorkflow with id
    "run-" + runId; returns {runId}), GET /runs (list from getAllRuns, sorted newest-first),
    GET /runs/{id} (one row), GET /runs/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- GraphTasks.java declaring three Task<R> constants:
    PLAN_GRAPH = Task.name("Plan graph").description("Parse the graph definition and produce
      an ordered ExecutionPlan with cost estimates").resultConformsTo(ExecutionPlan.class);
    EXECUTE_NODE = Task.name("Execute nodes").description("Invoke each planned node in order
      and collect typed NodeOutputs").resultConformsTo(NodeOutputSet.class);
    FINALIZE_OUTPUT = Task.name("Finalize output").description("Aggregate all NodeOutputs
      into a coherent FinalResult").resultConformsTo(FinalResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {PLAN, EXECUTE, FINALIZE}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "parseGraphDefinition", phase = Phase.PLAN)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- PlanTools.java — @FunctionTool parseGraphDefinition(String input) -> List<PlannedNode>
  reading from src/main/resources/sample-data/graphs/*.json keyed by graphId;
  @FunctionTool estimateNodeCost(String nodeType) -> int returning token estimates from a
  static lookup table.

- ExecuteTools.java — @FunctionTool invokeNode(String nodeId, String input) -> NodeOutput
  (executes the node against the in-process fixture for the graphId; output.guardrailPassed
  starts as false until OutputGuardrail accepts the response);
  @FunctionTool checkNodeStatus(String nodeId) -> NodeStatus returning status from the
  in-memory execution map.

- FinalizeTools.java — @FunctionTool aggregateOutputs(List<NodeOutput>) -> List<ResultItem>
  (one ResultItem per output, heading from the nodeId, body composed from outputSummary);
  @FunctionTool formatResult(List<ResultItem>) -> FinalResult (title from graphId, summary
  composed from all outputSummary values joined with linking phrases).

- OutputGuardrail.java — implements the after-llm-response hook. Receives the raw LLM
  response string and the current TaskDef metadata (runId, phase). Applies three checks:
  (1) schema conformance — attempts to deserialise into the task's declared result type;
  (2) prohibited-content scan — tests against patterns loaded from
  src/main/resources/policy/prohibited-patterns.txt;
  (3) reference provenance — for FINALIZE responses, every ResultItem.sourceNodeIds entry
  must match a nodeId in the current ExecutionPlan (read from GraphRunEntity by runId).
  On block ALSO calls GraphRunEntity.recordViolation(nodeId, violationType, reason) so the
  violation is visible in the UI's violation-log strip and in the audit log.

- PolicyChecker.java — pure deterministic logic (no LLM). Inputs: FinalResult,
  NodeOutputSet, ExecutionPlan. Outputs: EvalResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1: node coverage (every
  PlannedNode has a NodeOutput), output integrity (every NodeOutput.guardrailPassed is true),
  reference provenance (every ResultItem.sourceNodeIds entry appears in the ExecutionPlan's
  nodeIds), and output parity (items.size() == nodes.size()). Score range 1-5. Rationale
  names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9994 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/graphs.jsonl with 5 seeded graph-id lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/graphs/*.json — three files keyed by seeded graphId, each
  carrying 3-6 PlannedNode entries with deterministic content so PlanTools.parseGraphDefinition
  returns the same list across restarts.

- src/main/resources/policy/prohibited-patterns.txt — one prohibited-content regex per line
  (e.g., patterns for injected instructions, schema-breaking tokens, personally-identifying
  patterns) used by OutputGuardrail for the prohibited-content scan.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (graph inputs are
  user-defined graph ids, not person-level data), decisions.authority_level = recommend-only
  (the final result is advisory), oversight.human_in_loop = true (a human reviews the output
  before acting on it), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "prohibited-content-output",
  "hallucinated-node-reference", "incomplete-node-coverage", "schema-nonconformance";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/graph-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Functional API Plugin", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with graph id header, planned nodes
  table, per-node execution outputs, final result items, eval-score chip, violation-log strip).
  Browser title exactly: <title>Akka Sample: Functional API Plugin</title>. No subtitle on
  the Overview tab.

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
    plan-graph.json — 5 ExecutionPlan entries, each with 3-5 PlannedNode items covering
      the seeded graphIds. Each entry's tool_calls array contains 1-2 calls:
      parseGraphDefinition(graphId) + estimateNodeCost(nodeType) per node.
    execute-nodes.json — 5 NodeOutputSet entries paired one-to-one with the plan entries,
      each with 3-5 NodeOutput items matching the planned nodes. Plus 1 deliberately
      POLICY-VIOLATING entry whose first NodeOutput.rawOutput contains a prohibited-content
      pattern — the guardrail blocks it; the mock then falls through to a clean execute
      sequence. The mock should select the violating entry on the FIRST iteration of every
      3rd run (modulo seed) so J2 is reproducible.
    finalize-output.json — 5 FinalResult entries paired one-to-one. Each carries 3-5
      ResultItem items matching the paired NodeOutputs, with sourceNodeIds referencing the
      paired PlannedNode nodeIds. Plus 1 deliberately HALLUCINATED-REFERENCE entry whose
      first ResultItem.sourceNodeIds contains a nodeId absent from the paired ExecutionPlan
      — the workflow's evalStep scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. GraphAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion GraphTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (planStep
  60s, executeStep 60s, finalizeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the GraphRunRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: GraphTasks.java with PLAN_GRAPH, EXECUTE_NODE, FINALIZE_OUTPUT constants is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9994 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (GraphAgent). The
  on-decision eval is rule-based (PolicyChecker.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  after-llm-response guardrail (OutputGuardrail) is the runtime mechanism that checks every
  LLM response. Do NOT skip registering tool sets or conditionally suppress them.
- Task dependency is carried by typed task results: planStep writes ExecutionPlan onto the
  entity, executeStep reads it and builds the EXECUTE task's instruction context from it,
  finalizeStep reads both. The agent itself is stateless across phases.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
