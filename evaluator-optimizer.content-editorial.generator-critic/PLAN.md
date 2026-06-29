# PLAN — generator-critic

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Gen[GeneratorAgent]:::agent
  Ref[ReflectorAgent]:::agent

  WF[ReflectionWorkflow]:::wf
  Essay[EssayEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[EssaysView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[TopicSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[EssayEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit topic| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|draft / revise| Gen
  WF -->|reflect| Ref
  WF -->|emit events| Essay
  Essay -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Essay
```

## Interaction sequence — J1 (convergence on round 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EssayEndpoint
  participant Q as SubmissionQueue
  participant C as SubmissionConsumer
  participant W as ReflectionWorkflow
  participant G as GeneratorAgent
  participant R as ReflectorAgent
  participant E as EssayEntity
  participant V as EssaysView

  U->>API: POST /api/essays {topic, wordCeiling}
  API->>Q: append TopicSubmitted
  API-->>U: 202 {essayId}
  Q->>C: TopicSubmitted
  C->>W: start({essayId, topic, wordCeiling, maxRounds=4})
  W->>E: emit EssayCreated (DRAFTING)

  W->>G: DRAFT_ESSAY(topic, wordCeiling)
  G-->>W: EssayDraft #1 (380 words)
  W->>E: emit RoundDrafted (n=1)
  W->>R: REFLECT(EssayDraft #1)
  R-->>W: Reflection{REVISE, score=3, 3 bullets}
  W->>E: emit RoundReflected (n=1, REVISE)
  Note over W: status → REFLECTING

  W->>G: REVISE_ESSAY(topic, prior, notes)
  G-->>W: EssayDraft #2 (360 words)
  W->>E: emit RoundDrafted (n=2)
  W->>R: REFLECT(EssayDraft #2)
  R-->>W: Reflection{ACCEPT, score=5, rationale}
  W->>E: emit RoundReflected (n=2, ACCEPT)
  W->>E: emit EssayAccepted (n=2)
  Note over W: guardrailStep (deterministic policy check)
  W->>E: emit RoundGuardrailVerdictRecorded (passed=true)
  W->>E: emit EssayReleased
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `EssayEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> REFLECTING: RoundDrafted
  REFLECTING --> DRAFTING: Reflection = REVISE, rounds < max
  REFLECTING --> ACCEPTED: Reflection = ACCEPT
  ACCEPTED --> RELEASED: Guardrail passed
  ACCEPTED --> DRAFTING: Guardrail POLICY_VIOLATION, policy-revise
  REFLECTING --> REJECTED_FINAL: Reflection = REVISE, rounds = max
  RELEASED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  EssayEntity ||--o{ EssayCreated : emits
  EssayEntity ||--o{ RoundDrafted : emits
  EssayEntity ||--o{ RoundGuardrailVerdictRecorded : emits
  EssayEntity ||--o{ RoundReflected : emits
  EssayEntity ||--o{ EssayAccepted : emits
  EssayEntity ||--o{ EssayReleased : emits
  EssayEntity ||--o{ EssayRejectedFinal : emits
  EssayEntity ||--o{ ReflectionRecorded : emits
  EssaysView }o--|| EssayEntity : projects
  SubmissionQueue ||--o{ TopicSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GeneratorAgent` | `application/GeneratorAgent.java` |
| `ReflectorAgent` | `application/ReflectorAgent.java` |
| `EssayTasks` | `application/EssayTasks.java` |
| `ReflectionWorkflow` | `application/ReflectionWorkflow.java` |
| `EssayEntity` | `application/EssayEntity.java` (state in `domain/Essay.java`, events in `domain/EssayEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `EssaysView` | `application/EssaysView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `TopicSimulator` | `application/TopicSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `EssayEndpoint` | `api/EssayEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `draftStep` and `reflectStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `EssayEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordReflectionEval` calls on `(essayId, roundNumber)` so a tick that fires twice for the same round is a no-op on the entity side.
- **maxRounds ceiling:** read from `basic-reflection.reflection.max-rounds` (default 4). The workflow checks the count BEFORE calling `draftStep` for the next round; it never recurses past the ceiling.
- **Policy-revise loop:** the guardrail → policyReviseStep → guardrailStep mini-loop does not count toward `maxRounds`; it is bounded by the `stepTimeout` and `maxRetries` on the policyReviseStep. An irrecoverable policy failure also fails over to `rejectStep`.
- **Saga semantics:** there are no external side-effects to compensate. The halt mechanism is the only terminal boundary; it preserves the best draft and every reflection on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans the draft text against the phrase list from `basic-reflection.policy.forbidden-phrases`. The check runs once per acceptance; it does not run during the reflect-revise inner loop.
