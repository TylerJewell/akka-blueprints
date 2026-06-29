# PLAN — llm-auditor

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

  Critic[CriticAgent]:::agent
  Reviser[ReviserAgent]:::agent

  WF[AuditWorkflow]:::wf
  Session[SessionEntity]:::ese
  Queue[ResponseQueue]:::ese
  View[SessionsView]:::view
  Consumer[ResponseIngestConsumer]:::cons
  Sim[ResponseSimulator]:::ta
  Sampler[AuditSampler]:::ta
  API[AuditEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue response| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|guardrail check| WF
  WF -->|audit| Critic
  WF -->|revise| Reviser
  WF -->|emit events| Session
  Session -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| Session
```

## Interaction sequence — J1 (convergence on cycle 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as AuditEndpoint
  participant Q as ResponseQueue
  participant C as ResponseIngestConsumer
  participant W as AuditWorkflow
  participant K as CriticAgent
  participant R as ReviserAgent
  participant E as SessionEntity
  participant V as SessionsView

  U->>API: POST /api/sessions {text, channel}
  API->>Q: append ResponseReceived
  API-->>U: 202 {sessionId}
  Q->>C: ResponseReceived
  C->>W: start({sessionId, channel, maxRevisions=3})
  W->>E: emit SessionCreated (AUDITING)

  Note over W: guardrailStep (deterministic severity check)
  W->>E: emit GuardrailVerdictRecorded (passed=true, severity=4)
  W->>E: emit CycleStarted (n=1)
  W->>K: AUDIT(response text)
  K-->>W: AuditVerdict{REVISE, score=5, 2 findings}
  W->>E: emit AuditVerdictRecorded (n=1, REVISE)
  W->>E: status REVISING

  W->>R: REVISE_RESPONSE(text, findings)
  R-->>W: RevisedResponse (revised text)
  W->>E: emit RevisionProduced (n=1)
  W->>E: emit CycleStarted (n=2)
  W->>K: AUDIT(revised text)
  K-->>W: AuditVerdict{PASS, score=1, rationale}
  W->>E: emit AuditVerdictRecorded (n=2, PASS)
  W->>E: emit SessionApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> AUDITING
  AUDITING --> ESCALATED: guardrail blocked (severity >= threshold)
  AUDITING --> REVISING: AuditVerdict = REVISE, cycles < max
  AUDITING --> APPROVED: AuditVerdict = PASS
  REVISING --> AUDITING: RevisionProduced, next audit cycle starts
  REVISING --> ESCALATED: cycles = max, last verdict REVISE
  APPROVED --> [*]
  ESCALATED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ CycleStarted : emits
  SessionEntity ||--o{ GuardrailVerdictRecorded : emits
  SessionEntity ||--o{ AuditVerdictRecorded : emits
  SessionEntity ||--o{ RevisionProduced : emits
  SessionEntity ||--o{ SessionApproved : emits
  SessionEntity ||--o{ SessionEscalated : emits
  SessionEntity ||--o{ AuditRecorded : emits
  SessionsView }o--|| SessionEntity : projects
  ResponseQueue ||--o{ ResponseReceived : emits
  ResponseIngestConsumer }o--|| ResponseQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CriticAgent` | `application/CriticAgent.java` |
| `ReviserAgent` | `application/ReviserAgent.java` |
| `AuditTasks` | `application/AuditTasks.java` |
| `AuditWorkflow` | `application/AuditWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `ResponseQueue` | `application/ResponseQueue.java` |
| `SessionsView` | `application/SessionsView.java` |
| `ResponseIngestConsumer` | `application/ResponseIngestConsumer.java` |
| `ResponseSimulator` | `application/ResponseSimulator.java` |
| `AuditSampler` | `application/AuditSampler.java` |
| `AuditEndpoint` | `api/AuditEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `auditStep` and `reviseStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(escalateStep))` — the workflow degrades to `ESCALATED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `AuditEndpoint.submit` uses `(text hash, channel)` over a 10 s window as the dedup key.
- **AuditSampler idempotency:** the sampler keys its `recordAuditEvent` calls on `(sessionId, cycleNumber)` so a tick that fires twice for the same cycle is a no-op on the entity side.
- **maxRevisions ceiling:** read from `llm-auditor.audit.max-revisions` (default 3). The workflow checks the count BEFORE calling `auditStep` for the next cycle; it never recurses past the ceiling.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans keyword indicators in the response text, computes an `initialSeverity` (0–10), and either advances to `auditStep` or escalates immediately. The severity computation is deterministic and never makes an LLM call.
- **Saga semantics:** there is no external side-effect to compensate. The escalation mechanism is the only termination path beyond approval; it preserves the lowest-severity revision and every finding set on the entity.
