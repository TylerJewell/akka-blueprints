# PLAN — Fair CV Matcher

Architectural sketch for the sequential-pipeline × hr-recruiting blueprint. The generated system renders these diagrams on the Architecture tab. All mermaid blocks carry the Akka theme variables and the Lesson 24 CSS overrides for state labels and edge-label overflow.

## Component graph

Solid arrows = synchronous commands; dashed arrows = event subscriptions; dotted arrows = scheduled ticks.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#E6D200','lineColor':'#E6D200','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
flowchart TB
  SIM[CvSimulator<br/>TimedAction]
  EP[MatchingEndpoint<br/>HttpEndpoint]
  Q[CvIntakeQueue<br/>EventSourcedEntity]
  CON[CvIntakeConsumer<br/>Consumer]
  WF[MatchingWorkflow<br/>Workflow]
  EX[ExtractionAgent<br/>AutonomousAgent]
  MA[MatchingAgent<br/>AutonomousAgent]
  ENT[CandidateEntity<br/>EventSourcedEntity]
  VW[MatchesView<br/>View]
  FM[FairnessMonitor<br/>TimedAction]
  APP[AppEndpoint<br/>HttpEndpoint]

  SIM -.-> Q
  EP --> Q
  Q -.-> CON
  CON --> WF
  WF --> EX
  WF --> MA
  WF --> ENT
  ENT -.-> VW
  EP --> ENT
  EP --> VW
  FM -.-> VW
  FM --> ENT
  APP --> EP
```

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#E6D200','lineColor':'#E6D200','actorTextColor':'#ffffff','noteTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
sequenceDiagram
  participant U as User
  participant EP as MatchingEndpoint
  participant WF as MatchingWorkflow
  participant EX as ExtractionAgent
  participant MA as MatchingAgent
  participant ENT as CandidateEntity
  U->>EP: POST /api/candidates {cv, postings}
  EP->>ENT: CvSubmitted
  EP->>WF: start(candidateId)
  WF->>EX: runSingleTask(EXTRACT)
  EX-->>WF: CvProfile
  WF->>ENT: ProfileExtracted
  Note over WF: sanitize step strips special-category signals
  WF->>ENT: ProfileSanitized
  WF->>MA: runSingleTask(MATCH, sanitized profile)
  MA-->>WF: List<MatchResult>
  WF->>ENT: MatchesScored
  Note over WF,ENT: released for human-on-the-loop review
  U->>EP: POST /api/candidates/{id}/review
  EP->>ENT: ReviewRecorded
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#E6D200','lineColor':'#E6D200','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'}}}%%
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> EXTRACTED: ProfileExtracted
  EXTRACTED --> SANITIZED: ProfileSanitized
  SANITIZED --> MATCHED: MatchesScored
  MATCHED --> REVIEWED: ReviewRecorded
  REVIEWED --> [*]
```

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#E6D200','lineColor':'#E6D200','fontFamily':'Instrument Sans, sans-serif'}}}%%
erDiagram
  CANDIDATE ||--o{ REDACTED_SIGNAL : strips
  CANDIDATE ||--o{ MATCH_RESULT : scores
  CANDIDATE ||--|| CV_PROFILE : has
  CANDIDATE {
    string id
    string status
    instant submittedAt
    string slice
  }
  CV_PROFILE {
    int yearsExperience
    string education
    string currentTitle
  }
  REDACTED_SIGNAL {
    string field
    string category
  }
  MATCH_RESULT {
    string jobId
    string jobTitle
    int score
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| ExtractionAgent | AutonomousAgent | `application/ExtractionAgent.java` |
| MatchingAgent | AutonomousAgent | `application/MatchingAgent.java` |
| CvMatcherTasks | task constants | `application/CvMatcherTasks.java` |
| MatchingWorkflow | Workflow | `application/MatchingWorkflow.java` |
| CandidateEntity | EventSourcedEntity | `application/CandidateEntity.java` |
| CvIntakeQueue | EventSourcedEntity | `application/CvIntakeQueue.java` |
| MatchesView | View | `application/MatchesView.java` |
| CvIntakeConsumer | Consumer | `application/CvIntakeConsumer.java` |
| CvSimulator | TimedAction | `application/CvSimulator.java` |
| FairnessMonitor | TimedAction | `application/FairnessMonitor.java` |
| MatchingEndpoint | HttpEndpoint | `api/MatchingEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |
| Records | records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `extractStep` and `matchStep` call agents; each gets an explicit 60s `stepTimeout` (Lesson 4). `sanitizeStep` is deterministic and keeps the default.
- **Idempotency.** The workflow is keyed by `candidateId`; re-delivery of a `CvIntakeQueue` event with the same id is a no-op because the entity already holds later state. `CvIntakeConsumer` derives a stable workflow id from the submission id.
- **Recovery.** `defaultStepRecovery(maxRetries(2).failoverTo(MatchingWorkflow::error))` sends exhausted retries to a terminal error step rather than looping.
- **No saga.** Steps produce read-only artifacts (profile, redactions, scores); there is no external side effect to compensate. The review step is non-blocking — the pipeline completes whether or not a reviewer acts.
- **View indexing.** `MatchesView` exposes only `getAllCandidates`; status filtering is client-side because Akka cannot auto-index the enum column (Lesson 2).
