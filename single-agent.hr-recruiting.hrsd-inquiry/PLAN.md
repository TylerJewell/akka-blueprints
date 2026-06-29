# PLAN — hrsd-inquiry

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

  API[InquiryEndpoint]:::ep
  Entity[InquiryEntity]:::ese
  Screener[SpecialCategoryScreener]:::cons
  WF[InquiryWorkflow]:::wf
  Agent[HrInquiryAgent]:::agent
  Guard[ResponseGuardrail]:::guard
  Submitter[HrRequestSubmitter]:::guard
  View[InquiryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|InquirySubmitted| Screener
  Screener -->|attachScreened| Entity
  Screener -->|start workflow| WF
  WF -->|awaitScreenedStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|InquiryResponse| WF
  WF -->|recordAnswer| Entity
  WF -->|submitRequestStep| Submitter
  Submitter -->|ref| WF
  WF -->|recordServiceRequest| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Employee (UI)
  participant API as InquiryEndpoint
  participant E as InquiryEntity
  participant Sc as SpecialCategoryScreener
  participant W as InquiryWorkflow
  participant A as HrInquiryAgent
  participant G as ResponseGuardrail
  participant Sub as HrRequestSubmitter

  U->>API: POST /api/inquiries
  API->>E: submit(request)
  E-->>API: { inquiryId }
  E-.->>Sc: InquirySubmitted
  Sc->>Sc: redact special-category data
  Sc->>E: attachScreened
  Sc->>W: start(inquiryId)
  W->>E: poll getInquiry
  E-->>W: screened.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(message + policy attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: InquiryResponse
  W->>E: recordAnswer(response)
  alt submitRequestIfApplicable && serviceRequest != null
    W->>Sub: submit(serviceRequest, inquiryId)
    Sub-->>W: confirmationRef
    W->>E: recordServiceRequest(ref)
  end
  E-.->>U: SSE event(REQUEST_SUBMITTED or ANSWERED)
```

## State machine — `InquiryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SCREENED: InquiryScreened
  SCREENED --> ANSWERING: AnswerStarted
  ANSWERING --> ANSWERED: InquiryAnswered
  ANSWERED --> REQUEST_SUBMITTED: ServiceRequestSubmitted
  ANSWERING --> FAILED: InquiryFailed (agent error)
  SUBMITTED --> FAILED: InquiryFailed (screener error)
  ANSWERED --> [*]
  REQUEST_SUBMITTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  InquiryEntity ||--o{ InquirySubmitted : emits
  InquiryEntity ||--o{ InquiryScreened : emits
  InquiryEntity ||--o{ AnswerStarted : emits
  InquiryEntity ||--o{ InquiryAnswered : emits
  InquiryEntity ||--o{ ServiceRequestSubmitted : emits
  InquiryEntity ||--o{ InquiryFailed : emits
  InquiryView }o--|| InquiryEntity : projects
  SpecialCategoryScreener }o--|| InquiryEntity : subscribes
  InquiryWorkflow }o--|| InquiryEntity : reads-and-writes
  HrInquiryAgent ||--o{ InquiryResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `InquiryEndpoint` | `api/InquiryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `InquiryEntity` | `application/InquiryEntity.java` (state in `domain/Inquiry.java`, events in `domain/InquiryEvent.java`) |
| `SpecialCategoryScreener` | `application/SpecialCategoryScreener.java` |
| `InquiryWorkflow` | `application/InquiryWorkflow.java` |
| `HrInquiryAgent` | `application/HrInquiryAgent.java` (tasks in `application/InquiryTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `HrRequestSubmitter` | `application/HrRequestSubmitter.java` |
| `PolicyCatalogLoader` | `application/PolicyCatalogLoader.java` |
| `InquiryView` | `application/InquiryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitScreenedStep` 15 s, `answerStep` 60 s, `submitRequestStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(InquiryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"inquiry-" + inquiryId` as the workflow id; the `SpecialCategoryScreener` Consumer is allowed to redeliver `InquirySubmitted` events because `InquiryEntity.attachScreened` is event-version-guarded — a second screen attempt against an already-screened inquiry is a no-op.
- **One agent per inquiry**: the AutonomousAgent instance id is `"inquirer-" + inquiryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ResponseGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Request submission is synchronous and deterministic**: `HrRequestSubmitter` runs in-process inside `submitRequestStep`. No LLM call, no external service — the same inquiryId always produces the same reference number format.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
