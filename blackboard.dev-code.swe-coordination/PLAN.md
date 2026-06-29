# PLAN — blackboard-swe-coordination

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#1f1900,stroke:#C9A227,color:#C9A227;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Planner[PlannerAgent]:::agent
  Arch[ArchitectAgent]:::agent
  Backend[BackendDevAgent]:::agent
  Frontend[FrontendDevAgent]:::agent
  Reviewer[ReviewerAgent]:::agent
  Security[SecurityAnalystAgent]:::agent
  Tester[TesterAgent]:::agent
  Doc[DocWriterAgent]:::agent
  Integrator[IntegrationPlannerAgent]:::agent

  Controller[ControllerWorkflow]:::wf
  Signoff[SignoffWorkflow]:::wf

  Board[BlackboardEntity]:::ese
  Ticket[TicketEntity]:::ese
  SignoffE[SignoffEntity]:::ese
  Queue[TicketQueue]:::ese

  View[BlackboardView]:::view
  Consumer[TicketConsumer]:::cons
  Sim[TicketSimulator]:::ta
  Monitor[StuckBoardMonitor]:::ta
  API[BoardEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit ticket| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|create + start| Ticket
  Consumer -->|start workflow| Controller
  Controller -->|call| Planner
  Controller -->|call| Arch
  Controller -->|call| Backend
  Controller -->|call| Frontend
  Controller -->|call| Reviewer
  Controller -->|call| Security
  Controller -->|call| Tester
  Controller -->|call| Doc
  Controller -->|call| Integrator
  Controller -->|write contributions| Board
  Controller -->|start| Signoff
  Signoff -->|record request| SignoffE
  Signoff -->|approve / reject| Board
  Board -.->|projects| View
  Monitor -.->|every 60s| Board
  API -->|signoff decision| SignoffE
  API -->|query / SSE| View
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. The nine specialist agents are all distinct `AutonomousAgent` classes, each with its own `BoardTasks` constant. The `ControllerWorkflow` is the only component that calls agents; specialists never communicate directly.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as BoardEndpoint
  participant Q as TicketQueue
  participant C as TicketConsumer
  participant CW as ControllerWorkflow
  participant BB as BlackboardEntity
  participant P as PlannerAgent
  participant AD as ArchitectAgent
  participant BD as BackendDevAgent
  participant FD as FrontendDevAgent
  participant SW as SignoffWorkflow
  participant V as BlackboardView

  U->>API: POST /api/tickets {title, description}
  API->>Q: submitTicket(brief)
  API-->>U: 202 {ticketId}
  Q->>C: TicketSubmitted
  C->>CW: start({ticketId})
  CW->>BB: receiveTicket
  BB-->>V: INTAKE row
  CW->>P: plan(ticket)
  P-->>CW: TaskBreakdown
  CW->>BB: writePlan(breakdown)
  BB-->>V: PLANNED row
  CW->>AD: arch(breakdown)
  AD-->>CW: ArchDecision
  CW->>BB: writeArch(decision)
  Note over CW: invoke BackendDevAgent + FrontendDevAgent in parallel
  CW->>BD: code(arch, tasks)
  Note over BD: CodeValidator runs before BlackboardEntity persists event
  BD-->>CW: CodeArtifact[backend]
  CW->>FD: code(arch, tasks)
  FD-->>CW: CodeArtifact[frontend]
  CW->>BB: writeBackendCode + writeFrontendCode
  BB-->>V: CODED row
  Note over CW: reviewer + security in parallel, then tester + doc in parallel
  CW->>BB: writeReview + writeSecurity + writeTests + writeDocs
  CW->>BB: writeIntegrationPlan
  BB-->>V: INTEGRATION_PLANNED row
  CW->>SW: start(ticketId)
  SW->>BB: awaitSignoff
  BB-->>V: AWAITING_SIGNOFF row
  V-->>U: SSE update
  U->>API: POST /api/signoff/{ticketId} {approved:true}
  SW->>BB: approve
  BB-->>V: MERGED row
  V-->>U: SSE update
```

## State machine — `BlackboardEntity`

```mermaid
stateDiagram-v2
  [*] --> INTAKE
  INTAKE --> PLANNED: writePlan
  PLANNED --> ARCHITECTED: writeArch
  ARCHITECTED --> CODED: writeBackendCode + writeFrontendCode (both present)
  CODED --> REVIEWED: writeReview + writeSecurity (both present)
  REVIEWED --> VERIFIED: writeTests + writeDocs (both present)
  VERIFIED --> INTEGRATION_PLANNED: writeIntegrationPlan
  INTEGRATION_PLANNED --> AWAITING_SIGNOFF: awaitSignoff
  AWAITING_SIGNOFF --> MERGED: approve (lead-engineer HITL)
  AWAITING_SIGNOFF --> IN_REVIEW: reject
  IN_REVIEW --> REVIEWED: writeReview (re-review cycle)
  MERGED --> [*]
```

## Entity model

```mermaid
erDiagram
  BlackboardEntity ||--o{ TicketReceived : emits
  BlackboardEntity ||--o{ PlanWritten : emits
  BlackboardEntity ||--o{ ArchWritten : emits
  BlackboardEntity ||--o{ BackendCodeWritten : emits
  BlackboardEntity ||--o{ FrontendCodeWritten : emits
  BlackboardEntity ||--o{ ReviewWritten : emits
  BlackboardEntity ||--o{ SecurityWritten : emits
  BlackboardEntity ||--o{ TestsWritten : emits
  BlackboardEntity ||--o{ DocsWritten : emits
  BlackboardEntity ||--o{ IntegrationPlanWritten : emits
  BlackboardEntity ||--o{ SignoffRequested : emits
  BlackboardEntity ||--o{ SignoffApproved : emits
  BlackboardEntity ||--o{ SignoffRejected : emits
  BlackboardView }o--|| BlackboardEntity : projects
  TicketEntity ||--o{ TicketCreated : emits
  TicketEntity ||--o{ TicketCompleted : emits
  SignoffEntity ||--o{ SignoffRequestCreated : emits
  SignoffEntity ||--o{ SignoffDecisionRecorded : emits
  TicketQueue ||--o{ TicketSubmitted : emits
  TicketConsumer }o--|| TicketQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ArchitectAgent` | `application/ArchitectAgent.java` |
| `BackendDevAgent` | `application/BackendDevAgent.java` |
| `FrontendDevAgent` | `application/FrontendDevAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `SecurityAnalystAgent` | `application/SecurityAnalystAgent.java` |
| `TesterAgent` | `application/TesterAgent.java` |
| `DocWriterAgent` | `application/DocWriterAgent.java` |
| `IntegrationPlannerAgent` | `application/IntegrationPlannerAgent.java` |
| `BoardTasks` | `application/BoardTasks.java` |
| `CodeValidator` | `application/CodeValidator.java` |
| `ControllerWorkflow` | `application/ControllerWorkflow.java` |
| `SignoffWorkflow` | `application/SignoffWorkflow.java` |
| `BlackboardEntity` | `application/BlackboardEntity.java` (state in `domain/Blackboard.java`, events in `domain/BlackboardEvent.java`) |
| `TicketEntity` | `application/TicketEntity.java` (state in `domain/Ticket.java`) |
| `SignoffEntity` | `application/SignoffEntity.java` |
| `TicketQueue` | `application/TicketQueue.java` |
| `BlackboardView` | `application/BlackboardView.java` |
| `TicketConsumer` | `application/TicketConsumer.java` |
| `TicketSimulator` | `application/TicketSimulator.java` |
| `StuckBoardMonitor` | `application/StuckBoardMonitor.java` |
| `BoardEndpoint` | `api/BoardEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **9 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Single blackboard writer.** `BlackboardEntity` is a single-writer EventSourcedEntity. All specialist writes are serialised through it — no two specialists can corrupt the board by writing simultaneously.
- **Before-state-write guardrail.** The `BlackboardEntity.writeBackendCode` and `writeBackendCode` command handlers call `CodeValidator` before emitting the event. A failing validation returns an error reply; the controller logs it and retries the agent once with the validation message.
- **Parallel agent invocation.** `ControllerWorkflow.codeStep` invokes `BackendDevAgent` and `FrontendDevAgent` concurrently; `reviewStep` invokes `ReviewerAgent` and `SecurityAnalystAgent` concurrently; `verifyStep` invokes `TesterAgent` and `DocWriterAgent` concurrently. Each parallel pair waits for both results before the step completes.
- **HITL pause.** `SignoffWorkflow.waitStep` pauses the workflow durably. The board stays in `AWAITING_SIGNOFF` indefinitely until a human POSTs a decision. No polling, no timeout — the decision is a durable external event.
- **Rejection cycle.** On rejection, the controller re-enters from `reviewStep`, allowing the reviewer and security analyst to re-examine the board with any new context. This avoids re-running the planner and architect.
- **Stuck detection.** `StuckBoardMonitor` uses a wall-clock check against `createdAt` on the board row. It does not force state transitions — it only marks `stuckAlert` so an operator can investigate.
- **Step timeouts.** All agent-calling steps set explicit timeouts (90–120 s) to prevent the default 5 s timeout from expiring mid-LLM-call (Lesson 4).
- **Dependency ordering.** Stages encode the dependency graph: backend and frontend code cannot start until architecture is written; review cannot start until both code layers are present. No runtime dependency check is needed — the stage machine enforces it.
