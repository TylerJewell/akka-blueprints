# PLAN — ad-creator-pipeline

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
  WF[AdPipelineWorkflow]:::wf
  Agent[AdCreatorAgent]:::agent
  Scrape[ScrapeTools]:::tool
  Copy[CopyTools]:::tool
  Visual[VisualTools]:::tool
  ScrapeGuard[ScrapingPolicyGuardrail]:::guard
  BrandGuard[BrandSafetyGuardrail]:::guard
  Scorer[AdQualityScorer]:::guard
  View[AdJobView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|scrapeStep runSingleTask| Agent
  Agent -.->|before-tool-call| ScrapeGuard
  Agent -.->|before-agent-response| BrandGuard
  Agent -->|invokes| Scrape
  Agent -->|invokes| Copy
  Agent -->|invokes| Visual
  ScrapeGuard -->|recordScrapingRejection| Entity
  BrandGuard -->|recordBrandSafetyRejection| Entity
  Agent -->|ProductProfile / AdCopy / VisualSpec| WF
  WF -->|recordProfile/Copy/Visual| Entity
  WF -->|evalStep assemble + score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
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
  participant W as AdPipelineWorkflow
  participant A as AdCreatorAgent
  participant SG as ScrapingPolicyGuardrail
  participant BG as BrandSafetyGuardrail
  participant T as Tools (Scrape/Copy/Visual)
  participant Sc as AdQualityScorer

  U->>API: POST /api/ad-jobs { productUrl }
  API->>E: create(productUrl)
  E-->>API: { adJobId }
  API->>W: start(adJobId, productUrl)
  W->>E: startScrape
  W->>A: runSingleTask(SCRAPE_PRODUCT, productUrl)
  A->>SG: before-tool-call(fetchProductPage, SCRAPE)
  SG-->>A: accept (url on allow-list, status SCRAPING)
  A->>T: fetchProductPage + extractAttributes
  T-->>A: ProductProfile
  A-->>W: ProductProfile
  W->>E: recordProfile
  W->>A: runSingleTask(DRAFT_COPY, profile)
  A->>SG: before-tool-call(draftHeadline, COPY)
  SG-->>A: accept (status COPYING and profile present)
  A->>T: draftHeadline + draftBody
  T-->>A: AdCopy
  A->>BG: before-agent-response(AdCopy text)
  BG-->>A: accept (no violations)
  A-->>W: AdCopy
  W->>E: recordCopy
  W->>A: runSingleTask(GENERATE_VISUAL, copy)
  A->>SG: before-tool-call(buildImagePrompt, VISUAL)
  SG-->>A: accept (status GENERATING_VISUAL and copy present)
  A->>T: buildImagePrompt + selectAspectRatio
  T-->>A: VisualSpec
  A-->>W: VisualSpec
  W->>E: recordVisual
  W->>Sc: score(adPackage, copy, profile)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `AdJobEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SCRAPING: ScrapeStarted
  SCRAPING --> SCRAPED: ProductScraped
  SCRAPED --> COPYING: CopyStarted
  COPYING --> COPIED: CopyDrafted
  COPIED --> GENERATING_VISUAL: VisualStarted
  GENERATING_VISUAL --> VISUAL_GENERATED: VisualGenerated
  VISUAL_GENERATED --> EVALUATED: AdEvaluated
  SCRAPING --> FAILED: AdJobFailed
  COPYING --> FAILED: AdJobFailed
  GENERATING_VISUAL --> FAILED: AdJobFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`ScrapingRejected` and `BrandSafetyRejected` are side-events recorded on the entity for audit; they do not change the status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  AdJobEntity ||--o{ AdJobCreated : emits
  AdJobEntity ||--o{ ScrapeStarted : emits
  AdJobEntity ||--o{ ProductScraped : emits
  AdJobEntity ||--o{ CopyStarted : emits
  AdJobEntity ||--o{ CopyDrafted : emits
  AdJobEntity ||--o{ VisualStarted : emits
  AdJobEntity ||--o{ VisualGenerated : emits
  AdJobEntity ||--o{ AdEvaluated : emits
  AdJobEntity ||--o{ ScrapingRejected : emits
  AdJobEntity ||--o{ BrandSafetyRejected : emits
  AdJobEntity ||--o{ AdJobFailed : emits
  AdJobView }o--|| AdJobEntity : projects
  AdPipelineWorkflow }o--|| AdJobEntity : reads-and-writes
  AdCreatorAgent ||--o{ ProductProfile : returns
  AdCreatorAgent ||--o{ AdCopy : returns
  AdCreatorAgent ||--o{ VisualSpec : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AdJobEndpoint` | `api/AdJobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AdJobEntity` | `application/AdJobEntity.java` (state in `domain/AdJobRecord.java`, events in `domain/AdJobEvent.java`) |
| `AdPipelineWorkflow` | `application/AdPipelineWorkflow.java` |
| `AdCreatorAgent` | `application/AdCreatorAgent.java` (tasks in `application/AdTasks.java`) |
| `ScrapeTools` | `application/ScrapeTools.java` |
| `CopyTools` | `application/CopyTools.java` |
| `VisualTools` | `application/VisualTools.java` |
| `ScrapingPolicyGuardrail` | `application/ScrapingPolicyGuardrail.java` |
| `BrandSafetyGuardrail` | `application/BrandSafetyGuardrail.java` |
| `AdQualityScorer` | `application/AdQualityScorer.java` |
| `AdJobView` | `application/AdJobView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `scrapeStep` 60 s, `copyStep` 60 s, `visualStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AdPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips and guardrail-driven rewrites (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + adJobId` as the workflow id; restart of the same adJobId is rejected by the workflow runtime. The agent instance id is `"agent-" + adJobId` so each job has its own per-task conversation memory.
- **One agent per job**: `AdCreatorAgent` runs three tasks per job — SCRAPE, COPY, VISUAL — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives both guardrails room to reject on the first iteration and still let the agent self-correct.
- **Dual guardrail interaction**: `ScrapingPolicyGuardrail` fires before tool calls; `BrandSafetyGuardrail` fires before agent responses. Both are registered on the same agent. A rejection from either counts as one iteration toward `maxIterationsPerTask`. The guardrails are independent — a clean tool call can still produce a brand-safety rejection, and vice versa.
- **Eval is synchronous and deterministic**: `AdQualityScorer` runs in-process inside `evalStep`. No LLM call — the same ad package always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `scrapeStep` writes `ProductScraped` BEFORE returning; `copyStep` reads the recorded `ProductProfile` from the entity to build its task's instruction context; `visualStep` reads both `ProductProfile` and `AdCopy`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed job stays at the last successful event; the UI shows the partial state for the user.
