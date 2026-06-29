# SPEC — akka-stategraph-bridge

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Graph API Plugin.
**One-line pitch:** Define a `StateGraph` of typed nodes and edges; the runtime executes it durably — following conditional branches and back-edges — while tracking every transition on an append-only execution trace and vetting every tool-bearing node before it runs.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to graph-structured execution. A `PlannerAgent` validates and normalises the user-supplied graph definition into a `GraphPlan` (node registry, edge list with routing predicates, declared entry and terminal nodes, cycle annotations). A `GraphWorkflow` then drives the execution loop: on each iteration, it dispatches the current node to `NodeAgent`, routes to the next node via `EdgeRouterAgent`, records the updated `GraphState`, and checks cycle budgets. Back-edges (cycles) are bounded by a per-node visit counter; a node visited more than its declared `maxVisits` cap causes the run to fail rather than loop forever.

The blueprint wires one governance mechanism into this loop:

- a **before-tool-call guardrail** that inspects every `NodeDispatch` that carries a tool annotation before the node executes — rejecting tool calls to hosts or functions outside an allow-list, recording the block on the execution trace, and asking the `PlannerAgent` to revise the node's tool binding.

Two supplementary mechanisms protect the wider system:

- a **deployer runtime-monitoring** surface (operator halt control) that drains in-flight node executions before stopping new dispatches.
- a **secret sanitizer** that scrubs API-key-shaped and high-entropy strings from every `NodeOutput.content` before the content lands in `GraphState` or the trace.

## 3. User-facing flows

The user opens the App UI tab and submits a graph definition via the form.

1. The system creates a `GraphRun` record in `PLANNING` and starts a `GraphWorkflow`.
2. The `PlannerAgent` validates the definition and emits `RunPlanned` with a `GraphPlan { nodes, edges, entryNodeId, terminalNodeIds, cycleAnnotations }`.
3. The workflow enters the execution loop. Each iteration:
   - The workflow resolves the current node from `GraphPlan.nodes`.
   - If the node's `toolAnnotation` is present, the **before-tool-call guardrail** vets it. On rejection, a `NodeBlocked` entry is added to the trace and the `PlannerAgent` is asked to propose a revised tool binding; if revision fails after two attempts, the run fails.
   - `NodeAgent` executes the node, returns a `NodeOutput { nodeId, updatedFields, rawContent, ok, errorReason }`.
   - The **secret sanitizer** scrubs `NodeOutput.rawContent`.
   - The workflow emits `NodeExecuted` on `GraphRunEntity`, merging `updatedFields` into `GraphState`.
   - `EdgeRouterAgent` evaluates the outgoing edges from the current node against the updated `GraphState` and returns a `RoutingDecision { nextNodeId | TERMINAL }`.
   - If `nextNodeId` refers to a previously visited node, the cycle counter for that node is incremented; if it exceeds `maxVisits`, the run fails.
4. On `TERMINAL`, the workflow emits `RunCompleted` with a `RunResult { summary, finalState, traceLength }`.
5. The operator can click **Halt new dispatches** at any time. The in-flight `NodeAgent` call finishes; the next loop iteration reads the halt flag and ends with `RunHaltedOperator`.

A `RunSimulator` (TimedAction) drips a sample graph definition every 90 seconds.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Validates a raw graph definition; returns a `GraphPlan`. Revises a node's tool binding on guardrail rejection. | `GraphWorkflow` | returns typed result to workflow |
| `NodeAgent` | `AutonomousAgent` | Executes a single node: invokes the node's tool fixture (if any), applies node logic, returns `NodeOutput`. | `GraphWorkflow` | — |
| `EdgeRouterAgent` | `AutonomousAgent` | Evaluates the routing predicate for each outgoing edge given the current `GraphState`; returns `RoutingDecision`. | `GraphWorkflow` | — |
| `GraphWorkflow` | `Workflow` | Drives the plan → execute-node-guarded → sanitize → record-state → route → check-cycle → check-halt loop, plus replan and terminal branches. | `GraphEndpoint`, `RunRequestConsumer` | `GraphRunEntity` |
| `GraphRunEntity` | `EventSourcedEntity` | Holds the graph run lifecycle, `GraphState`, execution trace, and final result. | `GraphWorkflow` | `GraphRunView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `GraphEndpoint` (operator actions) | `GraphWorkflow` (polls) |
| `RunQueueEntity` | `EventSourcedEntity` | Audit log of submitted graph run requests. | `GraphEndpoint`, `RunSimulator` | `RunRequestConsumer` |
| `GraphRunView` | `View` | List-of-runs read model for the UI. | `GraphRunEntity` events | `GraphEndpoint` |
| `RunRequestConsumer` | `Consumer` | Subscribes to `RunQueueEntity` events; starts a `GraphWorkflow` per submission. | `RunQueueEntity` events | `GraphWorkflow` |
| `RunSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/graph-definitions.jsonl` and enqueues it. | scheduler | `RunQueueEntity` |
| `StuckRunMonitor` | `TimedAction` | Every 30 s, marks runs stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `GraphRunEntity` |
| `GraphEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, get, list, SSE, operator halt/resume. | — | `GraphRunView`, `RunQueueEntity`, `GraphRunEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record GraphDefinitionRequest(String graphJson, String requestedBy) {}

record NodeDef(
    String nodeId,
    String description,
    Optional<String> toolAnnotation,
    int maxVisits
) {}

record EdgeDef(
    String fromNodeId,
    String toNodeId,
    Optional<String> predicate,
    boolean isBackEdge
) {}

record GraphPlan(
    List<NodeDef> nodes,
    List<EdgeDef> edges,
    String entryNodeId,
    List<String> terminalNodeIds,
    List<String> cycleAnnotations
) {}

record NodeDispatch(
    String nodeId,
    Optional<String> toolAnnotation,
    GraphState currentState
) {}

record NodeOutput(
    String nodeId,
    Map<String, Object> updatedFields,
    String rawContent,
    boolean ok,
    Optional<String> errorReason
) {}

record RoutingDecision(
    String fromNodeId,
    Optional<String> nextNodeId,
    boolean terminal,
    String rationale
) {}

record TraceEntry(
    int stepIndex,
    String nodeId,
    NodeExecutionVerdict verdict,
    String scrubbedContent,
    Optional<String> blocker,
    Instant recordedAt
) {}

record GraphState(Map<String, Object> fields) {}

record RunResult(
    String summary,
    GraphState finalState,
    int traceLength,
    Instant producedAt
) {}

record GraphRun(
    String runId,
    String graphJson,
    RunStatus status,
    Optional<GraphPlan> plan,
    Optional<GraphState> currentState,
    Optional<List<TraceEntry>> trace,
    Optional<RunResult> result,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum NodeExecutionVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, SANITIZED }
enum RunStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`GraphRunEntity`)

`RunCreated`, `RunPlanned`, `NodeDispatched`, `NodeBlocked`, `NodeExecuted`, `StateUpdated`, `RunCompleted`, `RunFailed`, `RunHaltedOperator`, `RunHaltedAutomatic`, `RunFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RunQueueEntity`)

`RunSubmitted { runId, graphJson, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ graphJson, requestedBy? }` → `202 { runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=...`.
- `GET /api/runs/{id}` — one run (full plan + state + trace + result).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Graph API Plugin"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a graph definition (JSON textarea), operator halt/resume control, live list of runs with status pills, expand-row to see the graph plan, execution trace, and final result.

Browser title: `<title>Akka Sample: Graph API Plugin</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `NodeAgent`): every `NodeDispatch` that carries a `toolAnnotation` is checked against (a) a host allow-list (`akka.io`, `doc.akka.io`, `github.com`) and (b) a function allow-list (read-only fixture operations). Blocking. Failure → `NodeBlocked` trace entry + request to `PlannerAgent` for a revised tool binding.
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): the App UI operator pane shows all in-flight runs, the current node, and the last trace entry. Halt / Resume buttons drive `SystemControlEntity`. `GraphWorkflow` polls `SystemControlEntity` before each node dispatch and exits with `RunHaltedOperator` if the flag is set.
- **S1 — secret sanitizer** (`sanitizer`, flavor `secret`): every `NodeOutput.rawContent` is scrubbed before it lands on `GraphState` or in a `TraceEntry`. The scrubber matches AWS access keys, GitHub tokens, JWTs, OpenAI keys, and high-entropy strings ≥ 32 chars with Shannon entropy > 4.5 bits/char.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Validates graph definitions; revises tool bindings.
- `NodeAgent` → `prompts/node-executor.md`. Executes a single node against fixtures.
- `EdgeRouterAgent` → `prompts/edge-router.md`. Evaluates routing predicates.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a 4-node graph with a conditional edge. Run progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. Trace shows one entry per node. Final state contains expected fields.
2. **J2** — Submit a graph whose first node has a tool annotation naming a non-allow-listed host. Guardrail blocks the dispatch; planner is asked to revise; run either completes with a revised binding or fails with a clear reason.
3. **J3** — Submit a graph and click **Halt new dispatches** while `EXECUTING`. In-flight node finishes; no further dispatches; run ends in `HALTED`.
4. **J4** — A node output contains an `AKIA…` key shape; the trace entry's `scrubbedContent` shows `[REDACTED:aws-access-key]`; subsequent agent prompts never contain the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-stategraph-bridge demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-akka-stategraph-bridge.
Java package io.akka.samples.graphapiplugin. Akka 3.6.0. HTTP port 9454.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PlannerAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(PLAN_GRAPH).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(REVISE_TOOL_BINDING).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_GRAPH returns GraphPlan.
    REVISE_TOOL_BINDING returns a revised NodeDef for the blocked node.
  * NodeAgent — capability(TaskAcceptance.of(EXECUTE_NODE).maxIterationsPerTask(2)).
    Prompt from prompts/node-executor.md. Returns NodeOutput.
  * EdgeRouterAgent — capability(TaskAcceptance.of(ROUTE_EDGE).maxIterationsPerTask(2)).
    Prompt from prompts/edge-router.md. Returns RoutingDecision.

- 1 Workflow GraphWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> resolveNodeStep -> guardrailStep ->
  executeNodeStep -> sanitizeStep -> recordStateStep -> routeStep ->
  checkCycleStep -> decideStep
  -> [back to checkHaltStep or to completeStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), executeNodeStep ofSeconds(120),
    routeStep ofSeconds(45), completeStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(GraphWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits RunHaltedOperator on GraphRunEntity).
  guardrailStep runs ToolCallGuardrail.vet(nodeDispatch); on reject
  records a NodeBlocked trace entry via GraphRunEntity.recordBlock(nodeId, reason)
  and calls PlannerAgent.REVISE_TOOL_BINDING; if revision returns empty,
  transitions to failStep.
  executeNodeStep calls forAutonomousAgent(NodeAgent.class, EXECUTE_NODE)
  with the current NodeDispatch.
  sanitizeStep applies SecretScrubber.scrub to NodeOutput.rawContent.
  recordStateStep calls GraphRunEntity.recordNodeExecuted(entry, updatedFields).
  routeStep calls forAutonomousAgent(EdgeRouterAgent.class, ROUTE_EDGE)
  with current node id and GraphState.
  checkCycleStep increments per-node visit counter; on exceeding NodeDef.maxVisits
  transitions to failStep with reason "cycle limit exceeded: <nodeId>".
  decideStep: if RoutingDecision.terminal, transitions to completeStep; else
  loops back to checkHaltStep with the new current node id.

- 1 EventSourcedEntity GraphRunEntity holding GraphRun state. emptyState()
  returns GraphRun.initial("", null). Commands:
  createRun, recordPlan, recordNodeDispatched, recordBlock, recordNodeExecuted,
  updateState, completeRun, failRun, haltOperator, haltAutomatic,
  timeoutFail, getRun. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RunQueueEntity with command enqueueRun(runId, graphJson,
  requestedBy) emitting RunSubmitted.

- 1 View GraphRunView with row type GraphRunRow (mirror of GraphRun minus heavy
  trace payload — truncate to last 3 trace entries plus counts; UI fetches full
  run by id on click). Table updater consumes GraphRunEntity events.
  ONE query getAllRuns SELECT * AS runs FROM graph_run_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer RunRequestConsumer subscribed to RunQueueEntity events; on
  RunSubmitted starts a GraphWorkflow with runId as the workflow id.

- 2 TimedActions:
  * RunSimulator — every 90s, reads next line from
    src/main/resources/sample-events/graph-definitions.jsonl and calls
    RunQueueEntity.enqueueRun.
  * StuckRunMonitor — every 30s, queries GraphRunView.getAllRuns, filters
    EXECUTING runs whose createdAt is older than 5 minutes, calls
    GraphRunEntity.timeoutFail; GraphWorkflow polls GraphRunEntity.getRun in
    its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * GraphEndpoint at /api with POST /runs, GET /runs (filters client-side
    from getAllRuns), GET /runs/{id}, GET /runs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring two Task<R> constants: PLAN_GRAPH
  (resultConformsTo GraphPlan), REVISE_TOOL_BINDING (resultConformsTo NodeDef).
- NodeTasks.java declaring three Task<R> constants: EXECUTE_NODE
  (resultConformsTo NodeOutput), ROUTE_EDGE (resultConformsTo RoutingDecision),
  each declared in a shared NodeTasks companion.
- Domain records as listed in SPEC §5, plus SystemControl entity state record.
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36},
  JWT ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, high-entropy fallback for tokens ≥ 32 chars
  with Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token], [REDACTED:jwt],
  [REDACTED:openai-key], [REDACTED:bearer-token], [REDACTED:high-entropy].
- application/ToolCallGuardrail.java — deterministic vetter. Reject if
  toolAnnotation names a host not on the allow-list (akka.io, doc.akka.io,
  github.com), or if the function name matches a mutating-operation pattern
  (write|delete|update|post|create|drop|truncate — case-insensitive prefix).
  Write operations are fixture-read-only in this sample.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9454 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/graph-definitions.jsonl with 6 canned
  graph definitions: a linear 3-node DAG, a 4-node graph with a conditional
  branch, a 3-node graph with one back-edge (cycle), a 5-node graph with
  two parallel branches merging at a terminal node, a graph whose first
  node has a non-allow-listed tool annotation (exercises J2), a graph
  whose second node returns content with an AKIA-shaped key fragment
  (exercises J4).
- src/main/resources/sample-data/node-fixtures.jsonl — 12 canned node
  execution outputs (nodeId, updatedFields, rawContent). One entry's rawContent
  contains the literal substring "AKIAIOSFODNN7EXAMPLE".
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint).
- eval-matrix.yaml at the project root with 3 controls (G1, HO1, S1).
- risk-survey.yaml at the project root pre-filling purpose, data, decisions,
  failure, oversight, operations, compliance.
- prompts/planner.md, prompts/node-executor.md, prompts/edge-router.md loaded
  at agent startup as system prompts.
- README.md at the project root matching Section 1 of this SPEC.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs matching the formal exemplar.
  Browser title exactly: <title>Akka Sample: Graph API Plugin</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent task id (see Mock LLM
        provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java dispatching on
  agent class name and Task<R> id.
- planner.json — keyed by task id:
    "PLAN_GRAPH" → 4 GraphPlan entries covering the 6 sample graph topologies.
    "REVISE_TOOL_BINDING" → 3 revised NodeDef entries with allow-listed tool
    annotations.
- node-executor.json — 6 NodeOutput entries, ok=true; one entry's rawContent
  contains "AKIAIOSFODNN7EXAMPLE".
- edge-router.json — 6 RoutingDecision entries covering continue, conditional
  branch selection, and terminal.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent extends clause required for every agent.
- Lesson 4: WorkflowSettings.stepTimeout set on planStep, executeNodeStep,
  routeStep, completeStep.
- Lesson 6: Optional<T> for all nullable lifecycle fields.
- Lesson 7: PlannerTasks.java and NodeTasks.java companions required.
- Lesson 8: model-name values claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: HTTP port 9454.
- Lesson 11: source.platform never in user-facing surfaces.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names.
- Lesson 24: mermaid CSS overrides + themeVariables present.
- Lesson 25: API-key sourcing follows five-option flow; no key value written
  to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; no zombie panels.
- Overview tab Try-it card shows just "/akka:build".
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
