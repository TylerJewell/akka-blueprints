# PLAN — aws-ops-assistant

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef stub fill:#1a1a2e,stroke:#6a5acd,color:#a393eb;

  API[OpsEndpoint]:::ep
  Entity[OpsRequestEntity]:::ese
  ConfCons[ConfirmationConsumer]:::cons
  WF[OpsWorkflow]:::wf
  Agent[AwsOpsAgent]:::agent
  Guard[ActionGuardrail]:::guard
  Stubs[AwsMcpStubs]:::stub
  View[OpsView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|confirm / halt| Entity
  API -->|start workflow| WF
  Entity -.->|ConfirmationRequested| ConfCons
  ConfCons -->|recordConfirmationDelivered| Entity
  WF -->|planStep call| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|pass / reject| Agent
  Agent -->|MCP tool calls| Stubs
  Stubs -->|response snippets| Agent
  Agent -->|OperationReport| WF
  WF -->|requestConfirmation| Entity
  WF -->|pollConfirmation| Entity
  WF -->|recordReport| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J2 (mutating request with HITL confirmation)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as OpsEndpoint
  participant E as OpsRequestEntity
  participant W as OpsWorkflow
  participant A as AwsOpsAgent
  participant G as ActionGuardrail
  participant MCP as AwsMcpStubs

  U->>API: POST /api/ops-requests (scope=MUTATING)
  API->>E: submit(request)
  E-->>API: { opsRequestId }
  API->>W: start(opsRequestId)
  W->>E: startPlanning
  W->>A: runSingleTask(requestText + context)
  A->>G: before-tool-call(ModifyInstanceAttribute)
  G-->>A: pass (EC2 is allowed)
  A->>MCP: EC2.DescribeInstances (READ — no confirmation)
  MCP-->>A: instance list snippet
  Note over A,W: Agent plans mutating call; pauses for HITL
  W->>E: requestConfirmation(ConfirmationRequest)
  E-.->>U: SSE event(AWAITING_CONFIRMATION)
  U->>API: POST /confirm { actionId, outcome=APPROVED }
  API->>E: receiveConfirmation(decision)
  E-.->>W: ConfirmationReceived
  W->>A: resume with APPROVED signal
  A->>G: before-tool-call(ModifyInstanceAttribute)
  G-->>A: pass
  A->>MCP: EC2.ModifyInstanceAttribute
  MCP-->>A: synthetic 200-OK
  A-->>W: OperationReport{COMPLETED}
  W->>E: recordReport(report)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `OpsRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PLANNING: PlanningStarted
  PLANNING --> AWAITING_CONFIRMATION: ConfirmationRequested (mutating action)
  PLANNING --> EXECUTING: ExecutionStarted (read-only only)
  AWAITING_CONFIRMATION --> EXECUTING: ConfirmationReceived (APPROVED or DECLINED)
  AWAITING_CONFIRMATION --> HALTED: RequestHalted
  EXECUTING --> AWAITING_CONFIRMATION: next mutating action
  EXECUTING --> COMPLETED: ReportRecorded (happy)
  EXECUTING --> HALTED: RequestHalted
  EXECUTING --> FAILED: RequestFailed (agent error)
  COMPLETED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  OpsRequestEntity ||--o{ RequestSubmitted : emits
  OpsRequestEntity ||--o{ PlanningStarted : emits
  OpsRequestEntity ||--o{ ConfirmationRequested : emits
  OpsRequestEntity ||--o{ ConfirmationDelivered : emits
  OpsRequestEntity ||--o{ ConfirmationReceived : emits
  OpsRequestEntity ||--o{ ExecutionStarted : emits
  OpsRequestEntity ||--o{ ReportRecorded : emits
  OpsRequestEntity ||--o{ RequestHalted : emits
  OpsRequestEntity ||--o{ RequestFailed : emits
  OpsView }o--|| OpsRequestEntity : projects
  ConfirmationConsumer }o--|| OpsRequestEntity : subscribes
  OpsWorkflow }o--|| OpsRequestEntity : reads-and-writes
  AwsOpsAgent ||--o{ OperationReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OpsEndpoint` | `api/OpsEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `OpsRequestEntity` | `application/OpsRequestEntity.java` (state in `domain/OpsRequestState.java`, events in `domain/OpsRequestEvent.java`) |
| `ConfirmationConsumer` | `application/ConfirmationConsumer.java` |
| `OpsWorkflow` | `application/OpsWorkflow.java` |
| `AwsOpsAgent` | `application/AwsOpsAgent.java` (tasks in `application/OpsTasks.java`) |
| `ActionGuardrail` | `application/ActionGuardrail.java` |
| `AwsMcpStubs` | `application/AwsMcpStubs.java` |
| `OpsView` | `application/OpsView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `planStep` 15 s, `confirmStep` 600 s (10-minute human window), `executeStep` 120 s, `reportStep` 10 s, `error` 10 s. Default step recovery `maxRetries(1).failoverTo(OpsWorkflow::error)`. The 120 s on `executeStep` accommodates LLM latency across multi-action requests (Lesson 4).
- **Idempotency**: every workflow uses `"ops-" + opsRequestId` as the workflow id; `ConfirmationConsumer` is allowed to redeliver `ConfirmationRequested` because `OpsRequestEntity.recordConfirmationDelivered` is event-version-guarded — a second delivery for the same `actionId` is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"ops-agent-" + opsRequestId`, giving each request its own conversation context. `maxIterationsPerTask(12)` accommodates up to 12 action-plan-and-execute turns within a single request.
- **Guardrail on every tool call**: `ActionGuardrail` fires before each individual MCP tool call, not once per request. A request that mixes allowed and disallowed services will have some calls pass and others blocked — all recorded in the final `OperationReport`.
- **Halt is checked between actions**: the workflow reads `OpsRequestEntity.getRequest` between every action dispatch and checks `status == HALTED`. A halt during an in-flight single MCP call allows that call to finish (one atomic operation) before the workflow observes the flag and stops.
- **HITL loop**: the `confirmStep` can iterate multiple times within a single workflow execution — once per mutating action in the plan. The 600 s timeout applies per confirmation window, not to the total sum.
- **No saga / no compensation**: AWS MCP stub calls are fire-and-forget; if an action fails the workflow records `FAILED` in the `ActionResult` and continues to the next action. There is no rollback — a deployer adding real AWS credentials must decide their own rollback strategy outside this blueprint.
