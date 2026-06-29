# PLAN — open-loop-chaser

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#0e1e2a', 'primaryTextColor': '#7EC8E3', 'primaryBorderColor': '#7EC8E3', 'lineColor': '#aab3bd', 'background': '#0d1117', 'nodeBorder': '#333'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Poller[ActionItemPoller]:::ta
  Queue[SourceEventQueue]:::ese
  Sanitizer[PiiSanitizer]:::cons
  Extractor[ActionItemExtractorAgent]:::agent
  Composer[NudgeComposerAgent]:::agent
  ExWF[ExtractionWorkflow]:::wf
  NudgeWF[NudgeWorkflow]:::wf
  Entity[ActionItemEntity]:::ese
  View[ActionItemView]:::view
  Scanner[StaleItemScanner]:::ta
  API[ActionItemEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Sanitizer
  Sanitizer -->|emit ActionItemSanitized| Entity
  Entity -.->|on sanitized| ExWF
  ExWF -->|call| Extractor
  ExWF -->|create ActionItemDetected| Entity
  Scanner -.->|every 10m| Entity
  Scanner -->|start| NudgeWF
  NudgeWF -->|call| Composer
  NudgeWF -->|guardrail check + dispatch| Entity
  Entity -.->|projects| View
  API -->|confirm-owner / close / snooze| Entity
  API -->|query/SSE| View
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as ActionItemPoller
  participant Q as SourceEventQueue
  participant S as PiiSanitizer
  participant E as ActionItemEntity
  participant EW as ExtractionWorkflow
  participant X as ActionItemExtractorAgent
  participant SC as StaleItemScanner
  participant NW as NudgeWorkflow
  participant C as NudgeComposerAgent
  participant U as Operator (UI)
  participant API as ActionItemEndpoint

  P->>Q: emit SourceEventReceived
  Q->>S: SourceEventReceived
  S->>E: emit ActionItemSanitized
  E->>EW: start({eventId, sanitized})
  EW->>X: extract(sanitized)
  X-->>EW: ExtractionResult{items}
  EW->>E: emit ActionItemDetected (per item)
  Note over E,U: Item in PENDING state
  U->>API: POST /api/items/{id}/confirm-owner
  API->>E: emit OwnerConfirmed
  SC->>E: markStale (threshold exceeded)
  SC->>NW: start({itemId})
  NW->>C: compose(item, ownerConfirmation)
  C-->>NW: ComposedNudge
  NW->>NW: guardStep (ownerConfirmation present? ✓)
  NW->>E: emit NudgeComposed + NudgeDispatched
  E-->>U: SSE item-update (status NUDGED)
```

## State machine — `ActionItemEntity`

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'background': '#0d1117', 'primaryColor': '#1f1900', 'primaryTextColor': '#F5C518', 'primaryBorderColor': '#F5C518', 'lineColor': '#aab3bd', 'transitionLabelColor': '#cccccc', 'stateLabelColor': '#F5C518'}}}%%
stateDiagram-v2
  [*] --> DETECTED
  DETECTED --> SANITIZED : ActionItemSanitized
  SANITIZED --> PENDING : ActionItemDetected
  PENDING --> STALE : ItemMarkedStale
  STALE --> NUDGED : NudgeDispatched
  NUDGED --> STALE : staleness threshold (repeat nudge)
  NUDGED --> SNOOZED : ItemSnoozed
  SNOOZED --> PENDING : SnoozeLapsed
  PENDING --> CLOSED : ItemClosed
  NUDGED --> CLOSED : ItemClosed
  STALE --> CLOSED : ItemClosed
  PENDING --> FAILED : ExtractionFailed
  STALE --> FAILED : guardrail:no-confirmed-owner
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ActionItemEntity ||--o{ ActionItemDetected : emits
  ActionItemEntity ||--o{ ActionItemSanitized : emits
  ActionItemEntity ||--o{ OwnerConfirmed : emits
  ActionItemEntity ||--o{ ItemMarkedStale : emits
  ActionItemEntity ||--o{ NudgeComposed : emits
  ActionItemEntity ||--o{ NudgeDispatched : emits
  ActionItemEntity ||--o{ ItemSnoozed : emits
  ActionItemEntity ||--o{ SnoozeLapsed : emits
  ActionItemEntity ||--o{ ItemClosed : emits
  ActionItemEntity ||--o{ ExtractionFailed : emits
  ActionItemView }o--|| ActionItemEntity : projects
  SourceEventQueue ||--o{ SourceEventReceived : emits
  PiiSanitizer }o--|| SourceEventQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ActionItemPoller` | `application/ActionItemPoller.java` |
| `SourceEventQueue` | `application/SourceEventQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ActionItemExtractorAgent` | `application/ActionItemExtractorAgent.java` |
| `NudgeComposerAgent` | `application/NudgeComposerAgent.java` |
| `ExtractionWorkflow` | `application/ExtractionWorkflow.java` |
| `NudgeWorkflow` | `application/NudgeWorkflow.java` |
| `ActionItemEntity` | `application/ActionItemEntity.java` (state in `domain/ActionItem.java`, events in `domain/ActionItemEvent.java`) |
| `ActionItemView` | `application/ActionItemView.java` |
| `StaleItemScanner` | `application/StaleItemScanner.java` |
| `ActionItemEndpoint` | `api/ActionItemEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: extractor 15 s, nudge composer 30 s. On timeout, mark item FAILED.
- **Guardrail gate**: `NudgeWorkflow` guardStep reads `ActionItemEntity.getItem` and requires `ownerConfirmation.isPresent()` before proceeding to dispatchStep. If absent, the workflow ends with `ExtractionFailed(reason="guardrail:no-confirmed-owner")`.
- **Idempotency**: every workflow uses the source `eventId` (extraction) or `itemId` (nudge) as its workflow id so duplicate sanitize events and repeated scanner ticks fold safely.
- **Snooze expiry**: `SnoozeLapsed` is emitted by `StaleItemScanner` when it encounters a SNOOZED item whose `snooze.snoozedAt + snooze.snoozeDuration` is in the past. The item returns to PENDING.
- **Staleness threshold**: configurable via `application.conf`; defaults to 30 minutes. `StaleItemScanner` uses the later of `lastNudge.composedAt` and `detectedAt` for the threshold comparison.
