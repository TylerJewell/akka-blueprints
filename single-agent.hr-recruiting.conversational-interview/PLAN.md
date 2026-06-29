# PLAN — conversational-interview

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

  API[SessionEndpoint]:::ep
  Entity[InterviewSessionEntity]:::ese
  Sanitizer[AnswerSanitizer]:::cons
  WF[InterviewWorkflow]:::wf
  Agent[InterviewConductorAgent]:::agent
  Guard[QuestionGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|start| Entity
  API -->|submitAnswer| Entity
  Entity -.->|AnswerSubmitted| Sanitizer
  Sanitizer -->|attachScreenedAnswer| Entity
  Sanitizer -->|resume workflow| WF
  WF -->|awaitScreenedStep poll| Entity
  WF -->|conductTurnStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|InterviewTurn| WF
  WF -->|recordTurn| Entity
  WF -->|completeStep complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, first turn)

```mermaid
sequenceDiagram
  autonumber
  participant C as Coordinator (UI)
  participant API as SessionEndpoint
  participant E as InterviewSessionEntity
  participant S as AnswerSanitizer
  participant W as InterviewWorkflow
  participant A as InterviewConductorAgent
  participant G as QuestionGuardrail

  C->>API: POST /api/sessions
  API->>E: start(request)
  E-->>API: { sessionId }
  W->>A: conductTurnStep — runSingleTask(role.json + history.json attachments)
  A->>G: before-agent-response(candidate question)
  G-->>A: accept
  A-->>W: InterviewTurn (question 1)
  W->>E: recordTurn(turn)
  E-.->>C: SSE event(CONDUCTING → question ready)
  C->>API: POST /api/sessions/{id}/answers (answer 1)
  API->>E: submitAnswer(answer)
  E-.->>S: AnswerSubmitted
  S->>S: screen for protected categories
  S->>E: attachScreenedAnswer
  S->>W: resume(awaitScreenedStep)
  W->>E: poll getSession
  E-->>W: screened answer present
  W->>A: conductTurnStep — runSingleTask(role.json + updated history.json)
  A->>G: before-agent-response(candidate question)
  G-->>A: accept
  A-->>W: InterviewTurn (question 2 or sessionComplete: true)
  W->>E: recordTurn(turn)
  E-.->>C: SSE event(CONDUCTING or COMPLETED)
```

## State machine — `InterviewSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CONDUCTING: TurnConducted (first question)
  CONDUCTING --> ANSWER_RECEIVED: AnswerSubmitted
  ANSWER_RECEIVED --> ANSWER_SCREENED: AnswerScreened
  ANSWER_SCREENED --> CONDUCTING: TurnConducted (next question)
  CONDUCTING --> COMPLETED: SessionCompleted (sessionComplete flag)
  CONDUCTING --> FAILED: SessionFailed (agent error)
  OPEN --> FAILED: SessionFailed (startup error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  InterviewSessionEntity ||--o{ SessionStarted : emits
  InterviewSessionEntity ||--o{ AnswerSubmitted : emits
  InterviewSessionEntity ||--o{ AnswerScreened : emits
  InterviewSessionEntity ||--o{ TurnConducted : emits
  InterviewSessionEntity ||--o{ SessionCompleted : emits
  InterviewSessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| InterviewSessionEntity : projects
  AnswerSanitizer }o--|| InterviewSessionEntity : subscribes
  InterviewWorkflow }o--|| InterviewSessionEntity : reads-and-writes
  InterviewConductorAgent ||--o{ InterviewTurn : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `InterviewSessionEntity` | `application/InterviewSessionEntity.java` (state in `domain/InterviewSession.java`, events in `domain/SessionEvent.java`) |
| `AnswerSanitizer` | `application/AnswerSanitizer.java` |
| `InterviewWorkflow` | `application/InterviewWorkflow.java` |
| `InterviewConductorAgent` | `application/InterviewConductorAgent.java` (tasks in `application/InterviewTasks.java`) |
| `QuestionGuardrail` | `application/QuestionGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `conductTurnStep` 60 s, `awaitScreenedStep` 15 s, `completeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(InterviewWorkflow::error)`. The 60 s on `conductTurnStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"interview-" + sessionId` as the workflow id; the `AnswerSanitizer` Consumer is allowed to redeliver `AnswerSubmitted` events because `InterviewSessionEntity.attachScreenedAnswer` is event-version-guarded — a second screen attempt against an already-screened turn is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"conductor-" + sessionId`, giving each session its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `QuestionGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `conductTurnStep` fails over to `error` and the entity transitions to `FAILED`.
- **Turn loop**: after recording a turn with `sessionComplete: false`, the workflow waits for the next `AnswerSubmitted` event (triggered when the coordinator submits the candidate's answer). `AnswerSanitizer` screens the answer and resumes the workflow to `awaitScreenedStep`. This cycle repeats until `sessionComplete: true`.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
