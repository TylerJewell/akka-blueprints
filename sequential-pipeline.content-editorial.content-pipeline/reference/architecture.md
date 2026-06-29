# Architecture — content-pipeline

The system is a sequential pipeline: one Workflow chains four stages, each handing a typed result to the next. The four mermaid diagrams below are the same sources the generated UI renders on the Architecture tab (with the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow).

## Component graph

Topics arrive from either the HTTP endpoint or the simulator, land on `InboundTopicQueue`, and a consumer starts one workflow per topic. The workflow drives the two AutonomousAgents and the critique Agent, writing each result as an event on `ArticleEntity`. `ArticlesView` projects those events for the UI list and SSE stream.

```mermaid
flowchart TB
  ContentEndpoint -->|enqueueTopic| InboundTopicQueue
  TopicSimulator -.->|drips canned topics| InboundTopicQueue
  InboundTopicQueue -.->|TopicQueued| TopicConsumer
  TopicConsumer -->|start| ContentPipelineWorkflow
  ContentPipelineWorkflow -->|runSingleTask| ResearchAgent
  ContentPipelineWorkflow -->|runSingleTask| WriterAgent
  ContentPipelineWorkflow -->|critique| CritiqueAgent
  ContentPipelineWorkflow -->|events| ArticleEntity
  ArticleEntity -.->|events| ArticlesView
  ArticlesView -->|read / SSE| ContentEndpoint
  AppEndpoint -->|static UI| ContentEndpoint
```

## Interaction sequence

The primary journey: submit a topic, then research → write → critique → publish, with the two guardrails firing inside the research and write stages.

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant EP as ContentEndpoint
  participant W as ContentPipelineWorkflow
  participant R as ResearchAgent
  participant Wr as WriterAgent
  participant Cr as CritiqueAgent
  participant E as ArticleEntity
  U->>EP: POST /api/topics {topic}
  EP->>W: start(articleId)
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

The article lifecycle. A failure in any agent-calling stage moves the article to `FAILED` (forward-only, no rollback).

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

See `../PLAN.md` for the component-to-file-path table and concurrency notes.
