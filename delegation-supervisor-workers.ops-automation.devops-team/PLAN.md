# PLAN — DevOps Multi-Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  OE[OpsEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  PQ[PipelineQueue<br/>EventSourcedEntity]:::ese
  PC[PipelineConsumer<br/>Consumer]:::con
  WF[OpsWorkflow<br/>Workflow]:::wf
  CO[OpsCoordinator<br/>AutonomousAgent]:::ag
  IA[InfraAgent<br/>AutonomousAgent]:::ag
  DA[DeployAgent<br/>AutonomousAgent]:::ag
  OA[ObservabilityAgent<br/>AutonomousAgent]:::ag
  CE[ChangeRequestEntity<br/>EventSourcedEntity]:::ese
  HS[HaltSignalEntity<br/>EventSourcedEntity]:::ese
  VW[OpsView<br/>View]:::vw
  SIM[PipelineSimulator<br/>TimedAction]:::ta
  HM[HaltMonitor<br/>TimedAction]:::ta

  OE -->|POST /ops/changes| PQ
  OE -->|approve / reject / halt| CE
  SIM -.->|every 90s| PQ
  PQ -.->|ChangeSubmitted| PC
  PC -->|start workflow| WF
  WF -->|PLAN| CO
  WF -->|ASSESS_INFRA| IA
  WF -->|ASSESS_DEPLOY| DA
  WF -->|ASSESS_OBS| OA
  WF -->|CONSOLIDATE| CO
  WF -->|commands| CE
  CE -.->|events| VW
  HM -.->|every 30s| HS
  HM -->|halt command| WF
  OE -->|POST /ops/halt| HS
  OE -->|getAllChanges / SSE| VW
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions or scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant OE as OpsEndpoint
  participant PQ as PipelineQueue
  participant WF as OpsWorkflow
  participant CO as OpsCoordinator
  participant IA as InfraAgent
  participant DA as DeployAgent
  participant OA as ObservabilityAgent
  participant CE as ChangeRequestEntity

  U->>OE: POST /api/ops/changes {env, type, description}
  OE->>PQ: submitChange
  PQ-->>WF: PipelineConsumer starts workflow
  WF->>CE: createChange (PLANNING)
  WF->>CO: PLAN -> WorkPlan
  WF->>CE: workPlanEmitted (IN_PROGRESS)
  par parallel fan-out
    WF->>IA: ASSESS_INFRA -> InfraAssessment
  and
    WF->>DA: ASSESS_DEPLOY -> DeployAssessment
  and
    WF->>OA: ASSESS_OBS -> ObsAssessment
  end
  Note over WF: join; if any step times out (60s) -> degradeStep
  WF->>CO: CONSOLIDATE(infra, deploy, obs) -> ReadinessReport
  WF->>WF: guardStep — before-tool-call blocklist check
  alt production target
    WF->>CE: requestApproval (AWAITING_APPROVAL)
    U->>OE: POST /approve
    WF->>CE: approve (APPROVED)
  else non-production target
    WF->>CE: approve (APPROVED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: WorkPlan ready
  IN_PROGRESS --> AWAITING_APPROVAL: production target + report consolidated
  IN_PROGRESS --> APPROVED: non-production target + report consolidated
  IN_PROGRESS --> DEGRADED: a specialist timed out
  IN_PROGRESS --> BLOCKED: guardrail blocked a tool call
  IN_PROGRESS --> HALTED: operator halt signal
  AWAITING_APPROVAL --> APPROVED: operator approves
  AWAITING_APPROVAL --> REJECTED: operator rejects
  AWAITING_APPROVAL --> HALTED: operator halt signal
  DEGRADED --> [*]
  BLOCKED --> [*]
  HALTED --> [*]
  APPROVED --> [*]
  REJECTED --> [*]
```

## Entity model

```mermaid
erDiagram
  CHANGE_REQUEST ||--o| WORK_PLAN : plans
  CHANGE_REQUEST ||--o| INFRA_ASSESSMENT : has
  CHANGE_REQUEST ||--o| DEPLOY_ASSESSMENT : has
  CHANGE_REQUEST ||--o| OBS_ASSESSMENT : has
  CHANGE_REQUEST ||--o| READINESS_REPORT : produces
  PIPELINE_QUEUE ||--|| CHANGE_REQUEST : seeds
  HALT_SIGNAL_ENTITY ||--o{ CHANGE_REQUEST : halts
  CHANGE_REQUEST {
    string changeId
    string targetEnvironment
    string changeType
    enum status
    enum riskLevel
    string approvedBy
    instant createdAt
  }
  PIPELINE_QUEUE {
    string changeId
    string targetEnvironment
    string changeType
    string requestedBy
    instant submittedAt
  }
  HALT_SIGNAL_ENTITY {
    string signalId
    string reason
    instant issuedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `OpsCoordinator` | AutonomousAgent | `application/OpsCoordinator.java` |
| `InfraAgent` | AutonomousAgent | `application/InfraAgent.java` |
| `DeployAgent` | AutonomousAgent | `application/DeployAgent.java` |
| `ObservabilityAgent` | AutonomousAgent | `application/ObservabilityAgent.java` |
| `OpsTasks` | Task constants | `application/OpsTasks.java` |
| `OpsWorkflow` | Workflow | `application/OpsWorkflow.java` |
| `ChangeRequestEntity` | EventSourcedEntity | `domain/ChangeRequestEntity.java` |
| `PipelineQueue` | EventSourcedEntity | `domain/PipelineQueue.java` |
| `HaltSignalEntity` | EventSourcedEntity | `domain/HaltSignalEntity.java` |
| `OpsView` | View | `application/OpsView.java` |
| `PipelineConsumer` | Consumer | `application/PipelineConsumer.java` |
| `PipelineSimulator` | TimedAction | `application/PipelineSimulator.java` |
| `HaltMonitor` | TimedAction | `application/HaltMonitor.java` |
| `OpsEndpoint` | HttpEndpoint | `api/OpsEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `infraStep`, `deployStep`, and `obsStep` each get 60s; `consolidateStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `infraStep`, `deployStep`, and `obsStep` run concurrently via `CompletionStage` zip over three futures, not sequential calls.
- **Idempotency:** the workflow id is the `changeId`. Re-delivery of the same `ChangeSubmitted` event resolves to the same workflow instance — no duplicate change request.
- **Degrade path (compensation):** if any specialist times out, `defaultStepRecovery` routes to `degradeStep`, which consolidates from whichever assessments returned and ends with `ChangeDegraded`. No infinite retry.
- **Halt path:** `HaltMonitor` reads `HaltSignalEntity`; active signals cause halt commands to in-flight workflows. The workflow checks for halt at each step boundary and calls `ChangeRequestEntity.halt` on detection.
- **Approval gate:** the workflow pauses after `ApprovalRequested` and waits for an external resume. The endpoint's approve/reject handlers call the workflow's resume with the operator's decision.
