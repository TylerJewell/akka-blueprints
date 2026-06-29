# PLAN — statement-auditor

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

  Auditor[AuditorAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[AuditWorkflow]:::wf
  Eng[EngagementEntity]:::ese
  Queue[StatementQueue]:::ese
  View[EngagementsView]:::view
  Consumer[StatementConsumer]:::cons
  Sim[StatementSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[AuditEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue section| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|analyze / revise| Auditor
  WF -->|review| Reviewer
  WF -->|emit events| Eng
  Eng -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Eng
```

## Interaction sequence — J1 (convergence on iteration 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as AuditEndpoint
  participant Q as StatementQueue
  participant C as StatementConsumer
  participant W as AuditWorkflow
  participant A as AuditorAgent
  participant R as ReviewerAgent
  participant E as EngagementEntity
  participant V as EngagementsView

  U->>API: POST /api/engagements {sectionId, rawText, workingPaperRefs}
  API->>Q: append SectionSubmitted
  API-->>U: 202 {engagementId}
  Q->>C: SectionSubmitted
  C->>W: start({engagementId, sectionId, maxAttempts=3})
  W->>E: emit EngagementCreated (ANALYZING)

  W->>A: ANALYZE(sectionId, rawText, workingPaperRefs)
  A-->>W: FindingReport #1 (3 findings, WP-001..WP-003)
  W->>E: emit IterationAnalyzed (n=1)
  Note over W: citationGuardrailStep (deterministic ref check)
  W->>E: emit IterationCitationVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW(FindingReport #1)
  R-->>W: Review{REVISE, score=3, 3 bullets}
  W->>E: emit IterationReviewed (n=1, REVISE)

  W->>A: REVISE_ANALYSIS(sectionId, priorReport, reviewNotes)
  A-->>W: FindingReport #2 (2 findings, tighter citations)
  W->>E: emit IterationAnalyzed (n=2)
  W->>E: emit IterationCitationVerdictRecorded (passed=true)
  W->>R: REVIEW(FindingReport #2)
  R-->>W: Review{APPROVED, score=5, rationale}
  W->>E: emit IterationReviewed (n=2, APPROVED)
  W->>E: emit EngagementSignedOff (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `EngagementEntity`

```mermaid
stateDiagram-v2
  [*] --> ANALYZING
  ANALYZING --> REVIEWING: IterationAnalyzed + citations passed
  ANALYZING --> ANALYZING: citation guardrail blocked, re-analyze
  REVIEWING --> ANALYZING: Review = REVISE, iterations < max
  REVIEWING --> SIGNED_OFF: Review = APPROVED
  REVIEWING --> UNRESOLVED: Review = REVISE, iterations = max
  SIGNED_OFF --> [*]
  UNRESOLVED --> [*]
```

## Entity model

```mermaid
erDiagram
  EngagementEntity ||--o{ EngagementCreated : emits
  EngagementEntity ||--o{ IterationAnalyzed : emits
  EngagementEntity ||--o{ IterationCitationVerdictRecorded : emits
  EngagementEntity ||--o{ IterationReviewed : emits
  EngagementEntity ||--o{ EngagementSignedOff : emits
  EngagementEntity ||--o{ EngagementUnresolved : emits
  EngagementEntity ||--o{ EvalRecorded : emits
  EngagementsView }o--|| EngagementEntity : projects
  StatementQueue ||--o{ SectionSubmitted : emits
  StatementConsumer }o--|| StatementQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AuditorAgent` | `application/AuditorAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `AuditTasks` | `application/AuditTasks.java` |
| `AuditWorkflow` | `application/AuditWorkflow.java` |
| `EngagementEntity` | `application/EngagementEntity.java` (state in `domain/Engagement.java`, events in `domain/EngagementEvent.java`) |
| `StatementQueue` | `application/StatementQueue.java` |
| `EngagementsView` | `application/EngagementsView.java` |
| `StatementConsumer` | `application/StatementConsumer.java` |
| `StatementSimulator` | `application/StatementSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `AuditEndpoint` | `api/AuditEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `analyzeStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(unresolvedStep))` — the workflow degrades to `UNRESOLVED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `AuditEndpoint.submit` uses `(sectionId, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(engagementId, iterationNumber)` so a tick that fires twice for the same iteration is a no-op on the entity side.
- **maxAttempts ceiling:** read from `statement-auditor.audit.max-attempts` (default 3). The workflow checks the count BEFORE calling `analyzeStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. `unresolvedStep` is the only terminal fallback; it preserves the best-scored finding report and every review on the entity.
- **Citation guardrail step:** `citationGuardrailStep` is pure-function (no LLM call); it checks each `Finding.citedWorkingPaper` against the engagement's `workingPaperRefs` list and either advances to `reviewStep` or returns to `analyzeStep` with a structured feedback note listing the unresolvable references.
