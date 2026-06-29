# PLAN — sdlc-technical-designer

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

  API[DesignEndpoint]:::ep
  Entity[DesignRequestEntity]:::ese
  Loader[ContextLoader]:::cons
  WF[DesignWorkflow]:::wf
  Agent[DesignAgent]:::agent
  Guard[ProposalGuardrail]:::guard
  Scorer[ProposalScorer]:::guard
  View[DesignView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|DesignRequestSubmitted| Loader
  Loader -->|attachContext| Entity
  Loader -->|start workflow| WF
  WF -->|awaitContextStep poll| Entity
  WF -->|designStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|DesignProposal| WF
  WF -->|recordProposal| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as DesignEndpoint
  participant E as DesignRequestEntity
  participant L as ContextLoader
  participant W as DesignWorkflow
  participant A as DesignAgent
  participant G as ProposalGuardrail
  participant Sc as ProposalScorer

  U->>API: POST /api/designs
  API->>E: submit(request)
  E-->>API: { designId }
  E-.->>L: DesignRequestSubmitted
  L->>L: resolve project context
  L->>E: attachContext
  L->>W: start(designId)
  W->>E: poll getDesignRequest
  E-->>W: context.isPresent()
  W->>E: markDesigning
  W->>A: runSingleTask(context profile + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: DesignProposal
  W->>E: recordProposal(proposal)
  W->>Sc: score(proposal)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `DesignRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> CONTEXT_LOADED: ContextLoaded
  CONTEXT_LOADED --> DESIGNING: DesignStarted
  DESIGNING --> PROPOSAL_RECORDED: ProposalRecorded
  PROPOSAL_RECORDED --> EVALUATED: EvaluationScored
  DESIGNING --> FAILED: DesignFailed (agent error)
  SUBMITTED --> FAILED: DesignFailed (context-load error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DesignRequestEntity ||--o{ DesignRequestSubmitted : emits
  DesignRequestEntity ||--o{ ContextLoaded : emits
  DesignRequestEntity ||--o{ DesignStarted : emits
  DesignRequestEntity ||--o{ ProposalRecorded : emits
  DesignRequestEntity ||--o{ EvaluationScored : emits
  DesignRequestEntity ||--o{ DesignFailed : emits
  DesignView }o--|| DesignRequestEntity : projects
  ContextLoader }o--|| DesignRequestEntity : subscribes
  DesignWorkflow }o--|| DesignRequestEntity : reads-and-writes
  DesignAgent ||--o{ DesignProposal : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DesignEndpoint` | `api/DesignEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DesignRequestEntity` | `application/DesignRequestEntity.java` (state in `domain/DesignRequestState.java`, events in `domain/DesignEvent.java`) |
| `ContextLoader` | `application/ContextLoader.java` |
| `DesignWorkflow` | `application/DesignWorkflow.java` |
| `DesignAgent` | `application/DesignAgent.java` (tasks in `application/DesignTasks.java`) |
| `ProposalGuardrail` | `application/ProposalGuardrail.java` |
| `ProposalScorer` | `application/ProposalScorer.java` |
| `DesignView` | `application/DesignView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitContextStep` 15 s, `designStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DesignWorkflow::error)`. The 90 s on `designStep` accommodates LLM latency on larger feature descriptions (Lesson 4).
- **Idempotency**: every workflow uses `"design-" + designId` as the workflow id; the `ContextLoader` Consumer is allowed to redeliver `DesignRequestSubmitted` events because `DesignRequestEntity.attachContext` is event-version-guarded — a second context-load attempt against an already-loaded request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"designer-" + designId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ProposalGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `designStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `ProposalScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same proposal always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
