# PLAN — akka-bridge

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
  Plan[PlanTools]:::tool
  Execute[ExecuteTools]:::tool
  Finalize[FinalizeTools]:::tool
  Guard[OutputGuardrail]:::guard
  Checker[PolicyChecker]:::guard
  View[GraphRunView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|planStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Plan
  Agent -->|invokes| Execute
  Agent -->|invokes| Finalize
  Guard -->|recordViolation| Entity
  Agent -->|ExecutionPlan / NodeOutputSet / FinalResult| WF
  WF -->|recordPlan/NodeOutputs/Result| Entity
  WF -->|evalStep score| Checker
  Checker -->|EvalResult| WF
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
  participant G as OutputGuardrail
  participant T as Tools (Plan/Execute/Finalize)
  participant Ch as PolicyChecker

  U->>API: POST /api/runs { graphId }
  API->>E: create(graphId)
  E-->>API: { runId }
  API->>W: start(runId, graphId)
  W->>E: startPlan
  W->>A: runSingleTask(PLAN_GRAPH, graphId)
  A->>T: parseGraphDefinition + estimateNodeCost
  T-->>A: List<PlannedNode>
  A-->>G: after-llm-response(ExecutionPlan output)
  G-->>A: accept (schema valid, no prohibited patterns)
  A-->>W: ExecutionPlan
  W->>E: recordPlan
  W->>A: runSingleTask(EXECUTE_NODE, plan)
  A->>T: invokeNode + checkNodeStatus (per node)
  T-->>A: NodeOutput per node
  A-->>G: after-llm-response(NodeOutputSet output)
  G-->>A: accept
  A-->>W: NodeOutputSet
  W->>E: recordNodeOutputs
  W->>A: runSingleTask(FINALIZE_OUTPUT, nodeOutputs)
  A->>T: aggregateOutputs + formatResult
  T-->>A: ResultItem list / FinalResult
  A-->>G: after-llm-response(FinalResult output)
  G-->>A: accept
  A-->>W: FinalResult
  W->>E: recordResult
  W->>Ch: score(result, nodeOutputs, plan)
  Ch-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `GraphRunEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PLANNING: PlanStarted
  PLANNING --> PLANNED: GraphPlanned
  PLANNED --> EXECUTING: ExecuteStarted
  EXECUTING --> EXECUTED: NodeExecuted
  EXECUTED --> FINALIZING: FinalizeStarted
  FINALIZING --> FINALIZED: OutputFinalized
  FINALIZED --> EVALUATED: EvaluationScored
  PLANNING --> FAILED: GraphRunFailed
  EXECUTING --> FAILED: GraphRunFailed
  FINALIZING --> FAILED: GraphRunFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`GuardrailViolated` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  GraphRunEntity ||--o{ GraphRunCreated : emits
  GraphRunEntity ||--o{ PlanStarted : emits
  GraphRunEntity ||--o{ GraphPlanned : emits
  GraphRunEntity ||--o{ ExecuteStarted : emits
  GraphRunEntity ||--o{ NodeExecuted : emits
  GraphRunEntity ||--o{ FinalizeStarted : emits
  GraphRunEntity ||--o{ OutputFinalized : emits
  GraphRunEntity ||--o{ EvaluationScored : emits
  GraphRunEntity ||--o{ GuardrailViolated : emits
  GraphRunEntity ||--o{ GraphRunFailed : emits
  GraphRunView }o--|| GraphRunEntity : projects
  GraphExecutionWorkflow }o--|| GraphRunEntity : reads-and-writes
  GraphAgent ||--o{ ExecutionPlan : returns
  GraphAgent ||--o{ NodeOutputSet : returns
  GraphAgent ||--o{ FinalResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GraphEndpoint` | `api/GraphEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `GraphRunEntity` | `application/GraphRunEntity.java` (state in `domain/GraphRunRecord.java`, events in `domain/GraphRunEvent.java`) |
| `GraphExecutionWorkflow` | `application/GraphExecutionWorkflow.java` |
| `GraphAgent` | `application/GraphAgent.java` (tasks in `application/GraphTasks.java`) |
| `PlanTools` | `application/PlanTools.java` |
| `ExecuteTools` | `application/ExecuteTools.java` |
| `FinalizeTools` | `application/FinalizeTools.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `PolicyChecker` | `application/PolicyChecker.java` |
| `GraphRunView` | `application/GraphRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `planStep` 60 s, `executeStep` 60 s, `finalizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(GraphExecutionWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"run-" + runId` as the workflow id; restart of the same runId is rejected by the workflow runtime. The agent instance id is `"agent-" + runId` so each run has its own per-task conversation memory.
- **One agent per run**: `GraphAgent` runs three tasks per run — PLAN, EXECUTE, FINALIZE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to block a policy-violating output and still let the agent self-correct.
- **Guardrail-driven retry**: when `OutputGuardrail` blocks a response, the block is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `PolicyChecker` runs in-process inside `evalStep`. No LLM call, no external service — the same result always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `planStep` writes `GraphPlanned` BEFORE returning; `executeStep` reads the recorded `ExecutionPlan` from the entity to build its task's instruction context; `finalizeStep` reads both `ExecutionPlan` and `NodeOutputSet`. The agent itself is stateless across phases — it never holds plan + execute + finalize context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed run stays at the last successful event; the UI shows the partial state for the user.
