# PLAN — team-poems-multi-agent

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

  Director[PoetryDirector]:::agent
  Poet[PoetAgent]:::agent

  DirWF[DirectionWorkflow]:::wf
  PoetWF[PoetWorkflow]:::wf

  PoemE[PoemEntity]:::ese
  StanzaE[StanzaEntity]:::ese
  Mailbox[CoordinationMailbox]:::ese
  Queue[PromptQueue]:::ese
  Control[EditorialControl]:::kve
  Board[VerseBoardView]:::view
  Consumer[PromptSubmissionConsumer]:::cons
  Sim[PromptSimulator]:::ta
  Stuck[StuckStanzaMonitor]:::ta
  API[PoetryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit prompt| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|create + start| PoemE
  Consumer -->|start workflow| DirWF
  DirWF -->|call| Director
  DirWF -->|create one per stanza| StanzaE
  StanzaE -.->|projects| Board
  PoetWF -->|poll board| Board
  PoetWF -->|atomic claim| StanzaE
  PoetWF -->|call| Poet
  PoetWF -->|post message| Mailbox
  PoetWF -->|read flag| Control
  Stuck -.->|every 30s| StanzaE
  API -->|pause / resume| Control
  API -->|reply| Mailbox
  API -->|query / SSE| Board
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `PoetAgent` is one agent class run as several instances (`poet-1`, `poet-2`, `poet-3`); each instance is driven by its own `PoetWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PoetryEndpoint
  participant Q as PromptQueue
  participant C as PromptSubmissionConsumer
  participant DW as DirectionWorkflow
  participant D as PoetryDirector
  participant S as StanzaEntity
  participant PW as PoetWorkflow
  participant P as PoetAgent
  participant V as VerseBoardView

  U->>API: POST /api/poems {title, inspiration}
  API->>Q: submitPrompt(prompt)
  API-->>U: 202 {poemId}
  Q->>C: PromptSubmitted
  C->>DW: start({poemId})
  DW->>D: direct(prompt)
  D-->>DW: VersePlan{stanzas}
  DW->>S: createStanza (one per spec, status OPEN)
  S-->>V: board rows
  Note over PW: poet loops already polling
  PW->>V: getAllStanzas
  V-->>PW: OPEN stanza with deps DONE
  PW->>S: claim(poet-1)
  S-->>PW: StanzaClaimed (won the race)
  PW->>P: draft(stanza)
  Note over P: every tool call passes the content guardrail
  P-->>PW: DraftVerse{lines, approach}
  PW->>PW: qualityCheck(draft)
  PW->>S: passQuality -> DONE
  S-->>V: row DONE
  V-->>U: SSE update
```

## State machine — `StanzaEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CLAIMED: claim (atomic, single winner)
  CLAIMED --> DRAFTING: startDrafting
  CLAIMED --> OPEN: release (idle > 2 min)
  DRAFTING --> IN_REVIEW: submitVerse
  DRAFTING --> BLOCKED: coordination request raised
  DRAFTING --> BLOCKED: content guardrail refusal
  DRAFTING --> OPEN: release (idle > 2 min)
  IN_REVIEW --> DONE: qualityPassed
  IN_REVIEW --> DRAFTING: qualityFailed (retry)
  IN_REVIEW --> BLOCKED: quality failed (retries exhausted)
  BLOCKED --> OPEN: coordination reply unblocks / editor reopen
  DONE --> [*]
```

## Entity model

```mermaid
erDiagram
  StanzaEntity ||--o{ StanzaCreated : emits
  StanzaEntity ||--o{ StanzaClaimed : emits
  StanzaEntity ||--o{ DraftingStarted : emits
  StanzaEntity ||--o{ VerseSubmitted : emits
  StanzaEntity ||--o{ QualityPassed : emits
  StanzaEntity ||--o{ QualityFailed : emits
  StanzaEntity ||--o{ StanzaBlocked : emits
  StanzaEntity ||--o{ StanzaReleased : emits
  StanzaEntity ||--o{ StanzaCompleted : emits
  VerseBoardView }o--|| StanzaEntity : projects
  PoemEntity ||--o{ PoemCreated : emits
  PoemEntity ||--o{ PoemAssigned : emits
  PoemEntity ||--o{ PoemPublished : emits
  PoemEntity ||--o{ StanzaEntity : "owns N stanzas"
  PromptQueue ||--o{ PromptSubmitted : emits
  PromptSubmissionConsumer }o--|| PromptQueue : subscribes
  CoordinationMailbox ||--o{ CoordinationPosted : emits
  CoordinationMailbox ||--o{ CoordinationReplied : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PoetryDirector` | `application/PoetryDirector.java` |
| `PoetAgent` | `application/PoetAgent.java` |
| `PoemTasks` | `application/PoemTasks.java` |
| `QualityChecker` | `application/QualityChecker.java` |
| `DirectionWorkflow` | `application/DirectionWorkflow.java` |
| `PoetWorkflow` | `application/PoetWorkflow.java` |
| `StanzaEntity` | `application/StanzaEntity.java` (state in `domain/Stanza.java`, events in `domain/StanzaEvent.java`) |
| `PoemEntity` | `application/PoemEntity.java` (state in `domain/Poem.java`, events in `domain/PoemEvent.java`) |
| `CoordinationMailbox` | `application/CoordinationMailbox.java` (state + events in `domain/`) |
| `PromptQueue` | `application/PromptQueue.java` |
| `EditorialControl` | `application/EditorialControl.java` |
| `VerseBoardView` | `application/VerseBoardView.java` |
| `PromptSubmissionConsumer` | `application/PromptSubmissionConsumer.java` |
| `PromptSimulator` | `application/PromptSimulator.java` |
| `StuckStanzaMonitor` | `application/StuckStanzaMonitor.java` |
| `PoetryEndpoint` | `api/PoetryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic claim is the whole pattern.** `StanzaEntity` is a single-writer; `claim(poetId)` emits `StanzaClaimed` only when the current status is `OPEN`. Two poet workflows that read the same `OPEN` stanza from the view and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Stanza` and returns to polling. No lock, no external queue.
- **Workflow step timeouts:** `DirectionWorkflow.directStep` and `PoetWorkflow.draftStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4).
- **Idle polling:** `PoetWorkflow.pollStep` self-schedules a 5 s resume timer when the team is paused or no eligible `OPEN` stanza exists.
- **Dependency gate:** a stanza is eligible only when every title in its `dependsOn` resolves to a `DONE` stanza on the board. The poll filters client-side (Lesson 2).
- **Release for liveness:** `StuckStanzaMonitor` returns a stanza claimed-but-idle for more than two minutes to `OPEN`.
- **Quality gate:** `QualityChecker` is a deterministic pure function (no LLM call); the same draft always yields the same `QualityReport`.
- **Pause:** `EditorialControl` is read at the top of `pollStep` and inside the content guardrail, so a pause both stops new claims and can refuse any in-flight tool call.
- **Idempotency:** deterministic `stanzaId = poemId + "-s" + index` makes `createStanza` idempotent if `DirectionWorkflow.createStanzasStep` is retried.
