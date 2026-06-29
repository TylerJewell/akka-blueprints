# PLAN — merchandiser

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

  API[ProposalEndpoint]:::ep
  Entity[ProposalEntity]:::ese
  Reader[CatalogReader]:::cons
  WF[ProposalWorkflow]:::wf
  Agent[MerchandiserAgent]:::agent
  Guard[StorefrontGuardrail]:::guard
  View[ProposalView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|ObjectiveSubmitted| Reader
  Reader -->|attachContext| Entity
  Reader -->|start workflow| WF
  WF -->|awaitContextStep poll| Entity
  WF -->|generateStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|block write / pass read| Agent
  Agent -->|MerchandisingProposal| WF
  WF -->|recordProposal| Entity
  API -->|approve / reject| Entity
  WF -->|awaitApprovalStep poll| Entity
  WF -->|publishStep write tools| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant M as Merchant (UI)
  participant API as ProposalEndpoint
  participant E as ProposalEntity
  participant R as CatalogReader
  participant W as ProposalWorkflow
  participant A as MerchandiserAgent
  participant G as StorefrontGuardrail

  M->>API: POST /api/proposals
  API->>E: submit(objective)
  E-->>API: { proposalId }
  E-.->>R: ObjectiveSubmitted
  R->>R: fetch catalog seed
  R->>E: attachContext
  R->>W: start(proposalId)
  W->>E: poll getProposal
  E-->>W: context.isPresent()
  W->>E: markGenerating
  W->>A: runSingleTask(objective + catalog.json attachment)
  A->>G: before-tool-call(fetchProduct)
  G-->>A: pass (read-class tool)
  A-->>W: MerchandisingProposal
  W->>E: recordProposal(proposal)
  Note over W,E: awaitApprovalStep — paused
  M->>API: POST /api/proposals/{id}/approve
  API->>E: approve(decision)
  W->>E: poll — APPROVED
  W->>E: markPublishing
  W->>E: markPublished
  E-.->>M: SSE event(PUBLISHED)
```

## State machine — `ProposalEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> CONTEXT_LOADED: CatalogContextLoaded
  CONTEXT_LOADED --> GENERATING: GenerationStarted
  GENERATING --> PENDING_APPROVAL: ProposalGenerated
  PENDING_APPROVAL --> PUBLISHING: ProposalApproved
  PENDING_APPROVAL --> REJECTED: ProposalRejected (terminal)
  PENDING_APPROVAL --> EXPIRED: ProposalExpired (timeout)
  PUBLISHING --> PUBLISHED: ProposalPublished (terminal happy)
  GENERATING --> FAILED: ProposalFailed (agent error)
  SUBMITTED --> FAILED: ProposalFailed (context error)
  PUBLISHED --> [*]
  REJECTED --> [*]
  EXPIRED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ProposalEntity ||--o{ ObjectiveSubmitted : emits
  ProposalEntity ||--o{ CatalogContextLoaded : emits
  ProposalEntity ||--o{ GenerationStarted : emits
  ProposalEntity ||--o{ ProposalGenerated : emits
  ProposalEntity ||--o{ ProposalApproved : emits
  ProposalEntity ||--o{ ProposalRejected : emits
  ProposalEntity ||--o{ ProposalPublished : emits
  ProposalEntity ||--o{ ProposalExpired : emits
  ProposalEntity ||--o{ ProposalFailed : emits
  ProposalView }o--|| ProposalEntity : projects
  CatalogReader }o--|| ProposalEntity : subscribes
  ProposalWorkflow }o--|| ProposalEntity : reads-and-writes
  MerchandiserAgent ||--o{ MerchandisingProposal : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ProposalEndpoint` | `api/ProposalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ProposalEntity` | `application/ProposalEntity.java` (state in `domain/Proposal.java`, events in `domain/ProposalEvent.java`) |
| `CatalogReader` | `application/CatalogReader.java` |
| `ProposalWorkflow` | `application/ProposalWorkflow.java` |
| `MerchandiserAgent` | `application/MerchandiserAgent.java` (tasks in `application/ProposalTasks.java`) |
| `StorefrontGuardrail` | `application/StorefrontGuardrail.java` |
| `ProposalView` | `application/ProposalView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitContextStep` 15 s, `generateStep` 90 s, `awaitApprovalStep` 259200 s (72 h), `publishStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ProposalWorkflow::error)`. The 90 s on `generateStep` accommodates LLM latency and multi-turn tool calls (Lesson 4).
- **Idempotency**: every workflow uses `"proposal-" + proposalId` as the workflow id; `CatalogReader` is allowed to redeliver `ObjectiveSubmitted` events because `ProposalEntity.attachContext` is event-version-guarded — a second context attachment against an already-loaded proposal is a no-op.
- **One agent per proposal**: the AutonomousAgent instance id is `"merchandiser-" + proposalId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `StorefrontGuardrail` rejects a write-class tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail to produce a valid proposal, the workflow's `generateStep` fails over to `error` and the entity transitions to `FAILED`.
- **Approval is external**: `awaitApprovalStep` does not poll a second agent — it polls the entity's status field, which is updated by merchant action via `ProposalEndpoint`. The approval gate is a human action boundary, not an LLM boundary.
- **No saga / no compensation on publish**: `publishStep` applies each `ChangeRecommendation` sequentially. If one write-class tool call fails mid-publish, the entity transitions to `FAILED` with the partial set of changes recorded. A deployer needing compensation would extend the publish step with rollback logic per their storefront API's contract.
