# PLAN — on-brand-genmedia

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Brand[BrandAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[GenerationWorkflow]:::wf
  Asset[AssetEntity]:::ese
  Queue[CampaignQueue]:::ese
  View[AssetsView]:::view
  Consumer[CampaignRequestConsumer]:::cons
  Sim[CampaignSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[BrandEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Brand
  WF -->|review| Reviewer
  WF -->|emit events| Asset
  Asset -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Asset
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as BrandEndpoint
  participant Q as CampaignQueue
  participant C as CampaignRequestConsumer
  participant W as GenerationWorkflow
  participant B as BrandAgent
  participant R as ReviewerAgent
  participant E as AssetEntity
  participant V as AssetsView

  U->>API: POST /api/assets {product, channel}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {assetId}
  Q->>C: BriefSubmitted
  C->>W: start({assetId, product, channel, maxAttempts=4})
  W->>E: emit AssetCreated (GENERATING)

  W->>B: GENERATE(product, channel, tone)
  B-->>W: GeneratedAsset #1 (290 tokens)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: guardrailStep (deterministic prohibited-content check)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW(GeneratedAsset #1)
  R-->>W: Review{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptReviewed (n=1, REVISE)

  W->>B: REVISE_ASSET(brief, prior, notes)
  B-->>W: GeneratedAsset #2 (275 tokens)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>R: REVIEW(GeneratedAsset #2)
  R-->>W: Review{APPROVE, score=5, rationale}
  W->>E: emit AttemptReviewed (n=2, APPROVE)
  W->>E: emit AssetApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `AssetEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> REVIEWING: AttemptGenerated + guardrail passed
  GENERATING --> GENERATING: guardrail blocked, re-generate
  REVIEWING --> GENERATING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  AssetEntity ||--o{ AssetCreated : emits
  AssetEntity ||--o{ AttemptGenerated : emits
  AssetEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  AssetEntity ||--o{ AttemptReviewed : emits
  AssetEntity ||--o{ AssetApproved : emits
  AssetEntity ||--o{ AssetRejectedFinal : emits
  AssetEntity ||--o{ ReviewEvalRecorded : emits
  AssetsView }o--|| AssetEntity : projects
  CampaignQueue ||--o{ BriefSubmitted : emits
  CampaignRequestConsumer }o--|| CampaignQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BrandAgent` | `application/BrandAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `BrandTasks` | `application/BrandTasks.java` |
| `GenerationWorkflow` | `application/GenerationWorkflow.java` |
| `AssetEntity` | `application/AssetEntity.java` (state in `domain/Asset.java`, events in `domain/AssetEvent.java`) |
| `CampaignQueue` | `application/CampaignQueue.java` |
| `AssetsView` | `application/AssetsView.java` |
| `CampaignRequestConsumer` | `application/CampaignRequestConsumer.java` |
| `CampaignSimulator` | `application/CampaignSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `BrandEndpoint` | `api/BrandEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `BrandEndpoint.submit` uses `(product, channel, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(assetId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `on-brand-genmedia.generation.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only terminal path for ceiling-exhausted loops; it preserves the best asset and every review on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans the generated text against a configurable prohibited-terms list and either advances to `reviewStep` or returns to `generateStep` with a structured feedback note. The structured feedback stays a deterministic `ReviewNotes` payload with a single bullet.
