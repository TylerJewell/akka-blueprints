# PLAN — ab-model-eval

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

  CandA[CandidateAAgent]:::agent
  CandB[CandidateBAgent]:::agent
  Judge[JudgeAgent]:::agent

  WF[EvalWorkflow]:::wf
  Trial[TrialEntity]:::ese
  Queue[TaskQueueEntity]:::ese
  View[TrialsView]:::view
  Consumer[TaskSubmissionConsumer]:::cons
  Sim[TaskSimulator]:::ta
  Acc[AccuracySampler]:::ta
  API[EvalEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue task| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|answer A| CandA
  WF -->|answer B| CandB
  WF -->|judge both| Judge
  WF -->|emit events| Trial
  Trial -.->|projects| View
  API -->|query / SSE| View
  Acc -.->|every 30s| Trial
```

## Interaction sequence — J1 (normal trial, A wins)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EvalEndpoint
  participant Q as TaskQueueEntity
  participant C as TaskSubmissionConsumer
  participant W as EvalWorkflow
  participant A as CandidateAAgent
  participant B as CandidateBAgent
  participant J as JudgeAgent
  participant T as TrialEntity
  participant V as TrialsView

  U->>API: POST /api/trials {text, preferredCandidate="A"}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {trialId}
  Q->>C: TaskSubmitted
  C->>W: start({trialId, text, preferredCandidate="A"})
  W->>T: emit TrialCreated (RUNNING)

  par fan-out
    W->>A: ANSWER_A(text)
    A-->>W: CandidateResponse{id="A", text, tokens}
    W->>T: emit CandidateAResponded
  and
    W->>B: ANSWER_B(text)
    B-->>W: CandidateResponse{id="B", text, tokens}
    W->>T: emit CandidateBResponded
  end

  W->>J: JUDGE(text, responseA, responseB)
  J-->>W: JudgementRecord{winner="A", scoresA, scoresB, rationale}
  W->>T: emit TrialJudged (JUDGED)
  Note over W: recertStep checks win-rate
  W->>T: (no RecertificationFlagged — threshold not crossed)
  T-->>V: project
  V-->>U: SSE update
```

## State machine — `TrialEntity`

```mermaid
stateDiagram-v2
  [*] --> RUNNING
  RUNNING --> JUDGED: both responses received + JudgeAgent returns verdict
  RUNNING --> TIMED_OUT: candidateAStep or candidateBStep times out
  JUDGED --> JUDGED: RecertificationFlagged (flag set; status unchanged)
  JUDGED --> [*]
  TIMED_OUT --> [*]
```

## Entity model

```mermaid
erDiagram
  TrialEntity ||--o{ TrialCreated : emits
  TrialEntity ||--o{ CandidateAResponded : emits
  TrialEntity ||--o{ CandidateBResponded : emits
  TrialEntity ||--o{ TrialJudged : emits
  TrialEntity ||--o{ TrialTimedOut : emits
  TrialEntity ||--o{ RecertificationFlagged : emits
  TrialEntity ||--o{ EvalRecorded : emits
  TrialsView }o--|| TrialEntity : projects
  TaskQueueEntity ||--o{ TaskSubmitted : emits
  TaskSubmissionConsumer }o--|| TaskQueueEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CandidateAAgent` | `application/CandidateAAgent.java` |
| `CandidateBAgent` | `application/CandidateBAgent.java` |
| `JudgeAgent` | `application/JudgeAgent.java` |
| `EvalTasks` | `application/EvalTasks.java` |
| `EvalWorkflow` | `application/EvalWorkflow.java` |
| `TrialEntity` | `application/TrialEntity.java` (state in `domain/Trial.java`, events in `domain/TrialEvent.java`) |
| `TaskQueueEntity` | `application/TaskQueueEntity.java` |
| `TrialsView` | `application/TrialsView.java` |
| `TaskSubmissionConsumer` | `application/TaskSubmissionConsumer.java` |
| `TaskSimulator` | `application/TaskSimulator.java` |
| `AccuracySampler` | `application/AccuracySampler.java` |
| `EvalEndpoint` | `api/EvalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Fan-out parallelism:** `candidateAStep` and `candidateBStep` run in parallel branches of `EvalWorkflow`. Each carries `stepTimeout(Duration.ofSeconds(60))`; the default 5-second timeout never applies (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(1).failoverTo(timeoutStep))` — any unrecoverable agent failure ends in `TIMED_OUT`, not in a hung workflow.
- **Idempotency:** `EvalEndpoint.submit` deduplicates on `(text, submittedBy)` over a 10-second window.
- **AccuracySampler idempotency:** the sampler keys its `recordEval` calls on `trialId` so a tick that fires twice for the same trial is a no-op on the entity side.
- **Recertification evaluation:** `recertStep` is a pure-function step that reads the last `window-trials` TrialRow records from `TrialsView` in memory; it does not call any agent and is effectively instant.
- **Win-rate computation:** computed on the view side (TrialRow carries `winRateA` and `winRateB`) over all `JUDGED` trials. The workflow's `recertStep` reads these computed fields; it does not aggregate from scratch.
- **Promotion gate:** `POST /api/trials/promote` checks the `recertificationRequired` flag on any trial in the last `window-trials` window. The flag clears when `POST /api/trials/recertify` is called, which calls `TrialEntity.flagRecertification(false)` on all flagged trials and resets the view-side computed field.
