# PLAN — voice-agent

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
  classDef stub fill:#1a1a2e,stroke:#6c6cff,color:#6c6cff;

  API[ConversationEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  Sanitizer[TranscriptSanitizer]:::cons
  WF[ConversationWorkflow]:::wf
  Agent[VoiceConversationAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  TTS[AudioSynthesiser]:::stub
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|createSession / addTurn| Entity
  Entity -.->|AudioReceived| Sanitizer
  Sanitizer -->|attachSanitizedTranscript| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|respondStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|synthesiseStep| TTS
  TTS -->|AudioResponse| WF
  WF -->|recordAudio| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ConversationEndpoint
  participant E as ConversationEntity
  participant S as TranscriptSanitizer
  participant W as ConversationWorkflow
  participant A as VoiceConversationAgent
  participant G as ReplyGuardrail
  participant TTS as AudioSynthesiser

  U->>API: POST /api/conversations
  API->>E: createSession + addTurn
  E-->>API: { conversationId, turnId }
  E-.->>S: AudioReceived
  S->>S: redact PII from transcript
  S->>E: attachSanitizedTranscript
  S->>W: start(turnId)
  W->>E: poll getSession
  E-->>W: turn.sanitized.isPresent()
  W->>E: markResponding
  W->>A: runSingleTask(history + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AgentReply
  W->>E: recordReply(reply)
  W->>TTS: synthesise(replyText)
  TTS-->>W: AudioResponse
  W->>E: recordAudio(audio)
  E-.->>U: SSE event(AUDIO_SYNTHESISED)
```

## State machine — turn lifecycle in `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> AUDIO_RECEIVED
  AUDIO_RECEIVED --> TRANSCRIPT_SANITIZED: TranscriptSanitized
  TRANSCRIPT_SANITIZED --> RESPONDING: ResponseStarted
  RESPONDING --> REPLY_RECORDED: ReplyRecorded
  REPLY_RECORDED --> AUDIO_SYNTHESISED: AudioSynthesised
  RESPONDING --> FAILED: TurnFailed (agent error)
  AUDIO_RECEIVED --> FAILED: TurnFailed (sanitizer error)
  AUDIO_SYNTHESISED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ AudioReceived : emits
  ConversationEntity ||--o{ TranscriptSanitized : emits
  ConversationEntity ||--o{ ResponseStarted : emits
  ConversationEntity ||--o{ ReplyRecorded : emits
  ConversationEntity ||--o{ AudioSynthesised : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationEntity ||--o{ SessionClosed : emits
  ConversationView }o--|| ConversationEntity : projects
  TranscriptSanitizer }o--|| ConversationEntity : subscribes
  ConversationWorkflow }o--|| ConversationEntity : reads-and-writes
  VoiceConversationAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/ConversationSession.java`, events in `domain/ConversationEvent.java`) |
| `TranscriptSanitizer` | `application/TranscriptSanitizer.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `VoiceConversationAgent` | `application/VoiceConversationAgent.java` (tasks in `application/ConversationTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `AudioSynthesiser` | `application/AudioSynthesiser.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `respondStep` 60 s, `synthesiseStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ConversationWorkflow::error)`. The 60 s on `respondStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"turn-" + turnId` as the workflow id; the `TranscriptSanitizer` Consumer may redeliver `AudioReceived` events; `ConversationEntity.attachSanitizedTranscript` is event-version-guarded — a second sanitize attempt against an already-sanitized turn is a no-op.
- **One agent per turn**: the AutonomousAgent instance id is `"voice-" + turnId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `respondStep` fails over to `error` and the entity transitions to `FAILED`.
- **TTS is synchronous and deterministic**: `AudioSynthesiser` runs in-process inside `synthesiseStep`. The stub returns a WAV byte array without any external call — the same reply always produces the same audio length. Production deployers wire a real TTS interface without changing the step logic.
- **Multi-turn sessions**: a single `ConversationEntity` accumulates all `Turn` records in its state list. Each new turn starts an independent `ConversationWorkflow`; workflows do not share state.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. Nothing requires rollback.
