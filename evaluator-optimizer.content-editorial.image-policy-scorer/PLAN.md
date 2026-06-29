# PLAN — image-policy-scorer

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

  Gen[GeneratorAgent]:::agent
  Scorer[ScorerAgent]:::agent

  WF[ScoringWorkflow]:::wf
  Img[ImageEntity]:::ese
  Queue[PromptQueue]:::ese
  View[ImagesView]:::view
  Consumer[PromptConsumer]:::cons
  Sim[PromptSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[ImageEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue prompt| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|generate / revise| Gen
  WF -->|evaluate policy| Scorer
  WF -->|emit events| Img
  Img -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Img
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ImageEndpoint
  participant Q as PromptQueue
  participant C as PromptConsumer
  participant W as ScoringWorkflow
  participant G as GeneratorAgent
  participant S as ScorerAgent
  participant E as ImageEntity
  participant V as ImagesView

  U->>API: POST /api/images {promptText, audienceTier}
  API->>Q: append PromptSubmitted
  API-->>U: 202 {imageId}
  Q->>C: PromptSubmitted
  C->>W: start({imageId, promptText, audienceTier, maxAttempts=4})
  W->>E: emit ImageCreated (GENERATING)

  W->>G: GENERATE(promptText, audienceTier)
  G-->>W: ImageDescription #1 (brandSafetySignal=mild)
  W->>E: emit AttemptGenerated (n=1)
  Note over W: gateStep (deterministic signal check)
  W->>E: emit AttemptGateVerdictRecorded (passed=true)
  W->>E: status SCORING
  W->>S: EVALUATE_POLICY(ImageDescription #1, audienceTier)
  S-->>W: PolicyVerdict{FAIL, score=3, 3 bullets}
  W->>E: emit AttemptScored (n=1, FAIL)

  W->>G: REVISE_DESCRIPTION(promptText, prior, notes)
  G-->>W: ImageDescription #2 (brandSafetySignal=none)
  W->>E: emit AttemptGenerated (n=2)
  W->>E: emit AttemptGateVerdictRecorded (passed=true)
  W->>S: EVALUATE_POLICY(ImageDescription #2, audienceTier)
  S-->>W: PolicyVerdict{PASS, score=5, rationale}
  W->>E: emit AttemptScored (n=2, PASS)
  W->>E: emit ImageApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ImageEntity`

```mermaid
stateDiagram-v2
  [*] --> GENERATING
  GENERATING --> SCORING: AttemptGenerated + gate passed
  GENERATING --> GENERATING: gate blocked, re-generate
  SCORING --> GENERATING: PolicyVerdict = FAIL, attempts < max
  SCORING --> APPROVED: PolicyVerdict = PASS
  SCORING --> REJECTED_FINAL: PolicyVerdict = FAIL, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  ImageEntity ||--o{ ImageCreated : emits
  ImageEntity ||--o{ AttemptGenerated : emits
  ImageEntity ||--o{ AttemptGateVerdictRecorded : emits
  ImageEntity ||--o{ AttemptScored : emits
  ImageEntity ||--o{ ImageApproved : emits
  ImageEntity ||--o{ ImageRejectedFinal : emits
  ImageEntity ||--o{ PolicyEvalRecorded : emits
  ImagesView }o--|| ImageEntity : projects
  PromptQueue ||--o{ PromptSubmitted : emits
  PromptConsumer }o--|| PromptQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `ScorerAgent` | `application/ScorerAgent.java` |
| `ImageTasks` | `application/ImageTasks.java` |
| `ScoringWorkflow` | `application/ScoringWorkflow.java` |
| `ImageEntity` | `application/ImageEntity.java` (state in `domain/Image.java`, events in `domain/ImageEvent.java`) |
| `PromptQueue` | `application/PromptQueue.java` |
| `ImagesView` | `application/ImagesView.java` |
| `PromptConsumer` | `application/PromptConsumer.java` |
| `PromptSimulator` | `application/PromptSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ImageEndpoint` | `api/ImageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `generateStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `ImageEndpoint.submit` uses `(promptText, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(imageId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `image-policy-scorer.scoring.max-attempts` (default 4). The workflow checks the count BEFORE calling `generateStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. Images are never written to an external store in the default configuration; the halt mechanism preserves the best description and every policy note on the entity.
- **Gate step:** `gateStep` is pure-function (no LLM call); it checks `description.brandSafetySignal()` against the prohibited-signal set from application.conf and either advances to `scoreStep` or returns to `generateStep` with a structured feedback note. The structured feedback stays a deterministic `PolicyNotes` payload — it is never an LLM-generated verdict.
