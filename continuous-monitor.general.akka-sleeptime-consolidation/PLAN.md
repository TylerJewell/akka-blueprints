# PLAN — sleeptime-consolidation

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {
  "primaryColor": "#0e1e2a",
  "primaryTextColor": "#7EC8E3",
  "primaryBorderColor": "#7EC8E3",
  "lineColor": "#aab3bd",
  "secondaryColor": "#1c1330",
  "tertiaryColor": "#1f1900"
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Trigger[ConsolidationTrigger]:::ta
  Counter[InteractionCounterEntity]:::ese
  Block[MemoryBlockEntity]:::ese
  WF[ConsolidationWorkflow]:::wf
  Primary[PrimaryAgent]:::agent
  Consolidator[SleeptimeConsolidatorAgent]:::agent
  DriftEval[DriftEvalAgent]:::agent
  View[MemoryView]:::view
  DriftRunner[DriftEvalRunner]:::ta
  API[MemoryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -.->|incrementStep| Counter
  Counter -.->|projects| View
  Trigger -.->|every 30s| View
  Trigger -->|start workflow per block| WF
  WF -->|loadStep| Block
  WF -->|consolidateStep| Consolidator
  WF -->|commitStep| Block
  Block -.->|projects| View
  API -->|query/SSE| View
  API -->|createBlock / appendContent| Block
  API -->|agent/query| Primary
  Primary -.->|reads| Block
  DriftRunner -.->|every 30m| Block
  DriftRunner -->|call| DriftEval
```

## Interaction sequence — J1 (consolidation cycle)

```mermaid
sequenceDiagram
  autonumber
  participant C as ConsolidationTrigger
  participant V as MemoryView
  participant WF as ConsolidationWorkflow
  participant B as MemoryBlockEntity
  participant S as SleeptimeConsolidatorAgent
  participant U as Operator (UI)
  participant API as MemoryEndpoint

  C->>V: getAllBlocks (find sessions ≥ N steps)
  V-->>C: [eligible blocks]
  C->>WF: start(blockId)
  WF->>B: beginConsolidation(blockId)
  B-->>WF: ConsolidationStarted emitted (status → CONSOLIDATING)
  Note over B,U: SSE streams ConsolidationStarted to operator
  WF->>S: consolidate(rawContent, topicHints, priorConsolidated)
  S-->>WF: ConsolidatedContent
  WF->>B: applyConsolidation(consolidated)
  B-->>WF: BlockConsolidated emitted (status → CONSOLIDATED)
  Note over B,U: SSE streams BlockConsolidated to operator
  API-->>U: sse event: block-update
```

## State machine — `MemoryBlockEntity`

```mermaid
stateDiagram-v2
  [*] --> ACTIVE : BlockCreated
  ACTIVE --> CONSOLIDATING : ConsolidationStarted
  CONSOLIDATING --> CONSOLIDATED : BlockConsolidated
  CONSOLIDATING --> STALE : consolidateStep timeout
  CONSOLIDATED --> ACTIVE : BlockUpdated (new content appended)
  CONSOLIDATED --> CONSOLIDATING : next ConsolidationStarted
  STALE --> [*]
```

## Entity model

```mermaid
erDiagram
  MemoryBlockEntity ||--o{ BlockCreated : emits
  MemoryBlockEntity ||--o{ BlockUpdated : emits
  MemoryBlockEntity ||--o{ ConsolidationStarted : emits
  MemoryBlockEntity ||--o{ BlockConsolidated : emits
  MemoryBlockEntity ||--o{ BlockMarkedStale : emits
  MemoryBlockEntity ||--o{ DriftScored : emits
  MemoryBlockEntity ||--o{ GuardrailTriggered : emits
  InteractionCounterEntity ||--o{ StepRecorded : emits
  InteractionCounterEntity ||--o{ CounterReset : emits
  MemoryView }o--|| MemoryBlockEntity : projects
  MemoryView }o--|| InteractionCounterEntity : projects
  ConsolidationWorkflow }o--|| MemoryBlockEntity : drives
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConsolidationTrigger` | `application/ConsolidationTrigger.java` |
| `InteractionCounterEntity` | `application/InteractionCounterEntity.java` |
| `MemoryBlockEntity` | `application/MemoryBlockEntity.java` (state in `domain/MemoryBlock.java`, events in `domain/MemoryBlockEvent.java`) |
| `ConsolidationWorkflow` | `application/ConsolidationWorkflow.java` |
| `PrimaryAgent` | `application/PrimaryAgent.java` |
| `SleeptimeConsolidatorAgent` | `application/SleeptimeConsolidatorAgent.java` |
| `DriftEvalAgent` | `application/DriftEvalAgent.java` |
| `MemoryView` | `application/MemoryView.java` |
| `DriftEvalRunner` | `application/DriftEvalRunner.java` |
| `MemoryEndpoint` | `api/MemoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Consolidation guard**: `ConsolidationWorkflow` id equals `blockId`. A second `start(blockId)` call while a workflow is running returns the running instance — no double-consolidation.
- **Timeout**: `consolidateStep` carries `stepTimeout(Duration.ofSeconds(60))`. On timeout, the workflow emits `BlockMarkedStale` and terminates gracefully.
- **Drift sampling**: per tick, `DriftEvalRunner` picks up to 5 CONSOLIDATED blocks with no `driftScore`, oldest-first, to bound per-tick LLM spend.
- **Guardrail identity**: the before-tool-call hook on `writeMemoryBlock` checks a workflow-scoped lease token. The token is minted by `loadStep` and invalidated by `doneStep`; it is not user-supplied.
- **HOTL stream**: `MemoryEndpoint`'s SSE endpoint subscribes to `MemoryView` updates. Every entity event projection triggers a view row update which fans into the open SSE connections.
