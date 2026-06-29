# PLAN — impact-study-runner

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
  classDef scorer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[StudyEndpoint]:::ep
  Entity[StudyEntity]:::ese
  WF[StudyPipelineWorkflow]:::wf
  Agent[StudyAgent]:::agent
  LF[LoadFlowTools]:::tool
  CT[ContingencyTools]:::tool
  DR[DraftTools]:::tool
  Scorer[SimulatorScorer]:::scorer
  View[StudyView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|loadFlowStep runSingleTask| Agent
  Agent -->|invokes| LF
  Agent -->|invokes| CT
  Agent -->|invokes| DR
  Agent -->|LoadFlowResult / ContingencyResult / StudyReport| WF
  WF -->|recordLoadFlow/Contingency/Report| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  WF -->|approvalStep pause| Entity
  API -->|approve/reject| Entity
  Entity -->|resume| WF
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 + J2 (happy path with engineer approval)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as StudyEndpoint
  participant E as StudyEntity
  participant W as StudyPipelineWorkflow
  participant A as StudyAgent
  participant T as Tools (LF/CT/DR)
  participant Sc as SimulatorScorer
  participant Eng as Engineer (UI)

  U->>API: POST /api/studies { requestId }
  API->>E: create(requestId)
  E-->>API: { studyId }
  API->>W: start(studyId, requestId)
  W->>E: startLoadFlow
  W->>A: runSingleTask(RUN_LOAD_FLOW, requestId)
  A->>T: computeBusVoltages + computePowerFlows
  T-->>A: LoadFlowResult
  A-->>W: LoadFlowResult
  W->>E: recordLoadFlow
  W->>A: runSingleTask(RUN_CONTINGENCY, loadFlowResult)
  A->>T: runN1Analysis + checkVoltageThresholds
  T-->>A: ContingencyResult
  A-->>W: ContingencyResult
  W->>E: recordContingency
  W->>A: runSingleTask(DRAFT_REPORT, contingencyResult)
  A->>T: formatFinding + assembleExecutiveSummary
  T-->>A: StudyReport
  A-->>W: StudyReport
  W->>E: recordReport
  W->>Sc: score(studyReport, contingencyResult, loadFlowResult)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  W->>E: requestApproval
  E-.->>U: SSE event(AWAITING_APPROVAL)
  Eng->>API: POST /api/studies/{id}/approve { note }
  API->>E: approve(note)
  E-->>W: signal
  W->>E: studyApproved
  E-.->>U: SSE event(APPROVED)
```

## State machine — `StudyEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RUNNING_LOAD_FLOW: LoadFlowStarted
  RUNNING_LOAD_FLOW --> LOAD_FLOW_DONE: LoadFlowComputed
  LOAD_FLOW_DONE --> RUNNING_CONTINGENCY: ContingencyStarted
  RUNNING_CONTINGENCY --> CONTINGENCY_DONE: ContingencyAnalyzed
  CONTINGENCY_DONE --> DRAFTING: DraftStarted
  DRAFTING --> REPORT_DRAFTED: ReportDrafted
  REPORT_DRAFTED --> AWAITING_APPROVAL: EvaluationScored + ApprovalRequested
  AWAITING_APPROVAL --> APPROVED: StudyApproved
  AWAITING_APPROVAL --> REJECTED: StudyRejected
  RUNNING_LOAD_FLOW --> FAILED: StudyFailed
  RUNNING_CONTINGENCY --> FAILED: StudyFailed
  DRAFTING --> FAILED: StudyFailed
  APPROVED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

ApprovalRequested is a side-transition to document the workflow's pause point; it does not change status on its own — the combined sequence `EvaluationScored` then `ApprovalRequested` both land before the status settles at `AWAITING_APPROVAL`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  StudyEntity ||--o{ StudyCreated : emits
  StudyEntity ||--o{ LoadFlowStarted : emits
  StudyEntity ||--o{ LoadFlowComputed : emits
  StudyEntity ||--o{ ContingencyStarted : emits
  StudyEntity ||--o{ ContingencyAnalyzed : emits
  StudyEntity ||--o{ DraftStarted : emits
  StudyEntity ||--o{ ReportDrafted : emits
  StudyEntity ||--o{ EvaluationScored : emits
  StudyEntity ||--o{ ApprovalRequested : emits
  StudyEntity ||--o{ StudyApproved : emits
  StudyEntity ||--o{ StudyRejected : emits
  StudyEntity ||--o{ StudyFailed : emits
  StudyView }o--|| StudyEntity : projects
  StudyPipelineWorkflow }o--|| StudyEntity : reads-and-writes
  StudyAgent ||--o{ LoadFlowResult : returns
  StudyAgent ||--o{ ContingencyResult : returns
  StudyAgent ||--o{ StudyReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `StudyEndpoint` | `api/StudyEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `StudyEntity` | `application/StudyEntity.java` (state in `domain/StudyRecord.java`, events in `domain/StudyEvent.java`) |
| `StudyPipelineWorkflow` | `application/StudyPipelineWorkflow.java` |
| `StudyAgent` | `application/StudyAgent.java` (tasks in `application/StudyTasks.java`) |
| `LoadFlowTools` | `application/LoadFlowTools.java` |
| `ContingencyTools` | `application/ContingencyTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `SimulatorScorer` | `application/SimulatorScorer.java` |
| `StudyView` | `application/StudyView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `loadFlowStep` 60 s, `contingencyStep` 60 s, `draftStep` 60 s, `evalStep` 5 s, `approvalStep` null (waits indefinitely), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(StudyPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + studyId` as the workflow id; restart of the same studyId is rejected by the workflow runtime. The agent instance id is `"agent-" + studyId` so each study has its own per-task conversation memory.
- **One agent per study**: `StudyAgent` runs three tasks per study — LOAD_FLOW, CONTINGENCY, DRAFT — each with `capability(...).maxIterationsPerTask(4)`.
- **Eval is synchronous and deterministic**: `SimulatorScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same report always scores the same.
- **Approval gate is durable**: `approvalStep` persists the waiting state inside the Akka workflow runtime. The engineer's action arrives as an external signal via `StudyEntity.approve` or `StudyEntity.reject`; neither the workflow nor the entity auto-approves on timeout.
- **Task-boundary handoff is the dependency contract**: `loadFlowStep` writes `LoadFlowComputed` BEFORE returning; `contingencyStep` reads the recorded `LoadFlowResult` from the entity to build its task's instruction context; `draftStep` reads both `LoadFlowResult` and `ContingencyResult`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, a single-task agent call, or an external approval wait. A failed study stays at the last successful event; the UI shows the partial state.
