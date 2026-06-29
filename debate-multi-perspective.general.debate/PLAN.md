# PLAN — debate

Architectural sketch. All four mermaid diagrams + the component table are required. Diagrams render on the Architecture tab; the generated `index.html` must carry the Lesson 24 state-label CSS overrides.

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

  SIM[DebateSimulator]:::ta -. drips topics .-> IRQ[InboundRequestQueue]:::ese
  EP[DebateEndpoint]:::ep --> IRQ
  IRQ -. events .-> DRC[DebateRequestConsumer]:::cons
  DRC --> WF[DebateModerator]:::wf
  WF --> ADV[Advocate]:::agent
  WF --> CRI[Critic]:::agent
  WF --> SYN[Synthesizer]:::agent
  WF --> DE[DebateEntity]:::ese
  DE -. events .-> DV[DebatesView]:::view
  DE -. events .-> SEC[SynthesisEvalConsumer]:::cons
  SEC --> DE
  DV --> EP
  APP[AppEndpoint]:::ep -. serves UI .-> EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User / Simulator
  participant EP as DebateEndpoint
  participant IRQ as InboundRequestQueue
  participant WF as DebateModerator
  participant A as Advocate
  participant C as Critic
  participant S as Synthesizer
  participant DE as DebateEntity
  U->>EP: POST /api/debates {topic}
  EP->>IRQ: enqueueRequest(topic)
  IRQ-->>WF: start workflow (consumer)
  WF->>DE: DebateStarted
  loop up to 5 rounds
    WF->>A: runSingleTask(advocate)
    A-->>WF: Argument(for)
    WF->>C: runSingleTask(critic, advocate ctx)
    C-->>WF: Argument(against)
    WF->>DE: RoundRecorded
  end
  Note over WF,S: synthesis is guardrailed before persistence
  WF->>S: runSingleTask(synthesize)
  S-->>WF: Synthesis(conclusion, keyArguments)
  WF->>DE: SynthesisRecorded
  Note over DE: SynthesisEvalConsumer records SynthesisEvaluated (score)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> DEBATING: DebateStarted
  DEBATING --> DEBATING: RoundRecorded (round < maxRounds)
  DEBATING --> SYNTHESIZING: rounds exhausted / converged
  SYNTHESIZING --> CONCLUDED: SynthesisRecorded
  DEBATING --> FAILED: DebateFailed
  SYNTHESIZING --> FAILED: guardrail block / DebateFailed
  CONCLUDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DEBATE ||--o{ ROUND : contains
  DEBATE {
    string id
    string topic
    string status
    int currentRound
    int maxRounds
    string conclusion
    double qualityScore
  }
  ROUND {
    int number
    string advocateArgument
    string criticArgument
  }
  INBOUND_REQUEST {
    string topic
  }
  DEBATE ||--|| DEBATES_VIEW : projects
  INBOUND_REQUEST ||--|| DEBATE : starts
```

## Component table

| Component | Path (generated) |
|---|---|
| `Advocate` | `application/Advocate.java` |
| `Critic` | `application/Critic.java` |
| `Synthesizer` | `application/Synthesizer.java` |
| `DebateTasks` | `application/DebateTasks.java` |
| `DebateModerator` | `application/DebateModerator.java` |
| `DebateEntity` | `application/DebateEntity.java` |
| `InboundRequestQueue` | `application/InboundRequestQueue.java` |
| `DebatesView` | `application/DebatesView.java` |
| `DebateRequestConsumer` | `application/DebateRequestConsumer.java` |
| `SynthesisEvalConsumer` | `application/SynthesisEvalConsumer.java` |
| `DebateSimulator` | `application/DebateSimulator.java` |
| `DebateEndpoint` | `api/DebateEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| domain records | `domain/Debate.java`, `domain/Round.java`, `domain/Argument.java`, `domain/Synthesis.java` |

## Concurrency notes

- **Step timeouts:** `roundStep` and `synthesisStep` call agents — override `stepTimeout(60s)` on both (Lesson 4). Default 5s would time out on every LLM call.
- **Round bound:** `maxRounds` = 5 caps the loop; `roundStep` self-transitions until `currentRound == maxRounds` or the sides converge, then moves to `synthesisStep`.
- **Idempotency:** the workflow keys on the debate UUID minted by `DebateRequestConsumer`; re-delivery of the same inbound event resolves to the same workflow id.
- **Failure / compensation:** `defaultStepRecovery(maxRetries(2).failoverTo(error))`; the `error` step records `DebateFailed` so a stalled or guardrail-blocked debate reaches a terminal `FAILED` state rather than hanging.
- **Eval is non-blocking:** `SynthesisEvalConsumer` runs after `SynthesisRecorded` and writes `SynthesisEvaluated` asynchronously; the debate is already `CONCLUDED` and visible before the score lands.
