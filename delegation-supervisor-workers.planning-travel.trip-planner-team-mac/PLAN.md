# PLAN — Trip Planner Multi-Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  TE[TripEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TRQ[TripRequestQueue<br/>EventSourcedEntity]:::ese
  TRC[TripRequestConsumer<br/>Consumer]:::con
  WF[TripWorkflow<br/>Workflow]:::wf
  SUP[TripSupervisor<br/>AutonomousAgent]:::ag
  DR[DestinationResearcher<br/>AutonomousAgent]:::ag
  IP[ItineraryPlanner<br/>AutonomousAgent]:::ag
  BA[BookingAgent<br/>AutonomousAgent]:::ag
  TRIP[TripEntity<br/>EventSourcedEntity]:::ese
  APR[ApprovalEntity<br/>EventSourcedEntity]:::ese
  VW[TripView<br/>View]:::vw
  SIM[TripRequestSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  TE -->|POST /trips| TRQ
  SIM -.->|every 90s| TRQ
  TRQ -.->|TripRequestSubmitted| TRC
  TRC -->|start workflow| WF
  WF -->|DECOMPOSE| SUP
  WF -->|RESEARCH_DESTINATION| DR
  WF -->|PLAN_ITINERARY| IP
  WF -->|PREPARE_BOOKINGS| BA
  WF -->|COMPILE| SUP
  WF -->|commands| TRIP
  WF -->|issueToken / recordDecision| APR
  TE -->|approve / reject| APR
  TRIP -.->|events| VW
  EV -.->|every 10m| VW
  EV -->|recordEval| TRIP
  TE -->|getAllTrips / SSE| VW
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
  participant TE as TripEndpoint
  participant TRQ as TripRequestQueue
  participant WF as TripWorkflow
  participant SUP as TripSupervisor
  participant DR as DestinationResearcher
  participant IP as ItineraryPlanner
  participant BA as BookingAgent
  participant TRIP as TripEntity
  participant APR as ApprovalEntity

  U->>TE: POST /api/trips {destination, dates, budget}
  TE->>TRQ: enqueueRequest
  TRQ-->>WF: TripRequestConsumer starts workflow
  WF->>TRIP: createTrip (PLANNING)
  WF->>SUP: DECOMPOSE -> TripDecomposition
  WF->>TRIP: status IN_PROGRESS
  par parallel fan-out
    WF->>DR: RESEARCH_DESTINATION -> DestinationNotes
  and
    WF->>IP: PLAN_ITINERARY -> Itinerary
  and
    WF->>BA: PREPARE_BOOKINGS -> List<BookingProposal>
  end
  Note over WF: join; if any step times out (60s) -> degradeStep
  WF->>SUP: COMPILE(notes, itinerary, proposals) -> TripPlan
  WF->>WF: guardrailStep checks proposals vs budget
  alt guardrail passes
    WF->>TRIP: requestApproval (AWAITING_APPROVAL)
    WF->>APR: issueToken
    Note over WF: workflow pauses; waits for approve/reject
    U->>TE: POST /api/trips/{id}/approve {token}
    TE->>APR: recordDecision(approve)
    WF->>TRIP: confirmTrip (CONFIRMED)
  else guardrail fails
    WF->>TRIP: block (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> IN_PROGRESS: TripDecomposition ready
  IN_PROGRESS --> AWAITING_APPROVAL: guardrail pass
  IN_PROGRESS --> DEGRADED: a worker timed out
  IN_PROGRESS --> BLOCKED: guardrail fail
  AWAITING_APPROVAL --> CONFIRMED: user approves
  AWAITING_APPROVAL --> REJECTED: user rejects
  DEGRADED --> [*]
  BLOCKED --> [*]
  REJECTED --> [*]
  CONFIRMED --> CONFIRMED: PlanEvalScored
  CONFIRMED --> [*]
```

## Entity model

```mermaid
erDiagram
  TRIP ||--o| DESTINATION_NOTES : has
  TRIP ||--o| ITINERARY : has
  TRIP ||--o{ BOOKING_PROPOSAL : has
  TRIP ||--o| TRIP_PLAN : produces
  TRIP_REQUEST_QUEUE ||--|| TRIP : seeds
  TRIP ||--|| APPROVAL_ENTITY : awaits
  TRIP {
    string tripId
    string destination
    enum status
    double budgetUsd
    int travellerCount
    int evalScore
    instant createdAt
  }
  TRIP_REQUEST_QUEUE {
    string tripId
    string destination
    string startDate
    string endDate
    double budgetUsd
    instant submittedAt
  }
  APPROVAL_ENTITY {
    string tripId
    string token
    string decision
    instant issuedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `TripSupervisor` | AutonomousAgent | `application/TripSupervisor.java` |
| `DestinationResearcher` | AutonomousAgent | `application/DestinationResearcher.java` |
| `ItineraryPlanner` | AutonomousAgent | `application/ItineraryPlanner.java` |
| `BookingAgent` | AutonomousAgent | `application/BookingAgent.java` |
| `TripTasks` | Task constants | `application/TripTasks.java` |
| `TripWorkflow` | Workflow | `application/TripWorkflow.java` |
| `TripEntity` | EventSourcedEntity | `domain/TripEntity.java` |
| `ApprovalEntity` | EventSourcedEntity | `domain/ApprovalEntity.java` |
| `TripRequestQueue` | EventSourcedEntity | `domain/TripRequestQueue.java` |
| `TripView` | View | `application/TripView.java` |
| `TripRequestConsumer` | Consumer | `application/TripRequestConsumer.java` |
| `TripRequestSimulator` | TimedAction | `application/TripRequestSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `TripEndpoint` | HttpEndpoint | `api/TripEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `researchStep`, `planStep`, and `bookStep` each get 60s; `compileStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `researchStep`, `planStep`, and `bookStep` run concurrently via `CompletionStage` allOf, not three sequential step calls.
- **Idempotency:** the workflow id is the `tripId`. Re-delivery of the same `TripRequestSubmitted` event resolves to the same workflow instance — no duplicate trip.
- **Degrade path (compensation):** if any worker times out, `defaultStepRecovery` routes to `degradeStep`, which compiles from whichever partial outputs exist and ends with `TripDegraded`. No infinite retry.
- **HITL pause:** `approvalStep` stores the token in `ApprovalEntity` and calls a suspend-style workflow step. The workflow resumes only when the endpoint delivers the decision — no polling.
- **Eval sampling:** `EvalSampler` reads `TripView.getAllTrips` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `CONFIRMED` trip lacking an `evalScore`.
