# PLAN — nexshift

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ScheduleEndpoint]:::ep
  Entity[ScheduleEntity]:::ese
  WF[ScheduleWorkflow]:::wf
  Agent[NexShiftAgent]:::agent
  Guard[AssignmentGuardrail]:::guard
  View[ScheduleView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|buildRosterStep read| Entity
  WF -->|markBuilding| Entity
  WF -->|scheduleStep runSingleTask| Agent
  Agent -.->|before-tool-call AssignShift| Guard
  Guard -->|accept → recordAssignment| Entity
  Guard -.->|reject → tool error| Agent
  Agent -->|ScheduleDraft| WF
  WF -->|recordDraft| Entity
  API -->|confirm| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Manager (UI)
  participant API as ScheduleEndpoint
  participant E as ScheduleEntity
  participant W as ScheduleWorkflow
  participant A as NexShiftAgent
  participant G as AssignmentGuardrail

  U->>API: POST /api/schedules
  API->>E: submit(request)
  API->>W: start(scheduleId)
  E-->>API: { scheduleId }
  W->>E: markBuilding
  W->>E: read getSchedule
  E-->>W: ScheduleRequest (shifts + roster)
  W->>E: markScheduling
  W->>A: runSingleTask(roster + shifts)
  loop for each open shift
    A->>G: before-tool-call AssignShift(shiftId, employeeId)
    G-->>A: accept
    G->>E: recordAssignment(assignment)
  end
  A-->>W: ScheduleDraft
  W->>E: recordDraft(draft)
  E-.->>U: SSE event(DRAFT)
  U->>API: PUT /api/schedules/{id}/confirm
  API->>E: confirm
  E-.->>U: SSE event(CONFIRMED)
```

## State machine — `ScheduleEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> BUILDING: RosterBuilt
  BUILDING --> SCHEDULING: SchedulingStarted
  SCHEDULING --> DRAFT: DraftRecorded
  DRAFT --> CONFIRMED: ScheduleConfirmed
  SCHEDULING --> FAILED: ScheduleFailed (agent timeout or budget exhausted)
  PENDING --> FAILED: ScheduleFailed (bad input)
  DRAFT --> FAILED: ScheduleFailed (confirm window expired)
  CONFIRMED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ScheduleEntity ||--o{ ScheduleRequested : emits
  ScheduleEntity ||--o{ RosterBuilt : emits
  ScheduleEntity ||--o{ SchedulingStarted : emits
  ScheduleEntity ||--o{ DraftRecorded : emits
  ScheduleEntity ||--o{ ScheduleConfirmed : emits
  ScheduleEntity ||--o{ ScheduleFailed : emits
  ScheduleView }o--|| ScheduleEntity : projects
  ScheduleWorkflow }o--|| ScheduleEntity : reads-and-writes
  NexShiftAgent ||--o{ ScheduleDraft : returns
  AssignmentGuardrail }o--|| ScheduleEntity : guarded-writes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ScheduleEndpoint` | `api/ScheduleEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ScheduleEntity` | `application/ScheduleEntity.java` (state in `domain/Schedule.java`, events in `domain/ScheduleEvent.java`) |
| `ScheduleWorkflow` | `application/ScheduleWorkflow.java` |
| `NexShiftAgent` | `application/NexShiftAgent.java` (tasks in `application/ScheduleTasks.java`) |
| `AssignmentGuardrail` | `application/AssignmentGuardrail.java` |
| `ScheduleView` | `application/ScheduleView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `buildRosterStep` 10 s, `scheduleStep` 120 s, `confirmStep` 3600 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(ScheduleWorkflow::error)`. The 120 s on `scheduleStep` accommodates multi-shift LLM iteration (Lesson 4).
- **Idempotency**: every workflow uses `"schedule-" + scheduleId` as the workflow id; the `ScheduleEndpoint` submits idempotently — a duplicate POST with the same request returns the existing `scheduleId`.
- **One agent per run**: the AutonomousAgent instance id is `"scheduler-" + scheduleId`, which gives each run its own conversation context. The agent's `capability(...).maxIterationsPerTask(20)` supports large rosters requiring many tool calls.
- **Guardrail-driven retry**: when `AssignmentGuardrail` rejects a tool call, the rejection is returned as the tool's result to the agent loop. The agent reads the rejection reason and retries with a different employee within its iteration budget. The entity receives no write for the rejected call.
- **Manager confirm window**: `confirmStep` has a 3600 s timeout. If the manager does not confirm within 1 hour, the workflow fails over to `error` and the entity transitions to `FAILED`. The draft data is preserved for audit.
- **No saga / no compensation**: assignments are append-only within a run. If the run fails, the entity's draft (partial or empty) is preserved; a new run must be submitted.
