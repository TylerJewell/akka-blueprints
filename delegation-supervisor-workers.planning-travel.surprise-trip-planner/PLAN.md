# PLAN — surprise-trip-planner

Architectural sketch for the delegation-supervisor-workers x planning-travel cell. All four mermaid diagrams and the component table are below.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  SIM[RequestSimulator]:::ta -.->|enqueue| IQ[InboundRequestQueue]:::ese
  TEP[TripEndpoint]:::ep -->|enqueue| IQ
  IQ -.->|event| CON[TripRequestConsumer]:::cons
  CON -->|start| WF[TripWorkflow]:::wf
  WF -->|delegate| DEST[DestinationAgent]:::agent
  WF -->|delegate| LOG[LogisticsAgent]:::agent
  WF -->|delegate| ACT[ActivitiesAgent]:::agent
  WF -->|assemble| SUP[TripSupervisorAgent]:::agent
  WF -->|record| TE[TripEntity]:::ese
  TE -.->|events| TV[TripsView]:::view
  TV --> TEP
  MON[StalledTripMonitor]:::ta -.->|escalate| TE
  AEP[AppEndpoint]:::ep
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as Traveler
  participant TEP as TripEndpoint
  participant IQ as InboundRequestQueue
  participant CON as TripRequestConsumer
  participant WF as TripWorkflow
  participant DEST as DestinationAgent
  participant LOG as LogisticsAgent
  participant ACT as ActivitiesAgent
  participant SUP as TripSupervisorAgent
  participant TE as TripEntity

  U->>TEP: POST /api/trips (PreferenceSet)
  TEP->>IQ: enqueueRequest
  IQ-->>CON: InboundRequestQueued
  CON->>WF: start(tripId)
  WF->>DEST: runSingleTask(CHOOSE_DESTINATION)
  DEST-->>WF: Destination
  WF->>TE: recordDestination
  Note over WF: status RESEARCHED
  WF->>LOG: runSingleTask(PLAN_LOGISTICS)
  LOG-->>WF: Logistics
  WF->>ACT: runSingleTask(PLAN_ACTIVITIES)
  ACT-->>WF: Activities
  WF->>SUP: runSingleTask(ASSEMBLE)
  Note over SUP: before-tool-call (G1) + before-agent-response (G2) guardrails fire
  SUP-->>WF: Itinerary
  WF->>TE: recordItinerary
  Note over WF: status READY
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> RESEARCHED: DestinationChosen
  RESEARCHED --> READY: ItineraryAssembled
  REQUESTED --> ESCALATED: stalled > 3 min
  RESEARCHED --> ESCALATED: stalled > 3 min
  REQUESTED --> FAILED: TripFailed
  RESEARCHED --> FAILED: TripFailed
  READY --> [*]
  ESCALATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  INBOUND_REQUEST_QUEUE ||--o{ TRIP : "spawns"
  TRIP ||--o{ TRIP_EVENT : "emits"
  TRIP ||--|| TRIPS_VIEW : "projects to"
  TRIP {
    string id
    PreferenceSet preferences
    TripStatus status
    Optional destinationName
    Optional logistics
    Optional activities
    Optional groundingNotes
    Optional blockedToolNote
  }
  TRIP_EVENT {
    string type "TripRequested|DestinationChosen|LogisticsPlanned|ActivitiesPlanned|ItineraryAssembled|ToolCallBlocked|TripEscalated|TripFailed"
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| TripSupervisorAgent | `application/TripSupervisorAgent.java` |
| DestinationAgent | `application/DestinationAgent.java` |
| LogisticsAgent | `application/LogisticsAgent.java` |
| ActivitiesAgent | `application/ActivitiesAgent.java` |
| SupervisorTasks | `application/SupervisorTasks.java` |
| TripWorkflow | `application/TripWorkflow.java` |
| TripEntity | `domain/TripEntity.java` |
| InboundRequestQueue | `domain/InboundRequestQueue.java` |
| TripsView | `application/TripsView.java` |
| TripRequestConsumer | `application/TripRequestConsumer.java` |
| RequestSimulator | `application/RequestSimulator.java` |
| StalledTripMonitor | `application/StalledTripMonitor.java` |
| TripEndpoint | `api/TripEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts:** every agent-calling workflow step sets `stepTimeout(60s)` (Lesson 4); the default 5s timeout fails on LLM calls.
- **Idempotency:** `TripWorkflow` is keyed by a fresh UUID per inbound request; the consumer derives the workflow id from the queued event so replays do not double-start.
- **Failover:** `defaultStepRecovery(maxRetries(2).failoverTo(error))` routes exhausted steps to a terminal `FAILED` state via `TripEntity.markFailed`.
- **Compensation:** no irreversible external effect; assembly happens only after all three worker results are present, so a mid-plan failure leaves the trip in `REQUESTED`/`RESEARCHED` for the monitor to escalate rather than a partially-published itinerary.
- **Guardrail placement:** G1 runs before each worker's external-tool call; G2 runs before the supervisor's assembled response is persisted.
