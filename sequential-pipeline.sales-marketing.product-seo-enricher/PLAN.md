# PLAN — product-seo-enricher

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

  API[EnrichmentEndpoint]:::ep
  Entity[EnrichmentEntity]:::ese
  WF[EnrichmentPipelineWorkflow]:::wf
  Agent[SeoAgent]:::agent
  Fetch[FetchTools]:::tool
  Analyze[AnalyzeTools]:::tool
  Enrich[EnrichTools]:::tool
  Guard[BrowserGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[EnrichmentView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|fetchStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Fetch
  Agent -->|invokes| Analyze
  Agent -->|invokes| Enrich
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|SerpResult / SerpAnalysis / ProductEnrichment| WF
  WF -->|recordSerpResult/Analysis/Enrichment| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordQuality| Entity
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
  participant API as EnrichmentEndpoint
  participant E as EnrichmentEntity
  participant W as EnrichmentPipelineWorkflow
  participant A as SeoAgent
  participant G as BrowserGuardrail
  participant T as Tools (Fetch/Analyze/Enrich)
  participant Sc as QualityScorer

  U->>API: POST /api/enrichments { productName }
  API->>E: create(productName)
  E-->>API: { enrichmentId }
  API->>W: start(enrichmentId, productName)
  W->>E: startFetch
  W->>A: runSingleTask(FETCH_SERP, productName)
  A->>G: before-tool-call(fetchSerpPage, FETCH)
  G-->>A: accept (status FETCHING)
  A->>T: fetchSerpPage + fetchResultSnippet
  T-->>A: List<SerpEntry>
  A-->>W: SerpResult
  W->>E: recordSerpResult
  W->>A: runSingleTask(ANALYZE_SERP, serpResult)
  A->>G: before-tool-call(extractKeywords, ANALYZE)
  G-->>A: accept (status ANALYZING and serpResult present)
  A->>T: extractKeywords + identifyCompetitors
  T-->>A: List<KeywordCandidate> / List<CompetitorSignal>
  A-->>W: SerpAnalysis
  W->>E: recordSerpAnalysis
  W->>A: runSingleTask(WRITE_ENRICHMENT, serpAnalysis)
  A->>G: before-tool-call(rankKeywords, ENRICH)
  G-->>A: accept (status ENRICHING and serpAnalysis present)
  A->>T: rankKeywords + writeMetaDescription
  T-->>A: List<RankedKeyword> / String
  A-->>W: ProductEnrichment
  W->>E: recordEnrichment
  W->>Sc: score(enrichment, serpAnalysis, serpResult)
  Sc-->>W: QualityResult
  W->>E: recordQuality
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `EnrichmentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> FETCHING: FetchStarted
  FETCHING --> FETCHED: SerpFetched
  FETCHED --> ANALYZING: AnalyzeStarted
  ANALYZING --> ANALYZED: SerpAnalyzed
  ANALYZED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: EnrichmentWritten
  ENRICHED --> EVALUATED: QualityScored
  FETCHING --> FAILED: EnrichmentFailed
  ANALYZING --> FAILED: EnrichmentFailed
  ENRICHING --> FAILED: EnrichmentFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  EnrichmentEntity ||--o{ EnrichmentCreated : emits
  EnrichmentEntity ||--o{ FetchStarted : emits
  EnrichmentEntity ||--o{ SerpFetched : emits
  EnrichmentEntity ||--o{ AnalyzeStarted : emits
  EnrichmentEntity ||--o{ SerpAnalyzed : emits
  EnrichmentEntity ||--o{ EnrichStarted : emits
  EnrichmentEntity ||--o{ EnrichmentWritten : emits
  EnrichmentEntity ||--o{ QualityScored : emits
  EnrichmentEntity ||--o{ GuardrailRejected : emits
  EnrichmentEntity ||--o{ EnrichmentFailed : emits
  EnrichmentView }o--|| EnrichmentEntity : projects
  EnrichmentPipelineWorkflow }o--|| EnrichmentEntity : reads-and-writes
  SeoAgent ||--o{ SerpResult : returns
  SeoAgent ||--o{ SerpAnalysis : returns
  SeoAgent ||--o{ ProductEnrichment : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EnrichmentEndpoint` | `api/EnrichmentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `EnrichmentEntity` | `application/EnrichmentEntity.java` (state in `domain/EnrichmentRecord.java`, events in `domain/EnrichmentEvent.java`) |
| `EnrichmentPipelineWorkflow` | `application/EnrichmentPipelineWorkflow.java` |
| `SeoAgent` | `application/SeoAgent.java` (tasks in `application/SeoTasks.java`) |
| `FetchTools` | `application/FetchTools.java` |
| `AnalyzeTools` | `application/AnalyzeTools.java` |
| `EnrichTools` | `application/EnrichTools.java` |
| `BrowserGuardrail` | `application/BrowserGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `EnrichmentView` | `application/EnrichmentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `fetchStep` 60 s, `analyzeStep` 60 s, `enrichStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(EnrichmentPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + enrichmentId` as the workflow id; restart of the same enrichmentId is rejected by the workflow runtime. The agent instance id is `"agent-" + enrichmentId` so each enrichment has its own per-task conversation memory.
- **One agent per enrichment**: `SeoAgent` runs three tasks per enrichment — FETCH, ANALYZE, ENRICH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `BrowserGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same enrichment always scores the same.
- **Task-boundary handoff is the dependency contract**: `fetchStep` writes `SerpFetched` BEFORE returning; `analyzeStep` reads the recorded `SerpResult` from the entity to build its task's instruction context; `enrichStep` reads both `SerpResult` and `SerpAnalysis`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed enrichment stays at the last successful event; the UI shows the partial state for the user.
