# PLAN — competitor-research-pipeline

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
  classDef ext fill:#1a1a2e,stroke:#8888cc,color:#8888cc;

  API[CompetitorEndpoint]:::ep
  Entity[CompetitorEntity]:::ese
  WF[CompetitorPipelineWorkflow]:::wf
  Agent[ResearchAgent]:::agent
  Search[SearchTools]:::tool
  Summarize[SummarizeTools]:::tool
  Publish[PublishTools]:::tool
  Guard[NotionWriteGuardrail]:::guard
  Scorer[PublishQualityScorer]:::guard
  View[CompetitorView]:::view
  App[AppEndpoint]:::ep
  Exa[Exa.ai API]:::ext
  Notion[Notion API]:::ext

  API -->|create| Entity
  API -->|start| WF
  WF -->|searchStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Search
  Agent -->|invokes| Summarize
  Agent -->|invokes| Publish
  Guard -->|recordGuardrailRejection| Entity
  Search -->|live mode| Exa
  Publish -->|writeToNotion live| Notion
  Agent -->|SearchResultSet / ProfileSummary / CompetitorProfile| WF
  WF -->|recordResults/Summary/Profile| Entity
  WF -->|evalStep score| Scorer
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
  participant API as CompetitorEndpoint
  participant E as CompetitorEntity
  participant W as CompetitorPipelineWorkflow
  participant A as ResearchAgent
  participant G as NotionWriteGuardrail
  participant T as Tools (Search/Summarize/Publish)
  participant Sc as PublishQualityScorer

  U->>API: POST /api/competitors { name }
  API->>E: create(name)
  E-->>API: { competitorId }
  API->>W: start(competitorId, name)
  W->>E: startSearch
  W->>A: runSingleTask(SEARCH_COMPETITOR, name)
  A->>G: before-tool-call(searchCompetitor, SEARCH)
  G-->>A: accept (status SEARCHING)
  A->>T: searchCompetitor + fetchPage
  T-->>A: List<SearchResult>
  A-->>W: SearchResultSet
  W->>E: recordResults
  W->>A: runSingleTask(SUMMARIZE_FINDINGS, results)
  A->>G: before-tool-call(extractFields, SUMMARIZE)
  G-->>A: accept (status SUMMARIZING and results present)
  A->>T: extractFields + classifyDomain
  T-->>A: List<ProfileField> / String
  A-->>W: ProfileSummary
  W->>E: recordSummary
  W->>A: runSingleTask(PUBLISH_PROFILE, summary)
  A->>G: before-tool-call(buildNotionPage, PUBLISH)
  G-->>A: accept (status PUBLISHING and summary present)
  A->>T: buildNotionPage
  T-->>A: Map<String,Object> payload
  A->>G: before-tool-call(writeToNotion, PUBLISH)
  G-->>A: accept (schema+scope pass)
  A->>T: writeToNotion
  T-->>A: NotionPageRef
  A-->>W: CompetitorProfile
  W->>E: recordProfile
  W->>Sc: score(profile, summary, results)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `CompetitorEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SEARCHING: SearchStarted
  SEARCHING --> SEARCHED: ResultsFetched
  SEARCHED --> SUMMARIZING: SummarizeStarted
  SUMMARIZING --> SUMMARIZED: SummaryProduced
  SUMMARIZED --> PUBLISHING: PublishStarted
  PUBLISHING --> PUBLISHED: ProfilePublished
  PUBLISHED --> EVALUATED: PublishEvaluated
  SEARCHING --> FAILED: CompetitorFailed
  SUMMARIZING --> FAILED: CompetitorFailed
  PUBLISHING --> FAILED: CompetitorFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  CompetitorEntity ||--o{ CompetitorCreated : emits
  CompetitorEntity ||--o{ SearchStarted : emits
  CompetitorEntity ||--o{ ResultsFetched : emits
  CompetitorEntity ||--o{ SummarizeStarted : emits
  CompetitorEntity ||--o{ SummaryProduced : emits
  CompetitorEntity ||--o{ PublishStarted : emits
  CompetitorEntity ||--o{ ProfilePublished : emits
  CompetitorEntity ||--o{ PublishEvaluated : emits
  CompetitorEntity ||--o{ GuardrailRejected : emits
  CompetitorEntity ||--o{ CompetitorFailed : emits
  CompetitorView }o--|| CompetitorEntity : projects
  CompetitorPipelineWorkflow }o--|| CompetitorEntity : reads-and-writes
  ResearchAgent ||--o{ SearchResultSet : returns
  ResearchAgent ||--o{ ProfileSummary : returns
  ResearchAgent ||--o{ CompetitorProfile : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CompetitorEndpoint` | `api/CompetitorEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CompetitorEntity` | `application/CompetitorEntity.java` (state in `domain/CompetitorRecord.java`, events in `domain/CompetitorEvent.java`) |
| `CompetitorPipelineWorkflow` | `application/CompetitorPipelineWorkflow.java` |
| `ResearchAgent` | `application/ResearchAgent.java` (tasks in `application/ResearchTasks.java`) |
| `SearchTools` | `application/SearchTools.java` |
| `SummarizeTools` | `application/SummarizeTools.java` |
| `PublishTools` | `application/PublishTools.java` |
| `NotionWriteGuardrail` | `application/NotionWriteGuardrail.java` |
| `NotionSchema` | `application/NotionSchema.java` |
| `PublishQualityScorer` | `application/PublishQualityScorer.java` |
| `CompetitorView` | `application/CompetitorView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `searchStep` 90 s, `summarizeStep` 90 s, `publishStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CompetitorPipelineWorkflow::error)`. The 90 s on each agent-calling step accommodates LLM latency plus the round-trip to Exa.ai or Notion (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + competitorId` as the workflow id; restart of the same competitorId is rejected by the workflow runtime. The agent instance id is `"agent-" + competitorId` so each competitor has its own per-task conversation memory.
- **One agent per competitor**: `ResearchAgent` runs three tasks per competitor — SEARCH, SUMMARIZE, PUBLISH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered or schema-violating call and still let the agent self-correct.
- **Guardrail-driven retry**: when `NotionWriteGuardrail` rejects a tool call (phase-violation or schema-violation), the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `PublishQualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same profile always scores the same.
- **Task-boundary handoff is the dependency contract**: `searchStep` writes `ResultsFetched` BEFORE returning; `summarizeStep` reads the recorded `SearchResultSet` from the entity to build its task's instruction context; `publishStep` reads both `SearchResultSet` and `ProfileSummary`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed competitor stays at the last successful event; the UI shows the partial state for the user.
