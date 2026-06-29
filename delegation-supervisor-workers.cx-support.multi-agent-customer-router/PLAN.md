# PLAN — Multi-Agent Customer Router

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  TE[TicketEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TicketQueueEntity<br/>EventSourcedEntity]:::ese
  TQC[TicketQueueConsumer<br/>Consumer]:::con
  WF[TicketWorkflow<br/>Workflow]:::wf
  RA[RouterAgent<br/>AutonomousAgent]:::ag
  BA[BillingAgent<br/>AutonomousAgent]:::ag
  TA[TechnicalAgent<br/>AutonomousAgent]:::ag
  AA[AccountAgent<br/>AutonomousAgent]:::ag
  EN[TicketEntity<br/>EventSourcedEntity]:::ese
  VW[TicketView<br/>View]:::vw
  SIM[TicketSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  TE -->|POST /tickets| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TicketSubmitted| TQC
  TQC -->|start workflow| WF
  WF -->|ROUTE| RA
  WF -->|guardrail check| WF
  WF -->|RESOLVE_BILLING| BA
  WF -->|RESOLVE_TECHNICAL| TA
  WF -->|RESOLVE_ACCOUNT| AA
  WF -->|commands| EN
  EN -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| EN
  TE -->|getAllTickets / SSE| VW
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

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant TE as TicketEndpoint
  participant TQ as TicketQueueEntity
  participant WF as TicketWorkflow
  participant RA as RouterAgent
  participant SP as Specialist (Billing/Technical/Account)
  participant EN as TicketEntity

  U->>TE: POST /api/tickets {subject, body}
  TE->>TQ: submitTicket
  TQ-->>WF: TicketQueueConsumer starts workflow
  WF->>EN: openTicket (OPEN)
  WF->>EN: routeTicket (ROUTING)
  WF->>RA: ROUTE -> RoutingDecision
  WF->>WF: guardrailStep — confidence check + LLM judge
  alt guardrail passes
    WF->>EN: acceptTicket (IN_PROGRESS)
    WF->>SP: RESOLVE_{BILLING|TECHNICAL|ACCOUNT} -> Resolution
    WF->>EN: resolveTicket (RESOLVED)
  else guardrail fails
    WF->>EN: flagForReview (REVIEW_REQUIRED)
  end
  Note over WF: if specialist times out (90s) -> escalateTicket (ESCALATED)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> ROUTING: workflow started
  ROUTING --> IN_PROGRESS: guardrail passes
  ROUTING --> REVIEW_REQUIRED: guardrail fails
  IN_PROGRESS --> RESOLVED: specialist returns Resolution
  IN_PROGRESS --> ESCALATED: specialist timed out
  REVIEW_REQUIRED --> [*]
  ESCALATED --> [*]
  RESOLVED --> RESOLVED: ResolutionEvalScored
  RESOLVED --> [*]
```

## Entity model

```mermaid
erDiagram
  TICKET ||--o| ROUTING_DECISION : classified-by
  TICKET ||--o| RESOLUTION : has
  TICKET_QUEUE ||--|| TICKET : seeds
  TICKET {
    string ticketId
    string subject
    enum status
    enum category
    int evalScore
    instant createdAt
  }
  TICKET_QUEUE {
    string ticketId
    string subject
    string submittedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `RouterAgent` | AutonomousAgent | `application/RouterAgent.java` |
| `BillingAgent` | AutonomousAgent | `application/BillingAgent.java` |
| `TechnicalAgent` | AutonomousAgent | `application/TechnicalAgent.java` |
| `AccountAgent` | AutonomousAgent | `application/AccountAgent.java` |
| `TicketTasks` | Task constants | `application/TicketTasks.java` |
| `TicketWorkflow` | Workflow | `application/TicketWorkflow.java` |
| `TicketEntity` | EventSourcedEntity | `domain/TicketEntity.java` |
| `TicketQueueEntity` | EventSourcedEntity | `domain/TicketQueueEntity.java` |
| `TicketView` | View | `application/TicketView.java` |
| `TicketQueueConsumer` | Consumer | `application/TicketQueueConsumer.java` |
| `TicketSimulator` | TimedAction | `application/TicketSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `TicketEndpoint` | HttpEndpoint | `api/TicketEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `routeStep` gets 30s; `delegateStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Sequential delegation:** unlike a fan-out pattern, only one specialist runs per ticket. The branch is selected deterministically from `RoutingDecision.category`.
- **Guardrail gate:** `guardrailStep` runs between routing and delegation. A REVIEW_REQUIRED outcome ends the workflow without calling any specialist — no partial specialist execution.
- **Idempotency:** the workflow id is the `ticketId`. Re-delivery of the same `TicketSubmitted` event resolves to the same workflow instance — no duplicate ticket.
- **Escalation path (compensation):** if the specialist times out, `defaultStepRecovery` routes to `escalateStep`, which calls `TicketEntity.escalateTicket` and ends with `TicketEscalated`. No infinite retry.
- **Eval sampling:** `EvalSampler` reads `TicketView.getAllTickets` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `RESOLVED` ticket lacking an `evalScore`.
