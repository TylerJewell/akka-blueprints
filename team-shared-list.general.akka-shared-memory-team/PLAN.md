# PLAN — shared-memory-multi-agent

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#1f1900,stroke:#C9A227,color:#C9A227;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Writer[WriterAgent]:::agent
  Consolidator[ConsolidatorAgent]:::agent

  WriteWF[WriteWorkflow]:::wf
  ConsolidateWF[ConsolidationWorkflow]:::wf

  BlockE[MemoryBlockEntity]:::ese
  FragmentE[FragmentEntity]:::ese
  EventLog[MemoryEventLog]:::ese
  Registry[AgentRegistry]:::kve
  BlockView[MemoryBlockView]:::view
  FragCons[FragmentConsumer]:::cons
  Sim[BlockSimulator]:::ta
  Sched[ConsolidationScheduler]:::ta
  AgeMonitor[FragmentAgeMonitor]:::ta
  API[MemoryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|create block| EventLog
  Sim -.->|every 90s| EventLog
  EventLog -.->|subscribes| FragCons
  FragCons -->|start workflow| ConsolidateWF
  ConsolidateWF -->|call| Consolidator
  ConsolidateWF -->|consolidate snapshot| BlockE
  ConsolidateWF -->|mark merged| FragmentE
  BlockE -.->|projects| BlockView
  FragmentE -.->|projects| BlockView
  WriteWF -->|read snapshot| BlockView
  WriteWF -->|call| Writer
  WriteWF -->|write fragment| FragmentE
  WriteWF -->|increment pending| BlockE
  Sched -.->|every 120s| ConsolidateWF
  AgeMonitor -.->|every 60s| FragmentE
  API -->|consolidate on demand| ConsolidateWF
  API -->|query / SSE| BlockView
  API -->|register/deregister| Registry
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `WriterAgent` is one agent class run as several instances (`writer-1`, `writer-2`, `writer-3`); each instance is driven by its own `WriteWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as MemoryEndpoint
  participant EL as MemoryEventLog
  participant BE as MemoryBlockEntity
  participant WW as WriteWorkflow
  participant WA as WriterAgent
  participant FE as FragmentEntity
  participant CW as ConsolidationWorkflow
  participant CA as ConsolidatorAgent
  participant V as MemoryBlockView

  U->>API: POST /api/blocks {blockName, initialContent}
  API->>EL: logBlockCreated(seed)
  API->>BE: seedBlock(blockId)
  API-->>U: 202 {blockId}
  Note over WW: writer loops already running
  WW->>V: getBlock(blockId)
  V-->>WW: snapshot
  WW->>WA: contribute(snapshot)
  Note over WA: every write passes the before-tool-call guardrail (G1) and sanitizer (S1)
  WA-->>WW: MemoryFragment{content}
  WW->>FE: submit(fragment)
  FE-->>V: fragment row PENDING
  WW->>BE: incrementPending
  BE-->>V: block row updated
  Note over CW: FragmentConsumer triggers consolidation at threshold
  CW->>V: getPendingFragments(blockId)
  CW->>BE: startConsolidation
  CW->>CA: consolidate(fragments)
  CA-->>CW: ConsolidatedSnapshot
  CW->>BE: consolidate(snapshot) -> CONSOLIDATED
  CW->>FE: merge each fragmentId
  BE-->>V: block CONSOLIDATED
  V-->>U: SSE update
```

## State machine — `MemoryBlockEntity`

```mermaid
stateDiagram-v2
  [*] --> SEEDED
  SEEDED --> ACTIVE: activate (first fragment written)
  ACTIVE --> CONSOLIDATING: startConsolidation
  CONSOLIDATING --> CONSOLIDATED: consolidate (snapshot committed)
  CONSOLIDATED --> ACTIVE: new fragment written (re-enters write cycle)
  ACTIVE --> ARCHIVED: archive (operator action)
  CONSOLIDATED --> ARCHIVED: archive (operator action)
  ARCHIVED --> [*]
```

## Entity model

```mermaid
erDiagram
  MemoryBlockEntity ||--o{ BlockSeeded : emits
  MemoryBlockEntity ||--o{ BlockActivated : emits
  MemoryBlockEntity ||--o{ ConsolidationStarted : emits
  MemoryBlockEntity ||--o{ BlockConsolidated : emits
  MemoryBlockEntity ||--o{ BlockArchived : emits
  MemoryBlockView }o--|| MemoryBlockEntity : projects
  FragmentEntity ||--o{ FragmentSubmitted : emits
  FragmentEntity ||--o{ FragmentSanitized : emits
  FragmentEntity ||--o{ FragmentMerged : emits
  FragmentEntity ||--o{ FragmentRejected : emits
  FragmentEntity ||--o{ FragmentRequeued : emits
  MemoryBlockView }o--|| FragmentEntity : projects
  MemoryEventLog ||--o{ BlockCreated : emits
  MemoryEventLog ||--o{ ConsolidationTriggered : emits
  FragmentConsumer }o--|| FragmentEntity : subscribes
  MemoryBlockEntity ||--o{ FragmentEntity : "owns N fragments"
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WriterAgent` | `application/WriterAgent.java` |
| `ConsolidatorAgent` | `application/ConsolidatorAgent.java` |
| `MemoryTasks` | `application/MemoryTasks.java` |
| `WriteWorkflow` | `application/WriteWorkflow.java` |
| `ConsolidationWorkflow` | `application/ConsolidationWorkflow.java` |
| `MemoryBlockEntity` | `application/MemoryBlockEntity.java` (state in `domain/MemoryBlock.java`, events in `domain/MemoryBlockEvent.java`) |
| `FragmentEntity` | `application/FragmentEntity.java` (state in `domain/Fragment.java`, events in `domain/FragmentEvent.java`) |
| `MemoryEventLog` | `application/MemoryEventLog.java` |
| `AgentRegistry` | `application/AgentRegistry.java` |
| `MemoryBlockView` | `application/MemoryBlockView.java` |
| `FragmentConsumer` | `application/FragmentConsumer.java` |
| `BlockSimulator` | `application/BlockSimulator.java` |
| `ConsolidationScheduler` | `application/ConsolidationScheduler.java` |
| `FragmentAgeMonitor` | `application/FragmentAgeMonitor.java` |
| `MemoryEndpoint` | `api/MemoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 3 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 3 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic write is the whole pattern.** `MemoryBlockEntity` is a single-writer; `consolidate(snapshot)` emits `BlockConsolidated` only when the current status is `CONSOLIDATING`. Two consolidation workflows that race for the same block are serialised by the entity — the first wins, the second receives a rejection and is discarded.
- **Workflow step timeouts:** `WriteWorkflow.contributeStep` and `ConsolidationWorkflow.consolidateStep` call agents, so each sets an explicit `stepTimeout` (90 s and 120 s respectively — Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **Write cycle throttle:** `WriteWorkflow.scheduleStep` self-schedules a 30 s resume timer between cycles, so writers are staggered, not racing every tick.
- **Consolidation guard:** `FragmentConsumer` only starts a `ConsolidationWorkflow` if one is not already running for the block id (the workflow id matches the block id, so a duplicate start is a no-op).
- **Re-queue for liveness:** `FragmentAgeMonitor` re-queues fragments `PENDING` for more than three minutes to `REQUEUED` status so a stalled consolidation does not leave a block permanently dirty.
- **Guardrail placement:** the G1 before-tool-call guardrail fires inside `WriteWorkflow.contributeStep` before the write is dispatched; the S1 sanitizer runs in `sanitizeStep` after the agent returns but before `FragmentEntity.submit`; both are in-process, not network calls.
- **View query scope:** neither `getAllBlocks` nor `getFragmentsForBlock` has a WHERE filter on an enum column — callers filter status client-side (Lesson 2).
- **Idempotency:** `fragmentId = blockId + "-f-" + writerId + "-" + cycle` makes `submit` idempotent if `WriteWorkflow.writeFragmentStep` is retried.
