# PLAN — brand-aligner

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

  Copy[CopywriterAgent]:::agent
  Reviewer[BrandReviewerAgent]:::agent

  WF[AlignmentWorkflow]:::wf
  Mat[MaterialEntity]:::ese
  Queue[BriefQueue]:::ese
  View[MaterialsView]:::view
  Consumer[BriefConsumer]:::cons
  Sim[CampaignSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[BrandEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Copy
  WF -->|review| Reviewer
  WF -->|emit events| Mat
  Mat -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Mat
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as BrandEndpoint
  participant Q as BriefQueue
  participant C as BriefConsumer
  participant W as AlignmentWorkflow
  participant CW as CopywriterAgent
  participant BR as BrandReviewerAgent
  participant M as MaterialEntity
  participant V as MaterialsView

  U->>API: POST /api/materials {topic, audience, ceiling}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {materialId}
  Q->>C: BriefSubmitted
  C->>W: start({materialId, topic, audience, ceiling, maxAttempts=4})
  W->>M: emit MaterialCreated (DRAFTING)

  W->>CW: GENERATE(topic, audience, ceiling)
  CW-->>W: CopyVariant #1 (148 words)
  W->>M: emit VariantGenerated (n=1)
  Note over W: complianceStep (deterministic word-count check)
  W->>M: emit ComplianceVerdictRecorded (passed=true)
  W->>M: status REVIEWING
  W->>BR: REVIEW(CopyVariant #1)
  BR-->>W: BrandReview{REVISE, score=3, 2 bullets}
  W->>M: emit VariantReviewed (n=1, REVISE)

  W->>CW: REVISE_VARIANT(topic, prior, notes)
  CW-->>W: CopyVariant #2 (140 words)
  W->>M: emit VariantGenerated (n=2)
  W->>M: emit ComplianceVerdictRecorded (passed=true)
  W->>BR: REVIEW(CopyVariant #2)
  BR-->>W: BrandReview{APPROVE, score=5, rationale}
  W->>M: emit VariantReviewed (n=2, APPROVE)
  W->>M: emit MaterialApproved (n=2)
  M-->>V: project
  V-->>U: SSE update
```

## State machine — `MaterialEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> REVIEWING: VariantGenerated + compliance passed
  DRAFTING --> DRAFTING: compliance blocked, re-generate
  REVIEWING --> DRAFTING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  MaterialEntity ||--o{ MaterialCreated : emits
  MaterialEntity ||--o{ VariantGenerated : emits
  MaterialEntity ||--o{ ComplianceVerdictRecorded : emits
  MaterialEntity ||--o{ VariantReviewed : emits
  MaterialEntity ||--o{ MaterialApproved : emits
  MaterialEntity ||--o{ MaterialRejectedFinal : emits
  MaterialEntity ||--o{ BrandEvalRecorded : emits
  MaterialsView }o--|| MaterialEntity : projects
  BriefQueue ||--o{ BriefSubmitted : emits
  BriefConsumer }o--|| BriefQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CopywriterAgent` | `application/CopywriterAgent.java` |
| `BrandReviewerAgent` | `application/BrandReviewerAgent.java` |
| `BrandTasks` | `application/BrandTasks.java` |
| `AlignmentWorkflow` | `application/AlignmentWorkflow.java` |
| `MaterialEntity` | `application/MaterialEntity.java` (state in `domain/Material.java`, events in `domain/MaterialEvent.java`) |
| `BriefQueue` | `application/BriefQueue.java` |
| `MaterialsView` | `application/MaterialsView.java` |
| `BriefConsumer` | `application/BriefConsumer.java` |
| `CampaignSimulator` | `application/CampaignSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `BrandEndpoint` | `api/BrandEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `BrandEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(materialId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `brand-aligner.alignment.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best variant and every review on the entity.
- **Compliance step:** `complianceStep` is pure-function (no LLM call); it computes the word count from the variant and either advances to `reviewStep` or returns to `generateStep` with a structured feedback note. The structured feedback never becomes an LLM-generated review; it stays a deterministic `ReviewNotes` payload with a single bullet.
