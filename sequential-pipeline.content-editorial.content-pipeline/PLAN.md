# PLAN — content-pipeline

Architectural sketch for the sequential-pipeline content-editorial system. All four mermaid diagrams + the component table are below. The generated UI renders these on the Architecture tab with the Lesson 24 CSS overrides.

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

  ContentEndpoint:::ep -->|enqueueTopic| InboundTopicQueue:::ese
  TopicSimulator:::ta -.->|drips canned topics| InboundTopicQueue
  InboundTopicQueue -.->|TopicQueued| TopicConsumer:::cons
  TopicConsumer -->|start| ContentPipelineWorkflow:::wf
  ContentPipelineWorkflow -->|runSingleTask| ResearchAgent:::agent
  ContentPipelineWorkflow -->|runSingleTask| WriterAgent:::agent
  ContentPipelineWorkflow -->|critique| CritiqueAgent:::agent
  ContentPipelineWorkflow -->|events| ArticleEntity:::ese
  ArticleEntity -.->|events| ArticlesView:::view
  ArticlesView -->|read / SSE| ContentEndpoint
  AppEndpoint:::ep -->|static UI| ContentEndpoint
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant EP as ContentEndpoint
  participant Q as InboundTopicQueue
  participant C as TopicConsumer
  participant W as ContentPipelineWorkflow
  participant R as ResearchAgent
  participant Wr as WriterAgent
  participant Cr as CritiqueAgent
  participant E as ArticleEntity
  U->>EP: POST /api/topics {topic}
  EP->>Q: enqueueTopic
  Q-->>C: TopicQueued
  C->>W: start(articleId)
  W->>R: runSingleTask(RESEARCH)
  Note over R: before-tool-call guard filters sources by allowed domain
  R-->>W: ResearchBrief
  W->>E: recordResearch
  W->>Wr: runSingleTask(WRITE)
  Note over Wr: before-agent-response guard runs style/safety check
  Wr-->>W: ArticleDraft
  W->>E: recordDraft
  W->>Cr: critique(draft)
  Cr-->>W: CritiqueResult (non-blocking)
  W->>E: recordCritique
  W->>E: recordPublication
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RESEARCHING
  RESEARCHING --> WRITING: ResearchCompleted
  WRITING --> CRITIQUING: ArticleDrafted
  CRITIQUING --> PUBLISHED: ArticleCritiqued then ArticlePublished
  RESEARCHING --> FAILED: ArticleFailed
  WRITING --> FAILED: ArticleFailed
  CRITIQUING --> FAILED: ArticleFailed
  PUBLISHED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ARTICLE_ENTITY ||--o{ ARTICLE_EVENT : emits
  ARTICLE_ENTITY ||--|| ARTICLES_VIEW : projects
  INBOUND_TOPIC_QUEUE ||--o{ TOPIC_EVENT : emits
  ARTICLE_ENTITY {
    string id
    string topic
    string status
    double critiqueScore
    string publishedUrl
  }
  ARTICLE_EVENT {
    string type "ResearchCompleted|ArticleDrafted|ArticleCritiqued|ArticlePublished|ArticleFailed"
  }
  ARTICLES_VIEW {
    string id
    string status
    string title
  }
  TOPIC_EVENT {
    string type "TopicQueued"
  }
```

## Component table

| Component | Akka primitive | Path (generated) |
|---|---|---|
| ResearchAgent | AutonomousAgent | `application/ResearchAgent.java` |
| WriterAgent | AutonomousAgent | `application/WriterAgent.java` |
| CritiqueAgent | Agent | `application/CritiqueAgent.java` |
| ContentPipelineWorkflow | Workflow | `application/ContentPipelineWorkflow.java` |
| ArticleEntity | EventSourcedEntity | `domain/ArticleEntity.java` |
| InboundTopicQueue | EventSourcedEntity | `domain/InboundTopicQueue.java` |
| ArticlesView | View | `application/ArticlesView.java` |
| TopicConsumer | Consumer | `application/TopicConsumer.java` |
| TopicSimulator | TimedAction | `application/TopicSimulator.java` |
| ContentEndpoint | HttpEndpoint | `api/ContentEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |
| ContentPipelineTasks | companion | `application/ContentPipelineTasks.java` |

## Concurrency notes

- Workflow step timeouts: 60s on `researchStep`, `writeStep`, `critiqueStep` (agent calls run 10–30s; the 5s default times out — Lesson 4). `defaultStepRecovery(maxRetries(2).failoverTo(error))`; the `error` step writes `ArticleFailed`.
- Idempotency: `TopicConsumer` derives the workflow id from a fresh UUID per `TopicQueued` event; re-delivery of the same event starts at most one workflow because the workflow id is recorded on the article before the first step.
- No saga/compensation: the publish step is in-process and the pipeline is forward-only; a failure at any stage transitions the article to `FAILED` rather than rolling back prior stages.
- The critique stage is non-blocking — a low score records on the article but does not stop publishing.
