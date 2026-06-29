# PLAN — trip-planner-team

Architectural sketch for the delegation-supervisor-workers × planning-travel cell. All four mermaid diagrams + the component table are required. The generated system renders these on the Architecture tab with the Lesson 24 state-label CSS overrides.

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

  TripEndpoint[TripEndpoint]:::ep
  TravelSourceEndpoint[TravelSourceEndpoint]:::ep
  AppEndpoint[AppEndpoint]:::ep
  Sim[TripRequestSimulator]:::ta
  Queue[InboundRequestQueue]:::ese
  Consumer[TripRequestConsumer]:::cons
  WF[TripPlanningWorkflow]:::wf
  Local[LocalExpertAgent]:::agent
  Itin[ItineraryAgent]:::agent
  Log[LogisticsAgent]:::agent
  Trip[TripEntity]:::ese
  View[TripsView]:::view

  TripEndpoint -->|start| WF
  Sim -. drip .-> Queue
  Queue -. events .-> Consumer
  Consumer -->|start| WF
  WF -->|gather| Local
  WF -->|draft| Itin
  WF -->|plan| Log
  Local -->|search/scrape tools| TravelSourceEndpoint
  WF -->|record| Trip
  Trip -. events .-> View
  TripEndpoint -->|query/SSE| View
  AppEndpoint -->|static| TripEndpoint
```

Solid arrows are synchronous commands, dashed arrows are event subscriptions, dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor User
  participant EP as TripEndpoint
  participant WF as TripPlanningWorkflow
  participant LA as LocalExpertAgent
  participant TS as TravelSourceEndpoint
  participant IA as ItineraryAgent
  participant LG as LogisticsAgent
  participant TE as TripEntity

  User->>EP: POST /api/trips
  EP->>WF: start(tripId)
  WF->>LA: gather(destination)
  LA->>TS: search / scrape
  Note over LA,TS: before-tool-call guard (G1)<br/>blocks non-allowlisted scrape URLs
  LA-->>WF: Insights
  WF->>TE: recordInsights
  WF->>IA: draft(prefs, insights)
  Note over IA: before-agent-response guard (G2)<br/>itinerary must be grounded in insights
  IA-->>WF: Itinerary
  WF->>TE: recordItinerary
  WF->>LG: plan(itinerary)
  LG-->>WF: Logistics
  WF->>TE: recordLogistics, complete
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> GATHERING_INSIGHTS
  GATHERING_INSIGHTS --> PLANNING_ITINERARY: InsightsGathered
  PLANNING_ITINERARY --> PLANNING_LOGISTICS: ItineraryDrafted
  PLANNING_LOGISTICS --> COMPLETED: LogisticsPlanned + TripCompleted
  GATHERING_INSIGHTS --> FAILED: TripFailed
  PLANNING_ITINERARY --> FAILED: TripFailed
  PLANNING_LOGISTICS --> FAILED: TripFailed
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TRIP_ENTITY ||--o{ TRIP_EVENT : emits
  TRIP_ENTITY ||--|| TRIPS_VIEW : projects
  INBOUND_QUEUE ||--o{ QUEUE_EVENT : emits

  TRIP_ENTITY {
    string id
    string destination
    string preferences
    int days
    enum status
    optional localInsights
    optional itinerary
    optional logistics
    optional finalPlan
  }
  TRIP_EVENT {
    string kind "TripRequested|InsightsGathered|ItineraryDrafted|LogisticsPlanned|TripCompleted|TripFailed"
  }
  TRIPS_VIEW {
    string id
    enum status
    string destination
  }
  QUEUE_EVENT {
    string kind "TripRequestQueued"
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| `TripPlanningWorkflow` | `application/TripPlanningWorkflow.java` |
| `LocalExpertAgent` | `application/LocalExpertAgent.java` |
| `ItineraryAgent` | `application/ItineraryAgent.java` |
| `LogisticsAgent` | `application/LogisticsAgent.java` |
| `TripEntity` | `application/TripEntity.java` |
| `InboundRequestQueue` | `application/InboundRequestQueue.java` |
| `TripsView` | `application/TripsView.java` |
| `TripRequestConsumer` | `application/TripRequestConsumer.java` |
| `TripRequestSimulator` | `application/TripRequestSimulator.java` |
| `TripEndpoint` | `api/TripEndpoint.java` |
| `TravelSourceEndpoint` | `api/TravelSourceEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TripPlan`, events, `TripStatus` | `domain/` |

## Concurrency notes

- Every agent-calling workflow step sets `stepTimeout(60s)` (Lesson 4); LLM calls exceed the 5s default.
- `defaultStepRecovery(maxRetries(2).failoverTo(TripPlanningWorkflow::fail))` so a worker failure ends the trip in `FAILED` rather than retrying forever.
- The workflow id is the `tripId`; restarting the same id is idempotent because each step writes a distinct event the entity applier folds once.
- The G2 grounding rejection triggers a step retry (bounded by maxRetries); a persistently ungrounded itinerary fails the trip rather than looping.
- No saga/compensation: the simulated publish surface has no irreversible side effect, so a failed trip needs no rollback beyond the `TripFailed` event.
