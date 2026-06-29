# PLAN — policy-as-code

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

  API[PolicyEndpoint]:::ep
  Entity[ChangeRequestEntity]:::ese
  Validator[ChangeValidator]:::cons
  WF[EvaluationWorkflow]:::wf
  Agent[PolicyEnforcementAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Gater[GateEvaluator]:::guard
  View[PolicyView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|ChangeSubmitted| Validator
  Validator -->|attachValidated| Entity
  Validator -->|start workflow| WF
  WF -->|awaitValidatedStep poll| Entity
  WF -->|enforceStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|PolicyDecision| WF
  WF -->|recordDecision| Entity
  WF -->|gateStep evaluate| Gater
  Gater -->|GateResult| WF
  WF -->|recordGate| Entity
  Entity -.->|projects| View
  API -->|list/gate/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as PolicyEndpoint
  participant E as ChangeRequestEntity
  participant V as ChangeValidator
  participant W as EvaluationWorkflow
  participant A as PolicyEnforcementAgent
  participant G as ToolCallGuardrail
  participant Ge as GateEvaluator

  U->>API: POST /api/changes
  API->>E: submit(request)
  E-->>API: { changeId }
  E-.->>V: ChangeSubmitted
  V->>V: validate + normalize payload
  V->>E: attachValidated
  V->>W: start(changeId)
  W->>E: poll getChange
  E-->>W: validated.isPresent()
  W->>E: markEnforcing
  W->>A: runSingleTask(rules + attachment)
  A->>G: before-tool-call(lookup-policy-metadata)
  G-->>A: allow
  A-->>W: PolicyDecision
  W->>E: recordDecision(decision)
  W->>Ge: evaluate(decision, rules)
  Ge-->>W: GateResult
  W->>E: recordGate(gate)
  E-.->>U: SSE event(GATED)
```

## State machine — `ChangeRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> VALIDATED: ChangeValidated
  VALIDATED --> ENFORCING: EnforcementStarted
  ENFORCING --> DECISION_RECORDED: DecisionRecorded
  DECISION_RECORDED --> GATED: GateEvaluated
  ENFORCING --> FAILED: EvaluationFailed (agent error)
  SUBMITTED --> FAILED: EvaluationFailed (validator error)
  GATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ChangeRequestEntity ||--o{ ChangeSubmitted : emits
  ChangeRequestEntity ||--o{ ChangeValidated : emits
  ChangeRequestEntity ||--o{ EnforcementStarted : emits
  ChangeRequestEntity ||--o{ DecisionRecorded : emits
  ChangeRequestEntity ||--o{ GateEvaluated : emits
  ChangeRequestEntity ||--o{ EvaluationFailed : emits
  PolicyView }o--|| ChangeRequestEntity : projects
  ChangeValidator }o--|| ChangeRequestEntity : subscribes
  EvaluationWorkflow }o--|| ChangeRequestEntity : reads-and-writes
  PolicyEnforcementAgent ||--o{ PolicyDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PolicyEndpoint` | `api/PolicyEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ChangeRequestEntity` | `application/ChangeRequestEntity.java` (state in `domain/ChangeEvaluation.java`, events in `domain/ChangeEvent.java`) |
| `ChangeValidator` | `application/ChangeValidator.java` |
| `EvaluationWorkflow` | `application/EvaluationWorkflow.java` |
| `PolicyEnforcementAgent` | `application/PolicyEnforcementAgent.java` (tasks in `application/PolicyTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `GateEvaluator` | `application/GateEvaluator.java` |
| `PolicyView` | `application/PolicyView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidatedStep` 15 s, `enforceStep` 60 s, `gateStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(EvaluationWorkflow::error)`. The 60 s on `enforceStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"eval-" + changeId` as the workflow id; the `ChangeValidator` Consumer is allowed to redeliver `ChangeSubmitted` events because `ChangeRequestEntity.attachValidated` is event-version-guarded — a second validate attempt against an already-validated change is a no-op.
- **One agent per change**: the AutonomousAgent instance id is `"enforcer-" + changeId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps iterations at 3.
- **Guardrail-driven tool blocking**: when `ToolCallGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The agent continues reasoning from the attached payload and allowed tools. If the agent cannot produce a decision without the blocked tool, the step eventually times out and fails over to `error`.
- **Gate is synchronous and deterministic**: `GateEvaluator` runs in-process inside `gateStep`. No LLM call, no external service — the same decision always produces the same gate result. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
