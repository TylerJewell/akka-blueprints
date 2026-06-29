# PLAN — financial-research-pipeline

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

  Planner[PlannerAgent]:::agent
  Search[SearchAgent]:::agent
  Analyst[AnalystAgent]:::agent
  Writer[WriterAgent]:::agent
  Verifier[VerifierAgent]:::agent

  WF[ResearchWorkflow]:::wf
  Report[ResearchEntity]:::ese
  Queue[QueryQueue]:::ese
  View[ReportsView]:::view
  Consumer[QueryConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Eval[EvalSampler]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue query| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|plan query| Planner
  WF -->|retrieve sources| Search
  WF -->|synthesise| Analyst
  WF -->|write / revise| Writer
  WF -->|verify| Verifier
  WF -->|emit events| Report
  Report -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 45s| Report
```

## Interaction sequence — J1 (convergence on draft 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ResearchEndpoint
  participant Q as QueryQueue
  participant C as QueryConsumer
  participant W as ResearchWorkflow
  participant PL as PlannerAgent
  participant SE as SearchAgent
  participant AN as AnalystAgent
  participant WR as WriterAgent
  participant VE as VerifierAgent
  participant E as ResearchEntity
  participant V as ReportsView

  U->>API: POST /api/reports {topic, sectorTag, wordCeiling}
  API->>Q: append QuerySubmitted
  API-->>U: 202 {reportId}
  Q->>C: QuerySubmitted
  C->>W: start({reportId, topic, sectorTag, wordCeiling, maxVerifyAttempts=3})
  W->>E: emit ReportCreated (PLANNING)

  W->>PL: PLAN_QUERY(topic, sectorTag)
  PL-->>W: ResearchPlan{3 subTasks}
  W->>E: emit PlanRecorded (RESEARCHING)

  W->>SE: RETRIEVE_SOURCES(subTask 1)
  SE-->>W: SourceBundle #1
  W->>SE: RETRIEVE_SOURCES(subTask 2)
  SE-->>W: SourceBundle #2
  W->>SE: RETRIEVE_SOURCES(subTask 3)
  SE-->>W: SourceBundle #3
  W->>E: emit SourceBundleAdded ×3

  W->>AN: SYNTHESISE(all SourceBundles)
  AN-->>W: List<AnalysisSection> (4 sections)
  W->>E: emit AnalysisSectionsRecorded (ANALYSING)
  Note over W: sanitiseStep (pure-function sector scrub)
  W->>E: emit SanitizerLogRecorded

  W->>WR: WRITE_DRAFT(sanitisedSections, wordCeiling)
  WR-->>W: ReportDraft #1 (820 words)
  W->>E: emit DraftWritten (WRITING)
  Note over W: guardrailStep — 820 > 800 ceiling
  W->>E: emit DraftGuardrailVerdictRecorded (passed=false)
  W->>WR: REVISE_DRAFT(sections, draft#1, VerificationNotes{OVER_WORD_CEILING})
  WR-->>W: ReportDraft #2 (780 words)
  W->>E: emit DraftWritten #2 (guardrail passed)

  W->>VE: VERIFY_DRAFT(ReportDraft #2)
  VE-->>W: Verification{REVISE, score=3, 3 bullets}
  W->>E: emit DraftVerified (VERIFYING)

  W->>WR: REVISE_DRAFT(sections, draft#2, VerificationNotes)
  WR-->>W: ReportDraft #3 (760 words)
  W->>E: emit DraftWritten #3
  W->>VE: VERIFY_DRAFT(ReportDraft #3)
  VE-->>W: Verification{APPROVE, score=5, rationale}
  W->>E: emit DraftVerified (APPROVE)
  W->>E: emit ReportApprovedPending (AWAITING_COMPLIANCE)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ResearchEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> RESEARCHING: PlanRecorded
  RESEARCHING --> ANALYSING: all SourceBundles added
  ANALYSING --> WRITING: AnalysisSectionsRecorded + SanitizerLogRecorded
  WRITING --> WRITING: guardrail blocked, re-write
  WRITING --> VERIFYING: guardrail passed
  VERIFYING --> WRITING: Verification = REVISE, drafts < max
  VERIFYING --> AWAITING_COMPLIANCE: Verification = APPROVE
  AWAITING_COMPLIANCE --> APPROVED: compliance reviewer approves
  VERIFYING --> REJECTED_FINAL: Verification = REVISE, drafts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchEntity ||--o{ ReportCreated : emits
  ResearchEntity ||--o{ PlanRecorded : emits
  ResearchEntity ||--o{ SourceBundleAdded : emits
  ResearchEntity ||--o{ AnalysisSectionsRecorded : emits
  ResearchEntity ||--o{ SanitizerLogRecorded : emits
  ResearchEntity ||--o{ DraftWritten : emits
  ResearchEntity ||--o{ DraftGuardrailVerdictRecorded : emits
  ResearchEntity ||--o{ DraftVerified : emits
  ResearchEntity ||--o{ ReportApprovedPending : emits
  ResearchEntity ||--o{ ReportApproved : emits
  ResearchEntity ||--o{ ReportRejectedFinal : emits
  ResearchEntity ||--o{ EvalRecorded : emits
  ReportsView }o--|| ResearchEntity : projects
  QueryQueue ||--o{ QuerySubmitted : emits
  QueryConsumer }o--|| QueryQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `SearchAgent` | `application/SearchAgent.java` |
| `AnalystAgent` | `application/AnalystAgent.java` |
| `WriterAgent` | `application/WriterAgent.java` |
| `VerifierAgent` | `application/VerifierAgent.java` |
| `ResearchTasks` | `application/ResearchTasks.java` |
| `ResearchWorkflow` | `application/ResearchWorkflow.java` |
| `ResearchEntity` | `application/ResearchEntity.java` (state in `domain/Report.java`, events in `domain/ReportEvent.java`) |
| `QueryQueue` | `application/QueryQueue.java` |
| `ReportsView` | `application/ReportsView.java` |
| `QueryConsumer` | `application/QueryConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** all agent-calling steps carry `stepTimeout(Duration.ofSeconds(90))`. Sanitiser and guardrail steps use `stepTimeout(Duration.ofSeconds(5))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Search fan-out:** sub-task searches run sequentially in the default blueprint. A deployer can convert these to parallel workflow branches; the sequential version is safer for rate-limited model providers.
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — any unrecoverable agent failure ends in `REJECTED_FINAL`.
- **Compliance parking:** `complianceStep` parks the workflow. The workflow does not consume a thread while parked; it resumes on the reviewer's HTTP call.
- **Idempotency:** `ResearchEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(reportId, draftNumber)`.
- **Guardrail step:** pure-function; checks word count; over-ceiling drafts produce `OVER_WORD_CEILING` and loop back to `writeStep` with a structured `VerificationNotes` payload.
- **Sanitiser step:** pure-function; applies regex list from config; the empty default list is a no-op. The log is always emitted even when `changed = false`.
