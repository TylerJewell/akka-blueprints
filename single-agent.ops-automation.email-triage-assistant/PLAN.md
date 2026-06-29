# PLAN — email-triage-assistant

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[TriageEndpoint]:::ep
  EmailEnt[EmailEntity]:::ese
  ConfEnt[ConfirmationEntity]:::ese
  Sanitizer[EmailSanitizer]:::cons
  WF[TriageWorkflow]:::wf
  Agent[EmailTriageAgent]:::agent
  Guard[SendGuardrail]:::guard
  View[TriageView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| EmailEnt
  API -->|confirm| ConfEnt
  EmailEnt -.->|EmailSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| EmailEnt
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| EmailEnt
  WF -->|triageStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|check| ConfEnt
  Agent -->|TriageResult| WF
  WF -->|recordDraft| EmailEnt
  WF -->|confirmStep poll| ConfEnt
  WF -->|sendStep send_email| Agent
  WF -->|recordSent / recordRejected| EmailEnt
  EmailEnt -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J2 (approve path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as TriageEndpoint
  participant E as EmailEntity
  participant C as ConfirmationEntity
  participant S as EmailSanitizer
  participant W as TriageWorkflow
  participant A as EmailTriageAgent
  participant G as SendGuardrail

  U->>API: POST /api/emails
  API->>E: submit(submission)
  E-->>API: { emailId }
  E-.->>S: EmailSubmitted
  S->>S: strip PII
  S->>E: attachSanitized
  S->>W: start(emailId)
  W->>E: poll getEmail
  E-->>W: sanitized.isPresent()
  W->>E: markReviewing
  W->>A: runSingleTask(context + body.txt + attachments.txt)
  A-->>W: TriageResult
  W->>E: recordDraft(triage)
  E-.->>U: SSE event(DRAFT_READY)
  U->>API: POST /api/emails/{id}/confirm {APPROVED}
  API->>C: record(APPROVED)
  W->>C: poll getDecision
  C-->>W: APPROVED
  W->>A: invoke send_email tool
  A->>G: before-tool-call(send_email)
  G->>C: getDecision
  C-->>G: APPROVED
  G-->>A: pass
  A-->>W: tool result
  W->>E: recordSent
  E-.->>U: SSE event(SENT)
```

## State machine — `EmailEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: EmailSanitized
  SANITIZED --> REVIEWING: TriageStarted
  REVIEWING --> DRAFT_READY: DraftReady
  DRAFT_READY --> SENDING: ConfirmationReceived (APPROVED)
  DRAFT_READY --> REJECTED: ConfirmationReceived (REJECTED)
  SENDING --> SENT: EmailSent
  REVIEWING --> FAILED: EmailFailed (agent error)
  SUBMITTED --> FAILED: EmailFailed (sanitizer error)
  SENT --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EmailEntity ||--o{ EmailSubmitted : emits
  EmailEntity ||--o{ EmailSanitized : emits
  EmailEntity ||--o{ TriageStarted : emits
  EmailEntity ||--o{ DraftReady : emits
  EmailEntity ||--o{ ConfirmationReceived : emits
  EmailEntity ||--o{ EmailSent : emits
  EmailEntity ||--o{ EmailRejected : emits
  EmailEntity ||--o{ EmailFailed : emits
  ConfirmationEntity ||--o{ DecisionRecorded : emits
  TriageView }o--|| EmailEntity : projects
  EmailSanitizer }o--|| EmailEntity : subscribes
  TriageWorkflow }o--|| EmailEntity : reads-and-writes
  TriageWorkflow }o--|| ConfirmationEntity : reads
  EmailTriageAgent ||--o{ TriageResult : returns
  SendGuardrail }o--|| ConfirmationEntity : checks
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TriageEndpoint` | `api/TriageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `EmailEntity` | `application/EmailEntity.java` (state in `domain/Email.java`, events in `domain/EmailEvent.java`) |
| `ConfirmationEntity` | `application/ConfirmationEntity.java` (state in `domain/ConfirmationState.java`, event in `domain/ConfirmationEvent.java`) |
| `EmailSanitizer` | `application/EmailSanitizer.java` |
| `TriageWorkflow` | `application/TriageWorkflow.java` |
| `EmailTriageAgent` | `application/EmailTriageAgent.java` (tasks in `application/EmailTasks.java`) |
| `SendGuardrail` | `application/SendGuardrail.java` |
| `TriageView` | `application/TriageView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `triageStep` 90 s, `confirmStep` 300 s, `sendStep` 15 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TriageWorkflow::error)`. The 90 s on `triageStep` accommodates LLM latency on multi-attachment emails (Lesson 4).
- **confirmStep timeout**: 300 s gives the user 5 minutes to read the draft and decide. If the timer expires with no decision, the workflow fails over to `error` and the entity transitions to `FAILED`.
- **Idempotency**: every workflow uses `"triage-" + emailId` as the workflow id. `EmailSanitizer` is allowed to redeliver `EmailSubmitted` events; `EmailEntity.attachSanitized` is event-version-guarded — a second sanitize attempt on an already-sanitized email is a no-op.
- **One agent per email**: the AutonomousAgent instance id is `"triage-" + emailId`. Each email gets its own conversation context. `maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Guardrail-driven block**: when `SendGuardrail` blocks a `send_email` invocation, the block is returned as a structured error to the agent loop; the loop counts toward `maxIterationsPerTask`. The workflow's `confirmStep` is the correct synchronisation point; the guardrail is a defence-in-depth check, not the primary gate.
- **Two entities, one workflow**: `EmailEntity` owns the email lifecycle; `ConfirmationEntity` is a separate entity so the user's approve/reject action does not contend with the workflow's event-write path on `EmailEntity`.
- **No saga / no compensation**: the send is simulated in-process. A deployer wiring a real SMTP or API call would add compensation logic here; the blueprint defers that complexity.
