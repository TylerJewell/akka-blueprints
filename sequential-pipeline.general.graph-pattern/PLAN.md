# PLAN — graph-pattern

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[GraphEndpoint]:::ep
  Entity[GraphRunEntity]:::ese
  WF[GraphExecutionWorkflow]:::wf
  Agent[GraphAgent]:::agent
  Parse[ParseTools]:::tool
  Plan[PlanTools]:::tool
  Execute[ExecuteTools]:::tool
  Merge[MergeTools]:::tool
  Guard[DependencyGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[GraphRunView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Plan
  Agent -->|invokes| Execute
  Agent -->|invokes| Merge
  Guard -->|recordDependencyViolation| Entity
  Agent -->|ParsedRequest / GraphPlan / ExecutionResult / TaskResult| WF
  WF -->|recordParsedRequest/Plan/ExecutionResult/TaskResult| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as GraphEndpoint
  participant E as GraphRunEntity
  participant W as GraphExecutionWorkflow
  participant A as GraphAgent
  participant G as DependencyGuardrail
  participant T as Tools (Parse/Plan/Execute/Merge)
  participant Sc as CoverageScorer

  U->>API: POST /api/runs { description }
  API->>E: create(description)
  E-->>API: { runId }
  API->>W: start(runId, description)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_REQUEST, description)
  A->>G: before-tool-call(extractIntent, PARSE)
  G-->>A: accept (status PARSING)
  A->>T: extractIntent + identifyConstraints
  T-->>A: ParsedRequest
  A-->>W: ParsedRequest
  W->>E: recordParsedRequest
  W->>A: runSingleTask(PLAN_GRAPH, parsedRequest)
  A->>G: before-tool-call(buildNodes, PLAN)
  G-->>A: accept (status PLANNING and parsedRequest present)
  A->>T: buildNodes + defineEdges
  T-->>A: GraphPlan
  A-->>W: GraphPlan
  W->>E: recordPlan
  W->>A: runSingleTask(EXECUTE_NODES, plan)
  A->>G: before-tool-call(runNode nodeA, EXECUTE — root node)
  G-->>A: accept (no predecessors required)
  A->>T: runNode(nodeA)
  T-->>A: NodeOutput(nodeA)
  W->>E: recordNodeExecuted(nodeA)
  A->>G: before-tool-call(runNode nodeB, predecessors=[nodeA])
  G-->>A: accept (nodeA is recorded)
  A->>T: runNode(nodeB, [nodeA])
  T-->>A: NodeOutput(nodeB)
  W->>E: recordNodeExecuted(nodeB) then AllNodesExecuted
  A-->>W: ExecutionResult
  W->>A: runSingleTask(MERGE_OUTPUTS, executionResult)
  A->>G: before-tool-call(aggregateOutputs, MERGE)
  G-->>A: accept (status MERGING and executionResult present)
  A->>T: aggregateOutputs + formatResult
  T-->>A: TaskResult
  A-->>W: TaskResult
  W->>E: recordTaskResult
  W->>Sc: score(taskResult, executionResult, plan)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `GraphRunEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: RequestParsed
  PARSED --> PLANNING: PlanStarted
  PLANNING --> PLANNED: GraphPlanned
  PLANNED --> EXECUTING: ExecuteStarted
  EXECUTING --> EXECUTED: AllNodesExecuted
  EXECUTED --> MERGING: MergeStarted
  MERGING --> MERGED: OutputsMerged
  MERGED --> EVALUATED: EvaluationScored
  PARSING --> FAILED: RunFailed
  PLANNING --> FAILED: RunFailed
  EXECUTING --> FAILED: RunFailed
  MERGING --> FAILED: RunFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`DependencyViolated` is a side-event recorded on the entity for audit within the EXECUTING state; it does not change status. `NodeExecuted` events are recorded individually within the EXECUTING state as each node completes. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  GraphRunEntity ||--o{ RunCreated : emits
  GraphRunEntity ||--o{ ParseStarted : emits
  GraphRunEntity ||--o{ RequestParsed : emits
  GraphRunEntity ||--o{ PlanStarted : emits
  GraphRunEntity ||--o{ GraphPlanned : emits
  GraphRunEntity ||--o{ ExecuteStarted : emits
  GraphRunEntity ||--o{ NodeExecuted : emits
  GraphRunEntity ||--o{ AllNodesExecuted : emits
  GraphRunEntity ||--o{ MergeStarted : emits
  GraphRunEntity ||--o{ OutputsMerged : emits
  GraphRunEntity ||--o{ EvaluationScored : emits
  GraphRunEntity ||--o{ DependencyViolated : emits
  GraphRunEntity ||--o{ RunFailed : emits
  GraphRunView }o--|| GraphRunEntity : projects
  GraphExecutionWorkflow }o--|| GraphRunEntity : reads-and-writes
  GraphAgent ||--o{ ParsedRequest : returns
  GraphAgent ||--o{ GraphPlan : returns
  GraphAgent ||--o{ ExecutionResult : returns
  GraphAgent ||--o{ TaskResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GraphEndpoint` | `api/GraphEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `GraphRunEntity` | `application/GraphRunEntity.java` (state in `domain/RunRecord.java`, events in `domain/RunEvent.java`) |
| `GraphExecutionWorkflow` | `application/GraphExecutionWorkflow.java` |
| `GraphAgent` | `application/GraphAgent.java` (tasks in `application/GraphTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `PlanTools` | `application/PlanTools.java` |
| `ExecuteTools` | `application/ExecuteTools.java` |
| `MergeTools` | `application/MergeTools.java` |
| `DependencyGuardrail` | `application/DependencyGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `GraphRunView` | `application/GraphRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `planStep` 60 s, `executeStep` 120 s (node execution may span multiple tool round-trips), `mergeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(GraphExecutionWorkflow::error)`. The extended timeout on `executeStep` accommodates a multi-node DAG where each node's tool call goes through an LLM iteration.
- **Idempotency**: each workflow uses `"graph-" + runId` as the workflow id; restart of the same `runId` is rejected by the workflow runtime. The agent instance id is `"agent-" + runId`.
- **One agent per run**: `GraphAgent` runs four tasks per run — PARSE, PLAN, EXECUTE, MERGE — each with `capability(...).maxIterationsPerTask(5)`. The 5-iteration budget on EXECUTE gives the guardrail room to reject an out-of-order node-execution call and let the agent self-correct.
- **Incremental NodeExecuted writes**: within `executeStep`, the workflow writes `NodeExecuted{nodeId, output}` to the entity after each `runNode` tool call returns. `DependencyGuardrail` reads this partial state to enforce per-node predecessor checks. `AllNodesExecuted` is written once the full `ExecutionResult` is committed.
- **Eval is synchronous and deterministic**: `CoverageScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same result always scores the same. This is a deliberate single-agent invariant.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed run stays at the last successful event; the UI shows the partial state.
