# PLAN — doc-linter

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

  UI[Browser UI]:::ep
  APP[AppEndpoint]:::ep
  LE[LintEndpoint]:::ep
  AG[LinterAgent]:::agent
  ENT[LintEntity]:::ese
  RP[RuleProfileEntity]:::ese
  VW[LintView]:::view
  FC[FeedbackConsumer]:::cons
  SIM[LintSimulator]:::ta

  UI --> APP
  UI --> LE
  LE --> AG
  AG --> ENT
  LE --> ENT
  RP --> AG
  ENT -. events .-> VW
  ENT -. events .-> FC
  FC --> RP
  SIM -.->|every 30s| LE
  VW --> LE
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor User
  participant LE as LintEndpoint
  participant ENT as LintEntity
  participant AG as LinterAgent
  participant Tool as MarkdownLintTool
  participant VW as LintView

  User->>LE: POST /api/lint {filePath}
  LE->>ENT: requestLint(filePath)
  ENT-->>VW: LintRequested
  LE->>AG: lint(runId, filePath, profile)
  AG->>Tool: lintMarkdown(filePath)
  Note over Tool: before-tool-call guardrail<br/>canonicalize path, reject traversal
  Tool-->>AG: List<Finding>
  AG-->>LE: LintResult{ordered, summary}
  LE->>ENT: recordResult(findings, summary, gate)
  ENT-->>VW: LintCompleted
  User->>LE: GET /api/lint/sse
  VW-->>User: LintRun (LINTED + gate)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> LINTED: recordResult
  REQUESTED --> BLOCKED: path traversal
  LINTED --> FEEDBACK_APPLIED: recordFeedback
  FEEDBACK_APPLIED --> FEEDBACK_APPLIED: recordFeedback
  BLOCKED --> [*]
  LINTED --> [*]
  FEEDBACK_APPLIED --> [*]
```

## Entity model

```mermaid
erDiagram
  LINT_ENTITY ||--o{ FINDING : contains
  LINT_ENTITY ||--o| FEEDBACK : records
  LINT_ENTITY }o--|| RULE_PROFILE : weighted-by
  LINT_ENTITY {
    string id
    string filePath
    enum status
    instant requestedAt
    optional_instant lintedAt
    optional_string summary
    optional_enum gate
  }
  FINDING {
    string rule
    int line
    enum severity
    string message
  }
  RULE_PROFILE {
    map weights
  }
  LINT_VIEW {
    string id
    enum status
    optional_enum gate
  }
```

## Component table

| Component | Akka primitive | Path (generated) |
|---|---|---|
| LinterAgent | Agent | `application/LinterAgent.java` |
| MarkdownLintTool | function tool | `application/MarkdownLintTool.java` |
| LintEntity | EventSourcedEntity | `application/LintEntity.java` |
| RuleProfileEntity | KeyValueEntity | `application/RuleProfileEntity.java` |
| LintView | View | `application/LintView.java` |
| FeedbackConsumer | Consumer | `application/FeedbackConsumer.java` |
| LintSimulator | TimedAction | `application/LintSimulator.java` |
| LintEndpoint | HttpEndpoint | `api/LintEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts** — no Workflow in this blueprint, so the 5-second default does not bite. The single `LinterAgent.lint` call runs request/response; the endpoint awaits it directly. If a Workflow is added later, override `settings()` with `stepTimeout >= 60s` on the agent-calling step (Lesson 4).
- **Idempotency** — `runId` is a UUID minted per request and used as the `LintEntity` id, so retries of the same request target the same entity.
- **Guardrail ordering** — the before-tool-call guardrail runs before any file read; a blocked run never touches the filesystem and lands in a terminal `BLOCKED` state.
- **No saga / compensation** — every step is internal and reversible by replaying events; there is no external side effect to compensate.
