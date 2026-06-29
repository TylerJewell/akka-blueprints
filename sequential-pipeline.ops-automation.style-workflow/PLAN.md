# PLAN — workflow-orchestration

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

  API[WorkflowRunEndpoint]:::ep
  Entity[WorkflowRunEntity]:::ese
  WF[WorkflowRunPipeline]:::wf
  Agent[PipelineOrchestrationAgent]:::agent
  Validate[ValidateTools]:::tool
  Execute[ExecuteTools]:::tool
  Notify[NotifyTools]:::tool
  Guard[StageGuardrail]:::guard
  Scorer[StepCoverageScorer]:::guard
  View[WorkflowRunView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|validateStage runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Validate
  Agent -->|invokes| Execute
  Agent -->|invokes| Notify
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ValidationReport / ExecutionResult / RunResult| WF
  WF -->|recordValidation/Execution/Result| Entity
  WF -->|evalStage score| Scorer
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
  participant U as Operator (UI)
  participant API as WorkflowRunEndpoint
  participant E as WorkflowRunEntity
  participant W as WorkflowRunPipeline
  participant A as PipelineOrchestrationAgent
  participant G as StageGuardrail
  participant T as Tools (Validate/Execute/Notify)
  participant Sc as StepCoverageScorer

  U->>API: POST /api/runs { workflowId }
  API->>E: create(workflowId)
  E-->>API: { runId }
  API->>W: start(runId, workflowId)
  W->>E: startValidate
  W->>A: runSingleTask(VALIDATE_STEPS, workflowId)
  A->>G: before-tool-call(checkStepDependencies, VALIDATE)
  G-->>A: accept (status VALIDATING)
  A->>T: checkStepDependencies + verifyStepPreconditions
  T-->>A: List<StepValidation>
  A-->>W: ValidationReport
  W->>E: recordValidation
  W->>A: runSingleTask(EXECUTE_STEPS, validation)
  A->>G: before-tool-call(runStep, EXECUTE)
  G-->>A: accept (status EXECUTING and validation present)
  A->>T: runStep + recordStepOutcome
  T-->>A: List<StepOutcome>
  A-->>W: ExecutionResult
  W->>E: recordExecution
  W->>A: runSingleTask(NOTIFY_COMPLETION, execution)
  A->>G: before-tool-call(buildRunSummary, NOTIFY)
  G-->>A: accept (status NOTIFYING and execution present)
  A->>T: buildRunSummary + dispatchNotification
  T-->>A: RunSummary / NotificationReceipt
  A-->>W: RunResult
  W->>E: recordResult
  W->>Sc: score(runResult, execution, validation)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `WorkflowRunEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> VALIDATING: ValidateStarted
  VALIDATING --> VALIDATED: StepsValidated
  VALIDATED --> EXECUTING: ExecuteStarted
  EXECUTING --> EXECUTED: StepsExecuted
  EXECUTED --> NOTIFYING: NotifyStarted
  NOTIFYING --> NOTIFIED: NotificationSent
  NOTIFIED --> EVALUATED: RunEvaluated
  VALIDATING --> FAILED: RunFailed
  EXECUTING --> FAILED: RunFailed
  NOTIFYING --> FAILED: RunFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`StageGuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's stage continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  WorkflowRunEntity ||--o{ RunCreated : emits
  WorkflowRunEntity ||--o{ ValidateStarted : emits
  WorkflowRunEntity ||--o{ StepsValidated : emits
  WorkflowRunEntity ||--o{ ExecuteStarted : emits
  WorkflowRunEntity ||--o{ StepsExecuted : emits
  WorkflowRunEntity ||--o{ NotifyStarted : emits
  WorkflowRunEntity ||--o{ NotificationSent : emits
  WorkflowRunEntity ||--o{ RunEvaluated : emits
  WorkflowRunEntity ||--o{ StageGuardrailRejected : emits
  WorkflowRunEntity ||--o{ RunFailed : emits
  WorkflowRunView }o--|| WorkflowRunEntity : projects
  WorkflowRunPipeline }o--|| WorkflowRunEntity : reads-and-writes
  PipelineOrchestrationAgent ||--o{ ValidationReport : returns
  PipelineOrchestrationAgent ||--o{ ExecutionResult : returns
  PipelineOrchestrationAgent ||--o{ RunResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WorkflowRunEndpoint` | `api/WorkflowRunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `WorkflowRunEntity` | `application/WorkflowRunEntity.java` (state in `domain/WorkflowRunRecord.java`, events in `domain/WorkflowRunEvent.java`) |
| `WorkflowRunPipeline` | `application/WorkflowRunPipeline.java` |
| `PipelineOrchestrationAgent` | `application/PipelineOrchestrationAgent.java` (tasks in `application/WorkflowTasks.java`) |
| `ValidateTools` | `application/ValidateTools.java` |
| `ExecuteTools` | `application/ExecuteTools.java` |
| `NotifyTools` | `application/NotifyTools.java` |
| `StageGuardrail` | `application/StageGuardrail.java` |
| `StepCoverageScorer` | `application/StepCoverageScorer.java` |
| `WorkflowRunView` | `application/WorkflowRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-stage timeout**: `validateStage` 60 s, `executeStage` 60 s, `notifyStage` 60 s, `evalStage` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(WorkflowRunPipeline::error)`. The 60 s on each agent-calling stage accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + runId` as the workflow id; restart of the same runId is rejected by the workflow runtime. The agent instance id is `"agent-" + runId` so each run has its own per-task conversation memory.
- **One agent per run**: `PipelineOrchestrationAgent` runs three tasks per run — VALIDATE, EXECUTE, NOTIFY — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `StageGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow stage fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `StepCoverageScorer` runs in-process inside `evalStage`. No LLM call, no external service — the same run always scores the same. This is a deliberate single-agent invariant.
- **Stage-boundary handoff is the dependency contract**: `validateStage` writes `StepsValidated` BEFORE returning; `executeStage` reads the recorded `ValidationReport` from the entity to build its task's instruction context; `notifyStage` reads both `ValidationReport` and `ExecutionResult`. The agent itself is stateless across stages.
- **No saga / no compensation**: every stage is either pure read, append-only event write, or a single-task agent call. A failed run stays at the last successful event; the UI shows the partial state for the operator.
