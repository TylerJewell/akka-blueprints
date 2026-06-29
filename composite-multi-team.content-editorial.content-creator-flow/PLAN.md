# PLAN — content-creator-flow

Architectural sketch. All four mermaid diagrams + the component table. The generated system renders these on the Architecture tab with the Lesson 24 theme variables and CSS overrides.

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

  SIM[TopicSimulator]:::ta
  QUEUE[InboundTopicQueue]:::ese
  CONS[TopicConsumer]:::cons
  WF[ContentWorkflow]:::wf
  RA[ResearchAgent]:::agent
  BA[BlogAgent]:::agent
  LA[LinkedInAgent]:::agent
  BR[BrandReviewer]:::agent
  QE[QualityEvaluator]:::agent
  ENT[CampaignEntity]:::ese
  VIEW[CampaignsView]:::view
  CEP[ContentEndpoint]:::ep
  AEP[AppEndpoint]:::ep

  SIM -.->|tick| QUEUE
  CEP -->|enqueueTopic| QUEUE
  QUEUE -.->|events| CONS
  CONS -->|start| WF
  WF -->|runSingleTask| RA
  WF -->|runSingleTask| BA
  WF -->|runSingleTask| LA
  WF -->|review| BR
  WF -->|evaluate| QE
  WF -->|commands| ENT
  ENT -.->|events| VIEW
  CEP -->|getAllCampaigns| VIEW
  AEP -->|static| AEP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant CEP as ContentEndpoint
  participant Q as InboundTopicQueue
  participant C as TopicConsumer
  participant WF as ContentWorkflow
  participant AG as Writer agents
  participant BR as BrandReviewer
  participant QE as QualityEvaluator
  participant E as CampaignEntity
  U->>CEP: POST /api/campaigns {topic}
  CEP->>Q: enqueueTopic
  Q-->>C: TopicEnqueued
  C->>WF: start(campaignId, topic)
  WF->>AG: research, then blog + linkedin
  AG-->>WF: typed artifacts
  WF->>E: recordResearch / recordDraft
  WF->>BR: review(outputs)
  Note over WF,BR: off-brand -> recordReviewBlocked, end at BLOCKED
  BR-->>WF: BrandVerdict(passed)
  WF->>E: recordReviewPassed
  WF->>QE: evaluate(campaign)
  QE-->>WF: QualityResult(score)
  WF->>E: recordEvaluation, recordCompletion
  E-->>CEP: SSE Campaign (COMPLETED)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> RESEARCHING: start
  RESEARCHING --> DRAFTING: ResearchCompleted
  DRAFTING --> REVIEWING: ContentDrafted
  REVIEWING --> BLOCKED: BrandReviewBlocked
  REVIEWING --> EVALUATING: BrandReviewPassed
  EVALUATING --> COMPLETED: QualityEvaluated + CampaignCompleted
  BLOCKED --> [*]
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  CAMPAIGN ||--o{ EVENT : emits
  CAMPAIGN {
    string id
    string topic
    string status
    string researchReport
    string blogPost
    string linkedInPost
    string brandVerdict
    double qualityScore
  }
  EVENT {
    string type
    string payload
  }
  CAMPAIGNS_VIEW {
    string id
    string status
  }
  CAMPAIGN ||--|| CAMPAIGNS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ResearchAgent | `application/ResearchAgent.java` |
| BlogAgent | `application/BlogAgent.java` |
| LinkedInAgent | `application/LinkedInAgent.java` |
| BrandReviewer | `application/BrandReviewer.java` |
| QualityEvaluator | `application/QualityEvaluator.java` |
| ContentTasks | `application/ContentTasks.java` |
| ContentWorkflow | `application/ContentWorkflow.java` |
| CampaignEntity | `domain/CampaignEntity.java` |
| InboundTopicQueue | `domain/InboundTopicQueue.java` |
| CampaignsView | `application/CampaignsView.java` |
| TopicConsumer | `application/TopicConsumer.java` |
| TopicSimulator | `application/TopicSimulator.java` |
| ContentEndpoint | `api/ContentEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts.** Every workflow step that calls an agent overrides the 5s default to 60s (Lesson 4): `researchStep`, `draftStep`, `reviewStep`, `evaluateStep`. `defaultStepRecovery(maxRetries(2).failoverTo(error))`.
- **Idempotency.** The campaign id is the workflow id; re-delivering a `TopicEnqueued` event with the same id is a no-op because the workflow is already started. The Consumer derives a deterministic id from the queue event offset where possible, otherwise a fresh UUID per topic.
- **No saga/compensation.** The pipeline is forward-only. The `BLOCKED` terminal state is a normal end, not a compensation — no outputs are published, so nothing needs rolling back.
- **View indexing.** `CampaignsView` exposes one query (`getAllCampaigns`) with no enum `WHERE` clause; status filtering happens client-side in the endpoint (Lesson 2).
