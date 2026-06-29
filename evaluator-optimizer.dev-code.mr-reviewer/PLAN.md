# PLAN — mr-reviewer

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

  Reviewer[ReviewerAgent]:::agent
  Gate[GateAgent]:::agent

  WF[ReviewWorkflow]:::wf
  Mr[MrEntity]:::ese
  Queue[MrQueue]:::ese
  View[MrView]:::view
  Consumer[WebhookConsumer]:::cons
  Sim[MrSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[ReviewEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue webhook| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|sanitize diff| WF
  WF -->|review / refine| Reviewer
  WF -->|gate review| Gate
  WF -->|emit events| Mr
  Mr -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Mr
```

## Interaction sequence — J1 (convergence on pass 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ReviewEndpoint
  participant Q as MrQueue
  participant C as WebhookConsumer
  participant W as ReviewWorkflow
  participant S as SanitizerStep
  participant R as ReviewerAgent
  participant G as GateAgent
  participant E as MrEntity
  participant V as MrView

  U->>API: POST /api/mrs {projectPath, mrIid, diffText}
  API->>Q: append WebhookReceived
  API-->>U: 202 {mrId}
  Q->>C: WebhookReceived
  C->>W: start({mrId, diffText, maxPasses=3})
  W->>E: emit MrReceived (RECEIVED)

  Note over W,S: sanitizerStep — deterministic diff scan
  W->>E: emit SanitizerVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW_DIFF(diffText)
  R-->>W: ReviewResult #1 {findings, qualityScore=68}
  W->>E: emit ReviewPassDrafted (n=1)
  Note over W: guardrailStep — commentary pattern check
  W->>E: emit ReviewGuardrailVerdictRecorded (passed=true)
  W->>E: status GATE_CHECKING
  W->>G: GATE_REVIEW(ReviewResult #1)
  G-->>W: GateVerdict{REFINE, gateScore=55, 3 bullets}
  W->>E: emit ReviewPassGated (n=1, REFINE)

  W->>R: REFINE_REVIEW(diffText, priorReview, gateFeedback)
  R-->>W: ReviewResult #2 {findings, qualityScore=82}
  W->>E: emit ReviewPassDrafted (n=2)
  W->>E: emit ReviewGuardrailVerdictRecorded (passed=true)
  W->>G: GATE_REVIEW(ReviewResult #2)
  G-->>W: GateVerdict{PASS, gateScore=78, rationale}
  W->>E: emit ReviewPassGated (n=2, PASS)
  W->>E: emit MrCiPassed (CI_PASS)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `MrEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZER_BLOCKED: SanitizerVerdictRecorded passed=false
  RECEIVED --> REVIEWING: SanitizerVerdictRecorded passed=true
  REVIEWING --> REVIEWING: guardrail blocked, re-review
  REVIEWING --> GATE_CHECKING: ReviewGuardrailVerdictRecorded passed=true
  GATE_CHECKING --> REVIEWING: GateDecision = REFINE, passes < max
  GATE_CHECKING --> CI_PASS: GateDecision = PASS
  GATE_CHECKING --> CI_FAIL: GateDecision = REFINE, passes = max
  SANITIZER_BLOCKED --> [*]
  CI_PASS --> [*]
  CI_FAIL --> [*]
```

## Entity model

```mermaid
erDiagram
  MrEntity ||--o{ MrReceived : emits
  MrEntity ||--o{ SanitizerVerdictRecorded : emits
  MrEntity ||--o{ ReviewPassDrafted : emits
  MrEntity ||--o{ ReviewGuardrailVerdictRecorded : emits
  MrEntity ||--o{ ReviewPassGated : emits
  MrEntity ||--o{ MrCiPassed : emits
  MrEntity ||--o{ MrCiFailed : emits
  MrEntity ||--o{ ReviewEvalRecorded : emits
  MrView }o--|| MrEntity : projects
  MrQueue ||--o{ WebhookReceived : emits
  WebhookConsumer }o--|| MrQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `GateAgent` | `application/GateAgent.java` |
| `ReviewTasks` | `application/ReviewTasks.java` |
| `ReviewWorkflow` | `application/ReviewWorkflow.java` |
| `MrEntity` | `application/MrEntity.java` (state in `domain/MergeRequest.java`, events in `domain/MrEvent.java`) |
| `MrQueue` | `application/MrQueue.java` |
| `MrView` | `application/MrView.java` |
| `WebhookConsumer` | `application/WebhookConsumer.java` |
| `MrSimulator` | `application/MrSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ReviewEndpoint` | `api/ReviewEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `reviewStep` and `gateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(ciFailStep))` — the workflow degrades to `CI_FAIL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `ReviewEndpoint.submit` uses `(projectPath, mrIid)` over a 10 s window as the dedup key; a second webhook for the same MR returns the first `mrId` (200) rather than starting a new workflow.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(mrId, passNumber)` so a tick that fires twice for the same pass is a no-op on the entity side.
- **maxPasses ceiling:** read from `mr-reviewer.review.max-passes` (default 3). The workflow checks the count BEFORE calling `reviewStep` for the next iteration; it never recurses past the ceiling.
- **Sanitizer step:** `sanitizerStep` is pure-function (no LLM call); it scans the raw diffText against a fixed pattern set and either advances to `reviewStep` or transitions to `blockStep`. The diff is never sent to the LLM on a sanitizer failure.
- **Guardrail step:** `guardrailStep` is pure-function; it checks `ReviewResult.commentary` against the same pattern set and either advances to `gateStep` or returns to `reviewStep` with a structured `GateFeedback` payload.
- **CI gate signal:** `MrEntity.ciSignal` records `"CI_PASS"` or `"CI_FAIL"` as a plain string. `ReviewEndpoint GET /api/mrs/{id}/ci-signal` returns this field directly; external pipelines need no additional authentication to read it.
