# PLAN — Contract-Net Satellite Tasker

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  OE[ObservationEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  RQ[RequestQueue<br/>EventSourcedEntity]:::ese
  ORC[ObservationRequestConsumer<br/>Consumer]:::con
  WF[ObservationAuction<br/>Workflow]:::wf
  AC[AuctionCoordinator<br/>AutonomousAgent]:::ag
  SB[SatelliteBidder<br/>AutonomousAgent]:::ag
  ORE[ObservationRequestEntity<br/>EventSourcedEntity]:::ese
  SE[SatelliteEntity<br/>EventSourcedEntity]:::ese
  OV[ObservationView<br/>View]:::vw
  SV[SatelliteView<br/>View]:::vw
  SIM[ObservationRequestSimulator<br/>TimedAction]:::ta
  TCS[TaskCompletionSimulator<br/>TimedAction]:::ta
  FES[FleetEvalSampler<br/>TimedAction]:::ta

  OE -->|POST /observations| RQ
  SIM -.->|every 75s| RQ
  RQ -.->|ObservationSubmitted| ORC
  ORC -->|start workflow| WF
  WF -->|BID per satellite| SB
  WF -->|AWARD_TASK| AC
  WF -->|commands| ORE
  WF -->|commands| SE
  ORE -.->|events| OV
  SE -.->|events| SV
  TCS -.->|every 90s| OV
  TCS -->|completeTask| ORE
  FES -.->|every 10m| OV
  FES -->|recordFleetEval| ORE
  OE -->|getAllRequests / SSE| OV
  OE -->|getOperationalSatellites| SV
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

Solid arrows: synchronous commands. Dashed arrows: event subscriptions and scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant OE as ObservationEndpoint
  participant RQ as RequestQueue
  participant WF as ObservationAuction
  participant SB as SatelliteBidder
  participant AC as AuctionCoordinator
  participant ORE as ObservationRequestEntity
  participant SE as SatelliteEntity

  U->>OE: POST /api/observations {target, instrument}
  OE->>RQ: enqueueObservation
  RQ-->>WF: ObservationRequestConsumer starts workflow
  WF->>ORE: openRequest (OPEN)
  WF->>ORE: startBidding (BIDDING)
  par parallel bid fan-out
    WF->>SB: BID(LEO-1, request) -> Bid
    Note over WF: guardrail checks Bid vs ephemeris
    WF->>ORE: receiveBid / rejectBid
  and
    WF->>SB: BID(LEO-2, request) -> Bid
    Note over WF: guardrail checks Bid vs ephemeris
    WF->>ORE: receiveBid / rejectBid
  and
    WF->>SB: BID(LEO-N, request) -> Bid
    Note over WF: guardrail checks Bid vs ephemeris
    WF->>ORE: receiveBid / rejectBid
  end
  Note over WF: collectStep merges accepted bids
  alt bids received
    WF->>AC: AWARD_TASK(bidBundle) -> AwardDecision
    WF->>ORE: awardTask (AWARDED)
    WF->>SE: assignSatellite
  else no feasible bids
    WF->>ORE: unallocate (UNALLOCATED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> BIDDING: startBidding
  BIDDING --> AWARDED: AwardDecision issued
  BIDDING --> UNALLOCATED: deadline; no feasible bids
  AWARDED --> COMPLETED: TaskCompleted
  AWARDED --> AWARDED: FleetEvalScored
  COMPLETED --> [*]
  UNALLOCATED --> [*]
```

## Entity model

```mermaid
erDiagram
  OBSERVATION_REQUEST ||--o{ BID_BUNDLE : collects
  OBSERVATION_REQUEST ||--o| AWARD_DECISION : produces
  REQUEST_QUEUE ||--|| OBSERVATION_REQUEST : seeds
  SATELLITE ||--o{ OBSERVATION_REQUEST : awarded
  OBSERVATION_REQUEST {
    string requestId
    string target
    string instrument
    enum status
    double fleetCompletionRate
    instant createdAt
  }
  SATELLITE {
    string satelliteId
    string name
    string orbitType
    boolean operational
    int completedCount
  }
  REQUEST_QUEUE {
    string requestId
    string target
    string instrument
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `AuctionCoordinator` | AutonomousAgent | `application/AuctionCoordinator.java` |
| `SatelliteBidder` | AutonomousAgent | `application/SatelliteBidder.java` |
| `SatelliteTasks` | Task constants | `application/SatelliteTasks.java` |
| `ObservationAuction` | Workflow | `application/ObservationAuction.java` |
| `ObservationRequestEntity` | EventSourcedEntity | `domain/ObservationRequestEntity.java` |
| `SatelliteEntity` | EventSourcedEntity | `domain/SatelliteEntity.java` |
| `RequestQueue` | EventSourcedEntity | `domain/RequestQueue.java` |
| `ObservationView` | View | `application/ObservationView.java` |
| `SatelliteView` | View | `application/SatelliteView.java` |
| `ObservationRequestConsumer` | Consumer | `application/ObservationRequestConsumer.java` |
| `ObservationRequestSimulator` | TimedAction | `application/ObservationRequestSimulator.java` |
| `TaskCompletionSimulator` | TimedAction | `application/TaskCompletionSimulator.java` |
| `FleetEvalSampler` | TimedAction | `application/FleetEvalSampler.java` |
| `ObservationEndpoint` | HttpEndpoint | `api/ObservationEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** each `bidStep` gets 30s; `awardStep` gets 45s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel bid fan-out:** all satellite bid steps run concurrently via `CompletionStage` zip over the fleet list, not sequential step calls.
- **Idempotency:** the workflow id is the `requestId`. Re-delivery of the same `ObservationSubmitted` event resolves to the same workflow instance — no duplicate auction.
- **Guardrail placement:** the bid-feasibility guardrail runs inside the workflow immediately after each `SatelliteBidder` returns, before the bid is added to the pool. Infeasible bids never reach the AuctionCoordinator.
- **Empty-pool path:** if `collectStep` finds zero accepted bids it transitions to `unallocatedStep` without calling the AuctionCoordinator.
- **Fleet eval:** `FleetEvalSampler` reads `ObservationView.getAllRequests` (no enum WHERE clause) and filters client-side for AWARDED and COMPLETED counts.
