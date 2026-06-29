# PLAN — screenplay-writer

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

  INTAKE[ThreadIntakeEndpoint]:::ep
  APP[AppEndpoint]:::ep
  QUEUE[InboundThreadQueue]:::ese
  SIM[ThreadSimulator]:::ta
  CONS[ThreadIntakeConsumer]:::cons
  WF[ScreenplayWorkflow]:::wf
  AN[ThreadAnalyzerAgent]:::agent
  DW[DialogueWriterAgent]:::agent
  FM[ScreenplayFormatterAgent]:::agent
  ENT[ScreenplayEntity]:::ese
  VIEW[ScreenplayView]:::view

  SIM -. drips .-> QUEUE
  INTAKE --> QUEUE
  QUEUE -. events .-> CONS
  CONS --> WF
  INTAKE --> WF
  WF --> AN
  WF --> DW
  WF --> FM
  WF --> ENT
  ENT -. events .-> VIEW
  VIEW --> INTAKE
  APP --> INTAKE
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User / UI
  participant E as ThreadIntakeEndpoint
  participant W as ScreenplayWorkflow
  participant S as PiiSanitizer
  participant A as ThreadAnalyzerAgent
  participant D as DialogueWriterAgent
  participant F as ScreenplayFormatterAgent
  participant N as ScreenplayEntity
  U->>E: POST /api/threads { rawText }
  E->>W: start(jobId, rawText)
  W->>S: sanitize(rawText)
  S-->>W: sanitizedText, redactionCount
  W->>N: recordSanitized
  Note over W,N: status = SANITIZED
  W->>A: analyze(sanitizedText)
  A-->>W: ThreadAnalysis
  W->>N: recordAnalysis
  W->>D: write(analysis)
  D-->>W: DialogueDraft
  W->>N: recordDialogue
  W->>F: format(draft)
  Note over F: before-agent-response guardrail scans for PII
  F-->>W: Screenplay { containsPii }
  alt containsPii
    W->>N: recordBlocked
    Note over W,N: status = BLOCKED
  else clean
    W->>N: recordCompleted
    Note over W,N: status = COMPLETED
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ThreadSanitized
  SANITIZED --> ANALYZED: ThreadAnalyzed
  ANALYZED --> DIALOGUE_WRITTEN: DialogueWritten
  DIALOGUE_WRITTEN --> COMPLETED: ScreenplayCompleted
  DIALOGUE_WRITTEN --> BLOCKED: ScreenplayBlocked
  RECEIVED --> FAILED: JobFailed
  SANITIZED --> FAILED: JobFailed
  ANALYZED --> FAILED: JobFailed
  DIALOGUE_WRITTEN --> FAILED: JobFailed
  COMPLETED --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  INBOUND_THREAD_QUEUE ||--o{ THREAD_QUEUED : emits
  SCREENPLAY_ENTITY ||--o{ THREAD_RECEIVED : emits
  SCREENPLAY_ENTITY ||--o{ THREAD_SANITIZED : emits
  SCREENPLAY_ENTITY ||--o{ THREAD_ANALYZED : emits
  SCREENPLAY_ENTITY ||--o{ DIALOGUE_WRITTEN : emits
  SCREENPLAY_ENTITY ||--o{ SCREENPLAY_COMPLETED : emits
  SCREENPLAY_ENTITY ||--o{ SCREENPLAY_BLOCKED : emits
  SCREENPLAY_ENTITY ||--|| SCREENPLAY_VIEW : projects
  SCREENPLAY_VIEW {
    string id
    string sourceLabel
    string status
    int redactionCount
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| ThreadIntakeEndpoint | `api/ThreadIntakeEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| ScreenplayWorkflow | `application/ScreenplayWorkflow.java` |
| ThreadAnalyzerAgent | `application/ThreadAnalyzerAgent.java` |
| DialogueWriterAgent | `application/DialogueWriterAgent.java` |
| ScreenplayFormatterAgent | `application/ScreenplayFormatterAgent.java` |
| PiiSanitizer | `application/PiiSanitizer.java` |
| ScreenplayEntity | `domain/ScreenplayEntity.java` |
| ScreenplayView | `application/ScreenplayView.java` |
| InboundThreadQueue | `domain/InboundThreadQueue.java` |
| ThreadIntakeConsumer | `application/ThreadIntakeConsumer.java` |
| ThreadSimulator | `application/ThreadSimulator.java` |

## Concurrency notes

- Workflow step timeouts: 60s on `analyzeStep`, `dialogueStep`, `formatStep` (LLM latency, Lesson 4). `sanitizeStep` is deterministic and keeps the default timeout. `defaultStepRecovery(maxRetries(2).failoverTo(error))`.
- Idempotency: the workflow id is the `jobId`; the consumer derives one UUID per queued thread so re-delivery does not start duplicate workflows.
- No saga: stages are forward-only. A failure at any stage emits `JobFailed`; there is no external side effect to compensate (the publish target is in-process and the screenplay is only persisted on the entity).
- The final guardrail is a fast-fail, not a retry: `containsPii == true` ends the workflow in `BLOCKED` rather than re-prompting the agent.
