# PLAN — product-catalog-ad-generation

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[AdJobEndpoint]:::ep
  Entity[AdJobEntity]:::ese
  WF[AdGenerationWorkflow]:::wf
  Agent[AdGeneratorAgent]:::agent
  Enrich[EnrichTools]:::tool
  Draft[DraftTools]:::tool
  Review[ReviewTools]:::tool
  Guard[BrandGuardrail]:::guard
  Scorer[BrandComplianceScorer]:::guard
  View[AdJobView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|enrichStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|invokes| Enrich
  Agent -->|invokes| Draft
  Agent -->|invokes| Review
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|EnrichedProduct / AdDraft / ProductAd| WF
  WF -->|recordEnrichment/Draft/Ad| Entity
  WF -->|complianceStep score| Scorer
  Scorer -->|ComplianceResult| WF
  WF -->|recordCompliance| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AdJobEndpoint
  participant E as AdJobEntity
  participant W as AdGenerationWorkflow
  participant A as AdGeneratorAgent
  participant G as BrandGuardrail
  participant T as Tools (Enrich/Draft/Review)
  participant Sc as BrandComplianceScorer

  U->>API: POST /api/ad-jobs { productName }
  API->>E: create(productName)
  E-->>API: { jobId }
  API->>W: start(jobId, productName)
  W->>E: startEnrich
  W->>A: runSingleTask(ENRICH_PRODUCT, productName)
  A->>T: lookupCategory + inferAttributes
  T-->>A: EnrichedProduct
  A->>G: before-agent-response(EnrichedProduct)
  G-->>A: accept (ENRICH response; no brand rules apply)
  A-->>W: EnrichedProduct
  W->>E: recordEnrichment
  W->>A: runSingleTask(DRAFT_AD, enrichedProduct)
  A->>T: composeHeadline + composeCopy (×placements)
  T-->>A: AdDraft
  A->>G: before-agent-response(AdDraft)
  G-->>A: accept (headline element OK, no prohibited words)
  A-->>W: AdDraft
  W->>E: recordDraft
  W->>A: runSingleTask(REVIEW_AD, adDraft)
  A->>T: checkBrandElements + finaliseAd
  T-->>A: PolicyViolation[] + ProductAd
  A->>G: before-agent-response(ProductAd)
  G-->>A: accept (final ad passes all rules)
  A-->>W: ProductAd
  W->>E: recordAd
  W->>Sc: score(productAd, adDraft, enrichedProduct)
  Sc-->>W: ComplianceResult
  W->>E: recordCompliance
  E-.->>U: SSE event(SCORED)
```

## State machine — `AdJobEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: ProductEnriched
  ENRICHED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: AdDrafted
  DRAFTED --> REVIEWING: ReviewStarted
  REVIEWING --> APPROVED: AdApproved
  APPROVED --> SCORED: ComplianceScored
  ENRICHING --> FAILED: AdJobFailed
  DRAFTING --> FAILED: AdJobFailed
  REVIEWING --> FAILED: AdJobFailed
  SCORED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  AdJobEntity ||--o{ AdJobCreated : emits
  AdJobEntity ||--o{ EnrichStarted : emits
  AdJobEntity ||--o{ ProductEnriched : emits
  AdJobEntity ||--o{ DraftStarted : emits
  AdJobEntity ||--o{ AdDrafted : emits
  AdJobEntity ||--o{ ReviewStarted : emits
  AdJobEntity ||--o{ AdApproved : emits
  AdJobEntity ||--o{ ComplianceScored : emits
  AdJobEntity ||--o{ GuardrailRejected : emits
  AdJobEntity ||--o{ AdJobFailed : emits
  AdJobView }o--|| AdJobEntity : projects
  AdGenerationWorkflow }o--|| AdJobEntity : reads-and-writes
  AdGeneratorAgent ||--o{ EnrichedProduct : returns
  AdGeneratorAgent ||--o{ AdDraft : returns
  AdGeneratorAgent ||--o{ ProductAd : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AdJobEndpoint` | `api/AdJobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AdJobEntity` | `application/AdJobEntity.java` (state in `domain/AdJobRecord.java`, events in `domain/AdJobEvent.java`) |
| `AdGenerationWorkflow` | `application/AdGenerationWorkflow.java` |
| `AdGeneratorAgent` | `application/AdGeneratorAgent.java` (tasks in `application/AdTasks.java`) |
| `EnrichTools` | `application/EnrichTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `ReviewTools` | `application/ReviewTools.java` |
| `BrandGuardrail` | `application/BrandGuardrail.java` |
| `BrandComplianceScorer` | `application/BrandComplianceScorer.java` |
| `AdJobView` | `application/AdJobView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `enrichStep` 60 s, `draftStep` 60 s, `reviewStep` 60 s, `complianceStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AdGenerationWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + jobId` as the workflow id; restart of the same jobId is rejected by the workflow runtime. The agent instance id is `"agent-" + jobId` so each job has its own per-task conversation memory.
- **One agent per job**: `AdGeneratorAgent` runs three tasks per job — ENRICH, DRAFT, REVIEW — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the brand guardrail room to reject a non-compliant draft and still let the agent self-correct.
- **Guardrail-driven retry**: when `BrandGuardrail` rejects a response, the rejection is returned as a structured `BrandRejection{violations}` to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Compliance eval is synchronous and deterministic**: `BrandComplianceScorer` runs in-process inside `complianceStep`. No LLM call, no external service — the same ad always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `enrichStep` writes `ProductEnriched` BEFORE returning; `draftStep` reads the recorded `EnrichedProduct` from the entity to build its task's instruction context; `reviewStep` reads both `EnrichedProduct` and `AdDraft`. The agent itself is stateless across phases — it never holds enrich + draft + review context in one conversation.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. A failed job stays at the last successful event; the UI shows the partial state for the user.
