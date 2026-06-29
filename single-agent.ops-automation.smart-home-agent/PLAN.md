# PLAN — smart-home-agent

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

  API[CommandEndpoint]:::ep
  CmdEntity[CommandEntity]:::ese
  DevReg[DeviceRegistry]:::ese
  WF[CommandWorkflow]:::wf
  Agent[HomeControlAgent]:::agent
  Guard[ActionGuardrail]:::guard
  View[CommandView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| CmdEntity
  API -->|confirm/cancel| CmdEntity
  API -->|getDevices| DevReg
  CmdEntity -.->|CommandReceived| WF
  WF -->|guardStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|DeviceAction| WF
  WF -->|recordGuardPassed| CmdEntity
  WF -->|confirmStep pause| CmdEntity
  WF -->|dispatchStep applyAction| DevReg
  WF -->|dispatch| CmdEntity
  CmdEntity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J2 (lock with HITL)

```mermaid
sequenceDiagram
  autonumber
  participant U as Resident (UI)
  participant API as CommandEndpoint
  participant CE as CommandEntity
  participant WF as CommandWorkflow
  participant A as HomeControlAgent
  participant G as ActionGuardrail
  participant DR as DeviceRegistry

  U->>API: POST /api/commands {rawCommand: "Lock the front door"}
  API->>CE: submit(request)
  CE-->>API: { commandId }
  API->>WF: start(commandId)
  WF->>DR: getDevices (catalogue)
  DR-->>WF: [Device...]
  WF->>A: runSingleTask(rawCommand + devices.json attachment)
  A->>G: before-tool-call(proposed DeviceAction)
  G-->>A: accept
  A-->>WF: DeviceAction{LOCK, front-door-lock}
  WF->>CE: recordGuardPassed(action)
  WF->>CE: requestConfirmation(action)
  CE-->>U: SSE AWAITING_CONFIRMATION
  U->>API: POST /api/commands/{id}/confirm
  API->>CE: confirm()
  CE-->>WF: ActionConfirmed
  WF->>DR: applyAction(front-door-lock, LOCK, null)
  WF->>CE: dispatch(action)
  CE-->>U: SSE DISPATCHED
```

## State machine — `CommandEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> GUARD_PASSED: GuardPassed
  RECEIVED --> BLOCKED: CommandBlocked (guardrail rejected)
  GUARD_PASSED --> AWAITING_CONFIRMATION: ConfirmationRequested (sensitive action)
  GUARD_PASSED --> DISPATCHED: ActionDispatched (non-sensitive)
  AWAITING_CONFIRMATION --> DISPATCHED: ActionConfirmed + ActionDispatched
  AWAITING_CONFIRMATION --> CANCELLED: ActionCancelled
  RECEIVED --> FAILED: CommandFailed (agent error)
  GUARD_PASSED --> FAILED: CommandFailed (dispatch error)
  DISPATCHED --> [*]
  BLOCKED --> [*]
  CANCELLED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CommandEntity ||--o{ CommandReceived : emits
  CommandEntity ||--o{ GuardPassed : emits
  CommandEntity ||--o{ CommandBlocked : emits
  CommandEntity ||--o{ ConfirmationRequested : emits
  CommandEntity ||--o{ ActionConfirmed : emits
  CommandEntity ||--o{ ActionCancelled : emits
  CommandEntity ||--o{ ActionDispatched : emits
  CommandEntity ||--o{ CommandFailed : emits
  DeviceRegistry ||--o{ DeviceRegistered : emits
  DeviceRegistry ||--o{ DeviceStateChanged : emits
  CommandView }o--|| CommandEntity : projects
  CommandWorkflow }o--|| CommandEntity : reads-and-writes
  CommandWorkflow }o--|| DeviceRegistry : reads-and-writes
  HomeControlAgent ||--o{ DeviceAction : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CommandEndpoint` | `api/CommandEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CommandEntity` | `application/CommandEntity.java` (state in `domain/Command.java`, events in `domain/CommandEvent.java`) |
| `DeviceRegistry` | `application/DeviceRegistry.java` (state in `domain/DeviceRegistryState.java`, events in `domain/DeviceRegistryEvent.java`) |
| `CommandWorkflow` | `application/CommandWorkflow.java` |
| `HomeControlAgent` | `application/HomeControlAgent.java` (tasks in `application/CommandTasks.java`) |
| `ActionGuardrail` | `application/ActionGuardrail.java` |
| `CommandView` | `application/CommandView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `guardStep` 60 s, `confirmStep` 300 s, `dispatchStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CommandWorkflow::error)`. The 60 s on `guardStep` accommodates LLM latency (Lesson 4).
- **HITL pause**: `confirmStep` waits up to 300 s for an explicit confirm or cancel from the resident. If the timeout elapses without a response, the workflow's step timeout fires and transitions to `error`, which calls `CommandEntity.fail("confirmation-timeout")`.
- **Idempotency**: the workflow id is `"cmd-" + commandId`; `CommandEntity.submit` is idempotent via event-version guard.
- **Guardrail-driven retry**: `ActionGuardrail` rejection returns a structured error to the agent loop. The loop counts toward `maxIterationsPerTask(3)`; if all 3 iterations fail validation, the workflow's `guardStep` fails over to `error`.
- **Single-agent invariant**: exactly one component (`HomeControlAgent`) talks to a model. The HITL confirmation is a workflow step interacting with `CommandEntity` — no second LLM call, no second agent.
- **DeviceRegistry seeding**: `Bootstrap.java` calls `registerDevice` for each of the 4 seeded devices on first startup if the registry is empty, ensuring the UI displays a non-empty device grid without any setup step.
