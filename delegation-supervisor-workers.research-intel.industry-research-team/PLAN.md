# PLAN — industry-research-team

Architectural sketch. All four mermaid diagrams + the component table.

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

  SIM[RequestSimulator]:::ta -. drips .-> QUEUE
  EP[ResearchEndpoint]:::ep -- enqueue --> QUEUE[InboundRequestQueue]:::ese
  QUEUE -. events .-> CONS[RequestConsumer]:::cons
  CONS -- start --> WF[ResearchWorkflow]:::wf
  WF -- PLAN/SYNTHESIZE --> SUP[SupervisorAgent]:::agent
  WF -- ANALYZE per sector --> WORK[SectorAnalystAgent]:::agent
  WF -- record events --> BRIEF[BriefEntity]:::ese
  BRIEF -. events .-> VIEW[BriefsView]:::view
  EP -- query/SSE --> VIEW
  APP[AppEndpoint]:::ep -- static --> UI[(static-resources)]
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User / Simulator
  participant EP as ResearchEndpoint
  participant Q as InboundRequestQueue
  participant C as RequestConsumer
  participant WF as ResearchWorkflow
  participant SUP as SupervisorAgent
  participant WK as SectorAnalystAgent
  participant E as BriefEntity
  U->>EP: POST /api/research-request
  EP->>Q: enqueueRequest
  Q-->>C: InboundRequestQueued
  C->>WF: start (fresh UUID)
  WF->>SUP: PLAN(request)
  SUP-->>WF: ResearchPlan{title, sectors}
  WF->>E: recordPlan (PLANNED -> ANALYZING)
  loop each sector
    WF->>WK: ANALYZE(request, sector)
    WK-->>WF: SectorAnalysis
    WF->>E: recordSectorFinding
  end
  WF->>SUP: SYNTHESIZE(request, findings)
  SUP-->>WF: SynthesizedBrief{body, groundingScore}
  Note over WF: grounding guardrail (before-agent-response)
  alt groundingScore >= 0.6
    WF->>E: recordSynthesis (SYNTHESIZED)
    WF->>E: recordSanitization + recordCompletion (SANITIZED -> COMPLETED)
  else below bar
    WF->>E: recordBlock (BLOCKED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PLANNED
  PLANNED --> ANALYZING: recordPlan
  ANALYZING --> SYNTHESIZED: recordSynthesis
  ANALYZING --> BLOCKED: recordBlock (grounding < bar)
  SYNTHESIZED --> SANITIZED: recordSanitization
  SANITIZED --> COMPLETED: recordCompletion
  BLOCKED --> [*]
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  BRIEF ||--o{ SECTOR_FINDING : aggregates
  BRIEF {
    string id
    string status
    string request
    double groundingScore
  }
  SECTOR_FINDING {
    string sector
    string summary
  }
  BRIEF_EVENT {
    string type
  }
  BRIEFS_VIEW {
    string id
    string status
  }
  BRIEF ||--o{ BRIEF_EVENT : emits
  BRIEF_EVENT }o--|| BRIEFS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| SupervisorAgent | `application/SupervisorAgent.java` |
| SectorAnalystAgent | `application/SectorAnalystAgent.java` |
| IndustryResearchTasks | `application/IndustryResearchTasks.java` |
| ResearchWorkflow | `application/ResearchWorkflow.java` |
| BriefEntity | `application/BriefEntity.java` |
| InboundRequestQueue | `application/InboundRequestQueue.java` |
| BriefsView | `application/BriefsView.java` |
| RequestConsumer | `application/RequestConsumer.java` |
| RequestSimulator | `application/RequestSimulator.java` |
| ResearchEndpoint | `api/ResearchEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Brief, SectorFinding, records | `domain/*.java` |

## Concurrency notes

- **Step timeouts:** `planStep`, `analyzeStep`, `synthesizeStep` each call an agent — set `stepTimeout(60s)` explicitly (Lesson 4). The default 5s timeout would fail every LLM call.
- **Idempotency:** `RequestConsumer` derives the workflow id from a fresh UUID per queued event; re-delivery of the same `InboundRequestQueued` event is tolerated because the workflow start is keyed on that id.
- **Fan-out:** `analyzeStep` processes sectors sequentially per brief; each `SectorAnalyzed` event is recorded as it returns, so a mid-loop failure leaves recorded findings intact.
- **Compensation:** a synthesis below the grounding bar is not a failure — it is a terminal `BLOCKED` transition recorded on the entity, not a retry. `defaultStepRecovery(maxRetries(2).failoverTo(errorStep))` covers transient agent errors only.
