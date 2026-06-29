# PLAN — meeting-facilitator

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

  Poller[MeetingSessionPoller]:::ta
  Queue[MeetingSessionQueue]:::ese
  Sanitizer[TranscriptSanitizer]:::cons
  Notes[NotesSummaryAgent]:::agent
  Chat[ChatSummaryAgent]:::agent
  WF[MeetingFacilitatorWorkflow]:::wf
  Entity[MeetingSessionEntity]:::ese
  View[MeetingSessionView]:::view
  EvalRunner[EvalRunner]:::ta
  API[MeetingEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Sanitizer
  Sanitizer -->|emit SegmentSanitized| Entity
  Entity -.->|on sanitized| WF
  WF -->|call| Notes
  WF -->|call| Chat
  WF -->|guardrail check| WF
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|query/SSE| View
  EvalRunner -.->|every 30m| Entity
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant P as MeetingSessionPoller
  participant Q as MeetingSessionQueue
  participant S as TranscriptSanitizer
  participant E as MeetingSessionEntity
  participant W as MeetingFacilitatorWorkflow
  participant N as NotesSummaryAgent
  participant C as ChatSummaryAgent
  participant U as User (UI)
  participant API as MeetingEndpoint

  P->>Q: emit SegmentReceived
  Q->>S: SegmentReceived
  S->>E: emit SegmentSanitized
  E->>W: start({sessionId, sanitized})
  W->>N: summarize(sanitized)
  N-->>W: MeetingNotes
  W->>C: recap(sanitized)
  C-->>W: ChatRecap
  W->>W: guardrail check → passed
  W->>E: emit GuardrailPassed + SessionPublished
  Note over E,U: Summary visible in UI
  U->>API: GET /api/sessions/{id}
  API-->>U: MeetingSession (status PUBLISHED)
```

## State machine — `MeetingSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> ACTIVE
  ACTIVE --> SANITIZED: SegmentSanitized
  SANITIZED --> SUMMARIZED: NotesSummarized + ChatSummarized
  SUMMARIZED --> PUBLISHED: GuardrailPassed + SessionPublished
  SUMMARIZED --> SUPPRESSED: GuardrailFailed + SessionSuppressed
  PUBLISHED --> [*]
  SUPPRESSED --> [*]
```

## Entity model

```mermaid
erDiagram
  MeetingSessionEntity ||--o{ SegmentReceived : emits
  MeetingSessionEntity ||--o{ SegmentSanitized : emits
  MeetingSessionEntity ||--o{ NotesSummarized : emits
  MeetingSessionEntity ||--o{ ChatSummarized : emits
  MeetingSessionEntity ||--o{ GuardrailPassed : emits
  MeetingSessionEntity ||--o{ GuardrailFailed : emits
  MeetingSessionEntity ||--o{ SessionPublished : emits
  MeetingSessionEntity ||--o{ SessionSuppressed : emits
  MeetingSessionEntity ||--o{ EvalScored : emits
  MeetingSessionView }o--|| MeetingSessionEntity : projects
  MeetingSessionQueue ||--o{ SegmentReceived : emits
  TranscriptSanitizer }o--|| MeetingSessionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MeetingSessionPoller` | `application/MeetingSessionPoller.java` |
| `MeetingSessionQueue` | `application/MeetingSessionQueue.java` |
| `TranscriptSanitizer` | `application/TranscriptSanitizer.java` |
| `NotesSummaryAgent` | `application/NotesSummaryAgent.java` |
| `ChatSummaryAgent` | `application/ChatSummaryAgent.java` |
| `MeetingFacilitatorWorkflow` | `application/MeetingFacilitatorWorkflow.java` |
| `MeetingSessionEntity` | `application/MeetingSessionEntity.java` (state in `domain/MeetingSession.java`, events in `domain/MeetingSessionEvent.java`) |
| `MeetingSessionView` | `application/MeetingSessionView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `MeetingEndpoint` | `api/MeetingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: notes agent 30 s, chat agent 20 s. On timeout, emit GuardrailFailed + SessionSuppressed.
- **Guardrail gate**: `MeetingFacilitatorWorkflow` evaluates both agent outputs synchronously in `guardrailStep`. Both must pass for `SessionPublished` to be emitted.
- **Idempotency**: every workflow uses `sessionId` as the workflow id so duplicate sanitize events fold into one workflow instance.
- **Eval sampling**: per tick, EvalRunner picks up to 5 PUBLISHED sessions with no `evalScore`, oldest-first.
