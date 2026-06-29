# PLAN — reddit-search

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef scorer fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[ResearchEndpoint]:::ep
  Entity[ResearchJobEntity]:::ese
  WF[ResearchWorkflow]:::wf
  Agent[BrowserResearchAgent]:::agent
  Guard[NavigationGuardrail]:::guard
  Budget[BudgetEnforcer]:::scorer
  Scorer[RelevanceScorer]:::scorer
  View[ResearchView]:::view
  App[AppEndpoint]:::ep

  API -->|enqueue + start workflow| Entity
  API -->|start| WF
  WF -->|browseStep runSingleTask| Agent
  Agent -.->|before-tool-call navigate| Guard
  Guard -->|allow/reject| Agent
  WF -->|checkBudget per page| Budget
  Budget -->|BudgetExhausted| Entity
  Agent -->|ResearchReport| WF
  WF -->|scoreStep re-rank| Scorer
  Scorer -->|ranked report| WF
  WF -->|recordReport| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ResearchEndpoint
  participant E as ResearchJobEntity
  participant W as ResearchWorkflow
  participant A as BrowserResearchAgent
  participant NG as NavigationGuardrail
  participant BE as BudgetEnforcer
  participant RS as RelevanceScorer

  U->>API: POST /api/jobs
  API->>E: enqueue(topic)
  E-->>API: { jobId }
  API->>W: start(jobId)
  W->>E: markBrowsing
  W->>A: runSingleTask(topic instructions)
  loop per page visit
    A->>NG: before-tool-call navigate(url)
    NG-->>A: allow
    A->>E: recordPageVisit(url)
    W->>BE: checkBudget(jobId)
    BE-->>W: OK
  end
  A-->>W: ResearchReport (raw)
  W->>RS: score(report)
  RS-->>W: ranked report
  W->>E: recordReport(report)
  E-.->>U: SSE event(REPORT_READY)
```

## State machine — `ResearchJobEntity`

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> BROWSING: BrowsingStarted
  BROWSING --> REPORT_READY: ReportRecorded
  BROWSING --> BUDGET_EXHAUSTED: BudgetExhausted
  BROWSING --> FAILED: JobFailed (agent error)
  QUEUED --> FAILED: JobFailed (workflow error)
  REPORT_READY --> [*]
  BUDGET_EXHAUSTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchJobEntity ||--o{ JobQueued : emits
  ResearchJobEntity ||--o{ BrowsingStarted : emits
  ResearchJobEntity ||--o{ PageVisited : emits
  ResearchJobEntity ||--o{ ReportRecorded : emits
  ResearchJobEntity ||--o{ BudgetExhausted : emits
  ResearchJobEntity ||--o{ JobFailed : emits
  ResearchView }o--|| ResearchJobEntity : projects
  ResearchWorkflow }o--|| ResearchJobEntity : reads-and-writes
  BrowserResearchAgent ||--o{ ResearchReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ResearchJobEntity` | `application/ResearchJobEntity.java` (state in `domain/ResearchJob.java`, events in `domain/ResearchJobEvent.java`) |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `BrowserResearchAgent` | `application/BrowserResearchAgent.java` (tasks in `application/ResearchTasks.java`) |
| `NavigationGuardrail` | `application/NavigationGuardrail.java` |
| `BudgetEnforcer` | `application/BudgetEnforcer.java` |
| `RelevanceScorer` | `application/RelevanceScorer.java` |
| `ResearchView` | `application/ResearchView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `browseStep` 180 s (accommodates LLM latency plus multiple page loads), `scoreStep` 10 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(ResearchWorkflow::error)`. The 180 s on `browseStep` reflects that a 20-page session with real browser round-trips can approach 2–3 minutes (Lesson 4).
- **Idempotency**: every workflow uses `"research-" + jobId` as the workflow id. `ResearchJobEntity.recordPageVisit` is guarded by a URL dedup set on the entity so a redelivered `PageVisited` event with the same URL is a no-op.
- **One agent per job**: the AutonomousAgent instance id is `"researcher-" + jobId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` caps navigation retries at 5.
- **Budget halt path**: when `BudgetEnforcer` returns EXCEEDED, `browseStep` stops the agent task immediately via a workflow transition to a `budgetExhausted` terminal step, which calls `ResearchJobEntity.recordBudgetExhausted(pagesVisited)`. The partial ResearchReport is preserved as the final output — not discarded.
- **Guardrail-driven retry**: when `NavigationGuardrail` rejects a URL, the rejection is a structured error returned to the agent loop. The loop counts toward `maxIterationsPerTask`; the agent is expected to propose an alternative URL rather than retrying the blocked one.
- **Scorer is synchronous and deterministic**: `RelevanceScorer` runs in-process inside `scoreStep`. No LLM call, no external service.
- **No saga / no compensation**: steps are append-only entity writes plus a single-task agent call. Nothing external to roll back.
