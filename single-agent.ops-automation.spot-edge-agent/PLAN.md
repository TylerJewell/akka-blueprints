# PLAN — spot-edge-agent

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
  classDef sim fill:#0e1a1a,stroke:#22d3ee,color:#22d3ee;

  API[MissionEndpoint]:::ep
  Entity[MissionEntity]:::ese
  Monitor[TelemetryMonitor]:::cons
  WF[MissionWorkflow]:::wf
  Agent[SpotNavigatorAgent]:::agent
  Guard[CommandGuardrail]:::guard
  Spot[SimulatedSpotMcp]:::sim
  View[MissionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  Entity -.->|MissionStarted| Monitor
  Monitor -->|triggerSafetyHalt| Entity
  Monitor -->|spot.halt| Spot
  WF -->|planStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|reject / pass| Agent
  Agent -->|spot.navigate_to| Spot
  Spot -->|tool response| Agent
  Agent -->|MissionOutcome| WF
  WF -->|approvalGateStep pause| Entity
  API -->|approve / reject| WF
  WF -->|executeStep runSingleTask| Agent
  WF -->|completeMission| Entity
  Entity -.->|projects| View
  API -->|list / SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, trivial mission)

```mermaid
sequenceDiagram
  autonumber
  participant Op as Operator (UI)
  participant API as MissionEndpoint
  participant E as MissionEntity
  participant WF as MissionWorkflow
  participant A as SpotNavigatorAgent
  participant G as CommandGuardrail
  participant S as SimulatedSpotMcp

  Op->>API: POST /api/missions
  API->>E: submit(brief)
  E-->>API: { missionId }
  API->>WF: start(missionId)
  WF->>A: planStep runSingleTask(EXECUTE_MISSION, waypoints)
  A->>G: before-tool-call(spot.navigate_to)
  G-->>A: pass
  A->>S: spot.navigate_to(wp1)
  S-->>A: waypoint reached
  A->>G: before-tool-call(spot.capture_image)
  G-->>A: pass
  A->>S: spot.capture_image
  S-->>A: image stored
  A-->>WF: MissionOutcome(COMPLETED)
  WF->>E: completeMission(outcome)
  WF->>E: recordFinalTelemetry
  E-.->>Op: SSE event(COMPLETED)
```

## State machine — `MissionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PLANNING: PlanningStarted
  PLANNING --> PENDING_APPROVAL: ApprovalRequested (non-trivial)
  PLANNING --> EXECUTING: MissionStarted (trivial)
  PENDING_APPROVAL --> EXECUTING: ApprovalDecided(APPROVED)
  PENDING_APPROVAL --> FAILED: ApprovalDecided(REJECTED)
  PENDING_APPROVAL --> FAILED: approval timeout
  EXECUTING --> COMPLETED: MissionCompleted
  EXECUTING --> HALTED: SafetyHaltTriggered
  EXECUTING --> FAILED: MissionFailed (agent error)
  PLANNING --> FAILED: MissionFailed (plan error)
  COMPLETED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  MissionEntity ||--o{ MissionSubmitted : emits
  MissionEntity ||--o{ PlanningStarted : emits
  MissionEntity ||--o{ ApprovalRequested : emits
  MissionEntity ||--o{ ApprovalDecided : emits
  MissionEntity ||--o{ MissionStarted : emits
  MissionEntity ||--o{ WaypointReached : emits
  MissionEntity ||--o{ SafetyHaltTriggered : emits
  MissionEntity ||--o{ MissionCompleted : emits
  MissionEntity ||--o{ MissionFailed : emits
  MissionView }o--|| MissionEntity : projects
  TelemetryMonitor }o--|| MissionEntity : subscribes
  MissionWorkflow }o--|| MissionEntity : reads-and-writes
  SpotNavigatorAgent ||--o{ MissionOutcome : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MissionEndpoint` | `api/MissionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MissionEntity` | `application/MissionEntity.java` (state in `domain/Mission.java`, events in `domain/MissionEvent.java`) |
| `TelemetryMonitor` | `application/TelemetryMonitor.java` |
| `MissionWorkflow` | `application/MissionWorkflow.java` |
| `SpotNavigatorAgent` | `application/SpotNavigatorAgent.java` (tasks in `application/MissionTasks.java`) |
| `CommandGuardrail` | `application/CommandGuardrail.java` |
| `SimulatedSpotMcp` | `application/SimulatedSpotMcp.java` |
| `MissionView` | `application/MissionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `planStep` 30 s, `approvalGateStep` 600 s, `executeStep` 120 s, `telemetryStep` 10 s, `error` 10 s. Default step recovery `maxRetries(2).failoverTo(MissionWorkflow::error)`. The 120 s on `executeStep` accommodates multi-waypoint traversal latency (Lesson 4).
- **Idempotency**: every workflow uses `"mission-" + missionId` as the workflow id; `TelemetryMonitor` guards against double-halt by checking the entity's current status before writing `SafetyHaltTriggered`.
- **One agent per mission**: the AutonomousAgent instance id is `"navigator-" + missionId`, giving each mission its own conversation context. `maxIterationsPerTask(5)` caps guardrail-triggered replans.
- **Guardrail-driven replan**: when `CommandGuardrail` rejects a proposed command, the rejection is a structured error returned to the agent loop. The agent must propose an alternative command on the next iteration; each rejection consumes one iteration toward the cap of 5.
- **Halt independence**: `TelemetryMonitor` is a Consumer, not part of the workflow. The halt path fires even if `executeStep` is mid-await. The workflow's `executeStep` detects that the mission reached HALTED status when it polls the entity and terminates without writing `MissionCompleted`.
- **Approval gate is a pause, not a poll**: `approvalGateStep` uses `Workflow.pause()` and a resume callback tied to `MissionEndpoint.approve/reject`. It does not loop or sleep.
- **No saga / no compensation**: every step is append-only event writes plus single-task agent calls. The in-process `SimulatedSpotMcp` has no external state to roll back.
