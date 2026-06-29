# PLAN — web-scraper-agent

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
  classDef tool fill:#0e1e0e,stroke:#57ff8c,color:#57ff8c;

  API[ScrapeEndpoint]:::ep
  Entity[ScrapeEntity]:::ese
  Sanitizer[ContentSanitizer]:::cons
  WF[ScrapeWorkflow]:::wf
  Agent[WebScraperAgent]:::agent
  Guard[UrlGuardrail]:::guard
  Tool[FetchPageTool]:::tool
  View[ScrapeView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|markFetching| Entity
  WF -->|submitStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|pass/reject| Agent
  Agent -->|fetchPage tool call| Tool
  Tool -->|raw HTML| Agent
  Agent -->|ScrapeResult| WF
  WF -->|recordResult| Entity
  Entity -.->|RawContentExtracted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ScrapeEndpoint
  participant E as ScrapeEntity
  participant W as ScrapeWorkflow
  participant A as WebScraperAgent
  participant Gd as UrlGuardrail
  participant T as FetchPageTool
  participant S as ContentSanitizer

  U->>API: POST /api/scrapes
  API->>E: submit(request)
  E-->>API: { scrapeId }
  API->>W: start(scrapeId)
  W->>E: markFetching
  W->>A: runSingleTask(url + schema)
  A->>Gd: before-tool-call(fetchPage, url)
  Gd-->>A: accept
  A->>T: fetchPage(url)
  T-->>A: raw HTML
  A-->>W: ScrapeResult
  W->>E: recordResult(result)
  E-.->>S: RawContentExtracted
  S->>S: redact PII
  S->>E: attachSanitized
  W->>E: poll getScrape
  E-->>W: sanitized.isPresent()
  E-.->>U: SSE event(SANITIZED)
```

## State machine — `ScrapeEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> FETCHING: ScrapeFetching
  FETCHING --> EXTRACTED: RawContentExtracted
  EXTRACTED --> SANITIZED: ContentSanitized
  FETCHING --> BLOCKED: ScrapeBlocked (guardrail rejected)
  FETCHING --> FAILED: ScrapeFailed (agent error)
  SUBMITTED --> FAILED: ScrapeFailed (workflow error)
  SANITIZED --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ScrapeEntity ||--o{ ScrapeSubmitted : emits
  ScrapeEntity ||--o{ ScrapeFetching : emits
  ScrapeEntity ||--o{ RawContentExtracted : emits
  ScrapeEntity ||--o{ ContentSanitized : emits
  ScrapeEntity ||--o{ ScrapeBlocked : emits
  ScrapeEntity ||--o{ ScrapeFailed : emits
  ScrapeView }o--|| ScrapeEntity : projects
  ContentSanitizer }o--|| ScrapeEntity : subscribes
  ScrapeWorkflow }o--|| ScrapeEntity : reads-and-writes
  WebScraperAgent ||--o{ ScrapeResult : returns
  UrlGuardrail }o--|| WebScraperAgent : guards-tool-call
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ScrapeEndpoint` | `api/ScrapeEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ScrapeEntity` | `application/ScrapeEntity.java` (state in `domain/Scrape.java`, events in `domain/ScrapeEvent.java`) |
| `ContentSanitizer` | `application/ContentSanitizer.java` |
| `ScrapeWorkflow` | `application/ScrapeWorkflow.java` |
| `WebScraperAgent` | `application/WebScraperAgent.java` (tasks in `application/ScrapeTasks.java`) |
| `UrlGuardrail` | `application/UrlGuardrail.java` |
| `FetchPageTool` | `application/FetchPageTool.java` |
| `AllowlistLoader` | `application/AllowlistLoader.java` |
| `RateLimitStore` | `application/RateLimitStore.java` |
| `ScrapeView` | `application/ScrapeView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `submitStep` 90 s, `awaitSanitizedStep` 15 s, `done` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ScrapeWorkflow::error)`. The 90 s on `submitStep` accommodates LLM latency plus HTTP fetch time (Lesson 4).
- **Idempotency**: every workflow uses `"scrape-" + scrapeId` as the workflow id. The `ContentSanitizer` Consumer is event-version-guarded — a redelivery of `RawContentExtracted` against an already-sanitized entity is a no-op.
- **One agent per scrape**: the AutonomousAgent instance id is `"scraper-" + scrapeId`, giving each task its own context. `capability(...).maxIterationsPerTask(3)` caps internal retries.
- **Guardrail on the tool, not the agent**: `UrlGuardrail` is bound to `before-tool-call` on `fetchPage`. It fires for every invocation attempt, including retries. The rate-limit window is shared across all scrape sessions for the same origin — `RateLimitStore` is a singleton bean.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
- **BLOCKED is a terminal state**: when `UrlGuardrail` rejects a tool call, the agent receives the rejection as a structured tool-output error, returns a `ScrapeResult` indicating the block, and the workflow writes `ScrapeBlocked`. The entity transitions to `BLOCKED` (terminal). The agent is not retried on a guardrail-blocked URL — retrying would not change the allowlist check.
