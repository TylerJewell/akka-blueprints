# PLAN — morning-email-debrief

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

  Poller[MailboxPoller]:::ta
  Queue[EmailQueue]:::ese
  Sanitizer[EmailPiiSanitizer]:::cons
  Summarizer[EmailSummarizerAgent]:::agent
  Assembler[DebriefAssemblerAgent]:::agent
  WF[DebriefWorkflow]:::wf
  EmailEnt[EmailEntity]:::ese
  DebriefEnt[DebriefEntity]:::ese
  EmailV[EmailView]:::view
  DebriefV[DebriefView]:::view
  EvalRunner[EvalRunner]:::ta
  API[DebriefEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 60s| Queue
  Queue -.->|subscribes| Sanitizer
  Sanitizer -->|emit EmailSanitized| EmailEnt
  Sanitizer -->|create/join run| DebriefEnt
  EmailEnt -.->|on sanitized| WF
  WF -->|call per email| Summarizer
  WF -->|call once| Assembler
  WF -->|emit events| EmailEnt
  WF -->|emit events| DebriefEnt
  EmailEnt -.->|projects| EmailV
  DebriefEnt -.->|projects| DebriefV
  API -->|query/SSE| EmailV
  API -->|query/SSE| DebriefV
  EvalRunner -.->|every 60m| DebriefEnt
```

## Interaction sequence — J1 (batch arrives and debrief assembles)

```mermaid
sequenceDiagram
  autonumber
  participant P as MailboxPoller
  participant Q as EmailQueue
  participant S as EmailPiiSanitizer
  participant EE as EmailEntity
  participant DE as DebriefEntity
  participant W as DebriefWorkflow
  participant Sum as EmailSummarizerAgent
  participant Asm as DebriefAssemblerAgent
  participant U as User (UI)
  participant API as DebriefEndpoint

  P->>Q: emit EmailReceived (batch of N)
  loop for each email
    Q->>S: EmailReceived
    S->>EE: emit EmailSanitized
    S->>DE: create or increment run
  end
  EE->>W: start({runId})
  W->>W: awaitAllSanitizedStep (poll every 10s)
  loop for each email in run
    W->>Sum: summarize(sanitizedPayload)
    Sum-->>W: DebriefEntry
    W->>EE: emit EmailSummarised
  end
  W->>Asm: assemble(entries, runId, totalEmailCount)
  Asm-->>W: MorningDebrief
  W->>DE: emit DebriefReady
  Note over DE,U: Debrief READY
  U->>API: GET /api/debrief/{runId}
  API-->>U: DebriefRun with MorningDebrief
```

## State machine — `DebriefEntity`

```mermaid
stateDiagram-v2
  %%{init: {'theme': 'dark', 'themeVariables': {'transitionLabelColor': '#cccccc'}}}%%
  [*] --> CREATED: DebriefCreated
  CREATED --> ASSEMBLING: DebriefAssembling
  ASSEMBLING --> READY: DebriefReady
  ASSEMBLING --> FAILED: DebriefFailed (assembler error)
  READY --> READY: DebriefEvalScored
  READY --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EmailEntity ||--o{ EmailReceived : emits
  EmailEntity ||--o{ EmailSanitized : emits
  EmailEntity ||--o{ EmailSummarised : emits
  EmailView }o--|| EmailEntity : projects
  DebriefEntity ||--o{ DebriefCreated : emits
  DebriefEntity ||--o{ DebriefAssembling : emits
  DebriefEntity ||--o{ DebriefReady : emits
  DebriefEntity ||--o{ DebriefFailed : emits
  DebriefEntity ||--o{ DebriefEvalScored : emits
  DebriefView }o--|| DebriefEntity : projects
  EmailQueue ||--o{ EmailReceived : emits
  EmailPiiSanitizer }o--|| EmailQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MailboxPoller` | `application/MailboxPoller.java` |
| `EmailQueue` | `application/EmailQueue.java` |
| `EmailPiiSanitizer` | `application/EmailPiiSanitizer.java` |
| `EmailSummarizerAgent` | `application/EmailSummarizerAgent.java` |
| `DebriefAssemblerAgent` | `application/DebriefAssemblerAgent.java` |
| `DebriefWorkflow` | `application/DebriefWorkflow.java` |
| `EmailEntity` | `application/EmailEntity.java` (state in `domain/EmailRecord.java`, events in `domain/EmailEvent.java`) |
| `DebriefEntity` | `application/DebriefEntity.java` (state in `domain/DebriefRun.java`, events in `domain/DebriefEvent.java`) |
| `EmailView` | `application/EmailView.java` |
| `DebriefView` | `application/DebriefView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `DebriefEndpoint` | `api/DebriefEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: summarize step 15 s per email, assemble step 60 s. On assemble timeout, emit DebriefFailed.
- **Await gate**: `DebriefWorkflow` polls `EmailView` every 10 s in `awaitAllSanitizedStep` until all emails for the run are SUMMARISED (or timeout after 10 minutes).
- **Idempotency**: every workflow uses `runId` as the workflow id; the `EmailPiiSanitizer` starts or joins the workflow, never creates a duplicate.
- **Eval sampling**: per tick, `EvalRunner` picks up to 3 READY debriefs with no `evalScore`, oldest-first.
