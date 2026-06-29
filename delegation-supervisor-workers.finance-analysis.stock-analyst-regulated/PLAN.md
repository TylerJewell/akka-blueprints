# PLAN — `stock-analyst-regulated`

Architectural sketch for the Stock Analysis Team sample. Four mermaid diagrams + component table + concurrency notes.

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

  SIM[RequestSimulator]:::ta -. drips tickers .-> QUEUE[InboundRequestQueue]:::ese
  QUEUE -. events .-> CONS[RequestConsumer]:::cons
  CONS --> WF[AnalysisWorkflow]:::wf
  EP[StockAnalysisEndpoint]:::ep --> WF
  EP --> ENT[AnalysisEntity]:::ese
  WF --> NEWS[NewsResearchAgent]:::agent
  WF --> FILE[FilingsResearchAgent]:::agent
  WF --> SUMM[SummaryAgent]:::agent
  WF --> REC[RecommendationAgent]:::agent
  WF --> ENT
  ENT -. events .-> VIEW[AnalysisView]:::view
  EP --> VIEW
  DRIFT[DriftMonitor]:::ta -. samples .-> VIEW
  DRIFT --> ENT
  APP[AppEndpoint]:::ep --> STATIC[static-resources]:::ep
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant EP as StockAnalysisEndpoint
  participant WF as AnalysisWorkflow
  participant N as NewsResearchAgent
  participant F as FilingsResearchAgent
  participant S as SummaryAgent
  participant R as RecommendationAgent
  participant E as AnalysisEntity
  U->>EP: POST /api/analyses {ticker}
  EP->>WF: start(analysisId, ticker)
  WF->>E: recordQueued
  par research
    WF->>N: research(ticker)
    WF->>F: extract(ticker)
  end
  WF->>E: recordResearch
  WF->>S: summarize(bundle)
  WF->>E: recordSummary
  Note over WF: sanitize step scrubs fair-disclosure phrasing
  WF->>E: recordSanitized
  WF->>R: recommend(bundle)
  Note over R: before-agent-response guardrail<br/>enforces disclaimer + safety check
  WF->>E: recordRecommendation (ISSUED_PENDING_REVIEW)
  Note over E: awaits live compliance review
  U->>EP: POST /api/analyses/{id}/clear
  EP->>E: clear(reviewer)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> RESEARCHING
  RESEARCHING --> SUMMARIZING
  SUMMARIZING --> SANITIZING
  SANITIZING --> RECOMMENDING
  RECOMMENDING --> ISSUED_PENDING_REVIEW
  ISSUED_PENDING_REVIEW --> CLEARED: reviewer clears
  ISSUED_PENDING_REVIEW --> RETRACTED: reviewer retracts
  RESEARCHING --> FAILED
  SUMMARIZING --> FAILED
  RECOMMENDING --> FAILED
  CLEARED --> [*]
  RETRACTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ANALYSIS_ENTITY ||--o{ ANALYSIS_EVENT : emits
  ANALYSIS_ENTITY ||--|| ANALYSIS_VIEW : projects
  INBOUND_QUEUE ||--o{ QUEUE_EVENT : emits
  ANALYSIS_ENTITY {
    string id
    string ticker
    enum status
    string recommendation
    bool driftFlag
  }
  ANALYSIS_EVENT {
    string type
    string payload
  }
  ANALYSIS_VIEW {
    string id
    string ticker
    string status
  }
  INBOUND_QUEUE {
    string ticker
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| StockAnalysisEndpoint | `api/StockAnalysisEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| AnalysisWorkflow | `application/AnalysisWorkflow.java` |
| NewsResearchAgent | `application/NewsResearchAgent.java` |
| FilingsResearchAgent | `application/FilingsResearchAgent.java` |
| SummaryAgent | `application/SummaryAgent.java` |
| RecommendationAgent | `application/RecommendationAgent.java` |
| AnalysisEntity | `application/AnalysisEntity.java` |
| InboundRequestQueue | `application/InboundRequestQueue.java` |
| AnalysisView | `application/AnalysisView.java` |
| RequestConsumer | `application/RequestConsumer.java` |
| RequestSimulator | `application/RequestSimulator.java` |
| DriftMonitor | `application/DriftMonitor.java` |
| Domain records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `researchStep`, `summarizeStep`, and `recommendStep` call agents; each gets an explicit `stepTimeout(ofSeconds(60))` (Lesson 4). The 5s default would time out every LLM call.
- **Recovery.** `defaultStepRecovery(maxRetries(2).failoverTo(AnalysisWorkflow::error))` records `AnalysisFailed` and ends.
- **Idempotency.** The workflow id is the `analysisId`; re-delivery of the same `InboundRequestQueued` event uses a deterministic id derived from the queue offset so the consumer does not start duplicate analyses.
- **No compensation saga.** The sanitize and recommend steps are forward-only; the human-over-the-loop Retract is the corrective action, applied after issuance rather than as a rollback.
- **View indexing.** `AnalysisView` exposes one unfiltered query; status filtering is client-side because Akka cannot auto-index the enum column (Lesson 2).
