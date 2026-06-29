# PLAN — market-researcher

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

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

  Poller[ResearchPoller]:::ta
  Queue[ResearchQueue]:::ese
  Ingest[ResearchIngestConsumer]:::cons
  Extractor[SignalExtractorAgent]:::agent
  Synthesiser[SectorSynthesisAgent]:::agent
  WF[SectorSynthesisWorkflow]:::wf
  ItemEntity[ResearchItemEntity]:::ese
  SectorEntity[SectorStateEntity]:::ese
  ResearchView[ResearchView]:::view
  SectorView[SectorView]:::view
  EvalRunner[EvalRunner]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 30s| Queue
  Queue -.->|subscribes| Ingest
  Ingest -->|normalise + register| ItemEntity
  Ingest -->|trigger per sector window| WF
  WF -->|call| Extractor
  WF -->|call (if signals present)| Synthesiser
  WF -->|guardrail check| WF
  WF -->|emit events| ItemEntity
  WF -->|commit or pending| SectorEntity
  ItemEntity -.->|projects| ResearchView
  SectorEntity -.->|projects| SectorView
  API -->|confirm/dismiss| ItemEntity
  API -->|query/SSE| ResearchView
  API -->|query/SSE| SectorView
  EvalRunner -.->|every 24h| ItemEntity
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as ResearchPoller
  participant Q as ResearchQueue
  participant C as ResearchIngestConsumer
  participant I as ResearchItemEntity
  participant W as SectorSynthesisWorkflow
  participant X as SignalExtractorAgent
  participant S as SectorSynthesisAgent
  participant SE as SectorStateEntity
  participant U as Analyst (UI)
  participant API as ResearchEndpoint

  P->>Q: emit ItemReceived
  Q->>C: ItemReceived
  C->>I: registerIncoming + attachNormalised
  C->>W: start({sector, windowId})
  W->>X: extract(normalisedItem)
  X-->>W: SignalSet
  W->>I: emit SignalsExtracted
  W->>S: synthesise(signalSets, sector)
  S-->>W: SectorBrief (riskFlag=ELEVATED, confidenceScore=0.82)
  Note over W: guardrailStep: riskFlag != HIGH → pass
  W->>SE: emit BriefUpdated
  W->>I: emit ItemFlagged
  Note over I,U: Item visible in UI as FLAGGED
  U->>API: POST /api/research/{id}/confirm-flag
  API->>I: emit ItemFlagged (confirmed)
```

## State machine — `ResearchItemEntity`

```mermaid
stateDiagram-v2
  [*] --> INGESTED
  INGESTED --> NORMALISED: ItemNormalised
  NORMALISED --> SIGNALS_EXTRACTED: SignalsExtracted
  SIGNALS_EXTRACTED --> SYNTHESISED: ItemSynthesised
  SYNTHESISED --> FLAGGED: guardrail passes (riskFlag HIGH or ELEVATED with confidence >= 0.7)
  SYNTHESISED --> PENDING_REVIEW: guardrail blocks (HIGH risk + confidence < 0.7)
  SYNTHESISED --> ARCHIVED: riskFlag = NONE
  PENDING_REVIEW --> FLAGGED: analyst confirms
  PENDING_REVIEW --> ARCHIVED: analyst dismisses
  FLAGGED --> [*]
  ARCHIVED --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchItemEntity ||--o{ ItemReceived : emits
  ResearchItemEntity ||--o{ ItemNormalised : emits
  ResearchItemEntity ||--o{ SignalsExtracted : emits
  ResearchItemEntity ||--o{ ItemSynthesised : emits
  ResearchItemEntity ||--o{ ItemFlagged : emits
  ResearchItemEntity ||--o{ ItemPendingReview : emits
  ResearchItemEntity ||--o{ ItemArchived : emits
  ResearchItemEntity ||--o{ EvalScored : emits
  SectorStateEntity ||--o{ BriefUpdated : emits
  SectorStateEntity ||--o{ BriefPendingReview : emits
  ResearchView }o--|| ResearchItemEntity : projects
  SectorView }o--|| SectorStateEntity : projects
  ResearchQueue ||--o{ ItemReceived : emits
  ResearchIngestConsumer }o--|| ResearchQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchPoller` | `application/ResearchPoller.java` |
| `ResearchQueue` | `application/ResearchQueue.java` |
| `ResearchIngestConsumer` | `application/ResearchIngestConsumer.java` |
| `SignalExtractorAgent` | `application/SignalExtractorAgent.java` |
| `SectorSynthesisAgent` | `application/SectorSynthesisAgent.java` |
| `SectorSynthesisWorkflow` | `application/SectorSynthesisWorkflow.java` |
| `ResearchItemEntity` | `application/ResearchItemEntity.java` (state in `domain/ResearchItemState.java`, events in `domain/ResearchItemEvent.java`) |
| `SectorStateEntity` | `application/SectorStateEntity.java` (state in `domain/SectorState.java`, events in `domain/SectorEvent.java`) |
| `ResearchView` | `application/ResearchView.java` |
| `SectorView` | `application/SectorView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: signal extraction 15 s, sector synthesis 45 s. On timeout, escalate the item to PENDING_REVIEW rather than discarding.
- **Guardrail gate**: `guardrailStep` runs synchronously after `SectorSynthesisAgent` returns — no extra LLM call. Decision is deterministic on `riskFlag` + `confidenceScore`.
- **Idempotency**: workflow id is `(canonicalSector + ":" + windowId)`. Duplicate `ItemReceived` events for the same window fold into one workflow invocation.
- **Eval sampling**: per tick, EvalRunner picks up to 5 FLAGGED items with no `evalScore`, oldest-first.
