# PLAN — prospect-analysis

Architectural sketch for the delegation-supervisor-workers × sales-marketing cell. All four mermaid diagrams render on the Architecture tab with the Lesson 24 colour overrides.

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

  SIM[RequestSimulator]:::ta -.->|enqueueAnalysis| RQ[RequestQueueEntity]:::ese
  EP[ProspectEndpoint]:::ep -->|enqueueAnalysis| RQ
  RQ -.->|AnalysisQueued| CON[AnalysisRequestConsumer]:::cons
  CON -->|start| WF[ProspectWorkflow supervisor]:::wf
  WF -->|runSingleTask| CRA[CompanyResearchAgent]:::agent
  WF -->|runSingleTask| OAA[OrgAnalysisAgent]:::agent
  WF -->|runSingleTask| OSA[OutreachStrategyAgent]:::agent
  CRA -->|GET research tool| RT[ResearchToolEndpoint]:::ep
  WF -->|record* commands| PE[ProspectEntity]:::ese
  PE -.->|events| PV[ProspectsView]:::view
  MON[StalledAnalysisMonitor]:::ta -.->|getAllProspects| PV
  MON -->|markStalled| PE
  EP -->|getAllProspects / sse| PV
  APP[AppEndpoint]:::ep -->|static| UI[static-resources]:::ep
```

Solid arrows are synchronous commands/queries; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant EP as ProspectEndpoint
  participant RQ as RequestQueueEntity
  participant CON as AnalysisRequestConsumer
  participant WF as ProspectWorkflow
  participant CRA as CompanyResearchAgent
  participant OAA as OrgAnalysisAgent
  participant OSA as OutreachStrategyAgent
  participant PE as ProspectEntity
  U->>EP: POST /api/analyze {companyName}
  EP->>RQ: enqueueAnalysis
  RQ-->>CON: AnalysisQueued
  CON->>WF: start(prospectId)
  WF->>CRA: runSingleTask(RESEARCH)
  CRA-->>WF: CompanyProfile
  WF->>PE: recordResearch
  Note over WF,PE: status RESEARCHING -> ANALYZING
  WF->>OAA: runSingleTask(ANALYZE)
  OAA-->>WF: OrgAnalysis
  WF->>PE: recordAnalysis (sanitizer masks contactHint)
  Note over PE: status ANALYZING -> STRATEGIZING
  WF->>OSA: runSingleTask(STRATEGIZE)
  OSA-->>WF: OutreachStrategy
  WF->>PE: recordStrategy
  Note over PE: status STRATEGIZING -> COMPLETED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RESEARCHING
  RESEARCHING --> ANALYZING: recordResearch
  ANALYZING --> STRATEGIZING: recordAnalysis
  STRATEGIZING --> COMPLETED: recordStrategy
  RESEARCHING --> STALLED: markStalled
  ANALYZING --> STALLED: markStalled
  STRATEGIZING --> STALLED: markStalled
  COMPLETED --> [*]
  STALLED --> [*]
```

## Entity model

```mermaid
erDiagram
  REQUEST_QUEUE ||--o{ PROSPECT : "spawns workflow"
  PROSPECT ||--o{ PROSPECT_EVENT : emits
  PROSPECT ||--|| PROSPECTS_VIEW : projects
  PROSPECT {
    string id
    string companyName
    string domain
    enum status
  }
  PROSPECT_EVENT {
    string type "ProspectQueued|CompanyResearched|OrgAnalyzed|StrategyDeveloped|ProspectStalled"
  }
  PROSPECTS_VIEW {
    string id
    enum status
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| `CompanyResearchAgent` | `application/CompanyResearchAgent.java` |
| `OrgAnalysisAgent` | `application/OrgAnalysisAgent.java` |
| `OutreachStrategyAgent` | `application/OutreachStrategyAgent.java` |
| `ProspectWorkflow` | `application/ProspectWorkflow.java` |
| `ProspectTasks` | `application/ProspectTasks.java` |
| `ProspectEntity` | `application/ProspectEntity.java` |
| `RequestQueueEntity` | `application/RequestQueueEntity.java` |
| `ProspectsView` | `application/ProspectsView.java` |
| `AnalysisRequestConsumer` | `application/AnalysisRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StalledAnalysisMonitor` | `application/StalledAnalysisMonitor.java` |
| `ProspectEndpoint` | `api/ProspectEndpoint.java` |
| `ResearchToolEndpoint` | `api/ResearchToolEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Prospect`, records, enum | `domain/` |

## Concurrency notes

- Each agent-calling workflow step (`researchStep`, `analyzeStep`, `strategizeStep`) sets `stepTimeout(ofSeconds(60))` — the 5s default would expire mid-LLM-call (Lesson 4). `defaultStepRecovery(maxRetries(2).failoverTo(error))` routes a persistently failing step to a terminal error step rather than looping forever.
- Idempotency: the workflow is keyed by `prospectId` (a fresh UUID per `AnalysisQueued`), so a redelivered queue event cannot start a duplicate analysis for the same prospect instance.
- No multi-step saga/compensation: phases are append-only event records on `ProspectEntity`. A failed step leaves the prospect in its last good state until the monitor marks it `STALLED`; no rollback of earlier phases is required.
- The before-tool-call guardrail (G1) is synchronous and blocking — an off-domain call is refused before the worker's tool invocation reaches `ResearchToolEndpoint`.
